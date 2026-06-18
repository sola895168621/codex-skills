---
name: fbx-zhengli-2
description: Advanced FBX cleanup pipeline for Unity assets plus optional Maya quad-retopology. Use when Codex needs to process imported Unity FBX props, AI-generated/downloaded FBX models, Hunyuan/image-to-3D outputs, or models that should be organized for Unity and then imported into Maya for PBR material setup, Maya polyRetopo quad conversion, UV/normal transfer from the high-poly source, texture reassignment, and validation of all-quad retopology and material texture connections.
---

# FBX Zhengli 2

Use this workflow when the user asks for "fbx整理2", "fbx zhengli 2", asks to combine the old FBX整理 workflow with Maya retopology, or asks to process FBX models through Unity cleanup and Maya quad-retopo/texture transfer.

## Goal

Produce a clean Unity asset set and, when requested, a Maya scene result with:

- stable Chinese asset names in Unity folders
- isolated materials and texture ownership
- correct Unity normal/metallic/roughness setup
- optional Unity low-poly prefab
- optional Maya retopologized all-quad model
- original high-poly source preserved for transfer/baking
- PBR textures correctly connected to the retopo model

## Unity Asset Organization

Follow the original FBX整理 / fbx-zhengli structure unless the user specifies another destination:

```text
<feature-or-art-root>/
  Models/
    Generated/
      ModelName.fbx
    LowPolyFBX/
      ModelName_LowPoly.fbx
    LowPolyMeshes/
      ModelName_LowPoly.asset
  Materials/
    Generated/
      ModelName_Mat.mat
  Textures/
    Generated/
      ModelName/
        ModelName_BaseColor.png
        ModelName_Normal.png
        ModelName_Metallic.png
        ModelName_Roughness.png
        ModelName_MetallicSmoothness.png
  Prefabs/
    Generated/
      ModelName_LowPoly.prefab
```

Adapt `Generated` to a project-specific category such as `CandyProps` when processing batches.

## Unity Workflow

1. Discover source assets.
   - Locate the target FBX, `.meta`, `.fbm`, material, and texture files.
   - Check whether Unity imported the model with wrong shared materials or textures from another folder.
   - Preserve the original high-poly FBX under `Models/Generated/` or the user's requested art root.

2. Inspect appearance before naming.
   - Do not name from hash names or broken thumbnails.
   - Fix enough material/texture import state to see the real model.
   - Use short descriptive Chinese names in this user's Unity art folders.

3. Isolate material and texture ownership.
   - Create one material per model, named `ModelName_Mat.mat`.
   - Move each texture set to `Textures/<category>/<ModelName>/`.
   - Rename PBR textures to `ModelName_BaseColor`, `ModelName_Normal`, `ModelName_Metallic`, and `ModelName_Roughness`.
   - Move `.meta` files with assets when using filesystem moves; prefer Unity `AssetDatabase` when connected.

4. Fix Unity import settings.
   - Set every `*_Normal.*` texture to `TextureImporterType.NormalMap`.
   - Set normal, metallic, roughness, and packed metallic-smoothness textures to non-sRGB.
   - Do not treat a normal-map display problem as a mesh-normal problem until texture type is verified.

5. Wire Unity URP material maps.
   - Use `Universal Render Pipeline/Lit` unless the project clearly uses another shader.
   - Assign BaseColor to `_BaseMap` and Normal to `_BumpMap`.
   - URP Lit has no separate Roughness slot: pack `Metallic` into RGB and `1 - Roughness` into Alpha as `ModelName_MetallicSmoothness.png`.
   - Assign the packed texture to `_MetallicGlossMap`.
   - Enable `_NORMALMAP` and `_METALLICSPECGLOSSMAP`.
   - Set metallic workflow, `_SmoothnessTextureChannel = Metallic Alpha`, and use the user's requested smoothness value if given.

6. Generate Unity low-poly output only when requested.
   - Default to 1000-2000 triangles for small props/characters; use 1500 as a common target.
   - Preserve UVs and material references.
   - Use Blender decimate with UV seams/sharp edges protected for generated FBX files when Unity-only reduction would break UVs.
   - Save reduced FBX as `Models/LowPolyFBX/ModelName_LowPoly.fbx` or Unity mesh asset as `Models/LowPolyMeshes/ModelName_LowPoly.asset`.
   - Save prefab as `Prefabs/<category>/ModelName_LowPoly.prefab` and assign the cleaned material explicitly.

7. Clean duplicate import shells.
   - Delete old `.fbm` and temporary `Materials` folders only after organized textures/materials are verified.
   - Remove malformed empty folders only after checking they contain no real assets.

## Maya Quad-Retopology Workflow

Use this section when the user asks for Maya import, retopology, quads, transfer maps, or texture transfer after FBX cleanup.

1. Connect to Maya.
   - Prefer the configured Maya MCP gateway, usually `http://127.0.0.1:9765/mcp`.
   - Search/load `maya-scripting__execute_python` or typed geometry/material tools.
   - Verify `fbxmaya`, `polyRetopo`, and `polyRemesh` are available before mutating the scene.

2. Avoid Unicode node-name issues.
   - Maya can read Chinese file paths, but MCP-transmitted Python or Maya node renaming may mangle Chinese into invalid characters.
   - Use ASCII-safe Maya node names such as `Gingerbread_High`, `Gingerbread_Retopo_Quad`, and `Gingerbread_PBR_Mat`.
   - Keep the original Chinese file paths for FBX and textures.

3. Import the FBX and preserve the high-poly source.
   - Load `fbxmaya`.
   - Import the FBX into the current scene.
   - Detect imported mesh transforms by listing mesh shapes and parents, not only by root assembly diffs; some FBX imports arrive as `node_0`.
   - Rename the high-poly mesh to an ASCII-safe name.
   - Group and hide the high-poly source, for example `Gingerbread_High_GRP`, instead of deleting it.

4. Create Maya PBR material with correct texture color spaces.
   - Create `standardSurface` when available, otherwise use the project's established shader.
   - Connect BaseColor file node with `sRGB` color space to `baseColor`.
   - Connect Metallic file node with `Raw` color space to `metalness`.
   - Connect Roughness file node with `Raw` color space to `specularRoughness`.
   - Connect Normal file node with `Raw` color space through a `bump2d` node set to tangent-space normal, then into `normalCamera`.
   - Assign the material to both high-poly and retopo models.

5. Run Maya retopology.
   - Duplicate the high-poly mesh for work, for example `Gingerbread_Retopo_Work`.
   - Use `polyRetopo` to make a quad retopo model; for generated 50k triangle characters, a starting target around 6000 faces is reasonable unless the user requests otherwise.
   - Use flags equivalent to: target face count, tolerance, topology regularity, face uniformity, anisotropy, preserve hard edges, preprocess mesh, replace original, no construction history.
   - If `polyRetopo` fails, try `polyRemesh` first, then `polyRetopo`.
   - Rename the result to `ModelName_Retopo_Quad` or an ASCII-safe equivalent.

6. Transfer UVs/normals from high to retopo.
   - Use `transferAttributes` from the hidden high-poly mesh to the retopo mesh.
   - Transfer normals and UVs, do not transfer positions unless intentionally shrinkwrapping.
   - Try sample spaces in this order when UVs do not appear: `0`, `1`, `4`, `5`.
   - Use the actual source UV set name from the high-poly mesh; generated FBX may use `UVMap` instead of `map1`.
   - Ensure the retopo mesh has a UV set before transfer if needed.
   - Delete construction history after a successful transfer.
   - If transfer still produces zero UVs, create an automatic UV projection only as a fallback and tell the user it is not equivalent to a real texture bake.

7. Texture transfer vs texture reassignment.
   - If the user says "把贴图加上" or "传递贴图" casually, first transfer UVs/normals and assign the original PBR texture material to the retopo model.
   - If the user explicitly wants new baked texture files, use Maya Transfer Maps or an available baking workflow to bake BaseColor, Normal, Metallic, and Roughness from high to retopo and save new PNGs. Do not claim baked files were generated unless real output files exist.

## Maya Validation Checklist

Before final response after Maya work, verify:

- High-poly source still exists and is hidden.
- Retopo mesh exists and is selected or clearly named.
- Retopo face count is close to the requested target.
- Retopo polygon composition is all quads, or report exact triangle/other counts.
- Retopo UV count is greater than zero.
- Retopo material is assigned.
- BaseColor, Metallic, Roughness, and Normal file nodes are connected to the material.
- Texture paths resolve to the intended Unity texture files.
- Report if output is only UV/material transfer and not a baked Transfer Maps texture set.

## Combined Validation Checklist

For Unity + Maya combined jobs, verify:

- Unity organized FBX/material/textures/prefab paths.
- Unity normal import type and URP metallic-smoothness packing.
- Unity reports no compilation errors if Unity files were edited.
- Maya high-poly and retopo mesh state.
- Maya all-quad count and UV/material connections.

## Common Trigger Phrases

Apply this skill when the user says:

- "fbx整理2"
- "按 fbx整理2 处理"
- "把这个 FBX 导入 Maya 重新拓扑"
- "弄成四边面再把贴图传过去"
- "Unity 整理完再进 Maya retopo"
- "像姜饼人这次的流程处理"
