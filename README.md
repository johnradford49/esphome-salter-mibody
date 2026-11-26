# Salter MiBody scale → Home Assistant (ESPHome)

Reads weight, body fat, body water and muscle from a **Salter MiBody Bluetooth
analyser scale (9159 BK3R)** using an ESP32 running ESPHome, and publishes them to
Home Assistant. No phone, app or cloud needed.

The scale is made by Transtek (Lifesense) and uses their "A2/A3" Bluetooth
protocol, so other Transtek-built scales with service `0x7882` (or the
related `0x78B2`) may work too.

## What you get

| Entity | Unit |
|---|---|
| Salter weight | kg |
| Salter body fat | % |
| Salter body water | % |
| Salter muscle | % |
| Salter user | scale user number |
| Salter connected | connectivity (diagnostic) |
| Salter raw 8A21 / 8A22 / 8A82 | hex packets (diagnostic, disabled by default) |

The scale sends no bone mass on this model (that field is always zero).

## Requirements

- An ESP32 with Bluetooth (ESP32, ESP32-C3, ESP32-S3) running ESPHome. An existing
  Bluetooth proxy works fine; the scale only holds a connection for a few seconds.
- Home Assistant with the ESPHome integration.
- Your scale's Bluetooth address (see below).

## Setup

1. **Find the scale's MAC address.** Hold the **UNIT** button on the base of the scale
   until the pairing symbol shows. While it is in pairing mode it advertises a
   6-digit name starting with `1` (e.g. `113196`) and service `0x7882`. Look for it in
   Home Assistant (*Settings → Devices & services → Bluetooth → Advertisement
   monitor*) or with a phone app such as nRF Connect.
2. **Add the package** to your ESPHome device, that is closest to the scale, and set your values. Values set in your
   own `substitutions:` override the defaults in `salter.yaml`:

   ```yaml
   substitutions:
     scale_mac: "AA:BB:CC:DD:EE:FF"
     user_name: "Me"          # up to 16 characters
     user_age: "40"           # 18-100
     user_height_cm: "175"    # 100-204
     user_sex: "1"            # 1 = male, 2 = female

   packages:
     salter:
       url: https://github.com/johnradford49/esphome-salter-mibody
       ref: main
       files: [salter.yaml]
   ```

   > **If your device YAML already has a `substitutions:` section, add these lines to
   > it.** A second `substitutions:` section is silently ignored. The package then falls
   > back to its placeholder address `AA:BB:CC:DD:EE:FF` and never connects to the scale.
   > Run *Validate* in ESPHome and check that `scale_mac` shows your scale's address.

   Or copy `salter.yaml` into the same folder as your device config and use
   `salter: !include salter.yaml`. A complete stand-alone example is in
   [`examples/esp32-c3-bridge.yaml`](examples/esp32-c3-bridge.yaml).
3. **Install** the firmware.
4. **Pair once:** hold **UNIT** until the pairing symbol shows. The ESP32 connects,
   claims the first free user slot on the scale (the log says `Binding to user slot N`)
   and stores your profile on the scale. The display shows `[ ]` when pairing succeeds.

## Weighing

Each time:

1. Wake the scale, press the center button and select **your user number** (the slot the ESP32 claimed,
   usually 1) with the arrow buttons.
2. Wait for the display to show zero, then step on **barefoot**.

The scale connects to the ESP32 after the reading and the values appear in Home
Assistant a few seconds later.

- Just stepping on without selecting a user gives a weight-only reading, which the
  scale neither stores nor sends.
- A user number that was paired with the Salter app on a phone still goes to that
  phone. Use the slot the ESP32 claimed.
- If the ESP32 is offline, the scale keeps the reading and sends it at the next sync.
  Home Assistant records it at the time it arrives.
- Shoes or socks block the impedance measurement, so only weight is sent.

## Troubleshooting

- Enable the raw packet sensors and watch the ESPHome log (`salter` tag).
- `Salter connected` shows whether the ESP32 currently has a link to the scale.
- Only run **one** ESP32 with this package per scale. Two bridges with the same pairing
  race each other for the connection.
- Pairing again from another app (or another ESP32) replaces the stored pairing; just
  pair again with **UNIT**.

## How it works

See [`docs/protocol.md`](docs/protocol.md) for the packet-level description.

## Credits

Protocol details were worked out with help from:

- [openScale](https://github.com/oliexdev/openScale) wiki notes on the Medisana BS444
  and Trisa Body Analyze (same Transtek characteristics).
- [openble-lifesense](https://github.com/kadima-tech/openble) — Lifesense A2/A3 pair
  and sync handshake, recovered from the Philips Health app.

Not affiliated with or endorsed by Salter, Transtek or Lifesense.

## License

MIT
