# adamdebeers-minecraft-mod-fixes

Patched and added source files for Minecraft mods, kept here for reference
and as a base for upstream PRs.

## Contents

- [`opencomputers-rebooted-ae2-fixes/`](opencomputers-rebooted-ae2-fixes/) —
  four fixes for [CaitlynMainer/OpenComputers](https://github.com/CaitlynMainer/OpenComputers)
  (branch `main-MC1.21.1`, "OpenComputers: Rebooted" for NeoForge 1.21.1) that
  were breaking the Applied Energistics 2 integration (`me_controller`,
  `inventory_controller`).
- [`opencomputers-rebooted-createaddition-driver/`](opencomputers-rebooted-createaddition-driver/) —
  a new integration driver for the same OpenComputers fork, adding support
  for [Create: Crafts & Additions](https://github.com/mrh0/createaddition)'
  Modular Accumulator block as an OC component (`cca_accumulator`).
