# PV Zero Surplus (Solar Surplus Control)

This blueprint controls solar surplus in three different ways. It reacts to changes in the grid power sensor and either adjusts the inverter limit, switches devices on/off, or continuously controls the power of consumers.

---

## Installation

Simply integrate the blueprint into your Home Assistant using the button below. The file will be saved automatically under Blueprints.

<a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fjayjojayson%2Fblueprint_pv-zero-surplus%2Fmain%2Fpv-zero-surplus_en.yaml">
  <img width="250" alt="blueprint" src="https://github.com/user-attachments/assets/fa01530a-1d52-4b2b-b637-1269bd0cd747">
</a>  

> Trigger:
> The blueprint is executed on every change of the grid power sensor (except for "unavailable" or "unknown").


---

## Operating Modes

### ⚡ Mode 1: Inverter Limit (Zero Export)

Dynamically adjusts the power limit of the inverter to achieve zero export.

**Calculation:**
```
Target Limit = Current Limit + Grid Import - Buffer
```

- **Buffer:** Desired remaining grid import (e.g. 10W) to compensate for measurement fluctuations
- **Hysteresis:** Changes are only sent when the difference exceeds the hysteresis value (reduces radio traffic)

**Example:**
- Grid import: 200W (importing from grid)
- Buffer: 10W
- Calculated limit: 190W higher than current → inverter is reduced

---

### 🔌 Mode 2: Device On/Off

Switches devices on or off based on the available surplus.

**Logic for each device:**

```
IF solar power > 0 AND surplus >= turn-on threshold AND device is OFF
    AND (device has priority 1 OR all devices with higher priority are already switched on)
    → turn device ON

IF (surplus < turn-off threshold OR solar power = 0) AND device is ON
    → turn device OFF
```

**Example with priorities:**
- Device A: Priority 1, turn-on threshold 300W
- Device B: Priority 2, turn-on threshold 500W
- Surplus: 600W

→ Device A is switched on first (higher priority)
→ Device B is switched on if there is still surplus available

**Note:** The turn-off threshold should always be lower than the turn-on threshold (hysteresis) to avoid constant on/off flickering.

---

### 📊 Mode 3: Power Control

Continuously controls the power of consumers (e.g. heating element, wallbox).

**Calculation for each device:**

```
Available Surplus = Surplus - Buffer

IF solar power = 0 OR Available Surplus < minimum device power
    → Target Power = 0
ELSE IF (device has priority 1 OR all devices with higher priority are at maximum power)
    → Target Power = min(Available Surplus, Maximum Device Power)
ELSE
    → Target Power = 0
```

**Example with priorities:**
- Device A: Priority 1, Min 100W, Max 1000W
- Device B: Priority 2, Min 200W, Max 1500W
- Surplus: 2000W, Buffer: 10W → Available: 1990W

1. Device A (Priority 1) → 1000W (maximum)
2. Remaining: 990W
3. Device B (Priority 2) → 990W (limited by available surplus)

---

## Supported Device Types

### On/Off Devices
- Switch
- Input Boolean
- Light
- Fan
- All entities with `turn_on` / `turn_off` support

### Power Control Devices
- Number (e.g. OpenDTU Limit)
- Input Number
- All entities with `set_value` support

---

## Sensors

| Sensor | Description |
|--------|-------------|
| **Grid Power Sensor** | Grid import/export in watts. Positive = import, Negative = export |
| **Inverter Power** | Current solar production in watts |

---

## Important Notes

1. **Inverter Limit:** Make sure to use the NON-PERSISTENT limit entity! Frequent writes will otherwise destroy the EEPROM/Flash of the inverter.

2. **Buffer:** A buffer of 5-20W prevents unwanted export due to measurement fluctuations.

3. **Multiple Devices:** Up to 3 on/off devices and 3 power control devices can be configured. Empty slots are automatically ignored.

4. **Priorities:** 
   - Priority 1 = highest priority (switched on/controlled first)
   - Devices with lower priority (2, 3) are only switched on/controlled when all devices with higher priority are already switched on or have reached their maximum power

5. **Hysteresis:** Both for inverter limit and on/off devices, the hysteresis prevents constant flickering on small changes.
