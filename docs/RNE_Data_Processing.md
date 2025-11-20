
# RNE_Data_Processing

## 1. Résumé exécutif
Le pipeline RNE industrialise :
- la collecte du stock initial INPI et des flux différentiels quotidiens
- les valide via Pydantic,
- les normalise
- et les consolide dans une base SQLite versionnée (`rne_<date>.db.gz`) stockée sur MinIO

Cette base est ensuite utilisée dans l’ETL SIRENE pour enrichir les tables dirigeants, unités légales, sièges et immatriculations. 

Des garde-fous — dédoublonnage, upsert par SIREN, contrôles volumétriques, exclusion du dernier fichier de flux, notifications Mattermost — garantissent la cohérence des données.

Le traitement complet :
- Airflow : orchestre toutes les tâches du pipeline (télécharger, charger, nettoyer, publier).
- MinIO : sert de stockage d’objets pour les fichiers bruts et les bases compressées.
- SQLite : base de données locale utilisée pour structurer et versionner les données RNE.
- Pydantic : valide chaque JSON RNE pour garantir qu’il respecte le schéma attendu.
- pandas : sert à nettoyer et dédupliquer les données, notamment les dirigeants.
- Client API : le code qui appelle l’API RNE est conçu pour être solide :
    - pagination searchAfter : permet de récupérer les résultats par blocs ordonnés sur le SIREN suivant.
    - retries : retente automatiquement en cas d’erreur de l’API.
    - régulation dynamique du pageSize : si l’API renvoie une erreur (ex. 500/memory), le client réduit la taille des pages et retente.

## 2. Vue d’ensemble du pipeline
- Acquisition du stock via FTP → MinIO
- Collecte quotidienne API RNE → MinIO
- Construction / reprise de la base SQLite : ingestion du stock et des flux, mapping, nettoyage, versioning
- Mise à disposition dans MinIO
- Intégration des données RNE dans la base SIRENE via des opérations SQL + pandas

## 3. Architecture des composants
- Airflow : orchestration `get_flux_rne` et `fill_rne_database`
- MinIO : stockage des données brutes, bases versionnées et métadonnées
- SQLite : base normalisée `rne_<date>.db`
- Pydantic : validation des payloads JSON
- Pandas : nettoyage et consolidation dirigeants
- API RNE : récupération flux quotidiens via `companies/diff`

## 4. Flux de données détaillé
### Stock initial
- Téléchargement du ZIP via `get_stock.sh`
- Extraction JSON → MinIO `rne/stock/`
- Suppression locale

### Flux quotidiens
- Appels API RNE (pagination `searchAfter`, retries)
- Construction JSON ND → `.gz`
- Upload MinIO `rne/flux/`
- Sauvegarde même en cas d’erreur partielle

### Construction SQLite
- Récupération `latest_rne_date.json`
- Reprise ou création `rne_<date>.db`
- Ingestion stock si reconstruction
- Ingestion flux (ordre ascendant, dernier fichier ignoré)
- Validation Pydantic
- Upsert SIREN
- Dédoublonnage + VACUUM
- Vérifications volumétriques
- Compression + upload MinIO + mise à jour métadonnées

### Intégration SIRENE
- Attachement de `rne.db`
- Prétraitements dirigeants
- UPDATE + INSERT OR IGNORE
- Copie stricte de la table immatriculation
- Reconstruction tables dérivées

## 5. Schéma fonctionnel du pipeline
```mermaid
flowchart TD
    A[FTP INPI] --> B[MinIO rne/stock/]
    A2[RNE API companies/diff] --> C[MinIO rne/flux/]
    B --> D[fill_rne_database]
    C --> D
    D --> E[SQLite rne_<date>.db]
    E --> F[MinIO rne/database/]
    F --> G[ETL SIRENE]
    G --> H[Tables SIRENE enrichies]
```

## 6. Inputs & Outputs
### Inputs
- Stock historique INPI ZIP → JSON
- Flux différentiel quotidien `.json.gz`
- Métadonnées : `latest_rne_date.json`
- Base précédente : `rne_<date>.db.gz`

### Outputs
- Base SQLite versionnée
- Tables normalisées (UL, siège, dirigeants, immatriculation, établissements, activités)
- Tables SIRENE enrichies
- Notifications Mattermost

## 7. Mapping Source → Cible

## 7. Tableau de mapping source → cible

Les tableaux suivants détaillent la correspondance entre les chemins JSON et les colonnes finales (tables `rne.db` qui alimentent ensuite les tables SIRENE).

### 7.1 `unite_legale`

| Source JSON | Table cible | Champ cible | Type cible | Transformation / logique | Commentaires |
| --- | --- | --- | --- | --- | --- |
| `siren` | `unite_legale` | `siren` | TEXT | Affectation directe par `map_rne_company_to_ul`. | Sert aussi de clé primaire pour les suppressions de doublons avant insertion. |
| `formality.diffusionINSEE` | `unite_legale` | `statut_diffusion` | TEXT | Copie directe. | Propagé plus tard dans SIRENE (`statut_diffusion_unite_legale`). |
| `formality.formeJuridique`, `content.natureCreation.formeJuridique`, `identite.entreprise.formeJuridique/formeJuridiqueInsee` | `unite_legale` | `nature_juridique` | TEXT | `get_forme_juridique` priorise la forme issue de la formalité, sinon nature de création, sinon identite. | Combine plusieurs sources pour fiabiliser la forme juridique. |
| `createdAt` ou `content.natureCreation.dateCreation` | `unite_legale` | `date_creation` | TEXT | `get_date_creation` choisit `createdAt` puis la date de création déclarée. | Format conservé tel que fourni. |
| `formality.content.formeExerciceActivitePrincipale` | `unite_legale` | `forme_exercice_activite_principale` | TEXT | Copie directe. | Sert aussi de première valeur pour la nature d’entreprise (voir immatriculation). |
| `formality.content.natureCessationEntreprise.etatAdministratifInsee` | `unite_legale` | `etat_administratif` | TEXT | Copie directe. | |
| `identite.entreprise.denomination` | `unite_legale` | `denomination` | TEXT | Copie directe. | |
| `identite.entreprise.nomCommercial` | `unite_legale` | `nom_commercial` | TEXT | Copie directe. | |
| `identite.entreprise.effectifSalarie` | `unite_legale` | `tranche_effectif_salarie` | TEXT | Copie directe. | |
| `identite.entreprise.dateImmat` | `immatriculation` | `date_immatriculation` | DATE | Copie directe. | |
| `identite.entreprise.indicateurAssocieUnique` | `immatriculation` | `indicateur_associe_unique` | TEXT | Valeur bool sérialisée en texte par SQLite. | |
| `identite.entreprise.dateDebutActiv` | `immatriculation` | `date_debut_activite` | TEXT | Copie directe. | |
| `identite.description.{montantCapital, capitalVariable, deviseCapital, dateClotureExerciceSocial, duree}` | `immatriculation` | `capital_social`, `capital_variable`, `devise_capital`, `date_cloture_exercice`, `duree_personne_morale` | REAL/TEXT/INT | Copie directe. | |
| `detailCessationEntreprise.{dateRadiation, dateEffet, dateCessationTotaleActivite}` | `immatriculation` | `date_radiation` | DATE | Première date non nulle (radiation > effet > cessation). | Valeur priorisée via `get_detail_cessation`. |
| `updatedAt` | Toutes tables | `date_mise_a_jour` | DATE | Affecté à `unite_legale.date_mise_a_jour` et réutilisé lors de l’insertion des enfants (`siege`, `dirigeant_*`, `etablissement`, `activite`, `immatriculation`). | Permet un suivi temporel homogène. |
| `adresseEntreprise.adresse.*` | `unite_legale` | `adresse` | TEXT | Les champs structurés sont convertis en objet `Adresse` puis concaténés via `format_address()`. | Prépare l’export SIRENE et ElasticSearch. |
| `composition` ou `identite.entrepreneur` | `dirigeant_pp` / `dirigeant_pm` | voir §3.3 | cf. ci-dessous | cf. ci-dessous | cf. ci-dessous |
| `etablissementPrincipal` + `autresEtablissements` | `siege`, `etablissement`, `activite` | voir §3.2 | cf. ci-dessous | cf. ci-dessous | cf. ci-dessous |

### 7.2 `siege`, `etablissement` et `activite`

| Source JSON | Table cible | Champ cible | Type cible | Transformation / logique | Commentaires |
| --- | --- | --- | --- | --- | --- |
| `etablissementPrincipal.descriptionEtablissement.siret` | `siege` | `siret` | TEXT | Copie directe. | Sert aussi de clé pour les activités de siège. |
| `etablissementPrincipal.descriptionEtablissement.{nomCommercial, enseigne}` | `siege` | `nom_commercial`, `enseigne` | TEXT | Copie directe. | |
| `etablissementPrincipal.adresse.{...}` | `siege` | Colonnes d’adresse | TEXT | Mapping champ à champ via `map_rne_siege_to_ul`. | Toutes les composantes sont injectées (pays, code INSEE, etc.). |
| `etablissementPrincipal.activites[]` | `activite` | Colonnes `code_category`, `indicateur_*`, `date_debut`, `form_exercice`, `categorisation_*`, `code_ape`, `activite_rattachee_eirl` | TEXT/BOOL/DATE | Chaque activité devient une ligne `activite` avec `siret = siret_du_siege`. | La date de mise à jour provient d’`updatedAt`. |
| `autresEtablissements[].descriptionEtablissement.siret` | `etablissement` | `siret` | TEXT | Copie directe. | Les activités associées héritent du même SIRET. |
| `autresEtablissements[].activites[]` | `activite` | mêmes colonnes | TEXT/BOOL/DATE | Même logique que pour le siège. | |
| `activites[].formeExercice` + indicateur principal | `immatriculation` | `nature_entreprise` | TEXT | `get_nature_entreprise_list` construit un set (forme principale + activités principales des établissements) puis il est sérialisé en JSON. | Champ multi-source, peut être nul si aucun indicateur principal n’est renseigné. |

### 7.3 Dirigeants

| Source JSON | Table cible | Champ cible | Type cible | Transformation / logique | Commentaires |
| --- | --- | --- | --- | --- | --- |
| `composition.pouvoirs[].typeDePersonne == "INDIVIDU"` | `dirigeant_pp` | colonnes identitaires | TEXT | `map_rne_dirigeant_pp_to_ul` recopie les champs `nom`, `nomUsage`, `prenoms`, `genre`, `dateDeNaissance`, `nationalite`, `situationMatrimoniale`. Les listes de prénoms sont jointes avec un espace. | Le rôle stocké est `roleEntreprise`. |
| `composition.pouvoirs[].typeDePersonne == "ENTREPRISE"` | `dirigeant_pm` | `siren_dirigeant`, `denomination`, `role`, `pays`, `forme_juridique` | TEXT | `map_rne_dirigeant_pm_to_ul` nettoie le SIREN (suppression des espaces) et recopie les autres champs. | |
| `personnePhysique.identite.entrepreneur.descriptionPersonne` (cas personne physique sans composition) | `dirigeant_pp` | mêmes colonnes | TEXT | `get_dirigeants` renvoie une liste avec l’entrepreneur individuel. | Les champs rôle restent `None`. |
| `dirigeants` (tous types) | `unite_legale` | `nom`, `nom_usage`, `prenom` | TEXT | Pour les personnes physiques, la première ligne de dirigeant est recopiée sur l’UL afin de disposer d’un triplet nom/prénom. | Permet de distinguer les entreprises individuelles dans la base SIRENE. |
| `updatedAt` + `file_path` | `dirigeant_pp`/`dirigeant_pm` | `date_mise_a_jour`, `file_name` | DATE/TEXT | Ajoutés lors de l’insertion dans SQLite. | Utilisés ensuite pour le nettoyage et la traçabilité. |

### 7.4 Immatriculation

| Source JSON | Table cible | Champ cible | Type cible | Transformation / logique | Commentaires |
| --- | --- | --- | --- | --- | --- |
| `identite.entreprise.dateImmat` | `immatriculation` | `date_immatriculation` | DATE | Copie directe. | |
| `detailCessationEntreprise.{dates}` | `immatriculation` | `date_radiation` | DATE | Sélection de la première date disponible (priorité radiations). | |
| `identite.entreprise.indicateurAssocieUnique` | `immatriculation` | `indicateur_associe_unique` | TEXT | Valeur bool convertie en texte lors de l’insertion. | |
| `identite.description.montantCapital` | `immatriculation` | `capital_social` | REAL | Copie directe. | |
| `identite.description.dateClotureExerciceSocial` | `immatriculation` | `date_cloture_exercice` | TEXT | Copie directe. | |
| `identite.description.capitalVariable` | `immatriculation` | `capital_variable` | TEXT | Valeur booléenne sérialisée. | |
| `identite.description.deviseCapital` | `immatriculation` | `devise_capital` | TEXT | Copie directe. | |
| `identite.description.duree` | `immatriculation` | `duree_personne_morale` | INT | Copie directe. | |
| `identite.entreprise.dateDebutActiv` | `immatriculation` | `date_debut_activite` | TEXT | Copie directe. | |
| `nature_entreprise` (set issu des formes d’exercice) | `immatriculation` | `nature_entreprise` | TEXT | Sérialisation JSON (`json.dumps`) de la liste construite par `get_nature_entreprise_list`. | Multi-sources (activité principale, sièges, établissements). |

### 7.5 Mise à jour SIRENE et traitements complémentaires

| Étape | Source JSON / table RNE | Table SIRENE cible | Transformation / logique |
| --- | --- | --- | --- |
| Mise à jour `unite_legale` | `db_rne.unite_legale` | `unite_legale` | `UPDATE` ajoute `from_rne=TRUE` et `date_mise_a_jour_rne`. `INSERT OR IGNORE` copie toutes les colonnes pertinentes pour les SIREN absents. |
| Mise à jour `siege` | `db_rne.siege` | `siege` | `UPDATE` met à jour `date_mise_a_jour_rne`, `INSERT` ajoute les sièges manquants avec les colonnes d’adresse RNE. |
| Recréation `dirigeant_pp` / `dirigeant_pm` | `db_rne.dirigeant_*` | `dirigeant_*` | Les tables RNE sont lues par chunks, normalisées (`uppercase`, groupby, concaténation des rôles) puis insérées dans la base SIRENE. |
| Copie `immatriculation` | `db_rne.immatriculation` | `immatriculation` | Duplication stricte (structure + contenu) en une seule requête SQL. |

---
## 8. Artefacts générés
- `stock_rne.json` (sur MinIO)
- `rne_flux_YYYY-MM-DD.json.gz`
- `rne_<date>.db` local + version `.gz` sur MinIO
- `latest_rne_date.json`
- Tables SIRENE enrichies

## 9. Réutilisabilité du pipeline
*Section à compléter manuellement.*

## 10. Limites techniques / risques
*Section vide comme demandé.*

