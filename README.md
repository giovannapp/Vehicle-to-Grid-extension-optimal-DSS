# V2G as DSS Project
### Based on: *Optimal Allocation of Dispersed Energy Storage Systems in Active Distribution Networks* (Nick, Cherkaoui & Paolone, 2014)

---

## Core Concept Shift

The original paper finds *where to install* DSSs and *how big* to make them, treating storage as a capital investment. This V2G extension flips the question:

> **"How much should we pay EV owners for grid services, and which EVs should we aggregate at which buses, to replace what dedicated DSSs would have done — at lower total cost?"**

This reframes the optimization from a **placement + sizing problem** into a **pricing + aggregation + scheduling problem**.

---

## Key Conceptual Differences from the Paper

| Aspect | Nick et al. | V2G Version |
|---|---|---|
| Storage ownership | DNO-owned DSSs | Customer-owned EVs |
| Investment cost | Capital (equipment) | Compensation to EV owners |
| Location control | Fully controlled | Probabilistic (where EVs park) |
| Availability | Always available | Stochastic (driving schedules) |
| Capacity | Fixed after install | Variable (battery degradation, SoC on arrival) |
| Reactive power | Modeled explicitly | Depends on charger type (AC vs DC) |
| Operator control | Direct dispatch | Aggregator contract model |

---

## Problem Formulation

### New Objective Function

Replace the DSS investment cost term with a **compensation cost** term:

**Original:** Fixed install cost + power rating cost + energy capacity cost + maintenance

**V2G version:** Compensation paid per kWh of service × energy dispatched + degradation penalty to owner + aggregator operational cost

So the new investment-equivalent term becomes:

$$C_{V2G} = \sum_{i} \sum_{t} \lambda_i^{comp} \cdot |P_{i,t}^{EV}| \cdot \Delta t + \sum_{i} C_i^{deg} \cdot |P_{i,t}^{EV}| \cdot \Delta t$$

Where:
- $\lambda_i^{comp}$ = compensation rate paid to EV owner at bus $i$ ($/kWh) — the main new decision variable
- $C_i^{deg}$ = battery degradation cost per kWh cycled (a parameter, not a variable)

The optimization also needs to find the **optimal compensation rate** that makes participation attractive enough to guarantee a target fleet availability, while minimizing total DNO cost.

### New Decision Variables

Beyond what the paper already has:

- $\lambda_i^{comp}$ — compensation rate per bus (or zone)
- $N_{i,t}^{av}$ — number of EVs available at bus $i$ at time $t$ (stochastic, becomes a scenario parameter)
- $E_{i,t}^{arr}$ — energy on arrival of EVs at bus $i$ (stochastic)
- $x_{i,t}^{EV} \in \{0,1\}$ — whether the aggregated EV fleet at bus $i$ is contracted for grid service at time $t$
- $P_{i,t}^{ch}$, $P_{i,t}^{dis}$ — charging and discharging power of the EV fleet at bus $i$

### Constraints That Change or Are Added

#### 1. EV Availability Constraint
Replaces DSS installation constraint:

$$S_{i,t}^{EV} = N_{i,t}^{av} \cdot \bar{E}_{EV} \cdot DoD_{max}$$

Where $\bar{E}_{EV}$ is average battery size and $DoD_{max}$ is max allowed depth of discharge for V2G use.

#### 2. State of Charge on Departure
Critical — owner must leave with enough charge:

$$E_{i,t_{dep}}^{EV} \geq E_{i}^{min,dep} \quad \forall i, t_{dep}$$

This guarantees owners always leave with at least (say) 80% SoC. This directly limits how much energy the grid can extract and is likely the most binding constraint in the entire model.

#### 3. Participation Willingness Constraint
New — economic rationality constraint:

$$\lambda_i^{comp} \geq C_i^{deg} + C_i^{opp}$$

Where $C_i^{opp}$ is the opportunity cost of not using that charge for driving (related to electricity retail price). Below this threshold, rational owners won't participate.

#### 4. Fleet Size Uncertainty
Replaces fixed DSS capacity:

$$P_{i,t}^{EV,max} = N_{i,t}^{av} \cdot P_{charger}^{max}$$

This is now stochastic — unlike DSS capacity which is deterministic once installed.

#### 5. Battery Degradation Model

$$C_i^{deg} = \frac{C_{battery}}{L_{cycle} \cdot 2 \cdot E_{EV}}$$

Where $L_{cycle}$ is cycle life of the battery. A typical Li-ion EV battery costs ~$150/kWh to replace with ~2000 full cycles, giving ~**$0.038/kWh degradation cost** — this becomes the floor for owner compensation.

---

## Scenarios and Stochasticity

This is where the model becomes significantly more complex than the original paper. Three new stochastic dimensions are added on top of load, PV, wind, and price.

### New Scenario Dimensions

#### 1. Driving/Parking Patterns
- Arrival time distribution (residential: peak arrivals 6–8pm; commercial: 8–9am)
- Departure time distribution
- Distance driven per day → determines SoC on arrival
- **Sources:** UK National Travel Survey, US National Household Travel Survey (NHTS)

#### 2. Fleet Penetration
- What % of loads at each bus own EVs?
- What % of those opt into V2G contracts?
- Treated as a sensitivity sweep parameter, not an optimization variable

#### 3. Seasonal Variation
- Winter: shorter range, more heating load on battery, EVs arrive with lower SoC
- Summer: longer trips, more predictable parking patterns, higher PV production

#### 4. Charger Type Available
- Level 2 AC (up to ~7.4 kW): can do V2G but limited reactive power support
- DC fast charger (50–150 kW): bidirectional capable, better for ancillary services
- Charger type determines whether reactive power capability (key in the original paper) can be replicated

### Scenario Matrix

Cluster across all of the following using K-means (analogous to the paper's 150 scenarios):

| Dimension | Paper | V2G Version |
|---|---|---|
| Load profiles | incl | incl |
| PV/wind production | incl | incl |
| Electricity price | incl | incl |
| EV availability profiles | not | new |
| EV arrival SoC distribution | not | new |

With K-means clustering, target 30–50 representative scenarios, but each is now higher-dimensional than the original.

---

## Edge Cases to Model

### EV-Specific Edge Cases

#### 1. The "Monday Morning Problem"
EVs used heavily over the weekend arrive Monday with very low SoC. The grid cannot extract much energy precisely when post-weekend demand may be high. The model needs a minimum SoC guarantee that holds regardless of grid needs.

#### 2. Cold Weather Battery Derating
At −10°C, usable battery capacity drops ~20–30%. This correlates with exactly the high-demand winter periods when the grid needs storage most. Perversely, when the grid needs EVs most, they are least capable. Must be captured in seasonal scenarios.

#### 3. Simultaneous Departure Events
If many EVs at a residential bus all depart at 7:30am, the grid must stop discharging and actually *charge* all of them within a narrow window. This can create demand spikes worse than without V2G. Model by constraining net power flow in the final hour before peak departure time.

#### 4. Emergency Override
Owner needs the car unexpectedly (medical, etc.). Model a fraction of the fleet as permanently "interruptible" — always able to leave regardless of grid dispatch. This reduces effective dispatchable capacity by ~10–15%.

#### 5. Charger Bidirectionality Penetration
Not all EVs have V2G-capable chargers today. Assume 30–40% of parked EVs are actually dispatchable as a conservative baseline, and sweep this as a sensitivity parameter.

#### 6. Battery Warranty Concerns
Many manufacturers void warranties if V2G cycling exceeds certain thresholds. This is a real behavioral barrier. Model as a hard cap on annual V2G cycles per vehicle (e.g., no more than 100 full cycles/year for V2G use).

### Network Edge Cases

#### 7. Spatial Mismatch
EVs tend to park at residential buses, but voltage/congestion problems may be at commercial or industrial buses. The V2G fleet might not be geographically co-located with where the grid needs support. This is potentially the most important finding of the study.

#### 8. Harmonic Distortion
Many bidirectional chargers inject harmonics into the grid. The SOCP model ignores this (as does the original paper), but it is a real constraint on how many chargers can share one feeder. Flag as a limitation.

#### 9. Charger as Reactive Power Source
Modern bidirectional chargers can inject/absorb reactive power independently of active power. This means EV chargers could be modeled as *always-available* reactive power sources even when not doing V2G active discharge — potentially a significant advantage over the original DSS model that changes the economics considerably.

---

## Data Required

### Network Data
- IEEE 34-bus feeder (same as paper — good starting point for comparability)
- Optional: a more realistic feeder with explicit residential/commercial/industrial bus classification

### EV Fleet Data
| Parameter | Value | Source |
|---|---|---|
| Battery sizes | 40 kWh (Leaf), 75 kWh (Model 3 SR), 100 kWh (Model 3 LR) | Manufacturer specs |
| Charger power | 3.7 kW (L1), 7.4 kW (L2), 50+ kW (DC) | IEC/SAE standards |
| Degradation cost | ~$0.038/kWh (Li-ion, 2000 cycles, $150/kWh replacement) | NREL battery datasets |
| Cycle life | 1500–3000 cycles | Battery research literature |

### Mobility Data
- Arrival/departure time distributions by bus type
- Daily mileage distributions → SoC on arrival
- **US:** [National Household Travel Survey (NHTS)](https://nhts.ornl.gov/) — freely available
- **EU:** National travel surveys or synthetic data from MATSim

### Grid & Price Data
- Same sources as the paper (or CAISO/ERCOT public data for a US case)
- Time-of-use tariffs — important for computing owner opportunity cost $C_i^{opp}$

### Compensation Benchmarking
Real-world V2G pilot programs to anchor the $\lambda^{comp}$ range:
- Nissan/Enel (Europe): ~$0.10–0.15/kWh
- OVO Energy (UK): variable by grid demand
- Pacific Gas & Electric (US): ongoing pilot
- These set realistic upper/lower bounds for your compensation sweep

---

## Programs and Tools

### Optimization
| Tool | Notes |
|---|---|
 Python + CVXPY + GUROBI/MOSEK | More modern, better data pipeline integration, handles MISOCP natively |

### Power Flow / Network Simulation
| Tool | Notes |
|---|---|
| OpenDSS | Free, widely used for distribution networks, Python interface via `opendssdirect.py`. Good for validating SOCP results |
| pandapower | Python library with built-in OPF, integrates well with pandas for scenario management |

### EV/Mobility Simulation
| Tool | Notes |
|---|---|
| SimPy | Python discrete-event simulation — good for modeling EV arrival/departure stochastics |
| NumPy sampling | Sample from empirical distributions directly — sufficient for most scenario generation |

### Scenario Generation & Clustering
| Tool | Notes |
|---|---|
| scikit-learn K-means | Direct replacement for the paper's MATLAB clustering |
| tslearn | Time-series specific clustering — better for multi-dimensional scenario matrix |

### Sensitivity Analysis & Visualization
| Tool | Notes |
|---|---|
| SALib | Python sensitivity analysis (Sobol indices, Morris method) — useful for sweeping compensation rates and penetration levels |
| Plotly / Dash | Interactive visualization of Pareto frontiers and scenario results |

---

## Project Structure

### Phase 1 — Baseline Replication
Reproduce the Nick et al. result on the IEEE 34-bus feeder. Validate that the SOCP implementation matches Table III of the paper (DSS at buses 22 and 34, with the reported sizes). This isolates any implementation errors before modifications begin.

### Phase 2 — EV Fleet Substitution
Replace DSS units with an EV fleet of equivalent total energy capacity. Keep everything else identical to the baseline. Measure how much performance degrades due to stochastic availability alone. This isolates the **"controllability penalty"** of using EVs vs dedicated storage.

### Phase 3 — Compensation Optimization
Add the compensation rate $\lambda^{comp}$ as a decision variable. Find the Pareto frontier between DNO cost and owner compensation. Key output: *at what compensation rate does V2G become preferable to dedicated DSSs from the DNO's perspective?*

### Phase 4 — Sensitivity Analysis
Sweep across the following parameters independently and in combination:

- EV penetration: 10%, 30%, 50%, 70% of buses
- Participation rate: 20%–80% of EV owners opt in
- Charger power level: 3.7 kW vs 7.4 kW vs 22 kW
- Battery size distribution (small vs large vehicle mix)
- Departure guarantee SoC: 70%, 80%, 90%

### Phase 5 — Hybrid Scenario
Model a mixed fleet: some dedicated DSSs at the most critical buses (where EV availability is low or unreliable), supplemented by V2G elsewhere. This is likely the most realistic and policy-relevant outcome, a transitional infrastructure model as EV penetration grows.

---

## Key Research Questions the Simulation Can Answer

1. **Break-even compensation rate:** At what $/kWh does V2G become cost-neutral for DNOs vs dedicated DSSs?

2. **Spatial mismatch quantification:** How much value is lost because EVs park at the wrong buses relative to where grid support is needed?

3. **Availability threshold:** What minimum fleet participation rate is needed to guarantee reliable grid support (e.g., 95% of hours)?

4. **Reactive power bonus:** How much additional value comes from using chargers as reactive power sources even when not performing active V2G discharge?

5. **Hybrid optimality:** What is the optimal split between dedicated DSSs and V2G contracts as EV penetration grows from 0% to 100%?

---

## Feasibility Assessment

### What Works Well
The SOCP framework from the paper extends cleanly to this formulation. The main additions, probabilistic availability and compensation pricing, are well-posed and can be handled within the same MISOCP structure. The research questions are genuinely open in the V2G literature.

### Biggest Challenges

**Departure SoC guarantee is extremely constraining.** Realistically, only ~20–30% of parked EV energy is usable for grid services at any given time without risking owner dissatisfaction. The simulation will likely show this as the most binding constraint.

**Spatial mismatch may be severe.** On the IEEE 34-bus feeder specifically, the buses with voltage problems (identified in the original paper) may not correspond to where residential EVs park. This could be a central finding rather than just a limitation.

**Reactive power capability depends on hardware assumptions.** If only basic L2 chargers are assumed, a significant portion of what made the DSS effective in the original paper is lost. This must be clearly stated as a scenario assumption.


---

## Summary of Additions to the Nick et al. Framework

```
Original MISOCP:
  minimize  [DSS investment cost] + [operational cost]
  subject to: DSS constraints, network SOCP, DG constraints

Extended V2G MISOCP:
  minimize  [compensation cost] + [degradation cost] + [operational cost]
  subject to: EV availability (stochastic),
              SoC on departure (new hard constraint),
              participation willingness (economic),
              charger power limits (stochastic),
              network SOCP (unchanged),
              DG constraints (unchanged)
  also optimize: compensation rate λ per bus/zone
```

The problem remains MISOCP-class and can be solved with the same GUROBI + YALMIP/CVXPY stack. Scenario generation becomes higher-dimensional but the K-means clustering approach scales directly.
