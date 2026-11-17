# SmartMed AI
## Smart Medication Adherence System

SmartMed AI is a Raspberry Pi-based medication-adherence prototype. The supplied source code combines a Flask dashboard/API, SQLite storage, compartment sensors, reminder and dose-window logic, a buzzer/LED alert system, an HX711 load-cell path for one compartment, and lightweight behavioral analytics.

> **Safety notice:** This repository is a prototype for monitoring and reminders. It is not a medical device, does not verify that a person swallowed a medication, and must not prescribe, change, or recommend medication doses. AI outputs are experimental behavioral indicators, not clinical predictions.

## Current implementation at a glance

- Four medication compartments monitored by MC-38 magnetic sensors.
- Eight status LEDs, one green and one red per compartment.
- Buzzer alert output.
- DS3231 real-time clock interface over I²C.
- HX711/load-cell reading path assigned to compartment 4.
- Flask web application and JSON API.
- SQLite database with WAL mode.
- Anomaly analysis using a custom Isolation-Forest implementation and Mahalanobis distance.
- A rule-based behavior classifier and composite risk score.
- A separate next-dose risk heuristic based on event frequency, weekday/time buckets, and recent event-count trend.

## AI implementation status

The current AI is primarily **lightweight unsupervised/statistical analytics**, not a supervised deep-learning model.

The anomaly detector uses five event features:
1. Delay from the configured compartment schedule.
2. Hour of day.
3. Day of week.
4. Weekend indicator.
5. Minutes since the previous opening event.

The anomaly pipeline standardizes features, calculates Isolation Forest and Mahalanobis scores, combines percentile ranks (60% / 40%), and flags events above the 88th percentile.

The repository contains an evaluation script that generates synthetic labeled events. Running it with 200 events and seed 42 produced:

| Metric | Result |
|---|---:|
| Accuracy | 95.0% |
| Precision | 62.5% |
| Recall | 93.8% |
| F1-score | 75.0% |
| True positives | 15 |
| False positives | 9 |
| True negatives | 175 |
| False negatives | 1 |

**These are synthetic, in-sample algorithm-test results only. They are not patient accuracy, clinical validation, or evidence of real-world dose-miss prediction performance.** The evaluator fits and evaluates the detector on the same generated event set and does not use a held-out test set.

## Repository map

```text
smartmed-ai/
├── ai/
│   ├── ml_core.py
│   ├── cognitive_engine.py
│   ├── prediction_engine.py
│   ├── model_evaluation.py
│   ├── timeline_builder.py
│   └── usage_analyzer.py
├── app/
│   ├── routes/api.py
│   ├── services/reminder_service.py
│   ├── templates/
│   └── static/
├── config/pins.py
├── database/
│   ├── database.py
│   ├── schema.sql
│   └── schema_extensions.sql
├── hardware/
│   ├── compartment_sensors.py
│   ├── dose_window_manager.py
│   ├── weight_sensor.py
│   ├── rtc_manager.py
│   └── alert_controller.py
├── scripts/
├── tests/
├── main.py
└── requirements.txt
```

## Setup overview

The project `requirements.txt` lists Flask, gpiozero, lgpio, smbus2, and pytest. The AI source imports NumPy, but NumPy is not currently declared in `requirements.txt`; install a compatible NumPy package separately or update the dependency file before a clean deployment. The comments recommend installing the Raspberry Pi OS packages through `apt`.

Example on Raspberry Pi OS:

```bash
sudo apt update
sudo apt install python3-flask python3-gpiozero python3-lgpio python3-smbus2 python3-pytest
```

From the project directory:

```bash
python3 main.py
```

The application binds to `0.0.0.0:5000`. Open `http://<RASPBERRY_PI_IP>:5000/` from a device on the same network.

Hardware startup requires the wiring, GPIO permissions, I²C setup, and sensors to be correct. Do not run the hardware application on a normal desktop and expect GPIO components to work.

The base initializer executes `database/schema.sql` only. If using medical-profile, clinical-notes, or prediction-history features, back up the database and apply `database/schema_extensions.sql` as a separate migration.

## Useful API endpoints

| Endpoint | Purpose |
|---|---|
| `GET /api/health` | Basic health response |
| `GET /api/system/status` | System status |
| `GET /api/ai/insights` | Behavioral analysis via `UsageAnalyzer` |
| `GET /api/ai/insights/detailed` | Detailed cognitive analysis |
| `GET /api/ai/evaluation` | Synthetic anomaly evaluation |
| `GET /api/ai/predictions/next-dose` | Heuristic next-dose risk estimate |
| `GET /api/ai/predictions/history` | Stored prediction history |
| `GET /api/timeline` | Unified event/reminder/alert timeline |
| `GET /api/report/monthly` | Aggregated report |

## Documentation

- [System architecture](docs/ARCHITECTURE.md)
- [AI technical specification](docs/AI_TECHNICAL_SPEC.md)
- [Evaluation results and graph interpretation](docs/AI_EVALUATION.md)
- [Hardware and data collection](docs/HARDWARE_AND_DATA.md)
- [Known limitations and engineering recommendations](docs/LIMITATIONS_AND_NEXT_STEPS.md)
- [Raspberry Pi experiment protocol](docs/EXPERIMENT_PROTOCOL.md)
- [GitHub publishing guide](docs/GITHUB_GUIDE.md)

## Important data-handling note

The SQLite database under `data/` is intentionally excluded from source control. Do not publish patient names, phone numbers, medical profiles, notes, medication lists, or event histories. Keep `.db`, `.db-wal`, `.db-shm`, `.env`, and credentials out of public repositories.
