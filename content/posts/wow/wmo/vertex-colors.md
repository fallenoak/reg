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

## CMapObjGroup::FixColorVertexAlpha [#](#fix-color-vertex-alpha) {#fix-color-vertex-alpha}

WIP

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

            // Overwrite color values
            color->r = std::min(r, 255);
            color->g = std::min(g, 255);
            color->b = std::min(b, 255);

            // Overwrite alpha value
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
