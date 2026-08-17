# OpenComputers: Rebooted — AE2 integration fixes

Four fixes against [CaitlynMainer/OpenComputers](https://github.com/CaitlynMainer/OpenComputers),
branch `main-MC1.21.1` ("OpenComputers: Rebooted", the NeoForge 1.21.1 port),
commit `f3438a427ba46c7c88a5835f4edcbd0c750442e6`, found while debugging a
crash and a stub-looking result when using OpenComputers' Applied Energistics 2
integration (`me_controller`, `inventory_controller`) against AE2 `19.2.17`.

Each file below is the *whole file, patched* — drop it in at the same path
(`src/main/scala/li/cil/oc/...`) to apply the fix, or diff it against upstream
to produce a patch.

## 1. `ConverterItemStack.scala` — `scala.MatchError: null`

The actual crash. Two `match` expressions have no `null`/catch-all case:

```scala
stack.get(DataComponents.LORE) match {
  case lore: ItemLore => { ... }
}

stack.getCapability(Capabilities.EnergyStorage.ITEM) match {
  case storage: IEnergyStorage => { ... }
}
```

`stack.get(DataComponents.LORE)` and `stack.getCapability(...)` both return
`null` for the overwhelming majority of items (any item with no lore, any
item without an energy capability — including every AE2 item). Since these
run unconditionally on **every** `ItemStack` passed through
`Registry.convert` (which `inventory_controller` calls on every slot it
reads), this threw `scala.MatchError: null` on almost any read.

**Fix:** added `case _ =>` to both matches.

## 2. `Registry.scala` — `convertMap` non-exhaustive match

```scala
def convertMap[K, V](obj: Any, map: Map[K, V], memo: ...): AnyRef = {
  val converted = memo.asScala.getOrElseUpdate(obj, mutable.Map.empty[AnyRef, AnyRef]) match {
    case map: mutable.Map[AnyRef, AnyRef]@unchecked => map
    case map: java.util.Map[AnyRef, AnyRef]@unchecked => map.asScala
  }
  ...
}
```

No fallback case: if the memoized value for `obj` is ever something other
than a `mutable.Map` or `java.util.Map` (e.g. `null`, from re-entrant
conversion of AE2's memory-shared/copied `IAEKey`/`ItemStack` snapshots
tripping the `IdentityHashMap`-based memoization), this throws
`scala.MatchError: null` too — a secondary, harder-to-trigger path to the
same symptom as fix #1.

**Fix:** added a `case _ =>` that builds a fresh map and re-memoizes it
instead of throwing.

## 3. `Callbacks.scala` — `staticAnalyze` never scans interfaces

```scala
var c: Class[_] = seed
while (c != null && c != classOf[Object]) {
  val ms = c.getDeclaredMethods
  // ... register @Callback methods ...
  c = c.getSuperclass
}
```

`getDeclaredMethods()` only returns methods declared directly on a class —
never `default` methods inherited from an implemented interface. The loop
also only follows `getSuperclass()`, never `getInterfaces()`. AE2's network
callbacks (`getItemsInNetwork`, `getCraftables`, `getCpus`,
`getFluidsInNetwork`, `store`, power stats) are all `@Callback`-annotated
`default` methods on the `NetworkControl` interface, implemented by
`DriverController.Environment` — so they were silently never discovered.
Only `getEnergyStored`/`getMaxEnergyStored`/`canReceive`/`canExtract`,
declared directly on `Environment`, ever showed up on `me_controller`.

**Fix:** the traversal now recurses into `c.getInterfaces()` (and their
super-interfaces) as well as `c.getSuperclass()`, so `@Callback` default
methods on implemented interfaces are found too.

## 4. `CallbackWrapper.scala` — `INVOKEVIRTUAL` on an interface method

Fixing #3 alone wasn't enough: once `getItemsInNetwork` was discovered, it
was *callable* but always returned `nil`. `emitCallbackCall` generates ASM
bytecode to invoke each callback method, and always emitted `INVOKEVIRTUAL`:

```scala
mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, className, m.getName, Type.getMethodDescriptor(m), false)
```

That's correct when `m.getDeclaringClass` is a class, but wrong when it's an
interface (as is now the case for every `NetworkControl` default method
found via fix #3). `INVOKEVIRTUAL` against an interface-owned method throws
`java.lang.IncompatibleClassChangeError: Found interface X, but class was
expected` at call time — verified with a standalone ASM repro outside the
mod. That uncaught exception, with no surrounding `try`/`catch` in
`Component.invoke()`, is what surfaced in-game as a silent `nil` instead of
either a result or a visible error.

**Fix:** `emitCallbackCall` now checks `m.getDeclaringClass.isInterface` and
emits `INVOKEINTERFACE` in that case, `INVOKEVIRTUAL` otherwise (unchanged
for every pre-existing directly-declared callback).

## Result

After all four fixes: `me_controller` exposes the full `NetworkControl` API
(`getItemsInNetwork`, `getCraftables`, `getCpus`, `getFluidsInNetwork`,
`store`, power stats) alongside the energy convenience methods, and
`inventory_controller` reads against AE2 blocks no longer crash.
