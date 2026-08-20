# Device Mockup Generator

Drop a screen recording or screenshot into a device frame, then export it as a video or a PNG.

Everything runs in the browser. Your file is never uploaded — there are no network calls, no
analytics, and no backend. Closing the tab is the whole cleanup process.

**Live site:** _paste the Vercel URL here once the project is connected_

## Using it

1. Open the site.
2. Drop in a `.mp4`/`.mov`/`.webm` recording, or a `.png`/`.jpg` screenshot.
3. Pick a device, a background and an output size.
4. **Export video** or **Export PNG**.

Two things worth knowing:

- **Video export happens in real time.** A 40-second clip takes 40 seconds to record. The button
  tells you the expected wait before you click it.
- **Transparent video exports are WebM, not MP4.** H.264 has no alpha channel in any browser, so a
  transparent clip has to leave the MP4 family entirely. If you need transparency in Premiere or
  After Effects, WebM/VP9 is the format to ask for.

Chrome and Safari on desktop are the supported browsers. It works on mobile but the panel is cramped
and long video exports are unreliable there.

## Running it locally

No build step, no dependencies, no install. Either open the file directly:

```bash
open index.html
```

…or serve the folder, which is what you want if you're checking export behaviour:

```bash
python3 -m http.server 8000
```

Then visit http://localhost:8000.

## Deploying

The `main` branch is the live site. Vercel builds it on every push — there is no build command, it
serves `index.html` as a static file.

- Push to `main` → production deploy.
- Open a PR → Vercel comments with a preview URL. Use that to check a change before merging.

`vercel.json` sets a few response headers and nothing else. Note that it deliberately does **not**
set a `Content-Security-Policy`: the app loads your media as `blob:` URLs and writes exports the
same way, and a default-tight CSP silently breaks both — uploaded video simply never appears. If you
add a CSP later, it must allow `blob:` and `data:` for `img-src` and `media-src`, and `'unsafe-inline'`
for scripts and styles, and you must re-test a real video export before merging.

## Making changes

It's one self-contained HTML file — markup, styles and a single `<script>` IIFE. That's deliberate:
no toolchain to maintain, and anyone can open it and read the whole thing.

```bash
git checkout -b your-change
# edit index.html
git commit -am "Describe the change"
git push -u origin your-change
gh pr create
```

Before you merge, please check the preview URL with a real video: load a clip, trim it, and export
it. Most regressions in this app show up at export time rather than in the preview.

[`CLAUDE.md`](CLAUDE.md) is the engineering context — the device proportions and where they were
measured from, the lighting model, and a list of mistakes that have already been made once. Read the
relevant section before changing device geometry or shading; those values came from measured product
photography and are not safe to eyeball.
