# M-En Level-2 DDoS Detection Research Commands

This README lists the main commands used for the NS-3/OFSwitch13 multi-environment DDoS detection pipeline.

> Do not commit API keys. Keep `CEREBRAS_API_KEY` outside GitHub.

## Stage naming

- **Stage 1:** Offline split validation using Seed 1 features.
- **Stage 2:** Independent replay validation: train on Seed 1, test on Seed 2.
- **Stage 3:** Real-time online validation using Seed 3 and the NS-3 UDP socket bridge.
- **Stage 4:** LLM-assisted alert generation from Stage 3 model outputs.

---

## 0. Useful inspection commands

```bash
cd ~/ns-3-ofswitch13
tree -L 4 -h scratch/men_level2_research
./ns3 show targets | grep men_level2
pgrep -af men_level2_advanced_campaign_dataset_cleananim_stage2
pgrep -af python
```

Check PCAP packet count:

```bash
tcpdump -r scratch/men_level2_research/data/raw/seed3_live_llm/men_level2_external_router_only.pcap -n 2>/dev/null | wc -l
```

---

## 1. Compile / smoke-test NS-3 simulation

```bash
cd ~/ns-3-ofswitch13

./ns3 run "scratch/men_level2/men_level2_advanced_campaign_dataset_cleananim_stage2 \
  --simTime=5 \
  --iotN=3 \
  --tradN=3 \
  --sdnN=3 \
  --botPerEnv=1 \
  --seed=3 \
  --topologyTag=stage2_compile_test \
  --normalLoad=light \
  --attackProfile=udp_flood \
  --anim=0 \
  --pcap=0 \
  --stage2=0 \
  --outDir=scratch/men_level2/output/stage2_compile_test"
```

---

## 2. Generate Seed 1 training dataset

```bash
cd ~/ns-3-ofswitch13

./ns3 run "scratch/men_level2/men_level2_advanced_campaign_dataset_cleananim_stage2 \
  --simTime=600 \
  --iotN=30 \
  --tradN=25 \
  --sdnN=25 \
  --botPerEnv=5 \
  --seed=1 \
  --topologyTag=medium \
  --normalLoad=heavy \
  --attackProfile=all \
  --anim=0 \
  --pcap=1 \
  --stage2=0 \
  --outDir=scratch/men_level2/output/dataset_600s_all_seed1"
```

Extract Seed 1 features:

```bash
time python3 scratch/men_level2/extract_unified_features_level2_streaming.py \
  --pcap scratch/men_level2/output/dataset_600s_all_seed1/men_level2_external_router_only.pcap \
  --truth scratch/men_level2/output/dataset_600s_all_seed1/men_level2_ground_truth.csv \
  --out scratch/men_level2/output/dataset_600s_all_seed1/men_level2_unified_features_v2.csv \
  --window 1.0
```

Create leakage-resistant feature file:

```bash
python3 - <<'PY'
import pandas as pd
inp = "scratch/men_level2/output/dataset_600s_all_seed1/men_level2_unified_features_v2.csv"
out = "scratch/men_level2/output/dataset_600s_all_seed1/men_level2_unified_features_no_leak.csv"
df = pd.read_csv(inp)
df = df.drop(columns=[c for c in ["src_port", "dst_port", "environment"] if c in df.columns])
df.to_csv(out, index=False)
print("Saved:", out)
print("Shape:", df.shape)
print(df["label"].value_counts())
PY
```

---

## 3. Generate Seed 2 independent replay dataset

```bash
cd ~/ns-3-ofswitch13

./ns3 run "scratch/men_level2/men_level2_advanced_campaign_dataset_cleananim_stage2 \
  --simTime=120 \
  --iotN=20 \
  --tradN=15 \
  --sdnN=15 \
  --botPerEnv=3 \
  --seed=2 \
  --topologyTag=medium_test \
  --normalLoad=medium \
  --attackProfile=all \
  --anim=0 \
  --pcap=1 \
  --stage2=0 \
  --outDir=scratch/men_level2/output/dataset_120s_all_seed2"
```

Extract Seed 2 features:

```bash
time python3 scratch/men_level2/extract_unified_features_level2_streaming.py \
  --pcap scratch/men_level2/output/dataset_120s_all_seed2/men_level2_external_router_only.pcap \
  --truth scratch/men_level2/output/dataset_120s_all_seed2/men_level2_ground_truth.csv \
  --out scratch/men_level2/output/dataset_120s_all_seed2/men_level2_unified_features_v2.csv \
  --window 1.0
```

---

## 4. Create research folder layout

```bash
cd ~/ns-3-ofswitch13

mkdir -p scratch/men_level2_research/{configs,logs,models,reports}
mkdir -p scratch/men_level2_research/data/{raw,features,ground_truth}
mkdir -p scratch/men_level2_research/data/raw/{seed1,seed2,seed3}
mkdir -p scratch/men_level2_research/results/{comparison_tables,stage1_offline_split,stage2_replay_seed1_to_seed2,stage3_realtime_seed3}
mkdir -p scratch/men_level2_research/src/{ns3,realtime,replay,training,utils}
```

Copy files into research folder:

```bash
cp scratch/men_level2/output/dataset_600s_all_seed1/men_level2_external_router_only.pcap scratch/men_level2_research/data/raw/seed1/
cp scratch/men_level2/output/dataset_120s_all_seed2/men_level2_external_router_only.pcap scratch/men_level2_research/data/raw/seed2/
cp scratch/men_level2/output/dataset_600s_all_seed1/men_level2_ground_truth.csv scratch/men_level2_research/data/ground_truth/seed1_ground_truth.csv
cp scratch/men_level2/output/dataset_120s_all_seed2/men_level2_ground_truth.csv scratch/men_level2_research/data/ground_truth/seed2_ground_truth.csv
cp scratch/men_level2/output/dataset_600s_all_seed1/men_level2_unified_features_no_leak.csv scratch/men_level2_research/data/features/seed1_features_no_leak.csv
cp scratch/men_level2/output/dataset_120s_all_seed2/men_level2_unified_features_v2.csv scratch/men_level2_research/data/features/seed2_features.csv
cp scratch/men_level2/men_level2_advanced_campaign_dataset_cleananim_stage2.cc scratch/men_level2_research/src/ns3/
cp scratch/men_level2/extract_unified_features_level2_streaming.py scratch/men_level2_research/src/utils/
cp scratch/men_level2/evaluate_stage1_predictions.py scratch/men_level2_research/src/utils/
cp scratch/men_level2/pcap_stream_replay_predict.py scratch/men_level2_research/src/replay/
cp scratch/men_level2/stage2_online_ml_server.py scratch/men_level2_research/src/realtime/
```

---

## 5. Stage 1 — Offline split validation

Install XGBoost if needed:

```bash
python3 -m pip install xgboost
```

Train tabular models:

```bash
cd ~/ns-3-ofswitch13
python3 scratch/men_level2_research/src/training/train_tabular_models.py
```

Train deep sequence models:

```bash
cd ~/ns-3-ofswitch13
python3 scratch/men_level2_research/src/training/train_deep_sequence_models.py
```

Merge Stage 1 results:

```bash
python3 - <<'PY'
import pandas as pd
tabular_path = "scratch/men_level2_research/results/stage1_offline_split/stage1_tabular_model_comparison.csv"
deep_path = "scratch/men_level2_research/results/stage1_offline_split/stage1_deep_sequence_model_comparison.csv"
out = "scratch/men_level2_research/results/comparison_tables/VALID_stage1_all_models_comparison.csv"
tabular = pd.read_csv(tabular_path); deep = pd.read_csv(deep_path)
tabular["model_family"] = "tabular_ml"; deep["model_family"] = "deep_sequence"
combined = pd.concat([tabular, deep], ignore_index=True, sort=False)
cols = ["model_family","model","accuracy","precision_weighted","recall_weighted","f1_weighted","roc_auc","latency_ms_per_sample","throughput_predictions_per_sec","train_time_sec","memory_mb_delta"]
combined = combined[cols].sort_values("accuracy", ascending=False).reset_index(drop=True)
combined.insert(0, "rank", range(1, len(combined)+1))
combined.to_csv(out, index=False)
print(combined)
print("Saved:", out)
PY
```

Create Stage 1 report tables:

```bash
python3 - <<'PY'
import pandas as pd
path = "scratch/men_level2_research/results/comparison_tables/VALID_stage1_all_models_comparison.csv"
df = pd.read_csv(path)
df[["rank","model","model_family","accuracy","precision_weighted","recall_weighted","f1_weighted","roc_auc"]].to_csv("scratch/men_level2_research/reports/STAGE1_detection_performance_table.csv", index=False)
df[["rank","model","model_family","latency_ms_per_sample","throughput_predictions_per_sec","train_time_sec","memory_mb_delta"]].to_csv("scratch/men_level2_research/reports/STAGE1_computational_performance_table.csv", index=False)
PY
```

---

## 6. Stage 2 — Independent Seed 1 to Seed 2 validation

```bash
cd ~/ns-3-ofswitch13
python3 scratch/men_level2_research/src/replay/stage2_seed1_to_seed2_evaluate_all_models.py
```

View results:

```bash
cat scratch/men_level2_research/results/stage2_replay_seed1_to_seed2/stage2_all_models_seed1_to_seed2_comparison.csv
```

Create Stage 2 report tables:

```bash
python3 - <<'PY'
import pandas as pd
path = "scratch/men_level2_research/results/stage2_replay_seed1_to_seed2/stage2_all_models_seed1_to_seed2_comparison.csv"
df = pd.read_csv(path)
df[["rank","model","model_family","accuracy","precision_weighted","recall_weighted","f1_weighted","roc_auc"]].to_csv("scratch/men_level2_research/reports/STAGE2_detection_performance_table.csv", index=False)
df[["rank","model","model_family","latency_ms_per_sample","throughput_predictions_per_sec","total_prediction_time_sec","memory_mb_delta"]].to_csv("scratch/men_level2_research/reports/STAGE2_computational_performance_table.csv", index=False)
PY
```

---

## 7. Stage 3 — Real-time online inference

Terminal 1: start all-model Python server.

```bash
cd ~/ns-3-ofswitch13

python3 scratch/men_level2_research/src/realtime/stage3_online_ml_server_all_models.py \
  --modelDir scratch/men_level2_research/models \
  --host 127.0.0.1 \
  --port 9999 \
  --out scratch/men_level2_research/results/stage3_realtime_seed3/stage3_online_predictions_all_models_seed3.csv
```

Terminal 2: run NS-3 with socket bridge.

```bash
cd ~/ns-3-ofswitch13

./ns3 run "scratch/men_level2/men_level2_advanced_campaign_dataset_cleananim_stage2 \
  --seed=3 \
  --stage2=true \
  --stage2Host=127.0.0.1 \
  --stage2Port=9999 \
  --outDir=scratch/men_level2_research/data/raw/seed3"
```

Check if NS-3 has finished:

```bash
pgrep -af men_level2_advanced_campaign_dataset_cleananim_stage2
```

Summarize Stage 3 predictions:

```bash
python3 - <<'PY'
import pandas as pd
path = "scratch/men_level2_research/results/stage3_realtime_seed3/stage3_online_predictions_all_models_seed3.csv"
df = pd.read_csv(path)
print("Rows:", len(df))
print("\nStatus:"); print(df["status"].value_counts(dropna=False))
print("\nModel counts:"); print(df["model"].value_counts(dropna=False))
print("\nModel x prediction:"); print(df.groupby(["model","predicted_label"]).size())
print("\nLatency by model:"); print(df.groupby("model")["prediction_latency_ms"].describe())
print("\nWindow range by model:"); print(df.groupby("model")[["window_start","window_end"]].agg(["min","max"]))
PY
```

---

## 8. Cerebras LLM setup and test

Install SDK:

```bash
python3 -m pip install cerebras-cloud-sdk
```

Set key:

```bash
export CEREBRAS_API_KEY="YOUR_CEREBRAS_API_KEY_HERE"
```

Test:

```bash
python3 - <<'PY'
import os
from cerebras.cloud.sdk import Cerebras
client = Cerebras(api_key=os.environ["CEREBRAS_API_KEY"])
response = client.chat.completions.create(
    model="llama3.1-8b",
    messages=[{"role":"user","content":"In one sentence, explain why fast inference matters for real-time DDoS detection."}],
)
print(response.choices[0].message.content)
PY
```

---

## 9. Stage 4 — LLM alert generation from Stage 3 CSV

Post-processing alert generation:

```bash
python3 scratch/men_level2_research/src/realtime/generate_llm_alerts_from_stage3.py \
  --pred scratch/men_level2_research/results/stage3_realtime_seed3/stage3_online_predictions_all_models_seed3.csv \
  --out scratch/men_level2_research/results/stage3_realtime_seed3/stage3_llm_alerts_seed3.csv \
  --interval 10 \
  --minAttackRatio 0.60 \
  --minAvgProb 0.60
```

View alerts:

```bash
python3 - <<'PY'
import pandas as pd
path = "scratch/men_level2_research/results/stage3_realtime_seed3/stage3_llm_alerts_seed3.csv"
df = pd.read_csv(path)
for _, r in df.iterrows():
    print("="*80)
    print(f"Interval: {r['interval_start']:.0f}-{r['interval_end']:.0f}s")
    print(f"Attack ratio: {r['attack_ratio']:.2f}")
    print(f"Avg attack probability: {r['avg_attack_probability']:.2f}")
    print(r["llm_alert"])
PY
```

---

## 10. Live LLM alerting while NS-3 runs

Clean live files:

```bash
rm -f scratch/men_level2_research/results/stage3_realtime_seed3/stage3_online_predictions_live_llm.csv
rm -f scratch/men_level2_research/results/stage3_realtime_seed3/stage3_llm_alerts_live.csv
```

Terminal 1: Stage 3 all-model server.

```bash
python3 scratch/men_level2_research/src/realtime/stage3_online_ml_server_all_models.py \
  --modelDir scratch/men_level2_research/models \
  --host 127.0.0.1 \
  --port 9999 \
  --out scratch/men_level2_research/results/stage3_realtime_seed3/stage3_online_predictions_live_llm.csv
```

Terminal 2: live LLM watcher.

```bash
python3 scratch/men_level2_research/src/realtime/live_llm_alert_watcher.py \
  --pred scratch/men_level2_research/results/stage3_realtime_seed3/stage3_online_predictions_live_llm.csv \
  --out scratch/men_level2_research/results/stage3_realtime_seed3/stage3_llm_alerts_live.csv \
  --interval 10 \
  --pollSec 2 \
  --minAttackRatio 0.55 \
  --minAvgProb 0.55
```

Terminal 3: NS-3 live run.

```bash
./ns3 run "scratch/men_level2/men_level2_advanced_campaign_dataset_cleananim_stage2 \
  --seed=3 \
  --stage2=true \
  --stage2Host=127.0.0.1 \
  --stage2Port=9999 \
  --outDir=scratch/men_level2_research/data/raw/seed3_live_llm"
```

Terminal 4: watch alerts.

```bash
tail -f scratch/men_level2_research/results/stage3_realtime_seed3/stage3_llm_alerts_live.csv
```

---

## 11. Final Stage 3 report generation

```bash
python3 - <<'PY'
import subprocess
from pathlib import Path
import pandas as pd

base = Path("scratch/men_level2_research")
pred_path = base / "results/stage3_realtime_seed3/stage3_online_predictions_live_llm.csv"
alert_path = base / "results/stage3_realtime_seed3/stage3_llm_alerts_live.csv"
pcap_path = base / "data/raw/seed3_live_llm/men_level2_external_router_only.pcap"
out_dir = base / "reports"; out_dir.mkdir(parents=True, exist_ok=True)

packet_count = None
if pcap_path.exists():
    packet_count = int(subprocess.check_output(f"tcpdump -r {pcap_path} -n 2>/dev/null | wc -l", shell=True).decode().strip())

pred = pd.read_csv(pred_path)
valid = pred[pred["status"] == "ok"].copy() if "status" in pred.columns else pred.copy()
warmup_count = int((pred["status"] == "warmup").sum()) if "status" in pred.columns else 0

alerts = pd.read_csv(alert_path) if alert_path.exists() else pd.DataFrame()
if not alerts.empty:
    alerts = alerts.drop_duplicates(subset=["interval_start","interval_end"], keep="last")
    triggered = alerts[alerts["alert_generated"] == True].copy()
else:
    triggered = pd.DataFrame()

summary = pd.DataFrame([
    {"metric":"external_router_pcap_packets","value":packet_count},
    {"metric":"stage3_prediction_records_total","value":len(pred)},
    {"metric":"stage3_valid_predictions","value":len(valid)},
    {"metric":"stage3_warmup_records","value":warmup_count},
    {"metric":"models_evaluated","value":pred["model"].nunique()},
    {"metric":"window_start_min","value":pred["window_start"].min()},
    {"metric":"window_end_max","value":pred["window_end"].max()},
    {"metric":"llm_alert_windows_total","value":len(alerts)},
    {"metric":"llm_alerts_triggered","value":len(triggered)},
    {"metric":"llm_mean_latency_ms_triggered_alerts","value":triggered["llm_latency_ms"].mean() if len(triggered) else 0},
])
summary.to_csv(out_dir / "STAGE3_realtime_pipeline_summary.csv", index=False)

latency = (
    valid.groupby("model")["prediction_latency_ms"]
    .agg(["count","mean","median","min","max","std"])
    .reset_index()
    .rename(columns={"count":"valid_predictions","mean":"mean_latency_ms","median":"median_latency_ms","min":"min_latency_ms","max":"max_latency_ms","std":"std_latency_ms"})
    .sort_values("mean_latency_ms")
)
latency.to_csv(out_dir / "STAGE3_realtime_model_latency_table.csv", index=False)

pred_dist = valid.groupby(["model","predicted_label"]).size().reset_index(name="count")
pred_dist.to_csv(out_dir / "STAGE3_prediction_distribution_by_model.csv", index=False)

if not alerts.empty:
    alert_table = alerts[["interval_start","interval_end","total_predictions","attack_predictions","benign_predictions","attack_ratio","avg_attack_probability","dominant_protocol","alert_generated","llm_latency_ms","llm_alert"]]
    alert_table.to_csv(out_dir / "STAGE3_llm_alert_summary_table.csv", index=False)

print("\nStage 3 pipeline summary:")
print(summary.to_string(index=False))
print("\nLatency by model:")
print(latency.to_string(index=False))
if not alerts.empty:
    print("\nLLM alert windows:")
    print(alert_table[["interval_start","interval_end","attack_ratio","avg_attack_probability","dominant_protocol","alert_generated","llm_latency_ms"]].to_string(index=False))
PY
```

View final reports:

```bash
cat scratch/men_level2_research/reports/STAGE3_realtime_pipeline_summary.csv
column -s, -t scratch/men_level2_research/reports/STAGE3_realtime_model_latency_table.csv
column -s, -t scratch/men_level2_research/reports/STAGE3_llm_alert_summary_table.csv | less -S
```
