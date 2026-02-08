---
layout: post
title:  "Créer un moteur de rendu 3D avec OpenGL en c++"
date:   2026-02-06 17:40:00 +0100
categories: Blog Post Graphic Engine
---
## 1. Introduction & Motivation

Ce projet est né dans le cadre de mon cursus à la **SAE Institute de Genève** (2025-2026), où j'ai suivi le module de programmation graphique avancée. L'objectif ? Plonger au cœur du **pipeline de rendu moderne** en utilisant **OpenGL** pour comprendre de A à Z comment fonctionnent les moteurs 3D d'aujourd'hui.

**Pourquoi OpenGL ?**  
C'est une API accessible, cross-platform, et qui force à comprendre les bases (shaders GLSL, gestion GPU, maths linéaires) sans les abstractions trop hautes de Vulkan ou DirectX. J'ai voulu aller au-delà d'un simple "hello triangle" : implémenter un vrai moteur avec **deferred rendering**, ombres dynamiques, PBR basique, instancing massif et effets screen-space.

**Quand et où ?**  
Débuté en septembre 2025 à Genève, finalisé début 2026. Développé principalement sur Linux (Ubuntu) avec tests sur Windows. Environ 4-5 mois de travail intermittent, entre cours et nuits blanches.

**Ce qui a été accompli**  
Un moteur multiplateforme (Windows/Linux) capable de charger des modèles complexes, gérer des dizaines de lumières, appliquer SSAO, skybox, shadow mapping (directionnel + point lights), et instancier des milliers d'objets sans tuer les FPS.

Téléchargez les builds :  
- [Windows](https://github.com/Trijeux/Graphic-Engine/releases/download/1.0/Graphic_Engine_win.zip)  
- [Linux](https://github.com/Trijeux/Graphic-Engine/releases/download/1.0/Graphic_Engine_linux.zip)  

Code source complet : [GitHub – Trijeux/Graphic-Engine](https://github.com/Trijeux/Graphic-Engine)  
Démo playable : [Itch.io – Graphic Engine](https://trijeuxaxel.itch.io/graphic-engine)

**Stack technique**  
- C++23  
- CMake + vcpkg  
- SDL3 (fenêtre & input)  
- GLEW (extensions OpenGL)  
- GLM (maths)  
- ImGui (debug UI)  
- Assimp (modèles)  
- stb_image (textures)  
- GPR924-Engine (base SAE interne)

## 2. Architecture du Projet

L’organisation du code repose sur une séparation stricte entre la gestion des ressources, la logique de scène et le pipeline de rendu. Cette modularité permet de maintenir une base de code propre et d’ajouter facilement de nouvelles fonctionnalités graphiques avancées (instancing, SSAO, ombres multi-sources, skybox…).

### Gestion des Ressources
Pour optimiser les performances et éviter les fuites mémoire, le cycle de vie des objets GPU est centralisé :

- **Shaders** : Compilation, linking et gestion intuitive des uniforms via des shaders dédiés par passe (geometry, lighting, shadow, ssao, skybox…).
- **Texturation** : Intégration de **stb_image** pour le chargement d’images. Gestion automatique des unités de texture, modes de wrapping, filtrage (anisotrope et mipmaps).
- **Meshes & Models** : Structures optimisées (VAO, VBO, EBO) avec support intensif d’**instancing** (`instancing_mesh.h`, `instancing_model.h`, `instancing_cube_mesh.h`) pour rendre efficacement des milliers d’objets identiques. Grâce à **Assimp**, le moteur traite des formats complexes (obj, fbx, gltf…) en extrayant sommets, normales, UV et indices.

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
  - Spéculaire / Roughness / autres propriétés PBR
- **Lighting Pass** : Seconde passe en screen-space (`deferred.frag`, `generic_light.frag`) qui calcule l’éclairage + ombres à partir du G-Buffer. Cette séparation découple la complexité géométrique du nombre de lumières, permettant de gérer efficacement des dizaines de sources (point + directional).
- **Fonctionnalités avancées** :
  - **Shadow mapping** directionnel (`shadow_map.h`) et omnidirectionnel pour les point lights (`point_shadow_map.h`).
  - **SSAO** (Screen-Space Ambient Occlusion) avec blur (`ssao.h`, `ssao.frag`, `ssao_blur.frag`).
  - **Skybox** (`skybox.h`, `skybox.frag|vert`).
  - Post-processing basique (blur, fullscreen quad via `screen.frag|vert`, `blur.frag`).
  - Gizmos de debug pour les lumières (`light_gizmo.h`).

La scène finale est orchestrée depuis `Final_Scene.main.cc` (point d’entrée) et `Final_Scene.h`, avec un **LightManager** (`light_manager.h`) pour centraliser les sources lumineuses.

## 3. Chargement des Modèles et Textures

Le chargement efficace des modèles 3D et des textures est au cœur des performances du moteur, surtout dans une architecture deferred où les données géométriques et matériaux doivent être extraites et stockées de manière optimale dès la geometry pass.

#### Système de Mesh et Instancing
Le moteur utilise des structures de données OpenGL modernes pour minimiser les appels draw et maximiser le throughput GPU :

- **Vertex Array Objects (VAO)**, **Vertex Buffer Objects (VBO)** et **Element Buffer Objects (EBO)** sont créés et configurés une seule fois par mesh.
- Les attributs de vertex incluent typiquement :
  - Position (vec3)
  - Normales (vec3)
  - Coordonnées de texture (vec2)
  - Tangentes / Bitangentes (pour normal mapping – probable dans deferred PBR)
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
Les textures sont chargées depuis le dossier `data/` et associées aux matériaux PBR pour le G-Buffer :

- Intégration de **stb_image** pour les formats PNG, JPG, HDR…
- Textures typiques pour deferred PBR :
  - Albedo / Base Color
  - Normal Map
  - Roughness / Metallic / Specular
  - Ambient Occlusion (optionnel)
  - Emissive (pour les sources lumineuses auto-éclairées)
- Gestion GPU :
  - Bind automatique sur les bonnes unités de texture.
  - Paramètres : wrapping (repeat/clamp), filtrage (linear/mipmap), anisotropic filtering (si disponible).
  - Génération de mipmaps pour réduire l’aliasing sur les objets éloignés.

Ce système permet de gérer des scènes avec des matériaux variés, des assets professionnels issus de Blender/Maya/Substance, et un grand nombre d’objets instanciés.

## 4. Éclairage et Shaders

Le cœur visuel du moteur : un **éclairage dynamique** avec **deferred shading**.

**Comment ça a été fait**  
1. **Geometry Pass** : Les modèles sont rendus dans le G-Buffer (4-5 render targets) → albedo (base color), normals (world-space), positions, metallic/roughness/specular. Shaders : `g_buffer.vert` + `g_buffer.frag`.  
2. **Lighting Pass** : Un fullscreen quad sample le G-Buffer et calcule l'éclairage pour chaque lumière (Blinn-Phong ou mini-PBR). Shaders : `deferred.vert` + `generic_light.frag` / `point_light.frag`.  
3. **Types de lumières** : Directional (soleil avec shadow map cascade), Point (omni-shadow avec cubemap depth), Spot (si ajouté).  
4. **Shadow Mapping** : Passe dédiée pour chaque lumière → depth map depuis la vue de la lumière, puis comparaison dans le lighting shader. Classes : `shadow_map.h`, `point_shadow_map.h`.  
5. **SSAO** : Post-process screen-space : sample le depth + normals pour estimer l'occlusion ambiante, puis bilateral blur (`ssao.frag` + `ssao_blur.frag`).  
6. **Skybox** : Cubemap chargé et rendu en dernier avec depth <= pour horizon cohérent.

**Résultat**  
Des scènes avec dizaines de lumières dynamiques sans explosion de draw calls, ombres nettes sur directional, ambiance réaliste grâce à SSAO. Le bloom est prévu mais pas encore fully implémenté (juste un blur basique pour tests).

![G-Buffer visualization – Albedo, Normals, Positions, Specular](https://assets.alexandria.kodeco.com/books/6efcbfb1f9f195ced75feb43f202ab14a8cd87832cef086976b16658e23944a8/images/709ac5d6266df9e6f594192232eb917f/original.png)  
*(Exemple typique de G-Buffer – remplace par ta propre capture si possible !)*

![SSAO before / after – ombres de contact plus profondes](https://doc.x3dom.org/tutorials/lighting/ssao/ssao_before_after.jpg)  
*(Before/After SSAO – effet visible sur les coins et contacts)*

![Shadow mapping – depth map et scène finale avec ombres](https://www.codinglabs.net/public/contents/tutorial_opengl_deferred_rendering_shadow_mapping/images/deferred_rendering_shadow_mapping_2.jpg)  
*(Visualisation shadow map + résultat)*

## 5. Défis et Solutions

Quelques galères classiques et comment je les ai surmontées :

- **Trop de draw calls avec 1000+ objets** → Passage à **instancing** (SSBO pour matrices model par instance). Résultat : FPS multipliés par 10 sur scènes denses.  
- **Artefacts shadow acne / peter panning** → Ajout de bias dynamique + PCF (Percentage Closer Filtering) dans le shader shadow sampling.  
- **SSAO noisy** → Implémenté un blur bilateral pour garder les bords nets tout en lissant le bruit.  
- **Performance lighting pass** → Limité les lumières actives par tile (simple clustered forward-like, mais basique).  
- **Debug difficile** → ImGui + gizmos lumières (`light_gizmo.h`) pour déplacer/éditer en live.

Ces problèmes m'ont forcé à penser optimisation dès le début – une excellente leçon !

## 6. Conclusion et Prochaines Étapes

Ce projet m'a appris énormément : le vrai coût du rendu moderne n'est pas dans la géométrie, mais dans l'éclairage et les post-effects. Passer à **deferred** a été le turning point pour scaler les lumières et effets.

**Ce qui a été accompli**  
- Deferred PBR-like avec G-Buffer  
- Multi-lights + shadows (dir + point)  
- Instancing massif  
- SSAO + skybox  
- Caméra FPS fluide + debug UI

**Ce que j'aimerais ajouter**  
- Full PBR (IBL, environment map reflection)  
- Bloom + tonemapping HDR  
- Screen-Space Reflections (SSR)  
- Volumetric lights / god rays  
- Peut-être une migration Vulkan un jour...

Merci à la SAE Genève, aux profs, et à la communauté OpenGL pour les ressources !  

N'hésite pas à cloner, tester, ou me poser des questions sur GitHub.  

Code : [github.com/Trijeux/Graphic-Engine](https://github.com/Trijeux/Graphic-Engine)  
Itch.io : [trijeuxaxel.itch.io/graphic-engine](https://trijeuxaxel.itch.io/graphic-engine)

À bientôt pour la V2 ! 🚀