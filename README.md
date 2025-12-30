# ⚡ Load Shedding Blueprint

**Version:** 1.0

A smart, reliable Home Assistant blueprint for managing electrical loads to prevent exceeding power capacity limits.

**Requirements:** Home Assistant 2025.7.0 or later

[![Import Load Shedding Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDiaoul%2Fhass-load-shedding%2Fmain%2Fload_shedding.yaml)

Priority-based electrical load management to prevent exceeding power capacity limits. Perfect for homes with limited electrical capacity, solar installations, or time-of-use tariffs.

## ✨ Features

- ⚡ **Real-time Power Monitoring** - Tracks total consumption from your meter (Linky, Shelly, etc.)
- 🎯 **Order-Based Priority** - Priority determined by list order (top = highest, bottom = lowest)
- 🛡️ **Proactive Protection** - Sheds loads before hitting capacity (configurable safety margin)
- 🔄 **Intelligent Restoration** - Restores highest-priority loads first when power budget available
- 🚫 **Anti-Flapping Protection** - Prevents rapid on/off cycling with hysteresis and time delays
- 📝 **Structured Configuration** - Simple form-based setup with no input helpers needed per load

## 📦 Installation

Click the button above to import the blueprint directly into your Home Assistant.

## 🚀 Quick Start

1. **Create tracking helpers** - One-time setup (see below)
2. **Create an automation** from the blueprint
3. **Configure power monitoring** - Power sensor, capacity, margins
4. **Add your loads** - Click "Add Load" for each appliance
5. **Set tracking helpers** - Link the helpers you created

### ⏱️ Required: Tracking Helper

For anti-flapping protection, create 1 datetime helper:

**1. Create Helper**

Go to **Settings** → **Devices & Services** → [**Helpers**](https://my.home-assistant.io/redirect/helpers/)

Create a **Date and Time** helper:

- Name: `Load Shedding - Last Action`
- Entity ID: `input_datetime.load_shedding_last_action`
- Has date: ✅ Enabled
- Has time: ✅ Enabled

**2. Configure in Blueprint**

Select your helper in the **Tracking Helpers** section.

**Why?** This helper tracks when the last shed/restore action occurred to enforce minimum shed duration and prevent rapid on/off cycling.

### 💡 Configuration Example

**Power Monitoring:**

- Total Power Sensor: `sensor.linky_power_watts` (your VA→W template sensor if needed)
- Max Capacity Sensor: `input_number.max_capacity` (or template sensor for dynamic)
- Safety Margin: `90` % (shed at 8100W if max is 9000W)
- Restoration Margin: `80` % (restore when under 7200W)

**Managed Loads:**

Click "Add Load" for each load you want to manage. **Priority is determined by order** - loads at the top have highest priority (shed last), loads at the bottom have lowest priority (shed first).

**Example loads (in priority order):**

| Priority       | Load         | Primary Entity        | Control Switch             | Power |
| -------------- | ------------ | --------------------- | -------------------------- | ----- |
| 🔴 **Highest** | Heat Pump    | `climate.living_room` | `switch.heat_pump_breaker` | 2000W |
| 🟡 **Medium**  | EV Charger   | `switch.ev_charger`   | _(leave empty)_            | 7000W |
| 🟢 **Lowest**  | Water Heater | `switch.water_heater` | _(leave empty)_            | 3000W |

💡 **Tip**: Use drag & drop in the blueprint UI to reorder loads and adjust priorities.

## 🧠 How It Works

### Load Shedding Process

1. **Monitor** - Continuously tracks total power consumption
2. **Evaluate** - When power exceeds safety threshold (default 90%):
   - Build list of active loads that can be shed
   - Sort by list order (highest index first = lowest priority), then power (highest first)
3. **Shed** - Turn off loads until under threshold
4. **Track** - Remember which loads were shed

### Load Restoration Process

1. **Monitor** - When power drops below restoration threshold (default 80%):
   - Check minimum shed duration elapsed (default: 5 minutes)
2. **Evaluate** - Build list of shed loads:
   - Sort by list order (lowest index first = highest priority), then power (lowest first)
3. **Restore** - Turn on loads one at a time while budget available
4. **Track** - Update shed list

### Example Scenario

**Setup:**

- Max capacity: 9000W
- Safety margin: 90% (shed at 8100W)
- Restoration margin: 80% (restore at 7200W)

**Loads (in priority order - highest to lowest):**

1. Heat Pump - 2000W (index 0 - highest priority)
2. EV Charger - 7000W (index 1)
3. Water Heater - 3000W (index 2)
4. Dishwasher - 1500W (index 3 - lowest priority)

**Timeline:**

1. **Initial:** Heat pump ON (2000W)
2. **EV plugged in:** EV + heat pump = 9000W (under 8100W threshold) ✅
3. **Water heater starts:** 2000 + 7000 + 3000 = 12000W (exceeds 8100W) ⚠️
4. **Immediately shed** lowest priority first:
   - Shed dishwasher (index 3)... but it's not on, skip
   - Shed water heater (index 2, 3000W) → 9000W
   - Still over 8100W, shed EV charger (index 1, 7000W) → 2000W ✅
5. **Current state:** Only heat pump ON, water heater + EV charger shed
6. **Heat pump cycles off:** 0W (under 7200W restoration threshold)
7. **After 5min minimum shed duration:**
   - Restore highest priority first: EV charger (index 1) → 7000W
   - Try water heater (index 2): 7000 + 3000 = 10000W > 7200W ❌ (not enough budget)
8. **Final:** EV charging, water heater still shed (will restore when EV finishes)

## 🎛️ Configuration Tips

### Priority Guidelines

Priority is determined by the order of loads in your configuration:

- **Loads at the top** - Highest priority, shed last (e.g., heating/cooling, essential appliances)
- **Loads in the middle** - Medium priority (e.g., EV charging, water heater)
- **Loads at the bottom** - Lowest priority, shed first (e.g., pool pump, laundry, dishwasher)

**Tip:** If you have loads you never want shed, simply don't include them in the managed loads list. Only add loads you're willing to have turned off automatically.

### Timing Guidelines

- **Minimum Shed Duration (5m)** - Once a load is shed, it stays off for at least this duration. This prevents rapid on/off cycling that can damage appliances and cause power instability. This is the PRIMARY anti-flapping protection.

**Decision speed:** The blueprint makes instant shedding and restoration decisions based on your power sensor readings. Since most power sensors update every 30-60 seconds, the sensor refresh rate naturally debounces transient spikes. Combined with the 5-minute minimum shed duration, this provides robust protection against flapping without adding artificial delays.

**For slow sensors (1 minute+ refresh):** The blueprint's instant decisions work perfectly. Your sensor can't detect brief spikes anyway, so there's no benefit to additional delays.

**For fast sensors (< 10 seconds refresh):** The blueprint still makes instant decisions, but your faster sensor will better capture real power fluctuations. The minimum shed duration still prevents flapping.

### Threshold Guidelines

- **Safety Margin (90%)** - Higher = more proactive, lower = use more capacity
- **Restoration Margin (80%)** - Must be lower than safety margin to prevent flapping
- **Gap (10% default)** - Hysteresis prevents constant shed/restore cycles

### Max Capacity Configuration

The blueprint requires a **sensor or input_number** for max capacity. Choose based on your needs:

#### Static Capacity (Simple)

Create an **Input Number** helper for fixed capacity:

1. Go to **Settings** → **Helpers** → **Create Helper** → **Number**
2. Name: "Max Power Capacity"
3. Minimum: 0, Maximum: 50000, Step: 100
4. Unit: W
5. Set value to your limit (e.g., 9000)

#### Dynamic Capacity (Advanced)

Create a **Template Sensor** for capacity that changes based on conditions.

Add to your `configuration.yaml`:

**Solar + Grid:**

```yaml
template:
  - sensor:
      - name: "Available Capacity"
        unit_of_measurement: "W"
        state: >-
          {{ (9000 + states('sensor.solar_power')|float(0)) | int }}
```

_Capacity increases when solar is producing power_

**Time-of-use:**

```yaml
template:
  - sensor:
      - name: "Available Capacity"
        unit_of_measurement: "W"
        state: >-
          {% if now().hour >= 22 or now().hour < 6 %}
            12000
          {% else %}
            9000
          {% endif %}
```

_Higher limit during off-peak hours (10 PM - 6 AM)_

**Battery state:**

```yaml
template:
  - sensor:
      - name: "Available Capacity"
        unit_of_measurement: "W"
        state: >-
          {% set battery_soc = states('sensor.battery_soc')|float(0) %}
          {% if battery_soc > 80 %}
            12000
          {% elif battery_soc > 50 %}
            10000
          {% else %}
            9000
          {% endif %}
```

_Capacity varies based on battery charge level_

#### Converting VA to Watts

If your meter reports VA (apparent power) instead of W (real power), create a conversion sensor:

```yaml
template:
  - sensor:
      - name: "Linky Power Watts"
        unit_of_measurement: "W"
        device_class: power
        state: >-
          {{ (states('sensor.linky_power_va')|float(0) * 0.95) | int }}
```

_Replace 0.95 with your measured power factor_

### Primary Entity vs Control Switch

**Primary Entity** determines when the load is actually ON:

- **Climate entities**: Checks if `hvac_action` is 'heating' or 'cooling'
  - Useful: Only sheds when climate is actively consuming power
  - Example: `climate.living_room` won't shed if thermostat is idle
- **Switch entities**: Checks if state is 'on'
  - Simple loads that consume power when switch is on
  - Example: `switch.water_heater`

**Control Switch** (optional) provides separate control:

- **Accepts**: Switch or input_boolean entities
- **When to use**: You want to cut power via a different entity than the status indicator
  - Example: Monitor `climate.bedroom` but control via `switch.bedroom_breaker`
  - Example: Monitor `sensor.ev_charging_power` but control via `input_boolean.ev_charging_allowed`
- **When to skip**: Direct control of primary entity is fine
  - Example: `switch.pool_pump` can be controlled directly

### Inverted Control Logic

Some use cases require **inverted control signals**:

**Normal behavior (default):**

- Shedding: Sends `turn_off` to control entity
- Restoration: Sends `turn_on` to control entity

**Inverted behavior (when enabled):**

- Shedding: Sends `turn_on` to control entity
- Restoration: Sends `turn_off` to control entity

**Use cases:**

- Override switches: `input_boolean.block_ev_charging` (turn on = block charging)
- Automation triggers: Enable automation to prevent load
- Inverse logic devices: ON = disabled, OFF = enabled

**Example configuration:**

- **Primary Entity**: `switch.ev_charger` (monitors charger state)
- **Control Switch**: `input_boolean.block_charging` (override boolean)
- **Invert Control Logic**: ✅ Enabled
- **Behavior**: When shedding, blueprint turns ON the block_charging boolean to stop charging

---

## 📚 Advanced Documentation

<details>
<summary><b>🔍 Detailed Decision Logic (Click to expand)</b></summary>

### Load Shedding Algorithm

The blueprint uses a **priority-based algorithm** with the following decision flow:

**Shedding Mode** (when power > safety threshold):

| Step | Action                                      | Priority Sort                                                    |
| ---- | ------------------------------------------- | ---------------------------------------------------------------- |
| 1️⃣   | Identify active loads that can be shed      | -                                                                |
| 2️⃣   | Sort candidates                             | **List order (highest index first)**, then power (highest first) |
| 3️⃣   | Shed loads one by one until under threshold | Lowest priority shed first                                       |
| 4️⃣   | Track shed loads in helper                  | -                                                                |

**Restoration Mode** (when power < restoration threshold):

| Step | Action                                          | Priority Sort                                                  |
| ---- | ----------------------------------------------- | -------------------------------------------------------------- |
| 1️⃣   | Check minimum shed duration (5m default)        | -                                                              |
| 2️⃣   | Identify shed loads that are OFF                | -                                                              |
| 3️⃣   | Sort candidates                                 | **List order (lowest index first)**, then power (lowest first) |
| 4️⃣   | Calculate available power budget                | restoration_threshold - current_power                          |
| 5️⃣   | Restore loads one by one while budget available | Highest priority restored first                                |
| 6️⃣   | Update tracker after each restoration           | -                                                              |

**Key Points:**

- **Priority = List Order**: Top load in config = highest priority (shed last, restore first)
- **Power is secondary sort**: When same priority level, higher power shed first (makes more room), lower power restored first (fits in budget easier)
- **Instant decisions**: Makes immediate shedding/restoration decisions based on sensor readings
- **Minimum shed duration**: PRIMARY anti-flapping protection - prevents rapid cycling that damages appliances

### Load State Detection

How the blueprint determines if a load is ON or OFF:

| Entity Domain              | Detection Method | ON Condition                                                         | OFF Condition                   | Use Case                                      |
| -------------------------- | ---------------- | -------------------------------------------------------------------- | ------------------------------- | --------------------------------------------- |
| **Climate (hvac_action)**  | Default          | `hvac_action` = 'heating' or 'cooling'                               | `hvac_action` = 'idle' or 'off' | Thermostats that report hvac_action correctly |
| **Climate (sensor_based)** | Fallback         | `hvac_mode` + temp delta<br>(e.g., mode='heat' AND current < target) | At target temp or mode='off'    | Thermostats that don't report hvac_action     |
| **Switch**                 | N/A              | `state` = 'on'                                                       | `state` = 'off'                 | Simple on/off loads                           |

**Climate Detection Methods:**

- **hvac_action (recommended)**: Most accurate - detects when actively heating/cooling. Use when your thermostat reports hvac_action correctly.
- **sensor_based (fallback)**: Uses hvac_mode + temperature sensors to infer heating/cooling. Checks if current temperature is below/above target. More accurate than hvac_mode alone. Use for thermostats that don't report hvac_action.
- **Configure per load**: Set "Climate Detection Method" field when adding each climate entity

**Sensor-based detection logic:**

- **Heat mode**: ON when current < target, OFF when current ≥ target
- **Cool mode**: ON when current > target, OFF when current ≤ target
- **Auto mode**: ON when current ≠ target, OFF when current = target
- **Off mode**: Always OFF

**Control Switch (Optional):**

- **Primary Entity**: Used to detect load state (is it consuming power?)
- **Control Switch**: Used to actually turn load on/off (accepts switch or input_boolean)
- **Example**: Monitor `climate.bedroom` status, control via `switch.bedroom_breaker`
- **Example**: Monitor power sensor, control via `input_boolean.charging_allowed`
- **When to use**: Status entity differs from control entity
- **When to skip**: Primary entity can be controlled directly

### Anti-Flapping Protection

Multiple layers prevent rapid on/off cycling:

| Protection                | Default                           | Purpose                                          |
| ------------------------- | --------------------------------- | ------------------------------------------------ |
| **Minimum Shed Duration** | 5 minutes                         | Prevent damage from frequent power cycling       |
| **Hysteresis Gap**        | 10% (90% shed, 80% restore)       | Prevent threshold bounce                         |
| **Sensor Refresh Rate**   | User-dependent (typically 30-60s) | Natural debouncing of transient spikes           |
| **Margin Validation**     | Enforced                          | Blocks configurations where restoration ≥ safety |

**Note:** The blueprint makes instant decisions based on sensor readings. With typical sensor refresh rates of 30-60 seconds, transient spikes are naturally filtered out. The minimum shed duration provides robust anti-flapping protection.

**Example:**

- Safety margin: 90% (shed at 8100W)
- Restoration margin: 80% (restore at 7200W)
- Gap: 900W buffer prevents constant shed/restore

</details>

---

## 🤝 Support

If you encounter issues:

- **Validation errors:** Check for duplicate primary entities in your configuration
- **Loads not shedding:** Check primary entity and control switch entities are valid and loads are included in managed loads list
- **HA version:** Ensure you're running Home Assistant 2025.7.0 or later
- Review automation traces in **Settings** → **Automations & Scenes** → _your automation_ → **Traces**
- Check tracker state in **Developer Tools** → **States** → `input_text.load_shedding_shed_tracker`
  - Should contain JSON array of load indices: `[0, 3, 7]` (not entity IDs - saves space!)
- Open an issue on [GitHub](https://github.com/Diaoul/hass-load-shedding/issues)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

Made with ❤️ for the Home Assistant community
