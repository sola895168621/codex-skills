---
name: fbx-zhengli
description: Clean up imported Unity FBX prop/model assets and AI-generated/downloaded FBX models. Use when Codex needs to process external Unity FBX props such as hash-named or poorly imported models, Hunyuan/image-to-3D outputs, or single 50k-style generated characters by inspecting their real textured appearance, renaming models/materials/textures, separating per-model materials and texture sets, fixing normal texture import types, packing metallic/roughness for URP Lit, generating 1000-2000 triangle low-poly meshes/FBX files and prefabs when requested, organizing assets into Models/Materials/Textures/Prefabs folders, preserving Unity .meta GUIDs, and validating Unity references/import errors.
---

# FBX整理

Use this workflow for batches of imported Unity FBX prop models, or a single downloaded/generated FBX, that arrive with hash names, shared materials, broken material imports, wrong normal texture settings, oversized meshes, loose `.fbm` texture folders, or separate PBR textures that Unity did not wire into the material.

## Core Outcome

Produce a Unity-ready asset set with stable names, isolated material/texture ownership, optimized low-poly prefabs when requested, and no broken references.

Target structure:

```text
<feature-folder>/
  Models/
    CandyProps/                 # or a domain-appropriate category
      ModelName.fbx
    Generated/                  # useful for single AI-generated source FBX files
      ModelName.fbx
    LowPolyFBX/                 # useful when Blender exports a reduced FBX
      ModelName_LowPoly.fbx
    LowPolyMeshes/
      ModelName_LowPoly.asset
  Materials/
    CandyProps/
      ModelName_Mat.mat
  Textures/
    CandyProps/
      ModelName/
        ModelName_BaseColor.png
        ModelName_Normal.png
        ModelName_Metallic.png
        ModelName_Roughness.png
        ModelName_MetallicSmoothness.png
  Prefabs/
    LowPolyCandyProps/
      ModelName_LowPoly.prefab
    Generated/
      ModelName_LowPoly.prefab
```

Adapt category names to the project, but keep Models, Materials, Textures, and Prefabs separated unless the user asks otherwise.

## Workflow

1. Discover the batch.
   - Locate the target FBX files and their `.meta`, `.fbm`, material, and texture assets.
   - Check whether imported materials are shared across models or incorrectly sourced from another folder.
   - For AI/downloaded FBX outputs, first move the source FBX to the user's requested art root, then keep the high-poly source intact under a clear `Models/Generated/` or project-equivalent folder.
   - Prefer Unity `AssetDatabase` or MCP inspection when available; use filesystem checks only when it preserves `.meta` files safely.

2. Inspect real appearance before naming.
   - Do not name models from hash names or broken thumbnails.
   - Fix obvious material/texture import issues first enough to preview the true appearance.
   - Generate or view a preview if needed, then choose short descriptive Chinese names when working in this user's Unity art folders.

3. Fix material isolation.
   - Ensure each FBX has its own material, not a shared imported placeholder.
   - When Unity FBX Materials `Location` must be `Use External Materials (Legacy)`, guard against all models resolving to the first same-named material.
   - If a generated FBX resolves to unrelated existing textures/materials, set the ModelImporter to local/external material resolution and reimport before organizing.
   - Use per-model material names such as `ModelName_Mat.mat`.

4. Fix texture ownership and names.
   - Move each model's texture set into `Textures/<category>/<ModelName>/`.
   - Rename standard PBR files to `ModelName_BaseColor`, `ModelName_Normal`, `ModelName_Metallic`, and `ModelName_Roughness`.
   - Move `.meta` files with their assets to preserve GUID references.

5. Fix normal texture import settings.
   - Set every `*_Normal.*` texture to Unity `TextureImporterType.NormalMap`.
   - Set normal textures to non-sRGB.
   - Do not treat a normal-map problem as a mesh-normal problem unless the texture type has already been verified.

6. Wire PBR material maps for URP.
   - Use `Universal Render Pipeline/Lit` unless the project uses a different established shader.
   - Assign BaseColor to `_BaseMap` and Normal to `_BumpMap`.
   - URP Lit has no separate Roughness slot: pack `Metallic` into RGB and `1 - Roughness` into Alpha as `ModelName_MetallicSmoothness.png`.
   - Set the packed texture to Default, non-sRGB, alpha from input, then assign it to `_MetallicGlossMap`.
   - Enable `_NORMALMAP` and `_METALLICSPECGLOSSMAP`, set metallic workflow, set `_SmoothnessTextureChannel` to Metallic Alpha, and use the user's requested smoothness value if given.

7. Generate low-poly versions when requested.
   - Reduce each high-poly prop to about 1000-2000 triangles unless the user gives a different target; use 1500 as a good default for small candy/character props.
   - Preserve UVs and material references.
   - Prefer a UV-preserving Blender decimate pass for generated FBX files; mark UV seams/sharp edges before decimation when possible.
   - Save reduced FBX files as `Models/LowPolyFBX/ModelName_LowPoly.fbx` when exporting from Blender, or save Unity mesh assets as `Models/LowPolyMeshes/ModelName_LowPoly.asset` when reducing inside Unity.
   - Save prefabs as `Prefabs/<category>/ModelName_LowPoly.prefab` or `Prefabs/Generated/ModelName_LowPoly.prefab`.
   - Assign the cleaned material to the low-poly prefab; do not rely on the low-poly FBX importer to recreate materials.

8. Clean old shells.
   - Remove empty old `.fbm` folders after textures are moved.
   - Remove empty temporary folders only after verifying they contain no non-`.meta` assets.
   - For generated FBX imports, delete duplicate extracted `.fbm` and `Materials` shells only after the organized material and textures are verified and the source FBX no longer references them.
   - If Unity created malformed underscore folders from non-ASCII names, confirm they have no real assets before deleting them.

## Unity/MCP Notes

- Use Unity MCP `execute_code` for AssetDatabase validation and importer changes when connected.
- If moving assets through the filesystem, always move asset and `.meta` pairs together.
- If raw Chinese strings in generated Unity C# cause name mangling, use Unicode escapes or filesystem moves that preserve Unicode names.
- If Unity or MCP is busy after a large import/refresh, stop issuing new editor scripts, wait for the editor to finish, or ask the user to restart Unity before cleanup.
- For Blender reduction, keep temporary scripts outside final art folders when possible and delete them after validation.

## Validation Checklist

Before final response, verify:

- Expected FBX count in `Models/<category>`.
- Expected material count in `Materials/<category>`.
- Expected texture count in `Textures/<category>`; for PBR sets, usually 4 per model.
- Expected low-poly mesh and prefab counts if generated.
- No root-level leftover FBX from the batch.
- No old `.fbm` folders unless intentionally retained.
- No malformed empty underscore folders.
- Every `*_Normal` texture is `NormalMap` and non-sRGB.
- Materials still reference BaseColor and Normal textures.
- Materials reference the packed MetallicSmoothness texture on `_MetallicGlossMap`; smoothness source is Metallic Alpha.
- Low-poly meshes/prefabs keep valid UVs and use the cleaned material.
- Unity reports no compilation errors.

## Trigger Phrases

When the user says something like "按之前糖果模型那套流程处理这些 FBX", "用 fbx整理", "整理这些 FBX", "fbx整理这个模型", or asks to process an AI-generated model like the gingerbread/Hunyuan output, apply this skill: inspect appearance, rename, isolate materials/textures, set normal maps correctly, pack metallic/roughness for URP, reduce to 1000-2000 triangles if requested, organize folders, and validate Unity.
