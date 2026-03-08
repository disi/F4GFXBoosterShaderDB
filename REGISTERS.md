# Constant Buffer

This document maps the constant buffer (CB) registers used by the game's shaders. Use these mappings when writing replacement `.hlsl` code to ensure proper integration with engine-side variables.

> [!IMPORTANT]
> **Contextual Switching:** Slot `cb2` is dual-purpose. In **Vertex Shaders**, it primarily handles geometric transforms (Matrices). In **Pixel Shaders**, it stores material and lighting parameters.

---

## Pixel Shaders (PS)

### Camera & Projection (`cb0`)
| Register | Component | Description |
| :--- | :--- | :--- |
| `cb0[0]` | `.xy` | **Projected UV Scale**: `UV = SV_POSITION.xy * cb0[0].xy` |
| `cb0[0]` | `.z / .y` | **Depth Params**: Camera near/far bias and attenuation used in alpha/math logic. |

### UV Transforms & Tinting (`cb1`)
| Register | Component | Description |
| :--- | :--- | :--- |
| `cb1[0]` | `.xy` | **UV Offset**: Translation/Scrolling |
| `cb1[0]` | `.zw` | **UV Scale**: `outUV = inUV * cb1[0].zw + cb1[0].xy` |
| `cb1[0]` | `.xyz` | **Global Tint**: Color multiplier for the final pixel output. |

### Main Material & Lighting (`cb2`)
| Register | Component | Description |
| :--- | :--- | :--- |
| `cb2[0]` | `.xy` | **Environment Scale**: Scaling for env/reflection map lookups. |
| `cb2[0]` | `.z` | **Global Alpha**: Primary transparency multiplier. |
| `cb2[0]` | `.w` | **Light Intensity**: Global scalar for incoming light calculations. |
| `cb2[1]` | `.xyz` | **Material Color**: Primary diffuse/light color (often copied to output). |
| `cb2[2]` | `.x / .xy` | **LUT / UV Tweaks**: Coordinate offsets for Lookup Tables or projected UV tweaks. |
| `cb2[3]` | `.xy` | **Env Overrides**: Per-material environment flags or scale overrides. |
| `cb2[4]` | `.xyz` | **Dominant Light Vector**: Directional vector for lighting calculations. |

#### **Packed Material Flags (`cb2[5]`)**
* `.x` : Diffuse/Gloss slot index or flag.
* `.y` : **Has-Roughness** boolean flag.
* `.z` : AO (Ambient Occlusion) strength multiplier.
* `.w` : Upper slot index used in diffuse/parameter scaling.

#### **Specular & Gloss Group (`cb2[6]`)**
* `.x` : **Raw Glossiness**: Range `0.0 - 255.0` (stored as float).
* `.y` : Has-Roughness flag (secondary check).
* `.z` : Specular lower bound.
* `.w` : Specular upper bound.

---

## Vertex Shaders (VS)

### Camera & View (`cb0`)
| Register | Component | Description |
| :--- | :--- | :--- |
| `cb0` | `varies` | Near/Far parameters and camera scalars. |
| `cb0[2]` | `.xyz` | **Projection Origin**: Offset applied as `-cb0[2].xyz + coord`. |
| `cb0[3]` | `.xyz` | **UV Divisor**: Scaling applied as `coord / cb0[3].xyz`. |

### World Transform / Matrices (`cb2`)
In Vertex Shaders, `cb2` typically holds the Object-to-World or World-to-Projection matrices.

| Register | Component | Description |
| :--- | :--- | :--- |
| `cb2[0..2]` | `.xyz / .w` | **World Matrix Rows**: `worldPos = dot(inPos, cb2[0..2]) + cb2[..].w` |
| `cb2[3..6]` | `.xyzw` | **Projection Rows**: Cascade/Projected transform matrices. |
| `cb2[7]` | `.x` | **Disable Flag**: Found in certain terrain or LOD shaders. |

---

## Global Helpers & Specifics (`cb12` & Others)

* **Common Injected Multiplier (`cb12[30].x`)**: Used for global scene-wide brightness or scaling adjustments.
* **Projection Constants (`cb12[35..36]`)**: Shader-specific projection/offset constants referenced for vertex outputs.
* **Water Specialized Shaders (`cb1`)**:
    * `cb1[9]`: Per-layer normal weights.
    * `cb1[10] / cb1[12]`: Specular and Fresnel intensity/attenuation controls.

---

### Implementation Snippets (HLSL)

**Reconstructing World Position in VS:**
```hlsl
float4 GetWorldPos(float4 inPos) {
    float3 worldPos;
    worldPos.x = dot(inPos, cb2[0]);
    worldPos.y = dot(inPos, cb2[1]);
    worldPos.z = dot(inPos, cb2[2]);
    // The .w components often act as the translation column
    return float4(worldPos + float3(cb2[0].w, cb2[1].w, cb2[2].w), 1.0f);
}