# Spa Pro — how the site works (REA analysis)

Tool: REA 6.4.0 (`npx -y rea-agents@latest analyze-javascript-application . --json`), Node 22.22.
Raw REA output: [`rea-output.json`](rea-output.json). All line numbers refer to `index.html` at commit `013b86f`.

## What REA saw — and what it didn't

REA inventoried 1 file (`index.html`, 169 KB) and mapped its external script references:

| Reference | Line | What it is |
|---|---|---|
| `https://cdn.jsdelivr.net/npm/@emailjs/browser@4/dist/email.min.js` | 13 | EmailJS — sends booking e-mails from the browser |
| `//www.instagram.com/embed.js` | 844 | Instagram post embeds |

REA reports `parsed_javascript_files: 0`: it does **not** parse inline `<script>` blocks, and all of this
site's logic is inline (lines 14–18 and 1363–2731). Everything below was read directly from those
blocks; each claim cites the code.

## Architecture in one picture

```
Visitor browser ── loads index.html (one file: HTML + CSS + JS)
   │
   ├─ localStorage['spapro_site_db']  ← full copy of the "DB" object (cache)
   ├─ Firebase Realtime DB  /spaProDB ← the shared copy; every visitor reads AND writes the whole tree
   ├─ EmailJS                         ← booking e-mail to the owner
   ├─ wa.me link                      ← booking text handed to WhatsApp
   └─ nominatim.openstreetmap.org     ← GPS → street address
```

There is no backend. The "database" is one JS object (`DB`, line 1365) holding `services`, `packages`,
`cities`, `gallery`, `bookings` and `settings` (including the admin password).

## 1. Where the data lives and how it syncs

- **Save** (`saveDB`, 1383–1389): writes the *entire* `DB` to localStorage, then `set(ref(db,'/spaProDB'), DB)`
  — the whole tree is overwritten on every save.
- **Load** (`loadDB`, 1401–1419): read localStorage → `seedData()` → connect to Firebase with a hard-coded
  config (1406–1415).
- **Live sync** (`initFirebase`, 1435–1455): `onValue('/spaProDB')` replaces the local `DB` with the server copy
  whenever it changes (only if it has `_seedV`) and re-renders the page and admin panel.
- **Seeding** (`seedData`, 1480–1554): if the stored `_seedV` is below `SEED_VERSION = 6`, services, packages
  and gallery are wiped and replaced with the built-in defaults. Bumping `SEED_VERSION` therefore deletes
  any admin edits to those lists.

## 2. How a booking works

1. `openBooking` / `bookService` / `bookPackage` (1664–1697) open the 4-step modal.
2. Step 1: services are multi-select, toggled in `selectBookItem` (1752–1789); total = sum of prices (1769).
   Packages are single-select.
3. Step 2–3 validation in `bookStep` (1798–1827): city/date/time required; name/phone/address required;
   phone must match `/^[0-9+\s\-]{7,15}$/` (1816).
4. Optional extras: GPS via `detectLocation` (1971–2005, reverse-geocoded by OpenStreetMap at 1984),
   a pasted map link (`onMapLinkInput`, 1959), and a door photo resized to ≤800 px JPEG base64 (`bookingDoorPhoto`, 2007–2028).
5. `confirmBooking` (2030–2098): pushes the booking into `DB.bookings`, calls `saveDB()` (so it goes to Firebase),
   sends an EmailJS e-mail (`sendBookingEmail`, 1931–1956), then shows a button that opens
   `https://wa.me/<number>?text=<booking text>` (2083–2084). The customer must tap it; nothing is sent to WhatsApp automatically.

## 3. Admin panel

- Opened from the ⚙ button in the footer (line 889 → `openAdmin`, 2105).
- Login (`adminLogin`, 2119–2143) compares the typed password in the browser with
  `DB.settings.adminPass || 'spa2025'`. 5 wrong tries lock it for 5 minutes, but only in that tab (a reload resets it).
- The admin can edit services, packages, cities, gallery, site text (`saveContent`, 2499) and settings
  (`saveSettings`, 2405). Every change goes through `saveDB()` → Firebase.

## 4. Problems found (ranked)

### 🔴 Critical — customer data and the admin password are exposed to every visitor
`initFirebase` (1435) subscribes every anonymous visitor to the whole `/spaProDB` tree. That tree contains
`bookings` (name, phone, address, GPS map link, door photo) and `settings.adminPass`. The site imports no
Firebase Auth (lines 15–16), so it only works if the Realtime Database rules allow **public read and write**.
That means anyone can:
- open DevTools (or fetch `https://spapro-db-default-rtdb.firebaseio.com/spaProDB.json`) and download every
  customer's details and home-door photo, plus the admin password;
- write anything to `/spaProDB` — change prices, the WhatsApp number, or delete everything — without logging in.

I did not query the live database to confirm (no automated reads of production data). You can check it yourself:
open `https://spapro-db-default-rtdb.firebaseio.com/spaProDB.json?shallow=true` in a private window. If it returns
key names rather than `Permission denied`, the data is public.

**Fix direction:** keep bookings out of the public tree (write-only path such as `/bookings/{id}` with rules
`".read": false, ".write": "!data.exists()"`), keep public content in a read-only path, require Firebase Auth for
admin writes, and remove `adminPass` from the database (and from the code: default `'spa2025'` at 1373 and 2126).

### 🟠 High — stored XSS in the admin panel
Booking fields are inserted into HTML without escaping in `renderAdminBookings` (2171–2187: `${b.name}`,
`${b.address}`, `${b.notes}`…). A "customer" who types `<img src=x onerror=…>` as their name runs script in
the admin's browser when the admin opens the panel. The same unescaped pattern is in `renderAdminServices`,
`renderAdminPackages`, `renderAdminCities`, `renderAdminGallery`, `populateCitySelect` and `buildSummary`.
`escHtml` already exists (2116); it just isn't used there.

### 🟠 High — whole-tree overwrite loses data
`saveDB` replaces all of `/spaProDB` with this browser's copy. Two customers booking at the same moment,
or an admin editing while a booking arrives, means the last writer wins and the other booking or edit is lost.
Booking with `push()` to its own path (see fix above) also fixes this.

### 🟡 Bug — the map link never reaches WhatsApp
`confirmBooking` clears `window._bookingMapLink` and `window._bookingDoorPhoto` at 2051–2052, then reads them
at 2066–2067 to build the WhatsApp text. They are always `null` by then, so the 🗺️ map line and the 📸 photo
note never appear in the WhatsApp message. (The e-mail is fine; it uses the `booking` object built earlier.)

### 🟡 Performance — every visitor downloads every door photo
Door photos are stored as base64 inside each booking, and every page view syncs the whole tree. The page
gets slower and Firebase bandwidth grows with every booking.

### ⚪ Minor
- City WhatsApp links assume a leading-0 Saudi number (`'966' + phone.replace(/^0/,'')`, 1643) instead of reusing `normalizeWaNumber` (2377).
- `filterServices('${c}')` (1564) and `filterGallery('${c}')` (2627) break if a category name contains `'`.
- The EmailJS public key and recipient address (1926–1929) are public by design, but anyone can use them to send
  e-mails from your EmailJS quota. Set an allowed-origin / rate limit in the EmailJS dashboard.

## 5. Plain-language summary (5 lines)

1. The whole site is one web page. The services, prices, bookings and settings all live in one shared online notebook (Firebase).
2. When a customer books, the page writes the booking into that notebook, e-mails you, and gives the customer a button that opens WhatsApp with the details.
3. Every visitor's browser downloads the whole notebook, including other customers' names, phones, addresses and door photos, and the admin password.
4. Anyone with basic tech skills can read or change that notebook without logging in. This is the first thing to fix.
5. There is also a small bug: the customer's map location is left out of the WhatsApp message.

## ملخص بالعربي

1. الموقع كله صفحة واحدة. الخدمات والأسعار والحجوزات والإعدادات محفوظة في قاعدة بيانات واحدة مشتركة (Firebase).
2. عند الحجز تُحفظ البيانات في القاعدة، ويُرسل بريد للإدارة، ويظهر للعميلة زر يفتح واتساب بتفاصيل الحجز.
3. ⚠️ كل زائر يحمّل القاعدة كاملة: أسماء العميلات وأرقامهن وعناوينهن وصور أبواب منازلهن، وكلمة مرور الإدارة أيضاً.
4. ⚠️ أي شخص يستطيع قراءة القاعدة أو تعديلها أو حذفها بدون تسجيل دخول، وهذا أهم ما يجب إصلاحه.
5. خطأ بسيط: رابط موقع العميلة على الخريطة لا يصل في رسالة الواتساب.
