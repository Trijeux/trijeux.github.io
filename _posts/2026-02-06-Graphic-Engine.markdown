---
layout: post
title:  "Créer un moteur de rendu 3D avec OpenGL en c++"
date:   2026-02-06 17:40:00 +0100
categories: Blog Post Graphic Engine
---
## Pourquoi

Ce projet a été réalisé dans le cadre de mon cursus à la [SAE Institute de Genève](https://www.sae.edu/ch-ge/). L'objectif principal était d'appréhender les fondamentaux de la programmation d'un moteur graphique, en me concentrant plus spécifiquement sur l'API OpenGL

## Le projet

Ce projet est un moteur graphique multiplateforme tournant sous **Windows** et **Linux**. Vous pouvez télécharger les versions exécutables en cliquant sur les liens suivants :

* [Télécharger pour Windows](https://github.com/Trijeux/Graphic-Engine/releases/download/1.0/Graphic_Engine_win.zip)
* [Télécharger pour Linux](https://github.com/Trijeux/Graphic-Engine/releases/download/1.0/Graphic_Engine_linux.zip)

Vous pouvez également consulter la page dédiée au projet sur mon [Itch.io](https://trijeuxaxel.itch.io/graphic-engine).

Pour ceux qui souhaitent explorer directement le code source, l'intégralité du projet est disponible sur mon [répertoire GitHub](https://github.com/Trijeux/Graphic-Engine).

## L'objectif

L'objectif de ce projet était d'explorer les fondements de l'architecture d'un moteur graphique. Ce travail m'a permis d'approfondir des concepts avancés et d'implémenter des techniques modernes de rendu :

* **Le pipeline de shaders :** Écriture et gestion de Shaders GLSL pour le rendu et le post-process.
* **Calculs d'éclairage :** Implémentation de différents types de sources lumineuses (Directionnelle, Point et Spot lights) et intégration du modèle de réflexion.
* **Mapping avancé :** Utilisation de **Normal Maps** pour le relief des surfaces et de **Specular Maps** pour la gestion de la réflectivité.
* **Framebuffers (FBO) :** Mise en place de buffers personnalisés pour le rendu hors écran (*off-screen rendering*), étape indispensable pour les ombres et le post-process.
* **Système d'ombres :** Mise en place du **Shadow Mapping** via une *depth map* générée dans un framebuffer dédié.
* **Instancing :** Optimisation permettant de dessiner des milliers d'objets identiques en minimisant les appels de dessin (*Draw Calls*).
* **Deferred Rendering :** Architecture de rendu différé utilisant un **G-Buffer** (framebuffer multi-cibles) pour découpler la géométrie de l'éclairage.
* **Post-Processing & Effets :**
    * **SSAO (Screen Space Ambient Occlusion) :** Pour accentuer les ombres de contact et la profondeur.
    * **Bloom :** Simulation de l'éblouissement lumineux sur les zones à forte intensité.

## Stack Technique et Outils

Pour mener à bien ce projet, j'ai utilisé un environnement de développement moderne basé sur le langage **C++ 23** et l'API **OpenGL ES 3.0**. Voici les principales bibliothèques utilisées :

* **[C++ 23](https://en.cppreference.com/w/cpp/23.html) :** Pour bénéficier des dernières fonctionnalités du langage.
* **[CMake](https://cmake.org) :** Pour la gestion de la compilation et la génération des fichiers de projet.
* **[SDL3](https://wiki.libsdl.org/SDL3/FrontPage) :** Pour la gestion de la fenêtre, du contexte OpenGL et des entrées utilisateur.
* **[OpenGL ES 3.0](https://www.khronos.org/opengles/) :** L'API graphique utilisée pour le rendu.
* **[GLEW](https://glew.sourceforge.net) :** Pour le chargement des extensions OpenGL.
* **[ImGui](https://github.com/ocornut/imgui) :** Pour la création d'une interface utilisateur de debug dynamique.
* **[GLM](https://github.com/g-truc/glm) :** Une bibliothèque mathématique pour la manipulation des vecteurs et des matrices.
* **[Assimp](https://github.com/assimp/assimp) :** Pour l'importation de modèles 3D complexes (formats .obj, .fbx, etc.).
* **[GPR924-Engine](https://github.com/SAE-Geneve/GPR924-Engine) :** Une librairie interne développée par notre classe à la SAE, servant de base à l'architecture du moteur.