This page shows how to set up a port from scratch

## Requirements

- dotnet 10 SDK
- The game disc as a `.cue`
- Function names for the game. a decompilation project is the easiest source. Without one, the recompiler can still find functions, but you do more work by hand (see [Function maps](#function-maps)).

## Project layout

A port lives in its **own folder**, not in RecompOne's folder. RecompOne is just a submodule of your project. you have to reference the runtime from your project.

```
MyPort/
  config/
    mygame.json > config that drives the recompiler
    funcmaps/ > generated function maps (address - name)
  patches/ > your C# patches
  disc/ > the game disc (*do NOT commit* this)
  generated/ > recompiled C# (*do NOT commit* this)
  Program.cs > entry point
  MyPort.csproj > references RecompOne.Runtime (or any other lib you include)
```

Two folders must stay out of version control:

- `disc/` holds copyrighted game data.
- `generated/` holds recompiled game code, which is derived from that data. this may be subject to copyright, you should regenerate it from the disc to create a build (or automate this via ci/cd) instead of committing it

You have to add both to `.gitignore`

## The config file

The config is a JSON file that tells the recompiler what to read and where to write
A minimal config looks like this:

```json
{
  "game": { "id": "SLUS-1234", "name": "MyGame", "output": "../generated" },
  "cue": "../disc/MyGame.cue",
  "funcMap": "funcmaps/main.json",
  "overlays": [],
  "patches": []
}
```

Fields:

- `game.output` where the recompiler writes the C#. Point it at `generated/`.
- `cue` path to the disc cue.
- `main` the boot executable inside the disc.
- `funcMap` the function map for the main executable (see below). Use `elf` +/or `map` instead if you want to read a decomp ELF directly.
- `overlays` extra code files the game loads at runtime.
- `patches` hooks into game functions (see [Patches](#patches)).

other useful fields: 
- `stubs` functions to replace with a no-op)
- `ignored` addresses to skip
- `linearSweep` find functions by scanning, when the map is incomplete, be aware that this can read data as code, use with care). you can also run the scan once and commit its output instead, see [Function maps](#function-maps)
- `debug` makes the recompiler generate prints for each function so you can know exactly what function path was taken at any point
- `functions[]` explicit addresses for functions to be recompiled

### Overlays

An overlay is a section of code that gets loaded in and out of the memory as needed, most PS1 games split code into several overlays that load over each others, to declare each one in the config:

```json
"overlays": [
  {
    "name": "ovl",
    "file": "OVL.BIN",
    "base": "0x800A0000",
    "funcMap": "funcmaps/ovl.json"
  }
]
```

- `name` the id the recompiler uses to generate the filename
- `file` the overlay file path on the disc
- `base` the address the overlay loads at
- `funcMap` / `elf` +/or `map` used to help finding function boundaries and names

for overlays that are not plain files on the disc, use `lba`, `offset`, `skip`, `size`, and `decrypt` to describe where the bytes are and whether they are encrypted (more decryption modes need to be added in the future probably)

## Function maps

The recompiler needs to know where functions start and if possible what they are called. It reads this from a **function map** wich is a JSON file that maps addresses and names

here you have two options (see bellow)

### Use a decomp ELF directly

Point `elf` and/or `map`(elfs are preferable) at the decomp output, the recompiler reads them at recompile time, this is the simplest path, but it teis your project to the decomp

```json
{ "elf": "../mygame-decomp/mygame.elf", "map": "../mygame-decomp/mygame.map" }
```

### Generate a function map (recommended)

You do not have to depend on the decomp at recompile time, you can generate a function map JSON once, commit it, and point `funcMap` at it:

```
dotnet run --project RecompOne.Recompiler --generate-function-file -elf mygame.elf -map mygame.map -out funcmaps/main.json
```

Use `-rebase <hex>` if the overlay loads at a different address than the ELF assumes. generate one file per overlay.

After this your project builds from the committed maps alone

### No decomp is available

In case the game you want to target has no decomp availbale the recompiler can find function boundaries by scanning the code itself. You have two ways to use it

The first is `linearSweep` in the config, the recompiler scans on every run. The second is generating a function map from that same scan, so you commit the result and stop scanning at recompile time:

```
dotnet run --project RecompOne.Recompiler -c Release -- --generate-function-file -linear-sweep -disc ../disc/MyGame.cue -base 0x800A0000 -file OVL.BIN -out funcmaps/ovl.json
```

`-linear-sweep` needs `-disc`, `-base`, and either `-file` (the path inside the disc) or `-lba <n>`. The same layout arguments the config uses are available here: `-offset`, `-skip`, `-size`, `-decrypt` and `-rebase`. generate one file per overlay.

Prefer generating the map. it is faster, the result is stable between runs, and you can rename entries by hand as you identify them, committing the names as you go. The recompiler matches a patch on `OVERLAY_FunctionName` too, so a name like `OVL_HitDetection` is picked up by a patch that targets `HitDetection` on any overlay

Either way you get generic names and no automatic SDK detection, so you must route SDK calls to the runtime by hand (see below)

LinearSweep can also be used in cases where a decomp does exist but an overlay havent been decomped yet

Be aware that a scan can read data as code, so check the result if an overlay behaves oddly

## SDK functions

Games call the PSYQ SDK (libgpu, libpad, libspu) the runtime reimplements somme of these. When your function map has the real SDK names, the recompiler recognizes them and routes each call to the runtime automatically.

If you use `linearSweep` or an incomplete map, the SDK names are missing, you need to map each SDK adress to its runtime counterpart yourself, using `patches` with a `replace` that targets the runtime function

Prefer a named map if avaible. it removes this work

## Recompiling

Run the recompiler on your config:

```
dotnet build RecompOne.Recompiler -c Release
dotnet run --project RecompOne.Recompiler -c Release --no-build -- config/mygame.json
```

This writes C# into `generated/`. Then build your project:

```
dotnet build MyPort.csproj
```

The entry point is small, the recompiler will generate one if this doesnt exists, this also works for hooking custom stuff and etc:

```csharp
using RecompOne.Runtime.Memory;
using Recompiled;

var m = new PSMemory();
Entry.Run(m, args.Length > 0 ? args[0] : null);
return 0;
```

## Patches

A patch is a C# method that runs around a game function. Write the method in `patches/`, then wire it in the config, keep in mind the recompiled code is akin to the original assembly so modding is more close to rom modding than source modding.

The method takes the CPU context and memory:
```csharp
using RecompOne.Runtime.Context;
using RecompOne.Runtime.Memory;

namespace Recompiled;

public static class MyPatch
{
    public static void OnRoomLoad(CpuContext c, IMemory m)
    {
        // custom code goes here
    }
}
```

Wire it with a `patches` entry:

```json
{
  "overlay": "ovl",
  "function": "LoadRoomLayer",
  "mode": "post",
  "target": "Recompiled.MyPatch.OnRoomLoad"
}
```

- `overlay` the overlay the function is in, this can also take a list or take "*" to target any function with this name
- `function` the function name (or use `address` with a hex value).
- `target` the fully qualified C# method.
- `mode`:
  - `pre` runs before the function. use `bool` and return `false` to skip the original, if it's void it acts like a true returning function.
  - `post` runs after the function.
  - `replace` replaces the function completly

Patches are part of your project, so they compile into the game binary. be aware that changing a patch rebuilds only the binary while changing the config needs a rerun on the recompiler to apply it.

## Next steps

- [How to Make a Mod](https://github.com/BlackLabelHQ/RecompOne/wiki/How-to-Make-a-Mod%3F), how to write mods for a finished port