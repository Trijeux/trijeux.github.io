---
layout: post
title:  "Rollback Netcode — Jeu de Sumo en Ligne"
date:   2026-04-29 13:00:00 +0200
categories: Blog_Post Rollback Network
hidden : false
---

## Introduction

**Pour mon projet sommatif, j'ai développé un jeu de sumo en ligne en C++ avec du rollback netcode.** Le concept est simple : deux joueurs s'affrontent dans une arène circulaire et doivent pousser l'adversaire hors des limites pour marquer des points.

Ce qui rend ce projet intéressant, ce n'est pas le gameplay en lui-même — c'est tout ce qui se passe **sous le capot** pour que deux joueurs puissent jouer ensemble en ligne sans lag perceptible et sans désynchronisation. Dans cet article, j'explique les technologies utilisées, les problèmes que j'ai rencontrés et comment je les ai résolus, ainsi que des conseils pour les futurs étudiants qui devront travailler sur un projet similaire.

<img src="/image/Sumo_Game.png" alt="Sumo Game — Mode local" width="800">

---

## 1. Les Technologies Utilisées

### Photon Realtime

Pour la communication réseau, j'utilise **[Photon Realtime](https://www.photonengine.com/)** (ExitGames SDK). Photon gère la connexion entre les joueurs : création de salons, matchmaking, et envoi/réception de messages.

J'utilise deux types d'envoi :
- **Unreliable (UDP)** — pour les **inputs des joueurs** (envoyés chaque frame). La perte d'un paquet n'est pas grave grâce au système de redondance (expliqué plus bas)
- **Reliable (TCP)** — pour les **checksums et l'état du jeu**, qui doivent absolument arriver à destination

### Rollback Netcode

Le **rollback** est une technique de netcode qui permet de jouer en ligne **sans attendre** la réception des inputs de l'adversaire. Le principe :

1. Chaque joueur **simule le jeu localement** en prédisant les inputs de l'adversaire (on garde le dernier input connu)
2. Quand les vrais inputs arrivent du réseau, si la prédiction était **correcte**, tout continue normalement
3. Si la prédiction était **fausse**, le jeu **revient en arrière** (rollback) jusqu'au dernier état confirmé et **re-simule** toutes les frames avec les bons inputs

Pour implémenter ça, j'utilise **deux modèles** qui tournent en parallèle :
- **confirm_model** — le modèle autoritaire qui ne contient que des frames confirmées (les deux joueurs ont envoyé leurs inputs)
- **current_model** — le modèle spéculatif qui est en avance et affiche le jeu au joueur

Quand un rollback est nécessaire, `current_model` est **réinitialisé** à partir de `confirm_model`, puis toutes les frames entre les deux sont re-simulées avec les vrais inputs.

### Arithmétique en Point Fixe (Fixed-Point)

C'est probablement **la partie la plus importante du projet**. Pour que le rollback fonctionne, les deux machines doivent produire des résultats **exactement identiques** quand elles simulent les mêmes frames avec les mêmes inputs. Or, les **nombres à virgule flottante (float)** ne garantissent pas ça — deux CPU différents peuvent donner des résultats légèrement différents pour le même calcul.

La solution : remplacer tous les floats de la simulation par des **nombres en point fixe 16.16**. Chaque valeur est stockée dans un `int32`, avec 16 bits pour la partie entière et 16 bits pour la partie fractionnaire. Toutes les opérations (addition, multiplication, racine carrée, etc.) sont faites en **arithmétique entière pure**, ce qui garantit un résultat identique sur n'importe quelle machine.

```cpp
// Exemple simplifié de multiplication en point fixe
// (a.raw * b.raw) >> 16 avec un intermédiaire int64 pour éviter l'overflow
Fixed operator*(Fixed a, Fixed b) {
    return Fixed::FromRaw(
        static_cast<int32_t>((static_cast<int64_t>(a.raw) * b.raw) >> 16)
    );
}
```

Les floats sont utilisés **uniquement pour le rendu** — jamais dans la logique de jeu.

---

## 2. Le Système de Checksum et la Détection de Desync

### Le Problème

Même avec du point fixe, des **désynchronisations** (desyncs) peuvent arriver — par exemple si l'ordre d'itération des collisions diffère entre les deux machines, ou si un état n'est pas correctement rollbacké. Le problème, c'est que quand un desync arrive, **il est très difficile de savoir d'où il vient** si on n'a pas les bons outils.

### La Solution : Checksums et Sub-Checksums

J'ai mis en place un système de **checksum** basé sur **Adler-32** qui vérifie que l'état du jeu est identique entre les deux joueurs.

Le joueur **master** (joueur 1) envoie son checksum toutes les 10 frames confirmées. Le joueur **client** (joueur 2) compare avec son propre checksum. Si ça ne match pas — c'est un desync.

Mais le plus utile, c'est le système de **sub-checksums**. Au lieu de hasher tout l'état en un seul bloc, je le divise en **7 catégories** :

| Index | Catégorie | Ce qu'elle contient |
|-------|-----------|---------------------|
| 0 | **Game State** | État de la partie, timer, gagnant du round |
| 1 | **Player Info** | Scores, flags de chute |
| 2 | **Positions** | Positions X/Y des deux joueurs (point fixe) |
| 3 | **Velocities** | Vélocités X/Y des deux joueurs (point fixe) |
| 4 | **Momentum** | Momentum + uncapped timer |
| 5 | **PrevDir** | Direction précédente des joueurs |
| 6 | **Inputs** | Inputs réseau reçus |

Quand un desync est détecté, le client renvoie ses sub-checksums au master, qui compare **catégorie par catégorie**. Ça permet de voir immédiatement si le problème vient des **positions**, des **vélocités**, des **inputs**, etc.

```
// Exemple de log de desync
[DESYNC] Frame 247 — Mismatch:
  POSITIONS: DIFFER
  VELOCITIES: DIFFER
  MOMENTUM: OK
  INPUTS: OK
  → Le problème vient de la physique, pas des inputs
```

### Correction Automatique

En plus de la détection, le master envoie aussi **l'état complet du jeu** (30 valeurs uint32 sérialisées) avec le checksum. Si le client détecte un desync, il **applique l'état du master** directement et re-simule à partir de là. Le jeu se corrige donc automatiquement sans que le joueur le remarque (dans la plupart des cas).

---

## 3. Les Problèmes Rencontrés et Comment Je Les Ai Résolus

### Problème 1 : Desyncs causés par les floats

**Symptôme :** Les checksums divergeaient après quelques secondes de jeu, même quand les deux joueurs faisaient les mêmes actions.

**Cause :** Les calculs de physique utilisaient des `float`, et les deux machines produisaient des résultats légèrement différents (différences d'arrondi CPU).

**Solution :** J'ai implémenté une classe `Fixed` en **point fixe 16.16** et converti toute la simulation (positions, vélocités, momentum, friction, accélération) en arithmétique entière. La conversion de float à fixed-point utilise une décomposition directe des bits IEEE 754 pour éviter toute dépendance au mode d'arrondi du CPU.

### Problème 2 : Impossible de trouver la source des desyncs

**Symptôme :** Le checksum global ne matchait pas, mais je n'avais aucun moyen de savoir **quelle partie** de l'état était différente.

**Cause :** Un seul checksum sur tout l'état = aucune granularité pour le debug.

**Solution :** J'ai divisé l'état en **7 sub-checksums** (positions, vélocités, inputs, etc.). Quand un desync est détecté, le client envoie ses sub-checksums au master qui les compare un par un et affiche exactement quelles catégories divergent. Ça m'a permis de trouver les bugs en **minutes au lieu d'heures**.

### Problème 3 : Ordre non-déterministe des collisions

**Symptôme :** Desyncs intermittents qui ne se reproduisaient pas toujours.

**Cause :** Les paires de collision retournées par le QuadTree n'étaient pas toujours dans le même ordre sur les deux machines.

**Solution :** Ajout d'un `std::sort()` sur les paires de collision **avant** de déclencher les callbacks. Les deux machines traitent donc les collisions dans un ordre identique, peu importe l'ordre interne du QuadTree.

### Problème 4 : Perte de paquets réseau

**Symptôme :** Des rollbacks excessifs et des corrections visibles quand la connexion était instable.

**Cause :** Un seul paquet perdu = un input manquant = prédiction forcée.

**Solution :** Chaque tick, j'envoie une **fenêtre de redondance de 16 frames** d'historique d'inputs (sous forme de keymask 4 bits). Si un paquet est perdu, le prochain paquet contient tout l'historique nécessaire pour rattraper le retard. La reconstruction des inputs est **déterministe** — les keymasks sont convertis en vecteurs de direction avec des constantes exactes (`0.70710678118654752440` pour les diagonales).

```
 Joueur 1 (Master)                              Joueur 2 (Client)
 ═══════════════════                             ═══════════════════

 Frame F:                                        Frame F:
 ┌─────────────────────┐                         ┌─────────────────────┐
 │ 1. Lire input local │                         │ 1. Lire input local │
 │    (keymask 4 bits)  │                         │    (keymask 4 bits)  │
 └────────┬────────────┘                         └────────┬────────────┘
          │                                               │
          │  ── UDP (unreliable) ──────────────>          │
          │     [16 frames d'historique]                   │
          │                                               │
          │          <────────────── UDP (unreliable) ──  │
          │              [16 frames d'historique]          │
          │                                               │
 ┌────────▼────────────┐                         ┌────────▼────────────┐
 │ 2. Input différent?  │                         │ 2. Input différent?  │
 │    NON → continuer   │                         │    NON → continuer   │
 │    OUI → rollback    │                         │    OUI → rollback    │
 │    + re-simuler      │                         │    + re-simuler      │
 └────────┬────────────┘                         └────────┬────────────┘
          │                                               │
 ┌────────▼────────────┐                         ┌────────▼────────────┐
 │ 3. Tick simulation   │                         │ 3. Tick simulation   │
 │    (point fixe)      │                         │    (point fixe)      │
 └────────┬────────────┘                         └────────┬────────────┘
          │                                               │
     Toutes les 10 frames confirmées:                     │
          │                                               │
          │  ── TCP (reliable) ────────────────>          │
          │     [checksum + état complet]                  │
          │                                               │
          │                                      ┌────────▼────────────┐
          │                                      │ 4. Checksum match?   │
          │                                      │    OUI → tout va bien│
          │                                      │    NON → DESYNC!     │
          │                                      │    → appliquer état  │
          │                                      │      du master       │
          │          <────────────── TCP ───────  │    → renvoyer sub-  │
          │              [sub-checksums]           │      checksums      │
          │                                      └─────────────────────┘
 ┌────────▼────────────┐
 │ 5. Comparer sub-     │
 │    checksums pour    │
 │    trouver la source │
 │    du desync         │
 └─────────────────────┘
```

---

## 4. Conseils pour les Futurs Étudiants

### Conseils Techniques

**1. Commencez par le déterminisme — pas par le réseau.**
Avant même de toucher à Photon ou au rollback, assurez-vous que votre simulation est **100% déterministe**. Si deux instances du jeu reçoivent les mêmes inputs, elles doivent produire le **même état, bit pour bit**. Testez ça en local d'abord. Si ce n'est pas déterministe, le rollback ne marchera jamais.

**2. N'utilisez PAS de floats dans votre simulation.**
Les floats ne sont pas déterministes entre différentes machines (et parfois même entre différents compilateurs). Utilisez du **point fixe** ou des **entiers** pour tout ce qui touche à la logique de jeu. Les floats c'est uniquement pour le rendu.

**3. Mettez en place des sub-checksums dès le début.**
Ne vous contentez pas d'un seul checksum global. Divisez votre état en catégories et hashez-les séparément. Quand un desync arrive (et il va arriver), vous allez vouloir savoir **exactement** quelle partie de l'état diverge. Sans ça, vous allez chercher pendant des heures.

**4. Triez vos collisions.**
Si vous utilisez un QuadTree ou n'importe quel spatial partitioning, l'ordre des paires de collision peut varier. Triez-les avant de les traiter. C'est une ligne de code qui peut vous sauver des jours de debug.

**5. Envoyez plus d'inputs que nécessaire.**
Ne vous fiez pas à un seul paquet par frame. Envoyez une fenêtre de redondance avec l'historique des dernières frames. Si un paquet est perdu, le prochain contient tout ce qu'il faut pour rattraper.

### Conseils de Gestion de Projet

**1. Testez le réseau le plus tôt possible.**
Ne développez pas tout votre gameplay en local pour ensuite "ajouter le réseau". Les problèmes de réseau sont fondamentalement différents des problèmes locaux. Avoir une connexion basique entre deux machines dès le début vous permet de détecter les problèmes de déterminisme avant qu'ils ne s'accumulent.

**2. Gardez votre simulation simple.**
Plus votre simulation est complexe, plus il y a de chances d'introduire du non-déterminisme. Commencez avec le minimum (mouvement + collision) et ajoutez des features une par une, en vérifiant le déterminisme à chaque étape.

**3. Logguez tout.**
Quand un desync arrive, vous avez besoin de savoir **exactement** ce qui s'est passé, frame par frame. Mettez en place un système de logs détaillé dès le début — ça va vous sauver énormément de temps.

**4. Ne sous-estimez pas le temps de debug réseau.**
Débugger des problèmes de réseau prend beaucoup plus de temps que débugger du code local. Les bugs sont **intermittents**, **difficiles à reproduire**, et nécessitent souvent de coordonner deux instances du jeu. Planifiez votre temps en conséquence.

---

## Conclusion

Ce projet m'a appris qu'un bon netcode, c'est **beaucoup plus** que juste envoyer des données d'un point A à un point B. C'est un système complet qui demande du **déterminisme absolu**, des **outils de debug solides**, et une architecture qui sépare clairement la simulation de la présentation.

Le rollback netcode est une technique puissante, mais elle repose sur une fondation fragile : si votre simulation n'est pas déterministe, **rien ne fonctionne**. Le point fixe, les sub-checksums et la redondance d'inputs sont les trois piliers qui m'ont permis de livrer un jeu en ligne fonctionnel.

<img src="/image/Sumo_Game_online.png" alt="Sumo Game — Mode en ligne" width="800">
