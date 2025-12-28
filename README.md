# ⚡ Intelligent Load Shedding

A smart, reliable Home Assistant blueprint for managing electrical loads to prevent exceeding power capacity limits.

[![Import Load Shedding Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDiaoul%2Fhass-load-shedding%2Fmain%2Fload_shedding_control.yaml)

Intelligent, priority-based electrical load management to prevent exceeding power capacity limits. Perfect for homes with limited electrical capacity, solar installations, or time-of-use tariffs.

## ✨ Features

- ⚡ **Real-time Power Monitoring** - Tracks total consumption from your meter (Linky, Shelly, etc.)
- 🎯 **Priority-Based Control** - 5-level priority system (Critical, High, Medium, Low, Very Low)
- 🛡️ **Proactive Protection** - Sheds loads before hitting capacity (configurable safety margin)
- 🔄 **Intelligent Restoration** - Restores highest-priority loads first when power budget available
- 🚫 **Anti-Flapping Protection** - Prevents rapid on/off cycling with hysteresis and time delays
- 🔓 **Manual Overrides** - Global disable switch and per-load exemptions
- 📊 **Flexible Configuration** - Use input helpers for unlimited loads

## 🚀 Quick Start

### Prerequisites

You need:
1. **Total power sensor** - Measures household consumption (e.g., `sensor.linky_power`)
2. **Controllable loads** - Each needs:
   - Switch entity to turn load on/off
   - Power sensor showing current consumption
3. **Max capacity** - Your electrical limit in Watts (e.g., 9000W for 45A @ 230V)

### Setup Steps

#### 1. Create Input Helpers

For each load you want to manage, create **3 helpers in the same order**:

Go to **Settings** → **Devices & Services** → [**Helpers**](https://my.home-assistant.io/redirect/helpers/)

**Example for Water Heater:**

1. **Priority** (Input Select):
   - Name: `Load Priority - Water Heater`
   - Entity ID: `input_select.load_priority_water_heater`
   - Options: `Critical`, `High`, `Medium`, `Low`, `Very Low`

2. **Switch Entity** (Input Text):
   - Name: `Load Switch - Water Heater`
   - Entity ID: `input_text.load_switch_water_heater`
   - Initial value: `switch.water_heater`

3. **Power Sensor** (Input Text):
   - Name: `Load Power - Water Heater`
   - Entity ID: `input_text.load_power_water_heater`
   - Initial value: `sensor.water_heater_power`

**Repeat for each load** (EV charger, heat pump, dishwasher, etc.)

#### 2. Create Global Tracking Helpers

Create **2 global helpers** (one-time setup):

1. **Shed Tracker** (Input Text):
   - Name: `Load Shedding - Shed Tracker`
   - Entity ID: `input_text.load_shedding_shed_tracker`
   - Initial value: *(leave empty)*

2. **Last Action Time** (Input DateTime):
   - Name: `Load Shedding - Last Action`
   - Entity ID: `input_datetime.load_shedding_last_action`
   - Has date: ✅ Enabled
   - Has time: ✅ Enabled

#### 3. Configure Blueprint

Create automation from blueprint and configure:

**Power Monitoring:**
- Total Power Sensor: `sensor.linky_power`
- Max Capacity: `9000` W
- Safety Margin: `90` % (shed at 8100W)
- Restoration Margin: `80` % (restore when under 7200W)

**Load Configuration:**
- Load Priority Selects: Select all priority input_selects **in order**
- Load Switch Texts: Select all switch text inputs **in order**
- Load Power Texts: Select all power text inputs **in order**

**Tracking Helpers:**
- Shed Tracker: `input_text.load_shedding_shed_tracker`
- Last Action DateTime: `input_datetime.load_shedding_last_action`

**Optional:**
- Per-Load Exemption Toggles: Create input_boolean for loads you want to exempt
- Disable Load Shedding: Global on/off switch

## 🧠 How It Works

### Load Shedding Process

1. **Monitor** - Continuously tracks total power consumption
2. **Evaluate** - When power exceeds safety threshold (default 90%):
   - Wait for shedding delay (prevents reacting to transient spikes)
   - Build list of sheddable loads (excludes Critical priority and exempted loads)
   - Sort by priority (lowest first), then power (highest first within same priority)
3. **Shed** - Turn off loads until under threshold
4. **Track** - Remember which loads were shed

### Load Restoration Process

1. **Monitor** - When power drops below restoration threshold (default 80%):
   - Wait for restoration delay
   - Check minimum shed duration elapsed
2. **Evaluate** - Build list of shed loads:
   - Sort by priority (highest first), then power (lowest first within same priority)
3. **Restore** - Turn on loads one at a time while budget available
4. **Track** - Update shed list

### Example Scenario

**Setup:**
- Max capacity: 9000W
- Safety margin: 90% (shed at 8100W)
- Restoration margin: 80% (restore at 7200W)

**Loads:**
1. Heat Pump - 2000W - High priority
2. Water Heater - 3000W - Low priority
3. EV Charger - 7000W - Medium priority
4. Dishwasher - 1500W - Low priority

**Timeline:**
1. **Initial:** Heat pump ON (2000W)
2. **EV plugged in:** EV + heat pump = 9000W (under 8100W threshold) ✅
3. **Water heater starts:** 2000 + 7000 + 3000 = 12000W (exceeds 8100W) ⚠️
4. **After 10s delay:** Shed lowest priority first:
   - Shed water heater (Low, 3000W) → 9000W
   - Still over 8100W, shed dishwasher... but it's not on, skip
   - Shed next: EV charger (Medium, 7000W) → 2000W ✅
5. **Current state:** Only heat pump ON, water heater + EV charger shed
6. **Heat pump cycles off:** 0W (under 7200W restoration threshold)
7. **After 1min delay + 5min minimum shed:**
   - Restore highest priority first: EV charger (Medium) → 7000W
   - Try water heater (Low): 7000 + 3000 = 10000W > 7200W ❌ (not enough budget)
8. **Final:** EV charging, water heater still shed (will restore when EV finishes)

## 🎛️ Configuration Tips

### Priority Guidelines

- **Critical** - Never shed (e.g., refrigerator, security systems, medical equipment)
- **High** - Essential comfort (e.g., heating/cooling in main living areas)
- **Medium** - Important but deferrable (e.g., EV charging, water heater)
- **Low** - Nice to have (e.g., dishwasher, laundry)
- **Very Low** - Lowest priority (e.g., pool pump, electric car pre-conditioning)

### Timing Guidelines

- **Shedding Delay (10s)** - Prevents reacting to brief spikes (e.g., appliance startup)
- **Restoration Delay (1m)** - Allows power to stabilize before restoring loads
- **Minimum Shed Duration (5m)** - Prevents rapid on/off cycling (critical for appliance lifespan)

### Threshold Guidelines

- **Safety Margin (90%)** - Higher = more proactive, lower = use more capacity
- **Restoration Margin (80%)** - Must be lower than safety margin to prevent flapping
- **Gap (10% default)** - Hysteresis prevents constant shed/restore cycles

## 🤝 Support

If you encounter issues:
- Verify helper configuration (check entity IDs in input_text helpers)
- Test templates in **Developer Tools** → **Template**
- Review automation traces in **Settings** → **Automations & Scenes** → _your automation_ → **Traces**
- Check tracker state in **Developer Tools** → **States** → `input_text.load_shedding_shed_tracker`
- Open an issue on [GitHub](https://github.com/Diaoul/hass-load-shedding/issues)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

Made with ❤️ for the Home Assistant community
