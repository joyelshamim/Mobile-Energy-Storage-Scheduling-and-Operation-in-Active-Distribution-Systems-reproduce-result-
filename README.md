# Mobile Energy Storage System (MESS) Scheduling and Operation in Active Distribution Networks
(Referance paper:Abdeltawab, Hussein & Mohamed, Yasser. (2017). Mobile Energy Storage Scheduling and Operation in Active Distribution Systems. IEEE Transactions on Industrial Electronics. 64. 6828-6840. 10.1109/TIE.2017.2682779.)

A Python implementation of optimal scheduling and operation of Mobile Energy Storage Systems (MESS) in radial distribution networks with renewable energy sources.

---

## Table of Contents

- [Overview](#overview)
- [System Scenario](#system-scenario)
- [Methodology](#methodology)
- [Mathematical Formulation](#mathematical-formulation)
- [Installation](#installation)
- [Usage](#usage)
- [Results](#results)
- [Project Structure](#project-structure)
- [References](#references)

---

## Overview

This project implements a two-stage optimization framework for scheduling and operating Mobile Energy Storage Systems (MESS) in active distribution networks. MESS consists of a battery energy storage system mounted on a truck that can relocate between different stations in the distribution network to provide energy arbitrage, loss reduction, and voltage support services.

### Key Features

- **Two-Stage Optimization**: Stage 1 (MISOCP-based scheduling) + Stage 2 (PSO-based fine-tuning)
- **DistFlow Power Flow Model**: Accurate power flow calculations using Branch Flow Model
- **Real-Time Profiles**: Dynamic load, renewable generation, and price profiles
- **Comprehensive Cost Modeling**: Grid costs, MESS operating costs, and RES feed-in tariffs

---

## System Scenario

### 3-Bus Radial Distribution Network

The system consists of a 3-bus radial distribution feeder with the following configuration:

```
                    ┌─────────────────┐
                    │   MAIN GRID     │
                    │   (Slack Bus)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │     BUS 1       │
                    │   Station 1     │
                    └────────┬────────┘
                             │ Branch 1 (16.87 km)
                             │ r = 0.01 pu, x = 0.02 pu
                    ┌────────▼────────┐
                    │     BUS 2       │───── ☀️ PV: 0.5 MW
                    │   Station 2     │───── 🏭 Load: 2.0 MW
                    └────────┬────────┘
                             │ Branch 2 (16.83 km)
                             │ r = 0.015 pu, x = 0.025 pu
                    ┌────────▼────────┐
                    │     BUS 3       │───── 🌬️ Wind: 1.5 MW
                    │   Station 3     │───── 🏭 Load: 1.5 MW
                    └─────────────────┘
```

### Bus Description

| Bus | Type | Components | MESS Station |
|-----|------|------------|--------------|
| Bus 1 | Slack | Grid Connection | Station 1 |
| Bus 2 | PQ | Load (2.0 MW) + PV (0.5 MW) | Station 2 |
| Bus 3 | PQ | Load (1.5 MW) + Wind (1.5 MW) | Station 3 |

### MESS Specifications

| Parameter | Value | Unit |
|-----------|-------|------|
| Rated Power (P̄) | 3.25 | MW |
| Rated Energy (Ē) | 6.381 | MWh |
| Charge Efficiency (η_ch) | 75% | - |
| Discharge Efficiency (η_dh) | 133% | - |
| SOC Range | [0.2, 0.8] | - |
| Max Daily Trips | 3 | trips |
| Max Daily Cycles | 1 | cycle |
| Average Speed | 40 | km/hr |
| Installation Time | 5 | minutes |

### Distance Matrix (km)

|  | Station 1 | Station 2 | Station 3 |
|--|-----------|-----------|-----------|
| **Station 1** | 0 | 16.87 | 19.92 |
| **Station 2** | 16.87 | 0 | 16.83 |
| **Station 3** | 19.92 | 16.83 | 0 |

---

## Methodology

### Two-Stage Optimization Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TWO-STAGE OPTIMIZATION                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      STAGE 1                                 │   │
│  │              Forward Optimization (MISOCP)                   │   │
│  │                                                              │   │
│  │  • Determine optimal station locations (Z_sk)                │   │
│  │  • Schedule charge/discharge periods                         │   │
│  │  • Binary decision variables for station assignment          │   │
│  │  • Considers: prices, SOC limits, trip limits                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      STAGE 2                                 │   │
│  │            Particle Swarm Optimization (PSO)                 │   │
│  │                                                              │   │
│  │  • Fine-tune continuous variables (P_ch, P_dh)               │   │
│  │  • Optimize power levels within scheduled periods            │   │
│  │  • Refine transit times (±M samples)                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Power Flow Model (DistFlow)

The power flow is calculated using the **Branch Flow Model (DistFlow)** equations:

```
Power Balance:       P_l + P_s - P_g - P_r = Σ(p_t - r_t·ℓ_t)
Voltage Drop:        v_b = v_a - 2(r·p + x·q) + (r² + x²)·ℓ
Current-Voltage:     ℓ · v = p² + q²
Power Losses:        P_loss = Σ r_t · ℓ_t
```

**Note**: Voltage variables are **squared** (v = V²) for convexification.

### Station Selection Strategy

**For Charging (Minimize Losses):**
1. If PV generation > 0.3 MW → Charge at Station 2 (use local PV)
2. Else if Wind - Load₃ > 0.3 MW → Charge at Station 3 (use local surplus)
3. Else → Charge at Station 1 (closest to grid)

**For Discharging (Maximize Loss Reduction):**
1. If Net Load at Bus 3 > Net Load at Bus 2 → Discharge at Station 3
2. Else → Discharge at Station 2
3. Default → Station 3 (end of feeder for maximum benefit)

### Cost Optimization

**Objective Function:**
```
max Profit = Income - C_grid - C_mess - C_res

Where:
  Income = Σ (SP_k × P_load_k × Ts)      [Selling energy to loads]
  C_grid = Σ (BP_k × P_grid_k × Ts)      [Grid purchase cost]
  C_mess = C_truck + C_ess               [MESS operating cost]
  C_res  = Σ (P_res_k × Ts × C_FIT)      [RES feed-in tariff]
```

**MESS Cost Components:**
```
C_truck = FC × distance + tlc × travel_time
C_ess   = E_charged × C_kwh
```

---

## Mathematical Formulation

### Decision Variables

| Variable | Type | Description |
|----------|------|-------------|
| Z_sk | Binary | 1 if MESS at station s at time k |
| P_ch_k | Continuous | Charge power at time k (MW) |
| P_dh_k | Continuous | Discharge power at time k (MW) |
| SOC_k | Continuous | State of charge at time k |

### Constraints

**SOC Dynamics (Eq. 37):**
```
SOC_{k+1} = SOC_k + Ts/E_bar × (η_ch × P_ch - P_dh/η_dh)
```

**SOC Limits:**
```
SOC_min ≤ SOC_k ≤ SOC_max
```

**Power Limits:**
```
0 ≤ P_ch_k ≤ P̄ × Z_sk
0 ≤ P_dh_k ≤ P̄ × Z_sk
```

**Station Assignment:**
```
Σ_s Z_sk = 1  ∀k  (MESS at exactly one station)
```

**Trip Limit:**
```
Σ_k |Z_sk - Z_s(k-1)| / 2 ≤ N_trips_max
```

**Voltage Limits:**
```
0.95² ≤ v_k ≤ 1.05²  (squared voltage)
```

---

## Installation

### Prerequisites

- Python 3.8 or higher
- pip package manager


### DistFlow Equations (Paper Eqs. 6-16)

| Equation | Description | Formula |
|----------|-------------|---------|
| Eq. 6 | Active Power Balance | P_g + P_r + Σ(p - r·ℓ) = P_l + P_s |
| Eq. 7 | Reactive Power Balance | Q_g + Σ(q - x·ℓ) = Q_l + Q_s |
| Eq. 8 | Branch Active Power | p_t = P_l + P_s - P_g - P_r + Σ(p + r·ℓ) |
| Eq. 10 | Voltage Drop | v_b = v_a - 2(rp + xq) + (r² + x²)ℓ |
| Eq. 11 | Current-Voltage | ℓ·v = p² + q² |
| Eq. 14 | Power Losses | P_loss = Σ r_t·ℓ_t |
| Eq. 16 | Voltage Limits | v_min ≤ v ≤ v_max |

### MESS Equations (Paper Eqs. 27-38)

| Equation | Description | Formula |
|----------|-------------|---------|
| Eq. 27 | Total MESS Cost | C_mess = C_truck + C_ess |
| Eq. 28 | Truck Cost | C_truck = FC·N·D + tlc·T |
| Eq. 29 | ESS Cost | C_ess = Σ P_ch·Ts·C_kwh |
| Eq. 37 | SOC Dynamics | SOC_{k+1} = SOC_k + Ts/E·(η·P_ch - P_dh/η) |
| Eq. 38 | Cycle Dynamics | N_{k+1} = N_k + Ts/(2E)·\|P_ch + P_dh\| |

---

## References

1. H. H. Abdeltawab and Y. A.-R. I. Mohamed, "Mobile Energy Storage Scheduling and Operation in Active Distribution Systems," *IEEE Transactions on Industrial Electronics*, vol. 64, no. 9, pp. 6828-6840, Sept. 2017.
