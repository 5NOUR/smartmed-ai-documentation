# Raspberry Pi Experiment Protocol

Use this as a repeatable lab notebook. Run one test at a time and preserve the raw output.

## A. Capture system baseline

Run on the Raspberry Pi:

```bash
date -Is
hostname
cat /etc/os-release
uname -a
python3 --version
python3 -m pip list
free -h
df -h
vcgencmd get_throttled
vcgencmd measure_temp
```

Record the output in the experiment log. If a command is unavailable, record that rather than guessing.

## B. Capture project version

```bash
cd ~/smartmed-ai
git rev-parse HEAD 2>/dev/null || echo "No Git commit available"
find . -maxdepth 3 -type f | sort
```

## C. Run synthetic AI evaluation

```bash
cd ~/smartmed-ai
python3 scripts/evaluate_model.py
```

Save the output:

```bash
python3 scripts/evaluate_model.py | tee evaluation-$(date +%Y%m%d-%H%M%S).json
```

Note: `tee` saves terminal output, but the file extension `.json` is only appropriate if the output contains only valid JSON. The script currently prints a human-readable heading before the JSON, so save it as `.txt` unless the script is adjusted.

## D. Capture real event rows

```bash
sqlite3 data/smartmed.db \
"SELECT id, compartment_id, event_type, occurred_at, source FROM events ORDER BY occurred_at DESC LIMIT 100;"
```

Do not send this output publicly until it has been checked for personal or identifying information.

## E. Capture weight readings

```bash
sqlite3 data/smartmed.db \
"SELECT compartment_id, grams, occurred_at, is_stable FROM weight_readings ORDER BY occurred_at DESC LIMIT 100;"
```

For calibration, record:
- Empty platform raw reading.
- Known reference mass in grams.
- Loaded raw reading.
- Tare offset.
- Calibration factor.
- Repeated measurements after calibration.

## F. Capture API results

With the application running locally:

```bash
curl http://127.0.0.1:5000/api/health
curl http://127.0.0.1:5000/api/ai/insights/detailed
curl http://127.0.0.1:5000/api/ai/evaluation
curl http://127.0.0.1:5000/api/ai/predictions/next-dose
```

Redact patient details before sharing.

## G. Experiment log template

| Field | Value |
|---|---|
| Experiment ID | |
| Date/time + timezone | |
| Git commit | |
| OS/kernel/Python | |
| Hardware revision | |
| Test objective | |
| Schedule configuration | |
| Number of events | |
| Ground-truth labels reviewed by | |
| TP / FP / TN / FN | |
| Precision / Recall / F1 | |
| False alerts per day | |
| Sensor failures | |
| Notes | |

## H. What to send back for the next review

Send:
1. Synthetic evaluation JSON/text output.
2. De-identified event rows from a controlled test.
3. Weight calibration readings if the load cell is used.
4. Any error traceback.
5. The exact experiment ID and software commit.

Never send patient names, phone numbers, diagnoses, or a raw production database.
