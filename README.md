# Animation2D


# SpriteBox 

## 1. Présentation Générale

**SpriteBox ** est un outil de découpage et de prévisualisation d'animations 2D développé en C++ avec la bibliothèque **SDL3** et l'interface graphique **Dear ImGui**.

L'objectif principal du logiciel est de permettre à un utilisateur de charger une "Sprite Sheet" (planche de sprites), d'extraire automatiquement des frames d'animation par un algorithme de détection de contours (*Flood Fill*), et de visualiser l'animation en temps réel avec des réglages de vitesse (FPS) et de positionnement.

---

## 2. Architecture du Projet

Le projet est segmenté en plusieurs modules pour séparer la logique métier, l'interface utilisateur et les algorithmes de traitement d'image :

* **`Utils.h`** : Définitions globales, structures de données de base et inclusions des bibliothèques.
* **`Application.h/cpp`** : Gestion de l'état global de l'application, initialisation et cycle de vie.
* **`Algorithme.h/cpp`** : Logique de détection des sprites et traitement des pixels.
* **`Ui.h/cpp`** : Gestion de toutes les fenêtres et menus ImGui.
* **`main.cpp`** : Point d'entrée et boucle de rendu principale.

---

## 3. Structures de Données (`Utils.h` & `Application.h`)

### `AppState`

C'est le "cerveau" du programme. Cette structure contient tout l'état de l'application à un instant T.

* `window` & `renderer` : Objets SDL3 pour l'affichage.
* `sheetSurface` & `sheetTexture` : Stockent la planche de sprites actuelle (Surface pour l'analyse de pixels, Texture pour l'affichage GPU).
* `spriteBank[MAX_SPRITES]` : Un tableau statique stockant les sprites extraits.
* `bgR, bgG, bgB` : La couleur du fond détectée pour la transparence.

### `Sprite`

Représente une frame individuelle extraite.

* `texture` : La texture découpée et nettoyée.
* `rect` : Sa position et taille d'origine sur la planche.

### `SavedAnim`

Structure utilisée pour la bibliothèque d'animations sauvegardées dans le fichier `.sba`.

---

## 4. Logique de Détection (`Algorithme.cpp`)

L'une des parties les plus complexes est la fonction `DetectBox`. Contrairement à un simple découpage par grille, SpriteBox utilise un algorithme de **Flood Fill (Remplissage par diffusion)**.

### Fonctionnement de `DetectBox` :

1. **Point de départ** : L'utilisateur clique sur un pixel du sprite.
2. **Analyse de voisinage** : Le programme regarde les 8 pixels environnants.
3. **Critère de propagation** : Si le pixel voisin n'est pas "fond" (selon la `tolerance` et la couleur de fond), il est ajouté à une pile (`stack`).
4. **Calcul de la Bounding Box** : À chaque pixel valide trouvé, les bornes `minX, maxX, minY, maxY` sont mises à jour.
5. **Optimisation** : Une allocation dynamique sur le tas (`malloc`) est utilisée pour la pile et le tableau `visited` afin d'éviter les *Stack Overflow* sur de grandes images.

```cpp
// Extrait de l'algorithme de voisinage
for (int dy = -1; dy <= 1; dy++) {
    for (int dx = -1; dx <= 1; dx++) {
        // ... vérification des limites et de la couleur ...
        if (!visited[index] && !IsBackground(s, nx, ny, br, bg, bb, tolerance)) {
            visited[index] = true;
            stack[stackTop++] = { nx, ny }; // Empilement
        }
    }
}

```

---

## 5. Interface Utilisateur (`Ui.cpp`)

L'interface est divisée en 5 zones majeures :

### A. Menu Principal (`Menu`)

Permet d'accéder à la bibliothèque d'animations chargées depuis le disque et de basculer l'affichage des fenêtres.

### B. Création d'Animation (`EditAnimation`)

C'est le panneau de contrôle pour charger les fichiers images (`.png`, `.jpg`).

* **Bouton Load** : Charge l'image en mémoire.
* **Slider Tolerance** : Ajuste la sensibilité de la détection de couleur.
* **Bouton Sauvegarder** : Exporte les données au format `.sba`.

### C. Détection de Sprite Sheet (`SpriteSheetDetection`)

Affiche l'image source.

* **Premier clic** : Définit la couleur de fond (Pipette).
* **Clics suivants** : Déclenchent l'algorithme `DetectBox` pour extraire un sprite.
* **Rendu** : Dessine des rectangles verts autour des zones déjà détectées.

### D. Inspecteur de Frames (`FrameInspector`)

Un tableau détaillé listant tous les sprites extraits avec leur ID, un aperçu miniature, leurs dimensions et leurs coordonnées.

### E. Aperçu & Contrôles (`SettingAnimation`)

Fenêtre de prévisualisation de l'animation finale. L'utilisateur peut régler les FPS et déplacer le personnage sur un fond personnalisé.

---

## 6. Persistance des Données (Format `.sba`)

Le programme utilise un format texte personnalisé pour sauvegarder les animations.

* `ANIMATION:` contient les métadonnées (nom, fps, chemin de l'image, couleur de fond).
* `FRAME:` contient les coordonnées `x, y, w, h` de chaque découpe.
* `END_ANIM` marque la fin du bloc.

Cela permet de recharger une animation complexe sans avoir à recliquer sur chaque sprite manuellement.

---

## 7. Compilation et Dépendances

### Dépendances nécessaires :

* **SDL3** : Gestion fenêtre et rendu.
* **SDL3_image** : Chargement des formats PNG/JPG.
* **Dear ImGui** : Interface graphique (fichiers inclus dans le dossier `imgui/`).

### Commande de compilation (MinGW/GCC) :

```bash
g++ src/*.cpp imgui/*.cpp imgui/backends/imgui_impl_sdl3.cpp imgui/backends/imgui_impl_sdlrenderer3.cpp -o SpriteBox.exe -Iimgui -lSDL3 -lSDL3_image

```

---

## 8. Guide d'Utilisation Rapide

1. **Lancer l'application**.
2. Dans **CREATION D'ANIMATION**, entrez le chemin de votre image (ex: `assets/my_sprites.png`) et cliquez sur **Load**.
3. Ouvrez la fenêtre **SPRITE SHEET** via le menu "Fenetres".
4. **Cliquez une fois** sur la couleur de fond de l'image.
5. **Cliquez sur chaque personnage** de la planche pour les extraire.
6. Réglez les **FPS** dans la fenêtre **CONTROLES** et appuyez sur **PLAY**.
7. Donnez un nom à votre animation et cliquez sur **SAUVEGARDER**.

---

## 9. Gestion de la Mémoire

Le projet suit une approche "C-Style" stricte :

* Chaque surface SDL ou texture créée est libérée manuellement dans la fonction `Clean(AppState* app)`.
* Les tableaux de sprites sont statiques (`MAX_SPRITES`) pour éviter les fragmentations mémoires excessives pendant le développement rapide.
* Utilisation de `malloc/free` pour les calculs lourds de l'algorithme de détection.

---

*Développé dans le cadre du projet SpriteBox - 2026.*

---
