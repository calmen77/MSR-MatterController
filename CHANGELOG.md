# Changelog

## 26295

- `split_endpoints: true` now also keeps a sibling-merged wrapper's own PowerSource (battery/voltage) and disjoint-capability merges (e.g. a light + its own metering endpoint) as separate entities, instead of still merging them. Generic Switch button-pair merging is unaffected.

## 26294

- README: documented docker-container install steps.
- Added `x_matter_controller.version` attribute (plugin's own build version). `matter_capabilities.yaml` revision 4→5.
- `x_matter.bridge_name` now formats as `<vendorName> (<firmwareVersion>) <label>`, preferring `productLabel` when available.
- Bridged devices whose siblings expose complementary (non-overlapping) capabilities under the same per-device wrapper are now merged into a single Reactor entity, but only when exactly one sibling qualifies — ambiguous cases stay separate.
- A wrapper endpoint's own functional device type (e.g. a switch module's own temperature sensor) is now folded into its children's merged entity under the same rule, instead of always becoming a separate entity.

## 26283

- For bridged sub-devices (e.g. behind Matterbridge), `vendor_name`/`product_name`/
  `firmware_version` now reflect the actual physical device instead of the bridge itself.
- Added `x_matter.bridged` (bool) and `x_matter.bridge_name` (string, bridge name + firmware)
  attributes.
- `x_matter.available` now also reflects a bridged sub-device's own per-device reachability
  (e.g. a specific Zigbee device going offline behind an otherwise-online bridge), not just
  whether the whole bridge/node is online.

## 26282

**New capabilities / device support**
- Added mains power/voltage/current/energy metering support (`power_sensor`, `voltage_sensor`,
  `current_sensor`, `energy_sensor`) for devices exposing Matter's ElectricalPowerMeasurement/
  ElectricalEnergyMeasurement clusters (e.g. metering smart plugs).
- Added `hvac_control` and `hvac_heating_unit`/`hvac_cooling_unit` support for Thermostat/Room Air
  Conditioner devices, including `set_mode`/`set_setpoint` write actions. Heating/cooling
  capabilities are now added per device based on what it actually supports.
- Added full `hs_color` (hue/saturation) support for color lights, including `set_hs`, `set_hue`,
  and `set_saturation` write actions.
- Added a new `light_effect` capability (`start`/`stop`) for triggering Blink/Breathe/Okay/
  Channel Change visual effects, for lights that support Matter's Identify cluster.
- Added temperature support for thermostats without a separate temperature sensor cluster.

**Fixes**
- Fixed entity naming/merging for bridges that label their own sub-devices (e.g. Matterbridge).
- Fixed endpoints reporting multiple device types silently losing all but one capability.
- Fixed a missing device type mapping for some bridge-exposed relay/switch controls.
- Fixed sibling entities under a shared wrapper getting ambiguous/duplicate names.
- Fixed a metering relay/plug losing its controllable on/off identity in favor of a read-only
  sensor entity.
- Fixed `motion_sensor` never reflecting real motion/occupancy changes.
- Fixed missing units on temperature/humidity/pressure/voltage/power/current/energy/light/air
  quality sensor readings, on both initial sync and live updates.
- Fixed incorrect pressure sensor unit conversion (was kPa, now hPa).
- Power/current sensor readings on a metering relay/plug now reset to 0 when it turns off, instead
  of showing a stale last-known value.
- Fixed `button.state` not updating for Generic Switch (button) devices; single/double/hold/
  release presses are now reported correctly.
- Fixed `leak_detector`/`freeze_detector` not updating their state.
- Fixed `light_effect` being incorrectly added to plain on/off plugs/relays (which only implement
  the Identify cluster for commissioning, with no actual dimmable/visual output) — now only added
  to real light device types.
- Changed Dimmable Light's primary attribute to on/off state instead of brightness level.

**UI / actions**
- Hidden several actions from the UI that have no corresponding Matter command and would
  otherwise silently do nothing: `energy_sensor.reset`, button press/click actions (buttons are
  read-only), EV charger control actions (EV charger support is currently read-only), and
  `light_effect` speed/duration (Matter's effect command has no such parameters).

## 26255

Initial release.
