# F5000LFP expansion bus — protocol notes

Captured passively on a Pecron F5000LFP (BMS firmware 1.0.3), November 2026.

## Physical

- RS485 on the two left signal pins of the XT120 2+4 expansion port.
- 115200 baud, 8 data bits, no parity, 1 stop bit.
- Modbus-RTU style frames, CRC-16/Modbus (poly 0xA001, init 0xFFFF, little-endian on the wire).

## Traffic pattern (~3 s cycle)

```
00 03 82 02 00 32 4C 76     poll slot 00   (no reply)
01 03 82 02 00 32 4D A7     poll slot 01   (no reply)
...                          slots 02-05    (no reply; expansion-pack slots)
FF 03 82 02 00 32 58 79     poll internal BMS
FF 03|04 32 <50 bytes> <crc> BMS reply (page alternates)
FE 10 00 00 00 11 22 <34 bytes> <crc>   identity broadcast (no reply)
```

Polls are ~200 ms apart; there is a quiet gap of ~1 s after the broadcast.

### Deviations from standard Modbus

- Slave addresses `00` (Modbus broadcast) and `FF`/`FE` are outside the valid 1–247 range.
- The reply's byte count is `0x32` (50) for a request of 50 *registers*; standard Modbus
  would return 100 bytes.
- The reply's function byte alternates `03` / `04` for identical `03` requests. It works as a
  **page number**, not a function code.

## Page 03 — live data (50 bytes)

| Offset | Size | Meaning |
|---|---|---|
| 0–31 | 16 × u16 BE | Cell 1–16 voltage, mV |
| 32–33 | u16 BE | Pack voltage, mV |
| 34–36 | 3 × u8 | Temperatures, °C + 50 |
| 37 | u8 | Current direction: 1 = charge, 0 = discharge |
| 38–39 | u16 BE | Current magnitude, 0.01 A |
| 40 | u8 | SOC, % |
| 41 | u8 | Charging flag (toggles with charge-current pulses at full) |
| 42 | u8 | Unknown (19 constant so far) |
| 43 | u8 | Unknown (36 constant so far) |
| 44–48 | u8 | Always 0 so far |
| 49 | u8 | Unknown; increments slowly at rest (6 → 11 over ~20 min after full) |

## Page 04 — firmware, temperatures, second current (50 bytes)

| Offset | Size | Meaning |
|---|---|---|
| 0–7 | ASCII | BMS firmware, e.g. `1.0.3`, zero-padded |
| 8–18 | 11 × u8 | Temperatures, °C + 50 |
| 19 | u8 | Direction of current 2 (1 = charge) |
| 20–21 | u16 BE | Current 2 magnitude, 0.01 A |
| 22–49 | — | Always 0 so far |

## Identity broadcast (`FE 10 0000 0011 22`, 34 data bytes)

| Offset | Meaning |
|---|---|
| 0–15 | Model, ASCII, zero-padded (`F5000LFP`) |
| 16–17 | `01 0C` (unknown — possibly version or pack count) |
| 18–29 | Device key, ASCII (`F8D8117CB639` = the station's Wi-Fi MAC) |
| 30–33 | Zero |

## Active probing results (read-only, function 03, sent in the post-broadcast gap)

- Requests to `FF` with register `xx02` (`8002`, `8102`, `8202`, `8302`) and length `0x32`
  get the next page in the 03/04 alternation. The **high byte is ignored**.
- Lengths other than `0x32`, registers `xx00`–`xx0F` other than `xx02`, and function `04`
  requests get **no reply**.
- No configuration or settings page was found. The low byte appears to act as a command
  (`02` = status). Unknown command bytes were deliberately not explored further because
  they could trigger actions.
- Probing did not disturb the head unit's polling (0 junk bytes, no missed polls).

## Observations

- Idle draw with DC output on, AC off: about 0.32–0.38 A from the pack (≈ 18–21 W).
- At full charge the station holds ≈ 56.76 V (3.548 V/cell average) and tapers current from
  ~11.5 A to ~1 A in about 12 minutes, then pulses ~0.76 A on and off.
- No sign of passive balancing at the top of charge or at rest; the cell spread relaxed from
  149 mV to 87 mV over 20 minutes with no current.
