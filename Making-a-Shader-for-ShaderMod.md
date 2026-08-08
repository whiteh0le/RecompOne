This page shows how to port or create a shader for [ShaderMod](github.com/BlackLabelHQ/ShaderMod)

Its also worth noting that the shader mod is not tied to any game, it works on any port made with RecompOne

## Shader layout

A shader is a folder inside the shader mod's `shaders/` directory:

```
mods/
  shaders/
    shaders/
      myshader/
        settings.json > name and parameters
        myshader.frag > the fragment shader
```

Drop the folder in, press Rescan in the mod settings, and it shows up in the dropdown

## The fragment shader

The shader is a single fragment shader that runs over the finished frame. The runtime provides the vertex stage and these uniforms:

```glsl
#version 330 core
in vec2 vUv;
out vec4 oColor;

uniform sampler2D uTex;      // the rendered frame
uniform vec2 uTexSize;       // size of uTex in pixels
uniform vec2 uOutputSize;    // size of the output, same as uTexSize as of now
uniform float uTime;         // seconds since start
uniform int uFrame;          // frames since start, use this for anything per frame
```

The smallest shader that does something:

```glsl
#version 330 core
in vec2 vUv;
out vec4 oColor;
uniform sampler2D uTex;

void main()
{
    vec3 c = texture(uTex, vUv).rgb;
    oColor = vec4(1.0 - c, 1.0);
}
```

Use `uFrame` and not `uTime` for effects that must advance once per frame, like interlacing. `uTime` moves with real time, so it changes with the framerate

## settings.json

```json
{
    "name": "My Shader",
    "author": "you",
    "description": "what it does",
    "fragment": "myshader.frag",
    "parameters": [
        { "uniform": "strength", "name": "Strength", "default": 0.5, "min": 0.0, "max": 1.0, "step": 0.05 },
        { "uniform": "enableThing", "name": "Enable thing", "type": "bool", "default": 1.0 }
    ]
}
```

Fields:

- `name` shown in the dropdown, falls back to the folder name
- `fragment` the shader file, falls back to the first `.frag` or `.glsl` in the folder
- `parameters` values pushed into the shader as uniforms

Each parameter needs a `uniform` that matches a `uniform float` declared in the shader. The value comes from `value` if present, oterwise from `default`, so you can keep the original default around while overriding it:

```json
{ "uniform": "strength", "default": 0.5, "value": 0.8 }
```

`min`, `max`, `step` and `type` are not used yet, they are there for a future settings ui. as of now you edit the json by hand

The file accepts `//` comments so you can annotate it if needed, it wont give erros when using it

Parameters are declared in the shader as plain floats, there are no bools or ints:

```glsl
uniform float strength;
uniform float enableThing;
```

A uniform that is in the json but not in the shader is ignored, no error. That means a typo in `uniform` fails silently, check the name if a parameter seems to do nothing

## Iterating

Editing a shader does not need a restart. Change the `.frag` or the `settings.json`, then press Reload in the mod settings and it recompiles from disk

If the shader fails to compile, the error from the driver is shown in red in the mod settings. so fix and press Reload again

## Porting an existing shader

Most shaders you find are written for RetroArch or libretro, they are close enough to port, the work is mapping their uniforms onto RecompOne's

A libretro shader is usually one file holding both stages, split by `#if defined(VERTEX)` and `#elif defined(FRAGMENT)`, take the fragment half and drop the rest

Then map the names. The cleanest way is a block of defines at the top, so the body of the original stays untouched, so it can looks something like this:

```glsl
#define Texture     uTex
#define TextureSize uTexSize
#define InputSize   uTexSize
#define OutputSize  uOutputSize
#define FrameCount  uFrame
#define FragColor   oColor
#define COMPAT_TEXTURE texture
```

`InputSize` and `TextureSize` are the same value here, the frame recompone hands for you has no padding around it

Some of the things usually need real edits:

**Values computed in the vertex stage.** libretro shaders precompute things there and pass them as varyings, almost all of them derive only from uniforms, so declare them as globals and fill them at the top of `main`:

```glsl
vec2 sinangle;
vec2 cosangle;

void main()
{
    sinangle = sin(vec2(x_tilt, y_tilt));
    cosangle = cos(vec2(x_tilt, y_tilt));
    // rest of the original main
}
```

of a varying depends on the texture coordinate rather than only on uniforms, recompute it from `vUv`, this should work

**Helper functions that lived in the vertex stage.** Copy them over verbatim, they usually only call each other in theory

**Output wrappers.** some shaders write through a struct, replace it with a plain write to `oColor`

Their `#pragma parameter` lines map one to one onto `settings.json`. The format is:

```
#pragma parameter NAME "Label" default min max step
```

so a line like

```
#pragma parameter CRTgamma "Target Gamma" 2.4 0.1 5.0 0.1
```

becomes

```json
{ "uniform": "CRTgamma", "name": "Target Gamma", "default": 2.4, "min": 0.1, "max": 5.0, "step": 0.1 }
```

Keep every parameter, even the ones you do not plan to expose. They are already the shader's defaults and leaving them out means the uniform stays at zero

The crt-geom shader that ships with the mod was ported this way, read it side by side with the libretro original if you get stuck, this can help you understanding recompone's shader system!

## Licensing

Shaders carry licenses, crt-geom is GPL, many others are too. If you ship a port of someone else's shader, keep the original header comment and credit the author, there is a `license` field in `settings.json` for this, only ship shaders that the license allow.