# Modèle de données final (SQLite RNE + SIRENE)

Ce document inventorie les tables finales produites par les pipelines RNE et SIRENE, puis détaille pour chaque table les champs cibles et leurs correspondances (source, type, transformations).

### Champs avec double provenance (RNE **et** SIRENE)

- **`unite_legale`** : `nom`, `nom_usage`, `prenom` (personnes physiques) proviennent des identités RNE et des colonnes d'identité SIRENE.
- **`siege`** : `siren`, `siret`, les noms/enseignes et l'ensemble des colonnes d'adresse sont remplis par les sièges RNE et par les enregistrements SIRENE filtrés avec `estSiege = 1`.
- **`etablissement`** : `siren`, `siret` (établissements secondaires) sont alimentés par les blocs `autresEtablissements` RNE et par les lignes établissement SIRENE.

### Mécanique de fusion et règles de priorité

1. **Chargement initial SIRENE** : les stocks/flux CSV créent les tables `unite_legale`, `siege` et `etablissement` avec un indicateur `from_insee = TRUE`.
2. **Attache de la base RNE** : la base SQLite RNE est attachée à celle de SIRENE ; les tâches d'ETL exécutent des `INSERT OR IGNORE`/upserts depuis RNE vers les tables cibles.
3. **Priorité implicite RNE** : lorsqu'une donnée RNE existe, elle remplace ou complète la valeur SIRENE sur les champs communs ; les lignes uniquement présentes dans RNE sont ajoutées.
4. **Traçabilité** : les colonnes `from_rne` et `date_mise_a_jour_rne` sont mises à jour lors de cette phase pour signaler les enregistrements enrichis ou rafraîchis par RNE.
5. **Unicité** : cette stratégie garantit une seule ligne par SIREN/SIRET dans les tables finales, sans doublons simultanés de sources différentes.

#### Pourquoi une base intermédiaire `db_rne` ?

- **Normalisation en amont** : les JSON RNE sont d'abord convertis en une base SQLite autonome (`rne.db`). Certaines tables sont nettoyées (uppercases, regroupement des rôles, dédoublonnage) avant toute fusion.
- **Attache maîtrisée** : cette base est ensuite montée en base secondaire (`ATTACH DATABASE ... AS db_rne`) sur la base SIRENE pour exécuter les `INSERT OR IGNORE`/`UPDATE`. Cela évite de charger l'intégralité des flux RNE en mémoire et limite les écritures aux seules jointures SQL nécessaires.
- **Reproductibilité** : disposer d'un artefact SQLite intermédiaire permet de relancer la fusion ou de rejouer des étapes spécifiques sans retraiter les JSON bruts.

## Liste des tables finales

- `unite_legale`
- `siege`
- `etablissement`
- `activite`
- `dirigeant_pp`
- `dirigeant_pm`
- `immatriculation`
- `flux_unite_legale`
- `historique_unite_legale`
- `ancien_siege`
- `flux_etablissement`
- Tables dérivées établissement (`count_etablissement`, `count_etablissement_ouvert`)

---

## Table `unite_legale`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `siren` | Copie directe du SIREN de la formalité. | `siren` | TEXT |
| RNE | JSON | `identite.entreprise.denomination` | Copie directe quand l'identité entreprise est disponible. | `denomination` | TEXT |
| RNE | JSON | `dirigeants[0].descriptionPersonne.{nom, nomUsage, prenoms}` | Pour une personne physique, reprend l'identité du premier dirigeant détecté. | `nom`, `nom_usage`, `prenom` | TEXT |
| RNE | JSON | `identite.entreprise.nomCommercial` | Copie directe. | `nom_commercial` | TEXT |
| RNE | JSON | `createdAt` ou `formality.content.natureCreation.dateCreation` | Choix prioritaire : timestamp de création, sinon date déclarée dans la formalité. | `date_creation` | TEXT |
| RNE | JSON | `updatedAt` | Copie directe de la date de mise à jour. | `date_mise_a_jour` | DATE |
| RNE | JSON | `formality.content.formeExerciceActivitePrincipale` ou activité principale détectée | Ajout du code de forme d'exercice principale ; alimente aussi `nature_entreprise` dans `immatriculation`. | `forme_exercice_activite_principale` | TEXT |
| RNE | JSON | `formality.content.natureCessationEntreprise.etatAdministratifInsee` | Copie directe de l'état administratif. | `etat_administratif` | TEXT |
| RNE | JSON | `formality.formeJuridique` puis `natureCreation.formeJuridique` puis `identite.entreprise.{formeJuridique, formeJuridiqueInsee}` | Sélection hiérarchique de la meilleure forme juridique disponible. | `nature_juridique` | TEXT |
| RNE | JSON | `identite.entreprise.effectifSalarie` | Copie directe de la tranche d'effectif. | `tranche_effectif_salarie` | TEXT |
| RNE | JSON | Adresse siège `adresseEntreprise.adresse.*` | Mapping champ à champ vers une adresse structurée puis concaténée. | `adresse` | TEXT |
| RNE | JSON | `formality.diffusionINSEE` | Copie directe. | `statut_diffusion` | TEXT |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
| SIRENE | CSV | `dateCreationUniteLegale` | Copie directe. | `date_creation_unite_legale` | DATE |
| SIRENE | CSV | `sigleUniteLegale` | Copie directe. | `sigle` | TEXT |
| SIRENE | CSV | `prenom1UniteLegale` | Copie directe pour la personne physique. | `prenom` | TEXT |
| SIRENE | CSV | `identifiantAssociationUniteLegale` | Copie directe. | `identifiant_association_unite_legale` | TEXT |
| SIRENE | CSV | `trancheEffectifsUniteLegale` | Copie directe. | `tranche_effectif_salarie_unite_legale` | TEXT |
| SIRENE | CSV | `categorieEntreprise` | Copie directe. | `categorie_entreprise` | TEXT |
| SIRENE | CSV | `etatAdministratifUniteLegale` | Copie directe. | `etat_administratif_unite_legale` | TEXT |
| SIRENE | CSV | `nomUniteLegale` | Copie directe. | `nom` | TEXT |
| SIRENE | CSV | `nomUsageUniteLegale` | Copie directe. | `nom_usage` | TEXT |
| SIRENE | CSV | `denominationUniteLegale` | Copie directe. | `nom_raison_sociale` | TEXT |
| SIRENE | CSV | `denominationUsuelle{1,2,3}UniteLegale` | Copie directe. | `denomination_usuelle_1`, `denomination_usuelle_2`, `denomination_usuelle_3` | TEXT |
| SIRENE | CSV | `categorieJuridiqueUniteLegale` | Copie directe. | `nature_juridique_unite_legale` | TEXT |
| SIRENE | CSV | `activitePrincipaleUniteLegale` | Copie directe. | `activite_principale_unite_legale` | TEXT |
| SIRENE | CSV | `economieSocialeSolidaireUniteLegale` | Copie directe. | `economie_sociale_solidaire_unite_legale` | TEXT |
| SIRENE | CSV | `statutDiffusionUniteLegale` | Copie directe. | `statut_diffusion_unite_legale` | TEXT |
| SIRENE | CSV | `estSocieteMissionUniteLegale` | Copie directe. | `est_societe_mission` | TEXT |
| SIRENE | CSV | `anneeCategorieEntreprise` | Copie directe. | `annee_categorie_entreprise` | TEXT |
| SIRENE | CSV | `anneeTrancheEffectifsUniteLegale` | Copie directe. | `annee_tranche_effectif_salarie` | TEXT |
| SIRENE | CSV | `caractereEmployeurUniteLegale` | Copie directe. | `caractere_employeur` | TEXT |
| Enrichissement interne | N/A | Indicateur de provenance INSEE | Champ forcé à TRUE lors du chargement CSV. | `from_insee` | BOOLEAN |
| Enrichissement RNE | N/A | Rattachement au RNE lors de l'`ATTACH` | Mise à TRUE et mise à jour de `date_mise_a_jour_rne`. | `from_rne`, `date_mise_a_jour_rne` | BOOLEAN, DATE |
| SIRENE/RNE | N/A | Dates de diffusion | Copie directe des champs Insee et RNE respectifs. | `date_mise_a_jour_insee`, `date_mise_a_jour_rne` | DATE |
| SIRENE (flux CSV) | CSV | `dateFermetureUniteLegale` | Copie directe. | `date_fermeture_unite_legale` | DATE |

---

## Table `siege`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `siren` | Copie directe. | `siren` | TEXT |
| RNE | JSON | `etablissementPrincipal.descriptionEtablissement.siret` | Copie directe. | `siret` | TEXT |
| RNE | JSON | `etablissementPrincipal.descriptionEtablissement.{enseigne, nomCommercial}` | Copie directe. | `enseigne`, `nom_commercial` | TEXT |
| RNE | JSON | `etablissementPrincipal.adresse.{pays, codePays, commune, codePostal, codeInseeCommune, voie, numVoie, typeVoie, indiceRepetition, complementLocalisation, distributionSpeciale}` | Mapping champ à champ vers l'adresse du siège. | Champs adresse | TEXT |
| RNE | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
| SIRENE | CSV | `siren`, `siret` | Copie directe des identifiants. | `siren`, `siret` | TEXT |
| SIRENE | CSV | Champs adresse (`numeroVoieEtablissement`, `typeVoieEtablissement`, `libelleVoieEtablissement`, `codePostalEtablissement`, `libelleCommuneEtablissement`, `communeEtablissement`, `complementAdresseEtablissement`, `cedexEtablissement`, `libelleCedexEtablissement`, `distributionSpecialeEtablissement`, `indiceRepetitionEtablissement`) | Copie champ à champ pour l'adresse postale. | Colonnes adresse correspondantes | TEXT |
| SIRENE | CSV | `estSiege` | Filtre `estSiege = 1` pour isoler le siège. | `est_siege` | TEXT |
| SIRENE | CSV | `dateDebut` | Copie directe. | `date_debut_activite` | DATE |
| SIRENE | CSV | `etatAdministratifEtablissement` | Copie directe. | `etat_administratif_etablissement` | TEXT |
| SIRENE | CSV | `nomCommercialEtablissement`, `enseigne{1,2,3}Etablissement` | Copie directe. | `nom_commercial`, `enseigne_1/2/3` | TEXT |
| SIRENE | CSV | Champs étranger (`libelleCommuneEtrangerEtablissement`, `codePaysEtrangerEtablissement`, `libellePaysEtrangerEtablissement`) | Copie directe. | `libelle_commune_etranger`, `code_pays_etranger`, `libelle_pays_etranger` | TEXT |
| Data.gouv géoloc | CSV | Champs géocodage BAN (`x`, `y`, `latitude`, `longitude`, `geo_adresse`, `geo_id`, `geo_score`) | Copie directe depuis le stock géolocalisé. | Colonnes géo | TEXT |
| Enrichissement RNE | N/A | Date de mise à jour RNE | Mise à jour lors de l'attache RNE (et `date_fermeture_etablissement`). | `date_mise_a_jour_rne`, `date_fermeture_etablissement` | DATE |
| SIRENE (flux CSV) | CSV | Identiques aux colonnes ci-dessus | Chargement des flux pour réécriture/merge dans `siege`. | Colonnes miroir | Types identiques |

---

## Table `etablissement`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `siren` | Copie directe. | `siren` | TEXT |
| RNE | JSON | `autresEtablissements[].descriptionEtablissement.siret` | Copie directe. | `siret` | TEXT |
| RNE | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
| SIRENE | CSV | `siren`, `siret` | Copie directe des identifiants. | `siren`, `siret` | TEXT |
| SIRENE | CSV | `dateCreationEtablissement` | Copie directe. | `date_creation` | DATE |
| SIRENE | CSV | `trancheEffectifsEtablissement` | Copie directe. | `tranche_effectif_salarie` | TEXT |
| SIRENE | CSV | `caractereEmployeurEtablissement` | Copie directe. | `caractere_employeur` | TEXT |
| SIRENE | CSV | `anneeTrancheEffectifsEtablissement` | Copie directe. | `annee_tranche_effectif_salarie` | TEXT |
| SIRENE | CSV | `activitePrincipaleRegistreMetiersEtablissement` | Copie directe. | `activite_principale_registre_metier` | TEXT |
| SIRENE | CSV | `estSiege` | Copie directe. | `est_siege` | TEXT |
| SIRENE | CSV | Champs adresse (`numeroVoieEtablissement`, `typeVoieEtablissement`, `libelleVoieEtablissement`, `codePostalEtablissement`, `libelleCommuneEtablissement`, `communeEtablissement`, `complementAdresseEtablissement`, `cedexEtablissement`, `libelleCedexEtablissement`, `distributionSpecialeEtablissement`, `indiceRepetitionEtablissement`) | Copie champ à champ pour l'adresse postale. | Colonnes adresse correspondantes | TEXT |
| SIRENE | CSV | `dateDebut` | Copie directe. | `date_debut_activite` | DATE |
| SIRENE | CSV | `etatAdministratifEtablissement` | Copie directe. | `etat_administratif_etablissement` | TEXT |
| SIRENE | CSV | `enseigne{1,2,3}Etablissement` | Copie directe. | `enseigne_1`, `enseigne_2`, `enseigne_3` | TEXT |
| SIRENE | CSV | `activitePrincipaleEtablissement` | Copie directe. | `activite_principale` | TEXT |
| SIRENE | CSV | `nomCommercialEtablissement` | Copie directe. | `nom_commercial` | TEXT |
| SIRENE | CSV | Champs étranger (`libelleCommuneEtrangerEtablissement`, `codePaysEtrangerEtablissement`, `libellePaysEtrangerEtablissement`) | Copie directe. | `libelle_commune_etranger`, `code_pays_etranger`, `libelle_pays_etranger` | TEXT |
| Data.gouv géoloc | CSV | Champs géocodage BAN (`x`, `y`, `latitude`, `longitude`, `geo_adresse`, `geo_id`, `geo_score`) | Copie directe depuis le stock géolocalisé. | Colonnes géo | TEXT |
| Enrichissement RNE | N/A | Date de mise à jour RNE | Mise à jour lors de l'attache RNE (et `date_fermeture_etablissement`). | `date_mise_a_jour_rne`, `date_fermeture_etablissement` | DATE |
| SIRENE (flux CSV) | CSV | Identiques aux colonnes ci-dessus | Chargement des flux pour réécriture/merge dans `etablissement`. | Colonnes miroir | Types identiques |

---

## Table `activite`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `siren` | Repris du SIREN UL courant. | `siren` | TEXT |
| RNE | JSON | `activites[].siret` ou SIRET d'établissement | Copie du SIRET porteur de l'activité. | `siret` | TEXT |
| RNE | JSON | `activites[].categoryCode` | Copie directe. | `code_category` | TEXT |
| RNE | JSON | `activites[].indicateurPrincipal` | Copie directe. | `indicateur_principal` | BOOLEAN |
| RNE | JSON | `activites[].indicateurProlongement` | Copie directe. | `indicateur_prolongement` | BOOLEAN |
| RNE | JSON | `activites[].dateDebut` | Copie directe. | `date_debut` | DATE |
| RNE | JSON | `activites[].formeExercice` | Copie directe. | `form_exercice` | TEXT |
| RNE | JSON | `activites[].categorisationActivite{1,2,3}` | Copie directe. | `categorisation_activite1/2/3` | TEXT |
| RNE | JSON | `activites[].indicateurActiviteeApe` | Copie directe. | `indicateur_activitee_ape` | BOOLEAN |
| RNE | JSON | `activites[].codeApe` | Copie directe. | `code_ape` | TEXT |
| RNE | JSON | `activites[].activiteRattacheeEirl` | Copie directe. | `activite_rattachee_eirl` | BOOLEAN |
| RNE | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

---

## Table `dirigeant_pp`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `composition.pouvoirs[].individu.descriptionPersonne.siren` (hérité) | Copie du SIREN UL associé. | `siren` | TEXT |
| RNE | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| RNE | JSON | `descriptionPersonne.{nom, nomUsage, prenoms, genre, dateDeNaissance, nationalite, situationMatrimoniale}` | Copie directe, concaténation des prénoms si liste. | Identité (`nom`, `nom_usage`, `prenoms`, `genre`, `date_de_naissance`, `nationalite`, `situation_matrimoniale`) | TEXT |
| RNE | JSON | `composition.pouvoirs[].roleEntreprise` | Copie directe du code rôle. | `role` | TEXT |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
| RNE (sqlite attaché) | SQL | `db_rne.dirigeant_pp` | Copie par chunks, normalisation (uppercase, dédoublonnage) via `preprocess_personne_physique`. | `siren`, `date_mise_a_jour`, `date_de_naissance`, `role`, `nom`, `nom_usage`, `prenoms`, `nationalite`, `role_description` | TEXT/DATE |

---

## Table `dirigeant_pm`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `siren` | Copie du SIREN UL associé. | `siren` | TEXT |
| RNE | JSON | `composition.pouvoirs[].entreprise.siren` | Nettoyage des espaces avant stockage. | `siren_dirigeant` | TEXT |
| RNE | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| RNE | JSON | `composition.pouvoirs[].entreprise.denomination` | Copie directe. | `denomination` | TEXT |
| RNE | JSON | `composition.pouvoirs[].entreprise.roleEntreprise` | Copie directe du rôle. | `role` | TEXT |
| RNE | JSON | `composition.pouvoirs[].entreprise.pays` | Copie directe. | `pays` | TEXT |
| RNE | JSON | `composition.pouvoirs[].entreprise.formeJuridique` | Copie directe. | `forme_juridique` | TEXT |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
| RNE (sqlite attaché) | SQL | `db_rne.dirigeant_pm` | Copie par chunks + `preprocess_dirigeant_pm` (nettoyage, regroupement des rôles). | `siren`, `date_mise_a_jour`, `denomination`, `siren_dirigeant`, `role`, `forme_juridique`, `role_description` | TEXT/DATE |

---

## Table `immatriculation`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| RNE | JSON | `siren` | Copie directe. | `siren` | TEXT |
| RNE | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| RNE | JSON | `identite.entreprise.dateImmat` | Copie directe. | `date_immatriculation` | DATE |
| RNE | JSON | `detailCessationEntreprise.{dateRadiation,dateEffet,dateCessationTotaleActivite}` | Sélection de la première date disponible dans l'ordre radiation → effet → cessation totale. | `date_radiation` | DATE |
| RNE | JSON | `identite.entreprise.indicateurAssocieUnique` | Copie directe. | `indicateur_associe_unique` | TEXT |
| RNE | JSON | `description.montantCapital` | Copie directe. | `capital_social` | REAL |
| RNE | JSON | `description.dateClotureExerciceSocial` | Copie directe. | `date_cloture_exercice` | TEXT |
| RNE | JSON | `description.duree` | Copie directe. | `duree_personne_morale` | INT |
| RNE | JSON | `nature_entreprise` dérivée | Ensemble des formes d'exercice détectées (siège + établissements) sérialisées en liste. | `nature_entreprise` | TEXT |
| RNE | JSON | `identite.entreprise.dateDebutActiv` | Copie directe. | `date_debut_activite` | TEXT |
| RNE | JSON | `description.capitalVariable` | Copie directe. | `capital_variable` | TEXT |
| RNE | JSON | `description.deviseCapital` | Copie directe. | `devise_capital` | TEXT |
| RNE | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
| RNE (sqlite attaché) | SQL | `db_rne.immatriculation` | Copie intégrale avec `INSERT DISTINCT` et index sur `siren`. | Colonnes identiques à la table source RNE | Types identiques |

---

## Table `flux_unite_legale`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| SIRENE (flux CSV) | CSV | Champs flux miroir des colonnes UL | Chargement direct des fichiers flux pour préparer le `REPLACE` vers `unite_legale`. | Colonnes `unite_legale` | Types identiques |

---

## Tables `historique_unite_legale` et `ancien_siege`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| SIRENE (historique CSV) | CSV | `dateDebut` / `dateFin` / état administratif | Copie directe des périodes d'historique UL. | `date_debut_periode`, `date_fin_periode`, `etat_administratif_unite_legale` | DATE, TEXT |
| SIRENE (historique CSV) | CSV | `nicSiege` | Copie directe. | `nic_siege` | TEXT |
| SIRENE (flux historique) | CSV | NIC de siège précédent | Alimentation de la table tampon pour repérer les anciens sièges. | `siren`, `nic_siege`, `siret` | TEXT |

---

## Table `flux_etablissement`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| SIRENE (flux CSV) | CSV | Identiques aux colonnes d'`etablissement`/`siege` | Chargement des flux pour réécriture/merge dans les tables finales. | Colonnes miroir | Types identiques |

---

## Tables dérivées établissement

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| SIRENE (après chargement) | SQL | `etablissement` | Agrégation du nombre total d'établissements par SIREN. | `count_etablissement` | INTEGER |
| SIRENE (après chargement) | SQL | `etablissement` filtré `etat_administratif_etablissement='A'` | Comptage des établissements actifs. | `count_etablissement_ouvert` | INTEGER |

