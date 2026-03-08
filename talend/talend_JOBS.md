# ⚙️ Documentation Détaillée des Jobs Talend
## Projet SI Décisionnels — Entrepôt de Données RH

**Projet** : Entrepôt de données pour l’analyse RH  
**Auteurs** : Hamza Bouguerra · Fares Messedi  
**Niveau / Groupe** : ESPRIT — 1BA2 — Systèmes d’Information Décisionnels  
**Année universitaire** : 2025–2026  
**Date limite** : 14 mars 2026  

**Outil ETL** : Talend Open Studio for Data Integration  

## Sources de données
- PostgreSQL **rh_entreprise**
- `absences_presences.csv`
- `formations.xlsx`

## Base cible
PostgreSQL **rh_dw**

---

# 📋 Table des matières

1. Architecture globale
2. Analyse des sources
3. Ordre d’exécution des jobs
4. Description détaillée des jobs Talend
5. Modèle de données cible
6. Résumé quantitatif
7. Gestion des erreurs
8. Structure du projet Talend

---

# 🏗️ Architecture générale

```
        ┌─────────────────────────────┐
        │        SOURCES DATA         │
        │                             │
        │ PostgreSQL rh_entreprise    │
        │   • employes                │
        │   • salaires                │
        │                             │
        │ absences_presences.csv      │
        │ formations.xlsx             │
        └─────────────┬───────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │        STAGING TALEND       │
        │                             │
        │ Nettoyage données           │
        │ Normalisation texte         │
        │ Suppression doublons        │
        │ Correction valeurs NULL     │
        └─────────────┬───────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │      DATA WAREHOUSE         │
        │         rh_dw               │
        │                             │
        │ Dimensions :                │
        │ dim_employe                 │
        │ dim_service                 │
        │ dim_temps                   │
        │ dim_absence                 │
        │ dim_motif_absence           │
        │ dim_formation               │
        │                             │
        │ Table de faits :            │
        │ fait_rh                     │
        └─────────────────────────────┘
```

---

# 📁 Analyse des sources de données

### employes_salaires.sql
| Indicateur | Valeur |
|---|---|
| Table employes | **112** (dont 12 doublons) |
| Table salaires | **1200** (100 emp × 12 mois) |
| NULL poste | **14** |
| NULL date_embauche | **7** |
| NULL ville | **9** |
| Salaire min/max/moy | 815 DT / 4445 DT / 2592 DT |
| Primes possibles | 0 · 100 · 150 · 200 · 300 DT |
| Services distincts | 10 |

---

## Fichier `absences_presences.csv`

| Indicateur | Valeur |
|---|---|
| Total lignes | **582** |
| Doublons exacts | **123** (21.1%) |
| NULL motif | **54** (9.3%) |
| NULL duree_jours | **50** (8.6%) |
| NULL justifie | **53** (9.1%) |
| Colonne remarque | **582 vides** → à ignorer |
| Matricules distincts | 86 employés concernés |
| Plage de dates | 2022-01-02 → 2024-12-27 |
| Motifs existants | Accident de travail · Congé annuel · Congé sans solde · Formation · Maladie · Maternité/Paternité |

---

## Fichier `formations.xlsx`

| Indicateur | Valeur |
|---|---|
| Total lignes | **266** |
| Doublons exacts | **54** (20.3%) |
| NULL Coût (DT) | **31** (11.7%) |
| NULL Statut | **30** (11.3%) |
| NULL Durée (h) | **24** (9.0%) |
| Formations distinctes | 16 intitulés |
| Années couvertes | 2022 · 2023 · 2024 |
| Coût min/max/moy | 500 / 2000 / 1176 DT |
| Durées possibles (h) | 8 · 16 · 20 · 24 · 32 · 40 |
| Statuts | Complétée (77) · En cours (89) · Planifiée (70) · NULL (30) |


---

# ⚡ Ordre d’exécution des jobs

```
J0_InitialLoad
      ↓
J1_Load_DIM_EMPLOYE ──┐
J2_Load_DIM_TEMPS   ──┘
      ↓
J3_Load_DIM_ABSENCE
J4_Load_DIM_FORMATION
      ↓
J5_Load_FAIT_RH
```

Le job **J5 doit être exécuté en dernier**.

---

# 🧩 Description des Jobs Talend

## J0 — Job_InitialLoad

### Objectif
Initialiser la base `rh_dw` et créer les tables du Data Warehouse.

### Composants Talend
- tPostgresqlConnection
- tFileCheck
- tPostgresqlRow
- tLogRow
- tDie

---

## J1 — Job_Load_DIM_EMPLOYE

### Source
PostgreSQL `rh_entreprise`  
table `employes`

### Objectif
Créer :
- dim_employe
- dim_service

### Nettoyage

| Colonne | Correction |
|---|---|
| poste | "Non renseigné" |
| ville | "Non renseignée" |
| email | "non-renseigne@entreprise.tn" |
| telephone | "Non renseigné" |

### Résultat

| Table | Lignes |
|---|---|
| dim_employe | 100 |
| dim_service | 10 |

---

## J2 — Job_Load_DIM_TEMPS

### Objectif
Créer une dimension calendrier couvrant **2022 → 2024**.

### Attributs générés
- date_complete
- jour
- mois
- année
- trimestre
- semaine
- nom_jour
- nom_mois
- weekend

### Résultat
**1096 lignes**

---

## J3 — Job_Load_DIM_ABSENCE

### Source
`absences_presences.csv`

### Nettoyage

| Problème | Correction |
|---|---|
| doublons | supprimés |
| motif NULL | "Non précisé" |
| duree_jours NULL | 0 |
| justifie NULL | "Non renseigné" |
| remarque | ignorée |

### Résultat

| Table | Lignes |
|---|---|
| dim_absence | 520 |
| dim_motif_absence | 7 |

---

## J4 — Job_Load_DIM_FORMATION

### Source
`formations.xlsx`

### Nettoyage

| Problème | Correction |
|---|---|
| doublons | supprimés |
| coût NULL | 0 |
| statut NULL | "Inconnu" |
| durée NULL | 0 |

### Résultat
**238 lignes dans dim_formation**

---

## J5 — Job_Load_FAIT_RH

### Objectif
Créer la table de faits RH.

### Sources
- dim_employe
- dim_service
- dim_absence
- dim_formation
- salaires

### Mesures calculées

| Mesure | Calcul |
|---|---|
| salaire_mensuel | moyenne salaires |
| salaire_annuel | salaire_mensuel × 12 |
| nb_jours_absence | somme duree_jours |
| nb_formations | count formations |

### Résultat
**100 lignes (1 par employé)**

---

# 🗄️ Modèle de données cible

Dimensions :
- dim_employe
- dim_service
- dim_temps
- dim_absence
- dim_motif_absence
- dim_formation

Table de faits :
- fait_rh

---

# 📊 Résumé quantitatif

| Job | Source | Entrée | Sortie |
|---|---|---|---|
| J1 | employes | 112 | 100 |
| J2 | calendrier | — | 1096 |
| J3 | CSV absences | 582 | 520 |
| J4 | Excel formations | 266 | 238 |
| J5 | dimensions | 100 | 100 |
