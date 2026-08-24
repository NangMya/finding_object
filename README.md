# Treasure Radar

A prototype web game. Five crates are scattered around you in a virtual field; a radar
shows the distance and bearing to the closest one. Walk to a crate and tap it to collect it.

**Movement is GPS-only.** Shaking or waving the phone does nothing — you must physically
change location outdoors. The compass only sets facing; it never advances your position.

Everything lives in a single `index.html`. There is no build step, no server, and no
dependencies.

## Running it

**Do not** double-click or share the HTML file on a phone — GPS is blocked for local files
(`file://`, `content://`). Upload to a server and open over **HTTPS**.

- **Phone (real play):** `https://your-domain/…/index.html` — allow Location, walk outdoors.
- **Desktop testing:** `https://…/index.html?debug=1` or localhost with `?debug=1` for keyboard/pad.

## Testing on a phone

Geolocation and compass need a **secure context** (HTTPS or `localhost`). A LAN address
like `http://192.168.x.x:8123` will load the page but GPS/compass stay blocked. Use a tunnel:

```powershell
# 1. serve the folder
python -m http.server 8123 --bind 0.0.0.0

# 2. in a second terminal, expose it over HTTPS
cloudflared tunnel --url http://127.0.0.1:8123 --no-autoupdate
```

Open the printed `https://….trycloudflare.com` URL on the phone.

On the phone: open the **https://** URL (not http), tap **PLAY**, allow **Location**,
stand still until the HUD shows `GPS · locked`, then walk toward the target.

### If you see “Location permission denied”

1. Confirm the address starts with `https://` (plain `http://` is blocked by browsers).
2. Site / browser settings → Location → **Allow** for your domain, then reload.
3. Phone Settings → Location must be ON for the browser app.
4. Some hosts/CDNs send `Permissions-Policy: geolocation=()` which blocks GPS — remove that
   header or allow geolocation for your site.
5. For desktop layout testing only, use `?debug=1` (no GPS required).

## Controls

| Input | Action |
| --- | --- |
| Walking outdoors (GPS) | Real-world displacement moves you 1:1 on the radar |
| Compass | Sets facing — auto-on at hunt start; toggle anytime |
| GPS track (compass off) | Facing is inferred from your walk direction |
| Tap a crate | Collect it once it glows green (within pickup range) |
| `?debug=1` + WASD / pad | Desktop-only movement for layout testing |

The on-screen D-pad is **hidden** unless `?debug=1` is set, so it cannot be used to cheat
on a phone.

## Anti-cheat notes

- Accelerometer / shake detection was removed entirely.
- Position updates only come from `navigator.geolocation.watchPosition`.
- Fixes worse than ±50 ft accuracy are ignored.
- Tiny GPS hops under 3 ft are ignored (standing still / jitter).
- Your starting GPS fix becomes the field center; crates are virtual offsets from there.

Phone GPS is typically ±15–40 ft outdoors. Pickup range is set wide for that reason.
Indoors GPS is usually unusable — this game expects open sky.

## How much space do you need?

Scale is **1 real foot = 1 virtual foot**. Switch presets with:

```js
const PRESET = PRESETS.outdoor;   // or PRESETS.compact
```

| | `outdoor` (default) | `compact` |
| --- | --- | --- |
| Field size | 160 ft | 80 ft |
| Crates spawn | 25–75 ft out | 15–35 ft out |
| Pickup range | 25 ft | 20 ft |
| Open space needed | **~150 ft across** | **~70 ft across** |

## Tuning

```js
const PRESETS = {
  outdoor: { roomSize: 160, pixelScale: 14, pickupRange: 25, minRadius: 25 },
  compact: { roomSize: 80, pixelScale: 24, pickupRange: 20, minRadius: 15 },
};

const GPS_MAX_ACCURACY_FT = 50;  // reject weak fixes
const GPS_MIN_MOVE_FT = 3;       // ignore jitter under this
```

## Saved data

Scores are kept in `localStorage` under the `treasure_radar_save` key — username, best clear
time, cumulative points, and run count. Clearing site data resets everything.

## Notes

This started life as a Next.js app with Prisma/MySQL and an accelerometer pedometer. Both
were removed: there is no backend, and movement is GPS-based so shake-cheating does not work.
The current build is a single `index.html` (emoji and CSS only).
