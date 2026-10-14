# Smart Manufacturing Predictive Maintenance (DevSecMLOps)

Predicts whether an industrial machine will fail in the next maintenance window from sensor and
operating data, and returns a failure probability, health score, risk level and recommended action.

## Dataset
`data/raw/manufacturing_machine_data.csv` – 10,000 synthetic machine records (AI4I-style failure modes),
~13% failures, ~2% missing sensor readings. 35 model inputs: operating conditions (hours, age, load,
cycles, maintenance history), sensors (temperatures, rpm, torque, tool wear, vibration, sound, oil, coolant,
hydraulic pressure, voltage, current, power, lubrication), alarms and categorical context
(machine type, line, shift, operator level, ventilation). Targets: `machine_failure` (model) and
`failure_type` (secondary label in the data).

## Models and parameters
| Model | Key parameters |
|---|---|
| Logistic Regression | `max_iter=1000`, `class_weight=balanced` (L2, C=1.0, lbfgs) |
| Random Forest | `n_estimators=300`, `max_depth=10`, `min_samples_leaf=5`, `class_weight=balanced` |
| HistGradientBoosting | RandomizedSearchCV (8 iters, 3-fold, ROC-AUC) over learning_rate, max_depth, max_iter, l2_regularization; `class_weight=balanced` |

Pipeline: median imputation → StandardScaler → OneHotEncoder → engineered features (`temp_delta`,
`mech_power_kw`, `wear_x_torque`, `alarm_score`) → model → isotonic calibration → decision threshold for ≥80% recall.
Best model chosen by PR-AUC. Split 80/20 stratified, `random_state=42`.
Risk bands (config): 0–0.29 Low, 0.30–0.59 Medium, 0.60–0.79 High, 0.80+ Critical.
Health score 0–100 combines failure probability (60%), vibration, oil temperature and tool wear.

## Run
```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
python -m src.generate_data
python -m src.validate
python -m src.train
mlflow ui                        # http://127.0.0.1:5000
uvicorn src.api:app --reload     # http://127.0.0.1:8000/docs
pytest -q
```

## API
`GET /`, `/health`, `/ready`, `/model-info` · `POST /predict` · `POST /predict/batch` (CSV, ≤5 MB).
Set `API_KEY` (see `.env.example`) to require an `x-api-key` header.

## DevSecOps included
Pydantic validation, optional API-key auth, request IDs, secure headers, no stack traces, upload limits,
env-var secrets, non-root Docker, CI with tests + Bandit + pip-audit + Trivy (fails on CRITICAL).

## Not included (from the full outline)
Prometheus/Grafana, Evidently drift reports, DVC, Airflow/Prefect, CodeQL, Gitleaks, Checkov,
scheduled retraining and the RUL / downtime secondary models.
