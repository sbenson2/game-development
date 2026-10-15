# Asset craft: pixel art, 3D assets and AI-assisted generation

Read this when making, reviewing, importing or cleaning up game art: pixel sprites and tiles, 3D models and their textures, the Blender → glTF → Godot 4 path, or generated assets that need judging before use. Style rules, value hierarchies, readability at the gameplay camera and asset-family consistency are in `presentation.md`; this file covers the craft and pipeline that carry those rules out. Engine facts are from the Godot 4.7 documentation; check the project's version.

## Quick diagnosis

| Symptom | Likely cause | Fix | Evidence to collect |
| --- | --- | --- | --- |
| Pixels of uneven width, or blurry sprites | Fractional scale factor; linear filtering; `canvas_items` stretch; VRAM compression | Integer scale mode, Nearest filter, Lossless import, `viewport` stretch | Zoomed screenshot at each target resolution; project and import settings |
| Pixel art shimmers or jitters while the camera moves | Sub-pixel positions; snapping fighting camera smoothing; physics and refresh rates out of step | One snap setting; physics interpolation | 1× video capture with each setting toggled |
| Lines or bleeding at tile edges | Filtering or mipmaps on an atlas without padding | Nearest filter, tile padding, mipmaps off | Zoomed capture at several camera positions |
| Generated "pixel art" looks soft or noisy | Pixels off the grid; many near-duplicate colours; anti-aliased edges | Find the true pixel size, downscale, map to the project palette, repaint by hand | Unique-colour count; measured block sizes |
| Normal-mapped model shows seams or inverted bumps | DirectX-style green channel; baker and engine disagree on tangents | Invert Y on import; MikkTSpace on both sides; export tangents | The test asset in Godot beside the baker's preview |
| Model imports at the wrong size or facing the wrong way | Unapplied transforms; wrong unit assumptions; front not at +Z | Apply transforms, model in metres, face +Z in Godot (−Y in Blender) | A 1 m cube and a character-height reference in the same scene |
| Materials look different in Godot than in Blender | Node setups the exporter does not recognise; AO painted into albedo; wrong colour space | Bake to images; occlusion in its own channel; sRGB only for colour maps | Albedo-only view; Blender and Godot captures under the same light |
| Generated 3D asset is heavy, smeary and lit wrong | Dense iso-surface mesh; lighting baked into colour; missing or uniform material maps | Treat it as a sculpt: retopologise, unwrap, rebake, repaint albedo | Triangle count, wireframe, albedo-only and relit renders |
| A batch of generated assets does not look like one game | Style drift between generations | Shared style rules, curation, one cleanup pass for the whole set | Side by side with three shipped assets in game (`presentation.md`) |

These are starting hypotheses. Confirm each with the capture or count in the last column before changing assets in bulk.

## Pixel art

### Fix the canvas before drawing

Choose the base resolution from the camera and the displays the game must support, then keep it. Godot's documentation notes that fractional factors give uneven pixels (640×360 in a 1366×768 window is about 2.13×), that 640×360 scales by whole numbers to 720p, 1080p, 1440p and 4K, and that typical pixel-art bases run from 256×224 to 640×480 [@godot2026-multiple-resolutions]. By the same arithmetic, 320×180 scales by 4, 6, 8 and 12 to those sizes.

A fixed canvas is a production decision as much as a look. FAITH's artist set a 192×160 canvas, a 15 fps cap and a small on-screen colour count early, and found that keeping them consistent mattered more than accuracy [@smith2019-faith-pixel-art]. On Owlboy, moving from 800×600 to widescreen meant resizing rooms by hand, and with about 800 hero animations a full redraw was impossible, so some new art was simplified to match the old sprites [@madsen2017-owlboy].

Give every sprite, tile, effect and UI element one pixel density: one art pixel becomes one block on screen at the chosen scale. Mixing densities, or rotating and scaling sprites in the engine, breaks a set quickly. Plummer showed diagonal pixel lines breaking up when sprites rotate at native resolution, while horizontal and vertical lines hold [@plummer2020-hi-bit].

Checks: Are the base resolution and integer factor written in the project's notes? Does every asset import at the same pixel density?

### Colour, ramps and lines

- **Palette.** Working on 8-bit-style backgrounds, Ferrari used three to seven colours per object and ordered each ramp from highlight through the turning edge to shadow, so a gradient fill can pre-shade a form. Value, saturation and warm-cool contrast all carry form; even an all-blue scene needs a touch of warmth or it reads as monochrome [@ferrari2016-8bit-graphics]. Bunk indexed a fixed palette of about 32 colours before animating, so one palette edit recoloured every frame [@bunk2018-high-res-pixel-art]. On UFO 50, Plummer followed self-imposed colours-per-tile and sprite limits but broke them where readability needed it [@plummer2020-hi-bit].
- **Dither and banding.** Both speakers moved from hand dither to banding for production reasons. For Ferrari, placing dither pixel by pixel was too slow across many backgrounds; for Bunk, dither was hard to keep aligned from frame to frame [@ferrari2016-8bit-graphics; @bunk2018-high-res-pixel-art].
- **Tools.** Ferrari turns off anti-aliasing and interpolation for selections, pencils, fills and resizing [@ferrari2016-8bit-graphics].
- **Lines.** Research on line rasterization codifies the conventions pixel artists follow by hand: a smooth rhythm of jaggies; preserved symmetry, aspect ratio and sharp angles; and hand-placed anti-aliasing. Image editors' automatic vector-to-raster conversion produces jaggies and broken lines instead [@inglis2012-pixelating-line-art; @inglis2013-superpixelator]. On the square grid, diagonal neighbours touch at a single point, so one-pixel diagonal features fall apart and become ambiguous [@kopf2011-depixelizing].

Hue-shifted ramps, selective outlines, avoiding pillow shading and removing orphan pixels are widespread practitioner conventions with no registered source here. Practitioners also disagree about some of them, hue shifting and outline colour in particular. Treat them as options to test against the game's style rules, not as laws.

Checks: Count each sprite's colours against the project palette. View each asset at 1× on the target screen, and in greyscale to check value contrast (`presentation.md`).

### Readability at native size

Spend detail where it reads. Smith built FAITH's sprites up from black only until they were recognisable, then removed over-produced detail; Plummer added a button with a single pixel [@smith2019-faith-pixel-art; @plummer2020-hi-bit]. Bunk showed a frame that collapsed into an unreadable blob, and checked that the silhouette reads in every pose [@bunk2018-high-res-pixel-art]. Plummer also dropped rendered detail, such as zippers and laces, that would need its own secondary animation, because rendering and animating are separate costs [@plummer2020-hi-bit]. For readability at a farther camera, see `presentation.md` [@zadziuk2017-animation-tricks].

### Animation at low frame counts

- **Key poses first.** Skullgirls' animators snap to the active key pose and let smears, follow-through and overshoot carry the motion within designer-set frame counts [@cartwright2014-skullgirls]. Research on pixel-art animation also builds on key frames, noting that frame-by-frame pixel editing is tedious and error-prone [@kuo2016-pixel-art-animation].
- **Smears.** A large change between frames reads as a blink. A smear shows the motion and can double as a hitbox cue [@bunk2018-high-res-pixel-art].
- **Sub-pixel motion.** Mid-tones between a highlight and its base colour can suggest a half-pixel shift [@bunk2018-high-res-pixel-art].
- **Frame rate as a budget.** FAITH's 15 fps cap kept a rotoscoped workload feasible [@smith2019-faith-pixel-art].
- **Ambient motion.** Owlboy kept static scenes alive with drifting clouds, parallax layers and wind-reactive animated textures [@madsen2017-owlboy].

Frame counts per cycle (walk, run, idle) and per-frame holds are game-specific, and no registered source gives a general table. Tune holds in the engine at gameplay speed, not in an editor preview.

### Tiles

Godot's terrain sets match tiles by corners and sides, by corners only, or by sides only; each tile's peering bits name the terrain expected at each neighbour. Alternative tiles do not inherit the base tile's properties, and Use Texture Padding adds a transparent border per tile to stop bleeding when filtering is on [@godot2026-using-tilesets]. Weight variation tiles with `TileData.probability` instead of scattering them by hand [@godot2026-tiledata]. Edge-matched (Wang) tile sets can fill large areas without a repeating period [@cohen2003-wang-tiles], and constraint-solving generators such as WaveFunctionCollapse work from local adjacency patterns in the same way [@karth2017-wfc].

Checks: Paint a large test area with every variant, then look for repeats and seams at gameplay zoom.

## 3D assets

### Silhouette first, then the budget

Geometry earns its cost at the silhouette; interior detail can live in textures. Appearance-preserving simplification keeps colour in textures and normals in normal maps, so a mesh can be simplified with little or no visible loss [@cohen1998-appearance-preserving]. A coarse, normal-mapped mesh still gives itself away at its polygonal outline [@sander2000-silhouette-clipping]. In Redd's low-poly work, curvature has a point of sufficiency: 16 sides read as a circle, and sphere rings could be halved unnoticed until banding showed. Parts that light and shadow must pass around were modelled in geometry rather than with alpha [@redd2017-low-poly].

No registered source gives triangle budgets that hold across games. Godot picks automatic LODs by screen-space error, 1 px by default [@godot2026-mesh-lod], so on-screen size and the target hardware decide what is affordable. Set budgets per asset class by profiling on the target device.

Topology matters more for editing and deformation than for the GPU. Frostbite's quad-preserving LOD tool kept the edge rings where meshes flex and gave artists a better base to touch up. Artists could paint priority areas such as faces, and hand-made LODs still differed, for example mittens instead of fingers and detail kept in flexing parts [@mason2020-quad-lod].

### UVs, texel density, trims and kits

- **Even sampling.** Minimising texture stretch balances sampling across a surface [@sander2001-texture-mapping-pm], and seams between UV charts cause artifacts and make large painted patterns hard to keep continuous [@levy2002-lscm]. Hold one texel density across a set so neighbouring assets match. The number is a project decision; no registered source gives one.
- **Padding.** Blender's bake margin exists because filtering and mipmapping bleed across UV seams [@blender2026-baking].
- **Trim sheets.** Insomniac's small environment team standardised early on one trim-sheet layout and normal-map structure, for production speed, memory and runtime performance, with a UV-mapping script and a variation shader [@olsen2015-sunset-trims]. On The Ascent, one shared trim texture with mid-poly geometry, weighted normals and mesh decals covered almost all hard-surface assets. A uniform texture and shader setup kept materials swappable and optimisation predictable [@frick2022-the-ascent].
- **Modular kits.** Bethesda defines a kit's footprint first, as a maximum extent: pieces build out only on tiling edges, with a buffer elsewhere. Pivots are fixed early, because moving one breaks every placement already made, and blank grey kits backfired, since early kits need the details that matter [@burgess2016-fallout4-modular]. Fortnite needed standard sizes, such as a minimum doorway width, so collision worked for every character [@ellis2018-fortnite].

### Baking

- **One tangent space end to end.** glTF says clients should compute MikkTSpace tangents when none are supplied [@khronos2026-gltf-spec]. Godot's Ensure Tangents option generates MikkTSpace tangents but prefers exported ones [@godot2026-import-configuration], and Blender's glTF exporter has a Tangents option [@khronos2026-gltf-blender-io].
- **Green channel.** glTF normal maps are +Y, or OpenGL-style [@khronos2026-gltf-spec]. Godot offers Normal Map Invert Y at import for DirectX-style maps [@godot2026-importing-images].
- **Triangulate once, before the bake.** Godot calls import-time triangulation of n-gons unpredictable [@godot2026-model-export], and the exporter triangulates anyway [@khronos2026-gltf-blender-io]. Triangulating in Blender makes the bake and the export agree.
- **High to low.** Blender bakes with Selected to Active, using cage extrusion or a maximum ray distance where the high poly sticks out [@blender2026-baking].

Checks: Bake one test asset and render it in Godot under the game's lighting before batch-baking.

### PBR materials

glTF fixes the conventions. Base colour and emissive are sRGB. The metal-roughness texture is linear, with roughness in green and metal in blue. Occlusion is linear, in red, and applies to indirect light only. Metalness 0 is non-metal and 1 is metal, and intermediate values blend the two [@khronos2026-gltf-spec]. Godot's ORM material takes the packed texture, and its AO map darkens only ambient light [@godot2026-standard-material]. Since occlusion acts only on indirect light, AO painted into albedo also darkens direct light, so keep it in its own channel. Roughness can be filtered from the normal map at import to reduce specular aliasing [@godot2026-importing-images].

### Stylised workflows

Stylised 3D is its own pipeline, not realistic 3D with less detail. Guilty Gear Xrd's artists hand-edited normals on every major feature, because a lit-or-unlit shading step turns any normal noise into blotches. Painted vertex colour offset the shading threshold, vertex data replaced normal maps on characters, outlines were inverted hulls with per-vertex width, and stepped keys without interpolation sold the 2D look [@motomura2015-guilty-gear-xrd]. Team Fortress 2's and Fortnite's style rules are summarised in `presentation.md` [@mitchell2007-tf2; @ellis2018-fortnite].

### LODs and outsourcing

Godot generates LODs at import with the meshoptimizer library. Generation can produce errors, especially on skinned meshes, so it can be turned off per mesh, and hand-authored LODs and HLOD go in visibility ranges [@godot2026-mesh-lod; @godot2026-visibility-ranges]. Halo: Reach's automatic LOD system also kept an off switch for broken cases (the captions are partly garbled) [@wang2011-halo-reach-lod]. The underlying methods are quadric-error edge contraction and progressive meshes [@garland1997-quadric; @hoppe1996-progressive-meshes]. On Horizon Zero Dawn, Guerrilla outsourced objective work (final high-poly detail, baking, LODs) and kept subjective, collaborative work in house [@calvert2018-horizon-characters]. A written spec covering units, texel density, naming and the map set makes such a hand-off checkable.

## Blender → glTF → Godot 4

| Decision | Do | Source |
| --- | --- | --- |
| Format | Prefer glTF 2.0: `.gltf` diffs well, `.glb` is smaller. `.blend` import runs Blender's exporter in the background and needs Blender on every team machine | [@godot2026-3d-formats] |
| Units and axes | 1 unit = 1 m, +Y up. Characters face +Z in Godot, −Y in Blender; the exporter's +Y Up option converts | [@khronos2026-gltf-spec; @godot2026-model-export; @khronos2026-gltf-blender-io] |
| Transforms | Apply them before rigging and export. Blender's Unit Scale changes only the display, so model at true size | [@blender2026-apply-transforms; @blender2026-scene-units; @godot2026-model-export] |
| Materials | Only node setups the exporter recognises survive, so bake the rest to images. Turn on Backface Culling in Blender, or Godot imports the material with culling off | [@khronos2026-gltf-blender-io; @godot2026-3d-formats] |
| Edits after import | Never edit the imported scene; use an inherited scene or fix the source. Extracted `.tres` materials survive reimport, but renaming the material in Blender breaks the link | [@godot2026-import-configuration; @godot2026-advanced-import] |
| Collision and hints | Object-name suffixes such as `-col`, `-convcol`, `-colonly`, `-noimp`, `-rigid`, `-navmesh` and `-occ` set node types at import | [@godot2026-node-suffixes] |
| Lightmaps | Static Lightmaps mode generates UV2 with a lightmap texel size; OBJ cannot carry UV2 | [@godot2026-import-configuration; @godot2026-3d-formats] |
| Textures | Textures used in 3D get VRAM compression and mipmaps; Size Limit helps mobile and web | [@godot2026-importing-images] |

### Godot settings for pixel art

| Setting | Value | Why | Source |
| --- | --- | --- | --- |
| `display/window/stretch/mode` | `viewport` | Recommended for pixel art. `canvas_items` drops the 1:1 mapping of sprite pixels to screen pixels and is the default for projects created in 4.7 or later, so set it explicitly | [@godot2026-projectsettings; @godot2026-multiple-resolutions] |
| `display/window/stretch/scale_mode` | `integer` | Each art pixel becomes an even block (Godot 4.2+). Set `Window.min_size` to the base size and prefer Exclusive Fullscreen | [@godot2026-multiple-resolutions; @godot2026-projectsettings] |
| `rendering/textures/canvas_textures/default_texture_filter` | Nearest | The documented default (1) is not Nearest. A CanvasItem's `texture_filter` inherits from its parent, so one override covers a subtree | [@godot2026-projectsettings; @godot2026-canvasitem] |
| Texture import | Lossless, VRAM compression off, mipmaps off unless the camera zooms far out | VRAM compression shows artifacts in 2D | [@godot2026-importing-images] |
| `rendering/2d/snap/snap_2d_transforms_to_pixel` | On, without vertex snapping as well | Crisper, but motion is less smooth, especially with Camera2D smoothing; read only at startup | [@godot2026-projectsettings] |

`viewport` mode cannot show sub-pixel motion, so if smooth scrolling matters more than strict pixels, the docs suggest `canvas_items` [@godot2026-multiple-resolutions]. Jitter while scrolling mostly comes from physics and refresh rates out of step, or from objects updated in different callbacks, rather than from the art. Physics interpolation addresses it [@godot2026-jitter-stutter].

## AI-assisted generation

### Where it fits

Studies of professionals agree on where generators fit:
- In interviews with 14 Finnish game professionals, most used image generators for inspiration, prototyping and pitching. More than half had made production art with them, yet most said the output could not be used as is, and nearly all edited it [@vimpari2023-adapt-or-die].
- A synthesis of ten qualitative studies found the value clearest in ideation and prototyping, and weakest when assets enter the engine pipeline, where slowdowns appear. Unedited 3D output was rarely usable, and usable results came from curating many generations [@ternar2026-genai-synthesis].
- In a study of 16 designers and developers, participants wanted such tools early in design, traded quality for many variations, and asked for standard, editable formats that fit their existing tools [@fukaya2026-asset-heuristics].
- Romero presented machine-learning outputs such as style transfer as material for exploration, with artifacts and short of production quality; the artist decides when something is ready [@romero2019-deep-learning-tech-art].

Keeping one style across a set is the recurring weakness. Vimpari's interviewees named it [@vimpari2023-adapt-or-die]. It was among the most frequent concerns in an analysis of 3,091 indie-developer posts [@panchanadikar2024-indie-genai], and the synthesis warns of stylistic convergence [@ternar2026-genai-synthesis]. The same indie posts show image generators used for placeholder content [@panchanadikar2024-indie-genai]. An untracked placeholder can ship, so log each one and make its replacement a release check.

### What generators get wrong

| Output | Reported failure | Evidence |
| --- | --- | --- |
| Pixel art | Downscaling then quantising blurs and loses detail; automatic pixelization lacks edge highlighting and selective dithering | [@gerstner2012-pixelated-abstraction] |
| Pixel art | Prompted or fine-tuned diffusion models cannot enforce a strict palette or resolution | [@binninger2024-sd-pixl] |
| Sprites | Many near-identical shades instead of a palette; features leak into back views; quality drops with fewer reference views | [@coutinho2024-sprite-imputation] |
| Sprite animation | Animation frames under palette and grid constraints are left to future work | [@binninger2024-sd-pixl] |
| 3D shape | The same face repeated on several sides (the Janus problem), and appearance drifting between views | [@shi2024-mvdream; @poole2023-dreamfusion] |
| 3D shape | Content painted onto flat surfaces; colour carved into geometry | [@poole2023-dreamfusion] |
| Topology | Iso-surface (marching-cubes) meshes are dense, over-tessellated, oversmoothed or stair-stepped, lose sharp edges and decimate poorly | [@siddiqui2024-meshgpt; @chen2025-meshanything; @boss2025-sf3d] |
| "Artist-like" meshes | Trained on meshes of up to about 800 faces; cleaner topology costs shape accuracy | [@siddiqui2024-meshgpt; @chen2025-meshanything] |
| Textures | Shading and highlights from the input baked into the asset; partial delighting fails in dark regions | [@xiang2025-trellis; @boss2025-sf3d; @wang2024-pbr-boosting] |
| Materials | No material maps, or one roughness and one metallic value for the whole object | [@xiang2025-trellis; @boss2025-sf3d; @zeng2024-paint3d] |
| UVs | Per-vertex colour instead of a UV layout, which forces dense meshes; view-by-view painting leaves seams and conflicting details | [@xiang2025-trellis; @boss2025-sf3d; @richardson2023-texture] |
| Evaluation | CLIP-based scores miss geometry; single metrics disagree with people on 3D plausibility | [@poole2023-dreamfusion; @wu2024-gpteval3d] |

Two consequences for judging output. First, quality claims may rest on renders rather than on the mesh that ships; TRELLIS's user study judged appearance on its Gaussian output [@xiang2025-trellis]. Judge the exported asset in the target engine. Second, no peer-reviewed work found for this file measures UV quality, real-world scale, rig readiness or consistency across a set, so those checks fall to the artist.

### How a professional cleans it up

This order is the skill's synthesis; no registered study tests it.

Pixel art:
1. Find the true pixel size and grid offset, then downscale to native resolution with one colour per block.
2. Map colours to the project palette rather than an automatic one, and count unique colours afterwards.
3. Repaint by hand what automatic methods skip, such as outlines, edge highlights and selective dither [@gerstner2012-pixelated-abstraction]. Remove soft edges the way Bunk cleans exported vector frames: cut slightly past the blur, then add pixels back [@bunk2018-high-res-pixel-art].
4. Clean animation frames together against one palette, and check the silhouette in every pose.
5. Scale for display only by whole numbers, with nearest-neighbour filtering.

3D:
1. Treat the output as a sculpt or a reference, not a game mesh. Retopologise for the asset's job. Field-aligned remeshing gives quad-dominant (not all-quad) output aligned to sharp features [@jakob2015-instant-meshes]. Edge loops for deformation are a separate requirement, so deforming assets need hand-placed loops, while static props can often be decimated.
2. Unwrap the new mesh and rebake normals and colour from the generated mesh onto it, following the baking rules above. Remeshing invalidates the original UVs.
3. Repaint or delight the albedo. Check it alone and under the game's lights, since baked-in lighting is the most consistently reported failure above.
4. Rebuild materials to the glTF channel conventions, and do not assume generated roughness or metalness is meaningful.
5. Set scale, pivot and front axis to the project's conventions, then generate LODs.

Tool note: commercial generators differ in face counts, in whether material maps and lighting removal are on by default, and in their unit and origin options. Record the export settings with each asset.

### Provenance and rights

- **US copyright.** The Copyright Office's 2025 report holds that output is protected only where a human determined sufficient expressive elements, such as perceptible human work within the output or creative arrangement or modification of it. Prompts alone are not enough [@uscopyright2025-part2]. Registration must disclose AI material that is more than de minimis and exclude it from the claim [@uscopyright2023-registration-guidance]. This covers US law only.
- **Steam.** Steam's content survey asks developers to disclose Pre-Generated and Live-Generated AI content, and to describe guardrails for live generation [@valve2026-steam-content-survey].
- **Licences.** Romero warned that a GPLv3 dependency in a machine-learning tool would oblige open-sourcing the product, whereas MIT, Apache 2 and BSD need only credit [@romero2019-deep-learning-tech-art].
- **Records.** Record, per asset, the tool, the inputs, the date and the human edits made. The synthesis recommends provenance records and clear licensing [@ternar2026-genai-synthesis]. Vimpari's interviewees also raised training on artists' work without consent, and concerns about jobs and devalued skills [@vimpari2023-adapt-or-die]. These belong in the production decision, not only the art review.

## What a pro checks before shipping an asset

Pixel assets:
- Drawn at the project's pixel density, and imported with Nearest filtering, Lossless compression and no mipmaps.
- Every colour comes from the project palette.
- Reads at 1× on the target screen, in greyscale, and in every animation pose.
- No soft edges left by scaling or generation, and no in-engine rotation or fractional scaling unless the style allows it.
- Tiles: a large painted test area with every variant shows no seams or obvious repeats.

3D assets:
- Real-world scale in metres, +Y up, front at +Z in Godot; transforms applied; pivot at the agreed point.
- Triangle and material counts within the asset class's budget, measured on target hardware; LOD transitions checked in motion.
- Texel density matches neighbouring assets, and UV padding suits the texture size and its mipmaps.
- Normal map +Y and MikkTSpace tangents, with no seams under the game's lighting.
- Albedo free of baked light and AO; ORM channels correct; colour maps sRGB and data maps linear.
- Imported through the project's import settings, with no edits to the imported scene; name suffixes produce the intended collision.

Every asset:
- Placed in game beside three shipped assets at gameplay camera distance (`presentation.md`).
- Provenance recorded (author, tool, inputs, licence, human edits), and the store's AI disclosure updated if needed.

## Sources

- `godot2026-multiple-resolutions` Godot Engine contributors (2026). Multiple resolutions. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/rendering/multiple_resolutions.html (official documentation)
- `smith2019-faith-pixel-art` Mason Smith (2019). The Art of 'FAITH': Horror at 192x160 Pixels. Game Developers Conference 2019, Design track. https://gdcvault.com/play/1026045/The-Art-of-FAITH-Horror (GDC talk)
- `madsen2017-owlboy` Jo-Remi Madsen and Simon Stafsnes Andersen (2017). 'Owlboy': The Evolving Art of a 10 Year Project. Game Developers Conference 2017, Independent Games Summit. https://gdcvault.com/play/1024144/-Owlboy-The-Evolving-Art (GDC talk)
- `plummer2020-hi-bit` Tyriq Plummer (2020). Skill-Building Series: Pixel Art and Animation in the Hi-Bit Age. GDC Summer 2020, Skill-Building Series (Student Program track). https://gdcvault.com/play/1026805/Skill-Building-Series-Pixel-Art (GDC talk)
- `ferrari2016-8bit-graphics` Mark Ferrari (2016). 8 Bit & '8 Bitish' Graphics-Outside the Box. Game Developers Conference 2016, Design track. https://gdcvault.com/play/1023586/8-Bit-8-Bitish-Graphics (GDC talk)
- `bunk2018-high-res-pixel-art` Kyle Bunk (2018). Animation Bootcamp: High Resolution Pixel Art and Animation. Game Developers Conference 2018, Animation Bootcamp (Visual Arts track). https://gdcvault.com/play/1025042/Animation-Bootcamp-High-Resolution-Pixel (GDC talk)
- `inglis2012-pixelating-line-art` Tiffany C. Inglis and Craig S. Kaplan (2012). Pixelating Vector Line Art. NPAR 2012, International Symposium on Non-Photorealistic Animation and Rendering (Eurographics Association). https://diglib.eg.org/items/3468d785-2fb6-498f-8947-e1fc587f6675 (peer-reviewed)
- `inglis2013-superpixelator` Tiffany C. Inglis et al. (2013). Rasterizing and antialiasing vector line art in the pixel art style. Proceedings of the Symposium on Non-Photorealistic Animation and Rendering (NPAR 2013), ACM. https://doi.org/10.1145/2486042.2486044 (peer-reviewed)
- `kopf2011-depixelizing` Johannes Kopf and Dani Lischinski (2011). Depixelizing pixel art. ACM Transactions on Graphics 30(4) (Proceedings of SIGGRAPH 2011). https://doi.org/10.1145/2010324.1964994 (peer-reviewed)
- `zadziuk2017-animation-tricks` Kristjan Zadziuk et al. (2017). Animation Bootcamp: Tricks of the Trade. Game Developers Conference 2017, Animation Bootcamp. https://gdcvault.com/play/1024320/Animation-Bootcamp-Tricks-of-the (GDC talk)
- `cartwright2014-skullgirls` Mariel Cartwright (2014). Animation Bootcamp: Fluid and Powerful Animation within Frame Restrictions. Game Developers Conference 2014, Animation Bootcamp. https://gdcvault.com/play/1020017/Animation-Bootcamp-Fluid-and-Powerful (GDC talk)
- `kuo2016-pixel-art-animation` Ming‐Hsun Kuo et al. (2016). Feature‐Aware Pixel Art Animation. Computer Graphics Forum 35(7). https://doi.org/10.1111/cgf.13038 (peer-reviewed)
- `godot2026-using-tilesets` Godot Engine contributors (2026). Using TileSets. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/2d/using_tilesets.html (official documentation)
- `godot2026-tiledata` Godot Engine contributors (2026). TileData. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/classes/class_tiledata.html (official documentation)
- `cohen2003-wang-tiles` Michael F. Cohen et al. (2003). Wang Tiles for image and texture generation. ACM Transactions on Graphics 22(3) (Proceedings of SIGGRAPH 2003). https://doi.org/10.1145/882262.882265 (peer-reviewed)
- `karth2017-wfc` Isaac Karth and Adam M. Smith (2017). WaveFunctionCollapse is constraint solving in the wild. Proceedings of the 12th International Conference on the Foundations of Digital Games (FDG '17). https://doi.org/10.1145/3102071.3110566 (peer-reviewed)
- `cohen1998-appearance-preserving` Jonathan Cohen et al. (1998). Appearance-preserving simplification. Proceedings of the 25th annual conference on Computer graphics and interactive techniques - SIGGRAPH '98, pp. 115-122. https://doi.org/10.1145/280814.280832 (peer-reviewed)
- `sander2000-silhouette-clipping` Pedro V. Sander et al. (2000). Silhouette clipping. Proceedings of the 27th annual conference on Computer graphics and interactive techniques - SIGGRAPH '00, pp. 327-334. https://doi.org/10.1145/344779.344935 (peer-reviewed)
- `redd2017-low-poly` Ethan Redd (2017). Low Poly Modeling: Style Through Economy. Game Developers Conference 2017, Game Career Seminar. https://gdcvault.com/play/1024405/Low-Poly-Modeling-Style-Through (GDC talk)
- `godot2026-mesh-lod` Godot Engine contributors (2026). Mesh level of detail (LOD). Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/3d/mesh_lod.html (official documentation)
- `mason2020-quad-lod` Ashton Mason (2020). Quad Mesh Simplification in Frostbite. Game Developers Conference 2020, Programming track. https://gdcvault.com/play/1026752/Quad-Mesh-Simplification-in (GDC talk)
- `sander2001-texture-mapping-pm` Pedro V. Sander et al. (2001). Texture mapping progressive meshes. Proceedings of the 28th annual conference on Computer graphics and interactive techniques (SIGGRAPH 2001), pp. 409-416. https://doi.org/10.1145/383259.383307 (peer-reviewed)
- `levy2002-lscm` Bruno Lévy et al. (2002). Least squares conformal maps for automatic texture atlas generation. ACM Transactions on Graphics 21(3), pp. 362-371 (SIGGRAPH 2002). https://doi.org/10.1145/566654.566590 (peer-reviewed)
- `blender2026-baking` Blender Foundation (2026). Render Baking (Cycles). Blender 5.4 Manual. https://projects.blender.org/blender/blender-manual/raw/branch/main/manual/render/cycles/baking.rst (official documentation)
- `olsen2015-sunset-trims` Morten Olsen (2015). The Ultimate Trim: Texturing Techniques of Sunset Overdrive. Game Developers Conference 2015. https://gdcvault.com/play/1022323/The-Ultimate-Trim-Texturing-Techniques (GDC talk)
- `frick2022-the-ascent` Tor Frick (2022). Building the World of 'The Ascent'. Game Developers Conference 2022, Visual Arts track. https://gdcvault.com/play/1027800/Building-the-World-of-The (GDC talk)
- `burgess2016-fallout4-modular` Joel Burgess and Nathan Purkeypile (2016). 'Fallout 4's' Modular Level Design. Game Developers Conference 2016, Visual Arts track. https://gdcvault.com/play/1022930/-Fallout-4-s-Modular (GDC talk)
- `ellis2018-fortnite` Peter Ellis (2018). Developing the Art of 'Fortnite'. Game Developers Conference 2018. https://gdcvault.com/play/1024936/Developing-the-Art-of-Fortnite (GDC talk)
- `khronos2026-gltf-spec` Khronos Group (2026). glTF 2.0 Specification. Khronos glTF 2.0 Specification, version 2.0.1. https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html (official documentation)
- `godot2026-import-configuration` Godot Engine contributors (2026). Import configuration. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_3d_scenes/import_configuration.html (official documentation)
- `khronos2026-gltf-blender-io` Khronos Group (2026). glTF 2.0 (Blender import-export add-on documentation). glTF-Blender-IO documentation (source of the Blender Manual glTF 2.0 page). https://github.com/KhronosGroup/glTF-Blender-IO/blob/main/docs/blender_docs/scene_gltf2.rst (official documentation)
- `godot2026-importing-images` Godot Engine contributors (2026). Importing images. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_images.html (official documentation)
- `godot2026-model-export` Godot Engine contributors (2026). Model export considerations. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_3d_scenes/model_export_considerations.html (official documentation)
- `godot2026-standard-material` Godot Engine contributors (2026). Standard Material 3D and ORM Material 3D. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/3d/standard_material_3d.html (official documentation)
- `motomura2015-guilty-gear-xrd` Junya Christopher Motomura (2015). GuiltyGearXrd's Art Style : The X Factor Between 2D and 3D. Game Developers Conference 2015, Visual Arts track. https://gdcvault.com/play/1022031/GuiltyGearXrd-s-Art-Style-The (GDC talk)
- `mitchell2007-tf2` Jason Mitchell et al. (2007). Illustrative Rendering in Team Fortress 2. Proceedings of the 5th International Symposium on Non-Photorealistic Animation and Rendering (NPAR 2007). https://doi.org/10.1145/1274871.1274883 (peer-reviewed)
- `godot2026-visibility-ranges` Godot Engine contributors (2026). Visibility ranges (HLOD). Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/3d/visibility_ranges.html (official documentation)
- `wang2011-halo-reach-lod` Xi Wang (2011). Automated Level of Detail Generation for HALO: REACH. Game Developers Conference 2011, Programming track. https://gdcvault.com/play/1014343/Automated-Level-of-Detail-Generation (GDC talk)
- `garland1997-quadric` Michael Garland and Paul S. Heckbert (1997). Surface simplification using quadric error metrics. Proceedings of the 24th annual conference on Computer graphics and interactive techniques - SIGGRAPH '97, pp. 209-216. https://doi.org/10.1145/258734.258849 (peer-reviewed)
- `hoppe1996-progressive-meshes` Hugues Hoppe (1996). Progressive meshes. Proceedings of the 23rd annual conference on Computer graphics and interactive techniques (SIGGRAPH '96), pp. 99-108. https://doi.org/10.1145/237170.237216 (peer-reviewed)
- `calvert2018-horizon-characters` Daniel Calvert (2018). Repopulating the Earth: Character Production on 'Horizon Zero Dawn'. Game Developers Conference 2018, Visual Arts track. https://gdcvault.com/play/1024992/Repopulating-the-Earth-Character-Production (GDC talk)
- `godot2026-3d-formats` Godot Engine contributors (2026). Available 3D formats. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_3d_scenes/available_formats.html (official documentation)
- `blender2026-apply-transforms` Blender Foundation (2026). Apply (Object > Apply). Blender 5.4 Manual. https://projects.blender.org/blender/blender-manual/raw/branch/main/manual/scene_layout/object/editing/apply.rst (official documentation)
- `blender2026-scene-units` Blender Foundation (2026). Scene Properties: Units. Blender 5.4 Manual. https://projects.blender.org/blender/blender-manual/raw/branch/main/manual/scene_layout/scene/properties.rst (official documentation)
- `godot2026-advanced-import` Godot Engine contributors (2026). Advanced Import Settings. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_3d_scenes/advanced_import_settings.html (official documentation)
- `godot2026-node-suffixes` Godot Engine contributors (2026). Node type customization using name suffixes. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_3d_scenes/node_type_customization.html (official documentation)
- `godot2026-projectsettings` Godot Engine contributors (2026). ProjectSettings. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/classes/class_projectsettings.html (official documentation)
- `godot2026-canvasitem` Godot Engine contributors (2026). CanvasItem. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/classes/class_canvasitem.html (official documentation)
- `godot2026-jitter-stutter` Godot Engine contributors (2026). Fixing jitter, stutter and input lag. Godot Engine 4.7 documentation. https://docs.godotengine.org/en/4.7/tutorials/rendering/jitter_stutter.html (official documentation)
- `vimpari2023-adapt-or-die` Veera Vimpari et al. (2023). "An Adapt-or-Die Type of Situation": Perception, Adoption, and Use of Text-to-Image-Generation AI by Game Industry Professionals. Proceedings of the ACM on Human-Computer Interaction 7 (CHI PLAY). https://doi.org/10.1145/3611025 (peer-reviewed)
- `ternar2026-genai-synthesis` Dan-Alexandru Ternar et al. (2026). Generative AI in Game Development: A Qualitative Research Synthesis. Proceedings of the 2026 CHI Conference on Human Factors in Computing Systems. https://doi.org/10.1145/3772318.3791206 (peer-reviewed)
- `fukaya2026-asset-heuristics` Kaisei Fukaya et al. (2026). Heuristics for AI-Driven Graphical Asset Generation Tools in Game Design and Development Pipelines: A User-Centered Approach. International Journal of Human-Computer Interaction. https://doi.org/10.1080/10447318.2026.2632170 (peer-reviewed)
- `romero2019-deep-learning-tech-art` Cesar Romero (2019). Super Powering Technical Artists Using Deep Learning. Game Developers Conference 2019, Visual Arts track. https://gdcvault.com/play/1026027/Super-Powering-Technical-Artists-Using (GDC talk)
- `panchanadikar2024-indie-genai` Ruchi Panchanadikar and Guo Freeman (2024). "I'm a Solo Developer but AI is My New Ill-Informed Co-Worker": Envisioning and Designing Generative AI to Support Indie Game Development. Proceedings of the ACM on Human-Computer Interaction 8 (CHI PLAY). https://doi.org/10.1145/3677082 (peer-reviewed)
- `gerstner2012-pixelated-abstraction` Timothy Gerstner et al. (2012). Pixelated Image Abstraction. NPAR 2012: Proceedings of the 10th International Symposium on Non-Photorealistic Animation and Rendering (Eurographics). https://pixl.cs.princeton.edu/pubs/Gerstner_2012_PIA/index.php (peer-reviewed)
- `binninger2024-sd-pixl` Alexandre Binninger and Olga Sorkine-Hornung (2024). SD-πXL: Generating Low-Resolution Quantized Imagery via Score Distillation. SIGGRAPH Asia 2024 Conference Papers. https://doi.org/10.1145/3680528.3687570 (peer-reviewed)
- `coutinho2024-sprite-imputation` Flávio Coutinho and Luiz Chaimowicz (2024). A Missing Data Imputation GAN for Character Sprite Generation. Anais do XXIII Simpósio Brasileiro de Jogos e Entretenimento Digital (SBGames 2024). https://doi.org/10.5753/sbgames.2024.241116 (peer-reviewed)
- `shi2024-mvdream` Yichun Shi et al. (2024). MVDream: Multi-view Diffusion for 3D Generation. International Conference on Learning Representations (ICLR 2024). https://proceedings.iclr.cc/paper_files/paper/2024/hash/adbe936993aa7cf41e45054d8b72f183-Abstract-Conference.html (peer-reviewed)
- `poole2023-dreamfusion` Ben Poole et al. (2023). DreamFusion: Text-to-3D using 2D Diffusion. International Conference on Learning Representations (ICLR 2023). https://iclr.cc/virtual/2023/oral/12607 (peer-reviewed)
- `siddiqui2024-meshgpt` Yawar Siddiqui et al. (2024). MeshGPT: Generating Triangle Meshes with Decoder-Only Transformers. 2024 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR), pp. 19615-19625. https://doi.org/10.1109/cvpr52733.2024.01855 (peer-reviewed)
- `chen2025-meshanything` Yiwen Chen et al. (2025). MeshAnything: Artist-Created Mesh Generation with Autoregressive Transformers. International Conference on Learning Representations (ICLR 2025). https://proceedings.iclr.cc/paper_files/paper/2025/hash/7fc69f53a6ca2664ef23a9597903ca6f-Abstract-Conference.html (peer-reviewed)
- `boss2025-sf3d` Mark Boss et al. (2025). SF3D: Stable Fast 3D Mesh Reconstruction with UV-unwrapping and Illumination Disentanglement. 2025 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR), pp. 16240-16250. https://doi.org/10.1109/cvpr52734.2025.01514 (peer-reviewed)
- `xiang2025-trellis` Jianfeng Xiang et al. (2025). Structured 3D Latents for Scalable and Versatile 3D Generation. 2025 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR), pp. 21469-21480. https://doi.org/10.1109/cvpr52734.2025.02000 (peer-reviewed)
- `wang2024-pbr-boosting` Yitong Wang et al. (2024). Boosting 3D Object Generation through PBR Materials. SIGGRAPH Asia 2024 Conference Papers. https://doi.org/10.1145/3680528.3687676 (peer-reviewed)
- `zeng2024-paint3d` Xianfang Zeng et al. (2024). Paint3D: Paint Anything 3D with Lighting-Less Texture Diffusion Models. 2024 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR), pp. 4252-4262. https://doi.org/10.1109/cvpr52733.2024.00407 (peer-reviewed)
- `richardson2023-texture` Elad Richardson et al. (2023). TEXTure: Text-Guided Texturing of 3D Shapes. SIGGRAPH 2023 Conference Proceedings (ACM). https://doi.org/10.1145/3588432.3591503 (peer-reviewed)
- `wu2024-gpteval3d` Tong Wu et al. (2024). GPT-4V(ision) is a Human-Aligned Evaluator for Text-to-3D Generation. 2024 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR), pp. 22227-22238. https://doi.org/10.1109/cvpr52733.2024.02098 (peer-reviewed)
- `jakob2015-instant-meshes` Wenzel Jakob et al. (2015). Instant Field-Aligned Meshes. ACM Transactions on Graphics 34(6) (SIGGRAPH Asia 2015), 189:1-189:15. https://doi.org/10.1145/2816795.2818078 (peer-reviewed)
- `uscopyright2025-part2` U.S. Copyright Office (2025). Copyright and Artificial Intelligence, Part 2: Copyrightability. U.S. Copyright Office. https://copyright.gov/newsnet/2025/1060.html (official documentation)
- `uscopyright2023-registration-guidance` U.S. Copyright Office (2023). Copyright Registration Guidance: Works Containing Material Generated by Artificial Intelligence. Federal Register 88 FR 16190 (Document 2023-05321). https://www.govinfo.gov/content/pkg/FR-2023-03-16/html/2023-05321.htm (official documentation)
- `valve2026-steam-content-survey` Valve Corporation (2026). Content Survey (Steamworks Documentation). Steamworks Documentation. https://partner.steamgames.com/doc/gettingstarted/contentsurvey (official documentation)
