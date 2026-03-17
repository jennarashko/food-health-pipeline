# Food Health Classification Pipeline

Automated data pipeline that ingests USDA food data, labels foods as healthy or unhealthy using nutritional thresholds, and trains a Random Forest classifier to predict food health labels.

This project was independently rebuilt from a group project completed in MSDS 697: Distributed Data Systems, Spring 2026. The original project was completed collaboratively with a team. This version was rebuilt to deepen my understanding of each component.

## Architecture

```
USDA FoodData Central API
        │
        ▼
   [DAG 1: Ingest]
   Fetch JSON → GCS Buckets → MongoDB Atlas (3 collections)
        │
        ▼
   [MongoDB Aggregation Pipeline]
   $unwind foodNutrients → pivot key nutrients →
   score & label healthy/unhealthy → $out to new collections
        │
        ▼
   [DAG 2: Spark ML]
   Load DataFrames from MongoDB → SparkSQL queries →
   Feature engineering → Random Forest Classifier → Evaluate
        │
        ▼
   [DAG 3: Save Results]
   Model metrics + predictions + query timing → GCS
```

## Data Sources

Three datasets from the USDA FoodData Central API:

| Collection | Documents | Description |
|---|---|---|
| survey-food-data-collection | ~5,432 | Food and Nutrient Database for Dietary Studies (FNDDS) |
| legacy-food-data-collection | ~7,793 | Historic USDA National Nutrition Database |
| foundation-food-data-collection | ~365 | Minimally processed commodity foods |

## Health Labeling Criteria

Foods are scored on 6 criteria (1 point each). Score >= 4 = healthy:

- Calories < 300 kcal
- Total sugars < 10g
- Sodium < 400mg
- Saturated fat < 5g
- Fiber > 3g
- Protein > 5g

## ML Models

Two models were trained on 8 nutritional features: calories, protein, total fat, total sugars, sodium, fiber, saturated fat, and cholesterol.

**Random Forest**
- 100 trees, max depth 8

**Gradient Boosted Trees**
- 100 iterations, max depth 8, step size 0.1

| Metric | Random Forest | Gradient Boosted Trees |
|---|---|---|
| Accuracy | 0.9570 | 0.9646 |
| Precision | 0.9571 | 0.9645 |
| Recall | 0.9570 | 0.9646 |
| F1 | 0.9567 | 0.9645 |
| AUC | 0.9896 | 0.9934 |
| Train Time | 3.69s | 66.22s |

The GBT model achieves slightly higher accuracy but takes about 18 times longer to train. The most predictive features were total sugars and calories for both models.

## Setup

```bash
conda activate dds-spring-2026
pip install -r requirements.txt
```

Create a `.env` file with:

```
GCP_SERVICE_ACCOUNT_KEY=/path/to/service-account-key.json
GCP_PROJECT_ID=your-project-id
USDA_API_KEY=your-usda-api-key
MONGODB_ATLAS_USERNAME=your-username
MONGODB_ATLAS_PASSWORD=your-password
MONGODB_CLUSTER_HOST=your-cluster.mongodb.net
MONGODB_DB_NAME=food_health_db
```

## Usage

```bash
# Fetch data from USDA API and store in GCS
python store_data_in_gcs.py

# Load data from GCS into MongoDB
python load_into_mongodb.py

# Run MongoDB aggregation and label foods
python aggregate_and_query.py

# Train ML models
python spark_ml_pipeline.py
```

## (OR) Trigger Airflow DAGs
```
airflow dags trigger usda_ingest_pipeline
airflow dags trigger spark_ml_pipeline
airflow dags trigger save_results_to_gcs
```

## Project Structure
```
├── store_data_in_gcs.py        # USDA API → GCS ingestion
├── load_into_mongodb.py        # GCS → MongoDB Atlas loading
├── aggregate_and_query.py      # MongoDB aggregation pipeline + queries
├── spark_ml_pipeline.py        # PySpark ML (Random Forest + GBT) + SparkSQL
├── explore_usda_api.ipynb      # Data exploration notebook
├── dags/
│   ├── dag_ingest.py           # Airflow DAG 1: API → GCS → MongoDB
│   ├── dag_spark_ml.py         # Airflow DAG 2: Aggregation → Spark ML
│   └── dag_save_results.py     # Airflow DAG 3: Results → GCS
├── output/                     # Local ML outputs (gitignored)
├── requirements.txt
└── .env                        # Credentials (gitignored)
```
