# Note

you do not need the dotnet SDK to write a mod since mods ship as C# source and the runtime compiles them on startup, so there is no build step on your side!

## Mod layout

A mod is a folder inside the port's `mods/` directory, next to the executable:

```
mods/
  mymod/
    mod.json > the manifest
    mod-icon.png > optional, shown in the mods panel
    source/
      MyMod.cs > your code, can be more files, just showing one as an example
```

every `.cs` file under the mod folder is compiled, `source/` is jst a convention, a mod can also be a single `.zip` with the same contents since the runtime reads it without extracting

## The mods manfest

`mod.json` describes the mod:

```json
{
  "id": "mymod",
  "name": "My Mod",
  "version": "1.0.0",
  "author": "you",
  "description": "what the mod does",
  "dependencies": []
}
```

Fields:

- `id` unique id, mods are keyed by it. Duplicated ids are skipped
- `name` shown in the mods panel
- `dependencies` ids of mods that must load before this one

keep in mind that mods are disabled by default, the user enables them in the mods panel (choice is remembered for the next startup)

## The mod class

Any public class that implements `IMod` is created and loaded as in:

```csharp
using RecompOne.Runtime.Modding;

public class MyMod : IMod
{
    public void OnLoad()
    {
        // runs once after the mod is loaded
    }

    public void OnUnload()
    {
        // undo whatever OnLoad did, the user can disable a mod at runtime
    }

    public void DrawSettings()
    {
        // optional, ImGui calls are fine here, this is used to draw the settings pane on the mod menu
    }
}
```

`OnUnload` is important, a mod can be turned off without restarting the game so anything you registered has to be removed here or it keeps running

## Hooks

A hook is a static method that runs around a game function, you mark it with an attribute then the runtime finds it by reflection and attaches it

```csharp
using RecompOne.Runtime.Context;
using RecompOne.Runtime.Memory;
using RecompOne.Runtime.Modding;

public class MyMod : IMod
{
    public void OnLoad() { }

    [PostHook("dra", "LoadRoomLayer")]
    public static void OnRoomLoad(CpuContext c, IMemory m)
    {
        // custom code goes here
    }
}
```

the three attributes you have to know, them take the overlay and the function name (or address)

- `[PreHook(overlay, function)]` runs before the function. Use `bool` and return `false` to skip the original, if it is void it acts like a true returning method
- `[PostHook(overlay, function)]` runs after the function
- `[Replace(overlay, function)]` replaces the function. It can also optionally take a `Action<CpuContext, IMemory>` parameter to call the original when you want to

```csharp
[Replace("dra", "SomeFunction")]
public static void MyVersion(Action<CpuContext, IMemory> orig, CpuContext c, IMemory m)
{
    // do something before
    orig(c, m);
    // do something after
}
```

you can also use `Address` instead of the name if needed

```csharp
[PreHook("dra", Address = 0x800FE044)]
```

Rules worht knowing:

- Hook methods must be `static`. if your logic needs instance state, keep a static reference to your mod in `OnLoad` (or a singletron)
- The same method can carry several attributes, so one method can hook many functions
- Every `post` hook runs, they wont compete with each other ideally
- Only one `replace` can own a function, the first mod to claim it wins and the others are logged and ignored, priority will be added in the future

keep in mind the recompiled code mirrors the original assembly, so modding here is closer to rom hacking than to source modding.

## Events

the runtime and the port can dispatch events and listening to them is usually simpler than hooking a function as in:

```csharp
using RecompOne.Runtime.Events;

public void OnLoad()
{
    Event.AddListener<VSyncEvent>(OnVSync);
}

public void OnUnload()
{
    Event.RemoveListener<VSyncEvent>(OnVSync);
}

void OnVSync(VSyncEvent e)
{
    // runs once per frame
}
```

a port can declare its own events, so always check what the target port exposes

## Settings

`DrawSettings` is drawn inside the mods panel, use it for options. You have to make them persist yourself with the runtime config:

```csharp
using ImGuiNET;

const string Prefix = "mods.mymod.";

public void DrawSettings()
{
    if (ImGui.Checkbox("Enable thing", ref _enabled)) Save();
}

void Save()
{
    RecompOne.Runtime.Runtime.View.SetBool(Prefix + "enabled", _enabled); //this api is subject to change in the near future
    RecompOne.Runtime.Runtime.SaveView();
}

void Load()
{
    _enabled = RecompOne.Runtime.Runtime.View.GetBool(Prefix + "enabled", true);
}
```

Ideally prefix your keys with the mod id so two mods never collide

## Wrappers

Hooks give you a `CpuContext` and an `IMemory`, which means raw addresses, that works, but it is verbose and every mod ends up rediscovering the same offsets

A port can ship wrappers to hide that, and mods can use them directly. This is very worth doing and it is the difference between a mod being readable and being a miserable pile of magic numbers

Reading the player's hp without wrappers means knowing the address and the width: (following example uses SymphonyRecomp wraper as an example)


```csharp
int hp = (int)m.ReadU32(0x80012340);
```

With a wrapper it reads like normal code, and the address lives in one place instead of in every mod:

```csharp
int hp = Player.Hp;
```

SymphonyRecomp is a port that does this, see [SymphonyRecomp Modding Wrappers](https://github.com/BlackLabelHQ/SymphonyRecomp/wiki/SymphonyRecomp-Modding-Wrappers) for what a finished set can look like

### Writing wrappers for your port

Wrappers are plain C# in the port, there are some "patterns" that cover most of what a game needs if you are going to make a port, such as:

**Values at a fixed address.** A static class with properties over a base and offsets:

```csharp
namespace MyGame;

public static class Player
{
    const uint Base = 0x80012340;
    static IMemory M => RecompOne.Runtime.Runtime.Mem!;

    public static int Hp { get => (int)M.ReadU32(Base + 0x00); set => M.WriteU32(Base + 0x00, (uint)value); }
    public static int HpMax { get => (int)M.ReadU32(Base + 0x04); set => M.WriteU32(Base + 0x04, (uint)value); }
}
```

**A struct the game does heavy use** Take the address in the constructor and expose the fields, so the same class works for any instance as of:

```csharp
public sealed class Enemy
{
    public const int Stride = 0x40;
    public readonly uint Addr;
    public Enemy(uint addr) => Addr = addr;

    static IMemory M => RecompOne.Runtime.Runtime.Mem!;

    public int PosX { get => (int)M.ReadU32(Addr + 0x00); set => M.WriteU32(Addr + 0x00, (uint)value); }
    public int Hp { get => (short)M.ReadU16(Addr + 0x10); set => M.WriteU16(Addr + 0x10, (ushort)value); }
    public bool IsAlive => Hp > 0;
}
```

Then give mods a way to walk the table instead of doing pointer math wich can be prone to error:

```csharp
public static class Enemies
{
    const uint Table = 0x80020000;
    public const int Count = 64;

    public static Enemy At(int i) => new(Table + (uint)(i * Enemy.Stride));

    public static IEnumerable<Enemy> Alive()
    {
        for (int i = 0; i < Count; i++)
        {
            var e = At(i);
            if (e.IsAlive) yield return e;
        }
    }
}
```

**Constants as enums.** Ids the game uses for stuff:

```csharp
public enum Weapon { None = 0, Sword = 1, Axe = 2, Spear = 3 }
```

**Calling game functions.** This is the one that is not obvious so here is the explanation: Save the cpu context, put the arguments in the argument registers, call through the dispatcher, read the return from `V0`(or what the function uses), then restore the context to not break the game function path:

```csharp
using RecompOne.Runtime.Context;
using RecompOne.Runtime.Dispatch;

public static class GameApi
{
    const uint PlaySfxAddr = 0x80045678;

    public static uint Call(uint funcAddr, uint a0 = 0, uint a1 = 0, uint a2 = 0, uint a3 = 0)
    {
        var c = RecompOne.Runtime.Runtime.Cpu;
        var m = RecompOne.Runtime.Runtime.Mem;
        if (c == null || m == null) return 0;

        var snap = c.Snapshot();
        c.A0 = a0; c.A1 = a1; c.A2 = a2; c.A3 = a3;
        Dispatcher.Call(c, m, funcAddr);
        uint ret = c.V0;
        c.Restore(snap);
        return ret;
    }

    public static void PlaySfx(int id) => Call(PlaySfxAddr, (uint)id);
}
```

The snapshot and restore is inportant, you are using the same cpu context the game is running on so by leaving registers dirty corrupts whatever was in the middle of executing in the game's function path

If you are the one making the port, adding wrappers is the best thing you can do for modders! try to provide at least wrpaers for the player, the entity table, and the game state, the more wrapers your port exposes the easier modding will be for your port!

## Extra Note
You can create a .csproj referencing the port to get autocompletion and linter working, this is not deeded but it helps

## Next steps

- todo