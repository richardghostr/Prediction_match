# Prediction_match

Résumé
------

`Prediction_match` est un projet académique visant à prédire le résultat des matchs de Ligue 1 (saison 2025-2026) selon trois classes : victoire domicile (`1`), match nul (`0`) et victoire extérieur (`-1`). Le projet met l'accent sur l'ingénierie de features, la qualité des données et l'évaluation rigoureuse de modèles de classification.

Table des matières
------------------

- Contexte & objectifs
- Structure du dépôt
- Description des données
- Notebook principal
- Pipeline recommandé
- Exécution & installation
- Feature engineering (extraits)
- Modèles testés & évaluation
- Reproductibilité & bonnes pratiques
- Améliorations futures
- Auteur / Contact

Contexte & objectifs
---------------------

Objectif : construire un classifieur multi-classes capable de prédire le résultat d'un match de Ligue 1 en s'appuyant sur l'historique 2012–2024 et des métadonnées (joueurs, clubs, événements). Le livrable attendu est :

- une analyse claire et documentée (EDA),
- un ou plusieurs modèles entraînés et évalués,
- un fichier de prédictions pour les matchs `match_2025.csv` (format `sample_results.csv`).

Structure du dépôt
------------------

Racine du projet :

- `README.md` : ce fichier.
- `data/` : dossier contenant les CSV sources (voir section Données).
- `prediction.ipynb` : notebook principal (EDA, feature engineering, modélisation, génération des prédictions).

Description des données
-----------------------

Les données se trouvent dans le dossier `data/` et comprennent :

- `matchs_2013_2024.csv` — historique des matchs (clé : `game_id`). Colonnes importantes : `game_id`, `season`, `date`, `home_club_id`, `away_club_id`, `home_club_goals`, `away_club_goals`, `home_club_position`, `away_club_position`, `results` (-1/0/1), `attendance`, `home_club_formation`, `away_club_formation`.
- `match_2025.csv` — matchs à prédire (pas de colonne `results`). Même schéma que `matchs_2013_2024.csv` hors scores.
- `clubs_fr.csv` — métadonnées des clubs : `club_id`, `club_code`, `name`, `squad_size`, `average_age`, `foreigners_percentage`, `stadium_seats`, `net_transfer_record`, `coach_name`.
- `player_appearance.csv` — statistiques individuelles par match : `appearance_id`, `game_id`, `player_id`, `player_club_id`, `date`, `goals`, `assists`, `yellow_cards`, `red_cards`, `minutes_played`.
- `player_valuation_before_season.csv` — valeurs de marché des joueurs dans le temps : `player_id`, `date`, `market_value_in_eur`, `current_club_id`.
- `game_events_before2025.csv` — événements par minute (but, carton, substitution) : `game_event_id`, `date`, `game_id`, `minute`, `type`, `club_id`, `player_id`, `description` (texte libre à parser).
- `game_lineups.csv` — compositions d'équipes par match : `game_lineups_id`, `date`, `game_id`, `player_id`, `club_id`, `player_name`, `type` (starting_lineup / substitutes), `position`, `team_captain`.
- `sample_results.csv` — exemple du format attendu pour la soumission : `game_id,results`.

Remarques qualité & prétraitement
--------------------------------

- Clés primaires et jointures : `game_id`, `player_id`, `club_id` sont cohérents entre fichiers ; privilégier ces colonnes pour les merges.
- Dates : convertir en type `datetime` pour filtres temporels et rolling windows.
- Valeurs monétaires : nettoyer `net_transfer_record` et `market_value_in_eur` (supprimer `€`, `m`, `k`, convertir en float en euros).
- `game_events` contient du texte libre : il faudra parser `type` et `description` pour extraire événements utiles (passes décisives, penalties, etc.).
- Données manquantes notables : `home_club_formation` / `away_club_formation`, `attendance` et parfois `coach_name`. Imputer ou traiter selon l'utilisation.

Notebook principal
------------------

Le notebook `prediction.ipynb` contient :

1. Chargement et vérification des datasets.
2. Analyse exploratoire (distributions, corrélations, visualisations temporelles).
3. Prétraitement (nettoyage valeurs, conversion, agrégations).
4. Feature engineering détaillé :
	- Forme récente (rolling sur N derniers matchs),
	- Statistiques domicile/extérieur par saison,
	- Historique des confrontations directes (H2H),
	- Agrégats joueurs (buts, assists, minutes),
	- Valeur marchande de l'effectif par saison,
	- Features clubs (taille effectif, âge moyen, transferts).
5. Assemblage du dataset final (jointures et imputation).
6. Entraînement et comparaison de plusieurs modèles (LogisticRegression, DecisionTree, RandomForest, GradientBoosting, XGBoost, LightGBM) avec split temporel.
7. Tuning (RandomizedSearchCV pour LightGBM) et entraînement final sur toutes les données historiques.
8. Génération du fichier `predictions_2025.csv`.

Pipeline recommandé (haute‑niveau)
--------------------------------

1. Préparation de l'environnement Python (voir Exécution).
2. Charger et valider les CSV (contrôles d'intégrité, types, duplications).
3. Nettoyage simple : dates, types numériques, conversion valeurs monétaires.
4. Construire features vectorisées (éviter boucles lourdes) : groupby/rolling/agg.
5. Imputation & encodage (LabelEncoder pour `club_id` si utile, OneHot si petites cardinalités).
6. Pipeline ML : `SimpleImputer` -> `StandardScaler` (si besoin) -> `Estimator` (via `sklearn.pipeline.Pipeline`).
7. Validation : split temporel (entraînement sur 2012–2023, test sur 2024) + CV temporelle si possible.
8. Tuning & entraînement final.
9. Génération des prédictions sur `match_2025.csv` et export CSV.

Exécution & installation
------------------------

Exemple d'environnement recommandé :

- Python 3.10+ (3.8+ acceptable)
- Installer dépendances usuelles : pandas, numpy, scikit-learn, matplotlib, seaborn, xgboost, lightgbm

Commande rapide :

```bash
python -m venv .venv
source .venv/bin/activate    # Linux / macOS
.venv\Scripts\Activate.ps1  # Windows PowerShell
pip install -r requirements.txt
```

Si `requirements.txt` n'existe pas, installer :

```bash
pip install pandas numpy scikit-learn matplotlib seaborn xgboost lightgbm joblib
```

Feature engineering (extraits et recommandations)
-------------------------------------------------

- Forme récente : utiliser `groupby(club).apply(rolling(...))` ou construire des tables pivot avant d'appliquer `rolling` par club pour scaler correctement.
- H2H : agréger les derniers N matchs entre deux clubs via filtres sur `game_id`/`date` (éviter itérations ligne-à-ligne si possible en pré-computant key=(home,away) et en utilisant `groupby`).
- Valeur d'effectif : garder la dernière valeur connue avant le début de la saison (filtrer `valuation` par date puis groupby `current_club_id`).
- Agrégats joueurs : agréger `player_appearance` par `game_id`+`player_club_id` pour obtenir stats par équipe par match.

Modélisation & évaluation
-------------------------

- Split temporel : entraîner sur saisons < 2024, tester sur saison 2024.
- Métriques : accuracy globale, matrice de confusion (surtout pour la classe `0`), F1 par classe, courbes de calibration si probabilités utilisées.
- Baseline simple : prédire toujours victoire à domicile (mesure de référence ≈ proportion des victoires domicile ~44%).
- Recommandation modèles : commencer par arbres (RandomForest, LightGBM), puis calibrer et essayer stacking.

Reproductibilité & bonnes pratiques
----------------------------------

- Fixer une `SEED` pour l'ensemble des expériences.
- Utiliser `Pipeline` pour enchaîner imputation/scaling/estimation et faciliter le CV.
- Sauvegarder modèles et hyperparamètres (`joblib.dump`) après tuning.
- Versionner les transformations (ex. normalisations, encodages) pour pouvoir appliquer exactement les mêmes sur `match_2025.csv`.
- Documenter toute décision (features créées, colonnes exclues pour éviter les fuites de données).

Risques, limites et points d'attention
------------------------------------

- Fuite de données : ne pas utiliser les scores (`home_club_goals`, `away_club_goals`) comme features pour prédictions pré‑match.
- Données manquantes dans `lineups` et `appearances` pour les matchs futurs : prévoir features robustes qui n'exigent pas la composition exacte.
- Performances limitées : même de bonnes approches atteindront souvent 50–60% d'accuracy maximum sur ce type de problème.

Améliorations futures
---------------------

- Intégrer données avancées (xG, tirs, possession) si disponibles.
- Modélisation probabiliste et calibration (pour produire cotes prédictives).
- Approche temporelle avancée (LSTM, TCN, Transformers temporels) pour capter séquences de performances.
- Incorporer informations externes : blessures, suspensions, météo, calendrier de compétitions.

Fichiers clés et livrables
-------------------------

- `data/` : fichiers sources (voir ci‑dessus).
- `prediction.ipynb` : notebook de bout en bout.
- `predictions_2025.csv` : sortie attendue (format identique à `sample_results.csv`).

Licence & crédit
----------------

Ce dépôt est destiné à un usage pédagogique / académique. Merci d'indiquer l'auteur dans toute réutilisation.

Auteur / Contact
-----------------

N'hésite pas à me donner des instructions pour :

- générer des scripts automatisés (`src/`) pour le pipeline, ou
- vectoriser et optimiser certaines étapes (formule de rolling/H2H), ou
- écrire un `requirements.txt` et un petit `Makefile` pour reproduire les étapes.

---

Date de génération : 2026-04-17
