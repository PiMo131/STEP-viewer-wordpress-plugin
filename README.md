# ⬡ WP STEP Viewer

> Upload STEP/STP CAD files to WordPress and instantly share them as a fully interactive 3D viewer — no external services, no API keys, no iframe embeds.

![WordPress](https://img.shields.io/badge/WordPress-6.0%2B-blue?logo=wordpress)
![PHP](https://img.shields.io/badge/PHP-7.4%2B-purple?logo=php)
![License](https://img.shields.io/badge/license-GPL--2.0-green)

---

## What it does

Engineering teams upload a `.step` or `.stp` file through a dedicated WordPress admin page. The plugin stores the file on your own server and generates a unique, shareable URL — for example:

```
https://yoursite.com/model/a3f9c2b108e4d1f0
```

Anyone with that link opens a fullscreen 3D viewer directly in the browser, powered by [Online3DViewer](https://github.com/kovacsv/Online3DViewer) and the [occt-import-js](https://github.com/kovacsv/occt-import-js) WASM library. Everything runs on your server — no data leaves your infrastructure.

---

## Screenshots

| Upload interface | 3D Viewer |
|---|---|
| Drag & drop upload with auto-delete timer | Fullscreen viewer with tools panel and NavCube |

---

## Features

### Upload
- Drag & drop or file browser upload of `.step` / `.stp` files (max 256 MB)
- Optional model name/description
- **Auto-delete timer** — set expiry in days (default: 14), or mark as permanent
- Each upload gets a unique 16-character hex URL token
- Dedicated `step_uploader` role — give engineering accounts upload-only access

### 3D Viewer
- Fullscreen, no theme wrapping — works on any WordPress theme
- STEP/STP parsing via WebAssembly (occt-import-js), runs entirely in the browser
- All assets self-hosted — no CDN dependency, works on intranets

### Tools panel (top-left)
| Tool | Description |
|---|---|
| **Bounding box** | X / Y / Z dimensions in mm |
| **Volume** | Approximate volume in mm³ / cm³ / dm³ |
| **Shaded** | Default solid view |
| **Wireframe** | Edge-only view |
| **Perspective toggle** | Switch between perspective and orthographic projection |
| **📏 Afstand** | Click 2 points → distance in mm + dimension line overlay |
| **📐 Hoek** | Click 3 points → angle in degrees |
| **🍌 Banaan voor schaal** | Procedurally generated 3D banana (180 mm × 34 mm) placed next to the model for instant size reference |

### NavCube (bottom-right)
- CSS 3D cube that syncs live with the camera orientation via `requestAnimationFrame`
- Click any face to snap the camera to that view (Front, Back, Left, Right, Top, Bottom)

### Admin
- Model overview with search, pagination, open/download/delete per model
- Expiry date shown per model
- Settings: site title, login-required toggle

---

## How it works technically

```
Browser → yoursite.com/model/{uuid}
         ↓ WordPress template_redirect (bypasses theme)
         ↓ Serves standalone HTML page
              ↓ Loads o3dv.min.js (bundles Three.js + WASM loader)
              ↓ Fetches .step file from /wp-content/uploads/step-files/{uuid}.step
              ↓ occt-import-js.wasm parses STEP → Three.js geometry
              ↓ THREE.js internals extracted via deep-search (isCamera / isScene duck typing)
              ↓ Tools use same THREE instance for correct raycasting + projection
```

**No CORS issues** — the STEP file and all viewer assets are served from the same origin.

**No external THREE.js** — o3dv bundles its own. The plugin extracts `THREE.Vector3` via `gCam.position.constructor` to ensure all raycasting, unprojection and vertex transforms use the same instance as the scene meshes.

---

## Installation

### From ZIP
1. Download the latest release ZIP
2. Go to **WordPress Admin → Plugins → Upload Plugin**
3. Upload the ZIP and click **Activate**
4. Go to **Settings → Permalinks → Save Changes** (flushes rewrite rules)

### Via SSH (recommended for first install)
```bash
rm -rf ~/app/wp-content/plugins/wp-step-viewer
cd ~ && unzip wp-step-viewer.zip -d ~/app/wp-content/plugins/
```

---

## Setup

### 1. Create an upload account for your engineering team
Go to **Users → Add New** and assign the role **STEP Uploader**.

This role can:
- Log in to WordPress admin
- Upload STEP files
- See their own upload history

This role **cannot**:
- Access any other WordPress content
- Delete models (admin only)
- Change settings

### 2. Configure settings (optional)
Go to **STEP Viewer → Instellingen**:
- **Viewer page title** — shown in the browser tab
- **Download button** — show/hide download link on viewer page
- **Login required** — restrict viewer to logged-in users only (for internal use)

---

## File structure

```
wp-step-viewer/
├── wp-step-viewer.php          # Plugin bootstrap
├── includes/
│   ├── class-roles.php         # Custom role + capabilities
│   ├── class-post-types.php    # step_model CPT
│   ├── class-uploader.php      # AJAX upload handler
│   ├── class-viewer.php        # Fullscreen viewer (template_redirect)
│   ├── class-cleanup.php       # Daily WP-cron auto-delete
│   └── class-admin.php         # Admin menu + pages
├── admin/
│   ├── views/                  # Upload, models list, settings pages
│   ├── css/admin.css
│   └── js/admin.js
└── public/
    ├── js/o3dv.min.js          # Online3DViewer 0.18.0 (bundles Three.js r128)
    └── occt/
        ├── occt-import-js.js
        ├── occt-import-js.wasm # ~7 MB — STEP/IGES parser (WebAssembly)
        └── occt-import-js-worker.js
```

---

## Requirements

- WordPress 6.0+
- PHP 7.4+
- Modern browser (Chrome 90+, Firefox 88+, Edge 90+, Safari 15+)
- WebAssembly support (all modern browsers)
- ~8 MB of server disk space for the WASM assets

---

## Upload limits

Default limits (editable in `class-uploader.php`):

| Setting | Default |
|---|---|
| Max file size | 256 MB |
| Allowed extensions | `.step`, `.stp` |
| Auto-delete | 14 days (configurable per upload) |

PHP upload limits (`upload_max_filesize`, `post_max_size`) must also be set accordingly in your server config.

---

## URL structure

Each model gets a permanent URL:
```
https://yoursite.com/model/{16-char-hex-uuid}/
```

The UUID is generated with `bin2hex(random_bytes(8))` at upload time. URLs are routed via WordPress's rewrite system (`post_type = step_model`, `rewrite slug = model`).

---

## Banana for scale 🍌

The plugin includes a procedurally generated 3D banana for scale reference.

- **Dimensions:** 180 mm tall × ~34 mm wide (average banana)
- **Geometry:** Catmull-Rom spline with 38 cross-sections × 12 segments = smooth tube with tapering ends
- **Color:** Yellow vertex colors with 9 hardcoded brown spots + brown stem
- **Placement:** Automatically positioned to the right of the model's bounding box, at floor level
- **No assets required** — built entirely from Three.js BufferGeometry

Enable it with the **🍌 Banaan** button in the tools panel.

---

## Known limitations

- Measurement tools require Three.js internals to be accessible via o3dv's viewer object. If o3dv changes its internal structure in a future version, picking may stop working. The plugin logs a warning in the browser console: `WPSV: cam= scene= meshes=`.
- Volume calculation is approximate (signed tetrahedron summation — accurate for solid, watertight meshes; less accurate for open surfaces).
- The WASM parser (`occt-import-js`) loads ~7 MB on first visit. Subsequent visits are cached by the browser.

---

## Libraries used

| Library | Version | License |
|---|---|---|
| [Online3DViewer](https://github.com/kovacsv/Online3DViewer) | 0.18.0 | MIT |
| [occt-import-js](https://github.com/kovacsv/occt-import-js) | 0.0.22 | MIT |
| Three.js | r128 (bundled with o3dv) | MIT |

---

## License

GPL-2.0+ — see [LICENSE](LICENSE)
