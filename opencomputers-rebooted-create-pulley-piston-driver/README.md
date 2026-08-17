# OpenComputers Rebooted — Create pulley, piston, and bearing drivers

New OC integration drivers against [CaitlynMainer/OpenComputers](https://github.com/CaitlynMainer/OpenComputers),
branch `main-MC1.21.1` ("OpenComputers: Rebooted", the NeoForge 1.21.1 port),
adding direct block-adjacent support for five vanilla [Create](https://github.com/Creators-of-Create/Create)
blocks. Upstream PR: [CaitlynMainer/OpenComputers#56](https://github.com/CaitlynMainer/OpenComputers/pull/56).

These blocks previously had no OpenComputers support at all — the only way
to reach them was indirectly, via Create: Crafts & Additions' Digital
Adapter, as CC:Tweaked peripherals. There is no OC equivalent of the
Digital Adapter's hub model here: an OC Adapter is placed directly against
each block, the same way `Create_Speedometer`/`Create_Stressometer` and
every other Create-native driver in this codebase already works.

## Components added (all read-only except `gotoElevatorFloor`)

| Component | Block | Methods |
|---|---|---|
| `Create_ElevatorPulley` | Elevator Pulley | `getPulleyDistance()`, `getElevatorFloor()`, `getElevatorFloors()`, `getElevatorFloorName(index)`, `hasElevatorArrived()`, **`gotoElevatorFloor(index)`** (mutating) |
| `Create_MechanicalBearing` | Mechanical Bearing | `getBearingAngle()` |
| `Create_RopePulley` | Rope Pulley | `getPulleyDistance()` |
| `Create_HosePulley` | Hose Pulley | `getPulleyDistance()` |
| `Create_MechanicalPiston` | Mechanical Piston (regular and sticky) | `getPistonDistance()`, `isSticky()` |

## Implementation notes

- All five are `CreateEnvironment<T>` subclasses in a new file,
  `CreateContraptionEnvironments.java`, each matched on its concrete
  block-entity type via `CreateBlockDriver<>` in `CreateDrivers.java` — the
  same pattern already used for `Create_Speedometer`/`Create_Stressometer`/etc.
- **Class hierarchy** (confirmed via direct bytecode inspection of the
  Create 6.0.10 jar, not guesswork): `ElevatorPulleyBlockEntity extends
  PulleyBlockEntity` (the Rope Pulley's own class). A driver matched naively
  on `PulleyBlockEntity` would therefore also fire on real elevator pulleys
  via `instanceof`, producing a redundant `Create_RopePulley` alongside
  `Create_ElevatorPulley` on the same block. The `RopePulley` driver is
  registered with a predicate excluding `ElevatorPulleyBlockEntity`
  instances — mirroring the `Packager`/`RepackagerBlockEntity` exclusion
  pattern already present elsewhere in this file — to avoid that.
- `MechanicalPistonBlockEntity` is a sibling of `PulleyBlockEntity` under a
  shared `LinearActuatorBlockEntity` parent (explains why both expose
  `getInterpolatedOffset`). `HosePulleyBlockEntity` extends
  `KineticBlockEntity` directly — an unrelated branch entirely. Kept as
  three separate driver classes rather than one shared driver, since the
  blocks are genuinely distinct types and only their single offset-reading
  method happens to share a signature.
- **Sticky piston distinction**: Sticky Mechanical Piston and regular
  Mechanical Piston share the exact same `MechanicalPistonBlockEntity`
  class — confirmed via bytecode inspection of `AllBlocks`/
  `AllBlockEntityTypes`, both variants are `BlockEntry<MechanicalPistonBlock>`
  sharing one block-entity type. The sticky/regular distinction lives on the
  registered `Block` instance (`create:sticky_mechanical_piston` vs.
  `create:mechanical_piston`), not the block entity, so `isSticky()` reads
  it via Create's own `MechanicalPistonBlock.isStickyPiston(BlockState)`
  helper rather than needing a second driver class.

## Files

- [`CreateContraptionEnvironments.java`](CreateContraptionEnvironments.java) — all five driver classes (new file)
- [`CreateDrivers.diff`](CreateDrivers.diff) — the registration diff wiring them in

## Testing

Tested locally against NeoForge 1.21.1 + Create + a locally built
`opencomputers-1.9.4-dev.local` jar. Confirmed:

- All getters live-update correctly in-game (elevator floor tracking,
  bearing angle, pulley/piston offsets).
- `gotoElevatorFloor` correctly moves the elevator and returns the
  target-Y delta.
- `isSticky()` returns `true` on a sticky mechanical piston and `false` on
  a regular one — verified with both idle/retracted, since Create swaps in
  a different block entity mid-animation (same as vanilla pistons), which
  made idle state necessary for a meaningful test.
