# 🗄️ Entrepôt de Données RH — Projet SI Décisionnels

> ESPRIT · 1BA2 · Enseignant : Sofien Boutaib · 2025–2026  
> Deadline : 14 Mars 2026

## Contexte

Centralisation des données RH d'une entreprise via un pipeline ETL complet (Talend),
modélisé en schéma étoile et chargé dans un entrepôt de données PostgreSQL.

## Sources de données

| Fichier | Type | Lignes | Anomalies |
|---|---|---|---|
| `absences_presences.csv` | CSV | ~624 | 66 doublons · nulls motif/durée |
| `employes_salaires.sql` | SQL | 100 emp + 1200 salaires | 12 doublons · nulls ville/email |
| `formations.xlsx` | Excel | ~280 | 30 doublons · nulls coût/statut |

> ⚠️ Les données sont intentionnellement imparfaites (~12% doublons, ~10% nulls)
> pour simuler un cas réel et justifier les étapes de nettoyage dans Talend.

## Structure du projet