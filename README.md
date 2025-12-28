# ⚡ Intelligent Load Shedding

A smart, reliable Home Assistant blueprint for managing electrical loads to prevent exceeding power capacity limits.

**Requirements:** Home Assistant 2025.7.0 or later

[![Import Load Shedding Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDiaoul%2Fhass-load-shedding%2Fmain%2Fload_shedding.yaml)

Intelligent, priority-based electrical load management to prevent exceeding power capacity limits. Perfect for homes with limited electrical capacity, solar installations, or time-of-use tariffs.

## ✨ Features

- ⚡ **Real-time Power Monitoring** - Tracks total consumption from your meter (Linky, Shelly, etc.)
- 🎯 **Order-Based Priority** - Priority determined by list order (top = highest, bottom = lowest)
- 🛡️ **Proactive Protection** - Sheds loads before hitting capacity (configurable safety margin)
- 🔄 **Intelligent Restoration** - Restores highest-priority loads first when power budget available
- 🚫 **Anti-Flapping Protection** - Prevents rapid on/off cycling with hysteresis and time delays
- 🔓 **Manual Override** - Global disable switch to bypass load shedding
- 📝 **Structured Configuration** - Simple form-based setup with no input helpers needed per load

## 🚀 Quick Start

### Prerequisites

You need:
1. **Total power sensor** - Measures household consumption (e.g., `sensor.linky_power`)
2. **Controllable loads** - Each needs:
   - Primary entity (climate or switch) to detect load status
   - Estimated power consumption in Watts (rated power)
   - (Optional) Separate control switch for on/off control
3. **Max capacity** - Your electrical limit in Watts (e.g., 9000W for 45A @ 230V)

### Setup Steps

#### 1. Create Tracking Helpers (One-Time Setup)

Create **2 global helpers** to track automation state:

Go to **Settings** → **Devices & Services** → [**Helpers**](https://my.home-assistant.io/redirect/helpers/)

1. **Shed Tracker** (Input Text):
   - Name: `Load Shedding - Shed Tracker`
   - Entity ID: `input_text.load_shedding_shed_tracker`
   - Initial value: *(leave empty)*
   - Max length: 255 (or higher if managing many loads)

2. **Last Action Time** (Input DateTime):
   - Name: `Load Shedding - Last Action`
   - Entity ID: `input_datetime.load_shedding_last_action`
   - Has date: ✅ Enabled
   - Has time: ✅ Enabled

#### 2. Import and Configure Blueprint

1. Click the import button at the top of this README
2. Create automation from blueprint
3. Configure each section:

**Power Monitoring:**
- Total Power Sensor: `sensor.linky_power`
- Max Capacity: `9000` W
- Safety Margin: `90` % (shed at 8100W)
- Restoration Margin: `80` % (restore when under 7200W)

**Managed Loads:**

Click "Add Load" for each load you want to manage. **Priority is determined by order** - loads at the top have highest priority (shed last), loads at the bottom have lowest priority (shed first).

**Example loads (in priority order - highest to lowest):**

1. **Heat Pump** (highest priority - shed last)
   - Primary Entity: `climate.living_room`
   - Control Switch: `switch.heat_pump_breaker` (optional - to cut power)
   - Estimated Power: `2000` W

2. **EV Charger** (medium priority)
   - Primary Entity: `switch.ev_charger`
   - Control Switch: *(leave empty - control switch directly)*
   - Estimated Power: `7000` W

3. **Water Heater** (lowest priority - shed first)
   - Primary Entity: `switch.water_heater`
   - Control Switch: *(leave empty)*
   - Estimated Power: `3000` W

Use drag & drop to reorder loads and adjust priorities.

**Tracking Helpers:**
- Shed Tracker: `input_text.load_shedding_shed_tracker`
- Last Action DateTime: `input_datetime.load_shedding_last_action`

**Optional:**
- Disable Load Shedding: Create `input_boolean.disable_load_shedding` to globally disable

## 🧠 How It Works

### Load Shedding Process

1. **Monitor** - Continuously tracks total power consumption
2. **Evaluate** - When power exceeds safety threshold (default 90%):
   - Wait for shedding delay (prevents reacting to transient spikes)
   - Build list of active loads that can be shed
   - Sort by list order (highest index first = lowest priority), then power (highest first)
3. **Shed** - Turn off loads until under threshold
4. **Track** - Remember which loads were shed

### Load Restoration Process

1. **Monitor** - When power drops below restoration threshold (default 80%):
   - Wait for restoration delay
   - Check minimum shed duration elapsed
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
4. **After 10s delay:** Shed lowest priority first:
   - Shed dishwasher (index 3)... but it's not on, skip
   - Shed water heater (index 2, 3000W) → 9000W
   - Still over 8100W, shed EV charger (index 1, 7000W) → 2000W ✅
5. **Current state:** Only heat pump ON, water heater + EV charger shed
6. **Heat pump cycles off:** 0W (under 7200W restoration threshold)
7. **After 1min delay + 5min minimum shed:**
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

- **Shedding Delay (10s)** - Prevents reacting to brief spikes (e.g., appliance startup)
- **Restoration Delay (1m)** - Allows power to stabilize before restoring loads
- **Minimum Shed Duration (5m)** - Prevents rapid on/off cycling (critical for appliance lifespan)

### Threshold Guidelines

- **Safety Margin (90%)** - Higher = more proactive, lower = use more capacity
- **Restoration Margin (80%)** - Must be lower than safety margin to prevent flapping
- **Gap (10% default)** - Hysteresis prevents constant shed/restore cycles

### Primary Entity vs Control Switch

**Primary Entity** determines when the load is actually ON:

- **Climate entities**: Checks if `hvac_action` is 'heating' or 'cooling'
  - Useful: Only sheds when climate is actively consuming power
  - Example: `climate.living_room` won't shed if thermostat is idle
- **Switch entities**: Checks if state is 'on'
  - Simple loads that consume power when switch is on
  - Example: `switch.water_heater`

**Control Switch** (optional) provides separate control:

- **When to use**: You want to cut power via a different switch than the status indicator
  - Example: Monitor `climate.bedroom` but control via `switch.bedroom_breaker`
- **When to skip**: Direct control of primary entity is fine
  - Example: `switch.pool_pump` can be controlled directly

## 🤝 Support

If you encounter issues:
- **Validation errors:** Check for duplicate primary entities in your configuration
- **Loads not shedding:** Check primary entity and control switch entities are valid and loads are included in managed loads list
- **HA version:** Ensure you're running Home Assistant 2025.7.0 or later
- Review automation traces in **Settings** → **Automations & Scenes** → _your automation_ → **Traces**
- Check tracker state in **Developer Tools** → **States** → `input_text.load_shedding_shed_tracker`
  - Should contain JSON array of entity IDs: `["climate.living_room", "switch.water_heater"]`
- Open an issue on [GitHub](https://github.com/Diaoul/hass-load-shedding/issues)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

Made with ❤️ for the Home Assistant community
