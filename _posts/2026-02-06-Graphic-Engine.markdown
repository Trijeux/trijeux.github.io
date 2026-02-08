---
layout: post
title:  "Créer un moteur de rendu 3D avec OpenGL en c++"
date:   2026-02-06 17:40:00 +0100
categories: Blog Post Graphic Engine
---
## 1. Introduction & Motivation

Ce projet est né dans le cadre de mon cursus à la **SAE Institute de Genève** (2025-2026), où j'ai suivi le module de programmation graphique. 
L'objectif ? Plonger au cœur du **pipeline de rendu moderne** en utilisant **OpenGL** pour comprendre de A à Z comment fonctionnent les moteurs 3D d'aujourd'hui.

**Pourquoi OpenGL ?**  
C'est une API accessible, cross-platform, et qui force à comprendre les bases (shaders GLSL, gestion GPU, maths linéaires) sans les abstractions trop hautes de Vulkan ou DirectX. J'ai voulu aller au-delà d'un simple "hello triangle" : implémenter un vrai moteur avec **deferred rendering**, ombres dynamiques, instancing massif et effets screen-space.

**Quand et où ?**  
Débuté en septembre 2025 à Genève, finalisé début 2026. Développé principalement sur Windows avec tests sur Linux (Ubuntu). Environ 4-5 mois de travail intermittent, entre cours et nuits blanches.

**Ce qui a été accompli**  
Un moteur multiplateforme (Windows/Linux) capable de charger des modèles complexes, gérer des dizaines de lumières, appliquer SSAO, skybox, shadow mapping (directionnel + point lights), et instancier des milliers d'objets sans tuer les FPS.

Téléchargez les builds :  
- [Windows](https://github.com/Trijeux/Graphic-Engine/releases/download/1.0/Graphic_Engine_win.zip)  
- [Linux](https://github.com/Trijeux/Graphic-Engine/releases/download/1.0/Graphic_Engine_linux.zip)  

Code source complet : [GitHub – Trijeux/Graphic-Engine](https://github.com/Trijeux/Graphic-Engine)  
Démo playable : [Itch.io – Graphic Engine](https://trijeuxaxel.itch.io/graphic-engine)

**Stack technique**  
* **[C++ 23](https://en.cppreference.com/w/cpp/23.html) :** Pour bénéficier des dernières fonctionnalités du langage.
* **[CMake](https://cmake.org) :** Pour la gestion de la compilation et la génération des fichiers de projet.
* **[SDL3](https://wiki.libsdl.org/SDL3/FrontPage) :** Pour la gestion de la fenêtre, du contexte OpenGL et des entrées utilisateur.
* **[OpenGL ES 3.0](https://www.khronos.org/opengles/) :** L'API graphique utilisée pour le rendu.
* **[GLEW](https://glew.sourceforge.net) :** Pour le chargement des extensions OpenGL.
* **[ImGui](https://github.com/ocornut/imgui) :** Pour la création d'une interface utilisateur de debug dynamique.
* **[GLM](https://github.com/g-truc/glm) :** Une bibliothèque mathématique pour la manipulation des vecteurs et des matrices.
* **[Assimp](https://github.com/assimp/assimp) :** Pour l'importation de modèles 3D complexes (formats .obj, .fbx, etc.).
* **[GPR924-Engine](https://github.com/SAE-Geneve/GPR924-Engine) :** Une librairie interne développée par notre classe à la SAE, servant de base à l'architecture du moteur.

## 2. Architecture du Projet

L’organisation du code repose sur une séparation stricte entre la gestion des ressources, la logique de scène et le pipeline de rendu. Cette modularité permet de maintenir une base de code propre et d’ajouter facilement de nouvelles fonctionnalités graphiques avancées (instancing, SSAO, ombres multi-sources, skybox…).

### Gestion des Ressources
Pour optimiser les performances et éviter les fuites mémoire, le cycle de vie des objets GPU est centralisé :

- **Shaders** : Compilation, linking et gestion intuitive.

```cpp
void Shader::Load(std::string_view shader_path, GLenum shader_stage) {
  size_t data_size = 0;
  auto* shader_content = static_cast<const char*>(SDL_LoadFile(shader_path.data(), &data_size));
  if (shader_content == nullptr) {
    throw std::runtime_error(std::format("Failed to load shader {}", shader_path));
  }
  LoadFromMemory(shader_content, shader_stage);
}
```

```cpp
void Pipeline::Load(const Shader& vertex_shader, const Shader& fragment_shader) {
  auto& pipelineInfo = get();
  pipelineInfo.pipeline_name = glCreateProgram();
  if (vertex_shader->shader_name == 0) {
    throw std::runtime_error("Vertex shader was not loaded");
  }
  if (fragment_shader->shader_name == 0) {
    throw std::runtime_error("Fragment shader was not loaded");
  }
  glAttachShader(pipelineInfo.pipeline_name, vertex_shader->shader_name);
  glAttachShader(pipelineInfo.pipeline_name, fragment_shader->shader_name);
  glLinkProgram(pipelineInfo.pipeline_name);
  // Check link status...
}
```

- **Texturation** : Intégration de **stb_image** pour le chargement d’images. Gestion automatique des unités de texture, modes de wrapping, filtrage (anisotrope et mipmaps).

```cpp
inline unsigned int TextureFromFile(const char *path, const std::string &directory) {
  std::string filename = std::string(path);
  filename = directory + '/' + filename;

  unsigned int textureID;
  glGenTextures(1, &textureID);

  int w, h, c;
  unsigned char *data = stbi_load(filename.c_str(), &w, &h, &c, 0);

  if (data) {
    GLenum format;
    if (c == 1) format = GL_RED;
    else if (c == 3) format = GL_RGB;
    else if (c == 4) format = GL_RGBA;
    else format = GL_RGB;

    glBindTexture(GL_TEXTURE_2D, textureID);
    glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
    glTexImage2D(GL_TEXTURE_2D, 0, format, w, h, 0, format, GL_UNSIGNED_BYTE, data);
    glGenerateMipmap(GL_TEXTURE_2D);

    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    stbi_image_free(data);
  } else {
    std::cerr << "Texture failed to load at path: " << filename << std::endl;
  }

  return textureID;
}
```

- **Meshes & Models** : Structures optimisées (VAO, VBO, EBO) avec support intensif d’**instancing** (`instancing_mesh.h`, `instancing_model.h`, `instancing_cube_mesh.h`) pour rendre efficacement des milliers d’objets identiques. Grâce à **Assimp**, le moteur traite des formats complexes (obj, fbx, gltf…) en extrayant sommets, normales, UV et indices.

```cpp
void SetupInstancingAttributes(const GLuint instanceVBO) const {
  glBindVertexArray(VAO);
  glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);

  for (int i = 0; i < 4; i++) {
    constexpr std::size_t vec4Size = sizeof(glm::vec4);
    glEnableVertexAttribArray(5 + i);
    glVertexAttribPointer(5 + i, 4, GL_FLOAT, GL_FALSE, 4 * vec4Size, reinterpret_cast<void *>(i * vec4Size));
    glVertexAttribDivisor(5 + i, 1);
  }

  glBindVertexArray(0);
  glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```

```cpp
void InstancingModel::Load(const std::string& path) {
  minBounds = glm::vec3(1e9);
  maxBounds = glm::vec3(-1e9);

  Assimp::Importer importer;
  const aiScene *scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs | aiProcess_GenNormals | aiProcess_JoinIdenticalVertices | aiProcess_CalcTangentSpace);

  if(!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) {
    std::cout << "ERROR::ASSIMP::" << importer.GetErrorString() << std::endl;
    return;
  }

  size_t lastSlash = path.find_last_of("/\\");
  directory = (lastSlash == std::string::npos) ? "." : path.substr(0, lastSlash);

  processNode(scene->mRootNode, scene);

  boundingCenter = (minBounds + maxBounds) / 2.0f;
  boundingRadius = glm::length(maxBounds - boundingCenter);
}
```

### Système de Caméra
Caméra dynamique conçue pour une exploration fluide des scènes 3D :

- **Style FPS** : Navigation libre à 360° basée sur les angles d’Euler (**Pitch** / **Yaw**).
- **Calcul de direction** : Vecteur avant et **LookAt** mis à jour dynamiquement.
- **Réactivité** : Matrices de **vue** et de **projection** recalculées avec le **DeltaTime** et les entrées utilisateur pour un mouvement indépendant du framerate.

### Pipeline de Rendu – Deferred Shading
Le moteur adopte une architecture **Deferred Rendering** (rendu différé), particulièrement adaptée aux scènes complexes avec de multiples sources lumineuses :

- **Geometry Pass** : Première passe qui remplit le **G-Buffer** (`g_buffer.h` / `g_buffer.frag|vert`) sans calcul d’éclairage. La scène est décomposée en textures techniques :
  - Albedo
  - Normales
  - Positions (world-space)
  - Spéculaire

```cpp
#version 300 es
precision highp float;
layout (location = 0) out vec4 gPosition;
layout (location = 1) out vec4 gNormal;
layout (location = 2) out vec4 gAlbedoSpec;

in vec3 fragWorldPos;
in vec2 fragTexCoords;
in vec3 Normal;
in mat3 TBN;

uniform sampler2D material_diffuse;
uniform sampler2D material_specular;
uniform sampler2D material_normal;
uniform bool useNormalMap;

void main() {
    gPosition = vec4(fragWorldPos, 1.0);

    vec3 N = normalize(Normal);
    if(useNormalMap) {
        vec3 normalMap = texture(material_normal, fragTexCoords).rgb;
        normalMap = normalMap * 2.0 - 1.0;
        N = normalize(TBN * normalMap);
    }
    gNormal = vec4(N * 0.5 + 0.5, 1.0);

    gAlbedoSpec.rgb = texture(material_diffuse, fragTexCoords).rgb;
    gAlbedoSpec.a = texture(material_specular, fragTexCoords).r;
}
```

- **Lighting Pass** : Seconde passe en screen-space (`deferred.frag`, `generic_light.frag`) qui calcule l’éclairage + ombres à partir du G-Buffer. Cette séparation découple la complexité géométrique du nombre de lumières, permettant de gérer efficacement des dizaines de sources (point + directional).

```cpp
#version 300 es
precision mediump float;

in vec2 fragTexCoords;
in vec3 fragNormal;
in vec3 fragWorldPos;

uniform float opacity;
uniform vec3 viewPos;
uniform vec3 lightColor;

uniform int lightType;
uniform int enableAttenuation;

uniform vec3 lightPos;
uniform vec3 lightDir;

uniform float cutOff;
uniform float outerCutOff;

uniform float ambientStrength;
uniform float specularPow;

uniform sampler2D material_diffuse;
uniform sampler2D material_specular;

out vec4 outColor;

void main() {
    vec3 diffuseColor = vec3(texture(material_diffuse, fragTexCoords));
    vec3 specularMap = vec3(texture(material_specular, fragTexCoords));

    vec3 ambient = ambientStrength * lightColor * diffuseColor;

    vec3 norm = normalize(fragNormal);
    vec3 L;

    float attenuation = 1.0;
    float intensity = 1.0;

    if(lightType == 1)// Directional Light
    {
        L = normalize(-lightDir);
    }
    else// Point Light
    {
        L = normalize(lightPos - fragWorldPos);

        if (enableAttenuation == 1) { // Attenuation
            float distance = length(lightPos - fragWorldPos);
            // Constantes pour une portée d'environ 100 unités
            float constant = 1.0;
            float linear = 0.045;
            float quadratic = 0.0075;

            attenuation = 1.0 / (constant + linear * distance + quadratic * (distance * distance));
        }

        if (lightType == 2) { // Spot Light
            float theta = dot(L, normalize(-lightDir));
            float epsilon = cutOff - outerCutOff;
            intensity = clamp((theta - outerCutOff) / epsilon, 0.0, 1.0);
        }
    }

    float diff = max(dot(norm, L), 0.0);
    vec3 diffuse =  diff * diffuseColor * lightColor;

    vec3 V = normalize(viewPos - fragWorldPos);
    vec3 R = reflect(-L, norm);

    float spec = pow(max(dot(V, R), 0.0), float(specularPow));

    vec3 specular = spec * specularMap * lightColor;

    vec3 lighting = (ambient + diffuse + specular) * intensity * attenuation;

    outColor = vec4(lighting * opacity, 1.0);
    //outColor = vec4(diffuseColor, 1.0f);
}
```

- **Fonctionnalités avancées** :
  - **Shadow mapping** directionnel (`shadow_map.h`) et omnidirectionnel pour les point lights (`point_shadow_map.h`).
  - **SSAO** (Screen-Space Ambient Occlusion) avec blur (`ssao.h`, `ssao.frag`, `ssao_blur.frag`).
  - **Skybox** (`skybox.h`, `skybox.frag|vert`).
  - Post-processing basique (blur, fullscreen quad via `screen.frag|vert`, `blur.frag`).
  - Gizmos de debug pour les lumières (`light_gizmo.h`).

La scène finale est orchestrée depuis `Final_Scene.main.cc` (point d’entrée) et `Final_Scene.h`, avec un **LightManager** (`light_manager.h`) pour centraliser les sources lumineuses.

```cpp
void Final_Scene::Draw() {
    // Pass 0: Shadow maps (directional + point)
    // Pass 1: Geometry → G-Buffer
    // Pass 1.5: SSAO
    // Pass 2: Lighting → HDR
    // Pass 3: Forward gizmos + skybox
    // Pass 4: Post-process → écran final
}
```

## 3. Chargement des Modèles et Textures

Le chargement efficace des modèles 3D et des textures est au cœur des performances du moteur, surtout dans une architecture deferred où les données géométriques et matériaux doivent être extraites et stockées de manière optimale dès la geometry pass.

#### Système de Mesh et Instancing
Le moteur utilise des structures de données OpenGL modernes pour minimiser les appels draw et maximiser le throughput GPU :

- **Vertex Array Objects (VAO)**, **Vertex Buffer Objects (VBO)** et **Element Buffer Objects (EBO)** sont créés et configurés une seule fois par mesh.
- Les attributs de vertex incluent typiquement :
  - Position (vec3)
  - Normales (vec3)
  - Coordonnées de texture (vec2)
  - Tangentes / Bitangentes (pour normal mapping)
- Support intensif de **l’instancing** via des classes dédiées :
  - `instancing_mesh.h` / `instancing_cube_mesh.h` : un seul mesh partagé par des milliers d’instances.
  - `instancing_model.h` : gestion des transformations par instance (matrice model via SSBO ou UBO).
  - Cela permet de rendre des scènes très denses (forêts, villes, particules…) avec un coût minimal en draw calls.

#### Chargement des Modèles
Les modèles sont chargés via **Assimp** (Open Asset Import Library) pour supporter nativement de nombreux formats standards :

- Formats pris en charge : .obj, .fbx, .gltf / .glb, .dae, .blend…
- Extraction automatique des meshes, matériaux, animations et hiérarchies de nœuds (`model.h` / `model_instance.h` / `model_transform.h`).
- Support des transformations hiérarchiques (`model_transform.h`, `cube_transform.h`) pour les scènes composées.
- Préparation immédiate des buffers GPU (VAO/VBO/EBO) lors du chargement.

#### Gestion des Textures
Les textures sont chargées depuis le dossier `data/` et associées aux G-Buffer :

- Intégration de **stb_image** pour les formats PNG, JPG, HDR…
- Textures typiques pour deferred:
  - Albedo / Base Color
  - Normal Map
  - Specular
  - Ambient Occlusion (optionnel)
- Gestion GPU :
  - Bind automatique sur les bonnes unités de texture.
  - Paramètres : wrapping (repeat/clamp), filtrage (linear/mipmap).
  - Génération de mipmaps pour réduire l’aliasing sur les objets éloignés.

Ce système permet de gérer des scènes avec des matériaux variés, des assets professionnels issus de Blender/Maya/Substance, et un grand nombre d’objets instanciés.

## 4. Éclairage et Shaders

Le cœur visuel du moteur : un **éclairage dynamique** avec **deferred shading**.

**Comment ça a été fait**  
1. **Geometry Pass** : Les modèles sont rendus dans le G-Buffer (4-5 render targets) → albedo (base color), normals (world-space), positions, metallic/roughness/specular. Shaders : `g_buffer.vert` + `g_buffer.frag`. 

```cpp
#version 300 es
precision highp float;
layout (location = 0) out vec4 gPosition;
layout (location = 1) out vec4 gNormal;
layout (location = 2) out vec4 gAlbedoSpec;

in vec3 fragWorldPos;
in vec2 fragTexCoords;
in vec3 Normal;
in mat3 TBN;

uniform sampler2D material_diffuse;
uniform sampler2D material_specular;
uniform sampler2D material_normal;
uniform bool useNormalMap;

void main() {
    // 1. Position
    gPosition = vec4(fragWorldPos, 1.0);

    // 2. Normal (Apply Normal Map)
    vec3 N = normalize(Normal);
    if(useNormalMap) {
        vec3 normalMap = texture(material_normal, fragTexCoords).rgb;
        normalMap = normalMap * 2.0 - 1.0;
        N = normalize(TBN * normalMap);
    }
    gNormal = vec4(N * 0.5 + 0.5, 1.0);

    // 3. Color + Specular
    gAlbedoSpec.rgb = texture(material_diffuse, fragTexCoords).rgb;
    gAlbedoSpec.a = texture(material_specular, fragTexCoords).r;
}
```

2. **Lighting Pass** : Un fullscreen quad sample le G-Buffer et calcule l'éclairage pour chaque lumière. Shaders : `deferred.vert` + `generic_light.frag` / `point_light.frag`.

```cpp
#version 300 es
precision highp float;

layout (location = 0) out vec4 FragColor;   // Output 0: Scene Color
layout (location = 1) out vec4 BrightColor; // Output 1: Brightness (For Bloom)

in vec2 TexCoords;

uniform sampler2D gPosition;
uniform sampler2D gNormal;
uniform sampler2D gAlbedoSpec;

// --- Shadow Map Inputs ---
uniform highp sampler2DShadow shadowMap;
uniform mat4 lightSpaceMatrix;

uniform sampler2D ssao;
uniform bool useSSAO;

uniform vec3 viewPos;
uniform float specularPow;
uniform int debugMode; // 0=None, 1=Pos, 2=Norm, 3=Alb, 4=Spec


#define MAX_POINT_SHADOWS 4
uniform highp samplerCube pointShadowMaps[MAX_POINT_SHADOWS];
uniform float pointFarPlane;


// --- Light Structures ---
struct DirLight {
    vec3 direction;
    vec3 color;
    float ambientStrength;
};

struct PointLight {
    vec3 position;
    vec3 color;
    float constant;
    float linear;
    float quadratic;
};

struct SpotLight {
    vec3 position;
    vec3 direction;
    vec3 color;
    float cutOff;
    float outerCutOff;
    float constant;
    float linear;
    float quadratic;
};

// --- Uniform Arrays ---
#define MAX_POINT_LIGHTS 50
#define MAX_SPOT_LIGHTS 50

uniform bool hasDirLight;
uniform DirLight dirLight;

uniform bool hasFlashLight;
uniform SpotLight flashLight;

uniform int pointLightCount;
uniform PointLight pointLights[MAX_POINT_LIGHTS];

uniform int spotLightCount;
uniform SpotLight spotLights[MAX_SPOT_LIGHTS];

// --- Function Prototypes ---
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir, vec3 albedo, float specular, vec3 fragPos);
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir, vec3 albedo, float specular, int index);
vec3 CalcSpotLight(SpotLight light, vec3 normal, vec3 fragPos, vec3 viewDir, vec3 albedo, float specular);
float ShadowCalculation(vec3 fragPosWorld, vec3 normal, vec3 lightDir);
float PointShadowCalculation(highp samplerCube shadowMap, vec3 fragPos, vec3 fragPos);

// Workaround function to select the correct cube map based on index
float CalcPointShadowSelect(int index, vec3 fragPos, vec3 lightPos) {
    if (index == 0) return PointShadowCalculation(pointShadowMaps[0], fragPos, lightPos);
    if (index == 1) return PointShadowCalculation(pointShadowMaps[1], fragPos, lightPos);
    if (index == 2) return PointShadowCalculation(pointShadowMaps[2], fragPos, lightPos);
    if (index == 3) return PointShadowCalculation(pointShadowMaps[3], fragPos, lightPos);
    return 0.0; // No shadow if index >= 4
}

void main() {
    // 1. Retrieve data from G-Buffer
    vec3 FragPos = texture(gPosition, TexCoords).rgb;

    vec3 encodedNormal = texture(gNormal, TexCoords).rgb;
    vec3 Normal = normalize(encodedNormal * 2.0 - 1.0);

    vec3 Albedo = texture(gAlbedoSpec, TexCoords).rgb;
    float Specular = texture(gAlbedoSpec, TexCoords).a;

    // --- DEBUG MODES ---
    if (debugMode == 1) { FragColor = vec4(abs(FragPos) / 20.0, 1.0); BrightColor = vec4(0.0); return; }
    if (debugMode == 2) { FragColor = vec4(encodedNormal, 1.0); BrightColor = vec4(0.0); return; }
    if (debugMode == 3) { FragColor = vec4(Albedo, 1.0); BrightColor = vec4(0.0); return; }
    if (debugMode == 4) { FragColor = vec4(vec3(Specular), 1.0); BrightColor = vec4(0.0); return; }

    // --- LIGHTING CALCULATION ---
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 result = vec3(0.0);

    // A. Directional Light (Sun) + SHADOWS
    if(hasDirLight) {
        // We pass FragPos now to calculate shadows
        result += CalcDirLight(dirLight, Normal, viewDir, Albedo, Specular, FragPos);
    }

    // B. Point Lights
    for(int i = 0; i < pointLightCount; i++) {
        result += CalcPointLight(pointLights[i], Normal, FragPos, viewDir, Albedo, Specular, i);
    }

    // C. Spot Lights
    for(int i = 0; i < spotLightCount; i++) {
        result += CalcSpotLight(spotLights[i], Normal, FragPos, viewDir, Albedo, Specular);
    }

    // D. Flashlight
    if(hasFlashLight) {
        result += CalcSpotLight(flashLight, Normal, FragPos, viewDir, Albedo, Specular);
    }

    // 1. Output Final Scene Color
    FragColor = vec4(result, 1.0);

    // 2. Output Brightness (BLOOM LOGIC)
    float brightness = dot(result, vec3(0.2126, 0.7152, 0.0722));
    if(brightness > 0.9)
    BrightColor = vec4(result, 1.0);
    else
    BrightColor = vec4(0.0, 0.0, 0.0, 1.0);
}

// --- Shadow Calculation Function ---
float ShadowCalculation(vec3 fragPosWorld, vec3 normal, vec3 lightDir)
{
    vec4 fragPosLightSpace = lightSpaceMatrix * vec4(fragPosWorld, 1.0);

    // Perspective divide
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    projCoords = projCoords * 0.5 + 0.5;

    if(projCoords.z > 1.0) return 0.0;

    // Bias
    float bias = max(0.005 * (1.0 - dot(normal, lightDir)), 0.00005);

    float shadow = 0.0;
    vec2 texelSize = 1.0 / vec2(textureSize(shadowMap, 0));

    // Use a tighter 3x3 loop (Radius 1) because hardware filtering handles the micro-blur
    // increasing performance and quality simultaneously.
    for(int x = -1; x <= 1; ++x)
    {
        for(int y = -1; y <= 1; ++y)
        {
            // We subtract bias from the Z comparison value directly
            float pcfDepth = projCoords.z - bias;

            // The hardware does the comparison and blending for us!
            shadow += texture(shadowMap, vec3(projCoords.xy + vec2(x, y) * texelSize, pcfDepth));
        }
    }
    shadow /= 9.0;

    // The texture returns 1.0 for "Lit" and 0.0 for "Shadow"
    // So we invert it to match your logic (1.0 = In Shadow)
    return 1.0 - shadow;
}

// Point Shadow (Perspective, Cube Map)
float PointShadowCalculation(highp samplerCube shadowMap, vec3 fragPos, vec3 lightPos)
{
    vec3 fragToLight = fragPos - lightPos;
    float currentDepth = length(fragToLight);

    float shadow = 0.0;
    float bias = 0.15; // Higher bias needed for perspective shadows

    // Sample the linear depth we wrote in shadow_point.frag
    float closestDepth = texture(shadowMap, fragToLight).r;
    closestDepth *= pointFarPlane; // Remap [0,1] back to [0, far]

    if(currentDepth - bias > closestDepth)
    shadow = 1.0;

    return shadow;
}

// ------------------------------------------------------------------
//                      Lighting Calculations
// ------------------------------------------------------------------

vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir, vec3 albedo, float specular, vec3 fragPos) {
    vec3 lightDir = normalize(-light.direction);

    // Diffuse
    float diff = max(dot(normal, lightDir), 0.0);

    // Specular (Blinn-Phong)
    vec3 halfwayDir = normalize(lightDir + viewDir);
    float spec = pow(max(dot(normal, halfwayDir), 0.0), specularPow);

    // --- SHADOW CALCULATION ---
    // 0.0 = Shadow, 1.0 = Lit (Or partial for PCF)
    float shadow = ShadowCalculation(fragPos, normal, lightDir);

    // Ambient Occlusion
    float ao = useSSAO ? texture(ssao, TexCoords).r : 1.0;
    ao *= ao;

    // Combine
    vec3 ambient = light.ambientStrength * light.color * albedo * ao;

    // Apply (1.0 - shadow) to Diffuse and Specular only
    vec3 diffuse = (1.0 - shadow) * (light.color * diff * albedo);
    vec3 specColor = (1.0 - shadow) * (light.color * spec * specular);

    return (ambient + diffuse + specColor);
}

vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir, vec3 albedo, float specular, int index) {
    vec3 lightDir = normalize(light.position - fragPos);

    // Diffuse
    float diff = max(dot(normal, lightDir), 0.0);

    // Specular
    vec3 halfwayDir = normalize(lightDir + viewDir);
    float spec = pow(max(dot(normal, halfwayDir), 0.0), specularPow);

    // Attenuation
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));

    // Ambient Occlusion
    float ao = useSSAO ? texture(ssao, TexCoords).r : 1.0;
    ao *= ao;

    // --- POINT SHADOW LOGIC ---
    float shadow = 0.0;
    if(index < MAX_POINT_SHADOWS) {
        shadow = CalcPointShadowSelect(index, fragPos, light.position);
    }

    vec3 diffuse = (1.0 - shadow) * (light.color * diff * albedo);
    vec3 specColor = (1.0 - shadow) * (light.color * spec * specular);
    vec3 ambient = 0.05 * light.color * albedo * ao;

    return (ambient + diffuse + specColor) * attenuation;
}

vec3 CalcSpotLight(SpotLight light, vec3 normal, vec3 fragPos, vec3 viewDir, vec3 albedo, float specular) {
    vec3 lightDir = normalize(light.position - fragPos);

    // Diffuse
    float diff = max(dot(normal, lightDir), 0.0);

    // Specular
    vec3 halfwayDir = normalize(lightDir + viewDir);
    float spec = pow(max(dot(normal, halfwayDir), 0.0), specularPow);

    // Attenuation
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));

    // Spot Intensity
    float theta = dot(lightDir, normalize(-light.direction));
    float epsilon = light.cutOff - light.outerCutOff;
    float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);

    // Ambient Occlusion
    float ao = useSSAO ? texture(ssao, TexCoords).r : 1.0;
    ao *= ao;

    // Combine
    vec3 ambient = 0.0 * light.color * albedo * ao;
    vec3 diffuse = light.color * diff * albedo;
    vec3 specColor = light.color * spec * specular;

    return (ambient + diffuse + specColor) * attenuation * intensity;
}
```

3. **Types de lumières** : Directional (soleil avec shadow map cascade), Point (omni-shadow avec cubemap depth), Spot (si ajouté).  
4. **Shadow Mapping** : Passe dédiée pour chaque lumière → depth map depuis la vue de la lumière, puis comparaison dans le lighting shader. Classes : `shadow_map.h`, `point_shadow_map.h`.  

```cpp
// === Shadow Sampling (Directional Light) ===
// Récupération de la position dans l'espace lumière
vec4 fragPosLightSpace = lightSpaceMatrix * vec4(FragPos, 1.0);
vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;  // perspective divide
projCoords = projCoords * 0.5 + 0.5;  // [-1,1] → [0,1]

// Profondeur actuelle du fragment (du point de vue lumière)
float currentDepth = projCoords.z;

// On lit la profondeur stockée dans la shadow map
float closestDepth = texture(shadowMap, projCoords.xy).r;

// Shadow acne / peter panning fix → bias dynamique
float bias = max(0.005 * (1.0 - dot(gNormal, lightDir)), 0.0005);

// === PCF (Percentage Closer Filtering) - anti-aliasing des bords d'ombre ===
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);  // taille d'un texel

for(int x = -1; x <= 1; ++x) {
    for(int y = -1; y <= 1; ++y) {
        float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r;
        shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;
    }
}
shadow /= 9.0;  // 3×3 = 9 échantillons

// Si on est hors de la shadow map → pas d'ombre (ou ombre complète selon ton choix)
if(projCoords.z > 1.0) shadow = 0.0;

float shadowFactor = 1.0 - shadow;  // 0 = pleine ombre, 1 = pas d'ombre
```

```cpp
// === Point Light Shadow Sampling ===
vec3 fragToLight = FragPos - lightPos;
float currentDepth = length(fragToLight);
float farPlane = 25.0;  // doit matcher ton far dans la passe shadow

// Direction vers la lumière → sélectionne la bonne face du cubemap
vec3 sampleDir = normalize(fragToLight);

// Lecture profondeur dans le cubemap (utilise texture() sur samplerCubeShadow)
float closestDepth = texture(pointShadowMap, vec4(sampleDir, currentDepth / farPlane)).r;

// Bias pour point lights (souvent plus petit)
float bias = 0.005;

// PCF pour cubemap (plus simple ou avec kernel sphérique)
float shadow = currentDepth - bias > closestDepth * farPlane ? 1.0 : 0.0;

// Option PCF 3D (plus cher) : sampler plusieurs directions proches
// ...

float shadowFactor = 1.0 - shadow;
```

5. **SSAO** : Post-process screen-space : sample le depth + normals pour estimer l'occlusion ambiante, puis bilateral blur (`ssao.frag` + `ssao_blur.frag`).

```cpp
#version 300 es
precision highp float;
out float FragColor;
in vec2 TexCoords;

uniform sampler2D gPosition;
uniform sampler2D gNormal;
uniform sampler2D texNoise;

uniform vec3 samples[64];
uniform mat4 projection;
uniform mat4 view;
uniform float radius;
uniform float bias;

void main() {
    // Convert World Pos to View Pos
    vec3 fragPosWorld = texture(gPosition, TexCoords).xyz;
    vec3 fragPos = (view * vec4(fragPosWorld, 1.0)).xyz;

    // Convert World Normal to View Normal
    vec3 normalWorld = normalize(texture(gNormal, TexCoords).rgb * 2.0 - 1.0);
    vec3 normal = mat3(view) * normalWorld;

    vec2 noiseScale = vec2(textureSize(gPosition, 0)) / 4.0;
    vec3 randomVec = texture(texNoise, TexCoords * noiseScale).xyz;

    vec3 tangent = normalize(randomVec - normal * dot(randomVec, normal));
    vec3 bitangent = cross(normal, tangent);
    mat3 TBN = mat3(tangent, bitangent, normal);

    float occlusion = 0.0;

    for(int i = 0; i < 64; ++i) {
        vec3 samplePos = TBN * samples[i];
        samplePos = fragPos + samplePos * radius;

        vec4 offset = vec4(samplePos, 1.0);
        offset = projection * offset;
        offset.xyz /= offset.w;
        offset.xyz = offset.xyz * 0.5 + 0.5;

        vec3 samplePosWorld = texture(gPosition, offset.xy).xyz;
        float sampleDepth = (view * vec4(samplePosWorld, 1.0)).z;

        float rangeCheck = smoothstep(0.0, 1.0, radius / abs(fragPos.z - sampleDepth));
        occlusion += (sampleDepth >= samplePos.z + bias ? 1.0 : 0.0) * rangeCheck;
    }
    FragColor = 1.0 - (occlusion / 64.0);
}
```

6. **Skybox** : Cubemap chargé et rendu en dernier avec depth <= pour horizon cohérent.

**Résultat**  
Des scènes avec dizaines de lumières dynamiques sans explosion de draw calls, ombres nettes sur directional, ambiance réaliste grâce à SSAO. Le bloom est prévu mais pas encore fully implémenté (juste un blur basique pour tests).

G-Buffer visualization 
![Albedo](https://github.com/Trijeux/trijeux.github.io/blob/main/image/Albedo.png)
![Normals]()
![Positions]()
![Specular]()

![SSAO before]()
![SSAO]()
![SSAO after]()

![Shadow mapping]() 
![scène finale avec ombres]()  

## 5. Défis et Solutions

Quelques problèmes classiques et comment je les ai surmontées :

- **Trop de draw calls avec 1000+ objets** → Passage à **instancing** (SSBO pour matrices model par instance). Résultat : FPS multipliés par 10 sur scènes denses.  

```cpp
void Draw(common::Pipeline &pipeline, const int instanceCount) const {
            if (instanceCount <= 0) return;
            // Unbind textures first to be clean
            glActiveTexture(GL_TEXTURE0); glBindTexture(GL_TEXTURE_2D, 0);
            glActiveTexture(GL_TEXTURE1); glBindTexture(GL_TEXTURE_2D, 0);
            glActiveTexture(GL_TEXTURE2); glBindTexture(GL_TEXTURE_2D, 0);

            // Bind Textures (Same logic as standard Draw)
            for(const auto & texture : textures) {
                std::string name = texture.type;
                if(name == "texture_diffuse") {
                    glActiveTexture(GL_TEXTURE0);
                    glBindTexture(GL_TEXTURE_2D, texture.id);
                    pipeline.SetInt("material_diffuse", 0);
                }
                else if(name == "texture_specular") {
                    glActiveTexture(GL_TEXTURE1);
                    glBindTexture(GL_TEXTURE_2D, texture.id);
                    pipeline.SetInt("material_specular", 1);
                }
                else if(name == "texture_normal") {
                    glActiveTexture(GL_TEXTURE2);
                    glBindTexture(GL_TEXTURE_2D, texture.id);
                    pipeline.SetInt("material_normal", 2);
                }
            }

            glBindVertexArray(VAO);
            //Instanced Draw Call
            glDrawElementsInstanced(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, nullptr, instanceCount);

            glBindVertexArray(0);
            glActiveTexture(GL_TEXTURE0);
        }
```

- **Artefacts shadow acne / peter panning** → Ajout de bias dynamique + PCF (Percentage Closer Filtering) dans le shader shadow sampling. 

```cpp
float shadow = 0.0;
    vec2 texelSize = 1.0 / vec2(textureSize(shadowMap, 0));

    // Use a tighter 3x3 loop (Radius 1) because hardware filtering handles the micro-blur
    // increasing performance and quality simultaneously.
    for(int x = -1; x <= 1; ++x)
    {
        for(int y = -1; y <= 1; ++y)
        {
            // We subtract bias from the Z comparison value directly
            float pcfDepth = projCoords.z - bias;

            // The hardware does the comparison and blending for us!
            shadow += texture(shadowMap, vec3(projCoords.xy + vec2(x, y) * texelSize, pcfDepth));
        }
    }
    shadow /= 9.0;
```

- **SSAO noisy** → Implémenté un blur bilateral pour garder les bords nets tout en lissant le bruit.  
- **Performance lighting pass** → Limité les lumières actives par tile (simple clustered forward-like, mais basique).  
- **Debug difficile** → ImGui + gizmos lumières (`light_gizmo.h`) pour déplacer/éditer en live.

```cpp
void LightGizmo::Draw(const glm::mat4& view, const glm::mat4& proj, const glm::vec3& position, const glm::vec3& color, float scale) {
        pipeline.Bind();

        glm::mat4 model = glm::mat4(1.0f);
        model = glm::translate(model, position);
        model = glm::scale(model, glm::vec3(scale));

        pipeline.SetMat4("view", glm::value_ptr(view));
        pipeline.SetMat4("projection", glm::value_ptr(proj));
        pipeline.SetMat4("model", glm::value_ptr(model));
        pipeline.SetVec3("objectColor", color.x, color.y, color.z);

        glBindVertexArray(VAO);
        glDrawArrays(GL_TRIANGLES, 0, 36);
        glBindVertexArray(0);
    }
```

Ces problèmes m'ont forcé à penser optimisation dès le début – une excellente leçon !

## 6. Conclusion et Prochaines Étapes

Ce projet m'a appris énormément : le vrai coût du rendu moderne n'est pas dans la géométrie, mais dans l'éclairage et les post-effects. Passer à **deferred** a été le turning point pour scaler les lumières et effets.

**Ce qui a été accompli**  
- Deferred + Bloom
- G-Buffer  
- Multi-lights + shadows (dir + point)  
- Instancing massif  
- SSAO + skybox  
- Caméra FPS fluide + debug UI

Merci à la SAE Genève, aux profs, et à la communauté OpenGL pour les ressources !  