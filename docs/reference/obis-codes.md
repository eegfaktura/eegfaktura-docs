# OBIS Codes

OBIS (Object Identification System) codes identify energy-meter readings. eegfaktura uses a small canonical subset, with vendor extensions for the EEG-specific quantities (`G.01`, `G.02`, `G.03`, `P.01`). This page is the reference for which code carries which meaning, and how `energystore` maps each code into its on-disk slots.

## Per direction

### Consumer (CONSUMPTION)

| Code | Meaning | Source |
|------|---------|--------|
| `1-1:1.9.0 G.01` | Total consumption per measurement | measured |
| `1-1:1.9.0 G.01T` | `G.01 × Teilnahmefaktor / 100` | computed; **mandatory for billing periods from 2024-04-08** |
| `1-1:2.9.0 G.02` | Theoretically available share | computed by the network operator |
| `1-1:2.9.0 G.03` | Actually covered share from the EEG | computed; **billing-relevant** for consumers |
| `1-1:2.9.0 G.03R` | Subset of G.03 from renewable production | computed (sub-variant) |

### Producer (GENERATION)

| Code | Meaning | Source |
|------|---------|--------|
| `1-1:2.9.0 G.01` | Total production per measurement | measured |
| `1-1:2.9.0 G.01T` | `G.01 × Teilnahmefaktor / 100` | computed; mandatory for billing periods from 2024-04-08 |
| `1-1:2.9.0 P.01` | Surplus (legacy, pre-2024-04-08) | computed |
| `1-1:2.9.0 P.01T` | `P.01 × Teilnahmefaktor / 100` — surplus to the EVU | computed; billing-relevant for producers |

## Identities

- `G.01T = G.01 × Teilnahmefaktor / 100` (both directions)
- Consumer: `G.03 ≤ G.02` (covered ≤ offered) and `G.03 ≤ G.01` (covered ≤ consumed)
- Producer: `P.01T = G.01T − Σ G.03_all_consumers × (own_G.01T / Σ all_producers_G.01T)`
- Consumer residual grid draw = `G.01T − G.03`. Not in the energy-data report — taken from the EVU invoice.

## Billing-relevant values

- Consumer: **`Verbrauchertarif × G.03`** (own coverage)
- Producer: **`Erzeugertarif × (G.01T − P.01T)`** (amount delivered to the EEG)

## energystore storage mapping

energystore decodes each `meterCode` in an inbound `MqttEnergyMessage` into a fixed slot in its Badger store. The mapping:

| meterCode | Type | Source delta | Storage slot |
|-----------|------|--------------|--------------|
| `1-1:1.9.0 G.01` / `G.01T` | CON | 0 | `consumer[0]` = consumption |
| `1-1:2.9.0 G.02` | SHARE | 1 | `consumer[1]` = G.02 (`allocation` field in report) |
| `1-1:2.9.0 G.03` / `G.03R` | COVER | 2 | `consumer[2]` = G.03 (`utilization` field in report) |
| `1-1:2.9.0 G.01` / `G.01T` | GEN | 0 | `producer[0]` = production |
| `1-1:2.9.0 P.01` / `P.01T` | PLUS | 1 | `producer[1]` = P.01 (`allocation` field in report) |

### Direction detection

A new metering point is registered as a **producer** only if its first payload contains a `P.01` or `P.01T` code. A payload with only `G.01` is ambiguous — both directions use `G.01` — and the meter is then registered as a **consumer**.

This matters for sample-data generation: a producer's first MQTT publish must include a `P.01` value, even if zero, otherwise the meter ends up in the wrong direction.

## AllocDynamicV2 — pass-through

The `AllocDynamicV2` allocation in `energystore` does **no computation**. It filters matrix slots only. All three consumer values (`G.01`, `G.02`, `G.03`) and both producer values (`G.01`, `P.01`) must be pre-computed by the network operator and arrive populated.

Consequences:

- For mock / sample data, the publisher must produce all codes consistently. Missing G.02 / G.03 means consumers see zero EEG share in the customer SPA.
- For real data, this is what the network operator delivers via EDA — eegfaktura does not re-derive G.03 from raw G.01.

## Customer SPA chart mapping

The customer SPA's chart helper uses these field names internally:

```ts
// CONSUMPTION: dataKey "distributed" = EEG (green), "consumed" = EVU (purple)
allocated: utilization[i],                  // = G.03  → EEG bar
consumed:  consumption[i] - utilization[i], // = G.01 - G.03 → EVU bar (residual draw)

// GENERATION
allocated: production[i] - allocation[i],   // = G.01 - P.01 → EEG bar (delivered to EEG)
consumed:  allocation[i],                   // = P.01 → EVU bar (residual surplus)
```

The field names are misleading: in the TS code, `allocated` is the **EEG portion** (the green bar in the UI, the billing-relevant share), not "what was allocated to the operator". Read carefully when modifying the chart code.

## OBIS code format

The OBIS format used here is the abbreviated form. A full OBIS code is
`A-B:C.D.E*F` where:

- `A` = medium (1 = electricity)
- `B` = channel
- `C` = measurand
- `D` = quantity
- `E` = storage / processing
- `F` = vendor extension

eegfaktura uses the abbreviated `<A-B>:<C.D.E>` prefix followed by a vendor token (`G.01`, `G.02`, `G.03`, `P.01`, `G.01T`, `P.01T`, `G.03R`).

## Related

- [Architecture / Messaging](../architecture/messaging.md) — energy-data MQTT pipeline
- [services/energystore](../services/energystore.md) — Badger storage internals
- [Glossary](glossary.md) — Teilnahmefaktor, EVU, EEG terminology
