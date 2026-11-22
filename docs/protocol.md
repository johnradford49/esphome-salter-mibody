# Salter MiBody 9159 Bluetooth protocol

Observed on a Salter MiBody 9159 BK3R (Transtek/Lifesense "A2/A3" protocol).
All multi-byte values are little-endian.

## GATT

| UUID | Properties | Use |
|---|---|---|
| Service `0x7882` | | Weight scale service |
| `0x8A21` | indicate | Weight measurement |
| `0x8A22` | indicate | Body composition |
| `0x8A20` | read | Feature (unused) |
| `0x8A81` | write | Commands, host → scale |
| `0x8A82` | indicate | Commands, scale → host |

Device Information (`0x180A`) is also present. No Bluetooth bonding or PIN is used.

## Advertising

- Name is a 6-digit number. In pairing mode it starts with `1` (e.g. `113196`).
- Service UUID `0x7882`, no manufacturer or service data. The weight is **not** in the
  advertisement; a connection is required.
- The scale only advertises after a weigh-in when a user number was selected, or in
  pairing mode (hold **UNIT** on the base).

## Pairing (scale in pairing mode)

| Direction | Packet | Meaning |
|---|---|---|
| scale → | `A0 pp pp pp pp` | 4-byte password |
| → scale | `21 bb bb bb bb` | broadcast id chosen by the host |
| scale → | `A1 rr rr rr rr` | random challenge |
| → scale | `20 xx xx xx xx` | `random XOR password`, byte by byte |
| scale → | `83 nn <16-byte name, space padded> 00 00` | one per user slot offered for binding |
| → scale | `03 nn <16-byte name, space padded>` | bind to slot `nn` |
| → scale | `51 DF nn sex age hh hh 00 00 00 00 00 FF 00 00` | user profile (see below) |
| → scale | `02 tt tt tt tt` | time: local wall-clock seconds since 2010-01-01 00:00 |
| → scale | `22` | end session |

Profile: `sex` 1 = male, 2 = female; `age` in years; height `hh hh` = `0xE000 + cm`;
bytes 9–12 are goal weight (`00 00 00 FF` = 0.0).

## Sync (after a weigh-in with a user selected)

| Direction | Packet |
|---|---|
| scale → | `A1 rr rr rr rr` |
| → scale | `20 <random XOR password>` |
| → scale | `21 <broadcast id>` |
| → scale | `02 <time>` |
| scale → | `C0 …` (scale recognised the paired user) |
| → scale | `51 …` profile |
| scale → | `8A21` weight packets (stored history first, newest last) |
| scale → | `8A22` body composition packets |
| → scale | `22` (sent here 3 s after the last measurement) |

## Weight packet (8A21, 20 bytes)

`flags | weight(4) | [timestamp(4)] | [resistance1(4)] | [resistance2(4)] | [user(1)] | [status(1)] | 00`

- Flags: bit0 timestamp, bit1 resistance 1, bit2 resistance 2 (impedance), bit3 user,
  bit4 status present; bits 5–6 unit (0 = kg, 1 = lb, 2 = st).
- Weight and resistances: 3-byte unsigned mantissa + signed 8-bit decimal exponent.
  `B4 1E 00 FE` → 7860 × 10⁻² = 78.60 kg.
- Timestamp: seconds since 2010-01-01, local time.
- Status bit0 = stable.

Example: `1F B4 1E 00 FE B3 8B C5 1F 00 00 00 FF 00 00 00 FF 01 09 00`
→ 78.6 kg, 2026-11-22 10:40:51, no impedance, user 1, stable.

## Body composition packet (8A22, 20 bytes)

`flags | timestamp(4) | user(1) | fat(2) | water(2) | muscle(2) | 00 F0 | 00 …`

Values are 16-bit SFLOAT (12-bit signed mantissa, 4-bit signed exponent):
`CC F0` → 0x0CC × 10⁻¹ = 20.4 %.

Example: `6D 49 A1 C5 1F 01 CC F0 66 F2 77 F1 00 F0 …`
→ fat 20.4 %, water 61.4 %, muscle 37.5 %, verified against the scale display.

Note that the Medisana BS444 puts kcal first and fat/water/muscle/bone after it; this
Salter model starts directly with fat and has no bone value.
