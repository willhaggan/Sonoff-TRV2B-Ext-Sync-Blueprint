# Sonoff TRVZB External Sensor Sync — V4

Synchronise one or more external room temperature sensors with a **SONOFF TRVZB** through **Home Assistant and Zigbee2MQTT**.

The blueprint averages the valid readings, sends that temperature to the TRV's external input and manages the temperature-source selection. Create **one automation per TRV**; each automation can use multiple external sensors.

[![Import blueprint into Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fwillhaggan%2FSonoff-TRV2B-Ext-Sync-Blueprint%2Fblob%2Fmain%2FTRV2B_External_Sync)

[View the blueprint](https://github.com/willhaggan/Sonoff-TRV2B-Ext-Sync-Blueprint/blob/main/TRV2B_External_Sync) · [Community discussion](https://community.home-assistant.io/t/sonoff-trvzb-external-sensor-sync-with-external-mode-and-null-checks-zigbee2mqtt/983633) · [Report an issue](https://github.com/willhaggan/Sonoff-TRV2B-Ext-Sync-Blueprint/issues)

**Author:** whag

## What changed in V4

- **Zigbee2MQTT compatibility:** automatically detects the newer `remote_temperature` / `local_temperature` options and legacy `external` / `internal` options.
- **Corrected drift timing:** the temperature difference must persist for the configured duration. The readings can continue changing during that time.
- **Independent refresh:** sends the current valid average every 15 minutes, even when the temperature remains steady or within the deadband.
- **Fallback recovery:** handles `remote_source_offline` and legacy fallback states, sends a fresh temperature and restores external sensing when possible.
- **Invalid reading checks:** excludes unavailable, unknown, non-numeric, non-finite, unsupported-unit and out-of-range values.
- **Sensor report-age check:** optionally excludes sensors whose last report is too old.
- **Optional internal fallback:** can select the TRV's built-in sensor when no usable external readings remain.
- **Recovery triggers:** resynchronises after Home Assistant startup, automation reloads and entity recovery.
- **Improved logging:** records the requested temperature, number of valid sensors, reported mode and trigger reason. A mode-confirmation timeout is logged separately.

All original V3 input identifiers are retained.

## Requirements

- **Home Assistant 2024.4 or later.** The report-age check uses `last_reported` timestamps.
- A **SONOFF TRVZB** integrated through **Zigbee2MQTT**.
- The TRV's climate entity, temperature-source select entity and external-temperature number entity exposed in Home Assistant.
- At least one external temperature sensor available in Home Assistant.

External sensors can use a different integration or communication protocol from the TRV. They must expose a numeric temperature with a supported unit: **°C, °F or K**. The TRV external-input number entity must use **°C or °F**.

## Installation

1. Click the **Import blueprint** button above and complete the import in your Home Assistant instance.
2. Open **Settings → Automations & scenes → Blueprints** and select **Sonoff TRVZB Sensor Sync V4**.
3. Select the external sensors and the three entities belonging to the same TRV, as described below.
4. Review the settings and save the automation.
5. Run the automation's actions once to request an initial synchronisation. Check its trace and the TRV's reported temperature-source mode.

Repeat for each TRV. Several TRVs in the same room can use the same external sensor or group of sensors.

For manual import, use this blueprint address:

```text
https://github.com/willhaggan/Sonoff-TRV2B-Ext-Sync-Blueprint/blob/main/TRV2B_External_Sync
```

### Required entity selections

The entity IDs below are examples; choose the corresponding entities from your installation.

| Blueprint input | Select | Example |
| --- | --- | --- |
| Temperature Sensors | One or more external room temperature sensors | `sensor.living_room_temperature` |
| TRV Climate Entity | The radiator valve's climate entity | `climate.living_room_trv` |
| TRV Mode Select Entity | The TRV's `temperature_sensor_select` entity | `select.living_room_trv_temperature_sensor_select` |
| External Temperature Input Entity | The TRV's `external_temperature_input` number entity | `number.living_room_trv_external_temperature_input` |

The mode-select input controls the **temperature source**. Select the entity offering `local_temperature` / `remote_temperature` or `internal` / `external`.

## Settings

| Setting | Default | Behaviour |
| --- | --- | --- |
| Sustained Temperature Drift Duration | **60 seconds** | Normal drift must persist for this duration. Set to `0` for an immediate drift-triggered update. |
| Temperature Drift Threshold | **0.3°C** | Minimum absolute difference between the sensor average and the TRV's external-input value. The comparison uses Celsius regardless of the source units. |
| Enable Sync Logbook | **Enabled** | Records sync requests, reported mode and relevant recovery information. |
| Maximum Sensor Report Age | **120 minutes** | Excludes sensors whose latest Home Assistant entity report is older than this. Set to `0` to disable the age check. |
| Use Internal Sensor When All External Readings Are Invalid | **Enabled** | Attempts to select the built-in sensor when no external reading can be used. Temperature writes stop while no usable readings exist. |

The **15-minute refresh is fixed** and runs at **:00, :15, :30 and :45** each hour. It is independent of the drift threshold and duration.

### Choosing a sensor report age

Set the maximum age above your sensors' normal reporting interval. For example, a sensor reporting hourly needs a limit comfortably longer than one hour.

The check uses **`last_reported`**, which can advance when an unchanged value is reported. It does not use `last_changed`, so a steady room temperature can remain valid.

This timestamp describes reports received by Home Assistant. An integration republishing cached values can also refresh it. Configure source availability correctly so a disconnected thermometer does not remain indefinitely available with an old numeric reading.

## How synchronisation works

### Averaging and validation

Valid readings are converted to Celsius, equally averaged and rounded to one decimal place. If one sensor becomes unusable, the remaining valid readings continue to be used.

Readings outside **0–99.9°C** are excluded. The calculated write value is also checked against the number entity's advertised minimum and maximum. Genuine readings of **0°C and 1°C** are accepted.

### Sustained drift

Normal updates compare the average against the **last reported value of the external-temperature input entity**. The climate entity's `current_temperature` attribute is used for diagnostics rather than the drift calculation.

For example, with the default settings:

- The external input currently contains **20.0°C**.
- The sensor average reaches **20.4°C** and stays at least **0.3°C** away from that input for **60 seconds**.
- The blueprint sends the latest average, even if it changed during those 60 seconds.

If the difference drops below the threshold before the duration completes, the drift timer resets. Pending drift timers also reset when Home Assistant restarts or automations reload.

Startup, recovery, manual runs and the scheduled refresh can request a sync without waiting for the drift timer. They still check that usable temperature data exists.

### Temperature-source recovery

| Current Zigbee2MQTT value | Legacy value | Meaning handled by the blueprint |
| --- | --- | --- |
| `local_temperature` | `internal` | Built-in sensor selected; external sensing is requested when usable external data exists. |
| `remote_temperature` | `external` or `external_3` | External sensing selected or recovered. |
| `remote_source_offline` | `external_2` | External source has timed out; a fresh temperature is sent to attempt recovery. |

A fresh write may restore external sensing automatically. If needed, the blueprint explicitly selects the supported external option and waits up to **10 seconds** for the entity to report an external state. After forcing the mode, it recalculates and sends the latest valid average again.

`remote_source_offline` is handled as a fallback status. External sensing is selected using `remote_temperature` or the legacy `external` option.

If every external reading becomes unusable, temperature writes stop. With internal fallback enabled, the blueprint attempts to select the built-in sensor. Its dedicated all-invalid trigger waits **30 seconds**; other sync runs also check the current data and may act sooner. External sensing is requested again when usable readings return.

## Upgrading from V3

**The original GitHub URL and import button remain unchanged.** New imports receive the version currently published at that location. GitHub changes do not automatically update existing Home Assistant installations.

For an imported blueprint:

1. Open **Settings → Automations & scenes → Blueprints**.
2. Find this blueprint, open its **three-dot menu** and select **Re-import blueprint**.
3. Confirm the re-import.
4. Review the new settings on each automation using the blueprint, especially the report-age limit and internal fallback.
5. Run the automation's actions once and check its trace.

Existing automations still linked to the blueprint retain their entity selections and configured inputs because the V3 input identifiers are preserved. The new inputs use their defaults unless configured otherwise. Updating a shared blueprint affects every automation using that local blueprint file.

For a manually installed blueprint, replace the contents of its existing local file **without changing its filename or path**, then reload automations.

If you create a separate V4 automation, disable its V3 equivalent so both do not write to the same TRV. Automations previously converted using **Take control** are independent of the blueprint and are not updated by re-importing it.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| No temperature is sent | Check the selected entities, numeric readings, units, report ages, target range and automation trace. |
| The TRV reports `remote_source_offline` | Confirm the automation is enabled and the external-input number receives writes. Run the actions once and inspect the trace and Zigbee2MQTT logs. |
| A steady sensor is excluded | Inspect `last_reported` and the source's reporting behaviour. Increase the age limit above the normal report interval, or set it to `0` to disable age filtering. |
| The TRV remains on its internal sensor | Check whether any external reading is usable. With no valid readings, internal fallback may be operating as configured. |
| External mode cannot be confirmed | Check the selected temperature-source entity, available options, device availability and Zigbee2MQTT communication. |
| Mode or temperature keeps changing unexpectedly | Check for another automation or integration writing the temperature source or external input. Disable the old V3 automation if it is still running alongside V4. |
| Re-import is unavailable | The local blueprint may have no saved source URL. Update that existing local file manually while preserving its path. |

Logbook entries describe **service requests and entity-reported state**. MQTT entities may be optimistic, so those entries are not independent confirmation that the TRV received a fresh radio command.

## Support

Use the [GitHub issues page](https://github.com/willhaggan/Sonoff-TRV2B-Ext-Sync-Blueprint/issues) or the [Home Assistant Community topic](https://community.home-assistant.io/t/sonoff-trvzb-external-sensor-sync-with-external-mode-and-null-checks-zigbee2mqtt/983633).

Include your Home Assistant version, Zigbee2MQTT version, TRV firmware version, configured blueprint settings and a relevant automation trace. For sensor-age issues, also include the source integration and normal reporting interval.

See [Home Assistant's blueprint documentation](https://www.home-assistant.io/docs/automation/using_blueprints/) for importing, re-importing and managing blueprint automations.
