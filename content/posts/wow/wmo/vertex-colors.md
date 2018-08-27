+++
title     = "WMO: Vertex Colors"
author    = "fallenoak"
date      = "2018-08-26T02:48:00-06:00"
summary   = """\
            Learn how WMOs use and abuse vertex colors. \
            """
slug      = "vertex-colors"
type      = "post"
tags      = [ "WoW", "WMO" ]
draft     = false
+++

WMOs, also known as MapObjs, make use of vertex colors to light everything from group geometry to
units and game objects. These vertex colors are baked by the tooling employed by Blizzard's
artists--likely some kind of export plugin for the 3D modeling software they use. Vertex colors
provide an efficient way of approximating a large number of lights at runtime.

The vertex colors baked into WMO data files undergo a few different runtime manipulations before
being fed to the vertex and pixel shaders.

These manipulations are explained below.

## CMapObj::AttenTransVerts [#](#atten-trans-verts) {#atten-trans-verts}

If `SMOHeader->flags` **is not flagged** with `0x1`, the client triggers the
`CMapObj::AttenTransVerts` function. This function is called from inside `CMapObj::RenderGroup`,
and only runs once per WMO group.

`CMapObj::AttenTransVerts` overwrites vertex alpha values based on the distance from the associated
vertex to the nearest portal. The function only modifies vertex colors for vertices contained in
`trans` batches. Vertex colors for vertices in `int` and `ext` batches are left alone.

## CMapObjGroup::FixColorVertexAlpha [#](#fix-color-vertex-alpha) {#fix-color-vertex-alpha}

The logic described in this section matches the behavior of the Wrath of the Lich King client.
Newer versions of the client contain different / additional logic.

If a WMO group is flagged with `SMOGroup::CVERTS` (`0x4`) and `SMOHeader->flags` **is not flagged**
with `0x8`, the client triggers the `CMapObjGroup::FixColorVertexAlpha` function shown below. This
function is called from inside `CMapObjGroup::CreateOptionalDataPointers`, and only runs once per
WMO group.

`CMapObjGroup::FixColorVertexAlpha` overwrites vertex color values in the following ways:

- `trans` batch type vertices:
   1. Divide existing vertex color by `2`
   2. Overwrite existing vertex color with adjusted vertex color

- `int` and `ext` batch type vertices:
   1. Add `(alpha * rgb) / 64` to existing vertex color
   2. Divide adjusted vertex color by `2`
   3. Clamp adjusted vertex color to no more than `255`
   4. Overwrite existing vertex color with adjusted vertex color
   5. Overwrite existing vertex alpha with `255`

Note that the WMO pixel shaders multiply the incoming color by `2`, offsetting the division by `2`
that occurs in `FixColorVertexAlpha`.

{{< highlight cpp >}}

void CMapObjGroup::FixColorVertexAlpha() {

    if (this->colorVertexCount == 0) {
        return;
    }

    int32_t intBatchStart;

    // Identify the first post-trans-batch vertex
    if (this->transBatchCount > 0) {
        intBatchStart = this->batchList[this->transBatchCount].maxIndex + 1;
    } else {
        intBatchStart = 0;
    }

    for (int32_t i = 0; i < this->colorVertexCount; i++) {

        CImVector* color = &this->colorVertexList[i];

        // Int / ext batches
        if (i >= intBatchStart) {

            // Calculated adjusted color values
            int32_t r = (color->r + (color->a * color->r / 64)) / 2;
            int32_t g = (color->g + (color->a * color->g / 64)) / 2;
            int32_t b = (color->b + (color->a * color->b / 64)) / 2;

            // Overwrite existing color values
            color->r = std::min(r, 255);
            color->g = std::min(g, 255);
            color->b = std::min(b, 255);

            // Overwrite existing alpha value
            color->a = 255;

        // Trans batches
        } else {

            color->r /= 2;
            color->g /= 2;
            color->b /= 2;

        }

    }

}

{{< /highlight >}}
