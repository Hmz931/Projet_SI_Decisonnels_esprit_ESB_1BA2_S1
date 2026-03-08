# ⚙️ Documentation Technique Complète des Jobs Talend

## Projet SI Décisionnels --- Data Warehouse RH

**Projet :** Entrepôt de données pour l'analyse RH\
**Auteurs :** Hamza Bouguerra · Fares Messedi\
**École :** ESPRIT\
**Module :** Systèmes d'Information Décisionnels\
**Année :** 2025--2026

**Outil ETL :** Talend Open Studio for Data Integration\
**SGBD :** PostgreSQL

------------------------------------------------------------------------

# 1. Architecture globale du système

Le pipeline ETL suit une architecture classique en trois couches :

1.  **Sources opérationnelles**
2.  **Zone de transformation (Talend)**
3.  **Data Warehouse décisionnel**

```{=html}
<!-- -->
```
    Sources → Nettoyage → Transformation → Chargement DW

Sources utilisées :

-   Base PostgreSQL `rh_entreprise`
-   Fichier CSV `absences_presences.csv`
-   Fichier Excel `formations.xlsx`

Base cible :

    rh_dw

------------------------------------------------------------------------

# 2. Modèle en étoile du Data Warehouse

## Dimensions

-   dim_employe
-   dim_service
-   dim_temps
-   dim_absence
-   dim_motif_absence
-   dim_formation

## Table de faits

    fait_rh

Cette table permet d'analyser :

-   salaires
-   absences
-   formations
-   indicateurs RH

------------------------------------------------------------------------

# 3. Ordre d'exécution des Jobs Talend

    J0_InitialLoad
          ↓
    J1_Load_DIM_EMPLOYE
    J2_Load_DIM_TEMPS
          ↓
    J3_Load_DIM_ABSENCE
    J4_Load_DIM_FORMATION
          ↓
    J5_Load_FAIT_RH

Le job **J5 doit être exécuté en dernier** car il dépend des dimensions.

------------------------------------------------------------------------

# 4. Job J0 --- Initialisation du Data Warehouse

## Objectif

Créer les tables du Data Warehouse dans PostgreSQL.

## Composants Talend

    tPostgresqlConnection
            ↓
    tPostgresqlRow
            ↓
    tLogRow

## Script SQL exécuté

``` sql
CREATE TABLE dim_service (
id_service SERIAL PRIMARY KEY,
nom_service VARCHAR(100)
);

CREATE TABLE dim_employe (
id_employe SERIAL PRIMARY KEY,
matricule INT,
nom VARCHAR(100),
poste VARCHAR(100),
ville VARCHAR(100),
email VARCHAR(150),
telephone VARCHAR(30),
id_service INT
);

CREATE TABLE dim_temps (
id_temps SERIAL PRIMARY KEY,
date_complete DATE,
jour INT,
mois INT,
annee INT,
trimestre INT,
semaine INT,
nom_jour VARCHAR(20),
nom_mois VARCHAR(20),
weekend BOOLEAN
);

CREATE TABLE dim_motif_absence (
id_motif SERIAL PRIMARY KEY,
motif VARCHAR(100)
);

CREATE TABLE dim_absence (
id_absence SERIAL PRIMARY KEY,
matricule INT,
date_absence DATE,
duree_jours INT,
justifie VARCHAR(20),
id_motif INT
);

CREATE TABLE dim_formation (
id_formation SERIAL PRIMARY KEY,
matricule INT,
nom_formation VARCHAR(200),
duree_heures INT,
cout DECIMAL,
statut VARCHAR(50)
);

CREATE TABLE fait_rh (
id_fait SERIAL PRIMARY KEY,
id_employe INT,
id_service INT,
salaire_mensuel NUMERIC,
salaire_annuel NUMERIC,
nb_jours_absence INT,
nb_formations INT
);
```

------------------------------------------------------------------------

# 5. Job J1 --- Chargement dimension employés

## Source

PostgreSQL :

    rh_entreprise.employes

## Flux Talend

    tPostgresqlInput
            ↓
    tMap (nettoyage)
            ↓
    tUniqRow (suppression doublons)
            ↓
    tPostgresqlOutput (dim_employe)

## Requête SQL source

``` sql
SELECT *
FROM employes;
```

## Nettoyage des données (tMap)

### Gestion des NULL

    poste → "Non renseigné"
    ville → "Non renseignée"
    email → "non-renseigne@entreprise.tn"
    telephone → "Non renseigné"

Expression Talend :

    row1.poste == null ? "Non renseigné" : row1.poste

### Suppression des doublons

Composant :

    tUniqRow
    clé = matricule

## Chargement de dim_service

Requête utilisée :

``` sql
INSERT INTO dim_service(nom_service)
SELECT DISTINCT service
FROM employes;
```

------------------------------------------------------------------------

# 6. Job J2 --- Création de la dimension temps

## Objectif

Créer un calendrier complet entre **2022 et 2024**.

## Composants Talend

    tRowGenerator
            ↓
    tMap
            ↓
    tPostgresqlOutput

## Colonnes générées

-   date_complete
-   jour
-   mois
-   annee
-   trimestre
-   semaine
-   nom_jour
-   nom_mois
-   weekend

### Expression weekend

    TalendDate.getPartOfDate("DAY_OF_WEEK",row1.date)==1
    || TalendDate.getPartOfDate("DAY_OF_WEEK",row1.date)==7

Nombre total de lignes générées :

    1096

------------------------------------------------------------------------

# 7. Job J3 --- Chargement des absences

## Source

    absences_presences.csv

## Composants

    tFileInputDelimited
            ↓
    tMap
            ↓
    tUniqRow
            ↓
    tPostgresqlOutput

## Nettoyage

  Problème        Solution
  --------------- -----------------
  doublons        tUniqRow
  motif NULL      "Non précisé"
  duree NULL      0
  justifie NULL   "Non renseigné"

Expression Talend :

    row1.motif == null ? "Non précisé" : row1.motif

### Création dimension motif

    SELECT DISTINCT motif
    FROM absences_presences;

------------------------------------------------------------------------

# 8. Job J4 --- Chargement des formations

## Source

    formations.xlsx

## Composants

    tFileInputExcel
            ↓
    tMap
            ↓
    tUniqRow
            ↓
    tPostgresqlOutput

## Nettoyage

  Problème      Correction
  ------------- ------------
  coût NULL     0
  statut NULL   "Inconnu"
  durée NULL    0

Expression Talend :

    row1.cout == null ? 0 : row1.cout

------------------------------------------------------------------------

# 9. Job J5 --- Chargement de la table de faits

## Sources

-   dim_employe
-   dim_service
-   dim_absence
-   dim_formation
-   salaires

## Flux Talend

    tPostgresqlInput (employes)
            ↓
    tMap (jointures)
            ↓
    tAggregateRow
            ↓
    tPostgresqlOutput (fait_rh)

## Calculs

### Salaire mensuel

    AVG(salaire)

### Salaire annuel

    salaire_mensuel * 12

### Nombre de jours d'absence

    SUM(duree_jours)

### Nombre de formations

    COUNT(id_formation)

------------------------------------------------------------------------

# 10. Gestion des erreurs

Talend utilise :

    tLogRow
    tDie
    Reject Flow

Exemples :

-   fichier introuvable
-   erreur de type
-   clé étrangère inexistante

------------------------------------------------------------------------

# 11. Structure du projet Talend

    RH_DW_PROJECT
    │
    ├── Job Designs
    │   ├── J0_InitialLoad
    │   ├── J1_Load_DIM_EMPLOYE
    │   ├── J2_Load_DIM_TEMPS
    │   ├── J3_Load_DIM_ABSENCE
    │   ├── J4_Load_DIM_FORMATION
    │   └── J5_Load_FAIT_RH
    │
    ├── Metadata
    │   ├── PostgreSQL connections
    │   ├── CSV files
    │   └── Excel files

------------------------------------------------------------------------

# 12. Résumé des volumes

  Job   Source       Entrée   Sortie
  ----- ------------ -------- --------
  J1    employes     112      100
  J2    calendrier   \-       1096
  J3    absences     582      520
  J4    formations   266      238
  J5    dimensions   100      100

------------------------------------------------------------------------

# Conclusion

Ce pipeline ETL permet :

-   l'intégration de données multi-sources
-   le nettoyage automatique
-   la création d'un Data Warehouse RH
-   l'analyse décisionnelle (absences, formations, salaires).
