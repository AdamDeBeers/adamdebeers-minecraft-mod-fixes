# OpenComputers Rebooted — Create: Crafts & Additions Accumulator driver

New OC integration driver against [CaitlynMainer/OpenComputers](https://github.com/CaitlynMainer/OpenComputers),
branch `main-MC1.21.1` ("OpenComputers: Rebooted", the NeoForge 1.21.1 port),
adding support for [Create: Crafts & Additions](https://github.com/mrh0/createaddition)'
Modular Accumulator block. Upstream PR:
[CaitlynMainer/OpenComputers#55](https://github.com/CaitlynMainer/OpenComputers/pull/55).

## What it adds

A new OC component, **`cca_accumulator`**, exposed when an OC **Adapter** is
placed directly adjacent to a Modular Accumulator block (any part of the
multiblock). No Inventory Controller Upgrade is required — this is a direct
block driver, not an inventory-slot driver.

## Exposed methods (all read-only)

| Method | Returns |
|---|---|
| `getEnergyStored()` | current energy stored, in FE |
| `getMaxEnergyStored()` | max energy capacity, in FE |
| `getEnergyPercent()` | current charge, 0–100 |
| `getMaxInsert()` | max energy transfer per side, in FE/t |
| `getMaxExtract()` | max energy extraction per side, in FE/t |
| `getHeight()` | multiblock height, in blocks |
| `getWidth()` | multiblock width, in blocks |

These mirror the mod's own CC:Tweaked peripheral (`ModularAccumulatorPeripheral`)
one-to-one.

## Implementation notes

- `DriverAccumulator.java` — mirrors the existing `DriverController`/
  `ManagedBlockEntityEnvironment` pattern already used for AE2's controller
  block: a `DriverSidedBlockEntity` matched on
  `com.mrh0.createaddition.blocks.modular_accumulator.ModularAccumulatorBlockEntity`,
  with an `EnvironmentProvider` gated on the accumulator's item.
- Energy and percent charge are read through the multiblock's controller part
  (`ModularAccumulatorBlockEntity.getControllerBE()`) — the same indirection
  the mod's own ComputerCraft peripheral uses, since only the controller part
  actually tracks energy; every other part of the multiblock forwards to it.
- `getMaxInsert`/`getMaxExtract` come from the mod's static server config
  (`CommonConfig.ACCUMULATOR_MAX_INPUT`/`ACCUMULATOR_MAX_OUTPUT`), not
  per-instance state — the block has no per-side transfer rate to read.
- `ModCreateAddition.java` — the `ModProxy` gate, registered as a soft
  dependency through the existing `Mods.scala`/`Proxies` mechanism
  (`SimpleMod` + `ModProxy`), the same pattern used for the AE2 and Mekanism
  integrations: `compileOnly`/`runtimeOnly` on the mod jar in `build.gradle`
  (pinned to `maven.modrinth:createaddition:neoforge-1.21.1-1.6.0`), with
  `initialize()` only running if `createaddition` is actually loaded.
- No entry was added to `neoforge.mods.toml` — checked, and none of the
  existing third-party integrations (AE2, Mekanism, JEI) declare one there
  either, so this follows the established convention.

## Files

- [`DriverAccumulator.java`](DriverAccumulator.java) — the driver, environment, and provider (new file)
- [`ModCreateAddition.java`](ModCreateAddition.java) — the mod-integration gate (new file)
- [`wiring.diff`](wiring.diff) — the accompanying changes to `Mods.scala`, `build.gradle`, and `gradle.properties` that register the driver and add the soft dependency

## Testing

Tested locally against NeoForge 1.21.1 + Create: Crafts & Additions + a
locally built `opencomputers-1.9.4-dev.local` jar. Confirmed:

- `getEnergyStored`/`getMaxEnergyStored`/`getEnergyPercent` update live as
  the accumulator charges.
- `getHeight`/`getWidth` correctly report multiblock size, verified on both
  a 1x1 and a 4x2 configuration.
