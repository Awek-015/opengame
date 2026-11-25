> See [source and execution notes](../PROVENANCE.md).

# 3D Assets, Animation & Payload Discipline (Three.js + GLB)

Use for 3D characters, vehicles, props, environments, and animations in Three.js. Pair with [game-stylization.md](game-stylization.md) for cohesive lighting, textures, and skyboxes.

## 1. Capability-Driven 3D Asset Selection

Always check live capability discovery (`openmotion_capability_discover` / `openmotion_media_use`) first — noting each model's live rigging, quality tier, multi-view support, latency, and cost — then route by **screen dominance** rather than sending every object to 3D generation:

- **Hero & focal actors → 3D models (Image-to-3D preferred):** Concentrate high-fidelity 3D models on near-camera focal subjects (player character, boss, hero vehicle, signature gear). Prefer generating a clean concept/turnaround image first and passing it to Image-to-3D over blind Text-to-3D so style and proportions stay locked.
- **Arenas, terrain, obstacles & VFX → procedural geometry + generated 2D PBR textures + lighting:** Keep large architecture, floors, collision geometry, and dynamic effects in Three.js code with seamless generated textures so silhouettes stay crisp up close and colliders match visuals 1:1.

| Priority | Route | Best For | Notes |
| :---: | :--- | :--- | :--- |
| **1** | **Rigged & animated GLB** (live `3d-asset` `operation:"rig_retarget"`, or user/CC0 `.glb` imported into Asset Library) | Humanoid heroes, NPCs, creatures with full action sets | `rig_retarget` chains up to 3 paid tasks (base → Mixamo rig → retarget) — disclose in consent & keep within 1–2 hero actors. Inspect via `game-glb-inspect.mjs`; clone with `SkeletonUtils.clone()`, drive with `AnimationMixer` cross-fades |
| **2** | **Static high-fidelity PBR GLB** (live `3d-asset` `image_to_model` / `text_to_model`, or user/CC0 `.glb` imported via `action:"import"`) | Focal heroes/equipment, vehicles, landmarks, signature props | Prefer `image_to_model` with 1 concept-art `references` entry (auto-upgrades from default). Never pass `quad:true` (returns FBX). Add runtime procedural motion where needed (see [game-procedural-animation.md](game-procedural-animation.md)) |
| **3** | **Procedural PBR Three.js geometry + generated 2D textures/skybox** | Modular environments, roads/bridges, mechanical/non-humanoid props, stylized low-poly worlds | Use bevelled/extruded profiles, PBR `MeshStandardMaterial`/`MeshPhysicalMaterial`, generated seamless tiles + IBL skybox (`PMREMGenerator`). Note: live `3d-world` (`worldlabs-atlas`) currently returns collider/physics `.glb` geometry only (`.spz` splat metadata is stored in `version.result.artifacts`, with no runtime `.spz` renderer yet) — do **not** rely on `3d-world` alone for a visible game environment. |
| **4** | **Bundled CC0 fallback** (`templates/game-kits/characters/Mage.glb`, `Knight.glb`, 12 clips, 41 bones) | Stylized fantasy/action humanoids when no `3d-asset` capability or supplied model is available | Keep `LICENSE-KayKit.txt`; retint materials and attach procedural accessories to bones; state realism ceiling honestly |

**Generating & importing `.glb` models:** When `openmotion_capability_discover` (`action:"list"`, `modality:"3d"`) reports `3d-asset` available, invoke `openmotion_capability_run` (`capabilityId:"3d-asset"`, stable `requestId`, `options.quality`: `hero` / `prop` / `lowpoly`, optional concept image in `references`). Because 3D generation can exceed short MCP client timeouts, **never resubmit with a new `requestId` on timeout** — re-invoke `openmotion_capability_run` with the **identical `requestId` and arguments** to resume from the saved remote task checkpoint without re-billing. Local `.glb` models can also be imported via `openmotion_asset_manage` / `openmotion_assets_use` (`action:"import"`, `sourcePath:"/abs/path/model.glb"`). Both generated and imported GLB 2 assets are automatically inspected into `version.result.glb` (with embedded PNG/JPEG textures WebP-optimized automatically via pure-Node `EXT_texture_webp` when saving/publishing) and served via `/motion-api/assets/<assetId>/versions/<versionId>/media`. Never encode binaries into `openmotion_game_write_files`. For high-importance focal actors, verify grounding, orientation, materials, and attachments in a clean view before layering heavy arena lighting or combat VFX.

## 2. Pre-Integration GLB Inspection Gate

Before wiring any external or generated `.glb` into a scene, inspect it with `node scripts/game-glb-inspect.mjs <file.glb>` (or read `version.result.glb` from an imported library asset):

- **Scale & grounding:** Check `bounds.size` (real world units) and `bounds.minY` (offset by `-box.min.y * scale` so feet/wheels sit flush on `y = 0`). Confirm forward facing (`+Z` vs `-Z`) and `rootTransforms` visually in a capture.
- **Materials & `alphaMode: BLEND` trap:** Check `materials.blendMaterials`. Third-party GLBs often misflag opaque meshes as `alphaMode: BLEND`, disabling depth-write so interior back faces draw over front faces (looks like inverted normals, except lit brightly instead of black). Force `material.transparent = false; material.depthWrite = true;` on parts meant to be opaque, while preserving genuine glass/windshield transparency and PBR roughness/metalness maps.
- **Animations:** Confirm `skins.length >= 1` and non-zero clip durations before referencing clip names; resolve via `THREE.AnimationClip.findByName` with explicit missing-clip handling. For one-shot hit/death clips, set `LoopOnce` + `clampWhenFinished = true`.

## 3. Payload Budget & CSP-Safe Compression

Inspect `totalBytes` and `textureRatio` in `game-glb-inspect.mjs` before optimizing — **on AI-generated 4K GLBs, 90–95% of file size is usually uncompressed PNG textures, not triangles**:

- **Preserve mesh geometry first, optimize textures:** `web_publish` and `exportZip` automatically transcode embedded PNG/JPEG textures inside `.glb` files to **`EXT_texture_webp`** at original resolution while preserving 100% of vertices, skins, and animations (`vendor/GLTFLoader.js` supports `EXT_texture_webp` natively with zero external decoders).
- **Avoid worker-based Draco decoders:** Game Studio's preview CSP enforces `worker-src 'none'`, which blocks Web Worker WASM decoders and external CDN loaders.
- **Keep raw masters out of delivery:** Archive raw 4K working masters or unused concept files under `assets-raw/` (excluded automatically from `web_publish` and `exportZip`), size procedural/terrain textures to their real on-screen resolution, and lazy-load secondary heavy forms (e.g., unlockable transformations/bosses) after initial boot.

## 4. Visual Craft & Performance Budget

- **Lighting, clarity & multi-pass trade-offs:** Derive key directional light, hemisphere fill, skybox/IBL (`PMREMGenerator`), and matching fog from `design/style.md`. Use `ACESFilmicToneMapping` + `SRGBColorSpace` on color/emissive maps only (linear on normal/roughness/metalness). Post-processing and multi-pass effects (bloom, real-time reflections, heavy clones) must serve sharpness and steady frame rate — prefer crisp direct rendering over washed-out/blurry post-processing or multiplied draw cost.
- **Asset rejection safety valve:** Visual cohesion and close-up clarity outrank asset provenance. If an external or generated 3D model reads blurry, fused, or stylistically jarring in live captures, discard it and replace or fall back to clean procedural geometry + lighting rather than defending sunk cost.
- **Draw calls & lifecycle:** Merge static compatible geometry once (`BufferGeometryUtils.mergeGeometries`), use `InstancedMesh` for swarms/props (never merge skinned meshes), cap DPR ≤ 1.5, keep typical scenes within ~100 draw calls / ~100k rendered triangles where practical, expose `renderer.info.render` (`calls`, `triangles`) in `window.__omGame.getStats()` so playtests surface real draw cost, and dispose detached resources cleanly.
- **Headless check note:** Server playtests run SwiftShader WebGL — FPS there is pacing only, not real GPU performance, but blank canvases, console errors, failed GLB/texture requests, and runaway `renderer.info` counts are real failures to fix.
