---
layout: post
title:  "Ui_Toolkit"
date:   2026-05-25 18:00:00 +0200
categories: Pres
hidden : true
---


<meta charset="utf-8" lang="fr">
<link rel="stylesheet" href="template/fonts.css">
<script>
window.courseConfig = {
    title: "UI Toolkit : un Visual Novel",
    instructor: "Nom du prof"
};
</script>
<script src="template/title-slide.js"></script>

## Le projet

On crée un court **visual novel** : *Le Phare de Kerlo*.

- 5 à 10 minutes de jeu
- 21 scènes, des choix, **5 fins**
- **Aucun sprite, aucun Canvas** : tout est fait avec **UI Toolkit**

![](images/uitoolkit/jeu_final.png style="height: 8rem")

---

## L'histoire en un coup d'œil

![](images/uitoolkit/arbre_histoire.png style="height: 8rem; float: right; margin-left: 1rem")

Léa arrive sur une île. Sa tante a disparu, le phare est éteint et une tempête arrive.

Deux sortes de choix :
- **Vrais choix** : ils changent la suite (et la fin)
- **Faux choix** : ils changent une réplique, puis l'histoire se rejoint

Question : pourquoi mettre des faux choix ?

---

## UI Toolkit vs UGUI (Canvas)

| | UGUI (Canvas) | UI Toolkit |
|---|---|---|
| Éléments | GameObjects | VisualElements |
| Structure | Hierarchy | Fichier **UXML** |
| Style | Inspector, un par un | Fichier **USS** (comme le CSS) |
| Inspiration | Unity "classique" | **Le web** |

UI Toolkit, c'est le système **recommandé** pour les nouvelles interfaces dans Unity 6.

---

## UI Toolkit = une page web

- **UXML** = le HTML → *quoi* est à l'écran
- **USS** = le CSS → *à quoi* ça ressemble
- **C#** = le JavaScript → *comment* ça réagit

Si vous savez faire un site web, vous savez déjà 80 % du travail.

---

## Les 5 fichiers du projet

| Fichier | Rôle |
|---|---|
| `VisualNovel.uxml` | La structure de l'écran |
| `VisualNovel.uss` | Le style |
| `phare_de_kerlo.json` | Toute l'histoire |
| `StoryData.cs` | Le format de l'histoire en C# |
| `VisualNovelController.cs` | Le cerveau qui relie tout |

**Règle d'or : le code ne contient aucune réplique.**

---

## Un seul GameObject

![](images/uitoolkit/hierarchy.png style="height: 7rem; float: right; margin-left: 1rem")

Pour afficher UI Toolkit en jeu, il faut **un** composant `UIDocument`, donc **un** GameObject.

C'est la "prise électrique" qui branche l'interface dans le jeu.

- *Create Empty* → `VisualNovel`
- *Add Component* → **UI Document**
- *Add Component* → **Visual Novel Controller**

---

## Panel Settings

![](images/uitoolkit/panel_settings.png style="height: 8rem; float: right; margin-left: 1rem")

L'asset qui règle **comment** l'interface s'affiche à l'écran.

- *Scale Mode* : **Scale With Screen Size**
- *Reference Resolution* : **1920 x 1080**
- *Match* : **0.5**

Sans ça : un texte minuscule en 4K, énorme sur un petit écran.

---

## UXML : des boîtes dans des boîtes

```xml
<ui:VisualElement name="dialogue-box" class="dialogue-box">
    <ui:Label name="speaker" class="speaker" text="Marius" />
    <ui:Label name="dialogue-text" class="dialogue-text" text="..." />
</ui:VisualElement>
```

- `name` = un identifiant **unique** (pour le code)
- `class` = un groupe de style (réutilisable)
- **L'ordre compte** : le premier est au fond, le dernier par-dessus

---

## Les composants de base

| Composant | C'est quoi ? | Dans le jeu |
|---|---|---|
| `VisualElement` | Une boîte vide (une `div`) | Décor, portrait |
| `Label` | Du texte | Nom, réplique |
| `Button` | Une boîte cliquable | Choix, Jouer |

Tous les autres composants sont des **VisualElement** avec des super-pouvoirs.

---

## UI Builder

![](images/uitoolkit/ui_builder.png style="height: 9rem; float: right; margin-left: 1rem")

Double-clic sur un `.uxml` → l'éditeur visuel.

- Glisser-déposer des éléments
- Voir la hiérarchie
- Tester les styles à la souris

**Astuce** : construisez à la souris, puis lisez le fichier texte généré.

---

## USS : le style

```css
.dialogue-box {
    position: absolute;
    bottom: 4%;
    left: 4%;
    right: 4%;
    background-color: rgba(10, 15, 30, 0.88);
    border-radius: 14px;
}
```

- `.nom` → une **classe**
- `#nom` → un **name**
- `Label` → un **type**

---

## La mise en page : Flexbox

![](images/uitoolkit/flexbox.png style="height: 7rem; float: right; margin-left: 1rem")

Par défaut, les éléments s'**empilent** de haut en bas.

- `flex-direction: row` → côte à côte
- `justify-content` → alignement principal
- `align-items` → alignement secondaire
- `flex-grow: 1` → "prends toute la place"

Pour un visual novel, on veut des **calques** : `position: absolute`.

---

## Pseudo-classes et sélecteurs

```css
.choice-button:hover  { background-color: rgb(60, 80, 130); }
.choice-button:active { background-color: rgb(255, 210, 120); }

.narration .speaker { display: none; }
```

- `:hover` → la souris est dessus
- `:active` → on clique
- `.a .b` → un `.b` **à l'intérieur** d'un `.a`

Le narrateur n'a pas de nom affiché : **zéro ligne de code**, que du USS.

---

## C# : les 6 gestes à connaître

```csharp
var label = root.Q<Label>("speaker");          // 1. Trouver
label.text = "Marius";                         // 2. Changer le texte
box.style.display = DisplayStyle.None;         // 3. Cacher
button.clicked += OnPlay;                      // 4. Réagir à un clic
box.EnableInClassList("narration", true);      // 5. Activer une classe
container.Add(new Button());                   // 6. Créer en code
```

Si vous comprenez ces 6 lignes, vous comprenez le projet.

---

## L'histoire est une donnée (JSON)

```json
{
  "id": "0.2",
  "background": "port",
  "lines": [ { "speaker": "Marius", "text": "Moi ? Marius..." } ],
  "choices": [],
  "choicesFrom": "0.1",
  "ending": ""
}
```

Une seule ligne pour tout charger :

`JsonUtility.FromJson<Story>(storyFile.text)`

---

## La boucle de jeu

![](images/uitoolkit/boucle_jeu.png style="height: 9rem; float: right; margin-left: 1rem")

1. Afficher une réplique
2. Attendre un clic
3. Encore une réplique ? → retour en 1
4. Sinon, la sortie du nœud :
   - `choices` → des boutons
   - `choicesFrom` → les boutons d'un autre nœud
   - `ending` → écran de fin

---

## Les choix : des boutons créés en code

```csharp
foreach (Choice choice in choices)
{
    Button button = new Button();
    button.text = choice.text;
    button.AddToClassList("choice-button");
    button.clicked += () => GoToNode(choice.target);
    choicesContainer.Add(button);
}
```

Le style vient de la classe `choice-button` : le code ne choisit **aucune** couleur.

---

## L'effet machine à écrire

Pas de coroutine, pas d'`Update()` : le **scheduler** de UI Toolkit.

```csharp
typingJob = label.schedule
    .Execute(TypeNextLetter)   // ce qu'on fait
    .Every(25);                // toutes les 25 ms
```

- `.StartingIn(1500)` → une seule fois, dans 1,5 s
- `typingJob.Pause()` → stop

Utilisé aussi pour la **lecture automatique**.

---

## L'historique : ScrollView

![](images/uitoolkit/historique.png style="height: 8rem; float: right; margin-left: 1rem")

Une boîte qui **défile** quand le contenu est trop grand.

```csharp
historyScroll.Add(entry);       // ajouter une ligne
historyScroll.ScrollTo(entry);  // descendre jusqu'à elle
```

Chaque réplique lue est ajoutée. Les choix faits apparaissent en surligné.

---

## Les templates UXML

Une ligne d'historique = un **modèle** dans son propre fichier : `HistoryEntry.uxml`.

```csharp
[SerializeField] VisualTreeAsset historyEntryTemplate;

var entry = historyEntryTemplate.Instantiate();
entry.Q<Label>("entry-text").text = "...";
```

Comme un **prefab**, mais pour l'interface.

---

## Les champs : une seule façon de les écouter

```csharp
speedSlider.RegisterValueChangedCallback(evt =>
{
    lettersPerSecond = evt.newValue;
});
```

Ça marche pareil pour **tous** les champs :

`Slider`, `Toggle`, `TextField`, `DropdownField`, `RadioButtonGroup`...

---

## Le panneau Options

![](images/uitoolkit/options.png style="height: 9rem; float: right; margin-left: 1rem")

- `TabView` + `Tab` → les onglets
- `Slider` → vitesse du texte
- `RadioButtonGroup` → taille du texte
- `DropdownField` → thème (Nuit, Parchemin, Clair)
- `Toggle` → lecture auto, portraits
- `Foldout` → section repliable "Avancé"
- `TextField` → prénom de l'héroïne (écran titre)

---

## Changer un thème entier en 1 ligne

Le code ne change **pas** les couleurs. Il change une **classe** :

```csharp
screen.EnableInClassList("theme-parchemin", true);
```

```css
.theme-parchemin .dialogue-box  { background-color: rgb(240, 225, 190); }
.theme-parchemin .dialogue-text { color: rgb(50, 35, 20); }
```

Le C# décide **quoi**, le USS décide **comment**.

---

## La galerie des fins : ListView

![](images/uitoolkit/galerie_fins.png style="height: 8rem; float: right; margin-left: 1rem")

Une liste **performante** : elle ne crée que les lignes visibles.

```csharp
list.itemsSource = allEndings;
list.makeItem = () => new Label();
list.bindItem = (element, i) =>
    ((Label)element).text = allEndings[i];
```

- `makeItem` → fabrique une ligne **vide**
- `bindItem` → la **remplit**

---

## ProgressBar et sauvegarde

"2 / 5 fins découvertes"

```csharp
progress.highValue = 5;
progress.value = 2;
progress.title = "2 / 5 fins découvertes";
```

Les fins trouvées sont gardées avec `PlayerPrefs`.

Pour remettre à zéro : *Edit → Clear All PlayerPrefs*.

---

## Les transitions USS

Animer **sans écrire de code** :

```css
.panel {
    opacity: 0;
    transition-property: opacity;
    transition-duration: 0.25s;
}
.panel--open { opacity: 1; }
```

- Le décor change de couleur en fondu
- Les panneaux apparaissent avec un zoom
- Les choix grossissent au survol

---

## Les images : Resources

Aucune ligne de code à changer : on dépose les fichiers au bon endroit.

- `Assets/Resources/Backgrounds/port.png`
- `Assets/Resources/Characters/marius.png`

```csharp
var image = Resources.Load<Texture2D>("Backgrounds/port");
background.style.backgroundImage = new StyleBackground(image);
```

Pas d'image ? Le jeu met une **couleur** à la place.

---

## Récap : 15 composants et outils

| Structure | Champs | Listes et affichage |
|---|---|---|
| VisualElement | TextField | ScrollView |
| Label | Slider | ListView |
| Button | Toggle | ProgressBar |
| TabView / Tab | DropdownField | Template UXML |
| Foldout | RadioButtonGroup | Transitions USS |

---

## Débogage

- **Écran vide** → Panel Settings ou Source Asset vide
- **NullReferenceException** → le `name` du UXML ≠ celui du `Q<>()`
- **Rien ne change** → regarder la Console en premier
- **Où suis-je dans l'histoire ?** → Options → Avancé → id du nœud

**UI Toolkit Debugger** : *Window → UI Toolkit → Debugger*

---

## À vous de jouer !

Du plus facile au plus difficile :

- Avancer le texte avec la touche **Espace**
- Sauvegarder les **options** avec `PlayerPrefs`
- Un bouton **"Passer"** qui affiche directement les choix
- Un bouton **"Effacer l'historique"**
- Retenir un choix dans une **variable** et l'utiliser plus loin
- Écrire **votre propre histoire** en JSON, sans toucher au code

---

## Conclusion

- **UXML** pour la structure, **USS** pour le style, **C#** pour le comportement
- Séparer les **données** (JSON) du **code**
- Le style passe par des **classes**, jamais par des couleurs dans le C#

Un visual novel, c'est surtout... **beaucoup d'interface** ;)

<script src="template/slide-init.js"></script>
