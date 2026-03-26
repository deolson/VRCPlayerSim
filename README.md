# VRCSim

Multiplayer simulation for VRChat worlds. Test station seating, ownership transfer, synced variables, and master-gated logic without leaving the Unity editor.

Built on top of [ClientSim](https://docs.vrchat.com/docs/clientsim). Where ClientSim gives you one local player, VRCSim adds remote player bots with perspective swapping — so you can verify what Player 2 actually sees.

## Install

Add to your VRChat project's `Packages/manifest.json`:

```json
{
  "dependencies": {
    "com.fire.vrcsim": "file:../../VRCPlayerSim"
  }
}
```

Adjust the relative path to wherever you cloned this repo. VRCSim requires the VRChat SDK (Worlds) and ClientSim — both should already be in your project.

## Usage

All calls are in play mode. Works from Editor scripts, `[MenuItem]` methods, or the Unity MCP `script-execute` tool.

```csharp
// 1. Initialize (once per play session)
var err = VRCSim.VRCSim.Init();

// 2. Spawn bots
var alice = VRCSim.VRCSim.SpawnPlayer("Alice");
var bob   = VRCSim.VRCSim.SpawnPlayer("Bob");

// 3. Interact — sit Alice in a station
VRCSim.VRCSim.SitInStation(alice, GameObject.Find("MyStation"));

// 4. Check state
var phase = VRCSim.VRCSim.GetVar(gameManagerObj, "gamePhase");

// 5. Test non-master perspective
VRCSim.VRCSim.RunAsClient(bob, () => {
    // Inside here:
    //   Networking.LocalPlayer == bob
    //   bob.isLocal == true
    //   _localPlayer on all UdonBehaviours == bob
    VRCSim.VRCSim.RunUpdate(gameManagerObj);  // Bob's Update tick
});

// 6. Snapshot & diff synced state
var before = VRCSim.VRCSim.TakeSnapshot();
VRCSim.VRCSim.RunUpdate(gameManagerObj);
var after = VRCSim.VRCSim.TakeSnapshot();
var changes = VRCSim.VRCSim.DiffSnapshots(before, after);

// 7. Cleanup
VRCSim.VRCSim.RemoveAllPlayers();
```

### `RunAsPlayer` vs `RunAsClient`

| Method | `Networking.LocalPlayer` | `_localPlayer` (cached in Start) | Use when |
|--------|-------------------------|----------------------------------|----------|
| `RunAsPlayer` | Swapped | **Not swapped** | Ownership checks only |
| `RunAsClient` | Swapped | **Swapped on all UdonBehaviours** | Everything else |

Most UdonSharp code caches `_localPlayer = Networking.LocalPlayer` in `Start()` and checks `_localPlayer.isMaster`. `RunAsPlayer` doesn't touch those cached refs. **Use `RunAsClient` for almost all multiplayer tests.**

## API

Full reference: **[API.md](API.md)** (auto-generated from source — never edit manually).

Key entry points on `VRCSim.VRCSim`:

| Category | Methods |
|----------|---------|
| Lifecycle | `Init`, `SpawnPlayer`, `RemovePlayer`, `RemoveAllPlayers` |
| Stations | `SitInStation`, `ExitStation` |
| Perspective | `RunAsPlayer`, `RunAsClient` |
| Ownership | `SetOwner`, `GetOwner`, `TransferMaster` |
| Variables | `GetVar`, `SetVar`, `SendEvent`, `GetSyncedVars` |
| Snapshots | `TakeSnapshot`, `DiffSnapshots` |
| Simulation | `RunUpdate`, `RunEvent`, `SimulateDeserialization`, `SimulateLateJoinerAll` |
| Networking | `SendNetworkEvent`, `EnforceKinematic`, `ValidateKinematic` |
| Reporting | `GetStateReport`, `ValidateVars` |

## Architecture

```
Runtime/
  VRCSim.cs           Public API facade
  SimNetwork.cs       Perspective swapping, ownership, kinematic enforcement
  SimReflection.cs    Cached reflection into ClientSim internals
  SimSnapshot.cs      Synced state capture and diffing
```

All ClientSim access is via reflection, isolated in `SimReflection.cs`. If a VRChat SDK update breaks an internal API, you get a clear `Required member not found: X` error at init time — not a cryptic null ref mid-test.

## Contributing

```sh
git clone https://github.com/deolson/VRCPlayerSim.git
cd VRCPlayerSim
pip install pre-commit    # or: uv tool install pre-commit
pre-commit install
```

The pre-commit hook regenerates `API.md` from source whenever `Runtime/*.cs` or `gen_api.py` changes. CI also checks this on PRs. Requires [uv](https://docs.astral.sh/uv/) on PATH.

## Limitations

- **Single-process** — all players share one Unity instance. No network latency or packet loss simulation.
- **Cached `_localPlayer`** — `RunAsClient` swaps these, but `RunAsPlayer` does not. This is intentional and matches real VRChat behavior.
- **`PlayerObjectStorage` file locks** — rapid spawn/remove cycles can trigger harmless ClientSim errors in the console.

## Compatibility

| Dependency | Version |
|-----------|---------|
| Unity | 2022.3.x |
| VRChat SDK | 3.x (Worlds) |
| ClientSim | 1.x |
| UdonSharp | Required |

## License

MIT
