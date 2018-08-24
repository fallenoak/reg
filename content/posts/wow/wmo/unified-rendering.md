+++
title     = "WMO: Unified Rendering"
author    = "fallenoak"
date      = "2017-11-18T13:17:55-06:00"
summary   = """\
            Dive into the unified WMO rendering path, first introduced in the Wrath of the Lich \
            King beta. Unified rendering is used whenever the `F_UNIFIED` flag (`0x2`) is set in \
            `SMOHeader->flags`. \
            """
slug      = "unified-rendering"
type      = "post"
tags      = [ "WoW", "WMO" ]
draft     = false
+++

<!-- 2016_11_22_11_00_32_139  -->

Blizzard debuted a new rendering path for WMOs in the Wrath of the Lich King beta: unified
rendering.

This rendering path is only used when `SMOHeader->flags` `F_UNIFIED` (`0x2`) is set. When
`F_UNIFIED` is not set, the classic rendering path is used.

From what I've observed, the unified rendering path was added to support using vertex coloring on
WMO exteriors.

Prior to unified rendering being added, vertex colors were always baked with an interior ambient
color term included. If these vertex colors were used in exterior batches, because the batches
already receive ambient light from the exterior lighting source, the batches would have become
overly bright.

Enter the unified rendering path.

Unified rendering involves a new suite of shaders (suffixed with `U`) and a new single rendering
function that replaces the previously separate `CMapObj::s_intFunc` and `CMapObj::s_extFunc`
functions.

In the unified rendering path, the ambient color term for interior batches is stored in
`SMOHeader->ambColor`, and is not baked into the vertex colors in the `MOCV` chunk. This permits
vertex colors to be used with exterior and interior batches alike. Exterior batches simply ignore
the color present in `SMOHeader->ambColor`.

A full explanation of the unified rendering path follows.

Note: when illustrating the output of the unified rendering path, we'll stick to a single WMO:
`ND_Human_Inn.wmo`. This WMO was introduced in Wrath of the Lich King, and is the stock human
inn used in various settlements around Northrend.

## Lighting Formula

Before diving into the guts of the rendering logic, let's take a look at the standard lighting
formula used for WMOs:

{{< highlight glsl >}}

lightColor.rgb  = saturate((dirColor.rgb * dirFactor) + ambColor.rgb);
lightColor.rgb *= matDiffuseColor.rgb;
lightColor.rgb += vertexColor.rgb;
lightColor.rgb  = saturate(lightColor.rgb + matEmissiveColor.rgb)

{{< /highlight >}}

This formula is found in the WMO vertex shaders, and its output is passed along to the WMO pixel
shaders. Once in the pixel shader, `lightColor` is doubled prior to being multiplied against the
sampled texture(s).

- `dirFactor` is the direct light factor. It determines the amount of the direct color term to
  apply to the final light color. The value is calculated as `saturate(dot(normal, -dir))`, similar
  to formulas found in any number of lighting implementations.

- `dirColor` is the direct color term. Like the ambient color term, the value has different sources
  depending on the specific lighting mode used for a given batch. See the lighting modes section
  below for more details.

- `ambColor` is the ambient color term. The value has different sources depending on the specific
  lighting mode used for a given batch. See the lighting modes section below for more details.

- `matDiffuseColor` is the material-specific diffuse color term. It is always set to
  `vec4(0.5, 0.5, 0.5, 1.0)`. Presumably, this term is meant to permit the use of material-specific
  diffuse coloring, and the value `vec4(0.5, 0.5, 0.5, 1.0)` is effectively a neutral default.
  I have yet to find logic that sets this value to something other than the default in Wrath of the
  Lich King.

- `vertexColor` is the vertex color term. It originates from the `MOCV` chunk, although the client
  has runtime logic to modify the values present on disk depending on certain flags.

- `matEmissiveColor` is the material-specific emissive color term. It is typically set to
  `vec3(0.0, 0.0, 0.0)`. When `SMOMaterial->flags` `F_SIDN` is set, this value is set the
  `SMOMaterial->frameSidn` color.

## Lighting Modes

As mentioned above, depending on the conditionals in the unified render path, the client sources
`dirColor` and `ambColor` from various places.

These conditionals are distilled into 4 distinct lighting modes:

| Mode | Name          | dirColor                            | ambColor                            |
| ---- | ------------- | ----------------------------------- | ----------------------------------- |
| `0`  | Unlit         | `vec3(0.0, 0.0, 0.0)`               | `vec3(0.0, 0.0, 0.0)`               |
| `1`  | Exterior      | `DNInfo->lightInfo->dirColor`       | `DNInfo->lightInfo->ambColor`       |
| `2`  | Window        | `DNInfo->lightInfo->windowDirColor` | `DNInfo->lightInfo->windowAmbColor` |
| `3`  | Interior      | `vec3(0.0, 0.0, 0.0)`               | `SMOHeader->ambColor`               |

- **Unlit** sets both `dirColor` and `ambColor` to zero, and effectively drops out the first part
  of the lighting formula.

- **Exterior** sets `dirColor` and `ambColor` to the standard exterior lighting colors found in
  `DayNight`. These colors are produced using the values present in the lighting tables in
  `DBFilesClient`.

- **Window** sets `dirColor` and `ambColor` to a variation of the standard exterior lighting colors
  found in `DayNight`. This lighting mode can only be selected when rendering batches in interior
  groups. It is selected by setting `SMOMaterial->flags` `F_WINDOW`.

- **Interior** sets `dirColor` to zero, and sources `ambColor` from `SMOHeader->ambColor`. This is
  typically used to light interior batches.

## Group Rendering Loop

The core of the unified rendering path is a loop that iterates across all batches defined in the
`MOBA` chunk for each WMO group.

This loop branches depending on the batch's type. Batches are broken into three types based on the
count values from the `MOGP` header: `trans`, `int`, and `ext` (in that order). `int` and `ext`
batches are handled together in one branch, while `trans` batches are handled separately.

{{< highlight c >}}

void CMapObj::RenderGroupUnified(CMapObjGroup* group, uint32_t frustumCount) {
    // ... snip ...

    for (int batchIdx = 0; batchIdx < group->batchCount; batchIdx++) {

        SMOBatch *batch = group->batchList[batchIdx];

        // common rendering logic

        if (batchIdx >= group->transBatchCount) {

            // int / ext batch rendering logic

        } else {

            // trans batch rendering logic

        }
    }

    // ... snip ...
}

{{< /highlight >}}

## Common Rendering Logic

Before branching for batch type specific logic, the client sets up general rendering state that
applies to all batches.

WIP

## Int / Ext Batch Rendering Logic

Now that the basics are out of the way, let's dive into the rendering logic for `int` / `ext`
batch types.

Within `int` / `ext` batch type rendering, there are two major branches: exterior and interior.

Interestingly, although assets flagged for unified rendering still divide batches across `trans`,
`int`, and `ext` batch types, the unified rendering path does not distinguish between `int` and
`ext` batch types. Rather, the client relies on the presence or absence of group flags
(`SMOGroup::EXTERIOR`, `SMOGroup::EXTERIOR_LIT`) to determine which branch to follow.

| Branch   | Batch Type      | Group Flags                                  | Possible Lighting Modes |
| -------- | --------------- | -------------------------------------------- | ----------------------- |
| Exterior | `int` / `ext`   | <code>EXTERIOR &#124; EXTERIOR_LIT</code>    | Unlit, Exterior         |
| Interior | `int` / `ext`   | <code>!(EXTERIOR &#124; EXTERIOR_LIT)</code> | Window, Interior        |

### Exterior / Exterior Lit Groups

The exterior branch within `int` / `ext` batch type rendering deals with batches in groups flagged
as either `SMOGroup::EXTERIOR` (`0x8`) or `SMOGroup::EXTERIOR_LIT` (`0x40`).

{{< highlight c >}}

// SMOGroup::EXTERIOR      0x8
// SMOGroup::EXTERIOR_LIT  0x40
if (group->flags & (SMOGroup::EXTERIOR | SMOGroup::EXTERIOR_LIT)) {

    // Exterior / Exterior Lit Groups

    int32_t lightingMode;

    // Choose the lighting mode
    // - F_UNLIT  0x1
    if (material->flags & F_UNLIT) {

        // Unlit (0)
        lightingMode = 0;

    } else {

        // Exterior (1)
        lightingMode = 1;

    }

    // Set the lighting mode (dirColor, ambColor, etc)
    CMapObjGroup::SetLighting(group, lightingMode);

    if (dword_CFBEA8) {

        dword_CFBEA8 = 0;
        CShadowCache::SetShadowMapGenericInterior(0);
        dword_D43010 = sub_873FF0();

    }

    if (dword_CFBEB0 != 2) {

        dword_CFBEB0 = 2;

        DNInfo* dnInfo = DayNightGetInfo();

        CShaderEffect::SetFogParams(
            dnInfo->fogInfo.start,
            dnInfo->fogInfo.end,
            dnInfo->fogInfo.fogRate,
            dnInfo->fogInfo.color
        );

        CShaderEffect::SetFogEnabled(1);

    }

} else {

    // Interior Groups (see below section)

}

{{< /highlight >}}

### Interior Groups

The interior branch within `int` / `ext` batch type rendering deals with batches in all remaining
groups (ie. groups not flagged as `SMOGroup::EXTERIOR` or `SMOGroup::EXTERIOR_LIT`).

{{< highlight c >}}

// SMOGroup::EXTERIOR      0x8
// SMOGroup::EXTERIOR_LIT  0x40
if (group->flags & (SMOGroup::EXTERIOR | SMOGroup::EXTERIOR_LIT)) {

    // Exterior / Exterior Lit Groups (see above section)

} else {

    // Interior Groups

    int lightingMode;

    // Choose the lighting mode
    // - F_WINDOW  0x20
    if (material->flags & F_WINDOW) {

        // Window (2)
        lightingMode = 2;

    } else {

        // Interior (3)
        lightingMode = 3;

    }

    // Set the lighting mode (dirColor, ambColor, etc)
    CMapObjGroup::SetLighting(group, lightingMode);

    if (dword_CFBEA8 != 1) {

        dword_CFBEA8 = 1;
        CShadowCache::SetShadowMapGenericInterior(1);
        dword_D43010 = sub_873FF0() != 0;

    }

    sub_7A8440(v81);

}

{{< /highlight >}}

WIP

## Trans Batch Rendering Logic

TODO
