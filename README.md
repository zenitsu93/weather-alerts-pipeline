# Pipeline d'alertes météo

Collecter les alertes météo actives du National Weather Service américain, les charger dans Snowflake, et les transformer en modèle dimensionnel exploitable avec dbt.

## Le trajet de la donnée

| Étape | Fichier | Ce qui se passe |
| --- | --- | --- |
| **Extraction** | `alerts.py` | Appel à `api.weather.gov/alerts/active`, aplatissement du JSON en tableau |
| **Nettoyage** | `clean.py` | Normalisation des champs, traitement des valeurs manquantes |
| **Chargement** | `snowflake_connection.py` | Insertion dans l'entrepôt |
| **Transformation** | `weather_data/` | Projet dbt : staging, intermédiaire, puis marts |

## L'architecture dbt

C'est la partie qui fait la différence avec un simple script.

```
models/
  staging/        stg_weather_alerts.sql    nettoyage et typage, un modèle par source
  intermediate/   int_alerts_metrics.sql    agrégations et logique métier
  marts/core/     dim_events.sql            dimension : types d'événements
                  dim_alert_areas.sql       dimension : zones géographiques
```

Cette séparation en trois couches est la pratique standard, et elle se justifie : la couche **staging** ne fait que rendre les données propres et typées, sans logique métier — elle est donc stable et réutilisable. La couche **intermédiaire** porte les calculs. Les **marts** exposent un modèle dimensionnel directement interrogeable par un outil de visualisation.

Quand une règle métier change, on modifie une seule couche. Quand la source change de format, on ne touche qu'au staging.

Les fichiers `sources.yml` et `schema.yml` déclarent les sources et les tests — c'est ce qui permet à dbt de vérifier automatiquement l'unicité des clés et l'absence de valeurs nulles là où elles ne devraient pas être.

## Mise en route

```bash
pip install -r requirements.txt
```

Renseignez les identifiants Snowflake, puis :

```bash
python alerts.py                 # collecte
python clean.py                  # nettoyage
python snowflake_connection.py   # chargement

cd weather_data
dbt deps
dbt run
dbt test
```

## Contenu du dépôt

| Chemin | Rôle |
| --- | --- |
| `alerts.py` | Collecte depuis l'API NWS |
| `clean.py` | Nettoyage |
| `snowflake_connection.py` | Chargement dans l'entrepôt |
| `weather_data/` | Projet dbt |
| `weather_alerts.csv` | Extraction d'exemple |

## Limites

Le pipeline s'exécute à la main, étape par étape. Un ordonnanceur — Airflow, ou dbt Cloud — serait la suite logique, les alertes météo n'ayant d'intérêt que rafraîchies régulièrement.

L'API ne renvoie que les alertes **actives**. Sans collecte périodique et historisation, il n'y a pas de série temporelle à analyser, seulement un instantané.
