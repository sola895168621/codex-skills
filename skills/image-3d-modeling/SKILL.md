---
name: image-3d-modeling
description: End-to-end image-to-3D asset pipeline. Use when Codex needs to start from a user-provided or generated reference image, operate a browser-based image-to-3D service such as Hunyuan 3D, download an FBX/3D model, process it in Blender or Maya for decimation or quad retopology, then import and organize it in Unity with correct PBR materials, normal maps, metallic/roughness handling, textures, prefabs, and validation.
---

# 图片3D建模

Use this skill when the user says "图片3D建模", "按图生3D流程", "用混元3D生成模型", or asks to turn an image into a Unity-ready 3D model.

## Goal

Turn a reference image into a usable Unity asset:

1. prepare or validate the image
2. generate/download a 3D model with Hunyuan 3D or a similar web tool
3. process the model in Blender or Maya
4. organize the result in Unity
5. fix materials, PBR textures, normal maps, metallic/roughness, and prefab references
6. verify geometry, UVs, material links, and Unity import state

## Reference Image Rules

Before uploading an image to an image-to-3D service, check whether it is suitable:

- Prefer a single clear subject on a simple or white background.
- Prefer full-body/object visibility, minimal cropping, and no heavy occlusion.
- For characters, prefer front-facing, symmetrical, simple pose, or T-pose when possible.
- Avoid overly human-like detail if the user wants a simple stylized prop or creature.
- If the reference is poor, first create or edit a better image using the available image generation/editing workflow, then upload that result.
- Keep generated reference images in an obvious project or Codex image folder and record the path.

## Browser Generation Workflow

Use browser automation when available.

1. Open the image-to-3D site.
   - For Hunyuan 3D, use `https://3d.hunyuan.tencent.com/`.
   - Use the existing logged-in browser profile when possible.

2. Select the correct generation mode.
   - Choose image-to-3D / 图生3D.
   - Choose single image unless the user provides multiple views.

3. Upload the prepared image.
   - Confirm the image is accepted by the page.
   - If the site shows a size/format limit, convert or downscale the image rather than guessing.

4. Choose model quality/face count.
   - Default to 50k faces when the site offers it and the model will be processed later.
   - Use higher counts only when the user asks for more detail and downstream tools can handle it.
   - Use lower counts only when the user asks for a lightweight result immediately.

5. Generate and download.
   - Wait for generation to finish.
   - Prefer FBX when the target is Unity.
   - Keep the original downloaded file, then copy/move the working FBX into the project art folder.

## Recommended Unity Paths

Unless the user gives a specific target, use:

```text
Assets/Art/Model/
  Models/
    Generated/
      ModelName.fbx
    LowPolyFBX/
      ModelName_LowPoly.fbx
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

Use the user's existing feature folder/category when they point at a specific Unity game art folder.

## DCC Processing Choice

Choose the DCC route based on the user's goal:

- Use Blender for fast game optimization and UV-preserving decimation.
- Use Maya for quad retopology, cleaner topology for animation/modeling, or when the user asks for four-sided polygons.
- Use Unity-only processing only for simple import/material/prefab tasks; do not rely on Unity for robust retopology.

## Blender Route

Use Blender when the goal is "reduce faces", "make low-poly", or "optimize for Unity".

1. Import the generated FBX.
2. Inspect mesh count, material slots, UVs, and texture links.
3. Protect UV seams and sharp edges when decimating.
4. Reduce to the requested range.
   - Small Unity props/characters: 1000-2000 triangles if requested for lightweight gameplay.
   - Medium generated characters: choose a higher target if the user needs silhouette quality.
5. Export a low-poly FBX to `Models/LowPolyFBX/ModelName_LowPoly.fbx`.
6. Do not overwrite the original generated FBX.
7. Validate triangle count and UV count after import into Unity.

## Maya Route

Use Maya when the user asks for retopology, quads, "重新拓扑", "四边面", or texture transfer.

1. Connect to Maya MCP.
   - Use the configured gateway, usually `http://127.0.0.1:9765/mcp`.
   - Search/load scripting or typed Maya tools.
   - Verify `fbxmaya`, `polyRetopo`, and `polyRemesh` are available.

2. Import and preserve source.
   - Import the FBX.
   - Detect mesh transforms by listing mesh shapes and parents; generated FBX can import as `node_0`.
   - Use ASCII-safe Maya node names such as `Gingerbread_High`, `Gingerbread_Retopo_Quad`, and `Gingerbread_PBR_Mat` because MCP-transmitted Chinese node names may become invalid.
   - Group and hide the high-poly source instead of deleting it.

3. Connect Maya PBR material.
   - BaseColor file node: color space `sRGB`, connect to `baseColor`.
   - Metallic file node: color space `Raw`, connect to `metalness`.
   - Roughness file node: color space `Raw`, connect to `specularRoughness`.
   - Normal file node: color space `Raw`, connect through tangent-space `bump2d` to `normalCamera`.

4. Retopologize.
   - Duplicate the high-poly mesh.
   - Use `polyRetopo` for an all-quad result.
   - For generated 50k triangle character-like models, use about 6000 target faces unless the user gives a target.
   - If `polyRetopo` fails, run `polyRemesh` first and then `polyRetopo`.
   - Keep the high-poly source hidden for transfer/baking.

5. Transfer UVs/normals.
   - Use `transferAttributes` from high-poly to retopo.
   - Transfer UVs and normals; do not transfer positions unless the user wants shrinkwrap-style projection.
   - If UV transfer returns zero UVs, try sample spaces `0`, `1`, `4`, then `5`.
   - Use the actual source UV set name; generated FBX may use `UVMap`, not `map1`.
   - Delete construction history after successful transfer.

6. Texture transfer meaning.
   - If the user says "把贴图加上" or "传递贴图" casually, transfer UVs/normals and assign the original PBR material to the retopo model.
   - If the user explicitly asks to bake new texture files, use Maya Transfer Maps or another baking workflow and save actual new PNG files. Do not claim baked textures exist unless files were generated.

## Unity Finalization

After Blender or Maya processing, import or refresh assets in Unity.

1. Move/organize FBX, materials, textures, and prefabs.
2. Isolate each model material as `ModelName_Mat.mat`.
3. Set normal textures to Unity `TextureImporterType.NormalMap` and non-sRGB.
4. Set metallic/roughness source textures to non-sRGB.
5. For URP Lit:
   - BaseColor -> `_BaseMap`
   - Normal -> `_BumpMap`
   - Metallic RGB + inverted Roughness Alpha -> `ModelName_MetallicSmoothness.png`
   - `ModelName_MetallicSmoothness.png` -> `_MetallicGlossMap`
   - Smoothness Source -> Metallic Alpha
6. Create or update prefab.
7. Explicitly assign the cleaned material to the prefab; do not rely on FBX auto-material import.
8. Clean duplicate `.fbm` and temporary `Materials` folders only after references are verified.

## Validation Checklist

Before final response, verify the relevant items:

- Reference image path and uploaded image are known.
- Downloaded FBX path is known.
- Original high-poly source is preserved.
- Blender low-poly output has the target triangle range and valid UVs, if Blender was used.
- Maya retopo output exists, has nonzero UVs, and is all quads when Maya was used.
- Maya material texture file nodes point to BaseColor, Normal, Metallic, and Roughness.
- Unity FBX/material/texture/prefab paths are organized.
- Unity Normal textures are NormalMap and non-sRGB.
- Unity URP material uses packed MetallicSmoothness correctly.
- Prefab references the intended mesh and material.
- Unity compilation/import check is clean when Unity assets were edited.

## Failure Handling

- If browser generation fails, report the site state and preserve any uploaded/generated artifacts.
- If the site downloads a non-FBX format, convert it with Blender or Maya when practical.
- If Maya MCP returns stale or delayed output, send a short marker command to drain/confirm the queue before trusting final stats.
- If retopo creates UV count zero, fix UV transfer before claiming texture transfer is done.
- If only original textures were reassigned and no new baked textures were written, say so explicitly.

## Trigger Phrases

Apply this skill when the user says:

- "图片3D建模"
- "按图生3D流程处理"
- "从这张图生成3D模型并导入Unity"
- "用混元3D生成FBX再整理"
- "图片到混元3D到Unity"
- "生成模型后用 Blender 减面"
- "生成模型后用 Maya 重新拓扑"
