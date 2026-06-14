# weatherless

Minimal weather app. One HTML file, no build step, no API keys, no tracking.

Inspired by [newsless.cc](https://newsless.cc) — same philosophy: monospace, centered column, dark/light mode, get the info and move on.

---

## How it works

### Stack

Zero dependencies. Everything runs in a single `index.html`:

- **HTML/CSS/JS** — no framework, no bundler
- **[Nominatim](https://nominatim.openstreetmap.org)** (OpenStreetMap) — free geocoding and reverse geocoding
- **[Open-Meteo](https://open-meteo.com)** — free weather API, no key required
- **[Windy embed](https://embed.windy.com)** — radar iframe, also free

---

## Features

### Search with autocomplete
As you type, the app hits Nominatim's search endpoint (debounced 300ms) and shows up to 6 results in a dropdown. Works globally — cities, ZIP codes, postcodes, addresses.

```
GET https://nominatim.openstreetmap.org/search
  ?q=Pittsburgh
  &format=json
  &limit=6
  &addressdetails=1
```

US results show `City, State ZIP`. International results show `City, Country`. Arrow keys navigate the list, Enter or click picks one.

### Geocoding
Selecting a result gives you the lat/lon directly from Nominatim — no second request needed. For geolocation (browser permission), a reverse geocode call converts coords to a human-readable name:

```
GET https://nominatim.openstreetmap.org/reverse
  ?lat=40.44&lon=-79.99
  &format=json
```

### Weather data
All weather comes from Open-Meteo, queried once per location load:

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude=40.44&longitude=-79.99
  &current=temperature_2m,apparent_temperature,weather_code,
           wind_speed_10m,wind_direction_10m,relative_humidity_2m
  &hourly=temperature_2m,weather_code
  &daily=weather_code,temperature_2m_max,temperature_2m_min
  &temperature_unit=fahrenheit
  &wind_speed_unit=mph
  &timezone=auto
  &forecast_days=7
```

The API always returns °F. The °C toggle converts client-side: `(f - 32) * 5/9`.

### WMO weather codes
Open-Meteo uses [WMO weather interpretation codes](https://open-meteo.com/en/docs#weathervariables). The app maps these to emoji + label locally — no extra API call:

```js
const WMO = {
  0: ["☀️", "Clear sky"],
  61: ["🌧", "Rain"],
  95: ["⛈", "Thunderstorm"],
  // ...
};
```

### Hourly forecast
The API returns 168 hours of hourly data. The app finds the index of the current hour in that array and slices the next 24:

```js
let startIdx = h.time.findIndex(t => t >= c.time);
const hourSlice = h.time.slice(startIdx, startIdx + 24);
```

Displayed as a horizontally scrollable strip.

### Radar
Windy's embed URL takes lat/lon parameters directly. The key implementation detail: the `src` is set imperatively *after* `innerHTML` is written, not baked into the template string. This forces the browser to treat it as a fresh navigation on every location change:

```js
document.getElementById("weather-output").innerHTML = `...
  <iframe id="radar-frame" frameborder="0"></iframe>
...`;
document.getElementById("radar-frame").src = radarSrc; // set after
```

If `src` is in the template string, the browser may reuse the old iframe and not reload it.

### localStorage
Two keys, both set on every relevant change:

| Key | Value | When |
|---|---|---|
| `wl_location` | Last searched query string | On every successful search |
| `wl_unit` | `"F"` or `"C"` | On unit toggle |

On init, `wl_location` is re-fetched through Nominatim to resolve fresh coordinates (stored query → geocode → weather). This avoids stale lat/lon across sessions.

### Geolocation
If no saved location exists, the app tries the browser Geolocation API with a 3-second timeout. On success it reverse-geocodes for a display name. On denial or timeout it does nothing — no error, no prompt.

---

## Design

Same rules as newsless:

- `font: 1em/1.5 monospace` — system monospace, no web fonts
- Single centered column, `max-width: 64ch`
- `@media (prefers-color-scheme: dark)` — automatic dark mode, no toggle needed
- `zoom: 120%` — slight scale-up matching the newsless aesthetic
- No images, no icons (emoji only), no external CSS

---

## Deploy

Drop `index.html` anywhere that serves static files:

```bash
# local
npx serve .

# GitHub Pages, Netlify, Cloudflare Pages — just push the file
```

No server, no backend, no environment variables.

---

## Nominatim usage policy

Nominatim is free but requires a valid `User-Agent` and asks you not to exceed 1 request/second. The 300ms debounce on the search input handles rate limiting. If you expect high traffic, self-host Nominatim or swap in a geocoding provider like Mapbox or Geoapify.

---

## License

GPL-3.0 — same as newsless.
