# 📊 Rapport Power BI — Gestion RH

> Visualisation décisionnelle des données de l'entrepôt RH
> Projet SI Décisionnels · ESPRIT · 1BA2 · 2025–2026

## Pages du rapport

### 1. Vue Générale RH
- KPI : nombre total d'employés, masse salariale, taux d'absence
- Répartition des employés par service (camembert)
- Évolution de la masse salariale mensuelle (courbe)

### 2. Analyse des Absences
- Total jours d'absence par service (barres)
- Répartition par motif d'absence (camembert)
- Calendrier des absences (matrice mois × service)
- Top 10 employés les plus absents

### 3. Analyse des Formations
- Nombre de formations par service (barres groupées)
- Coût total des formations par année (colonnes)
- Statut des formations : Complétée / En cours / Planifiée (anneau)
- Formations les plus suivies (nuage de mots ou barres)

### 4. Profil Employé (drill-through)
- Fiche individuelle : absences, formations, salaire
- Historique sur 12 mois

## Connexion à la source
- Source : base MySQL/PostgreSQL `rh_dw`
- Tables utilisées : `FAIT_RH`, `DIM_EMPLOYE`, `DIM_TEMPS`, `DIM_SERVICE`, `DIM_FORMATION`, `DIM_ABSENCE`
- Mode : Import (refresh manuel)

## Fichiers
| Fichier | Description |
|---|---|
| `rapport_RH.pbix` | Fichier Power BI Desktop |
| `screenshots/` | Captures d'écran des pages |