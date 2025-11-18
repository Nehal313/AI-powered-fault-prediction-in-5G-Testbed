# Issue Analysis Report: 100% Model Accuracy

**Date:** November 6, 2025  
**Reported By:** ML Engineer  
**Issue:** Achieving 100% accuracy even with noise and overlaps  
**Severity:** Critical - Indicates data leakage or perfect separation

---

## Problem Statement

The ML Engineer reported achieving **100% accuracy** on the fault prediction task, even after adding noise and creating overlaps in the data. This is a red flag indicating potential issues with the synthetic data generation process.

---

## Root Cause Analysis

### 1. **Data Generation Method - Perfect Linear Separation**

Upon reviewing `scripts/generate_synthetic_data.py`, the following critical issues were identified:

#### **Issue #1: Non-Overlapping Range Definitions**

The data generation uses **completely separate ranges** for Normal vs Faulty states:

```python
NETWORK_PARAMS = {
    'rssi': {
        'normal_range': (-70, -50),      # NO OVERLAP
        'faulty_range': (-110, -90),     # COMPLETELY SEPARATE
        'threshold': -85
    },
    'sinr': {
        'normal_range': (15, 30),        # NO OVERLAP
        'faulty_range': (-5, 5),         # COMPLETELY SEPARATE
        'threshold': 10
    },
    'throughput': {
        'normal_range': (80, 150),       # NO OVERLAP
        'faulty_range': (10, 40),        # COMPLETELY SEPARATE
        'threshold': 50
    },
    'latency': {
        'normal_range': (5, 20),         # NO OVERLAP
        'faulty_range': (80, 200),       # COMPLETELY SEPARATE
        'threshold': 50
    },
    'jitter': {
        'normal_range': (1, 5),          # NO OVERLAP
        'faulty_range': (20, 50),        # COMPLETELY SEPARATE
        'threshold': 15
    },
    'packet_loss': {
        'normal_range': (0, 1),          # NO OVERLAP
        'faulty_range': (5, 20),         # COMPLETELY SEPARATE
        'threshold': 3
    }
}
```

**Problem:** There is a **huge gap** between normal and faulty ranges:
- **RSSI:** Normal ends at -50, Faulty starts at -110 → **40 dBm gap**
- **SINR:** Normal starts at 15, Faulty ends at 5 → **10 dB gap**
- **Throughput:** Normal starts at 80, Faulty ends at 40 → **40 Mbps gap**
- **Latency:** Normal ends at 20, Faulty starts at 80 → **60 ms gap**
- **Jitter:** Normal ends at 5, Faulty starts at 20 → **15 ms gap**
- **Packet Loss:** Normal ends at 1%, Faulty starts at 5% → **4% gap**

#### **Issue #2: Deterministic Binary Classification**

The generation logic uses a simple binary approach:

```python
if is_faulty:
    rssi = np.random.uniform(*NETWORK_PARAMS['rssi']['faulty_range'])
    # ... all faulty ranges
else:
    rssi = np.random.uniform(*NETWORK_PARAMS['rssi']['normal_range'])
    # ... all normal ranges
```

**Problem:** Every feature is 100% correlated with the label. There's no ambiguity or real-world complexity.

#### **Issue #3: Network Quality Score - Perfect Discriminator**

The derived feature `network_quality_score` combines all the perfectly separated features:

```python
df['network_quality_score'] = (
    (df['rssi_dbm'] + 100) / 50 * 0.2 +
    df['sinr_db'] / 30 * 0.2 +
    df['throughput_mbps'] / 150 * 0.2 +
    (100 - df['latency_ms']) / 100 * 0.2 +
    (100 - df['packet_loss_percent']) / 100 * 0.2
)
```

**Result from EDA:** 
- Normal Mean: **0.838**
- Faulty Mean: **0.104**
- **Correlation with target: -0.98** ⚠️

This single feature alone can predict the fault status with near-perfect accuracy!

#### **Issue #4: Minimal Noise Addition**

The script only adds small Gaussian noise to Normal samples:

```python
# Add small random variations
rssi += np.random.normal(0, 2)       # Only ±2 dBm variation
sinr += np.random.normal(0, 1)       # Only ±1 dB variation
throughput += np.random.normal(0, 5) # Only ±5 Mbps variation
```

This noise is **insufficient** to create any meaningful overlap between classes.

---

## Impact on ML Models

### Why 100% Accuracy?

1. **Any feature alone** can classify with near-perfect accuracy due to non-overlapping ranges
2. **Network Quality Score** is essentially a direct encoding of the label
3. **No boundary cases** - No samples fall in the "uncertain" region
4. **Unrealistic data** - Real-world faults have gradual degradation, not binary states

### Model Behavior

- **Decision Trees:** Can create perfect splits (e.g., `if rssi < -85: Faulty`)
- **Random Forest:** All trees vote 100% correctly
- **XGBoost:** Immediately finds perfect decision boundaries
- **Neural Networks:** Learn trivial mapping in first epoch
- **Even with noise:** The gap is too large for noise to matter

---

## Real-World Implications

This synthetic dataset **does not reflect real 5G network behavior**:

### Realistic Scenarios Missing:

1. **Intermittent Faults**
   - Networks can have good RSSI but high latency
   - Can have good throughput but high jitter

2. **Degradation Zones**
   - RSSI between -85 and -75 (uncertain region)
   - Latency between 30-60ms (degrading but not failed)

3. **Multi-Factor Faults**
   - Some faults affect only specific metrics
   - CPU overload might not affect RSSI
   - High user load might only increase latency

4. **False Positives/Negatives**
   - Temporary network congestion (looks faulty but recovers)
   - Measurement errors
   - Partial failures

---

## Recommended Solutions

### Option 1: **Add Overlapping Ranges (RECOMMENDED)**

Modify the parameter ranges to include realistic overlap:

```python
NETWORK_PARAMS = {
    'rssi': {
        'normal_range': (-80, -50),      # Extended lower bound
        'faulty_range': (-110, -70),     # Extended upper bound → OVERLAP: -80 to -70
        'threshold': -75
    },
    'sinr': {
        'normal_range': (10, 30),        # Lowered minimum
        'faulty_range': (-5, 15),        # Raised maximum → OVERLAP: 10 to 15
        'threshold': 12
    },
    'throughput': {
        'normal_range': (50, 150),       # Lowered minimum
        'faulty_range': (10, 70),        # Raised maximum → OVERLAP: 50 to 70
        'threshold': 60
    },
    'latency': {
        'normal_range': (5, 50),         # Raised maximum
        'faulty_range': (30, 200),       # Lowered minimum → OVERLAP: 30 to 50
        'threshold': 40
    },
    'jitter': {
        'normal_range': (1, 15),         # Raised maximum
        'faulty_range': (10, 50),        # Lowered minimum → OVERLAP: 10 to 15
        'threshold': 12
    },
    'packet_loss': {
        'normal_range': (0, 3),          # Raised maximum
        'faulty_range': (2, 20),         # Lowered minimum → OVERLAP: 2 to 3
        'threshold': 2.5
    }
}
```

### Option 2: **Independent Feature Degradation**

Not all features should degrade simultaneously:

```python
def generate_realistic_fault_data():
    # Choose which metrics are degraded (1-4 metrics, not all 6)
    num_degraded = random.randint(1, 4)
    degraded_metrics = random.sample(METRICS_LIST, num_degraded)
    
    # Only degrade selected metrics, others remain normal
    for metric in METRICS_LIST:
        if metric in degraded_metrics:
            value = generate_faulty_value(metric)
        else:
            value = generate_normal_value(metric)
```

### Option 3: **Add Uncertainty Regions**

Create a "degrading" state between Normal and Faulty:

```python
fault_type = random.choice(['normal', 'degrading', 'faulty'])
# probabilities: 50% normal, 30% degrading, 20% faulty

if fault_type == 'degrading':
    # Mix of normal and faulty characteristics
    rssi = np.random.uniform(-85, -70)  # Boundary region
    latency = np.random.uniform(25, 55) # Boundary region
```

### Option 4: **Add Measurement Noise**

Real sensors have measurement errors:

```python
# Add realistic sensor noise (±5-10% of value)
rssi += np.random.normal(0, 3)      # Increased from 2
sinr += np.random.normal(0, 2)      # Increased from 1
throughput += np.random.normal(0, 10) # Increased from 5
latency += np.random.normal(0, 5)   # Add noise
jitter += np.random.normal(0, 2)    # Add noise
packet_loss += np.random.normal(0, 0.5) # Add noise
```

### Option 5: **Remove Network Quality Score**

This derived feature is too perfect. Either:
- Remove it entirely
- Make it less correlated by adding significant noise
- Let the ML model learn complex relationships instead

---

## Validation Metrics

After regenerating data, the ML Engineer should see:

- **Accuracy:** 85-95% (not 100%)
- **Precision:** 80-90%
- **Recall:** 80-92%
- **F1-Score:** 82-91%
- **ROC-AUC:** 0.90-0.96

These metrics indicate a **challenging but solvable** problem, which is realistic for production 5G fault detection.

---

## Action Items

### Immediate (Priority 1):
1. ✅ **Stop using current dataset** - It's not suitable for ML training
2. ⚠️ **Regenerate data with overlapping ranges** - Use Option 1 above
3. ⚠️ **Remove or modify network_quality_score** - Too perfect

### Short-term (Priority 2):
4. ⚠️ **Add independent feature degradation** - Use Option 2
5. ⚠️ **Increase measurement noise** - Use Option 4
6. ⚠️ **Add temporal patterns** - Intermittent faults, recovery patterns

### Long-term (Priority 3):
7. ⚠️ **Validate against real 5G testbed data** - If available
8. ⚠️ **Add multi-class faults** - CPU overload, network congestion, hardware failure
9. ⚠️ **Add seasonal/time-based patterns** - Peak hours, maintenance windows

---

## Conclusion

The **100% accuracy is a symptom of unrealistic data generation**, not a success. The current dataset has:

- ❌ No overlapping feature ranges
- ❌ Perfect correlation between features and labels
- ❌ Unrealistically large separation between classes
- ❌ Missing boundary/uncertain cases
- ❌ No partial or intermittent faults

**Recommendation:** Regenerate the dataset using overlapping ranges (Option 1) and independent feature degradation (Option 2) as minimum fixes.

---

**Next Steps:** 
1. Review and approve the proposed parameter changes
2. Regenerate synthetic dataset
3. Re-run EDA to verify realistic distributions
4. Retrain ML models and expect 85-95% accuracy
5. Share updated dataset with ML team

**Estimated Time:** 2-3 hours to implement fixes and regenerate data

---

*Report prepared by: Data Engineer*  
*Status: Awaiting approval for data regeneration*
