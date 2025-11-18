# Response to ML Engineer - Issue Investigation

**Date:** November 6, 2025  
**From:** Data Engineer  
**To:** ML Engineer  
**Re:** 100% Accuracy Issue Investigation

---

## Issue Summary

You reported achieving **100% accuracy** even with noise and overlaps. I've investigated the data generation process and identified the **root cause**.

---

## Root Cause: Perfect Class Separation

The synthetic dataset was generated with **non-overlapping ranges** for Normal vs Faulty states, creating an unrealistically easy classification problem.

### Evidence from Analysis:

#### 1. **Zero Overlap Between Classes**

| Feature | Normal Range | Faulty Range | Gap |
|---------|--------------|--------------|-----|
| RSSI | [-75.56, -44.45] dBm | [-124.41, -90.00] dBm | **14.44 dBm** |
| SINR | [12.38, 31.92] dB | [-5.00, 5.00] dB | **7.38 dB** |
| Throughput | [66.55, 160.78] Mbps | [10.00, 39.99] Mbps | **26.56 Mbps** |
| Latency | [5.00, 20.00] ms | [80.01, 297.84] ms | **60.01 ms** |
| Jitter | [1.00, 5.00] ms | [20.00, 50.00] ms | **15.00 ms** |
| Packet Loss | [0.00, 1.00]% | [5.00, 29.65]% | **4.00%** |

**Result:** No ambiguous samples exist in the boundary regions.

#### 2. **Perfect Feature Discrimination**

Cohen's d effect sizes (measures separation):

- RSSI: **6.28** (Perfect)
- SINR: **6.03** (Perfect)
- Throughput: **5.72** (Perfect)
- Network Quality Score: **10.78** (Extremely Perfect)

*Note: Cohen's d > 0.8 is considered "large effect", ours are 5-10x that threshold!*

#### 3. **Single-Feature Classification**

Any single feature alone can classify with near-perfect accuracy:

```
rssi_dbm        →  100.00% accuracy
sinr_db         →  100.00% accuracy  
throughput_mbps →   99.97% accuracy
jitter_ms       →  100.00% accuracy
latency_ms      →   99.38% accuracy
packet_loss     →   96.50% accuracy
```

**Even a simple `if rssi < -81: Faulty` achieves 100% accuracy!**

---

## Why This Happened

The data generation script used:

```python
# Normal data - completely separate range
rssi = np.random.uniform(-70, -50)

# Faulty data - completely separate range  
rssi = np.random.uniform(-110, -90)
```

This creates a **14+ dBm gap** where no samples exist, making classification trivial.

---

## Why Your Model Gets 100%

1. **Decision Trees:** Can create a single split (e.g., `rssi < -81`) for perfect classification
2. **Random Forest:** All trees vote unanimously  
3. **XGBoost:** Finds perfect boundary immediately
4. **Neural Networks:** Learns trivial mapping in first epoch
5. **Adding noise:** The gaps are too large (14-60 units) for noise to bridge

---

## What Should Happen in Real 5G Networks

Real-world scenarios we're missing:

1. **Degrading Networks** (not just good/bad)
   - RSSI around -75 to -85 dBm (uncertain zone)
   - Latency 30-60ms (degrading but not failed)

2. **Partial Failures**
   - Good RSSI but high latency (routing issue)
   - Good throughput but high jitter (QoS issue)
   - Not all metrics degrade simultaneously

3. **Intermittent Issues**
   - Temporary congestion (looks faulty, recovers)
   - Measurement errors
   - Time-of-day variations

4. **Expected Realistic Accuracy**
   - Production 5G fault detection: **85-95% accuracy**
   - Precision: **80-90%**
   - Recall: **80-92%**

---

## Solution: Regenerate Data with Overlapping Ranges

I've prepared updated parameter ranges with realistic overlap:

### Proposed New Ranges:

```python
NETWORK_PARAMS = {
    'rssi': {
        'normal_range': (-80, -50),      # Extended down
        'faulty_range': (-110, -70),     # Extended up
        # OVERLAP: -80 to -70 (10 dBm uncertainty zone)
    },
    'sinr': {
        'normal_range': (10, 30),        # Lowered minimum
        'faulty_range': (-5, 15),        # Raised maximum
        # OVERLAP: 10 to 15 (5 dB uncertainty zone)
    },
    'throughput': {
        'normal_range': (50, 150),       # Lowered minimum
        'faulty_range': (10, 70),        # Raised maximum
        # OVERLAP: 50 to 70 (20 Mbps uncertainty zone)
    },
    'latency': {
        'normal_range': (5, 50),         # Raised maximum
        'faulty_range': (30, 200),       # Lowered minimum
        # OVERLAP: 30 to 50 (20 ms uncertainty zone)
    },
    'jitter': {
        'normal_range': (1, 15),         # Raised maximum
        'faulty_range': (10, 50),        # Lowered minimum
        # OVERLAP: 10 to 15 (5 ms uncertainty zone)
    },
    'packet_loss': {
        'normal_range': (0, 3),          # Raised maximum
        'faulty_range': (2, 20),         # Lowered minimum
        # OVERLAP: 2 to 3 (1% uncertainty zone)
    }
}
```

### Additional Improvements:

1. **Independent Degradation:** Not all features should fail simultaneously
2. **Increased Noise:** Add ±5-10% measurement error
3. **Remove/Modify Network Quality Score:** Currently too perfect (correlation: -0.98)
4. **Add Temporal Patterns:** Intermittent faults, recovery periods

---

## Action Plan

### For Data Engineer (Me):
- [ ] Modify `generate_synthetic_data.py` with overlapping ranges
- [ ] Add independent feature degradation logic
- [ ] Increase measurement noise
- [ ] Regenerate dataset (10,000 samples)
- [ ] Re-run EDA to verify realistic distributions
- [ ] Expected result: 10-20% overlap in boundary regions

### For ML Engineer (You):
- [ ] Wait for new dataset (ETA: 2-3 hours)
- [ ] Retrain models on new data
- [ ] Expected accuracy: **85-95%** (not 100%)
- [ ] If still getting >98%, notify me immediately

---

## Files Available

1. **`ISSUE_ANALYSIS_REPORT.md`** - Detailed technical analysis (5 pages)
2. **`scripts/analyze_data_separation.py`** - Analysis script you can run
3. **Analysis output** - Shows zero overlap for all features

---

## Next Steps

**Option A (Recommended):** I regenerate the data with fixes  
**Option B:** You provide feedback on proposed ranges first  
**Option C:** We discuss if we need multi-class labels (degrading state)

Please confirm which option you prefer, and I'll proceed immediately.

---

## Expected Timeline

- Data regeneration: 30 minutes
- EDA re-analysis: 1 hour  
- Updated handoff: 30 minutes
- **Total: 2-3 hours**

---

## Apologies

This is my fault - I should have validated the realistic difficulty of the classification task before handoff. The 100% accuracy is a red flag I should have caught during EDA.

The new dataset will be **challenging but solvable**, reflecting real-world 5G fault detection complexity.

---

**Ready to proceed with Option A?** Let me know and I'll start regenerating immediately.

Best regards,  
Data Engineer
