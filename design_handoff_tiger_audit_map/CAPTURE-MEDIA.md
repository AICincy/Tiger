# Capture media (run from the live prototypes)

Three of the four 2D map screenshots in `screenshots/` were captured automatically. The 3D map could not be — its rendering uses a WebGL canvas which the headless screenshot tool used to build this handoff cannot read pixels from. To get 3D screenshots and the orbit-sweep GIF, paste the snippets below into the **DevTools Console** of the running prototype.

## Prerequisites

Serve the prototypes from a real HTTP server (relative `fetch()` calls fail on `file://`):

```bash
cd prototypes/
python3 -m http.server 8000
# open http://localhost:8000/map_3d.html in a fresh browser tab
```

Wait until the aerial imagery has fully tiled in (you'll see Blue Ash / Montgomery streets clearly under the colored audit lines) before running anything below.

---

## 1) Six 3D screenshots — one paste

Paste this into the Console on `map_3d.html`. It will trigger six browser downloads named `3d-NN-state.png`. Move them into `screenshots/` next to the 2D ones.

```js
(async () => {
  const wait = (ms) => new Promise(r => setTimeout(r, ms));
  const dl = (name) => new Promise(resolve => {
    map.once('idle', async () => {
      // Give the GPU a beat to finish compositing
      await wait(200);
      map.getCanvas().toBlob((blob) => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = name;
        document.body.appendChild(a);
        a.click();
        a.remove();
        setTimeout(() => URL.revokeObjectURL(url), 1000);
        resolve();
      }, 'image/png');
    });
    // Force at least one render so 'idle' fires
    map.triggerRepaint();
  });

  // We need preserveDrawingBuffer so toBlob() returns real pixels.
  // If you didn't enable it at init, do this small hack: re-create the map.
  // Otherwise comment this block out.
  if (!map.getCanvas().getContext('webgl2')?.getContextAttributes()?.preserveDrawingBuffer
      && !map.getCanvas().getContext('webgl')?.getContextAttributes()?.preserveDrawingBuffer) {
    console.warn('[capture] preserveDrawingBuffer is OFF — your downloads will be blank PNGs.');
    console.warn('[capture] Edit map_3d.html and add `preserveDrawingBuffer: true` to the maplibregl.Map constructor, then reload.');
    return;
  }

  const b = DATA.summary.bounds;
  const fit = (opts = {}) => map.fitBounds(
    [[b.minLon, b.minLat], [b.maxLon, b.maxLat]],
    Object.assign({ padding: 60, duration: 0 }, opts)
  );

  // 1. Default — Esri Sat, pitch 55°, bearing -18°
  document.querySelector('[data-bm="esri-imagery"]')?.click();
  fit({ pitch: 55, bearing: -18 });
  await wait(2500);
  await dl('3d-01-default-imagery.png');

  // 2. Dark basemap, flat
  document.querySelector('[data-bm="dark"]')?.click();
  fit({ pitch: 0, bearing: 0 });
  await wait(2500);
  await dl('3d-02-dark-flat.png');

  // 3. OSM standard, low tilt
  document.querySelector('[data-bm="osm"]')?.click();
  fit({ pitch: 30, bearing: 12 });
  await wait(2500);
  await dl('3d-03-osm-low-tilt.png');

  // 4. Esri Clarity, high tilt looking northeast
  document.querySelector('[data-bm="esri-clarity"]')?.click();
  fit({ pitch: 70, bearing: -45 });
  await wait(2500);
  await dl('3d-04-clarity-high-tilt.png');

  // 5. Critical-only zoom, satellite
  document.querySelector('[data-bm="esri-imagery"]')?.click();
  document.getElementById('btn-critical-3d')?.click() ??
    document.querySelector('button[title*="Critical" i]')?.click();
  await wait(3000);
  await dl('3d-05-critical-zoom.png');

  // 6. Street-level zoom on a representative AB segment
  const ab = DATA.features.find(f => f.properties?.cls === 'AB' && f.properties?.gaps > 0);
  if (ab) {
    const [lon, lat] = ab.geometry.coordinates[Math.floor(ab.geometry.coordinates.length / 2)];
    map.flyTo({ center: [lon, lat], zoom: 18.5, pitch: 65, bearing: 30, duration: 0 });
    await wait(3500);
    await dl('3d-06-street-level-AB.png');
  }

  console.log('[capture] done — six PNGs in your Downloads folder.');
})();
```

> **`preserveDrawingBuffer` is already on.** The flag has been pre-applied to the 3D MapLibre constructor in this repo's copy of `prototypes/map_3d.html`, so the script above downloads real pixels rather than blank PNGs. (If you adapt these scripts for a fresh MapLibre instance elsewhere, the flag is required.)

---

## 2) Orbit-sweep GIF (24 frames, ~3–5 MB)

Paste this on `map_3d.html` after the screenshots above (or by itself). It loads `gif.js` from a CDN, captures 24 frames as the camera bearings sweeps a full 360°, and downloads `3d-orbit.gif`.

```js
(async () => {
  const FRAMES = 24;     // 24 frames @ 100ms = 2.4s loop
  const FRAME_MS = 100;
  const W = 720;         // downscale for size — original is ~1200px
  const H = 540;

  // Load gif.js + its worker
  await new Promise((res, rej) => {
    const s = document.createElement('script');
    s.src = 'https://cdn.jsdelivr.net/npm/gif.js.optimized@1.0.1/dist/gif.js';
    s.onload = res; s.onerror = rej;
    document.head.appendChild(s);
  });

  const wait = (ms) => new Promise(r => setTimeout(r, ms));

  const gif = new GIF({
    workers: 2,
    quality: 8,
    width: W,
    height: H,
    workerScript: 'https://cdn.jsdelivr.net/npm/gif.js.optimized@1.0.1/dist/gif.worker.js',
  });

  // Reset to default frame
  const b = DATA.summary.bounds;
  map.fitBounds([[b.minLon, b.minLat], [b.maxLon, b.maxLat]],
    { padding: 60, pitch: 55, bearing: 0, duration: 0 });
  await wait(2500);

  const tmp = document.createElement('canvas');
  tmp.width = W; tmp.height = H;
  const tctx = tmp.getContext('2d');

  for (let i = 0; i < FRAMES; i++) {
    const bearing = (i / FRAMES) * 360 - 180;
    map.setBearing(bearing);
    await new Promise(r => map.once('idle', r));
    await wait(80);
    tctx.clearRect(0, 0, W, H);
    tctx.drawImage(map.getCanvas(), 0, 0, W, H);
    gif.addFrame(tctx, { copy: true, delay: FRAME_MS });
    console.log(`[orbit] frame ${i + 1}/${FRAMES} @ bearing ${bearing.toFixed(1)}°`);
  }

  gif.on('finished', (blob) => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = '3d-orbit.gif';
    document.body.appendChild(a); a.click(); a.remove();
    console.log('[orbit] saved', (blob.size / 1024).toFixed(0), 'KB');
  });
  gif.render();
})();
```

If you also want a higher-frame-rate MP4 instead of a GIF, replace gif.js with a `MediaRecorder` capturing the canvas stream:

```js
const stream = map.getCanvas().captureStream(30);
const rec = new MediaRecorder(stream, { mimeType: 'video/webm;codecs=vp9' });
const chunks = [];
rec.ondataavailable = (e) => chunks.push(e.data);
rec.onstop = () => {
  const url = URL.createObjectURL(new Blob(chunks, { type: 'video/webm' }));
  const a = document.createElement('a');
  a.href = url; a.download = '3d-orbit.webm';
  document.body.appendChild(a); a.click(); a.remove();
};
rec.start();
let bearing = -180;
const t = setInterval(() => {
  bearing += 2;
  map.setBearing(bearing);
  if (bearing >= 180) { clearInterval(t); rec.stop(); }
}, 50);
```

---

## Notes

- The Esri / USGS / OSIP tile servers occasionally rate-limit aggressive captures. If a basemap looks half-loaded in a screenshot, just re-run the corresponding step.
- Both scripts assume the in-page globals `map` (the MapLibre instance) and `DATA` (the GeoJSON FeatureCollection) — both are exposed by the prototype as written.
- Selectors used (`[data-bm="..."]`, `#btn-critical-3d`) match the prototype's current DOM. If you refactor the markup, update them to match.
