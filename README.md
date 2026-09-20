# ⚡ Load Shedding Blueprint

**Version:** 1.1.0

A Home Assistant blueprint that keeps total power consumption under a capacity limit by shedding loads in priority order, and restoring them when there is room again.

**Requirements:** Home Assistant 2025.7.0 or later

[![Import Load Shedding Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDiaoul%2Fhass-load-shedding%2Fmain%2Fload_shedding.yaml)

Perfect for homes with limited electrical capacity, solar installations, or time-of-use tariffs.

## ✨ Features

- ⚡ **Real-time power monitoring** — tracks total consumption from your meter (Linky, Shelly, etc.)
- 🎯 **Order-based priority** — priority is the order of the list (top = highest, bottom = lowest)
- 🛡️ **Proactive protection** — sheds before hitting capacity, at a configurable safety margin
- 🔄 **Budget-aware restoration** — restores highest-priority loads first, only if they fit
- 🚫 **Anti-flapping protection** — margin gap plus a minimum shed duration
- 📝 **Structured configuration** — one form for every load, one datetime helper for the automation

## 📦 Installation

Click the button above to import the blueprint into your Home Assistant.

## 🚀 Quick Start

1. **Create the helpers** — one datetime tracker, plus one `input_boolean` per load (see below)
2. **Create an automation** from the blueprint
3. **Configure power monitoring** — power sensor, capacity sensor, margins
4. **Add your loads** — one entry per appliance, in priority order
5. **Select the tracking helper**

### ⏱️ Required: tracking helper

Go to **Settings** → **Devices & Services** → [**Helpers**](https://my.home-assistant.io/redirect/helpers/) and create a **Date and Time** helper:

- Name: `Load Shedding - Last Action`
- Entity ID: `input_datetime.load_shedding_last_action`
- Has date: ✅ Enabled
- Has time: ✅ Enabled

**Why?** It is stamped on every shed and every restore, and it is what enforces the minimum shed duration. Its initial value does not matter: an unset or unavailable helper simply reads as "long ago", so restoration is not blocked by a missing helper.

**Both date and time have to be enabled.** A helper with only one of them reports its `timestamp` as seconds since midnight rather than an epoch, which would read as "ages ago" on every run and silently switch off the anti-flapping protection. The automation refuses to restore anything while the helper has the wrong shape; shedding still works, since a breaker tripping is worse than a load staying off.

### 🔌 Required: one off override per load

Each load needs an `input_boolean` that *disables* it:

- **ON** = the load is shed
- **OFF** = the load is free to run

That boolean is also where the shed state lives — the automation keeps no list of its own, it reads the booleans every run. Wire it into whatever actually stops the load: a condition in your own heating automation, a `switch` template, an EV-charger `charging allowed` flag.

### 💡 Configuration example

**Power monitoring:**

- Total power sensor: `sensor.linky_power_watts` (your VA→W template sensor if needed)
- Max capacity sensor: `input_number.max_capacity` (or a template sensor for dynamic capacity)
- Safety margin: `90` % — shed above 8100 W if max is 9000 W
- Restoration margin: `80` % — restore below 7200 W

**Managed loads**, in priority order (drag & drop to reorder):

| Priority       | Load         | Detection Entity      | Off Override                   | Max Power |
| -------------- | ------------ | --------------------- | ------------------------------ | --------- |
| 🔴 **Highest** | Heat Pump    | `climate.living_room` | `input_boolean.hp_blocked`     | 2000 W    |
| 🟡 **Medium**  | EV Charger   | `switch.ev_charger`   | `input_boolean.ev_blocked`     | 7000 W    |
| 🟢 **Lowest**  | Water Heater | `switch.water_heater` | `input_boolean.water_blocked`  | 3000 W    |

**Tip:** a load you never want shed simply does not go in the list.

## 🧠 How It Works

Every run recomputes everything from current entity states — nothing is remembered between runs, so a missed run is harmless.

### Triggers

Event-driven only, no periodic tick:

- Home Assistant start
- the total power sensor changes state
- the max capacity sensor changes state

Both state triggers carry `not_to: [unavailable, unknown]`. That is not only about ignoring dropouts: a state trigger with no `to`/`from`/`not_to`/`not_from` matches everything, attribute updates included, so without it every attribute refresh on the meter would re-run the automation.

There is no trigger on the off overrides (they live inside the load list, and a state trigger needs entity ids), so an override flipped by hand stays flipped until the power sensor next updates — typically within its refresh interval.

### Guard

The run stops immediately, doing nothing at all, if:

- the total power sensor or the capacity sensor is unavailable
- capacity is 0
- the restoration margin is not lower than the safety margin
- the load list is empty, or an entry is missing its detection entity, off override or max power
- two entries share a detection entity

A broken entry makes the shed arithmetic wrong for *every* load, so the automation refuses to act rather than shed a partial list.

### Shedding (power above the safety threshold)

1. Candidates: every load that is **not already shed**, lowest priority (bottom of the list) first
2. Walk them while a deficit remains, shedding each one
3. Only a load that is **actually drawing power** subtracts from the deficit

So idle loads below the deficit-covering load are shed too. That is deliberate — an idle load that is free to start would blow the budget the moment it does — but it means a spike from an *unmanaged* load can shed the whole list.

### Restoration (power below the restoration threshold)

1. Only after the minimum shed duration has elapsed since the last shed or restore
2. Candidates: every load that is **shed**, highest priority first
3. A load is restored if its **rated** power still fits under the *shedding* threshold; the budget is then reduced by that amount
4. A load that does not fit is skipped, and the smaller ones behind it are still considered
5. A shed load that is still drawing power costs nothing: it is already in the meter reading, so its override is released whatever the budget says. Otherwise that override would stay on for as long as the load kept running

Rated power is used here, not measured: a shed load reads ~0 W, so its sensor says nothing about what it will draw once released.

#### Why the budget runs to the shedding threshold

The two margins answer different questions:

- The **restoration margin** decides *when it is calm enough to start giving power back*. It is the entry gate: nothing is restored while consumption sits above it.
- The **shedding margin** is where the automation intervenes, so it is the ceiling worth filling up to.

With a 9000 W capacity, restoration starts once consumption drops below 7200 W, and loads are then released until their rated powers reach 8100 W. A load rated 8000 W comes back in an empty house; budgeted against 7200 W it never could, however empty the house was.

This is the point of running load shedding at all: the sum of your loads is meant to exceed your supply, and the automation is what makes that safe. Restoration uses every watt of headroom that exists, and shedding takes it back when a load draws more than expected.

**A load rated above the shedding threshold stays shed.** Above 8100 W here, releasing it would only shed it again on the next run, so refusing beats a five-minute on/off cycle. The cure is a rated figure that matches reality, more capacity, or a wider gap between the margins — not a bigger number in the form.

Shedding and restoration are mutually exclusive — the thresholds cannot both be crossed in the same run.

### Example scenario

**Setup:** max 9000 W, shed above 8100 W, restore below 7200 W.

**Loads:** heat pump 2000 W (index 0), EV charger 7000 W (index 1), water heater 3000 W (index 2), dishwasher 1500 W (index 3).

1. Heat pump heating: 2000 W
2. EV plugged in: 2000 + 7000 = 9000 W, over the 8100 W threshold
3. Shed lowest priority first: dishwasher (idle, shed anyway, deficit unchanged), water heater (idle, shed anyway), EV charger (drawing 7000 W, deficit covered) → heat pump untouched
4. EV finishes elsewhere / heat pump cycles off: 0 W, below 7200 W
5. After the 5-minute minimum shed duration: budget is 8100 W, so the EV charger is released (7000 W); 1100 W is left, which holds neither the water heater nor the dishwasher
6. Next run, once consumption has settled, the rest come back

## 🎛️ Configuration Tips

### Priority

- **Top of the list** — highest priority, shed last, restored first (heating, essentials)
- **Bottom of the list** — lowest priority, shed first, restored last (pool pump, dishwasher)

Order is the *only* priority signal; two loads can never tie.

### Timing

- **Minimum shed duration (5 min)** — nothing is restored until this has elapsed since the last shed or restore. This is the primary anti-flapping protection.

The blueprint decides instantly on each sensor update. With a typical 30–60 s meter refresh, transient spikes are filtered by the sensor itself; with a faster sensor, the minimum shed duration still prevents cycling.

### Thresholds

- **Safety margin (90%)** — higher = more proactive, lower = use more of your capacity
- **Restoration margin (80%)** — must be lower than the safety margin. It decides *when* restoration may start, not how much may be restored: the budget itself runs up to the safety margin
- **Gap (10% default)** — hysteresis, 900 W of buffer at 9000 W capacity. Widen it if traces show loads restored and shed again shortly after

### Max capacity configuration

#### Static capacity

Create an **Input Number** helper: Settings → Helpers → Create Helper → Number, min 0, max 50000, step 100, unit W, set to your limit (e.g. 9000).

#### Dynamic capacity

A template sensor, in `configuration.yaml`:

**Solar + grid:**

```yaml
template:
  - sensor:
      - name: "Available Capacity"
        unit_of_measurement: "W"
        state: >-
          {{ (9000 + states('sensor.solar_power')|float(0)) | int }}
```

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

A capacity sensor that drops to 0 or becomes unavailable stops the automation rather than shedding everything.

#### Converting VA to Watts

```yaml
template:
  - sensor:
      - name: "Linky Power Watts"
        unit_of_measurement: "W"
        device_class: power
        state: >-
          {{ (states('sensor.linky_power_va')|float(0) * 0.95) | int }}
```

_Replace 0.95 with your measured power factor._

### Detection entity vs off override

They answer two different questions:

- **Detection entity** — *is this load drawing power right now?* Only this decides whether shedding it reduces the deficit.
- **Off override** — *the switch that stops it.* Turned ON to shed, OFF to restore.

Example: watch `climate.bedroom`, block via `input_boolean.bedroom_heating_blocked` that your heating automation honours.

### Power sensor (optional)

Per-load sensor giving real consumption. Used **only** when computing how much a shed will actually save. A negative or unavailable reading falls back to the rated max power. It is never used for the restoration budget.

It is read at the moment the decision is made, so a sensor lagging behind reality cuts both ways: one reporting 0 W for a load that has just started subtracts nothing from the deficit, and the plan reaches further down the list than it needed to. A sensor slower than your meter is worse than no sensor at all — leave the field empty and let the rated power stand in.

---

## 📚 Advanced Documentation

<details>
<summary><b>🔍 Detailed decision logic (click to expand)</b></summary>

### Shedding, step by step

| Step | Action                                     | Sort                                    |
| ---- | ------------------------------------------ | --------------------------------------- |
| 1️⃣   | Stop unless power > safety threshold       | –                                       |
| 2️⃣   | Candidates: loads not already shed, override readable | **List order, highest index first** |
| 3️⃣   | Shed each candidate while a deficit remains | –                                      |
| 4️⃣   | Subtract from the deficit only if the load is actually ON | –                         |
| 5️⃣   | One `homeassistant.turn_on` per entity, then stamp the tracker | –                        |

Initial deficit is `current_power - shedding_threshold`. The power subtracted is the load's power sensor reading when available, its rated max power otherwise.

### Restoration, step by step

| Step | Action                                                       | Sort                               |
| ---- | ------------------------------------------------------------ | ---------------------------------- |
| 1️⃣   | Stop unless power < restoration threshold                     | –                                  |
| 2️⃣   | Stop unless the tracker is usable and the minimum shed duration has elapsed | –                    |
| 3️⃣   | Candidates: every shed load with a readable override          | **List order, lowest index first** |
| 4️⃣   | Budget starts at `shedding_threshold - current_power`          | –                                  |
| 5️⃣   | Restore a load if its rated power fits, then reduce the budget; a load still drawing power costs 0 | –       |
| 6️⃣   | One `homeassistant.turn_off` per entity, then stamp the tracker | –                                |

A load that does not fit is skipped, not a stopping point: smaller lower-priority loads behind it can still come back.

### Load state detection

| Entity domain              | Method   | ON                                                       | OFF                            |
| -------------------------- | -------- | -------------------------------------------------------- | ------------------------------ |
| **Climate (hvac_action)**  | Default  | `hvac_action` is `heating` or `cooling`                   | anything else, including unset |
| **Climate (sensor_based)** | Fallback | `hvac_mode` + temperature delta                           | at target, or mode `off`       |
| **Switch**                 | –        | state `on`                                                | state `off`                    |

**sensor_based logic:**

- **heat**: ON when current < target
- **cool**: ON when current > target
- **heat_cool / auto**: ON when current ≠ target
- **off** (and any other mode): OFF

Use `hvac_action` when your thermostat reports it; `sensor_based` only when it does not.

### Anti-flapping protection

| Protection                | Default                        | Purpose                                          |
| ------------------------- | ------------------------------ | ------------------------------------------------ |
| **Minimum shed duration** | 5 minutes                      | No restoration until it elapses                  |
| **Hysteresis gap**        | 10% (90% shed, 80% restore)    | Prevents threshold bounce                        |
| **Sensor refresh rate**   | User-dependent, typically 30–60 s | Natural debouncing of transient spikes        |
| **Margin validation**     | Enforced by the guard          | Blocks configurations where restoration ≥ safety |

### Concurrency

`mode: single`. An overlapping run is dropped and logged as `Already running`; `max_exceeded` is left loud on purpose so those drops stay visible. Dropping a run is safe: the next one recomputes from current state.

</details>

---

## 🤝 Support

If you run into problems:

- **Nothing happens at all** — the guard stopped the run. Check the trace: unavailable power or capacity sensor, a load missing its off override or max power, a duplicate detection entity, or restoration margin ≥ safety margin.
- **Loads not shedding** — confirm the detection entity really reports the load as active (`hvac_action` for climate) and that the off override is an `input_boolean` something actually honours.
- **Loads not coming back** — check the minimum shed duration against the tracker helper, that the helper has both date and time enabled, and that the load's rated power fits under the shedding threshold. A load rated above the shedding threshold never fits, by design.
- **HA version** — 2025.7.0 or later.
- Read the trace: **Settings** → **Automations & Scenes** → _your automation_ → **Traces**. Changed Variables shows `load_status`, `shed_plan` and `restore_plan` as rendered.
- Open an issue on [GitHub](https://github.com/Diaoul/hass-load-shedding/issues)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

Made with ❤️ for the Home Assistant community
