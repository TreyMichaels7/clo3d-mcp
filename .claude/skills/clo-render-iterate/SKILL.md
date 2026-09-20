---
name: clo-render-iterate
description: Use when building or adjusting a garment in CLO3D through the clo3d MCP (pattern creation, sewing, arrangement, simulation) and iterating against a reference image without the user driving CLO's UI.
---

# CLO3D iterate loop

Drive CLO3D headlessly: fix → simulate → render → inspect → repeat. CLO's UI is frozen while the plugin serves, so the render is the only feedback.

## Setup (once per session)

1. User loads an avatar in CLO's UI first (File > Open > Avatar). API avatar/`.arr` imports leave the scene unstable and have crashed CLO.
2. User runs `plugin/clo3d_mcp_plugin.py` in Script > Script Editor and says "serving". Output must show `Serving commands in: /tmp/clo3d_mcp`.
3. Verify with `get_project_info`. If it hangs >10s the plugin isn't polling — check `/tmp/clo3d_mcp/request.json` is being consumed.
4. Plugin exits after 1h idle or on `stop_server`. Call `stop_server` at the end of every batch; the user must re-run the script after that.

Harness: tools are named bare here; your harness prefixes them (`mcp__clo3d__get_project_info` in Claude Code, `clo3d_get_project_info` in OpenCode). MCP wiring is `opencode.json` (OpenCode) or `claude mcp add` (Claude Code) — see `docs/agent-onboarding.md`.

## The loop

```
change → SetArrangement (re-seat affected pieces) → simulate 400–600
      → SetCamViewPoint → ExportThumbnail3D → view png → compare to reference
      → save_project → repeat
```

- View the PNG with your harness's file/image reader. If your model has no image input, stop and hand the render path to the user — don't infer the drape from tool output.
- Save the `.zprj` after every pass. CLO has crashed mid-session.
- Cap at ~8 passes before reporting back with renders.

## Rendering (via `call_api`)

| Works | Doesn't |
|---|---|
| `export_api.ExportThumbnail3D(path.png)` — silent capture of 3D window | `ExportSnapshot3D` — opens a dialog, hangs |
| `utility_api.SetCamViewPoint(n)`: 2 = front, 1 = 3/4, 0 = bottom | `ExportTurntableImages` — returns `[]`, no files |

## Geometry rules (verified)

- `create_pattern` points in mm, Y up. Line *i* runs point *i* → *i+1*, last line closes. Get indices with `pattern_api.GetPatternLineInfo(idx)`.
- `AddSeamlinePairGroup(patA, lineA, patB, lineB, dirA, dirB)` pairs **one line to one line**. Segment edges up front so partners match 1:1 (hood neckline, hem band, sleeve cap).
- Direction flags: `true,true` when both lines traverse the seam the same way (endpoint order), `true,false` when opposite. Mirroring a piece flips its world direction relative to a *fixed* partner.
- No delete-seam API. To redo seams, delete the piece (`DeletePatternPiece` — delete the last index to avoid reindexing) and recreate.
- Sewing pockets/lining to a face needs internal lines — not doable via outline seams; `SetPatternHide3D(idx, true)` them.

## Arrangement conventions (the L/R trap)

CLO lays front pieces out as seen from the front (2D right = wearer's left) but back pieces as seen from behind (2D right = wearer's right).

- A piece drawn with +x = "right" that's sewn to the back's +x side lands on the wearer's **left** → assign it `*_L` arrangement points.
- Mirror the **Back** with `FlipPatternPiece(idx, true, false)` so its +x edge sits on the same side as the fronts' +x edge. Seams stay on their lines.
- Sleeves: after fronts/back are consistent, if caps show a fold/inside-face at the shoulder, mirror the sleeves too.
- Height: `_2` points (75%) push a 58cm body into the shoulders; use `_3` (55%). Useful indices on Luka/Adult_V2: Back `Body_Back_Center_3`=44, Fronts `Body_Front_3_L/R`=53/54, Sleeves `Arm_Outside_2_L/R`=20/21, Hood `Head_L/R`=71/72, Cuffs `Wrist_Outside_L/R`=30/31, Hem band `Body_Back_Waist`=45. Confirm with `get_arrangement_list`.
- Re-seat every piece with `SetArrangement` before a sim after geometry changes; otherwise stale positions persist.

## Diagnosing renders

| Symptom | Cause | Fix |
|---|---|---|
| Fronts wrap to the back, hood over face | fronts on wrong side | swap `_L`/`_R` points |
| Diagonal inside-face band across chest | front↔back cross-body twist | mirror Back |
| Fold/grey at sleeve cap | sleeve rotated about arm | mirror sleeves |
| Hem at ribcage, cloth inside torso | pieces arranged too high | `_3` points |
| Twisted grey block at CF hem | band seam directions wrong | rebuild band, recheck flags |
