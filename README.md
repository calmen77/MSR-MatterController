# MatterController for Reactor

A [Reactor](https://reactor.toggledbits.com/) controller plugin that turns Reactor into a **Matter
controller**. It commissions Matter devices (Thread, Wi-Fi, or Ethernet based) using Matter's
Multi-Admin feature, creates Reactor entities for them, and lets you read/control them like any
other Reactor entity.

Built on [matter.js](https://github.com/matter-js/matter.js) (`@matter/main` /
`@matter/protocol`), which implements the **Matter 1.6.0** specification.

---

## Features

- Commission devices directly from the Reactor UI via Multi-Admin — pair the device to your existing ecosystem 
  (Apple Home, Google Home, Home Assistant, SmartThings, …) first, then add Reactor as an additional controller 
  using the 11‑digit pairing code.
- Automatic entity creation for each Matter endpoint, with live attribute updates
  (subscriptions, not polling).
- Broad device-type coverage — lights (on/off, dimmable, color temperature, extended color),
  plugs, locks, sensors, window coverings, and more. See [Supported device types](#supported-device-types).
- Bridge / aggregator support (e.g. Matterbridge, Aqara hubs) — every sub-device behind a bridge
  gets its own Reactor entity with its own name, read from the bridge's per-device label.
- Clean device removal (`x_matter.remove`) — properly decommissions the device so it can be
  re-paired elsewhere, with a `force` fallback for devices that are no longer reachable.
- System-entity status reporting (`x_matter_controller.status`) so commissioning success/failure
  is visible directly in the Reactor UI, with a descriptive `last_error` on failure.

---

## Requirements

- IPv6 on Reactor host
- Node.js **>= 18**
- A Matter-certified device that supports Multi-Admin commissioning (virtually all consumer Matter
  devices do)
- For Thread devices: a working Thread Border Router on your network and IPv6 routing between
  Reactor's host and the Thread mesh (see [Thread networking](#thread-networking) below)

---

## Installation

1. Download the release tarball (e.g. `MatterController-v26255.tar.gz`) and extract it into your
   Reactor instance's `config/ext/MatterController/` directory (create the directory first if it
   doesn't exist yet). The tarball contains the plugin files directly at its root — no top-level
   folder — so extracting it straight into `MatterController/` works as expected:

   ```bash
   mkdir -p config/ext/MatterController
   tar xzvf MatterController-v26255.tar.gz -C config/ext/MatterController
   ```

2. Run the installer:

```bash
cd config/ext/MatterController
./install.sh
```

This checks your Node.js/npm versions, runs `npm install` for the pinned `@matter/main` /
`@matter/protocol` dependencies, and verifies the plugin's syntax. Run `./install.sh --check` to
only verify prerequisites without installing anything.

Then add the controller to `reactor.yaml`:

```yaml
controllers:
  - id: matter_controller
    enabled: true
    implementation: MatterController
    name: Matter
    config:
      # optional: Matter protocol port (default: 5541)
      port: 5541
      # optional: label shown in the Matter fabric list on devices (default: MatterController)
      fabric_label: MatterController
      # optional: controller name in Matter metadata (default: Reactor)
      controller_name: Reactor
```

Restart Reactor after editing `reactor.yaml`.

---

## Commissioning a device

1. Put the device into pairing mode (usually a long button press — check the manufacturer's
   instructions). If it's already paired to another ecosystem (Apple Home, Google Home, Home
   Assistant, …), use that app's "Add to more apps / hubs" (or equivalent) flow to generate a new
   11‑digit pairing code — the device does **not** need to be removed from its existing app.
2. In the Reactor UI, find the Matter controller's **system entity** and run the
   `x_matter_controller.commission` action with:
   - `code` — the 11‑digit pairing code (hyphens allowed)
   - `name` *(optional)* — device name; only meaningful for single-device products, since
     bridges/hubs name each sub-device automatically
   - `split_endpoints` *(optional, default `false`)* — if `true`, an endpoint with no device type
     of its own (e.g. a battery reported on its own sibling endpoint) becomes a separate entity
     instead of being merged into the main device
3. Watch the `x_matter_controller.status` attribute on that system entity: `pending` while
   commissioning, then `ok` once done (or `error`, with details in
   `x_matter_controller.last_error`, if it failed).
4. Reload the Reactor UI page once status is `ok` to see the new entity/entities.

**Vendor ID:** the controller identifies itself to devices using matter.js's official
**"Test Vendor 1" / "Matter Test" vendor ID (0xFFF1 / 65521)**, reserved by the Matter
specification for test/development controllers. You may see this listed as the vendor for
Reactor's fabric in an app like Apple Home's "Apps with Access" — this is expected and does not
affect functionality.

## Removing a device

Deleting the entity in the Reactor UI only removes the Reactor-side object — the underlying Matter
pairing is left intact and the entity reappears on the next restart. Instead, run the `x_matter.remove`
action **on the device entity**:

```
x_matter.remove { force: false }
```

- Without `force`: properly notifies the device to forget Reactor's fabric (it can be re-paired
  elsewhere immediately afterward).
- With `force: true`: removes Reactor's local record only, without contacting the device — use
  this only when the device is unreachable/dead; you'll likely need to factory-reset it before
  re-pairing elsewhere.

All Reactor entities belonging to that device/node (e.g. every sub-device of a bridge) are removed
in one call.

---

## Supported device types

| Category | Device types |
|---|---|
| Lighting | On/Off, Dimmable, Color Temperature, Extended Color (RGB), Mounted On/Off & Dimmable Controls |
| Plugs/outlets | On/Off Plug, Dimmable Plug |
| Access | Door Lock |
| Climate | Thermostat*, Room Air Conditioner*, Window Covering†, Water Valve† |
| Air | Fan*, Air Purifier*, Air Quality Sensor† |
| Energy | EV Charger*† |
| Sensors | Temperature, Humidity†, Pressure†, Flow†, Light (Illuminance)†, Occupancy (motion), Contact (door/window), On/Off, Smoke/CO Alarm†, Water Leak†, Water Freeze†, Rain |
| Controls | Generic Switch/Button† |

`*` Read-only / partial support at present.

`†` **Untested** — implemented from the Matter specification only; not yet verified against a
real device. Should work, but treat with extra caution and please report back if you try one of
these.

---

## Thread networking

Thread-based devices communicate over their own IPv6 mesh network, separate from your regular
Wi-Fi/Ethernet LAN. Reactor's host needs an IPv6 route to that mesh via a Thread Border Router
(most HomePods, Apple TVs, and many Wi-Fi routers/hubs can act as one). If commissioning or control
of a Thread device times out with no response, this route is almost always the cause — see the
plugin's technical documentation for detailed diagnosis steps.

Wi-Fi and Ethernet based Matter devices don't need any of this — they're reachable directly.

---

## Troubleshooting

- **Commissioning fails with an invalid/incorrect pairing code error** — double-check the
  11‑digit code; a checksum validation catches typos before any network activity happens.
- **Device turns on physically right after pairing** — this is normal for some bulbs (a
  manufacturer-side "successfully joined" indicator); it's independent of the logical on/off
  state Reactor and other apps read, which correctly reports `false`.
- **`x_matter_controller.status` stuck at `pending` or showing an old error** — resolved
  automatically on the next Reactor (re)start or a `sys_system.restart` action on the system
  entity; both fully reset the controller's status attributes.
- Check `logs/reactor.log` for detailed error messages from the plugin; Thread-specific network
  issues are usually more visible via `journalctl -u <your-reactor-service-name>` (matter.js debug
  output) — the systemd unit name depends on how Reactor was installed/named on your host.

---

## License

MIT. See [package.json](package.json) for homepage/repository details.
