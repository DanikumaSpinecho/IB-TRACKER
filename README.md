# IB Tracker — v0.1

Page HTML autonome pour suivre des clients sur une carte, à usage privé.
Les données vivent dans un fichier **CSV** que vous chargez vous-même ; rien n'est
envoyé à un serveur applicatif.

## Utilisation

Ouvrir `index.html` :

- **en local** — double-clic sur le fichier (`file://`) : tout fonctionne, sauf le
  chargement automatique décrit plus bas ;
- **depuis un serveur web privé** — déposer `index.html` et les CSV dans le même
  dossier.

Puis **Charger** → choisir un `.csv` (ou un `.geojson`). Plusieurs fichiers CSV
différents peuvent être utilisés successivement.

### Chargement automatique (serveur web uniquement)

`index.html?csv=clients-nord.csv` charge le fichier au démarrage sans passer par
le sélecteur. Pratique pour créer un signet par jeu de données. En `file://` la
requête est bloquée par le navigateur : le chargement manuel prend le relais.

## Format du CSV

En-tête attendu (l'ordre des colonnes n'a pas d'importance, les colonnes absentes
sont simplement vides) :

| Colonne | Contenu | Alias acceptés à l'import |
|---|---|---|
| `name` | Nom du client | `titre`, `label` |
| `lat` | Latitude décimale | `latitude` |
| `lng` | Longitude décimale | `lon`, `long`, `longitude` |
| `address` | Adresse postale | `adresse` |
| `phone` | Téléphone du client | `telephone`, `téléphone` |
| `notes` | Notes libres | `description`, `desc` |
| `system` | Système installé | `système`, `categorie`, `category` |
| `nom_fe` | Nom de l'ingénieur de terrain | `fe_name`, `fe_nom` |
| `telephone_fe` | Téléphone du FE | `phone_fe`, `fe_phone` |
| `contact_nom` | Contact principal | `contact_name` |
| `contact_telephone` | Téléphone du contact | `contact_phone` |
| `contact_email` | Email du contact | `contact_mail`, `email` |
| `last_visit` | Dernière visite, `AAAA-MM-JJ` | `derniere_visite`, `dernière_visite` |

Les champs contenant une virgule doivent être entre guillemets :
`"12 rue des Charmilles, 76000 Rouen"`.

`pois.csv` fournit 12 clients factices comme exemple.

## Couleur des marqueurs

Calculée à partir de `last_visit`, par rapport au **1er novembre de l'année
précédente** :

- 🟢 **vert** — visite postérieure à cette date ;
- 🔴 **rouge** — visite antérieure : à replanifier ;
- 🟣 **violet** — aucune date renseignée.

La règle se modifie dans `VISIT_RULE` (constante `refMonth` / `refDay`), les
couleurs dans les variables CSS `--poi-recent`, `--poi-overdue`, `--poi-unknown`.

## Recherche dans la liste

Le champ de recherche accepte du texte libre et des filtres par champ, combinés
avec un ET logique :

```
system:"Systeme A" clinique
contact:dupont
last_visit:2026
```

Champs disponibles : `name`, `system`, `address`, `phone`, `notes`, `last_visit`
(ou `date`), `nom_fe`, `telephone_fe`, `contact` (les trois champs contact),
`contact_nom`, `contact_phone`, `contact_email`.

## Enregistrer les modifications

Les modifications ne sont **pas** conservées d'une session à l'autre : il faut
réenregistrer le CSV.

- **Chrome / Edge, en HTTPS ou en local** — « Enregistrer… » écrit directement
  dans le fichier choisi, et « Activer l'autosave » le réécrit après chaque
  modification.
- **Firefox, Safari iOS, ou serveur en HTTP simple** — ces navigateurs n'ont pas
  l'API d'écriture directe : le bouton devient « Télécharger CSV » et l'autosave
  est masqué. Le fichier téléchargé remplace l'ancien.

« Exporter CSV » et « Exporter GeoJSON » restent disponibles partout.

## Mobile

Sous 900 px, le panneau devient un tiroir (bouton ☰) et les actions d'export sont
regroupées sous le bouton **Partage**. Un appui sur la carte reprend les
coordonnées dans le formulaire.

## Accès réseau

L'application n'a pas de serveur applicatif, mais la page contacte :

- `unpkg.com` — bibliothèque Leaflet ;
- `tile.openstreetmap.org` / `server.arcgisonline.com` — fonds de carte ;
- `nominatim.openstreetmap.org` — uniquement lors d'une recherche d'adresse, et
  seul le texte saisi est transmis.

**Les données clients du CSV ne sont jamais transmises.** Un fond de carte hors
ligne reste possible en déposant des tuiles dans `./france-tiles/{z}/{x}/{y}.png`
(couche « Tuiles locales » du sélecteur).

## Historique

Les versions antérieures (`PoiMap*.html`, `poimap*.html`) sont conservées dans
`archive/` à titre de référence ; elles ne sont plus maintenues.
