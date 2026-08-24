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
- **Sandbox (seated, no walking):** `?debug=1&near=1` — virtual movement via D-pad/keys (see below).

## Sandbox testing (sit in one place)

Add **`?debug=1`** to skip GPS and move with the keyboard or on-screen D-pad:

```
https://your-domain.com/index.html?debug=1&near=1
```

| Flag | What it does |
| --- | --- |
| `?debug=1` | No GPS — **WASD** / **arrow keys** (PC) or **▲◀▼▶ pad** (phone) |
| `&near=1` | Crates spawn only **8–18 m** away (few taps to reach AR) |

**Seated test flow:** PLAY → turn with **◀ ▶** toward the yellow arc → tap **▲** to step forward → hear detector beeps → under 20 m AR opens → under 8 m flash lights → tap crate to collect.

Radar + detector work from a local file in debug mode; **AR camera** still needs **HTTPS** or localhost on most phones.

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

## Metal detector + AR discovery

While hunting, the game behaves like a **gold/metal detector**:

- **Distance** — target range shown continuously in **meters** (HUD + detector strip).
- **Audio alarm** — beeps faster and higher-pitched as you get closer (Web Audio).
- **AR view** — within **~20 m**, the rear camera opens and a large crate appears aligned to compass bearing; tap it when green/in range to collect.
- **Flash alert** — within **~8 m**, corner strobe lights pulse with the alarm.

Tap **← RADAR** in AR to return to the map view (AR re-opens when you enter 20 m again).

### Background / screen off

The hunt uses a **background audio persistence** stack so proximity beeps can continue
after the screen locks:

1. **Web Audio** continuous low pulse (keeps the audio thread alive)
2. **HTML5 looping audio** + **Media Session** (helps iOS/Android keep the page warm)
3. Beeps are **scheduled on the AudioContext clock** (not `setInterval`), so they keep
   firing even when the main JS thread is throttled
4. **GPS `watchPosition`** still updates distance on many phones while media is playing
5. Screen **Wake Lock** when the OS allows it

**Zones:** distance always shown · alarm beeps **only at ≤ 20 m** (faster as you close in) · rapid alarm + vibrate under **~8 m** when screen is on

**Limits:** Some phones still pause web pages after long lock periods. Vibration is often
blocked while locked (OS policy). Keep media volume up; do not force-stop the browser.

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
