# 🗄️ Entrepôt de Données RH - Projet SI Décisionnels

<div align="center">

![Talend](https://img.shields.io/badge/Talend-Open%20Studio-FF6D00?style=flat-square)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=flat-square&logo=postgresql&logoColor=white)
![PowerBI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?style=flat-square&logo=powerbi&logoColor=black)
![Excel](https://img.shields.io/badge/Excel-XLSX-217346?style=flat-square&logo=microsoftexcel&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-cyan?style=flat-square)
![ESPRIT](https://img.shields.io/badge/ESPRIT-1BA2%20·%202025--2026-8b5cf6?style=flat-square)

**Projet académique - Systèmes d'Information Décisionnels**  
Enseignant : **Sofien Boutaib** · Deadline : **14 Mars 2026**

[🌐 Demo interactive](https://hmz931.github.io/Projet_SI_Decisonnels_esprit_ESB_1BA2_S1/) · [📊 Rapport Power BI](powerbi/) · [⚙️ Jobs Talend](talend/JOBS.md)

</div>

---

## 👥 Auteurs

| | Nom | Rôle |
|---|---|---|
| 👨‍💻 | **Hamza Bouguerra** | ETL · Modélisation · Dashboard · Power BI · Documentation |
| 👨‍💻 | **Fares Messedi** | ETL · Modélisation · Dashboard · Power BI · Documentation |

---

## 📋 Contexte

Une entreprise souhaite **centraliser les données RH de ses 100 employés** afin d'améliorer la gestion des ressources humaines. Les données proviennent de **3 sources hétérogènes** (CSV, SQL, Excel), contiennent des anomalies intentionnelles (~10% doublons, ~10% nulls) et doivent être intégrées dans un **entrepôt de données modélisé en schéma étoile** via un pipeline ETL complet sur **Talend Open Studio**, chargé dans **PostgreSQL 16**.

---

## 📁 Sources de données réelles

| Fichier | Type | Lignes brutes | Doublons | NULLs détectés |
|---|---|---|---|---|
| [`absences_presences.csv`](data/raw/absences_presences.csv) | CSV | **582** | **62** lignes | motif: 54 · durée: 50 · justifié: 53 |
| [`employes_salaires.sql`](data/raw/employes_salaires.sql) | SQL | **112** inserts | **12** (casse nom) | poste: 14 · ville: 9 · date: 7 |
| [`formations.xlsx`](data/raw/formations.xlsx) | Excel | **266** | **28** lignes | coût: 31 · statut: 30 · durée: 24 |

> ⚠️ Les anomalies sont **intentionnelles** pour simuler un cas réel et justifier les étapes de nettoyage ETL dans Talend (`tUniqRow`, `tReplaceList`, `tMap`).

---

## ⭐ Modélisation - Schéma en Étoile

```
                    ┌─────────────────┐
                    │   DIM_TEMPS     │
                    │ sk_temps (PK)   │
                    │ date · mois     │
                    │ trimestre · ... │
                    └────────┬────────┘
                             │
┌──────────────┐    ┌────────▼────────┐    ┌──────────────────┐
│ DIM_EMPLOYE  │    │    FAIT_RH      │    │   DIM_SERVICE    │
│ sk_employe   ├────┤ sk_employe (FK) ├────┤ sk_service (PK)  │
│ matricule    │    │ sk_service (FK) │    │ nom_service      │
│ nom · prénom │    │ sk_temps   (FK) │    └──────────────────┘
│ poste · ville│    │ sk_absence (FK) │
└──────────────┘    │ sk_formation(FK)│    ┌──────────────────┐
                    │ sk_motif   (FK) │    │  DIM_FORMATION   │
                    ├─────────────────┤────┤ sk_formation(PK) │
                    │ salaire_mensuel │    │ intitule · durée │
                    │ salaire_annuel  │    │ cout · statut    │
                    │ nb_jours_abs    │    └──────────────────┘
                    │ nb_formations   │
                    │ prime           │    ┌──────────────────┐
                    └────────┬────────┘    │  DIM_ABSENCE     │
                             │             │ sk_absence (PK)  │
                    ┌────────▼────────┐    │ motif · durée    │
                    │DIM_MOTIF_ABSENCE│    │ justifié         │
                    │ sk_motif (PK)   │    └──────────────────┘
                    │ motif · categorie│
                    └─────────────────┘
```

---

## ⚙️ Pipeline ETL - Jobs Talend

| Job | Source → Cible | Entrée | Sortie | Anomalies traitées |
|---|---|---|---|---|
| `J0_InitialLoad` | - → PostgreSQL `rh_dw` | - | Schéma créé | DDL + vérification sources |
| `J1_Load_DIM_EMPLOYE` | SQL → DIM_EMPLOYE + DIM_SERVICE | 112 | **100 + 10** | 12 doublons · 30 NULLs |
| `J2_Load_DIM_TEMPS` | Générée → DIM_TEMPS | - | **1 096** | Calendrier 2022→2024 |
| `J3_Load_DIM_ABSENCE` | CSV → DIM_ABSENCE + DIM_MOTIF | 582 | **520 + 7** | 62 doublons · 157 NULLs |
| `J4_Load_DIM_FORMATION` | Excel → DIM_FORMATION | 266 | **238** | 28 doublons · 85 NULLs |
| `J5_Load_FAIT_RH` | DIM/* → FAIT_RH | 100 emp | **100** | Agrégations + calculs |

📄 **Documentation complète des jobs** → [`talend/JOBS.md`](talend/JOBS.md)

---

## 📊 Rapport Power BI

Le rapport décisionnel connecté à la base `rh_dw` (PostgreSQL) contient 4 pages :

- **Vue Générale RH** - KPIs, masse salariale, répartition par service
- **Analyse des Absences** - motifs, jours, top employés
- **Analyse des Formations** - coûts, statuts, évolution par année
- **Profil Employé** - fiche individuelle (drill-through)

📁 Fichier : [`powerbi/rapport_RH.pbix`](powerbi/rapport_RH.pbix)

---

## 🗂️ Structure du repo

```
📁 Projet_SI_Decisonnels_esprit_ESB_1BA2_S1/
│
├── 📁 data/
│   └── 📁 raw/                       ← sources brutes (avec anomalies)
│       ├── absences_presences.csv
│       ├── employes_salaires.sql
│       └── formations.xlsx
│
├── 📁 docs/
│   └── Projet_SI_Decisonnels.pdf     ← sujet officiel
│
├── 📁 powerbi/
│   ├── rapport_RH.pbix               ← fichier Power BI Desktop
│   ├── screenshots/                  ← captures des dashboards
│   └── README.md                     ← description du rapport
│
├── 📁 talend/
│   └── JOBS.md                       ← documentation détaillée des 6 jobs ETL
│
├── index.html                        ← dashboard interactif (visualisation ETL)
└── README.md                         ← ce fichier
```

---

## 🛠️ Stack technique

| Outil | Usage |
|---|---|
| **Talend Open Studio** | Pipeline ETL - 6 jobs |
| **PostgreSQL 16** | Base source `rh_entreprise` + DW `rh_dw` |
| **Power BI Desktop** | Rapport décisionnel |
| **HTML / JS / SheetJS** | Dashboard interactif de visualisation |

---

## 🚀 Lancer le projet

```bash
# 1. Cloner le repo
git clone https://github.com/Hmz931/Projet_SI_Decisonnels_esprit_ESB_1BA2_S1.git

# 2. Créer les bases PostgreSQL (dans pgAdmin ou psql)
psql -U postgres -c "CREATE DATABASE rh_entreprise;"
psql -U postgres -c "CREATE DATABASE rh_dw;"

# 3. Importer les données sources
psql -U postgres -d rh_entreprise -f data/raw/employes_salaires.sql

# 4. Ouvrir Talend et importer le projet "Projet_SI_Decisonnels"

# 5. Exécuter les jobs dans l'ordre :
#    J0 → J1 → J2 → J3 → J4 → J5

# 6. Ouvrir powerbi/rapport_RH.pbix dans Power BI Desktop
#    Connecteur : PostgreSQL · host=localhost · port=5432 · db=rh_dw
```

---

<div align="center">
  <sub>ESPRIT · Systèmes d'Information Décisionnels · 1BA2 · 2025–2026</sub>
</div>
