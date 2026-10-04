# Winter Optimization Guide for Anker MAX AC Control

## Overview

The enhanced control logic adapts your battery management strategy between summer (high PV, minimal grid import) and winter (low PV, mandatory grid import at night). This guide explains the winter-specific features and how to configure them.

## Winter Operation Strategy

### The Problem
- **Winter PV generation** is 60-80% lower than summer
- **Nighttime grid electricity must be purchased** since daytime PV cannot cover night consumption
- **Peak tariffs** are typically highest in evening hours (17:00-21:00)
- **Goal**: Minimize grid import costs by shifting consumption to off-peak hours and maximizing battery discharge during peak tariffs

### The Solution: Time-Based Discharge Management

```
WINTER SEASON (Nov-Feb):
┌─────────────────────────────────────────────────────────┐
│ 00:00-05:00 (CHEAP NIGHT ELECTRICITY)                   │
│ ✓ Charge both batteries from grid at cheapest rate      │
│ ✓ If LG SOC < 35% and Anker < 80%                       │
│ ✓ Pre-load batteries before expensive daytime           │
│ ✓ Typical tariff: €0.20-0.25/kWh (lowest of day)       │
└─────────────────────────────────────────────────────────┘
        ↓ 05:00
┌─────────────────────────────────────────────────────────┐
│ 06:00-17:00 (DAYTIME)                                   │
│ ✓ Charge aggressively when LG > 80% (lower threshold)   │
│ ✓ Fill both batteries with any available PV surplus      │
│ ✓ Prepare for evening peak tariff                        │
│ ✓ Switch to PV charging as soon as sun rises            │
└─────────────────────────────────────────────────────────┘
        ↓ 17:00
┌─────────────────────────────────────────────────────────┐
│ 17:00-21:00 (PEAK TARIFF HOURS)                         │
│ ✓ Discharge both batteries to offset peak consumption    │
│ ✓ Minimize grid import (most expensive: €0.45-0.50)     │
│ ✓ Maintain minimum 20% SOC reserve                       │
│ ✓ Typical tariff: €0.45-0.50/kWh (highest of day)      │
└─────────────────────────────────────────────────────────┘
        ↓ 21:00
┌─────────────────────────────────────────────────────────┐
│ 21:00-00:00 (NORMAL/OFF-PEAK HOURS)                     │
│ ✓ Discharge slowly to cover evening loads               │
│ ✓ Or continue if still in peak tariff period            │
│ ✓ Idle if batteries sufficient for night                │
└─────────────────────────────────────────────────────────┘
```

## Configuration Parameters

### 1. Time Windows (adjustable in Home Assistant UI)

**Winter Peak Hours Start** (input_number.anker_winter_peak_hour_start)
- Default: 17 (5 PM)
- Set to your local peak tariff start hour
- Example: If peak starts at 4 PM, set to 16

**Winter Peak Hours End** (input_number.anker_winter_peak_hour_end)
- Default: 21 (9 PM)
- Set to your local peak tariff end hour
- Example: If peak ends at 8 PM, set to 20

### 2. Battery Reserve Settings

**Winter Minimum SOC Reserve** (input_number.anker_winter_min_soc_reserve)
- Default: 20%
- Prevents complete discharge at night
- Ensures morning startup capability
- Adjust based on:
  - Typical night loads: Increase reserve if night consumption is high
  - Available battery capacity: Can be lower if total capacity (LG + Anker) is large
  - Minimum rule: Never go below 10% to protect battery health

**Winter LG Full Charge Threshold** (input_number.anker_winter_lg_full_threshold)
- Default: 80%
- LG must reach this SOC before Anker charging activates
- Lower value (80%) vs summer (95%) = more aggressive charging
- Why: In winter with low PV, every charging opportunity counts

### 3. Static Parameters (in anker_max_control.yaml)

These are hardcoded but can be edited if needed:

```yaml
lg_empty_soc_winter: 35        # Start discharging from LG at night (if > deficit)
coverage_ratio: 0.9            # Only use 90% of surplus/deficit (prevent overshooting)
max_charge_power: 800W         # Anker charging limit
max_discharge_power: 800W      # Anker discharge limit
min_soc_reserve: 20            # Absolute minimum reserve (overridden by input number)
```

## Operating Modes (Priority Order)

### Mode 1: Winter Off-Peak Night Charging (00:00-05:00 - Highest Priority)
**Objective**: Charge from cheapest night-rate electricity

**Trigger Conditions**:
- Current season is winter
- Current time is cheap night (00:00-05:00)
- LG SOC < 35%
- Anker SOC < 80%

**Action**: Charge Anker and LG from grid at night-rate tariff

**Benefit**: Arbitrage between cheap night (€0.20-0.25/kWh) and expensive peak (€0.45-0.50/kWh)

**Example**: 
- Cost to charge 6.4 kWh @ €0.25 = €1.60
- Value when discharged during peak @ €0.50 = €3.20
- Net savings: €1.60 per night = €48/month

---

### Mode 2: Winter Daytime (06:00-17:00)
**Objective**: Maximize battery charging

**Trigger Conditions**:
- Current season is winter (Nov-Feb)
- Current time is daytime (06:00-17:00)
- LG SOC ≥ 80% (winter threshold, lower than summer's 95%)
- Available PV surplus > 0W
- Anker SOC < 95%

**Action**: Charge Anker to balance excess PV production

**Benefit**: Pre-charges Anker for evening peak hours, reduces daytime grid import

---

### Mode 2: Winter Peak Hours (17:00-21:00)
**Objective**: Minimize grid import during expensive hours

**Trigger Conditions**:
- Current season is winter
- Current time is within peak tariff hours
- Grid import needed (deficit) > 0W
- Anker SOC > minimum reserve (20%)

**Action**: Aggressively discharge Anker and LG Chem to meet household demand

**Benefit**: Replaces expensive grid electricity with stored solar energy

**Example Scenario**:
```
Time: 19:00 (within peak hours 17:00-21:00)
Anker SOC: 85%
LG Chem SOC: 45%
Household load: 800W
Grid normally imports: 800W (@ €0.45/kWh = €0.36/hour)

WITH OPTIMIZATION:
→ Anker discharges 400W (discharge target: min(deficit=800W × 0.9, max=800W))
→ LG Chem discharges 400W (via SolarEdge inverter logic)
→ Grid import reduced to: 0W
→ Cost saved: €0.36/hour × 4 hours peak = €1.44/day
→ Monthly savings: ~€40 during winter months
```

---

### Mode 3: Winter Off-Peak Night Charging (00:00-05:00)
**Objective**: Charge from cheap night-rate electricity for daytime use

**Trigger Conditions**:
- Current season is winter
- Current time is cheap night hours (00:00-05:00)
- LG SOC < 35% (depleted from evening peak discharge)
- Anker SOC < 80%
- Grid import available

**Action**: Charge both batteries from grid at cheap night rates

**Financial Arbitrage**:
```
Buy: 1 kWh @ €0.25 (night rate, 00:00-05:00)
Store: Both batteries (~50% efficiency round-trip loss)
Discharge: 0.9 kWh @ €0.50 (peak hours, 17:00-21:00)
Net Profit: €0.50 × 0.9 - €0.25 = €0.20 per kWh stored

Example: 
- Charge 6.4 kWh Anker overnight @ €0.25 = €1.60
- Discharge during peak @ €0.50, value = €3.20
- Less efficiency loss and cost = €1.60 savings/cycle
- Nightly savings: €1.60 × 30 nights = €48/month
```

**Benefit**: Exploits tariff arbitrage between cheapest (night) and most expensive (peak) hours

**Efficiency Note**: This strategy only makes financial sense if:
- Night rate < 50% of peak rate (break-even point after battery losses)
- Example ✓: Night €0.25 vs Peak €0.50 (2x difference = worthwhile)
- Example ✗: Night €0.35 vs Peak €0.40 (1.14x difference = loss exceeds benefit)

---

### Mode 4: Winter Off-Peak Discharge (21:00-00:00)
**Objective**: Use stored battery energy for remaining evening loads

**Trigger Conditions**:
- Current season is winter
- Current time is off-peak but not cheap night (21:00-00:00)
- Grid import deficit > 0
- Anker SOC > minimum reserve

**Action**: Discharge batteries to cover household loads without charging overnight

**Benefit**: Reduces or eliminates need for overnight charging if peak discharge was sufficient

---

### Mode 4: Summer Mode (Sept-Oct, Mar-May) / High PV Days
**Objective**: Maximize solar utilization (original logic)

**Trigger Conditions**:
- NOT winter season
- Falls back to original summer control logic

**Action**:
- LG Chem is primary (managed by SolarEdge)
- Anker activates only when LG is saturated (≥95%) with surplus PV
- Anker discharges only when LG is depleted (≤20%) with grid import

**Benefit**: Optimizes for PV abundance, minimizes grid interaction

## Example Winter Configuration

### Scenario: German household with typical tariff structure

```
Peak Hours:  17:00-21:00 (€0.50/kWh)
Off-Peak:    21:00-06:00 (€0.25/kWh)
Normal Rate: 06:00-17:00 (€0.35/kWh)

Household Profile:
- Daytime consumption: 2 kWh (light, cooking, office)
- Evening consumption: 4 kWh (heating, cooking, 19:00-22:00)
- Night consumption: 2 kWh (heating, baseline)

Battery Capacity:
- LG Chem: 10 kWh
- Anker MAX: 6.4 kWh
- Total: 16.4 kWh

Configuration:
┌──────────────────────────────────┐
│ Winter Peak Hours Start: 17      │
│ Winter Peak Hours End: 21        │
│ Winter Min SOC Reserve: 20%      │
│ Winter LG Full Threshold: 80%    │
└──────────────────────────────────┘

Daily Operation:
06:00 - Start: Both batteries at 40% after night discharge
        Morning PV generates ~500W
        Charge both batteries (no discharge)
        
12:00 - Midday: Peak PV ~3000W
        LG reaches 80%
        Anker now charges from excess
        
17:00 - Evening starts: Peak tariff begins
        Both batteries at 90%+
        Household switches to heater load (500-800W)
        ANKER + LG discharge together
        Grid import minimized
        
21:00 - Peak tariff ends
        Both batteries at 60-70%
        Low tariff now active
        Could charge from grid if needed, or discharge slowly
        
06:00 - Morning: Batteries at 30-40%
        Day cycle repeats
```

## Implementation Steps

### 1. **Install the Updated YAML File**
```
Copy: homeassistant/packages/anker_max_control.yaml
To: /config/packages/
Restart Home Assistant
```

### 2. **Add Package Configuration** (if not already present)
Edit `/config/configuration.yaml`:
```yaml
homeassistant:
  packages:
    anker_max_control: !include_dir_named packages
```

### 3. **Configure Time Windows**
In Home Assistant UI:
- Go to: Settings → Devices & Services → Helpers
- Find: "Winter Peak Hours Start" and "Winter Peak Hours End"
- Set to match your local tariff structure

### 4. **Verify Winter Mode is Active**
In Home Assistant:
- Go to: Developer Tools → Templates
- Paste: `{{ 11 <= now().month or now().month <= 2 }}`
- Should show `True` if currently winter, `False` if summer

### 5. **Monitor Control Status**
- Create a lovelace card showing:
  - Sensor: `sensor.anker_control_status`
  - Shows: Current action (charge/discharge/idle)
  - Shows attributes: SOC levels, grid power, mode

**Example Lovelace YAML**:
```yaml
type: entities
entities:
  - entity: sensor.anker_control_status
    name: "Anker Control Action"
  - entity: sensor.solaredge_i1_b1_state_of_energy
    name: "LG Chem SOC"
  - entity: sensor.anker_solix_solarbank_max_ac_soc
    name: "Anker SOC"
  - entity: sensor.solaredge_i1_m1_ac_power
    name: "Grid Power"
```

## Winter Efficiency Tips

### 1. **Optimize Peak Hour Discharge**
- If both batteries are fully charged before 17:00, they'll both discharge during peak hours
- Stagger loads if possible: Do heavy loads (laundry, dishwasher) between 17:00-21:00 when batteries are discharging
- Avoid heavy loads outside peak if batteries are low

### 2. **Monitor Off-Peak Charging Decision**
- System can charge Anker from cheap grid power (21:00-06:00) if batteries are depleted
- Enable only if your tariff difference justifies the charge/discharge round-trip loss (~15%)
- Example: Off-peak €0.25 vs Peak €0.50 = 2x difference ✓ (worth it)
- Example: Off-peak €0.35 vs Peak €0.40 = only 14% difference ✗ (loss exceeds benefit)

### 3. **Adjust Reserve Based on Usage**
- If night loads are consistently high, increase min_soc_reserve to 25-30%
- If night loads are very low, can reduce to 15% (but not below 10% for battery health)
- Monitor battery state at 06:00 AM - if consistently below 20%, increase reserve

### 4. **Dynamic Tariff Integration** (Advanced)
If your utility offers real-time pricing (hour-ahead tariffs):
- Could further optimize by changing peak hours based on actual tariff
- Would require adding a REST sensor to read tariff data
- More complex, but potential for additional 5-10% savings

## Troubleshooting

### Problem: Anker not discharging during peak hours
**Check**:
1. Is winter mode active? (Check template in Dev Tools)
2. Is current time within peak hours? 
3. Is Anker SOC above minimum reserve?
4. Is there actually grid import (household needs more than PV produces)?

**Fix**:
- Manually increase household load during peak hours to verify discharge works
- Check Anker entity is controllable: Go to Settings → Devices → Anker entity
- Verify automation is enabled: Settings → Automations & Scenes

### Problem: Grid import still high during peak hours
**Causes**:
1. Peak hours not aligned with your actual tariff times
2. Household loads exceed battery discharge capacity (800W)
3. Batteries depleted before peak hours start

**Solutions**:
1. Verify peak tariff times with your utility provider
2. Install larger capacity batteries or stagger loads
3. Manual adjustment: Increase daytime charging threshold from 80% to 70% to pre-charge more aggressively

### Problem: Batteries fully depleted by morning
**Causes**:
1. Off-peak discharge too aggressive
2. Minimum reserve set too low
3. Night loads exceed battery capacity

**Solutions**:
1. Increase `min_soc_reserve` to 25-30%
2. Reduce household night-time loads or shift to off-peak hours
3. Consider adding more battery capacity

## Seasonal Transitions

### Autumn (October)
- System still in summer mode
- Monitor when to switch: Watch daily PV vs consumption
- Manually switch when consistent grid import needed in evening

### Spring (May)
- Switch back when consistent PV surplus appears
- Winter mode disables automatically May 1st
- Summer mode resumes (high PV, minimal grid import)

### Manual Override
Edit line in `anker_max_control.yaml`:
```yaml
is_winter: "{{ current_month in (11, 12, 1, 2) }}"
```
Change to match your climate zone:
- Germany/Central Europe: Keep as-is (Nov-Feb)
- Southern Europe: Use (12, 1) for shorter winter
- Northern Europe/Canada: Use (10, 11, 12, 1, 2, 3) for longer winter

## Expected Winter Savings

### Baseline (No Optimization)
- Winter grid import: 100% at peak tariffs
- Daily cost: 10 kWh × €0.45 avg = €4.50

### With Anker MAX Optimization
- Peak hour displacement: 30-50% (using 4-6 kWh battery discharge)
- Off-peak charging: Additional 10-20% if available
- Total savings: 40-60% of peak-hour consumption

### Example Math (German household)
```
Peak hours (17:00-21:00): 4 hours
Typical evening load: 800W average
Peak tariff cost: €0.50/kWh
Off-peak tariff cost: €0.25/kWh

WITHOUT optimization:
4 hours × 800W = 3.2 kWh from grid @ €0.50 = €1.60

WITH optimization:
Anker discharge: 400W average × 4 hours = 1.6 kWh (free, already paid for by PV)
LG Chem discharge: 400W average × 4 hours = 1.6 kWh (free, charged by PV)
Grid import: 0 kWh @ €0.50 = €0.00

Daily savings: €1.60
Monthly savings: €48
Yearly savings: €240+
```

This payoff improves if:
- Your tariff difference (peak vs off-peak) is larger
- Your evening loads are higher
- You have sunny days before winter

---

**Last Updated**: October 2026
**Version**: Winter Optimization v1.0
