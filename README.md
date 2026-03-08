# Unified Energy Flow Model for Greenhouse Tomato Production

**A single energy balance equation that unifies greenhouse climate physics, mechanistic crop growth, and economic optimization — designed and validated on a 16,000 m² (4,900 pyeong) commercial tomato greenhouse in South Korea.**

> Author: **Park Sihong** — 11-year commercial tomato grower, M.S. candidate
> Greenhouse: ~16,000 m² glass greenhouse, Priva Connext 906, large-fruited tomato (Korea)
> Data: 6 years of operational records (2020–2025), 5-minute sensor intervals
> Status: Conceptual design complete (20 rounds of critical review) — implementation in progress
> Last revised: 2026-03-08

---

### Why This Exists

No commercial product today integrates a physics-based greenhouse energy model, a mechanistic crop growth model, model predictive control, and a farmer-facing interface into a single system. Climate computers (Priva, Hoogendoorn) use rule-based PID controllers. AI startups (Blue Radix, Koidra) use black-box ML. Academic models (Vanthoor, GreenLight) remain simulation tools. This document describes a unified framework that bridges all three — built by someone who operates the greenhouse, writes the code, and grows the tomatoes.

### How to Read This Document

- **§1–3**: Core insight and unified equation — the "what"
- **§4–5**: Scale-dependent abstraction and MPC framework — the "how"  
- **§6–7**: Plant-internal sink dynamics and growth phases — the biology
- **§8–11**: Differentiability, root zone, self-calibration, SaaS output — engineering details
- **§12–13**: Sensor defense, cold start, actuator safety — production robustness
- **§14**: Grower's empirical formulas ↔ theoretical model integration — the hybrid core
- **§15**: Implementation roadmap — what's built and what's next

---

## 1. Core Insight

A plant's maximum productivity is already determined by the amount of light it receives.
Excluding supplemental lighting, solar radiation energy (R_ext) is the sole external driving force.
Weather, greenhouse environment, and plant growth are a single process in which this energy flows through changing media.

- In the atmosphere → **heat**
- In the greenhouse → **heat + humidity**
- In the plant → **carbon (assimilates)** + **water vapor (latent heat)** (two diverging pathways)

Therefore, weather prediction models, greenhouse environment models, and plant physiology models
are not three separate models but can be unified into **a single energy balance equation**.

### The Three-Currency Conversion Chain

This unified model is ultimately a system that optimizes the conversion efficiency among three currencies:

```
Energy (J, physics)  →  Carbon (g, biology)  →  Money (₩, economics)
  Solar radiation         Photosynthesis/partitioning    Harvest/sales
  + Heating energy        - Respiration losses           - Energy costs
```

Each conversion stage has efficiencies and losses, and the unified model **optimizes total efficiency across the entire conversion chain**.

## 2. Energy Flow Structure (v3 — Substrate/Irrigation System + Dynamic Greenhouse Parameters)

```
External Inputs (Exogenous)
  │
  ├── W(t): External weather vector [R_ext, T_out, RH_out, Wind, Rain, ...]
  │         (R_ext is dominant, but each variable is an independent boundary condition)
  │
  └── U(t): Control vector [Heating, Ventilation(windward/leeward), Screen, CO2, Irrigation, Feed EC/pH ...]
             (Artificial energy + nutrient/water input — cost incurred)
             ※ Ventilation = [V_windward, V_leeward] independent control (single vent for mono-span)
       │
       ▼
  ┌─────────────────────────────────────────────────────┐
  │  Greenhouse Energy Balance   θ_gh(t) ← Dynamic self-calibration │
  │  W(t) + U(t) + θ_gh(t) → T_in, RH_in, CO2_in      │
  │         ▲                                            │
  │         │ Latent heat feedback (Transpiration → Humidity/Cooling) │
  │         │                                            │
  │  ┌──────┴────────────────────────────────────┐       │
  │  │         Plant Energy Partitioning           │       │
  │  │  R_int + T_in + VPD + CO2_in               │       │
  │  │     │                                      │       │
  │  │     ├─→ Photosynthesis pathway → Assimilates (carbon)  │
  │  │     │     └─→ Pull-Sink Partitioning        │       │
  │  │     │          ├─ Leaves (Source expansion)  │       │
  │  │     │          ├─ Fruit (Sink, harvest)      │       │
  │  │     │          └─ Roots/Stem (structural)    │       │
  │  │     │                                      │       │
  │  │     └─→ Transpiration pathway → Water vapor (latent heat) ──┘  │
  │  │           ▲ (Sensible→Latent conversion, plant=cooling pump)   │
  │  │           │                                        │
  │  │    ┌──────┴─────────────────────────────┐          │
  │  │    │  Root Zone State                    │          │
  │  │    │  ┌─────────────────────────────┐   │          │
  │  │    │  │ Irrigation system constraints: │   │          │
  │  │    │  │  Fertigation unit max capacity (L/hr) │   │  │
  │  │    │  │  Dripper capacity (L/hr/unit)  │   │          │
  │  │    │  └─────────────────────────────┘   │          │
  │  │    │                                    │          │
  │  │    │  Feed → [EC_in, pH_in, Volume]     │          │
  │  │    │    ↓                                │          │
  │  │    │  Substrate state: WC, EC_slab, T_slab │          │
  │  │    │    ↓      (← Slab Weight Scale)     │          │
  │  │    │  Drain → [EC_drain, pH_drain, Drain%] │          │
  │  │    │                                    │          │
  │  │    │  WC↓ → Transpiration resistance↑ → Transpiration suppressed │
  │  │    │  EC↑ → Osmotic pressure↑ → Water uptake inhibited │
  │  │    └────────────────────────────────────┘          │
  │  └────────────────────────────────────────────┘       │
  └───────────────────────────────────────────────────────┘
```

**v3 Key Changes:**
- Root Zone node added: WC/EC/pH/T_slab, irrigation system physical constraints
- θ_gh → θ_gh(t): Greenhouse properties converted to dynamic self-calibrating parameters
- U(t) now explicitly includes irrigation control (feed volume, EC, pH)
- Slab Weight Scale reflected as real-time sensor input

## 3. Unified Equation (v3 — Substrate & Dynamic Greenhouse)

```
Y = g(W(t), U(t), θ_gh(t), θ_crop(t), θ_rootzone, t_plant)
```

| Symbol | Meaning | Nature |
|--------|---------|--------|
| W(t) | External weather vector [R_ext, T_out, RH_out, Wind, ...] | External boundary conditions (multivariate) |
| U(t) | Control vector [Heating, Ventilation, Screen, CO2, **Irrigation, EC_in, pH_in**] | Optimization variables (cost incurred) |
| θ_gh(t) | Greenhouse properties (transmittance, ventilation coefficient, thermal capacity, etc.) | **Dynamic self-calibrating** parameters |
| θ_crop(t) | Crop parameters (photosynthetic efficiency, sink strength, etc.) | Periodically corrected via crop surveys |
| θ_rootzone | Substrate/irrigation system parameters (fertigation capacity, dripper capacity, etc.) | Equipment constraints (fixed) |
| t_plant | Planting date | Decision variable |
| Y | Total yield | Model output (prediction target) |

**Cumulative Changes:**
- v1→v2: R_ext(t) → W(t), U(t) added
- v2→v3: θ_gh → θ_gh(t) made dynamic, θ_rootzone added, irrigation control included in U(t)
- v3→v4: Objective function transition — Profit maximization → Target Tracking cost minimization (see §5)

## 4. Scale-Dependent Model Abstraction

> **Key insight**: The same unified equation applies, but the level of abstraction changes depending on the decision-making scale.

| Scale | Time range | Key variables | Decision | Model characteristics |
|-------|-----------|---------------|----------|----------------------|
| **Strategic** | Season (monthly) | R_ext climate normals, energy prices | Planting timing, variety selection | R_ext dominant, statistical |
| **Tactical** | Weekly–daily | Weather forecast W(t) | Weekly temperature strategy, energy budget | Forecast-based, MPC |
| **Operational** | Hourly–minute | Real-time sensor values | Real-time ventilation/heating/CO2 | Physics model + feedback control |

- At the strategic level, the original insight that "R_ext determines everything" holds
- At tactical/operational levels, the full W(t) and U(t) become increasingly important

#### Proactive Tactical Control

> If weather forecasts can predict conditions 3–5 days ahead,
> **proactive preparation becomes possible instead of reactive response.**

```
Proactive tactical scenarios:
  ├─ "Cold wave forecast in 3 days" → Start charging thermal buffer today (SOC↑)
  │                                   + Preserve Reserve Pool (lower night temp slightly)
  ├─ "Heat wave forecast tomorrow" → Increase irrigation tonight (secure substrate moisture)
  │                                  + Pre-deploy screen at dawn tomorrow
  ├─ "Continuous overcast this week" → Pre-lower Target (early Graceful Degradation)
  │                                   + Switch to energy-saving mode
  └─ "Clear skies returning in 3 days" → Suppress Reserve withdrawal (prepare for sunny-day recharge)
                                         + Reduce Target reduction magnitude

Key: Leverage MPC Prediction Horizon according to forecast reliability
  → 1 day: High reliability → Strong proactive control
  → 3 days: Medium reliability → Conservative preparation
  → 5+ days: Low reliability → Reference trend only
```

### Temperature Control Strategy: RTR (24-hour Running Temperature Reference) Centered

> In practice, DIF (day/night temperature difference) itself is not set as a target.
> The key is achieving **RTR (Running Temperature Reference, 24-hour average temperature)** at minimum cost through ventilation,
> with DIF being a byproduct of the RTR achievement process.

```
MPC temperature control primary variable:
  RTR = 24-hour moving average temperature (corresponds to T in Grower's formulas)

MPC optimization objective:
  Match RTR to target value while minimizing energy cost through ventilation
  → Daytime: Maximize use of solar energy (free heat source)
  → Nighttime: Maintain RTR with minimum heating
  → DIF is a byproduct of RTR achievement (not a separate control target)
```

- **Strategic planner**: Substituting T = RTR in Grower's formulas is OK (rough monthly calculation, Jensen error tolerable)
- **MPC controller**: Grower's formulas must use **actual hourly T** (substituting averages into nonlinear functions causes error)
  - Especially R_m (maintenance respiration, Q10 exponential) shows up to 25% difference between fluctuating vs constant temperature
  - RTR is a control **target**, not a calculation **input**
- Since RTR is primary, slightly higher daytime and slightly lower nighttime is OK as long as RTR is met
- Ventilation optimization is key: same RTR can have very different energy costs depending on ventilation strategy

### Spatial Assumption: Representative Zone-Based Control

> Large greenhouses (e.g., 4,900 pyeong / ~16,000 m²) have position-dependent temperature/humidity variations,
> but homogenizing with air circulators and controlling based on a **representative zone (center, crop height)** is practical.

- Model assumes **Lumped Parameter** (single representative value) — spatial distribution modeling is excessive
- Representative sensor placement + **fixed safety margin** (e.g., setpoint ±1°C) is practically sufficient
- Safety margin is reflected in MPC constraints: `T_target - margin ≤ T_in ≤ T_target + margin`

## 5. Reverse-Engineered Control → MPC Framework

Existing approach: Empirical setpoints → Observe results → Adjust (forward, rule-based)
Unified approach: **Model Predictive Control (MPC)** — objective-based reverse optimization

### MPC Integrated State Vector

```
X(t) = Greenhouse + Crop + Substrate integrated state

  Continuous states (high-frequency update — minutes to hours):
    T_in(t)           Indoor greenhouse temperature    [Sensor]
    RH_in(t)          Indoor greenhouse humidity        [Sensor]
    CO2_in(t)         Indoor greenhouse CO2             [Sensor]
    T_soil_fast(t)    Soil fast-node temperature        [Model estimate]
    T_soil_slow(t)    Soil slow-node temperature        [Model estimate]
    WC(t)             Substrate water content           [Slab Weight Scale]
    EC_slab(t)        Substrate EC                      [Sensor]
    SOC(t)            Thermal buffer remaining energy   [Optional equipment]

  Biological states (medium-frequency update — daily to weekly):
    X_reserve(t)      Assimilate Reserve Pool           [Model estimate, weekly correction]
    RAI(t)            Root Activity Index               [Inverse estimation, weekly correction]
    GDD*_i(t)         Light-corrected GDD per truss     [Model estimate]
    N_fruit_i         Fruit count per truss             [Crop survey + Task Confirmation]
    N_leaf(t)         Current leaf count                [Crop survey, set boundary condition]
    Stress_Duration(t) Cumulative stress days           [Model tracking]

  Discrete states (event-triggered update):
    Phase             {Establishment, Continuous, Terminated}
    N_truss           Active truss count                [Crop survey]
```

### MPC Structure

```
[Weather forecast W(t)] ──→ ┌──────────────────────────┐
                             │   Unified Energy Model     │
[Current state X(t)] ──→    │   (Physics+Plant simulation)│──→ Expected yield/cost
[Crop survey D(t)] ──→      │   + State estimation correction │
                             └──────────┬────────────────┘
                                        │
                             ┌──────────▼────────────────┐
                             │   Cost Function Optimization │
                             │   min: ResourceCost         │
                             │      + Target tracking error │
                             │      + V/G·VPD penalties     │
                             │      + Crack_Risk            │
                             └──────────┬────────────────┘
                                        │
                             ┌──────────▼────────────────┐
                             │   Optimal control trajectory U*(t) │
                             │   → Execute first step only  │
                             │   → Recompute at next step   │
                             └───────────────────────────┘
```

1. **Prediction Horizon**: Simulate future N hours/days based on weather forecast
2. **Cost Function**: See extended cost function below
3. **Optimization**: Inverse-solve for optimal control trajectory U*(t) that minimizes cost function
4. **Receding Horizon**: Rolling update with measured data (real-time correction)

### Cost Function Design: Scale-Separated (v4 — Key Transition)

> **Key decision**: Completely **remove Price** from the real-time control loop.
> Eliminate at source the instability caused by market price volatility destabilizing MPC.

#### Two-Stage Architecture

```
┌─────────────────────────────────────────────────────┐
│  Strategic Planner (once per season, or monthly readjust) │
│                                                      │
│  Input: Climate normals, average prices (KAMIS 5yr), variety traits │
│  Process: Optimize planting timing + align harvest peak with high-price season │
│  Output: ★ Target Yield Trajectory ★                │
│          + Season energy budget                      │
│                                                      │
│  ※ 'Carbon→Money' conversion is considered here only │
│  ※ Price uses same-period lower bound of climate normals │
└──────────────────────┬──────────────────────────────┘
                       │ Target Trajectory
                       ▼
┌─────────────────────────────────────────────────────┐
│  MPC Controller (hourly)                             │
│                                                      │
│  Input: Real-time sensors + Weather forecast + Target Trajectory │
│  Objective function (Target Tracking):               │
│                                                      │
│  J = Σ [ ResourceCost(t)                             │
│        + λ₁·AsymLoss(Yield(t), TargetYield(t))       │
│        + λ₂·VG_Penalty(t)                            │
│        + λ₃·VPD_Penalty(t)                           │
│        + λ₄·Crack_Risk(t) ]                          │
│                                                      │
│  Yield(t) = Daily assimilate allocation achievement rate │
│           = L_eff(t) / L_eff_target(t)               │
│  (Measurable proxy within MPC horizon)               │
│                                                      │
│  AsymLoss (asymmetric tracking):                     │
│    Yield < Target: λ_under · (Yield - Target)²       │
│    Yield ≥ Target: 0  (no penalty for overproduction) │
│                                                      │
│  VPD_Penalty = s_vpd²·λ_vpd_weight                   │
│    (Soft Constraint — Slack Variable s_vpd introduced) │
│                                                      │
│  Crack_Risk = (dWC/dt)² + (dT_in/dt)²               │
│    (Rapid substrate moisture/temperature change → cracking risk) │
│                                                      │
│  ResourceCost = Heating cost + Electricity cost       │
│    + CO2_cost × CO2_inject × η_CO2(Vent_opening)     │
│    (η_CO2: CO2 retention efficiency by vent opening, 0–1) │
│                                                      │
│  Output: Optimal control trajectory U*(t)            │
│                                                      │
│  ※ No Price variable — pure physics+biology optimization │
│  ※ Only 'Energy→Carbon' conversion efficiency matters  │
│  ※ Non-convex but much more tractable than v3 (profit maximization) │
└─────────────────────────────────────────────────────┘
```

#### Cost Function Comparison

| Item | v3 (Profit Maximization) | v4 (Target Tracking) |
|------|-------------------------|---------------------|
| Form | max(Yield×Price - Cost) | min(ResourceCost + λ·TrackingError² + Penalties) |
| Mathematical property | Nonlinear, non-convex | Asymmetric quadratic, **non-convex but tractable** |
| Global optimum | Not guaranteed | Not guaranteed but **local optima quality is good** |
| Price dependency | Required every step (unstable) | **Not required** (already reflected in strategy) |
| Stability | Control values fluctuate with price noise | **Stable** |

> **Analogy**: An experienced grower makes a cultivation plan at the start of the season,
> and in daily operations manages according to that plan — they don't change the temperature every day by watching market prices.
> The system follows this field wisdom exactly.

#### Definition of Yield(t): Daily Assimilate Allocation

> MPC horizon is limited by weather forecast accuracy (several days), but actual harvest occurs D_harvest (45–70 days) later.
> If the cost function's Yield(t) is defined as "final harvest yield," MPC cannot optimize it.

```
Yield(t) = L_eff(t) / L_eff_target_adj(t)

L_eff_target(t) = "Today's required net photosynthesis" inverse-calculated from Target Trajectory

Daily weather correction (operational-level Graceful Degradation):
  L_eff_target_adj(t) = L_eff_target(t) × min(1.0, R_ext_actual(t) / R_ext_expected(t))

  → Cloudy day: R_ext insufficient → Target auto-lowered → MPC doesn't fight unwinnable battles
  → Sunny day: R_ext sufficient → Target maintained → Normal tracking
  → Separate from weekly Graceful Degradation (tactical), daily-resolution operational correction

Key: R_int, the dominant variable of L_eff, is uncontrollable by MPC.
     Imposing AsymLoss when L_eff=0 on cloudy days makes MPC "fight a losing battle."
     Daily correction grants MPC "weather immunity."
```

#### Asymmetric Yield Tracking

> Symmetric quadratic penalty `(Yield - Target)²` **penalizes overproduction**.
> When weather is good and the plant produces above target without stress,
> MPC generates aberrant control by "closing screens to pull back to target" — blocking photosynthesis.
> There's no reason for AI to reject a free bonus from good weather.

```
AsymLoss(Yield, Target):
  if Yield < Target:
      Loss = λ_under × (Yield - Target)²    (Deficit: strong penalty)
  else:
      Loss = 0                               (Surplus: no penalty)

※ When overproduction causes V/G imbalance, VG_Penalty handles it separately
※ i.e., "producing more" is OK; "producing more while breaking balance" is caught by VG
```

#### VPD/Humidity Control (MPC Constraints + Penalties)

> The most difficult variable in greenhouse control. Strongly coupled with temperature and ventilation.
> VPD↑ → Excessive transpiration / stomatal closure, VPD↓ → Condensation / Botrytis.

```
Constraint hierarchy (Hard vs Soft — Infeasibility prevention):

  Hard Constraint (absolutely non-negotiable — physical limits):
    T_min_abs ≤ T_in(t) ≤ T_max_abs     (e.g., 5°C ≤ T ≤ 40°C)
    → Frost/heat damage is immediate and irreversible, must be defended unconditionally

  Soft Constraint (Slack Variable — violation allowed with penalty):
    VPD(t) = VPD_target + s_vpd(t)       (s_vpd = slack variable)
    VPD_Penalty(t) = λ_vpd · s_vpd(t)²

    → When VPD constraint conflicts with temperature constraint:
      "We want to maintain VPD, but temperature defense comes first.
       Accept VPD penalty and close vents to save temperature."

  Key to infeasibility prevention:
    Temperature(Hard) + VPD(Hard) → Winter night: can't satisfy both → solver error
    Temperature(Hard) + VPD(Soft) → VPD yields, solver always finds solution
```

- VPD must be explicitly in the cost function so MPC doesn't wreck humidity while chasing RTR
- Especially prevents winter night condensation (VPD↓) and summer daytime water stress (VPD↑)
- **Mathematical key**: Temperature=Hard, VPD=Soft separation is a prerequisite for MPC stability

#### V/G Balance Penalty (P_score-Based Asymmetric)

```
P_delta = P_score(t) - P_score_target

if P_delta < 0:   (Vigor decline — dangerous)
    VG_Penalty(t) = λ_down × P_delta²       (λ_down = large value, e.g., 3.0)
else:              (Vegetative overgrowth — less dangerous)
    VG_Penalty(t) = λ_up × P_delta²         (λ_up = small value, e.g., 1.0)
```

- **Asymmetry rationale**: Vigor decline (P_score↓) is far more damaging to long-term harvest than vegetative overgrowth (P_score↑)
- P_score = Comprehensive indicator already produced by Grower's formulas → No need for separate LAI/root weight estimation
- Cumulative stress triggers Stress_Amplifier weighting (details in §14.2)

#### ResourceCost: Integrated Energy + CO2 Cost (CO2-Ventilation Trade-off)

> If EnergyCost only covers heating/electricity, CO2 costs are missing.
> Liquid CO2 is a non-trivial cost, and enriching while vents are 30% open
> means most injected CO2 escapes within a minute — "throwing money out the exhaust."

```
ResourceCost(t) = HeatCost(t) + ElecCost(t) + CO2Cost(t)

CO2Cost(t) = CO2_price × CO2_inject(t) × (1 / η_CO2(t))

η_CO2(t) = CO2 retention efficiency (0 – 1)
         = f(Vent_opening, Wind_speed)
         ≈ max(0.1, 1.0 - α_vent × Vent_opening - β_wind × Wind)

  Vent 0%:  η ≈ 1.0 (sealed, nearly all CO2 retained)
  Vent 30%: η ≈ 0.3 (mostly lost)
  Vent 60%+: η ≈ 0.1 (virtually all lost)
```

```
MPC CO2 control strategy (η-based):
  η_CO2 > 0.5: CO2 enrichment permitted (cost-effective)
  η_CO2 ≤ 0.5: Stop CO2 enrichment or lower target concentration

  → MPC automatically decides "if vents are open, turn off CO2"
  → No explicit mutual exclusion rules needed; cost function naturally suppresses
```

#### Quality Metric (Crack_Risk)

> If the cost function only tracks "quantity (kg)," the quality dimension
> that determines actual tomato value (₩/kg) — fruit weight distribution, Brix, appearance — is **completely blank**.

```
Crack_Risk(t) = (dWC/dt)² + (dT_in/dt)²

  → Rapid substrate moisture change (overwatering) + rapid temperature change → fruit cracking risk
  → Included in cost function as λ₄·Crack_Risk → MPC restrains abrupt changes
```

- Phase:Establishment triggers cost function mode switch (yield tracking→V/G steering) → details in §7
- Phase:Terminated triggers cost function mode switch (yield→quality optimization) → details in §7
- Strategic planner's "Carbon→Money" conversion rate itself depends on quality → filling this completes the cycle

#### λ Weight Calibration Strategy

> The cost function has λ₁–λ₄ + λ_down/λ_up + γ_stress = 6+ hyperparameters.
> Wrong defaults distort MPC's "personality," causing aberrant control.

```
3-stage calibration:

  Stage 1: Simulation-based initial value search
    → MPC simulation with historical weather + measured data
    → Compare virtual season results for each λ combination
    → Select defaults from Pareto Frontier

  Stage 2: Farm profile presets (selectable in SaaS)
    ├─ "Energy Saver": λ₁↓ λ₂↑ (minimize cost, conservative management)
    ├─ "Yield Maximizer": λ₁↑ λ₂↓ (prioritize production even with more energy)
    └─ "Balanced":       λ₁=λ₂ (default recommendation)

  Stage 3: Operational feedback auto-readjustment
    → Track cumulative Target achievement rate during season
    → Achievement < 70%: "Recommend increasing λ₁" alert
    → Excessive energy cost: "Recommend decreasing λ₁" alert
    → Adjusted after grower approval (no automatic changes)
```

### Biological Time Lag

```
Fast Dynamics (min–hours):  U(t) → T_in change       ← Physics engine
Slow Dynamics (days–weeks): T_in → Assimilate accumulation ← Growth model
Harvest Delay (weeks–months): Assimilates → Harvestable   ← D_harvest function
```

- Physical control responds immediately, but yield changes appear **D_harvest (≈45–70 days)** later
- MPC Prediction Horizon ≥ D_harvest needed for meaningful optimization
- But 45–70 day weather forecasts are inaccurate → **Strategic/tactical scale separation** is essential
  - Strategic (season): Planting timing/energy budget optimization based on climate normals
  - Tactical (weekly): Temperature strategy adjustment based on weekly forecast
  - Operational (real-time): Feedback control based on same-day measurements

### Dynamic Target Relaxation (Graceful Degradation)

> When the Target Trajectory based on climate normals diverges significantly from reality,
> a safety mechanism to prevent energy waste + plant destruction from forcing impossible targets.

```
Tactical level (weekly automatic check):

  Measured cumulative radiation vs Strategic planner assumed climate-normal radiation
    │
    ├─ Ratio ≥ 0.8 → Maintain Target (normal)
    ├─ Ratio 0.5–0.8 → Proportionally reduce Target (Relaxation)
    └─ Ratio < 0.5 → Drastically reduce Target + energy saving mode
                      (Plant vigor preservation priority, P_score defense)
```

```
Target_adjusted(t) = Target_original(t) × Relaxation_Factor

Relaxation_Factor = min(1.0, R_actual_cumul / R_expected_cumul)
```

- When radiation is below climate normals, Target auto-lowers → MPC doesn't overexert
- Energy saving mode: Minimize heating, focus on maintaining plant vigor (survival > production)
- When radiation recovers, Target auto-restores (with Rate Limit on rapid increase)
- **Analogy**: Trying to maintain top speed in a storm breaks the ship. Slow down and survive.

### Data Assimilation + State Observer

Internal model states (LAI, sink strength, etc.) diverge from reality over time.
Solved with a **Predict-Update cycle**:

```
┌─────────────────────────────────────────────────┐
│  Predict (every hour, automatic)                 │
│  Previous crop survey θ_crop + subsequent environment data (GDD, radiation) │
│  → MPC internal model continuously estimates current crop state │
│  (No separate observer needed — MPC model itself is the propagator) │
└──────────────────────┬──────────────────────────┘
                       │
           Weekly crop survey arrives
                       │
┌──────────────────────▼──────────────────────────┐
│  Update (weekly, manual input)                   │
│  Correct θ_crop with innovation (measured - estimated) │
│   - Stem diameter → V/G balance correction       │
│   - Truss position → developmental stage sync    │
│   - Fruit count → sink strength total recalculation │
│  → Recompute MPC control trajectory              │
└─────────────────────────────────────────────────┘
```

- Model continuously tracks state with environment data, so values don't jump on survey day
- RealGarden already has crop survey data collection → natural connection
- Long-term: Camera vision (automatic LAI estimation) can shorten Update cycle

#### Observability & Parsimony Principle

> Precisely modeling unobservable states is pointless if there's no way to correct them — drift just accumulates.
> Model complexity must match observation frequency.

```
Model precision guide by observation frequency:

  High frequency (minute-level)  → Precise modeling OK
    T_in, RH, CO2_in, R_ext, WC (Slab Weight Scale), ventilation state
    → Physics engine can adequately correct

  Medium frequency (weekly)   → Moderate complexity
    Stem diameter → V/G balance, Truss position → developmental stage
    Fruit count → total sink demand, Leaf length/width → LAI estimation
    → Periodically correctable via Predict-Update

  Unobservable       → Risk of over-refinement
    Per-truss real-time sink strength, cell-level assimilate partitioning
    → Cannot correct → Simplification required (replace with Grower's formula averages)
```

- **Practical conclusion**: Gompertz per-truss sink maintained as conceptual framework,
  but implementation limits precision of unobservable states — anchored to Grower's W_fruit (average)
- When "current fruit diameter of enlarging truss" is observed in weekly survey, only that truss's sink is corrected

#### Biological Anomaly Alert

> Sensor faults (§12) are mechanical failures. What's caught here is **plant failure (pests/diseases)**.
> If environment is perfect but growth plummets → not model error, but the plant is sick.

```
At Update time:
  Innovation = |Measured - Predicted| / Predicted

  if Innovation > Threshold (e.g., 20%):
      → Not simple θ_crop correction but "biological anomaly alert" triggered
      → SaaS notification: "Environment normal but growth plummeting — pest scouting needed"
      → MPC: Switch to vigor preservation mode (similar to Graceful Degradation)
```

- Powdery mildew, Botrytis, nematodes → rapid decline in photosynthetic/transpiration efficiency
- Model doesn't directly diagnose pests, but can detect **"something is wrong" signal**
- Alarm role prompting expert judgment (diagnosis itself is human responsibility)

### Irrigation Integration
→ Details in §9 (Root Zone and Irrigation System)

## 6. Plant Internal: Sink-Driven (Pull) Dynamics

> **Paradigm shift**: From Source→Sink (supply-driven) to **Sink→Source (demand-driven)**.
> Not "leaves produce energy and distribute it to fruit," but
> **"when fruit pulls energy strongly, leaves adjust photosynthesis accordingly."**

### Pull-Sink Causal Structure

```
Fruit Sink Strength (GDD-based developmental stage)
  │
  │  ← Fruit load (truss count, fruit count) = Crop management decision
  │
  ▼
Total Sink Demand (sum of pulling force across all trusses)
  │
  ├─→ Signal to leaves (Source): Adjust photosynthetic efficiency (Demand-Driven)
  │     └─ Strong Sink → Photosynthesis UP-regulation
  │     └─ Weak Sink → Photosynthesis DOWN-regulation (feedback inhibition)
  │
  ├─→ Determine partitioning priority
  │     ├─ Priority 1: Maintenance respiration (survival, non-negotiable)
  │     ├─ Priority 2: Active Sinks (enlarging fruit, GDD-based)
  │     ├─ Priority 3: New leaves/stems (structural growth)
  │     └─ Priority 4: Roots (priority rises under stress)
  │
  └─→ Surplus supply: Stem thickening, excessive leaf area (vegetative overgrowth)
       Deficit supply: Flower abortion, fruit enlargement delay, malformed fruit
```

### Mathematical Quantification of Sink Strength (Gompertz-Based)

Each truss (i)'s pulling force is a function of developmental stage based on **light-corrected GDD**:

```
S_i(t) = S_max × f'(GDD*_i(t))

f(x) = Gompertz curve: A × exp(-b × exp(-c × x))
f'(x) = derivative → bell-shaped curve (maximum at peak enlargement phase)
```

#### GDD Light Correction (Light Sufficiency Factor)

> Pure temperature-based GDD causes misjudgment in low-light periods.
> When light is extremely insufficient, the plant slows its own developmental clock.

```
Original: GDD(t) = Σ max(0, T_avg - T_base)              ← Pure temperature

Corrected: GDD*(t) = Σ max(0, T_avg - T_base) × LSF(t)   ← Light-corrected

LSF(t) = Light Sufficiency Factor
       = min(1.0, R_int(t) / R_threshold)

R_threshold = Minimum daily required radiation (e.g., 4–5 MJ/m²/day)
```

| R_int relative | LSF | Effect |
|---------------|-----|--------|
| R_int ≥ R_threshold | 1.0 | Normal developmental rate |
| R_int < R_threshold | < 1.0 | Developmental clock deceleration (proportional) |
| R_int ≈ 0 (extreme low light) | ≈ 0 | Development nearly halted |

- Sum_J → A_total → S_max pathway corrects **assimilate quantity (how much)**
- LSF → GDD* pathway corrects **developmental clock speed (when)**
- Two pathways must be separated to accurately reflect low-light reality of "both less and slower"

- Gompertz derivative is bell-shaped → **Sink strength is maximum during peak fruit enlargement**
- Since fruiting is continuous, sum across all active trusses:

```
Total_Sink(t) = Σᵢ S_max × f'(GDD*_i(t))    (i = active trusses 1–N)
```

- Each truss at different GDD stages → sum varies complexly over time
- This mathematically explains "why energy management of continuously fruiting crops is difficult"

#### Natural Termination of Gompertz: Physiological End of Enlargement (Point A) vs Commercial Harvest (Point B)

> The Gompertz derivative f'(GDD*) is maximum at peak enlargement and has
> **already converged to 0** by the breaker stage. Once color change begins,
> cell expansion ends and the fruit effectively no longer draws energy (carbon) from the plant.

```
Two milestones in truss timeline:

  ──Flowering──── Enlargement (Sink active) ────→ Point A ─── Color change wait ──→ Point B
                                                   │                                │
                                   Physiological end of enlargement    Commercial physical harvest
                                   (Breaker stage)                     (Harvested by scissors)
                                   S_i(t) → 0 (natural convergence)   D_harvest(T) determines timing
                                   Auto-excluded from energy partition  Only affects yield tracking + task orders

Point A → Point B: Energy-Neutral Ripening Phase
  → Fruit hangs on vine but consumes no energy
  → Remaining energy naturally redistributed to upper active trusses

Seasonal A→B gap:
  Summer (high temp): D_harvest short → A→B is days (early harvest at 50% color)
  Winter (low temp): D_harvest long → A→B is weeks (wait for 80% color)
  → D_harvest(T) automatically reflects this seasonal difference as a variable timer
```

- **Key**: No need to manage color threshold (50%/80%) as a separate calibration variable
- Gompertz f'(x) naturally converges to 0 at color change onset → **No forced 10% reduction needed**
- D_harvest(T) automatically adjusts seasonal harvest timing (early summer / delayed winter)
- Role separation between energy partition (Gompertz) and harvest event (D_harvest) is structurally clean

#### Truss Elongation & Kinking

> The real problem is not fruit weight causing breakage, but **abnormal elongation of the truss itself (overgrowth)**
> causing kinking. Kinking damages vascular bundles, blocking assimilate transport
> → sharp drop in that truss's Sink Strength.

```
Cause: V/G imbalance — excessive temperature (RTR) relative to light (Sum_J)
  → Truss pulled toward vegetative growth, internodes elongate
  → Fruit weight + long internodes → vascular kink

Prevention — P_score already detects (no separate indicator needed):
  P_score = f(T, R_int) → Automatically detects low light + high temperature
  Stress_Duration Tracker cumulatively tracks → VG_Penalty weighted

  SaaS warning translation (P_score result → farmer-friendly language):
    if P_delta < 0 AND cause is "high RTR relative to insufficient R_int":
      → Alert: "V/G imbalance worsening — risk of truss elongation and kinking"
      → Recommendation: "Lower night temperature or dehumidify to induce generative growth"
    ※ Not new math, just Translation of existing P_score results

Post-incident handling (State Override — responding to physical damage):
  When a truss kinks despite prevention:
  Grower reports "Truss N kinked" in SaaS UI
    → Force that truss's S_max down to 10–20% level
    → MPC recalculates: redistributes remaining energy to other trusses
    → Complete severance (severe kink): S_max = 0 (same as Truss Skipping)
```

- Truss kinking is a physical symptom of P_score↓ → **No separate Truss_Elongation_Risk indicator needed** (Parsimony)
- P_score + Stress_Duration for preemptive defense, SaaS translates warnings to farmer language
- Post-kink handling uses State Override mechanism similar to Task Confirmation

### Assimilate Reserve Pool

> Current model is "paycheck to paycheck" — earn today, spend today.
> But real tomato plants store surplus carbon as **starch** in stems/leaves (Buffer).
> On cloudy days, they draw from this "carbon savings account" (Remobilization) to feed fruit.
> Without this buffer, one cloudy day causes MPC to misjudge fruit enlargement halt → overreaction.

```
State variable: X_reserve(t) = Assimilate reserve storage [g/m²]

Daily balance:
  Inflow:  R_in = max(0, A_total(t) - Total_Sink_Demand(t))  (Surplus → save)
  Outflow: R_out = max(0, Total_Sink_Demand(t) - A_total(t)) (Deficit → withdraw)
           Subject to: R_out ≤ X_reserve(t) × κ_max  (Max daily withdrawal rate, e.g., 20%)

  X_reserve(t+1) = X_reserve(t) + R_in - R_out

Constraints:
  0 ≤ X_reserve(t) ≤ Reserve_max    (variety-specific maximum storage capacity)
  Reserve_max ≈ 2–3 days of A_total  (requires empirical estimation)
```

- **MPC effect**: 1–2 cloudy days buffered by Reserve → AsymLoss activation delayed → overreaction prevented
- When Reserve depletes → real deficit triggers → AsymLoss + Graceful Degradation engage
- Reserve level is not directly observable → tracked as internal model state only (Parsimony principle)
- Stem diameter change (weekly crop survey) is an indirect indicator of Reserve (storage↑ → stem slightly thickens)

#### Nighttime Energy Flow: Reserve as Sole Supply

> At night R_int = 0 → A_total = 0. However, maintenance respiration (R_m) continues 24 hours,
> and fruit enlargement also proceeds at night (cellulose synthesis, water absorption).
> The only energy source during this period is the **Reserve Pool (starch remobilization)**.

```
Nighttime energy balance:
  Supply: R_out = X_reserve(t) × κ_night     (nighttime withdrawal rate, κ_night ≤ κ_max)
  Consumption: R_m(T_night) + Sink_Demand_night

  Lower R_m(T_night) → Less nighttime Reserve consumption → More Reserve remaining next morning
  → Lowering night temperature while maintaining RTR is favorable for Reserve preservation
  → MPC naturally optimizes toward "DIF expansion" (no separate DIF rule needed)

  Sink_Demand_night:
    Peak-enlargement fruit → Strong nighttime pull (water absorption + cell expansion)
    Mature fruit → Weak pull (mostly satisfied by daytime photosynthesis products)
```

- If Reserve Pool depletes overnight → sunny next day can't immediately buffer → **consecutive cloudy days' danger** mathematically explained
- MPC tries to minimize nighttime R_m (= Reserve depletion rate) within RTR constraints

### Assimilate Competition under Shortage

> In the Pull-Sink model, when Total_Sink > A_available, **not all sinks can be satisfied.**
> Plants have evolutionarily optimized priority rules,
> and the model must implement these as explicit algorithms.

```
Available supply:
  A_available(t) = A_total(t) + R_out(t)     (Today's photosynthesis + Reserve withdrawal)

Total demand:
  Total_Demand(t) = R_m_total(t) + Σᵢ S_i(t) + G_veg(t)
                    Maintenance resp.  Fruit sinks  Vegetative growth
                    ※ R_m_total = R_m × Age_factor × Canopy correction (see §14.1)

Supply ratio:
  Supply_Ratio = A_available(t) / Total_Demand(t)
```

#### Priority-Based Allocation

```
Supply_Ratio ≥ 1.0  (Sufficient supply — normal)
  ├─ Priority 1: R_m (maintenance respiration, 100% fulfilled)
  ├─ Priority 2: Σ S_i (fruit sinks, non-uniform GDD*-based allocation, 100%)
  ├─ Priority 3: G_veg (vegetative growth, 100%)
  └─ Surplus → X_reserve savings

Supply_Ratio 0.7–1.0  (Mild deficit — proportional reduction)
  ├─ Priority 1: R_m (100% — non-negotiable)
  ├─ Priority 2: Σ S_i (proportionally reduced, enlargement rate slows)
  │               → Peak-enlargement trusses prioritized, lower-priority trusses cut first
  ├─ Priority 3: G_veg (strongly suppressed)
  └─ Reserve withdrawal continues, no savings

Supply_Ratio 0.3–0.7  (Moderate deficit — selective sacrifice)
  ├─ Priority 1: R_m (100%)
  ├─ Priority 2: Enlarging fruit (mid–late stage, sunk cost protection)
  ├─ Priority 3: G_veg → 0 (vegetative growth completely halted)
  │
  ├─ ★ Flowering Abort:
  │     Weakest flowers in currently flowering truss shed first
  │     Abort_Priority = reverse developmental stage (latest-opening flowers first)
  │     N_fruit_flowering = N_fruit × (1 - Abort_Rate)
  │     Abort_Rate = f(Supply_Ratio):
  │       0.5–0.7: Abort_Rate ≈ 0.2–0.3 (some weak flowers shed)
  │       0.3–0.5: Abort_Rate ≈ 0.5–0.7 (most shed)
  │
  └─ ★ Late Truss Starvation:
        Nearly mature late trusses deprioritized in allocation
        → Enlargement halted (size fixed), only maturation proceeds (low energy consumption)

Supply_Ratio < 0.3  (Severe deficit — survival mode)
  ├─ Priority 1: R_m (barely maintaining respiration)
  ├─ Fruit: Minimum allocation to peak-enlargement trusses only
  │
  ├─ ★★ Truss Skipping:
  │     All flowers in flowering truss shed → N_fruit = 0
  │     Plant says "skip this truss, try next time"
  │     → Energy concentrated on existing enlarging fruit
  │
  └─ ★★ Expanded natural fruit drop:
        Even early-enlargement fruit begins shedding (sunk cost but survival first)
        → N_fruit adjusted downward across the board
```

#### Flowering Abort Algorithm Detail

> Individual flower survival probability is unobservable (§5 Parsimony principle).
> Handled with statistical Abort_Rate at truss level.

```
Target: Currently flowering truss (GDD*_i < GDD*_fruit_set)

Per-truss abort rate:
  N_fruit_after = round(N_fruit × (1 - Abort_Rate(Supply_Ratio)))

  Abort_Rate(Supply_Ratio) = max(0, (0.7 - Supply_Ratio) / 0.7) × Abort_max
    Supply_Ratio ≥ 0.7: Abort_Rate = 0 (no abortion)
    Supply_Ratio = 0.5: Abort_Rate ≈ 0.3 × Abort_max
    Supply_Ratio = 0.3: Abort_Rate ≈ 0.6 × Abort_max
    Supply_Ratio < 0.1: Abort_Rate → Abort_max (≈ 0.8–1.0)

  Abort_max: Variety-specific parameter (e.g., large-fruited: 0.8, cherry: 0.6)

  → N_fruit_flowering updated → Sink Demand recalculated → reflected in MPC

Biological basis (not modeled, structural explanation):
  Within the same truss, distal (tip) flowers shed first (opened later, nutrient access disadvantaged)
  Proximal (stem-side) flowers survive longest (opened first, nutrient advantage)
  → Abort_Rate is the truss-level statistical result of this "weakest flowers first" pattern
```

#### Truss Skipping: MPC Emergency Measure

> Truss Skipping is both a natural plant response and
> a preemptive control action that MPC can **proactively recommend**.

```
Truss Skipping decision criteria:
  Condition 1: Supply_Ratio < 0.3 sustained for 3+ days
  Condition 2: P_score < P_score_critical (severe vigor decline)
  Condition 3: Stress_Days > 14 (chronic stress)

  → When conditions met, MPC recommends "abandon current flowering truss (N_fruit=0)"
  → Purpose: Concentrate energy on existing enlarging fruit + vigor recovery
  → Effect: If vigor recovers in 2–3 weeks, normal fruit set on next truss

  Irreversible decision, so Confidence threshold applied (§11):
    Confidence ≥ 80%: "Strongly recommend truss abandonment"
    Confidence < 80%: "Decision deferred — reassess in 3 days"
```

### Natural Fruit/Flower Drop (Auto-Abortion) — Direct Environmental Damage

> N_fruit doesn't change only through manual thinning with scissors.
> Under extreme stress (high temperature, water deficit), the plant **spontaneously sheds flowers/fruit**.
> If the model fails to detect this, it allocates energy to already-shed fruit → State Divergence.

```
Auto-Abortion vs Supply Allocation role separation:
  ┌─────────────────────────────────────────────────────┐
  │  Auto-Abortion (this section)                        │
  │  = Direct environmental damage — flower/fruit shed from physical damage │
  │  Trigger: Extreme environmental conditions (occurs even with sufficient energy) │
  │                                                      │
  │  Supply Allocation (§6 shortage allocation)           │
  │  = Energy competition — starvation-induced abortion from light deficit │
  │  Trigger: Supply_Ratio < 0.7 (even with normal environment, when light is insufficient) │
  │                                                      │
  │  ※ Extreme low light (R_int < R_abort) handled by Supply Allocation │
  │    (Light deficit → Energy deficit → Starvation = a type of supply shortage) │
  └─────────────────────────────────────────────────────┘

Natural fruit drop detection (Stress-Induced State Override):

  Direct environmental damage triggers (OR):
    T_in > T_abort (e.g., 35°C for 4+ continuous hours)    ← Heat-induced abortion (pollen sterility)
    VPD > VPD_abort (e.g., 2.5 kPa for 6+ continuous hours) ← Water stress abortion
    Stress_Days > Stress_abort (e.g., 14 continuous days)    ← Chronic vigor decline

  When triggered:
    → Conservatively lower N_fruit of topmost flowering truss
       N_fruit_est = N_fruit × (1 - Abort_Rate)
       Abort_Rate ≈ 0.3–0.8 (proportional to stress intensity)

    → SaaS alert: "Natural flower drop estimated from extreme conditions — field verification needed"
    → MPC: Recalculate with lowered N_fruit_est (prevent excess energy allocation)
    → Force-corrected by actual fruit count at next crop survey

  Double-deduction prevention:
    Auto-Abortion triggered → Exclude that truss from Supply Allocation's abortion calculation
    Supply Allocation abortion triggered → Skip Auto-Abortion environmental trigger check
    → Two mechanisms applied mutually exclusively (whichever triggers first takes priority)
```

### Practical Implications
- **Fruit load management (thinning)** is not merely fruit size adjustment but
  **a control action that changes the entire plant's energy flow direction and photosynthetic efficiency**
- Theoretical basis for including **fruit count adjustment** in MPC control variable U(t)
- V/G balance = Quantified by P_score (Grower's comprehensive indicator) (see §5 VG_Penalty)

## 7. Growth Phase Switch (Phase State)

> Long-term tomato cultivation is not continuous dynamics.
> At topping, the energy partitioning logic completely flips.

### Phase State Variable

```
Phase ∈ { Establishment, Continuous, Terminated }

Determined by strategic planner:
  t_establishment_end ≈ 1st truss flowering (planting + ~6–8 weeks)
  t_topping = t_season_end - D_harvest(T_avg) - margin(2 weeks)
  → Top at the minimum point where last truss can complete harvest
```

### Phase: Establishment (Planting to 1st truss flowering, generative steering period)

> The period from planting until the first truss flowers is a **critical period** that determines the entire season.
> Balance between vegetative growth (leaves/stems) and generative growth (flower induction) must be struck.

```
Phase: Establishment special control

  Goal: Generative Steering
    → Induce normal differentiation/flowering of 1st truss while
    → Securing sufficient plant structure (stem diameter, initial LAI)

  Control strategy:
    1. Temperature management:
       → Immediately after planting: T slightly high (promote establishment, 18–20°C)
       → 1st truss induction period: Lower T slightly (generative stimulus, 16–18°C)

    2. Irrigation management:
       → Strengthen initial Dryback (promote root development)
       → Maintain slightly higher EC_slab (osmotic stress → promote generative transition)

    3. MPC cost function mode:
       → VG_Penalty weight↑ (vigor balance is top priority in this period)
       → AsymLoss weight↓ (no fruit yet, yield tracking meaningless)
       → P_score_target: Set slightly lower than Continuous period
         (suppress excessive vegetative growth, induce generative transition)

  Establishment → Continuous transition conditions:
    Condition 1: 1st truss flowering confirmed (crop survey) → Immediate transition
    Condition 2: Timeout — Auto-transition 25 days after planting
      (Korean commercial transplants: ~20 days to 1st truss flowering)
      (25 days = 20 days + 5 day safety margin)
    → Phase = Continuous
    → Cost function normal mode (yield tracking + V/G balance)
    → On timeout: "1st truss flowering unconfirmed — field check needed" alert
```

### Phase-Specific Energy Partitioning

| Category | Phase: Establishment | Phase: Continuous | Phase: Terminated (post-topping) |
|----------|---------------------|------------------|--------------------------------|
| New leaves/stems | **Top priority** (building plant structure) | Priority 3 | **0** (growing point removed) |
| N_truss | 0→1 (1st truss induction) | Continuously increasing | Fixed (no new trusses) |
| A_total allocation | Leaves + stems + roots (no fruit) | Fruit + leaves + stems + roots | **Extreme focus on fruit enlargement/maturation** |
| MPC objective | V/G balance + generative induction | RTR + V/G balance | Remaining fruit maturation optimization + energy savings |
| Irrigation strategy | Initial Dryback (root development) | Normal | Strengthened Dryback (improve Brix) |

### Phase:Terminated Cost Function Mode Switch

```
Phase: Continuous (normal growth period)
  → Yield tracking + V/G balance + VPD + Crack prevention

Phase: Terminated (post-topping)
  → Cost function mode switch:
    Yield tracking weight↓ (no new fruit coming)
    Quality optimization weight↑:
      + Strengthen Dryback → Improve Brix (sugar content)
      + Raise EC_slab → Increase fruit firmness
      + Maturation uniformity → Improve harvest efficiency
```

- In Phase:Terminated, the goal is not "how much" but "how good"

## 8. Differentiability and Control Discreteness

### Problem
- Physics equations are continuous, but control actuators are discrete (vent open/close, heating ON/OFF)
- Pure auto-differentiation for end-to-end optimization has gradient discontinuity at discrete boundaries

### Practical Solution: Hybrid Architecture

```
┌─────────────────────────┐     ┌─────────────────────┐
│  Physics+Plant Model     │     │  Control Decisions    │
│  (Continuous)            │     │  (Discrete)           │
│  - Energy balance        │     │  - MPC or RL          │
│  - Photosynthesis/       │ ←→ │  - Simulation-based   │
│    partitioning/transp.  │     │  - Injected as BCs    │
│  - Kept differentiable   │     │                       │
└─────────────────────────┘     └─────────────────────┘
```

- Keep physics model differentiable (for parameter calibration, model training)
- Separate control decisions to MPC/RL (naturally handles discreteness)
- Don't force everything into a single differentiable function

## 9. Root Zone and Irrigation System

> For the plant to act as a 'cooling pump,' it must draw up water.
> If substrate state is not modeled, transpiration stops and the energy balance collapses.

### 9.1 Irrigation System Physical Constraints (θ_rootzone)

```
Irrigation system constraints (fixed equipment values):
  Fertigation unit max capacity: Q_max [L/hr]      ← Upper limit per cycle
  Dripper capacity:              q_drip [L/hr/unit] ← Individual delivery rate limit
  Dripper count:                 n_drip [units/m²]  ← Density
  Max irrigation rate:           Q_max_actual = min(Q_max, q_drip × n_drip × Area)
```

### 9.2 Substrate State Variables

```
Substrate state X_root(t):
  WC(t)      ← Water content (real-time tracking via Slab Weight Scale)
  EC_slab(t) ← Substrate electrical conductivity
  T_slab(t)  ← Substrate temperature
  pH_slab(t) ← Substrate pH
```

### 9.3 Feed/Drain Monitoring

| Indicator | Meaning | Diagnosis |
|-----------|---------|-----------|
| EC_drain / EC_in ratio | Nutrient uptake rate | Ratio↑ = Poor uptake |
| Drain % | Over/under-feeding | 10–30% normal |
| WC daily pattern | Dryback strategy | Evaluate by nighttime WC drop |
| pH_drain variation | Root health | Rapid changes indicate problems |

### 9.4 Substrate ↔ Transpiration Feedback

```
WC↓ → Root water availability↓ → Stomatal Resistance↑ → Transpiration↓
EC_slab↑ → Osmotic resistance↑ → Water uptake↓ → Transpiration↓
T_slab↓ → Root activity↓ → Water uptake↓ → Transpiration↓
```

- Transpiration suppression → Latent heat decrease → Indoor T↑, humidity↓ → **Aboveground energy balance changes**
- Substrate state affects the entire greenhouse environment (feedback loop)
- In V/G balance control, irrigation (Dryback, EC targeting) is the most immediate tool

### 9.4-1 Root Activity Index (RAI)

> Just as P_score measures aboveground health, belowground needs a health indicator too.
> Even with perfect WC/EC, if **roots (root hairs) are damaged**, transpiration stops.

```
RAI(t) = Actual water uptake capacity of roots (0 – 1.0)

RAI estimation methods (inverse from observable data):

  Method 1: Drain-rate based
    Expected drain rate = f(irrigation volume, WC, transpiration prediction)
    Compare with measured drain rate
    → Drain higher than expected = Cannot absorb = RAI↓

  Method 2: Daily max transpiration inverse
    Expected transpiration = f(R_int, VPD, LAI, WC)   ← Model prediction
    Measured transpiration ≈ Inverse from indoor humidity increase
    → Measured/Expected ratio = RAI estimate

  Method 3: Slab Weight Scale daily pattern
    Normal roots: Rapid daytime WC drop (vigorous absorption)
    Damaged roots: Gradual daytime WC decrease (poor absorption)
    → Estimate RAI indirectly from WC daily amplitude

MPC utilization of RAI:
  Transpiration model:
    Transpiration_actual = Transpiration_potential × RAI(t)

  RAI < 0.7: "Root damage estimated" alert
    → Reduce irrigation (can't absorb anyway, overwetting worsens)
    → Strengthen shading screen (reduce transpiration burden)
    → Lower EC_in (reduce osmotic burden)
```

#### Two Pathways of RAI Decline: Substrate Environment vs Root Starvation (Cross-compartment Feedback)

> RAI decline is not only from substrate environment (high temperature).
> **Fruit stealing all the energy, starving the roots** is a more common cause.
> This is the true mechanism behind the "wilting on sunny day after cloudy spell" phenomenon.

```
Two pathways of RAI decline:

  Pathway 1: Direct substrate environment damage (existing)
    T_slab > 28°C cumulative → Fine root heat death → RAI↓
    Prolonged overwatering (WC↑↑) → Root oxygen deficiency → Root disease → RAI↓

  Pathway 2: Root starvation from fruit load (new — Cross-compartment Feedback)
    High fruit load (3–6 trusses at peak enlargement) + consecutive cloudy days
      → Supply_Ratio < 0.7
      → Assimilate supply to roots (Priority 4) completely cut off
      → Fine roots endure days without energy → Begin dying
      → RAI↓

    Days later when weather recovers:
      Substrate conditions (WC, EC) = Perfect
      But no roots → Can't absorb water → Can't transpire → Wilting
      → True identity of "wilting on sunny day after cloudy spell"

  Above-below cross-feedback loop:
    Aboveground Supply_Ratio↓ → Root starvation → RAI↓
      → Transpiration↓ → Cooling↓ → Greenhouse T↑ → R_m↑ → Supply_Ratio further↓
      → Vicious cycle (Positive Feedback Loop)

RAI decline model (both pathways integrated):
  RAI(t+1) = RAI(t) × (1 - Damage_heat - Damage_starvation)

  Damage_heat = f(T_slab cumulative excess hours)         ← Pathway 1
  Damage_starvation = f(Root_Supply_Deficit cumulative)    ← Pathway 2

  Root_Supply_Deficit(t):
    if Supply_Ratio < 0.7:
      Root_Supply_Deficit += (0.7 - Supply_Ratio) × dt
    else:
      Root_Supply_Deficit = max(0, Root_Supply_Deficit - Recovery_Rate × dt)
    ※ Root recovery is slower than damage (fine root regeneration takes days–weeks)
```

### 9.5 Irrigation Integration in MPC

```
MPC control variable U(t) extension:
  U_climate(t) = [T_target, Ventilation, Heating, CO2]
  U_irrig(t)   = [Irrigation volume, Irrigation interval, EC_in, pH_in]

  subject to:
    Irrigation ≤ Q_max_actual           (Equipment constraint)
    WC_min ≤ WC(t) ≤ WC_max            (Substrate moisture range)
    EC_drain ≤ EC_max                   (Prevent salt accumulation)
    Drain% ∈ [target_min, target_max]
```

### 9.6 Thermal Buffer Tank — Optional Equipment

```
Farms with thermal buffer:
  State variable added: SOC(t) = Thermal buffer remaining energy (State of Charge)
  Control variable added: U_buffer(t) = Charge/discharge rate

  MPC utilization:
    "Off-peak electricity (cheap) → Charge buffer (SOC↑)"
    "Pre-dawn minimum temperature → Discharge buffer (SOC↓) → Replace heating boiler"
    → Time-shifting of energy costs

  Constraints:
    0 ≤ SOC(t) ≤ SOC_max           (Capacity limit)
    |dSOC/dt| ≤ Rate_max            (Charge/discharge rate limit)

Farms without thermal buffer:
  → SOC term removed, no impact on MPC (same principle as CO2_factor=1.0)
```

## 10. Dynamic Self-Calibration of Greenhouse Properties

### Problem
θ_gh (cover transmittance, insulation, ventilation coefficient) changes over time:
- Dust/algae → Light transmittance decline
- Screen aging → Insulation degradation
- Storm damage → Air-tightness changes

MPC trusting "stale design-time parameters" causes cumulative control error.

### Solution: Parameter Tracking (Partially implemented in RealGarden)

```
Hybrid Physics-ML model's XGBoost residual correction
  = Effectively "ML automatically compensates for greenhouse parameter drift"

Explicit tracking:
  Expected dT = f(W(t), U(t), θ_gh_old)
  Measured dT = Sensor value
  Error = Measured - Expected
    → Online update of θ_gh parameters (running calibration)
    → Or XGBoost residual absorbs this error (current method)
```

## 11. SaaS Output: Task Instruction Layer

> Translate MPC's mathematical output into **task items** that field workers can immediately execute.
> No yield loss predictions or monetary displays — task display only.

### Translation Examples

| MPC Mathematical Output | SaaS UI Task Item |
|------------------------|-------------------|
| N_fruit(truss_6) = 3 | "Truss 6 thinning: keep 3 fruit" |
| T_target_night = 14.5°C | "Night setpoint temperature: 14.5°C" |
| EC_in = 3.2 | "Feed EC: adjust to 3.2" |
| Irrigation interval = 45min | "Irrigation interval: 45 min" |
| Vent opening = 30% | "Side vent opening: 30%" |

### Task Priority (Reflecting Labor Constraints)

```
Priority ranking (fixed rules):

  1. Safety/Emergency — Frost prevention, heat damage, sensor anomaly response
  2. Time-sensitive tasks — Thinning (must execute 3–5 days post-flowering), Harvest (quality degrades past timing)
  3. Environment adjustment — Temperature/humidity/CO2 setpoint changes (low labor, just change settings)
  4. Irrigation adjustment — EC/pH/interval changes
  5. Observation/recording — Crop surveys, pest scouting

※ Within same priority, tasks with greater P_score impact come first
```

### Task Confirmation Feedback

> MPC instructing "Truss 6: keep 3 fruit" doesn't instantly change plant state.
> Thinning/deleafing are **manual tasks that people must physically perform**.
> If a grower is busy and doesn't thin for 3 days, MPC assumes "only 3 left"
> and allocates tomorrow's energy accordingly, but the real plant still has 6 fruit consuming energy.
> → **Severe State Divergence**.

```
Manual task workflow:

  MPC output: "Truss 6 thinning → keep 3 fruit"
    │
    ▼
  SaaS UI: Display task card (Status: Pending)
    │
    ├─ After grower executes, taps "Done" ──→ Update State to N_fruit(6) = 3
    │                                         → Reflected in MPC next step
    │
    └─ Not completed (24 hours elapsed) ──→ Keep N_fruit(6) = original value
                                             → MPC recalculates (based on original fruit count)
                                             → Send reminder notification
```

- **Core principle**: Do not change internal θ_crop state until 'Done' confirmation
- Automatic control (ventilation/heating/CO2): Actuators execute immediately → No Confirmation needed
- Only manual tasks (thinning/harvesting/training) need Confirmation
- When weekly crop survey arrives, force-correct with measured values regardless of Confirmation

### Deleafing Management: Target Leaf Count Setting (Boundary Condition Approach)

> Deleafing unlike fruit thinning is a **continuous, routine** management activity.
> Requiring Task Confirmation each time only increases UX friction.

```
Grower setting (SaaS UI):
  ┌──────────────────────────────────┐
  │  Deleafing Settings               │
  │                                   │
  │  Planting–1st harvest: Maintain 18–20 leaves │
  │  1st–3rd harvest:     Maintain 15–17 leaves │
  │  After 3rd:           Maintain 13–15 leaves │
  │  After topping:       Maintain 12–14 leaves │
  │                                   │
  │  ※ Presets: Standard / Vigor-boost │
  └──────────────────────────────────┘

Model utilization:
  N_leaf(t) = Grower-set target leaf count (by period)
  → Used as input for LAI estimation, R_m calculation, transpiration model
  → Treated as steady-state constant, smooth integration possible
  → 1–2 day deleafing delay error auto-corrected in weekly crop survey
```

#### Canopy Light Extinction — Physical Basis for Deleafing Optimization

> A 2.5m+ tomato canopy is not a flat solar panel.
> Top 5–6 leaves receive 100% light, but mid/lower leaves receive almost none, shaded by upper leaves.
> "20 leaves so 20 leaves' worth of photosynthesis" is a **serious overestimate**.
> Lower leaves can't photosynthesize but consume maintenance respiration (R_m) — becoming **'parasitic leaves'**.

```
Canopy light distribution (Beer-Lambert attenuation):
  I(n) = R_int × exp(-k × LAI_above(n))

  I(n) = Light intensity at nth leaf height
  k = Extinction coefficient (tomato ≈ 0.7–0.8)
  LAI_above(n) = Cumulative leaf area index above nth leaf

Optimal leaf count = Point where sum of all leaves' A_net is maximum:
  N_leaf_optimal(R_int) ← Changes dynamically with current radiation
  High R_int (sunny): N_leaf_optimal ↑ (more leaves above light compensation point)
  Low R_int (cloudy): N_leaf_optimal ↓ (lower leaves become deficit)
```

```
MPC deleafing recommendation:
  if N_leaf_current > N_leaf_optimal(R_int_recent):
    → "At current radiation, N leaves are in energy deficit.
       Reducing to M leaves will actually increase A_total."

  ※ Canopy attenuation integrated into R_m_total (§14.1):
    R_m_total = R_m × Age_factor × (1 + ε × max(0, N_leaf - N_leaf_threshold))
    → When leaves exceed threshold (e.g., 15), R_m accelerates
    → L_eff decreases → MPC naturally steers toward deleafing
```

### Confidence-Based Recommendation for Irreversible Decisions

> Fruit thinning and topping cannot be undone. Temperature settings can be changed tomorrow,
> but cut fruit and growing points never come back.

```
Confidence attached to MPC output:

  Confidence = |J(N_fruit=3) - J(N_fruit=4)| / J(N_fruit=4)
             = Cost function difference ratio between optimal and second-best

  For thinning/topping recommendations:
    Confidence ≥ 80%: "Truss 6: keep 3 fruit (Strong recommendation)"
    Confidence 50–80%: "Truss 6: keep 3 fruit (Recommended, re-verify after next survey)"
    Confidence < 50%: "Decision deferred — reassess after next weekly survey"
                       → Irreversible decision automatically postponed

  Topping:
    → Most irreversible decision → Recommend only at Confidence 90%+
    → Double confirmation: Strategic planner plan + MPC confirmation + Grower final approval
```

- Reversible decisions (temperature, ventilation, CO2, irrigation): Execute regardless of Confidence
- Irreversible decisions (thinning, topping): Confidence threshold ensures conservatism
- **"When uncertain, choose the reversible option"** — Option value principle

## 12. Sensor Fault Defense (Anomaly Detection & Fallback)

> The assumption that sensor data is "always truth" inevitably breaks in the field.
> Contaminated data entering MPC causes disaster — block at data pipeline front end.

### Real-World Sensor Faults

| Fault Type | Example | MPC Misjudgment |
|-----------|---------|-----------------|
| Sensor contamination | Bird droppings/dust on radiation sensor | "Sun has set" → Full heating at midday |
| Physical interference | Foot/tool placed on slab scale | "Substrate overwatered" → Irrigation halted |
| Communication error | Value reads 0 or Null | All control logic collapses |
| Sensor drift | Old EC sensor out of calibration | Wrong EC baseline for irrigation |
| Out of range | Temperature sensor outputs -40°C | Heating runs infinitely |

### Defense Layer Structure

```
Raw sensor data
  │
  ▼
┌─────────────────────────────────────────┐
│  Layer 1: Range Check                    │
│  T_in ∈ [-10, 60]°C ?                   │
│  R_ext ∈ [0, 1500] W/m² ?               │
│  WC ∈ [0, 100]% ?                       │
│  → Out of range = Immediate rejection    │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Layer 2: Rate of Change Check           │
│  |ΔT/Δt| ≤ 5°C/10min ?                  │
│  |ΔR/Δt| ≤ 500 W/m²/1min ?              │
│  |ΔWC/Δt| ≤ 10%/5min ?                  │
│  → Sudden change = Suspect sensor error  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Layer 3: Cross Validation               │
│  R_ext > 0 but T_out plunging? → Abnormal│
│  WC spiking but no irrigation record? → Scale interference │
│  → Check physical consistency across sensors │
└──────────────────┬──────────────────────┘
                   │
                   ├─ Normal → Pass to MPC
                   │
                   └─ Anomaly detected:
                        ├─ Minor: Replace with recent N-minute moving average (Fallback)
                        ├─ Major: Freeze that sensor value + Send alert
                        └─ Critical (multiple sensors simultaneously):
                             → Switch to Safe Manual Mode + Emergency alert
                             → Halt MPC output, maintain minimum safe settings
```

### Safe Manual Mode

```
Safe setpoints (fixed by time of day):
  Daytime: T_target = 20°C, Ventilation = Auto (temp-linked), Irrigation = Last normal schedule
  Nighttime: T_target = 15°C, Ventilation = Minimum, Irrigation = Halted
  → Goal is "don't kill" (abandon optimization, survival first)
```

### Slow Sensor Drift Detection (Macro-Calibration)

> 3-layer validation catches "acute illness" (bird droppings, communication errors).
> But if a PAR sensor dome gradually yellows over 10 months from UV exposure?
> A daily 0.05% drop is indistinguishable from "normal seasonal variation."
> 6 months later, Sum_J calculated with values 10% below actual → MPC makes wrong decisions.
> **Boiling frog phenomenon — slow poison.**

```
Macro-Calibration:

  Frequency: Monthly (during tactical level review)
  Method: Compare farm sensor values vs external reference

  Reference data sources:
    ├─ KMA satellite regional radiation data (public)
    ├─ Nearby Smart Farm Innovation Valley reference sensors (institutional)
    └─ Cross-comparison among same-region RealGarden farms (SaaS network)

  Calibration logic:
    Drift = Farm R_ext monthly avg / Reference R_ext monthly avg

    if |1.0 - Drift| > 0.05 (5%+ bias):
        → "Radiation sensor calibration needed" alert
        → Apply software correction factor: R_ext_corrected = R_ext_raw / Drift
        → Recommend physical sensor cleaning/replacement (root cause fix)

    if |1.0 - Drift| > 0.15 (15%+):
        → "Severe sensor degradation" warning + Downgrade MPC input confidence
```

- Cross-comparison among SaaS network farms is a **unique SaaS strength** of RealGarden
- Individual farms lack reference points, but outlier farms auto-detected network-wide

## 13. Cold Start and Actuator Conflict Prevention

### 13.1 Cold Start Initialization Protocol

> On Day 1 when a farm first subscribes to RealGarden, internal model states (θ_crop, X_root) are empty.

#### Fast-Forward Initialization

```
Farm initial input (minimum required):
  ├─ Planting date (t_plant)
  ├─ Approximate current truss position (e.g., "6th truss flowering")
  ├─ One recent crop survey (stem diameter, leaf length, etc.)
  └─ Basic greenhouse info (area, cover material, fertigation specs)
       │
       ▼
Fast-Forward Simulation:
  ├─ Query KMA historical weather data (planting date → today)
  ├─ Calculate cumulative GDD, Sum_J with Grower's formulas
  ├─ Estimate truss position via D_truss → Cross-validate with input
  ├─ Initial correction of θ_crop with crop survey data
  └─ Output: θ_crop(0) initial values as of today
              │
              ▼
       MPC normal operation begins
```

### 13.2 Actuator Conflict and Hunting Prevention

#### Resolved via MPC Constraints

```
Mutually exclusive operating conditions (Hard Constraints):

1. Heating-ventilation conflict prevention:
   if Heating_valve > 0%:
       Ventilation ≤ V_min_humidity    (Only minimum humidity-discharge ventilation, e.g., 5%)

2. Deadband — Prevent frequent switching:
   |U(t) - U(t-1)| ≤ ΔU_max          (Control value change rate limit)
   if Vent movement < N min since last movement:
       Maintain vent position (motor protection)

3. Control priority:
   Temperature risk (frost/heat) > Humidity risk (condensation) > Energy optimization
   → Safety constraints always take precedence over economic optimization

4. Rate Constraint:
   dT_target/dt ≤ 2°C/hr             (Prevent sudden setpoint changes)
   dVent/dt ≤ 20%/10min              (Vent motor protection)

5. Windward/Leeward vent safety (Wind Safety):
   if Wind > Wind_gust_threshold (e.g., 10 m/s):
       V_windward ≤ V_wind_safe       (Limit windward max opening, e.g., 20%)
       V_leeward = Normal operation    (Leeward less affected by wind)
   → Prevents structural damage + direct crop harm from windward over-opening in high wind
   → MPC secures ventilation volume primarily through leeward vent
```

### 13.3 Sunrise Transition Phase

> The most dangerous moment of the day for plants is **the 2 hours around sunrise**.
> Stomata opening en masse + thermal screen opening → Cold air above screen cascades down (Cold Drop).
> Fruit surface condensation → Botrytis, fruit cracking.

```
Sunrise transition window:
  Start: 1 hour before sunrise (t_sunrise - 1h)
  End: 1 hour after sunrise (t_sunrise + 1h)

MPC objective switch during this window:
  Normal: "RTR tracking + Energy optimization"
    ↓
  Transition: "Condensation prevention (Dew Point defense) + Stomatal opening facilitation"

Specific control strategy:
  1. Pre-emptive heating (1h before sunrise):
     → Raise heating pipe temperature → Create rising air currents
     → Mitigate Cold Drop when screen opens

  2. Screen micro-control:
     → Open 1% at a time (no rapid opening)
     → Prevent cold air cascade

  3. Dew Point monitoring:
     → If T_surface ≤ T_dew detected → Pause screen opening
     → Resume after condensation risk clears

  4. Crack_Risk weighting:
     → Temporarily double Crack_Risk penalty λ₄ during this window
```

### 13.4 Sunset Transition Phase

> Condensation and Botrytis have another major culprit at **sunset**.
> While sunrise is a physical collision of 'cold air dropping down',
> sunset is a **thermodynamic inversion where plants cool faster than the air**.

```
Sunset transition physics:
  1. Sun sets → Greenhouse roof (cover) rapidly cools
  2. Plant tissue (especially fruit) loses radiation heat to cold sky/cover (Radiation Cooling)
  3. Fruit surface temperature < Air temperature (thermodynamic inversion)
  4. Indoor humidity 90%+ from daytime transpiration (near saturation)
  5. Humid air contacts cold fruit surface → Immediate condensation
  6. Condensation → Optimal conditions for Botrytis infection

Specific control strategy (sequential — compliant with §13.2 heating-vent conflict constraints):
  1. Pre-emptive humidity purging (1h before sunset, heating OFF):
     → Briefly open vents before closing screens
     → Purge daytime accumulated moisture
     → Pull humidity below 80% before closing vents

  2. Pre-emptive heating (after purging complete, vents closed):
     → Raise heating pipe temperature in advance
     → Maintain plant surface above Dew Point
     → Prevent surface temperature inversion from radiation cooling
     ※ Sequence: Purging → Close vents → Start heating (compliant with §13.2 mutual exclusion)

  3. Screen timing coordination:
     → Close screen after purging complete + vents closed
     → Closing screen too early traps moisture → Worsens condensation
     → Closing too late → Temperature drops too fast
```

## 14. Grower's Empirical Formulas (Park, S.H.) and Theoretical Model Hybrid Integration

> **Core principle**: Don't separate academic theoretical models (structure) from Grower's empirical formulas (calibration),
> but design a structure where Grower's formulas serve as **parameter determinants and constraints** for the theoretical model.

### 14.1 Grower's Formula System (Current → Dynamicized)

#### Current Formulas (Including Hardcoded Values)

```
D_harvest = 205.74 × exp(-0.065 × T)          ← Temperature → Days to harvest
D_truss   = 20.911 × exp(-0.054 × T)          ← Temperature → Truss interval days
R_int     = R_ext × τ (τ=0.8)                 ← External → Internal radiation
R_m       = 180 × 2.0^((T-16)/10)             ← Maintenance respiration (Q10)
L_eff     = max(0, R_int - R_m)               ← Net photosynthesis
Sum_J     = L_eff × D_harvest                  ← Cumulative radiation
W_fruit   = (Sum_J/7200) / (density×6×3) × 1000   ← Fruit weight [g]
P_score   = -0.1113×T² + (0.826×ln(R_int)-1.57)×T + 28.062×ln(R_int) - 150
```

**Problem 1**: `6` (trusses), `3` (fruit/truss) are hardcoded → Valid only for fixed structure
**Problem 2**: CO2 absent from photosynthesis formula → CO2 in U(t) but effect not reflected in model
**Problem 3**: All formulas function of T only → Independent of plant age (2-month = 8-month)

#### Dynamicized Formulas (v7 — CO2 efficiency + Hardcoding removed + Age correction)

```
Age_factor(t) = 1 + δ_age × months_since_planting
                δ_age ≈ 0.02–0.05 (2–5% efficiency decline per month, variety-specific)

R_m_total = R_m × Age_factor(t) × (1 + ε × max(0, N_leaf - N_leaf_threshold))
            ← Age correction (§14.1) + Canopy attenuation correction (§11) integrated
            ← When N_leaf ≤ N_leaf_threshold, canopy correction term = 0 (identical to original formula)
D_harvest_adj = D_harvest(T) × Age_factor(t)   ← Harvest delays with age
L_eff     = max(0, R_int - R_m_total) × CO2_factor   ← Age+Canopy+CO2 reflected
Sum_J     = L_eff × D_harvest_adj                    ← Cumulative (CO2 already in L_eff)
A_total   = (Sum_J / 7200) × 1000                   ← Total assimilates [g/m²]
W_fruit   = A_total / (density × N_truss × N_fruit) ← Fruit weight [g]
            density = Planting density [plants/m²] (determined at planting, large tomato: 2.5–3.5)

CO2_factor = CO2 enrichment efficiency coefficient
           = min(1.3, 1.0 + β × max(0, CO2_in - CO2_ambient))
           CO2_ambient ≈ 400ppm, β ≈ 0.001 (0.1% efficiency increase per 1ppm)
           No CO2 enrichment → CO2_factor = 1.0 (identical to original formula)
           1000ppm enrichment → CO2_factor ≈ 1.3 (30% photosynthesis increase, capped)
```

**Reverse direction (Pull-Sink core):**
```
N_truss × N_fruit = A_total / (density × W_fruit_target)
```

> **Mindset shift**: "Given 6 trusses × 3 fruit, what weight?" → **"Given target weight, what's the optimal truss × fruit count?"**

### 14.2 Grower's Formulas → Unified Model Mapping

#### A. S_max Dynamic Calibration (Using W_fruit)

```
S_max(t) = α × W_fruit(Sum_J(t), θ_crop)
```

- Grower's W_fruit determines **expected final fruit weight (at harvest, Point B)** → Sets upper bound for theoretical model's S_max
- α is a **comprehensive calibration constant** including dry-to-fresh weight conversion + breaker moisture variation

#### B. V/G Penalty Reference (Using P_score) — Asymmetric Design

→ Formulas and asymmetry rationale in §5 "V/G Balance Penalty" (Single Source)

#### Cumulative Stress Memory (Stress Duration Tracker)

```
Stress_Duration(t):
  if P_score(t) < P_score_target:
      Stress_Days += 1
  else:
      Stress_Days = max(0, Stress_Days - 0.5)   ← Recovery slower than stress accumulation

Stress_Amplifier = 1.0 + γ_stress × max(0, Stress_Days - 3)

Final penalty:
  VG_Penalty_final(t) = VG_Penalty(t) × Stress_Amplifier
```

| Stress_Days | Amplifier (γ=0.1) | Effect |
|------------|-------------------|--------|
| 0–3 days | 1.0 | Normal penalty |
| 7 days | 1.4 | 40% weight increase |
| 14 days | 2.1 | 2x+ weight |
| 21 days | 2.8 | Nearly 3x — MPC forces vigor recovery |

#### C. Time Lag Function (D_harvest Direct Use)

```
MPC internal:
  t_harvest(i) = t_flowering(i) + D_harvest(T_avg)
```

- No complex differential equation-based delay models needed
- D_harvest directly maps temperature→days to harvest → Used as MPC's Array Shift function

#### D. N_truss Dynamic Determination (Using D_truss)

```
Active truss count N_truss(t) = Cultivation period / D_truss(T_avg)
```

#### D-1. Grower/Gompertz Structural Error Detection (Milestone Validation)

```
Solution: Early verification based on intermediate checkpoints (Milestones)

  Milestone 1: Truss emergence interval
    Predicted: D_truss(T_avg)
    Measured: Truss position change in crop survey
    if |Predicted - Measured| > 2 days: "D_truss coefficient warning"

  Milestone 2: Flowering → Color change duration
    Predicted: Developmental stage estimated from GDD* accumulation
    Measured: Actual color change start date
    → Validates Gompertz parameters (b, c)

  Milestone 3: Flowering → Harvest duration (most important)
    Predicted: D_harvest(T_avg)
    Measured: Actual harvest date
    → Verifiable every truss after first harvest
```

- Key: **Verify D_truss early with weekly truss position observation** — don't wait 45 days
- Physics engine=XGBoost, Sensors=3-layer, Crop model=Milestone → **All 3 models have error detection**

#### E. Fruit Count (N_fruit) State/Control Distinction — Time-Series Targeting

```
Truss status classification:

  Trusses 1–3 (harvest imminent/enlargement complete) → State variable (unchangeable)
  Trusses 4–5 (enlarging)                              → State variable (sunk cost)
  Truss 6   (flowering/just-set)                       → ★ Control variable ★
  Trusses 7+ (not yet flowering)                        → Future plan (strategic level)
```

- MPC's "thinning recommendation" applies **only to currently flowering truss**
- N_fruit of already-enlarging trusses included in state vector X(t) (fixed)
- Without this distinction, MPC generates unrealistic instructions like "remove 2 fruit from truss 4"

### 14.3 Integrated Coupling Structure

```
Strategic Planner
  │
  ├─ Grower: D_harvest(T_normal) → Harvest timing prediction
  ├─ Grower: Sum_J(R_ext_normal) → Seasonal total assimilates
  ├─ Target: W_fruit_target (market-preferred fruit weight)
  │
  ├─ Reverse calc: N_truss × N_fruit = A_total / (density × W_fruit_target)
  │                → Seasonal optimal truss/fruit count plan
  │
  └─ Output: Target Yield Trajectory + Fruit load management plan
              │
              ▼
MPC Controller
  │
  ├─ Grower: D_harvest(T_measured) → Harvest delay function
  ├─ Grower: P_score(T, R_int) → V/G penalty reference
  ├─ Gompertz: S_i(GDD) × S_max(W_fruit) → Daily sink demand
  │
  ├─ Optimization: min[ ResourceCost + λ₁·AsymLoss(Yield,Target)
  │                     + λ₂·VG_Penalty + λ₃·VPD_Penalty + λ₄·Crack_Risk ]
  │
  └─ Output: U*(t) = [T_target, Ventilation, Heating, CO2]
             + Thinning recommendation (only for currently flowering topmost truss)
             ※ Already-enlarging trusses are State — not touched
```

### 14.4 Existing RealGarden Engine Mapping (Updated)

| Unified Model Component | RealGarden Current Implementation | Role in Unified Model | Status |
|------------------------|----------------------------------|----------------------|--------|
| Greenhouse energy balance | dT physics engine (greenhouse_dt_v73.py) | MPC internal physics simulator | Implemented |
| Physics+ML hybrid | Hybrid Physics-ML (residual_xgb.json) | MPC accuracy enhancement | Implemented |
| D_harvest | Grower's formula | MPC Time Lag function | **Direct use** |
| D_truss | Grower's formula | N_truss dynamic determination | **Direct use** |
| Sum_J / L_eff | Grower's formula | A_total calculation | **Direct use** |
| W_fruit | Grower's formula (dynamicized) | S_max calibration + Reverse fruit load planning | **Expansion needed** |
| P_score | Grower's formula | V/G penalty reference | **Direct use** |
| R_m (maintenance resp.) | Grower's formula (Q10) | Net assimilate calculation | **Direct use** |
| Weather input | KAMIS/KMA API | W(t) external weather vector | Implemented |
| Crop survey | Data collection feature | Data Assimilation Update | Implemented |
| Control vector U(t) | Vent/heating sensor collection ongoing | MPC output receiver | Partial |
| Substrate state (WC) | Slab Weight Scale | Root Zone State Observer input | **Sensor integration needed** |
| Feed/drain monitoring | EC/pH sensors | Irrigation MPC feedback | **Sensor integration needed** |
| Irrigation system | Fertigation/drippers | U(t) irrigation hard constraint | **Equipment parameter input** |
| θ_gh dynamic correction | Hybrid XGBoost residual correction | Greenhouse parameter drift compensation | Partially implemented |
| Sink partitioning (Gompertz) | Partially implemented | Beta-function sink model, full dynamic TODO | Partial |
| MPC optimization | **Implemented** | DE-based 24h MPC (`mpc_optimizer.py`, 1809 lines) | **Implemented** |
| Strategic planner | Not implemented | Target Trajectory generation | TODO |
| Task instruction layer | Partially implemented | TaskConfirmation DB model, conversion logic TODO | Partial |

### 14.5 Harvest Feedback Loop

> Harvest is the model's **final output (ground truth)**.
> Environment sensors (minute-level), crop surveys (weekly), Milestones (truss emergence) are all intermediate indicators.
> The moment fruit is actually picked and weighed is the most powerful validation data.

```
On harvest event (per truss):

  ├─ Actual fruit weight W_fruit_actual → Validate Grower's W_fruit
  │   if |predicted - actual| / predicted > 10%: Trigger Grower's coefficient correction
  │
  ├─ Actual harvest date → Validate D_harvest
  │   → Second most powerful data point after Milestone D-1 (truss emergence)
  │   → 20–30 harvest events per season → Sufficient statistical validation
  │
  ├─ Harvest yield (kg/m²/week) → Validate Target Trajectory tracking
  │   → Cumulative actual vs Target → Determine strategic planner replan trigger
  │
  └─ Quality grade (Grade 1 / Grade 2 / B-grade ratio)
      → Validate Phase:Terminated quality strategy effectiveness
      → Accumulate as next season's quality model training data
```

### 14.6 Cross-Season Learning Cycle

> Running one season (10 months) accumulates massive data.
> If this isn't reflected in the next season, each season starts nearly cold.
> **The model must improve with each season** — the essential value of SaaS.

```
Season Wrap-Up (automatic settlement at season end):

  ├─ Grower's coefficient settlement:
  │   Milestone + Harvest feedback accumulated → Farm-specific coefficients stored
  │   (e.g., "This farm's D_harvest is 8% faster than standard" → Store δ_farm)
  │
  ├─ λ profile settlement:
  │   Reverse-calculate optimal λ combination from season results
  │   (This farm showed better results with λ₁↑ → Reflect as next season default)
  │
  ├─ θ_gh trend settlement:
  │   Season start vs season end θ_gh change → Predict next season initial value
  │   (Light transmittance dropped 5% → "Cover cleaning or replacement recommended")
  │
  └─ Strategic planner validation:
      Target Trajectory vs Actual gap analysis → Improve next season Target generation

Next Season Kickoff (Warm Start):
  → Not cold start but "Warm Start" — initialized with previous season's learned results
  → 2nd-season farm starts with more accurate model than 1st-season
  → Shared season data among similar farms (same variety/region) → Network learning
```

## 15. Implementation Status (Implementation Roadmap)

> Conceptual design completed through 20 rounds of critical review.
> Below is the implementation progress. (12/64 completed, as of 2026-03-08)

### Completed (12)

- [x] Target Tracking MPC prototype — DE-based 24h MPC (`mpc_optimizer.py`, 1809 lines)
- [x] P_score-based asymmetric VG_Penalty real-time calculation — `ssot_cost.py:70-85`
- [x] Sensor anomaly detection pipeline (3-stage: range/rate/cross-validation) — `sensor_qc.py`
- [x] Actuator conflict prevention constraints in MPC — `mpc_optimizer.py:47-102`
- [x] CO2_factor β parameter (Michaelis-Menten model) — `ceo_formulas.py:1033-1063`
- [x] Phase State transition logic: establishment→continuous→terminated — `phase_service.py`
- [x] VPD penalty λ₃ parameter (vpd_target=0.8 kPa) — `ssot_cost.py:55-67`
- [x] Asymmetric cost function (AsymLoss) MPC solver compatibility — `ssot_cost.py:39-52`
- [x] Crack_Risk parameters: dWC/dt, dT/dt thresholds — `ssot_cost.py:88-110`
- [x] Phase:Terminated cost function switch (PHASE_LAMBDAS presets) — `ssot_cost.py:15-33`
- [x] Age_factor δ_age: Exponential decline model (τ_age=150 dap) — `ceo_formulas.py:3073-3092`
- [x] Phase:Establishment cost function mode: VG_Penalty↑, AsymLoss↓ — `ssot_cost.py:16`

### Remaining (52)

- [ ] W_fruit dynamicization (N_truss, N_fruit as time-varying variables + variety constraints)
- [ ] Strategic planner prototype (climate normals + prices + target weight → Target Trajectory + fruit load plan)
- [ ] Substrate state model: WC/EC dynamics + Slab Weight Scale integration
- [ ] Feed/drain monitoring dashboard (EC_in/out, pH, drain% tracking)
- [ ] Irrigation MPC: Optimize irrigation strategy under fertigation/dripper capacity constraints
- [ ] GDD* light correction (LSF) parameter R_threshold determination (empirical data-based)
- [ ] Graceful Degradation logic implementation (tactical level weekly review)
- [ ] Gompertz parameters (α, b, c) calibration (empirical fruit enlargement data)
- [ ] Transpiration-substrate moisture feedback loop quantitative modeling
- [ ] θ_gh online parameter estimation (from current XGBoost residual to explicit estimation)
- [ ] Task instruction layer UI prototype
- [ ] Sensor Fallback logic (moving average substitution, safe manual mode)
- [ ] Cold Start Fast-Forward initialization logic
- [ ] Stress Duration Tracker γ_stress parameter determination (empirical vigor recovery data)
- [ ] Biological anomaly detection Innovation Threshold determination (minimize false alarm rate)
- [ ] Task priority engine: Daily task scheduling under labor constraints
- [ ] Task completion confirmation UI + State update trigger
- [ ] Thermal buffer SOC model (optional module for equipped farms)
- [ ] Gompertz↔Grower's W_fruit consistency verification (summed fruit weight = Grower's formula average weight?)
- [ ] η_CO2(Vent_opening, Wind) function parameter determination (empirical CO2 retention data)
- [ ] λ calibration simulation environment (virtual season from historical weather+data)
- [ ] 3 farm profile presets (Energy Saver / Yield Maximizer / Balanced)
- [ ] D_truss Milestone validation logic: Crop survey → Grower's coefficient auto fine-tuning
- [ ] Sensor Macro-Calibration: KMA satellite API integration + SaaS network cross-comparison
- [ ] Slack Variable MPC solver (Hard/Soft constraint separation)
- [ ] Reserve Pool parameters: Reserve_max, κ_max determination (stem starch measurement or literature)
- [ ] Natural abortion: T_abort, R_abort, VPD_abort thresholds + Abort_Rate per variety
- [ ] RAI estimation logic: Drain%/transpiration inverse → Root vitality dynamic estimation pipeline
- [ ] Sunrise transition: Auto-calculate sunrise time + Screen micro-control sequence
- [ ] Deleafing boundary condition UI: Period-specific target leaf count + Presets (Standard/Vigor-boost)
- [ ] Irreversible decision Confidence calculation: Optimal-vs-suboptimal cost difference ratio → UI display
- [ ] Harvest feedback: Harvest recording UI + Grower formula auto-validation + Coefficient correction trigger
- [ ] Season Wrap-Up auto-settlement: Store Grower's coefficients/λ/θ_gh/Age_factor → Warm Start
- [ ] Integrated model validation scenario design with real data
- [ ] Supply shortage allocation: Supply_Ratio priority logic + Abort_Rate parameter
- [ ] Truss Skipping: MPC emergency recommendation logic + Confidence threshold linkage
- [ ] Generative Steering: Temperature/irrigation pattern → 1st truss induction optimization parameters
- [ ] Windward/leeward ventilation: η_vent(V_windward, V_leeward, Wind_dir) function modeling
- [ ] Windward safety constraint: Wind_gust_threshold, V_wind_safe parameter determination
- [ ] Nighttime Reserve withdrawal: κ_night parameter determination (nighttime enlargement empirical data)
- [ ] Proactive tactical control: Prediction Horizon weighting by forecast reliability
- [ ] Gompertz f'(GDD*) → 0 convergence verification: coloring onset vs Sink Strength empirical comparison
- [ ] Truss bending State Override: SaaS UI "truss bent" report → S_max downward adjustment
- [ ] P_score warning translation: V/G imbalance cause analysis → Farmer-friendly warning message mapping
- [ ] CO2_in smoothing: Moving average window size (1h vs 2h vs weekly cumulative)
- [ ] Supply_Ratio threshold: Variety-specific empirical abortion data → Inverse calibration pipeline
- [ ] Root_Supply_Deficit tracking: Root starvation accumulation → RAI Damage_starvation parameter
- [ ] Canopy Light Extinction: Extinction coefficient k, N_leaf_threshold, ε parameters
- [ ] N_leaf_optimal(R_int): Optimal leaf count from 7-day average R_int
- [ ] R_m canopy correction: Accelerated R_m increase coefficient ε when N_leaf > threshold
- [ ] Sunset transition: Sunset Phase control sequence (Purging → Pre-heating → Screen)
- [ ] Radiation cooling model: Fruit surface temperature vs dew point → Condensation risk assessment

---

## Design History Summary (Architecture Evolution)

> Refined through 20 rounds of critical review.
> Each round identified ~5 issues, resolved by consensus: adopt/reject/modify.

### Key Turning Points

| Round | Major Transition |
|-------|-----------------|
| R1-2 | Single R_ext → Multivariate W(t)+U(t) |
| R3 | Source→Sink paradigm → **Pull-Sink (demand-driven)** |
| R5 | Profit maximization → **Target Tracking** (Price-free MPC) |
| R6 | Grower hardcoded (6 trusses×3 fruit) → **Dynamic optimization variables** |
| R8 | Root Zone + Irrigation system integration |
| R12 | DIF → **RTR-centered** temperature control |
| R16 | Reserve Pool, Auto-Abortion, RAI, Sunrise Transition |
| R17 | CO2 double-counting bug fix + Supply shortage allocation algorithm |
| R18 | Point A/B separation, CO2 smoothing, η_CO2 2D **rejected** |
| R19 | Root starvation cross-feedback, Canopy extinction, Sunset Transition |
| R20 | R_m triple-definition unification, State Vector definition, density explicit |

### Rejected Proposals (Parsimony Principle Applied)

| Proposal | Rejection Reason | Alternative |
|----------|-----------------|-------------|
| η_CO2 as 2D function of windward/leeward (R18) | Over-engineering | CO2_in_smooth smoothing |
| Separate Truss_Elongation_Risk indicator (R19) | Overlaps with P_score | P_score warning translation |
| Detailed spatial non-uniformity modeling (R10) | Air circulators provide homogenization | Lumped Parameter + Safety margin |
| Individual flower Flower_Strength model (R18) | Unobservable | Truss-level Abort_Rate |
| DIF as separate control variable (R12) | Not used in practice | RTR-centered + DIF as byproduct |

### Design Principles

1. **Parsimony**: Model complexity ≤ Observation frequency. Simplify unobservable states (anchor to Grower's formula averages)
2. **Reuse existing math**: Prioritize Translation of existing Grower's formula results over new mathematics
3. **Field-first**: Practical validity in Korean field conditions trumps theoretical elegance
4. **Conservatism for irreversible decisions**: Apply Confidence threshold to thinning/topping recommendations
5. **Optional equipment harmlessness**: CO2_factor, SOC, etc. = 0 when not equipped (no impact on existing formulas)
