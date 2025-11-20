# Cartographie des variables RNE

Ce document synthétise la structure des JSON RNE (stock et flux), les tables SQLite générées dans `rne.db`, ainsi que les enrichissements apportés aux tables SIRENE. Il s'appuie exclusivement sur le code des modèles Pydantic, des fonctions de mapping et des scripts de création de tables.

---

## 1. Variables d’entrée (JSON RNE)

Les modèles définis dans `rne_model.py` décrivent l’intégralité du schéma JSON fourni par la source RNE. Les tableaux ci‑dessous listent les champs par grande famille avec leur type attendu et leur signification fonctionnelle.

### 1.1 Métadonnées globales et formalités

| Chemin JSON | Type | Description fonctionnelle |
| --- | --- | --- |
| `siren` | string | Identifiant SIREN porté par l’entreprise dans la formalité RNE. |
| `updatedAt`, `createdAt` | datetime ISO | Timestamps de création/mise à jour du dossier dans l’API RNE. |
| `origin` | string | Libellé d’origine de la formalité (non exploité pour l’instant). |
| `formality.siren` | string | SIREN répété dans l’objet de formalité. |
| `formality.companyName` | string | Dénomination déclarée dans la formalité. |
| `formality.typePersonne` | string | Typologie de personne (morale/physique/exploitation). |
| `formality.formeJuridique` | string | Forme juridique saisie par l’INPI. |
| `formality.diffusionINSEE` | string | Indicateur de diffusion INSEE transmis par le RNE. |
| `formality.codeAPE` | string | Code APE porté par la formalité. |
| `formality.content.formeExerciceActivitePrincipale` | string | Modalité de l’activité principale (commerce de détail, profession libérale…). |
| `formality.content.natureCreation.dateCreation` | string (date) | Date de création juridique déclarée. |
| `formality.content.natureCreation.formeJuridique`, `formeJuridiqueInsee` | string | Formes juridiques alternatives (texte libre vs codification INSEE). |
| `formality.content.natureCreation.societeEtrangere`, `entrepriseAgricole` | bool | Indicateurs spécifiques renseignés à la création. |
| `formality.content.natureCessationEntreprise.etatAdministratifInsee` | string | État administratif INSEE (A, F…) porté par la formalité. |

### 1.2 Identité et description de l’entreprise / de l’entrepreneur

| Chemin JSON | Type | Description |
| --- | --- | --- |
| `identite.entreprise.denomination` | string | Raison sociale officielle. |
| `identite.entreprise.formeJuridique`, `formeJuridiqueInsee` | string | Forme juridique textuelle / codifiée. |
| `identite.entreprise.nomCommercial` | string | Nom commercial connu. |
| `identite.entreprise.effectifSalarie` | string | Tranche d’effectif salariale. |
| `identite.entreprise.dateImmat` | string (date) | Date d’immatriculation. |
| `identite.entreprise.codeApe` | string | Code APE déclaré (non réutilisé plus loin). |
| `identite.entreprise.indicateurAssocieUnique` | bool | Indicateur d’associé unique. |
| `identite.entreprise.dateDebutActiv` | string (date) | Date de début d’activité économique. |
| `identite.description.montantCapital` | float | Capital social déclaré. |
| `identite.description.capitalVariable` | bool | Indique si le capital est variable. |
| `identite.description.deviseCapital` | string | Devise du capital social. |
| `identite.description.dateClotureExerciceSocial` | string | Date de clôture d’exercice. |
| `identite.description.duree` | int | Durée statutaire de la personne morale. |
| `identite.entrepreneur.descriptionPersonne.{nom, nomUsage, prenoms[], genre, dateDeNaissance, nationalite, situationMatrimoniale}` | string/list | Identité de l’entrepreneur individuel. |
| `identite.entrepreneur.adresseDomicile.*` | string | Adresse personnelle déclarée (non répliquée dans les tables). |

### 1.3 Composition et dirigeants

| Chemin JSON | Type | Description |
| --- | --- | --- |
| `composition.pouvoirs[].typeDePersonne` | enum string | Type de dirigeant (`INDIVIDU` ou `ENTREPRISE`). |
| `composition.pouvoirs[].roleEntreprise` | string | Code rôle dirigeant (président, gérant…). |
| `composition.pouvoirs[].libelleRoleEntreprise` | string | Libellé texte du rôle (non utilisé). |
| `composition.pouvoirs[].individu.descriptionPersonne.{…}` | objets `DescriptionPersonne` | Identité détaillée du dirigeant personne physique. |
| `composition.pouvoirs[].individu.adresseDomicile.*` | Adresse | Adresse privée du dirigeant. |
| `composition.pouvoirs[].entreprise.{siren, denomination, pays, formeJuridique, autreIdentifiantEtranger, nicSiege, nomCommercial}` | objets `PouvoirEntreprise` | Métadonnées relatives aux dirigeants personnes morales. |
| `composition.pouvoirs[].adresseEntreprise.*` | Adresse | Adresse rattachée au pouvoir (non reprise). |

### 1.4 Adresses de l’entreprise et des établissements

| Chemin JSON | Type | Description |
| --- | --- | --- |
| `adresseEntreprise.adresse.{pays, codePays, commune, codePostal, codeInseeCommune, voie, numVoie, typeVoie, indiceRepetition, complementLocalisation, distributionSpeciale}` | Adresse | Adresse postale du siège social déclaré dans la formalité. |
| `etablissementPrincipal.descriptionEtablissement.{siret, enseigne, nomCommercial}` | string | Identification de l’établissement principal. |
| `etablissementPrincipal.adresse.*` | Adresse | Adresse du siège/établissement principal. |
| `etablissementPrincipal.activites[]` | Activite | Liste d’activités du siège (voir 1.5). |
| `autresEtablissements[].descriptionEtablissement.siret` | string | SIRET des établissements secondaires. |
| `autresEtablissements[].adresse.*` | Adresse | Adresse des établissements secondaires (non extraite actuellement). |

### 1.5 Activités

| Chemin JSON | Type | Description |
| --- | --- | --- |
| `activites[].categoryCode` | string | Famille/catégorie d’activité. |
| `activites[].indicateurPrincipal`, `.indicateurProlongement` | bool | Indicateurs principal / prolongement. |
| `activites[].dateDebut` | date | Date de début de l’activité déclarée. |
| `activites[].formeExercice` | string | Forme d’exercice (commerce ambulant, etc.). |
| `activites[].categorisationActivite1/2/3` | string | Catégorisations fines. |
| `activites[].indicateurActiviteeApe` | bool | Indique si l’activité porte le code APE. |
| `activites[].codeApe` | string | Code APE associé à l’activité. |
| `activites[].activiteRattacheeEirl` | bool | Spécificité EIRL. |

### 1.6 Création, cessation et état administratif

| Chemin JSON | Type | Description |
| --- | --- | --- |
| `detailCessationEntreprise.{dateCessationTotaleActivite, dateEffet, dateRadiation}` | date | Dates de cessation utilisées pour renseigner la radiation. |
| `natureCessationEntreprise.etatAdministratifInsee` | string | État administratif transmis à l’INSEE. |
| `content.natureCreation.{societeEtrangere, entrepriseAgricole}` | bool | Indicateurs contextuels lors de la création. |

---

## 2. Variables et tables de sortie

### 2.1 Tables RNE (`rne.db`)

Les tables créées dans `process_rne.py` capturent l’ensemble des informations mappées via `map_rne_company_to_ul`. Les colonnes et leur rôle sont listés ci‑dessous.

#### `unite_legale`

| Colonne | Type SQLite | Description |
| --- | --- | --- |
| `siren` | TEXT | Identifiant unique. |
| `denomination`, `nom`, `nom_usage`, `prenom` | TEXT | Identité de la personne morale ou physique. |
| `nom_commercial` | TEXT | Nom commercial principal. |
| `date_creation` | TEXT | Date de création (format ISO). |
| `date_mise_a_jour` | DATE | Dernière mise à jour transmise par RNE. |
| `activite_principale` | TEXT | (Non alimenté actuellement). |
| `tranche_effectif_salarie` | TEXT | Tranche d’effectif. |
| `nature_juridique` | TEXT | Forme juridique calculée. |
| `etat_administratif` | TEXT | Statut administratif RNE/INSEE. |
| `forme_exercice_activite_principale` | TEXT | Forme d’exercice principale. |
| `statut_diffusion` | TEXT | Indicateur de diffusion. |
| `adresse` | TEXT | Adresse normalisée (concaténation). |
| `file_name` | TEXT | Chemin du fichier JSON source. |

#### `siege`

| Colonne | Type | Description |
| --- | --- | --- |
| `siren`, `siret` | TEXT | Identifiants UL et établissement. |
| `enseigne`, `nom_commercial` | TEXT | Signalétique du siège. |
| `pays`, `code_pays`, `commune`, `code_postal`, `code_commune`, `voie`, `num_voie`, `type_voie`, `indice_repetition`, `complement_localisation`, `distribution_speciale` | TEXT | Détails de l’adresse du siège. |
| `date_mise_a_jour` | DATE | Date d’actualisation RNE. |
| `file_name` | TEXT | Provenance du JSON. |

#### `dirigeant_pp`

| Colonne | Type | Description |
| --- | --- | --- |
| `siren` | TEXT | SIREN rattaché. |
| `date_mise_a_jour` | DATE | Date de mise à jour du dossier. |
| `nom`, `nom_usage`, `prenoms`, `genre` | TEXT | Identité du dirigeant personne physique. |
| `date_de_naissance` | TEXT | Date de naissance (format chaîne). |
| `role` | TEXT | Code rôle issu du JSON. |
| `nationalite` | TEXT | Nationalité déclarée. |
| `situation_matrimoniale` | TEXT | Information matrimoniale. |
| `file_name` | TEXT | Fichier source. |

#### `dirigeant_pm`

| Colonne | Type | Description |
| --- | --- | --- |
| `siren` | TEXT | SIREN de l’UL. |
| `siren_dirigeant` | TEXT | SIREN de la personne morale dirigeante. |
| `date_mise_a_jour` | DATE | Date d’actualisation. |
| `denomination` | TEXT | Nom de la personne morale dirigeante. |
| `role` | TEXT | Code rôle. |
| `pays` | TEXT | Pays associé. |
| `forme_juridique` | TEXT | Forme juridique de la société dirigeante. |
| `file_name` | TEXT | Fichier source. |

#### `immatriculation`

| Colonne | Type | Description |
| --- | --- | --- |
| `siren` | TEXT | Identifiant UL. |
| `date_mise_a_jour` | DATE | Date de mise à jour RNE. |
| `date_immatriculation`, `date_radiation`, `date_debut_activite` | DATE/TEXT | Jalons d’immatriculation et de radiation. |
| `indicateur_associe_unique` | TEXT | Valeur booléenne sérialisée. |
| `capital_social` | REAL | Montant du capital. |
| `date_cloture_exercice` | TEXT | Date de clôture (texte). |
| `duree_personne_morale` | INT | Durée statutaire. |
| `nature_entreprise` | TEXT | Liste JSON sérialisée issue des formes d’exercice. |
| `capital_variable`, `devise_capital` | TEXT | Informations capitalistiques. |
| `file_name` | TEXT | Provenance. |

#### `etablissement`

| Colonne | Type | Description |
| --- | --- | --- |
| `siren`, `siret` | TEXT | Identifiants UL/SIRET. |
| `date_mise_a_jour` | DATE | Date d’actualisation RNE. |
| `file_name` | TEXT | Provenance. |

#### `activite`

| Colonne | Type | Description |
| --- | --- | --- |
| `siren`, `siret` | TEXT | Identifiants du siège ou de l’établissement porteur. |
| `code_category`, `code_ape` | TEXT | Codifications d’activité. |
| `indicateur_principal`, `indicateur_prolongement`, `indicateur_activitee_ape`, `activite_rattachee_eirl` | BOOLEAN | Indicateurs booléens copiés depuis l’API. |
| `date_debut` | DATE | Date de début d’activité. |
| `form_exercice` | TEXT | Forme d’exercice. |
| `categorisation_activite1/2/3` | TEXT | Catégories fines. |
| `date_mise_a_jour` | DATE | Date du JSON. |
| `file_name` | TEXT | Provenance. |

### 2.2 Tables SIRENE enrichies par la donnée RNE

| Table SIRENE | Champs alimentés | Mode d’alimentation |
| --- | --- | --- |
| `unite_legale` | `from_rne`, `date_mise_a_jour_rne`, et toutes les colonnes renseignées lors de l’insertion `INSERT OR IGNORE` (date de création, noms, natures juridiques, tranche d’effectif, etc.) | `add_rne_siren_data_to_unite_legale_table` attache `rne.db`, met à jour les drapeaux puis insère les lignes manquantes depuis `db_rne.unite_legale`. |
| `siege` | `date_mise_a_jour_rne` (pour les lignes existantes) et un bloc de colonnes d’adresse (numéro/type de voie, commune, code postal, etc.) lors de l’insertion des nouveaux sièges. | `add_rne_data_to_siege_table` applique `update_siege_table_fields_with_rne_data_query` puis `insert_remaining_rne_siege_data_into_main_table_query`. |
| `dirigeant_pp` | Colonnes identitaires consolidées + `role_description`. | Les tables SIRENE sont regénérées à partir de `rne.db` en chunks et nettoyées via `preprocess_personne_physique`. Les noms/prénoms sont mis en majuscules, les doublons supprimés et les codes rôles convertis en libellés.  |
| `dirigeant_pm` | `siren`, `siren_dirigeant`, `denomination`, `role_description`. | Même logique que ci-dessus avec `preprocess_dirigeant_pm` (suppression des `NaN`, upper-case, agrégation des rôles). |
| `immatriculation` | Table recopiée intégralement (structure + contenu). | `copy_immatriculation_table` crée la table dans la base SIRENE puis copie toutes les lignes distinctes de `db_rne.immatriculation`. |

---

## 3. Tableau de mapping source → cible

Les tableaux suivants détaillent la correspondance entre les chemins JSON et les colonnes finales (tables `rne.db` qui alimentent ensuite les tables SIRENE).

### 3.1 `unite_legale`

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

### 3.2 `siege`, `etablissement` et `activite`

| Source JSON | Table cible | Champ cible | Type cible | Transformation / logique | Commentaires |
| --- | --- | --- | --- | --- | --- |
| `etablissementPrincipal.descriptionEtablissement.siret` | `siege` | `siret` | TEXT | Copie directe. | Sert aussi de clé pour les activités de siège. |
| `etablissementPrincipal.descriptionEtablissement.{nomCommercial, enseigne}` | `siege` | `nom_commercial`, `enseigne` | TEXT | Copie directe. | |
| `etablissementPrincipal.adresse.{...}` | `siege` | Colonnes d’adresse | TEXT | Mapping champ à champ via `map_rne_siege_to_ul`. | Toutes les composantes sont injectées (pays, code INSEE, etc.). |
| `etablissementPrincipal.activites[]` | `activite` | Colonnes `code_category`, `indicateur_*`, `date_debut`, `form_exercice`, `categorisation_*`, `code_ape`, `activite_rattachee_eirl` | TEXT/BOOL/DATE | Chaque activité devient une ligne `activite` avec `siret = siret_du_siege`. | La date de mise à jour provient d’`updatedAt`. |
| `autresEtablissements[].descriptionEtablissement.siret` | `etablissement` | `siret` | TEXT | Copie directe. | Les activités associées héritent du même SIRET. |
| `autresEtablissements[].activites[]` | `activite` | mêmes colonnes | TEXT/BOOL/DATE | Même logique que pour le siège. | |
| `activites[].formeExercice` + indicateur principal | `immatriculation` | `nature_entreprise` | TEXT | `get_nature_entreprise_list` construit un set (forme principale + activités principales des établissements) puis il est sérialisé en JSON. | Champ multi-source, peut être nul si aucun indicateur principal n’est renseigné. |

### 3.3 Dirigeants

| Source JSON | Table cible | Champ cible | Type cible | Transformation / logique | Commentaires |
| --- | --- | --- | --- | --- | --- |
| `composition.pouvoirs[].typeDePersonne == "INDIVIDU"` | `dirigeant_pp` | colonnes identitaires | TEXT | `map_rne_dirigeant_pp_to_ul` recopie les champs `nom`, `nomUsage`, `prenoms`, `genre`, `dateDeNaissance`, `nationalite`, `situationMatrimoniale`. Les listes de prénoms sont jointes avec un espace. | Le rôle stocké est `roleEntreprise`. |
| `composition.pouvoirs[].typeDePersonne == "ENTREPRISE"` | `dirigeant_pm` | `siren_dirigeant`, `denomination`, `role`, `pays`, `forme_juridique` | TEXT | `map_rne_dirigeant_pm_to_ul` nettoie le SIREN (suppression des espaces) et recopie les autres champs. | |
| `personnePhysique.identite.entrepreneur.descriptionPersonne` (cas personne physique sans composition) | `dirigeant_pp` | mêmes colonnes | TEXT | `get_dirigeants` renvoie une liste avec l’entrepreneur individuel. | Les champs rôle restent `None`. |
| `dirigeants` (tous types) | `unite_legale` | `nom`, `nom_usage`, `prenom` | TEXT | Pour les personnes physiques, la première ligne de dirigeant est recopiée sur l’UL afin de disposer d’un triplet nom/prénom. | Permet de distinguer les entreprises individuelles dans la base SIRENE. |
| `updatedAt` + `file_path` | `dirigeant_pp`/`dirigeant_pm` | `date_mise_a_jour`, `file_name` | DATE/TEXT | Ajoutés lors de l’insertion dans SQLite. | Utilisés ensuite pour le nettoyage et la traçabilité. |

### 3.4 Immatriculation

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

### 3.5 Mise à jour SIRENE et traitements complémentaires

| Étape | Source JSON / table RNE | Table SIRENE cible | Transformation / logique |
| --- | --- | --- | --- |
| Mise à jour `unite_legale` | `db_rne.unite_legale` | `unite_legale` | `UPDATE` ajoute `from_rne=TRUE` et `date_mise_a_jour_rne`. `INSERT OR IGNORE` copie toutes les colonnes pertinentes pour les SIREN absents. |
| Mise à jour `siege` | `db_rne.siege` | `siege` | `UPDATE` met à jour `date_mise_a_jour_rne`, `INSERT` ajoute les sièges manquants avec les colonnes d’adresse RNE. |
| Recréation `dirigeant_pp` / `dirigeant_pm` | `db_rne.dirigeant_*` | `dirigeant_*` | Les tables RNE sont lues par chunks, normalisées (`uppercase`, groupby, concaténation des rôles) puis insérées dans la base SIRENE. |
| Copie `immatriculation` | `db_rne.immatriculation` | `immatriculation` | Duplication stricte (structure + contenu) en une seule requête SQL. |

---

## 4. Champs JSON non utilisés / ignorés

Les champs suivants sont définis dans les modèles Pydantic mais ne sont jamais référencés dans les fonctions de mapping ni dans l’insertion SQLite. Ils constituent des candidats pour de futurs enrichissements.

| Chemin JSON | Type | Utilisé ? | Commentaire |
| --- | --- | --- | --- |
| `origin` | string | non | Jamais lu lors du mapping. |
| `formality.companyName` | string | non | Non recopié dans `unite_legale`. |
| `formality.typePersonne` | string | non | Seule la présence de `personneMorale/Physique/Exploitation` est testée, pas ce champ. |
| `formality.codeAPE` | string | non | Aucun mapping vers les tables de sortie. |
| `content.natureCreation.formeJuridiqueInsee` | string | non | La forme INSEE utilisée est celle de `identite.entreprise`, pas celle-ci. |
| `content.natureCreation.societeEtrangere`, `entrepriseAgricole` | bool | non | Indicateurs de création ignorés. |
| `identite.entreprise.codeApe` | string | non | Non recopié (le code APE provient des activités). |
| `identite.entrepreneur.adresseDomicile.*` | Adresse | non | Non exporté vers les tables dirigeant. |
| `composition.pouvoirs[].libelleRoleEntreprise` | string | non | Seul `roleEntreprise` est utilisé pour les rôles. |
| `composition.pouvoirs[].adresseEntreprise.*` | Adresse | non | Jamais exploité. |
| `composition.pouvoirs[].entreprise.autreIdentifiantEtranger`, `nicSiege`, `nomCommercial` | string | non | Les dirigeants personnes morales ne stockent que siren, dénomination, pays, forme juridique. |
| `AdresseEntreprise.{caracteristiques, entrepriseDomiciliataire}` | dict | non | Ignorés lors du mapping d’adresse. |
| `DescriptionPersonne.titre`, `DescriptionPersonne.role` | string | non | Les titres/ rôles textuels ne sont pas recopiés. |



---

## Résumé

* Les tables principales générées sont `unite_legale`, `siege`, `dirigeant_pp`, `dirigeant_pm`, `immatriculation`, `etablissement` et `activite`. Elles rassemblent respectivement l’identité légale, les sièges, les dirigeants (PP/PM), les métadonnées d’immatriculation, la liste des établissements et leurs activités, avec traçabilité via `file_name` et `date_mise_a_jour`.
* Le pipeline exploite principalement les blocs JSON `identite` (dénomination, capital, dates), `adresseEntreprise` et `etablissementPrincipal/autresEtablissements` (adresses + activités), `composition.pouvoirs` (dirigeants) et `detailCessationEntreprise`. Les autres informations (type de formalité, indicateurs de création, code APE de la formalité) restent disponibles pour de futurs usages.
* Documenter ces mappings facilite le debugging (on sait d’où provient chaque colonne), la maintenance (ajout d’un champ = repérer la fonction associée) et l’évolution du pipeline (identifier immédiatement les champs JSON inutilisés qui pourraient enrichir les tables SIRENE).
