# PixelLab: consistent AI pixel art and animation

Read this before any PixelLab work: creating a character, rotating it, animating it or cleaning up its frames. It covers why generated animation frames lose consistency, which PixelLab tool fits a job, how to write the prompt fields, and the palette and scaling steps that keep pixel art crisp. For art direction and readability in general, read `presentation.md` alongside it.

## Quick diagnosis

| Observed problem | Plausible causes | Smallest useful change | Evidence to collect |
| --- | --- | --- | --- |
| A loop jumps or the character changes partway through | The loop was assembled from several calls, each started from another call's generated frame; frames patched with a different seed | Regenerate the cycle as one clip in one call from the approved still | Changed-pixel count between consecutive frames; whether the spikes sit on call boundaries |
| Costume details flicker or vanish | Micro-detail too small for the sprite; description worded differently between calls | Drop the micro-detail; reuse one fixed description string | Frames side by side at native size |
| Colours shimmer between frames | Frames quantized one by one; backgrounds removed per frame | One shared palette for every frame; transparent background at generation time | Colours per frame against colours in the whole clip |
| Legs swap or merge where they cross | Both legs the same colour; a large, detailed sprite | Different colours for near and far limbs; a smaller, simpler sprite | The crossing frames at native size |
| Pixel art looks soft or uneven in the game | Non-integer display scale; filtered scaling | Integer nearest-neighbour scaling | A capture at device resolution |
| Later directions drift from the design | Each direction rotated from the previous result | Rotate every direction from the one approved south-facing reference | Each direction beside the reference |

## Why consistency breaks

Research on generated video ties consistency to a shared reference. Applying an image process to each frame of a video independently often gives results that are inconsistent over time [@lai2018-temporal-consistency]. Text2Video-Zero preserves a foreground object's appearance and identity by making each frame attend to the first frame [@khachatryan2023-text2video-zero]. ControlNet adds spatial conditions such as edges, depth and human pose to a pretrained text-to-image diffusion model [@zhang2023-controlnet]. For cartoons, the ToonCrafter authors report that interpolation methods that assume linear motion struggle with the exaggerated, non-linear motion and occlusion common in cartoons, and that live-action video priors need adapting to the cartoon domain [@xing2024-tooncrafter].

This skill's reading of that work for PixelLab: frames made in one call come from one job with one set of inputs, so they stay close to each other. Separate calls share only the inputs passed to them. When each call starts from the previous call's generated frame, every hop carries the last hop's errors forward, and the largest jumps land on the joins. PixelLab's rotation guide describes the same effect for rotations: rotating step by step from the previous result is easier for the model, but errors accumulate after each generation [@pixellab2026-rotating].

## Choosing the animation tool

| Tool | Use it for | Limits and fields |
| --- | --- | --- |
| Template animation: MCP `animate_character` in template mode | Standard humanoid motion for a character created in PixelLab | The MCP guide lists about 50 humanoid templates, such as `walking-8-frames` and `breathing-idle`. `ai_freedom` runs 0-900, default 0, template mode only [@pixellab2026-mcp]; lower values give more consistent animations [@pixellab2026-api-v2]. |
| Skeleton v3: `POST /v2/animate-with-skeleton-v3`, or `animate_character` in `skeleton-v3` mode | Custom motion from your own poses | One sprite up to 256x256 plus all 18 joints for each output frame, 3 to 15 frames. `description` is what the character looks like and `action` is a short motion label; `first_frame_keypoints` is the pose the sprite is already in; `view` and `direction` describe the sprite. `/v2/estimate-skeleton` estimates a skeleton from an image. 3, 8 and 15 frames cost 2, 3 and 4 generations, and the spec advises one long clip over several short ones [@pixellab2026-api-v2]. The MCP guide says this mode moves the character instead of redrawing it each frame [@pixellab2026-mcp]. |
| Text v3: `POST /v2/animate-with-text-v3`, MCP `animate_image` | Motion with no template or pose data | 4 to 16 frames, even; width x height x frames at most 524,288; the input image is kept as frame 1 [@pixellab2026-animate-text]. Optional `last_frame` end pose, `seed`, and `drift_threshold`, which corrects frames whose foreground drifts from the first frame (0 corrects every frame) [@pixellab2026-api-v2]. |
| PixMiniMax: `POST /v2/animate-pixminimax` | Clips longer than 16 frames | 4 to 40 frames in multiples of 4. `description` is the motion: "describe movement, not appearance". `subject_description`, `initial_pose`, `view` and `direction` are caption fields, read from the image when omitted. `enhance_prompt` rewrites the description when true and defaults to false [@pixellab2026-api-v2]. |
| Pro animation | Several directions of one custom action | Generates directions one after another, using the finished sides as reference [@pixellab2026-api-v2]. The Pro text tool returns 16 frames for a 32 or 64 px reference and 4 frames from 65 px up [@pixellab2026-animate-text-pro]. |

The skill's preference order, as reasoning: pose-driven tools first (template mode with `ai_freedom` 0, or skeleton v3), because the poses fix the structure and the job only has to keep the look. Next, text v3 with a fixed `seed` and a low `drift_threshold`. Use PixMiniMax when a single clip needs more than 16 frames, so the clip still comes from one job.

## Writing the prompt fields

**Approve a still first.** Generate or edit one base frame, get the developer's approval, then animate from that exact file. Character creation treats whatever the reference faces as south, so a reference for rotations must face straight-on toward the viewer [@pixellab2026-mcp].

**Description: a short appearance noun phrase.** The skeleton v3 spec defines `description` as what the character looks like, as a noun phrase: colours, clothing, held items, not pose, motion, style or background [@pixellab2026-api-v2]. A workable pattern is build + role + three or four colour blocks + one or two silhouette features + face, for example "stocky knight, silver helmet, blue tabard, brown boots, red plume, visor down". Use the byte-identical string in every call for that character, and put motion only in the motion field (`action`, or PixMiniMax's `description`). Rewording the description between calls invites the drift this file is about.

**Size and detail.** v3 character sizes run from 32 to 256 px, and the spec notes that prompt complexity scales with size [@pixellab2026-api-v2]. The MCP `create_character` size defaults to 48 px [@pixellab2026-mcp]. For animated characters, stay near that default with low detail. Drop micro-detail such as polka dots, thin face paint and strings, which a few pixels cannot hold steadily. Give near and far limbs different colours so crossing legs stay readable.

**Other controls.** `create_character` takes `proportions` presets such as `heroic` [@pixellab2026-mcp]. In template mode, `animate-character` takes `outline`, `shading`, `detail` and `text_guidance_scale` (default 8); the endpoint also takes a `color_image` palette reference with `force_colors` [@pixellab2026-api-v2].

## Palette, background and scaling

- **One shared palette.** `reduce-colors` quantizes several frames together onto one shared palette, so an animation's frames do not drift apart; `palette_image` forces an existing game palette [@pixellab2026-api-v2]. Several generation endpoints also take a `target_palette` of 2 to 256 hex colours (beta) [@pixellab2026-api-v2]. Quantizing frames one at a time gives each its own palette.
- **Restore the grid.** `unzoom` converts upscaled pixel art back to its true pixel grid [@pixellab2026-api-v2].
- **Background at generation time.** Request a transparent background in the generation call (`no_background`) rather than removing backgrounds frame by frame afterwards, which cuts each frame's edge differently.
- **Integer scaling only.** Display sprites at whole-number multiples with nearest-neighbour sampling. A non-integer factor draws some source pixels wider than others, and the unevenness changes as the sprite moves.
- **No generic frame interpolation for in-betweens.** The ToonCrafter authors report that correspondence-based interpolation, which implicitly assumes linear motion, often struggles with the large, non-linear motion of cartoons [@xing2024-tooncrafter]. This skill treats generic video interpolators (RIFE-type tools) as that kind of method. Generate the clip with more frames instead, or play it at a lower frame rate.

## Workflow for a vague brief

1. Turn the developer's words into the fields above: the appearance noun phrase, size, view and direction, and palette.
2. Generate stills and show them. Animate nothing until one is approved.
3. Animate one complete clip per call from the approved still. If a frame is wrong, regenerate the whole clip; do not chain clips or patch frames with another seed.
4. Run one shared palette pass over every frame together.
5. Review the frames at native size and at the game's integer scale. Count changed pixels between consecutive frames; a spike at a boundary marks a seam.
6. Keep each call's request, seed and job id with the asset, so the result can be reproduced or regenerated.

## What to collect

- The request body of every call, including the exact description string and seed.
- Changed-pixel counts between consecutive frames, and where the largest ones fall.
- The colour count per frame and for the whole clip.
- Captures at native size and at the display scale on the target device.

## Sources

- `lai2018-temporal-consistency` Wei-Sheng Lai et al. (2018). Learning Blind Video Temporal Consistency. Computer Vision: ECCV 2018, Lecture Notes in Computer Science. https://doi.org/10.1007/978-3-030-01267-0_11 (peer-reviewed)
- `khachatryan2023-text2video-zero` Levon Khachatryan et al. (2023). Text2Video-Zero: Text-to-Image Diffusion Models are Zero-Shot Video Generators. 2023 IEEE/CVF International Conference on Computer Vision (ICCV). https://doi.org/10.1109/iccv51070.2023.01462 (peer-reviewed)
- `zhang2023-controlnet` Lvmin Zhang et al. (2023). Adding Conditional Control to Text-to-Image Diffusion Models. 2023 IEEE/CVF International Conference on Computer Vision (ICCV). https://doi.org/10.1109/iccv51070.2023.00355 (peer-reviewed)
- `xing2024-tooncrafter` Jinbo Xing et al. (2024). ToonCrafter: Generative Cartoon Interpolation. ACM Transactions on Graphics (SIGGRAPH Asia 2024). https://doi.org/10.1145/3687761 (peer-reviewed)
- `pixellab2026-rotating` PixelLab (2026). Rotating a character. PixelLab documentation. https://www.pixellab.ai/docs/guides/rotating-a-character (official documentation)
- `pixellab2026-mcp` PixelLab (2026). PixelLab MCP tools. PixelLab MCP documentation. https://api.pixellab.ai/mcp/docs (official documentation)
- `pixellab2026-api-v2` PixelLab (2026). PixelLab API v2. PixelLab API documentation (OpenAPI specification). https://api.pixellab.ai/v2/openapi.json (official documentation)
- `pixellab2026-animate-text` PixelLab (2026). Animate with text (New). PixelLab documentation. https://www.pixellab.ai/docs/tools/animate-with-text-new (official documentation)
- `pixellab2026-animate-text-pro` PixelLab (2026). Animate with text (Pro). PixelLab documentation. https://www.pixellab.ai/docs/tools/animate-with-text-pro (official documentation)
