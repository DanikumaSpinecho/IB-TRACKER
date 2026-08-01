# IB Tracker — contexte projet

## Ce qu'est ce projet

Page HTML **autonome** de suivi de clients sur une carte, à usage **privé**.
Pas de serveur applicatif, pas de build, pas de framework, pas de dépendances
npm : un seul fichier `index.html` qu'on ouvre au navigateur.

Les données vivent dans un **fichier CSV local** que l'utilisateur charge
lui-même. La page ne fait que l'afficher et l'éditer.

Deux modes d'utilisation prévus :

- ouverture directe du fichier (`file://`) sur le poste ;
- dépôt sur un **serveur web privé**, où plusieurs CSV différents peuvent
  coexister.

Contraintes structurantes, énoncées par l'utilisateur :

- tout doit rester local **sauf le fond de carte** ;
- la page doit être **utilisable au téléphone** (mise en page adaptée) ;
- le CSV contiendra des **coordonnées de clients réels** — jamais de données de
  santé, mais des données personnelles à ne pas divulguer.

Langue de l'interface, des commentaires et des commits : **français**.

## Structure

```
index.html     version de référence (v0.1) — tout est dedans : HTML, CSS, JS
pois.csv       12 clients FACTICES, sert d'exemple et de jeu de test
README.md      documentation utilisateur (format CSV, règles, navigateurs)
.gitignore     protège les CSV réels
archive/       5 versions antérieures, conservées pour référence, NON maintenues
CLAUDE.md      ce fichier
```

Ne pas travailler dans `archive/`. Si une fonctionnalité d'une ancienne version
est à récupérer, la porter dans `index.html`.

## Modèle de données (13 colonnes)

```
name, lat, lng, address, phone, notes, system,
nom_fe, telephone_fe, contact_nom, contact_telephone, contact_email, last_visit
```

`last_visit` au format `AAAA-MM-JJ`. `system` prend `Systeme A/B/C`.
Le parseur d'import accepte des alias (`adresse`, `latitude`, `dernière_visite`,
`email`…) — voir `parseCSV()` et le tableau du README. Les colonnes absentes
donnent des champs vides, jamais d'erreur.

## Règles métier

**Couleur des marqueurs**, calculée depuis `last_visit` par rapport au **1er
novembre de l'année précédente** (`VISIT_RULE`) :
vert = visité après, rouge = visité avant, violet = date absente.

**Recherche** (`quickSearch`) : texte libre + filtres `champ:valeur` combinés en
ET logique, avec surbrillance. Champs reconnus dans `fieldMap`.

## Décisions arrêtées — ne pas revenir dessus sans demander

- **Leaflet reste servi par le CDN** (`unpkg.com`). Les copies locales
  `leaflet.js` / `leaflet.css` ont été supprimées volontairement : le fond de
  carte étant déjà en ligne, une copie locale n'apportait rien.
- **Le CSV se charge manuellement**, à chaque ouverture. C'est voulu : il peut y
  avoir plusieurs CSV différents. Le paramètre optionnel `?csv=fichier.csv`
  permet un chargement automatique quand la page est servie en HTTP.
- **Pas de persistance entre sessions** (ni localStorage ni IndexedDB) —
  écarté sciemment pour l'instant, sujet à rouvrir plus tard.
- **Nominatim et la couche satellite Esri sont conservés** : jugés utiles. Seul
  le texte saisi part vers Nominatim, jamais les données clients.
- Couche « Tuiles locales » (`./france-tiles/{z}/{x}/{y}.png`) laissée en place
  pour un éventuel fond de carte hors ligne.

## Corrections déjà faites — ne pas les réintroduire

1. **Écouteur `popupopen`** : il était enregistré dans `renderMarkers()`, donc
   à chaque rendu, donc à chaque frappe dans la recherche. Les écouteurs
   s'accumulaient et le bouton « Tout afficher » devenait inopérant après un
   nombre pair de rafraîchissements. Il est maintenant enregistré **une seule
   fois dans `initMap()`**, avec un garde `dataset.wired`.
2. **Enregistrement sur mobile** : `showSaveFilePicker` n'existe pas sur Safari
   iOS ni Firefox, ni hors contexte sécurisé. Le bouton affichait une alerte
   d'incompatibilité. La constante `HAS_FS_ACCESS` déclenche désormais un repli
   par téléchargement du CSV, et masque l'autosave.
3. **Surbrillance de recherche** : le remplacement s'appliquait au HTML déjà
   échappé et corrompait les entités. `highlightText()` découpe maintenant le
   **texte brut** puis échappe morceau par morceau.
4. **Mise en page mobile** — deux bugs distincts, tous deux invisibles à la
   lecture du code, trouvés par capture d'écran :
   - l'en-tête en `flex-wrap:nowrap` ne tenait pas dans la largeur et
     **élargissait toute la page** (390 px → 416 px) ; corrigé par
     `min-width:0` sur le `h1` (un enfant flex refuse de rétrécir sans ça) ;
   - `.cm-panel` était en `content-box`, donc `padding` + `border` s'ajoutaient
     à `100vw` : panneau 27 px trop large ; corrigé par `box-sizing:border-box`.
5. **Grilles du panneau** : `grid-template-columns:1fr auto` laissait les inputs
   garder leur largeur intrinsèque et faisait déborder le panneau. Utiliser
   **`minmax(0,1fr)`**, jamais `1fr` nu, pour une colonne contenant un `input`.

## Pièges connus sur cette page

- `100vh` décale la mise en page à cause de la barre d'adresse mobile : la page
  utilise `100dvh` avec repli `100vh`.
- Les champs de saisie sous 16 px déclenchent un **zoom automatique d'iOS** à la
  mise au point : ils passent à `font-size:16px` sous 900 px.
- `--header-h` est recalculée en JS (`updateHeaderHeightVar`) et sert à la
  hauteur de la carte et du tiroir : toute modification de l'en-tête doit la
  laisser juste.
- Le statut (`#saveStatus`) est un bandeau flottant sous 900 px, précisément
  pour ne pas modifier la hauteur de l'en-tête.
- `state.data.indexOf(d)` dans les boucles de rendu est en O(n²) — sans
  conséquence à cette échelle, à revoir si le volume grossit beaucoup.

## Vérifier une modification

Il n'y a pas de suite de tests dans le dépôt. Ce qui a servi jusqu'ici, et qui
reste la bonne méthode : piloter Chromium avec Playwright sur la page servie en
HTTP local, et **regarder les captures d'écran** — plusieurs bugs de mise en
page ne se voient que là.

Points à revérifier après toute modification touchant la mise en page :

- largeurs 430 / 390 / 360 / 320 px : `window.innerWidth` doit rester égal à la
  largeur configurée et `document.documentElement.scrollWidth` ne pas la
  dépasser (c'est ce test qui a révélé le débordement de l'en-tête) ;
- panneau ≤ largeur d'écran, aucun contrôle dont le bord droit dépasse ;
- en-tête sur une seule ligne ;
- chargement du CSV, les 3 couleurs de marqueur, popups, export → réimport ;
- repli de sauvegarde en simulant `delete window.showSaveFilePicker`.

## Conventions de code

- Un seul fichier, pas d'outil de build, pas de dépendance à installer.
- Textes d'interface centralisés dans `UI_TEXT`, couleurs dans les variables CSS
  `--poi-*`, règle de visite dans `VISIT_RULE`, carte dans `CONFIG` : garder ces
  points de réglage groupés et documentés.
- Commentaires en français, et **expliquer le pourquoi** des correctifs
  non évidents (les `min-width:0`, `box-sizing`, `minmax(0,1fr)` portent déjà un
  commentaire : les conserver, ils évitent une régression).
- Échapper systématiquement via `esc()` tout ce qui est injecté en HTML.

## Git

- Branche de travail : `claude/client-tracker-html-yhevns`.
- Le dossier local est synchronisé avec **GitHub Desktop** : les commits partent
  de là, sous l'identité de l'utilisateur.
- **Le `.gitignore` exclut `*.csv`, `*.geojson` et `*.json`**, avec une exception
  explicite pour `pois.csv`. C'est délibéré : un CSV de clients réels ne doit
  jamais être commité. Ne pas assouplir ces règles. Pour suivre volontairement
  un fichier : `git add -f`.

## Pistes ouvertes

- Persistance entre sessions (écartée pour l'instant, à rouvrir).
- Si un CSV de clients réels a déjà été commité par le passé, il faut le retirer
  du suivi — et si le dépôt est public, considérer les données comme exposées.
- L'affichage réel du fond de carte et Nominatim n'ont jamais pu être testés en
  conditions réelles (réseau bloqué dans l'environnement d'origine) : à
  confirmer une fois la page ouverte sur un poste connecté.
