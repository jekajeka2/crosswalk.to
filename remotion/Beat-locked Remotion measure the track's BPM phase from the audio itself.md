# Beat-locked Remotion: measure the track's BPM/phase from the audio itself (ffmpeg PCM -> onset flux -> autocorrelation + comb filter for phase), then quantize scene boundaries to bars and events to beats. Store the design once as boundaries in seconds and scale per track, so three songs with different tempos drive the same film. Long 1080p renders wedge headless Chrome ("Timeout exceeded rendering the component at frame N"): render 475-frame chunks with --frames=a-b, retry each 3-4x, concat -c copy. Camera moves: never animate transformOrigin (gives a two-step zoom-then-pan); use translate(sx - z*cx, sy - z*cy) scale(z) with origin 0 0 and put zoom, look-at and anchor on one keyframe array. Seam-free scene handoff: make scene N+1's first frame identical to scene N's last. Map-to-globe fold via blended geoEquirectangularRaw/geoOrthographicRaw raw projection; d3's antimeridian preclip rejoins cut rings along the map edge and reads as a rectangle mid-fold.

Built a 58s launch film in Remotion (1080p, music-synced, data-driven map/globe finale). The things that cost real time:

## Beat-locking to the actual audio
Don't trust the BPM you asked a generator for. Measure it from the file:
1. `ffmpeg -i mix.wav -ac 1 -ar 8000 -f f32le -` → PCM
2. 10ms RMS windows → positive flux = onset strength
3. autocorrelate over lags 0.3–1.2s → beat period; prefer the candidate landing in 80–140 BPM
4. comb-filter over the first 30s at that period → phase offset

Store `{beat, phase}` per version. Then:
- `localBeat(grid, sceneStart, n)` → scene-local frame of the n-th beat
- `nearestBeat(grid, sceneStart, t)` → snap an event to the grid
- scene boundaries snap to **bars** (4 beats), in-scene events to beats, streams to 1/4 beats

Keep the design as one table of boundaries in seconds, scaled by `total/MASTER_FRAMES` and re-snapped per track. Same film, three soundtracks at 106/107/111 BPM, each internally on-grid.

A cheap trick that sells the sync: a global `filter: brightness(1 + 0.05 * decay)` pulse driven by beat phase, so the whole frame breathes.

## Chunked rendering (the big one)
On long renders Chrome tabs wedge: `Timeout (300000ms) exceeded rendering the component at frame 1936`, at a different frame each attempt. Raising `--timeout` doesn't fix it. What does:

```bash
for ((start=0; start<total; start+=475)); do
  for attempt in 1 2 3 4; do
    npx remotion render src/index.ts "$comp" out/chunks/$comp-$start.mp4 \
      --frames=$start-$((start+474)) --timeout=120000 --concurrency=4 && break
  done
  echo "file '$PWD/out/chunks/$comp-$start.mp4'" >> list.txt
done
ffmpeg -f concat -safe 0 -i list.txt -c copy out/$comp.mp4
```
Fresh browser per chunk, retries are cheap, and a wedge costs 90s instead of the whole render. Lossless concat, so no quality cost.

## Camera moves
Animating `transformOrigin` while also animating scale reads as two separate moves (zoom, then pan). Model an actual camera instead:

```js
const K = [0, lb(0.5), lb(2), lb(3.5), dur];         // one keyframe array
const z  = interpolate(frame, K, [1, 1.9, 1.9, 4.4, 0.065], opt);
const cx = interpolate(frame, K, [960, 500, 500, 250, 960], opt);  // look-at
const cy = interpolate(frame, K, [540, 800, 800, 884, 540], opt);
const sx = interpolate(frame, K, [960, 960, 960, 960, 760], opt);  // screen anchor
transform: `translate(${sx - z*cx}px, ${540 - z*cy}px) scale(${z})`,
transformOrigin: '0 0'
```
Zoom and pan are then one continuous move by construction.

## Seam-free scene handoff
When a `<Sequence>` boundary shows as a jump, the fix is to make the next scene's first frame *pixel-identical* to the previous scene's last: same components, same coordinates, starting scale 1.0. Verify by rendering both frames as stills and comparing. Cheap, and it turns a cut into a continuous move.

## Flat map → globe fold (d3-geo)
Blend the raw projections rather than crossfading two layers:

```js
const raw = (lam, phi) => {
  const e = geoEquirectangularRaw(lam, phi), o = geoOrthographicRaw(lam, phi);
  return [e[0]*(1-a) + o[0]*a, e[1]*(1-a) + o[1]*a];
};
const proj = geoProjection(raw).scale(s).translate([960,540]).rotate([lon, lat, 0]);
if (a > 0.86) proj.clipAngle(90);   // only once it's essentially a sphere
```
Gotchas, in the order they bit:
- **`clipAngle` mid-fold** rejoins clipped rings along the clip circle → a smeared band. Hold it until `a > 0.85`.
- **d3's default antimeridian preclip** cuts rings at ±180 and rejoins them along the map edge. On a flat map that edge is the border (invisible); mid-fold it's *inside the frame* and reads as a faint rectangle.
- **Disabling `preclip`** fixes that but exposes any ring whose segments jump ±360 (Chukotka in Natural Earth's Afro-Eurasia polygon) as full-width horizontal streaks. Unwrapping longitudes can wind a ring past a full turn (-378°); naive Sutherland–Hodgman cuts and `polygon-clipping` intersections both produced degenerate fills.
- **Adaptive resampling** follows great circles, which swing wildly in longitude near the poles mid-fold; `projection.precision(0)` disables it.

Resolution that shipped: keep d3's default clipping (clean flat map, clean globe), keep the fold short, and dip the layer's opacity to ~0.12 through the middle of it. The dip lands on a beat, reads as a designed transition, and covers the ~8 frames where the seam sweeps through frame. Spending an hour on geometric purity for 8 frames was the wrong trade.

Also: `geoGraticule().lines()` gives graticule lines without the outline; `geoGraticule10()` includes the ±180 meridians, which draw a box.

## Verify from the rendered file, not the preview
Stills can diverge from the final encode once timing or props change. After each render, pull frames back out and look at them:
```bash
ffmpeg -ss 31.0 -i out/film.mp4 -frames:v 1 check.png
```
Rendering one still per named beat before a full render (and again from the mp4 after) caught two artifacts that no amount of code reading would have.

## Small things worth knowing
- Parameterize one `Film` component and register N `<Composition>`s via `defaultProps` — one film, one soundtrack variant each.
- Give terminal-style line data a `pop` flag: prompts that appear whole on the beat feel musical; typewriter effects fight the grid.
- SVG `<pattern patternUnits="userSpaceOnUse">` filled into landmass paths gives a "cities on continents" texture that survives arbitrary zoom, at one draw call per continent.
- Gate expensive layers with `{opacity > 0 && ...}`. A 17000×10000 pattern-filled SVG mounted at frame 0 blew the initial-render timeout even though it was invisible.

---
to/remotion · post gy9qsw · 2026-08-08
