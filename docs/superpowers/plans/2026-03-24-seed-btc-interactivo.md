# MatheyBTC Seed BTC — Interactive Explainer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-page interactive explainer showing the complete 25-step Bitcoin process from entropy to address, with a fixed educational example and an optional live WebCrypto generator.

**Architecture:** Everything inline in `index.html` — HTML, CSS, JS, vendored crypto libraries, BIP39 wordlist, encoders. No build step, no backend, works offline. Two modes: Mode A (pre-calculated fixed example) and Mode B (live computation via WebCrypto + @noble libs). ES/EN i18n via `applyLang()`.

**Tech Stack:** Vanilla HTML/CSS/JS · @noble/hashes (vendored inline) · @noble/secp256k1 (vendored inline) · BIP39 wordlist (2048 words inline) · Base58Check + Bech32/Bech32m (inline ~90 lines) · GitHub Pages

---

## File Structure

```
index.html   ← entire app, all inline. Sections in order:
             1. <head> CSS variables + base styles
             2. <body> header + progress bar + step cards
             3. <script> vendored libs (noble-hashes, noble-secp256k1)
             4. <script> BIP39 wordlist array (2048 words)
             5. <script> encoders: Base58Check, Bech32, Bech32m
             6. <script> fixed example data (EXAMPLE object)
             7. <script> live computation engine (compute*)
             8. <script> i18n (TRANSLATIONS + applyLang)
             9. <script> UI wiring (mode toggle, passphrase, progress bar)
```

---

## Verification approach (browser app — no test framework)

Each task that adds computation includes a **console verification step**: open `index.html` in browser, open DevTools console, run the provided snippet, confirm output matches known BIP39 test vector.

**BIP39 official test vector used throughout:**
- Entropy: `00000000000000000000000000000000` (16 zero bytes = 128 bits)
- Mnemonic: `abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about`
- Passphrase: `""` (empty)
- Root seed: `5eb00bbddcf069084889a8ab9155568165f5c453ccb85e70811aaed6f6da5fc19a5ac40b389cd370d086206dec8aa6c43daea6690f20ad3d8d48b2d2ce9e38e48` (128 hex chars = 64 bytes)

---

## Task 1: HTML skeleton, CSS, header, phase progress bar

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace placeholder with full skeleton**

Replace the entire `index.html` with the following structure (keep `.nojekyll` untouched):

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>₿ MatheyBTC — De Entropía a Dirección Bitcoin</title>
  <style>
    :root {
      --bg: #0a0a0a; --bg2: #111827; --bg3: #1e293b;
      --border: #334155; --text: #f1f5f9; --text2: #94a3b8;
      --celeste: #67e8f9; --amarillo: #facc15;
      --naranja: #fb923c; --blanco: #f1f5f9;
      --green: #4ade80; --red: #f87171;
    }
    * { margin:0; padding:0; box-sizing:border-box; }
    body { background:var(--bg); color:var(--text); font-family:'Segoe UI',system-ui,sans-serif; }

    /* HEADER */
    .header {
      position:sticky; top:0; z-index:100;
      background:var(--bg2); border-bottom:1px solid var(--border);
      padding:10px 24px; display:flex; align-items:center;
      justify-content:space-between; flex-wrap:wrap; gap:8px;
    }
    .header-logo { font-size:20px; font-weight:800; color:var(--amarillo); text-decoration:none; }
    .header-links { display:flex; gap:16px; align-items:center; flex-wrap:wrap; }
    .header-links a { color:var(--text2); font-size:13px; text-decoration:none; }
    .header-links a:hover { color:var(--text); }
    .header-controls { display:flex; gap:8px; align-items:center; }

    /* TOGGLE BUTTONS */
    .toggle-btn {
      padding:5px 14px; border-radius:20px; border:1px solid var(--border);
      background:transparent; color:var(--text2); font-size:12px;
      cursor:pointer; font-weight:600; transition:all 0.2s;
    }
    .toggle-btn.active { background:var(--amarillo); color:#000; border-color:var(--amarillo); }

    /* PHASE PROGRESS BAR */
    .phase-bar {
      position:sticky; top:49px; z-index:99;
      background:var(--bg2); border-bottom:1px solid var(--border);
      display:flex; padding:0 24px;
    }
    .phase-tab {
      padding:8px 16px; font-size:12px; font-weight:700;
      border-bottom:3px solid transparent; cursor:default;
      opacity:0.4; transition:opacity 0.2s; white-space:nowrap;
    }
    .phase-tab.active { opacity:1; }
    .phase-tab[data-phase="1"] { color:var(--celeste); }
    .phase-tab[data-phase="1"].active { border-color:var(--celeste); }
    .phase-tab[data-phase="2"] { color:var(--amarillo); }
    .phase-tab[data-phase="2"].active { border-color:var(--amarillo); }
    .phase-tab[data-phase="3"] { color:var(--naranja); }
    .phase-tab[data-phase="3"].active { border-color:var(--naranja); }
    .phase-tab[data-phase="4"] { color:var(--blanco); }
    .phase-tab[data-phase="4"].active { border-color:var(--blanco); }

    /* MAIN CONTENT */
    .main { max-width:760px; margin:0 auto; padding:32px 20px 80px; }

    /* SECURITY BANNER */
    .security-banner {
      display:none; background:#1c1207; border:1px solid #854d0e;
      border-radius:8px; padding:12px 16px; margin-bottom:24px;
      font-size:13px; color:#fed7aa; line-height:1.6;
    }
    .security-banner.visible { display:block; }

    /* STEP CARD */
    .step-card {
      border-radius:10px; background:var(--bg2);
      border:1px solid var(--border); margin-bottom:8px;
      border-left:4px solid transparent; padding:20px;
    }
    .step-card[data-phase="1"] { border-left-color:var(--celeste); }
    .step-card[data-phase="2"] { border-left-color:var(--amarillo); }
    .step-card[data-phase="3"] { border-left-color:var(--naranja); }
    .step-card[data-phase="4"] { border-left-color:var(--blanco); }

    .step-header { display:flex; align-items:center; gap:12px; margin-bottom:12px; }
    .step-badge {
      width:28px; height:28px; border-radius:50%; flex-shrink:0;
      display:flex; align-items:center; justify-content:center;
      font-size:12px; font-weight:800; color:#000;
    }
    .step-card[data-phase="1"] .step-badge { background:var(--celeste); }
    .step-card[data-phase="2"] .step-badge { background:var(--amarillo); }
    .step-card[data-phase="3"] .step-badge { background:var(--naranja); }
    .step-card[data-phase="4"] .step-badge { background:var(--blanco); }

    .step-title { font-size:15px; font-weight:700; }
    .step-desc { font-size:13px; color:var(--text2); margin-bottom:14px; line-height:1.6; }

    /* VALUE BOX */
    .value-box {
      background:var(--bg3); border:1px solid var(--border);
      border-radius:6px; padding:12px 14px; font-family:monospace;
      font-size:13px; word-break:break-all; line-height:1.7;
    }
    .value-label { font-size:11px; color:var(--text2); margin-bottom:4px; font-family:'Segoe UI',sans-serif; font-weight:600; text-transform:uppercase; letter-spacing:0.5px; }
    .hl { font-weight:700; }
    .step-card[data-phase="1"] .hl { color:var(--celeste); }
    .step-card[data-phase="2"] .hl { color:var(--amarillo); }
    .step-card[data-phase="3"] .hl { color:var(--naranja); }
    .step-card[data-phase="4"] .hl { color:var(--blanco); }

    /* TECHNICAL EXPAND */
    .tech-toggle {
      margin-top:12px; background:none; border:1px solid var(--border);
      color:var(--text2); font-size:12px; padding:4px 10px;
      border-radius:4px; cursor:pointer;
    }
    .tech-toggle:hover { color:var(--text); border-color:var(--text2); }
    .tech-content {
      display:none; margin-top:10px; padding:12px;
      background:var(--bg3); border-radius:6px;
      font-size:12px; color:var(--text2); line-height:1.8;
      border-left:2px solid var(--border);
    }
    .tech-content.open { display:block; }

    /* ARROW BETWEEN STEPS */
    .step-arrow { text-align:center; color:var(--text2); font-size:18px; margin:2px 0; }

    /* PHASE HEADER */
    .phase-header {
      padding:24px 0 12px; margin-bottom:8px;
      border-bottom:1px solid var(--border);
    }
    .phase-header h2 { font-size:17px; font-weight:800; }
    .phase-header p { font-size:13px; color:var(--text2); margin-top:4px; }

    /* PASSPHRASE FIELD */
    .passphrase-row { margin-bottom:24px; }
    .passphrase-row label { font-size:12px; color:var(--text2); display:block; margin-bottom:6px; font-weight:600; }
    .passphrase-row input {
      background:var(--bg3); border:1px solid var(--border); color:var(--text);
      padding:8px 12px; border-radius:6px; font-size:14px; width:100%; max-width:400px;
    }

    /* FINAL ADDRESS */
    .address-result {
      background:var(--bg3); border:1px solid var(--blanco);
      border-radius:10px; padding:20px; text-align:center; margin-top:16px;
    }
    .address-result .addr { font-size:15px; font-family:monospace; color:var(--blanco); word-break:break-all; }
    .address-result .addr-label { font-size:11px; color:var(--text2); margin-bottom:8px; text-transform:uppercase; letter-spacing:1px; }
  </style>
</head>
<body>
  <!-- HEADER -->
  <header class="header">
    <div style="display:flex;align-items:center;gap:20px;">
      <a href="#" class="header-logo">₿ MatheyBTC</a>
      <nav class="header-links">
        <a href="https://matheybtc.github.io/MatheyBTC-invest-BTC/">Invest DCA</a>
        <a href="https://matheybtc.github.io/MatheyBTC-HW-list/">HW List</a>
      </nav>
    </div>
    <div class="header-controls">
      <button class="toggle-btn active" id="btn-mode-a" onclick="setMode('a')" data-i18n="mode-a">📖 Ejemplo</button>
      <button class="toggle-btn" id="btn-mode-b" onclick="setMode('b')" data-i18n="mode-b">⚡ Generar</button>
      <button class="toggle-btn" id="btn-lang" onclick="toggleLang()">EN</button>
    </div>
  </header>

  <!-- PHASE PROGRESS BAR -->
  <div class="phase-bar">
    <div class="phase-tab active" data-phase="1" id="ptab-1" data-i18n="phase-1">🔵 Seed BIP39</div>
    <div class="phase-tab" data-phase="2" id="ptab-2" data-i18n="phase-2">🟡 Árbol HD</div>
    <div class="phase-tab" data-phase="3" id="ptab-3" data-i18n="phase-3">🟠 Derivación</div>
    <div class="phase-tab" data-phase="4" id="ptab-4" data-i18n="phase-4">⚪ Dirección</div>
  </div>

  <main class="main">
    <!-- SECURITY BANNER -->
    <div class="security-banner" id="security-banner">
      <span data-i18n="security-text">⚠️ Solo educativo...</span>
    </div>

    <!-- PASSPHRASE -->
    <div class="passphrase-row">
      <label data-i18n="passphrase-label">Passphrase opcional (BIP39)</label>
      <input type="text" id="passphrase-input" data-i18n-placeholder="passphrase-placeholder"
             placeholder="Dejar vacío para el estándar" oninput="onPassphraseChange()">
    </div>

    <!-- STEPS INJECTED BY JS -->
    <div id="steps-container"></div>
  </main>

  <script>
    // Scripts will be added in subsequent tasks
    let currentLang = 'es';
    let currentMode = 'a';

    function setMode(m) {
      currentMode = m;
      document.getElementById('btn-mode-a').classList.toggle('active', m === 'a');
      document.getElementById('btn-mode-b').classList.toggle('active', m === 'b');
      document.getElementById('security-banner').classList.toggle('visible', m === 'b');
      renderSteps();
    }

    function toggleLang() {
      currentLang = currentLang === 'es' ? 'en' : 'es';
      document.getElementById('btn-lang').textContent = currentLang === 'es' ? 'EN' : 'ES';
      applyLang();
    }

    function applyLang() { /* populated in Task 8 */ }
    function renderSteps() { /* populated in Task 5 */ }
    function onPassphraseChange() { /* populated in Task 7 */ }
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser, verify**

Open `index.html` directly in browser.
Expected: dark page with header, ₿ MatheyBTC logo, two toggle buttons (Ejemplo/Generar), EN toggle, 4 phase tabs, empty main area.

- [ ] **Step 3: Commit**

```bash
cd "D:/claude/Projects/MatheyBTC-seed-BTC"
git add index.html
git commit -m "feat: HTML skeleton, CSS, header, phase progress bar"
git push
```

---

## Task 2: Vendor crypto libraries inline

**Files:**
- Modify: `index.html` (add vendored libs in `<script>` tags before closing `</body>`)

The goal is to embed `@noble/hashes` and `@noble/secp256k1` as inline JS so the app works offline without CDN.

- [ ] **Step 1: Download minified library files**

Run in a temp directory (requires Node.js / npm):

```bash
cd /tmp
npm init -y
npm install @noble/hashes@1.6.1 @noble/secp256k1@1.7.1

# Copy the minified bundles — check actual paths:
ls node_modules/@noble/hashes/
ls node_modules/@noble/secp256k1/
```

The files you need are typically:
- `node_modules/@noble/hashes/noble-hashes.min.js` or the `esm/index.js`
- `node_modules/@noble/secp256k1/noble-secp256k1.js`

If minified files aren't present, use the CDN to download them:
```bash
curl -o noble-hashes.js "https://unpkg.com/@noble/hashes@1.6.1/esm/index.js"
curl -o noble-secp256k1.js "https://unpkg.com/@noble/secp256k1@1.7.1/noble-secp256k1.js"
```

- [ ] **Step 2: Add libraries inline to index.html**

Add two `<script>` blocks immediately before the existing `<script>` tag at the bottom of `<body>`:

```html
<!-- VENDORED: @noble/secp256k1 v1.7.1 -->
<script>
/* paste full content of noble-secp256k1.js here */
</script>

<!-- VENDORED: @noble/hashes v1.6.1 -->
<script>
/* paste full content of noble-hashes esm bundle here */
</script>
```

**Important:** `@noble/secp256k1` v1.7.x exposes `nobleSecp256k1` or you access via `secp.getPublicKey(privkey, true)`. Check the library's README for the exact API.

- [ ] **Step 3: Console verification**

Open browser console:
```javascript
// Test noble-secp256k1 is available
const priv = '0000000000000000000000000000000000000000000000000000000000000001';
const pub = secp.getPublicKey(priv, true);
console.log('pubkey bytes:', pub.length); // Expected: 33
console.log('first byte:', pub[0]); // Expected: 2 or 3

// Test noble-hashes is available
const hash = nobleHashes.sha256(new Uint8Array([1,2,3]));
console.log('sha256 length:', hash.length); // Expected: 32
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: vendor @noble/secp256k1 and @noble/hashes inline"
git push
```

---

## Task 3: BIP39 wordlist + Base58Check + Bech32/Bech32m encoders

**Files:**
- Modify: `index.html` (add after vendored libs, before main app script)

- [ ] **Step 1: Add BIP39 wordlist**

Download the official BIP39 English wordlist (2048 words):
```bash
curl -o /tmp/bip39-english.txt "https://raw.githubusercontent.com/trezor/python-mnemonic/master/src/mnemonic/wordlist/english.txt"
```

Then in `index.html`, add a new `<script>` block with the wordlist as an array:

```html
<script>
const BIP39_WORDLIST = [
  "abandon","ability","able","about","above",/* ... all 2048 words ... */,"zoo"
];
// Verify: BIP39_WORDLIST.length === 2048
// Verify: BIP39_WORDLIST[0] === 'abandon', BIP39_WORDLIST[2047] === 'zoo'
</script>
```

- [ ] **Step 2: Add Base58Check encoder**

```html
<script>
// Base58Check encoder
const BASE58_CHARS = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';

function base58Encode(bytes) {
  let n = BigInt('0x' + Array.from(bytes).map(b => b.toString(16).padStart(2,'0')).join(''));
  let result = '';
  while (n > 0n) {
    result = BASE58_CHARS[Number(n % 58n)] + result;
    n = n / 58n;
  }
  for (const b of bytes) {
    if (b !== 0) break;
    result = '1' + result;
  }
  return result;
}

function base58CheckEncode(payload) {
  const checksum = nobleHashes.sha256(nobleHashes.sha256(payload)).slice(0, 4);
  const full = new Uint8Array(payload.length + 4);
  full.set(payload); full.set(checksum, payload.length);
  return base58Encode(full);
}
</script>
```

- [ ] **Step 3: Add Bech32/Bech32m encoder**

```html
<script>
// Bech32 / Bech32m encoder (BIP173 / BIP350)
const BECH32_CHARSET = 'qpzry9x8gf2tvdw0s3jn54khce6mua7l';
const BECH32_GENERATOR = [0x3b6a57b2, 0x26508e6d, 0x1ea119fa, 0x3d4233dd, 0x2a1462b3];

function bech32Polymod(values) {
  let chk = 1;
  for (const v of values) {
    const top = chk >> 25;
    chk = ((chk & 0x1ffffff) << 5) ^ v;
    for (let i = 0; i < 5; i++) if ((top >> i) & 1) chk ^= BECH32_GENERATOR[i];
  }
  return chk;
}

function bech32HrpExpand(hrp) {
  const ret = [];
  for (const c of hrp) ret.push(c.charCodeAt(0) >> 5);
  ret.push(0);
  for (const c of hrp) ret.push(c.charCodeAt(0) & 31);
  return ret;
}

function bech32CreateChecksum(hrp, data, isBech32m) {
  const values = bech32HrpExpand(hrp).concat(data).concat([0,0,0,0,0,0]);
  const mod = bech32Polymod(values) ^ (isBech32m ? 0x2bc830a3 : 1);
  return Array.from({length:6}, (_,i) => (mod >> (5*(5-i))) & 31);
}

function bech32Encode(hrp, data, isBech32m = false) {
  const combined = data.concat(bech32CreateChecksum(hrp, data, isBech32m));
  return hrp + '1' + combined.map(d => BECH32_CHARSET[d]).join('');
}

function convertBits(data, fromBits, toBits, pad = true) {
  let acc = 0, bits = 0;
  const result = [];
  const maxv = (1 << toBits) - 1;
  for (const val of data) {
    acc = (acc << fromBits) | val;
    bits += fromBits;
    while (bits >= toBits) { bits -= toBits; result.push((acc >> bits) & maxv); }
  }
  if (pad && bits > 0) result.push((acc << (toBits - bits)) & maxv);
  return result;
}

// P2WPKH Bech32 address (BIP84, bc1q...)
function p2wpkhAddress(hash160) {
  const words = [0].concat(convertBits(Array.from(hash160), 8, 5));
  return bech32Encode('bc', words, false);
}

// P2TR Bech32m address (BIP86, bc1p...)
function p2trAddress(tweakedPubkeyX) {
  const words = [1].concat(convertBits(Array.from(tweakedPubkeyX), 8, 5));
  return bech32Encode('bc', words, true);
}
</script>
```

- [ ] **Step 4: Console verification**

```javascript
// Verify wordlist
console.log(BIP39_WORDLIST.length);    // Expected: 2048
console.log(BIP39_WORDLIST[0]);        // Expected: 'abandon'
console.log(BIP39_WORDLIST[2047]);     // Expected: 'zoo'

// Verify Base58Check (Bitcoin mainnet xpub version bytes produce 'xpub' prefix)
// A known xpub starts with bytes [0x04, 0x88, 0xB2, 0x1E, ...]
// Just verify no crash:
console.log(base58CheckEncode(new Uint8Array([0x04,0x88,0xB2,0x1E,0,0,0,0,0,0,0,0,0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43]).slice(0,45)).startsWith('xpub'));

// Verify Bech32 (P2WPKH address starts with bc1q)
const fakeHash = new Uint8Array(20).fill(1);
console.log(p2wpkhAddress(fakeHash).startsWith('bc1q')); // Expected: true
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: BIP39 wordlist, Base58Check, Bech32/Bech32m encoders"
git push
```

---

## Task 4: Pre-calculated fixed example data (EXAMPLE object)

**Files:**
- Modify: `index.html`

These are the hardcoded values for Mode A. All derived from BIP39 test vector:
- Mnemonic: `abandon abandon abandon abandon abandon about` × 2 (12 words)
- Passphrase: `""` (empty)
- Derivation path: `m/84'/0'/0'/0/0`

- [ ] **Step 1: Add EXAMPLE data object**

Add this `<script>` block (all values are real BIP39 test vector outputs — implementer must compute using the crypto functions added in Task 2 if any value below is wrong, by running `computeAll()` after Task 6):

```html
<script>
// Pre-calculated values for BIP39 test vector
// Mnemonic: "abandon×11 about", passphrase: "", path: m/84'/0'/0'/0/0
const EXAMPLE = {
  // Phase 1
  entropy:    '00000000000000000000000000000000',
  entropyBits: '0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000',
  checksum:   '0000', // first 4 bits of SHA256(entropy)
  checksumBit: '0000',
  bits132:    '00000000000 00000000000 00000000000 00000000000 00000000000 00000000000 00000000000 00000000000 00000000000 00000000000 00000000000 00000000011',
  indices:    [0,0,0,0,0,0,0,0,0,0,0,3],
  mnemonic:   'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about',
  words:      ['abandon','abandon','abandon','abandon','abandon','abandon','abandon','abandon','abandon','abandon','abandon','about'],
  passphrase: '',
  mnemonicNFKD: 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about',
  rootSeed:   '5eb00bbddcf069084889a8ab9155568165f5c453ccb85e70811aaed6f6da5fc19a5ac40b389cd370d086206dec8aa6c43daea6690f20ad3d8d48b2d2ce9e38e48',

  // Phase 2
  hmacOutput:    'bbe4c4bc...', // HMAC-SHA512("Bitcoin seed", rootSeed) — compute on init
  masterPrivKey: null, // IL — first 32 bytes of hmacOutput
  masterChainCode: null, // IR — last 32 bytes of hmacOutput
  masterPubKey:  null,  // masterPrivKey · G, compressed 33 bytes
  xprv:          null,  // base58check serialization
  xpub:          null,

  // Phase 3
  derivPath:     "m/84'/0'/0'/0/0",
  childPrivKey:  null,  // after full derivation
  childPubKey:   null,  // compressed 33 bytes

  // Phase 4
  hash160:       null,  // RIPEMD160(SHA256(childPubKey))
  address:       null,  // Bech32 bc1q...
};
// Note: null values will be computed by computeAll() called on DOMContentLoaded.
// They are null here because the computation requires the vendored libraries to be loaded first.
</script>
```

- [ ] **Step 2: Console verify entropy → mnemonic mapping**

After Task 6 (computeAll) is done, re-verify this. For now just confirm the structure exists:
```javascript
console.log(EXAMPLE.mnemonic); // 'abandon abandon abandon...'
console.log(EXAMPLE.indices);  // [0,0,0,0,0,0,0,0,0,0,0,3]
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: EXAMPLE data object for BIP39 test vector (Mode A)"
git push
```

---

## Task 5: Step cards HTML + Phase 1 (steps 1–7)

**Files:**
- Modify: `index.html` — populate `renderSteps()` and add step HTML for Phase 1

- [ ] **Step 1: Add step card helper function**

In the main `<script>` block, replace `renderSteps()` stub with:

```javascript
function makeCard(num, phase, titleKey, descKey, valueHtml, techHtml) {
  return `
  <div class="step-card" data-phase="${phase}" data-step="${num}" id="step-${num}">
    <div class="step-header">
      <div class="step-badge">${num}</div>
      <div class="step-title" data-i18n="${titleKey}">—</div>
    </div>
    <div class="step-desc" data-i18n="${descKey}">—</div>
    <div class="value-box">${valueHtml}</div>
    <button class="tech-toggle" onclick="this.nextElementSibling.classList.toggle('open');this.textContent=this.nextElementSibling.classList.contains('open')?(currentLang==='es'?'− técnico':'− technical'):(currentLang==='es'?'+ técnico':'+ technical')" data-i18n="tech-btn">+ técnico</button>
    <div class="tech-content" data-i18n="${techHtml ? techKey(num) : ''}">${techHtml||''}</div>
  </div>
  <div class="step-arrow">↓</div>`;
}

function techKey(n) { return 'tech-' + n; }
```

- [ ] **Step 2: Add renderSteps() for Phase 1**

```javascript
function renderSteps() {
  const d = currentMode === 'a' ? EXAMPLE : (window.LIVE || EXAMPLE);
  const container = document.getElementById('steps-container');

  const html = `
  <div class="phase-header" data-step-start="1">
    <h2 data-i18n="phase-1-title">🔵 Fase 1 — Crear Seed (BIP39)</h2>
    <p data-i18n="phase-1-desc">Pasos 1–7</p>
  </div>

  ${makeCard(1, 1, 's1-title', 's1-desc',
    `<div class="value-label" data-i18n="s1-val-label">Entropía (hex)</div>
     <span class="hl" id="v-entropy">${d.entropy}</span>
     <br><small style="color:var(--text2)" data-i18n="s1-note">128 bits • 16 bytes • solo modo Generar</small>`,
    '<span data-i18n="tech-1">CSPRNG genera 16 bytes aleatorios. Nunca reutilizar, nunca predecir. 2^128 combinaciones posibles.</span>'
  )}

  ${makeCard(2, 1, 's2-title', 's2-desc',
    `<div class="value-label" data-i18n="s2-val-label">SHA256(entropía) → primer byte</div>
     <span id="v-checksum" class="hl">${d.checksumBit || '0000'}</span>
     <span style="color:var(--text2)"> ← 4 bits de checksum</span>`,
    '<span data-i18n="tech-2">SHA256 produce 32 bytes. Se toman los primeros 4 bits (1 bit por cada 32 bits de entropía). Esto permite detectar errores al ingresar el mnemónico.</span>'
  )}

  ${makeCard(3, 1, 's3-title', 's3-desc',
    `<div class="value-label" data-i18n="s3-val-label">132 bits → 12 segmentos de 11 bits</div>
     <div id="v-bits" style="word-break:break-all;color:var(--text2);font-size:12px">${(d.bits132||'').split(' ').map(b=>`<span class="hl">${b}</span>`).join(' ')}</div>`,
    '<span data-i18n="tech-3">128 bits de entropía + 4 bits de checksum = 132 bits totales. 132 ÷ 11 = 12 segmentos exactos. Cada segmento es un número entre 0 y 2047.</span>'
  )}

  ${makeCard(4, 1, 's4-title', 's4-desc',
    `<div class="value-label" data-i18n="s4-val-label">Índices → Palabras BIP39</div>
     <div id="v-words">${(d.words||[]).map((w,i)=>`<span class="hl">${w}</span><span style="color:var(--text2);font-size:11px">(${d.indices?.[i]||0})</span>`).join(' ')}</div>`,
    '<span data-i18n="tech-4">El diccionario BIP39 tiene exactamente 2048 palabras (2^11). Las primeras 4 letras de cada palabra son únicas, lo que permite detectar errores de escritura.</span>'
  )}

  ${makeCard(5, 1, 's5-title', 's5-desc',
    `<div class="value-label" data-i18n="s5-val-label">Mnemónico (Recovery Code)</div>
     <div id="v-mnemonic" class="hl" style="font-size:15px;letter-spacing:0.5px">${d.mnemonic}</div>
     <small style="color:var(--text2);margin-top:6px;display:block" data-i18n="s5-note">Guardar en papel o metal. Nunca en digital.</small>`,
    '<span data-i18n="tech-5">El mnemónico es la representación humana de la entropía, no de la seed final. Con el mismo mnemónico + distinta passphrase → wallets completamente distintas.</span>'
  )}

  ${makeCard(6, 1, 's6-title', 's6-desc',
    `<div class="value-label" data-i18n="s6-val-label">NFKD(mnemónico) + NFKD(passphrase)</div>
     <div id="v-nfkd"><span class="hl">${d.mnemonicNFKD||d.mnemonic}</span><br><span style="color:var(--text2)">+ passphrase: "<span id="v-passphrase" class="hl">${d.passphrase||''}</span>"</span></div>`,
    '<span data-i18n="tech-6">NFKD es normalización Unicode. Garantiza que el mismo texto escrito de diferentes formas (ej. caracteres compuestos) produzca los mismos bytes. El prefijo "mnemonic" es ASCII fijo, no se normaliza.</span>'
  )}

  ${makeCard(7, 1, 's7-title', 's7-desc',
    `<div class="value-label">PBKDF2-HMAC-SHA512 → Root Seed</div>
     <div class="value-label" style="margin-top:8px">iterations: <span class="hl">2048</span> · dkLen: <span class="hl">64 bytes</span></div>
     <div id="v-rootseed" class="hl" style="font-size:11px;word-break:break-all">${d.rootSeed||'—'}</div>`,
    '<span data-i18n="tech-7">PBKDF2 itera el HMAC-SHA512 2048 veces. Este proceso lento es intencional — hace que los ataques de fuerza bruta sean 2048× más costosos. La seed de 64 bytes es la derivación criptográfica real.</span>'
  )}

  <div class="phase-header" data-step-start="8">
    <h2 data-i18n="phase-2-title">🟡 Fase 2 — Árbol HD (BIP32)</h2>
    <p data-i18n="phase-2-desc">Pasos 8–13</p>
  </div>

  ${makeCard(8, 2, 's8-title', 's8-desc',
    `<div class="value-label" data-i18n="s8-val-label">Root Seed (64 bytes)</div>
     <div id="v-rootseed-2" class="hl" style="font-size:11px;word-break:break-all">${d.rootSeed||'—'}</div>`,
    '<span data-i18n="tech-8">La root seed de 512 bits entra al proceso BIP32. Desde este punto en adelante trabajamos con el árbol HD. La seed original ya no aparece en el proceso de derivación.</span>'
  )}

  ${makeCard(9, 2, 's9-title', 's9-desc',
    `<div class="value-label">HMAC-SHA512(key="Bitcoin seed", data=rootSeed)</div>
     <div id="v-hmac" class="hl" style="font-size:11px;word-break:break-all">${d.hmacOutput||'—'}</div>
     <small style="color:var(--text2)">→ 64 bytes</small>`,
    '<span data-i18n="tech-9">La cadena literal "Bitcoin seed" es la clave HMAC — esto es específico de BIP32. Este HMAC actúa como función de derivación de clave (KDF) determinista.</span>'
  )}

  ${makeCard(10, 2, 's10-title', 's10-desc',
    `<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px">
       <div><div class="value-label">IL — Master Private Key</div><div id="v-il" class="hl" style="font-size:11px;word-break:break-all">${d.masterPrivKey||'—'}</div></div>
       <div><div class="value-label">IR — Chain Code</div><div id="v-ir" class="hl" style="font-size:11px;word-break:break-all">${d.masterChainCode||'—'}</div></div>
     </div>`,
    '<span data-i18n="tech-10">Los 64 bytes del HMAC se dividen exactamente a la mitad. IL (izquierda) se convierte en la clave privada maestra. IR (derecha) es el chain code — "entropía extra" que hace que la derivación sea única.</span>'
  )}

  ${makeCard(11, 2, 's11-title', 's11-desc',
    `<div><div class="value-label">Master Private Key (IL)</div><div id="v-mpk" class="hl" style="font-size:11px;word-break:break-all">${d.masterPrivKey||'—'}</div></div>
     <div style="margin-top:8px"><div class="value-label">Master Public Key (IL·G)</div><div id="v-mpubk" class="hl" style="font-size:11px;word-break:break-all">${d.masterPubKey||'—'}</div></div>
     <div style="margin-top:8px"><div class="value-label">Chain Code (IR)</div><div id="v-cc" class="hl" style="font-size:11px;word-break:break-all">${d.masterChainCode||'—'}</div></div>`,
    '<span data-i18n="tech-11">La clave pública se deriva de la privada multiplicando por el punto generador G de secp256k1. Es imposible revertir este proceso (problema del logaritmo discreto). El chain code es irreemplazable en su nodo.</span>'
  )}

  ${makeCard(12, 2, 's12-title', 's12-desc',
    `<div class="value-label">Master Node m = (privKey, pubKey, chainCode)</div>
     <div id="v-masternode" class="hl">m / depth:0 / index:0</div>`,
    '<span data-i18n="tech-12">El nodo maestro es la raíz del árbol HD. Cada nodo tiene: clave privada (32B), clave pública comprimida (33B), chain code (32B), profundidad (1B), fingerprint del padre (4B), índice (4B).</span>'
  )}

  ${makeCard(13, 2, 's13-title', 's13-desc',
    `<div><div class="value-label">xprv (clave privada extendida)</div><div id="v-xprv" class="hl" style="font-size:11px;word-break:break-all">${d.xprv||'—'}</div></div>
     <div style="margin-top:8px"><div class="value-label">xpub (clave pública extendida)</div><div id="v-xpub" class="hl" style="font-size:11px;word-break:break-all">${d.xpub||'—'}</div></div>`,
    '<span data-i18n="tech-13">Serialización de 78 bytes: version(4)+depth(1)+fingerprint(4)+index(4)+chainCode(32)+key(33). Para xprv: key = 0x00 || privKey. Para xpub: key = pubKey comprimida. Luego Base58Check → string portable.</span>'
  )}

  <div class="phase-header" data-step-start="14">
    <h2 data-i18n="phase-3-title">🟠 Fase 3 — Derivar Claves (BIP44/49/84/86)</h2>
    <p data-i18n="phase-3-desc">Pasos 14–19</p>
  </div>

  ${makeCard(14, 3, 's14-title', 's14-desc',
    `<div class="value-label">xprv del Master Node</div>
     <div id="v-xprv-2" class="hl" style="font-size:11px;word-break:break-all">${d.xprv||'—'}</div>`,
    '<span data-i18n="tech-14">La derivación comienza siempre desde el xprv del master node. Cada nivel del árbol produce un hijo determinista a partir del padre. Misma seed → mismo árbol siempre.</span>'
  )}

  ${makeCard(15, 3, 's15-title', 's15-desc',
    `<div class="value-label" data-i18n="s15-val-label">Ruta de Derivación (BIP84)</div>
     <div class="hl" style="font-size:18px;letter-spacing:1px">m / <span class="hl">84'</span> / <span class="hl">0'</span> / <span class="hl">0'</span> / <span class="hl">0</span> / <span class="hl">0</span></div>
     <small style="color:var(--text2);display:block;margin-top:6px">purpose' / coin_type' / account' / change / index</small>`,
    '<span data-i18n="tech-15">84\' = BIP84 (Native SegWit). 0\' = Bitcoin mainnet. 0\' = primera cuenta. 0 = dirección de recepción (1 = cambio interno). 0 = primer índice. Los apóstrofes indican derivación hardened (usa clave privada).</span>'
  )}

  ${makeCard(16, 3, 's16-title', 's16-desc',
    `<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px">
       <div style="border:1px solid var(--border);border-radius:6px;padding:10px">
         <div class="value-label">Normal (sin ')</div>
         <div style="font-size:12px;color:var(--text2)">Usa clave pública del padre.<br>xpub puede derivar hijos.<br>Útil para wallets watch-only.</div>
       </div>
       <div style="border:1px solid var(--naranja);border-radius:6px;padding:10px">
         <div class="value-label" style="color:var(--naranja)">Hardened (' o H)</div>
         <div style="font-size:12px;color:var(--text2)">Usa clave privada del padre.<br>xpub NO puede derivar hijos.<br>Más seguro para cuentas.</div>
       </div>
     </div>`,
    '<span data-i18n="tech-16">Normal: HMAC-SHA512(key=chainCode, data=pubKey||index). Hardened: HMAC-SHA512(key=chainCode, data=0x00||privKey||index). El índice hardened usa index + 2^31 (bit 31 activado).</span>'
  )}

  ${makeCard(17, 3, 's17-title', 's17-desc',
    `<div style="display:grid;grid-template-columns:repeat(2,1fr);gap:8px">
       ${[['BIP44','m/44\'','1xxx...','Legacy P2PKH'],['BIP49','m/49\'','3xxx...','P2SH-P2WPKH'],['BIP84','m/84\'','bc1q...','Native SegWit ✓'],['BIP86','m/86\'','bc1p...','Taproot P2TR']].map(([b,p,a,d2])=>`
       <div style="border:1px solid ${b==='BIP84'?'var(--naranja)':'var(--border)'};border-radius:6px;padding:10px">
         <div class="hl" style="font-size:13px">${b}</div>
         <div style="font-size:11px;color:var(--text2)">${p}</div>
         <div style="font-size:13px;font-family:monospace">${a}</div>
         <div style="font-size:11px;color:var(--text2)">${d2}</div>
       </div>`).join('')}
     </div>`,
    '<span data-i18n="tech-17">BIP84 produce las direcciones más usadas hoy (Native SegWit, menores fees). BIP86/Taproot es el futuro (bc1p). BIP44 legacy todavía activo para compatibilidad. Misma seed → 4 wallets distintas.</span>'
  )}

  ${makeCard(18, 3, 's18-title', 's18-desc',
    `<div class="value-label" data-i18n="s18-val-label">Child Key Derivation — nivel por nivel</div>
     <div style="font-size:12px;color:var(--text2)">m → m/84' → m/84'/0' → m/84'/0'/0' → m/84'/0'/0'/0 → m/84'/0'/0'/0/0</div>
     <div style="margin-top:8px"><div class="value-label">Child Private Key</div><div id="v-childpriv" class="hl" style="font-size:11px;word-break:break-all">${d.childPrivKey||'—'}</div></div>`,
    '<span data-i18n="tech-18">Cada nivel aplica CKD: IL_child = HMAC izquierda + parent_privkey (mod n). chain_code_child = HMAC derecha. El proceso se repite 5 veces para llegar al nodo hoja m/84\'/0\'/0\'/0/0.</span>'
  )}

  ${makeCard(19, 3, 's19-title', 's19-desc',
    `<div class="value-label" data-i18n="s19-val-label">Clave Pública Hija (comprimida, 33 bytes)</div>
     <div id="v-childpub" class="hl" style="font-size:12px;word-break:break-all">${d.childPubKey||'—'}</div>
     <small style="color:var(--text2)">prefijo <span class="hl">02</span> o <span class="hl">03</span> + coordenada X (32 bytes)</small>`,
    '<span data-i18n="tech-19">child_pubkey = child_privkey · G en secp256k1. La compresión usa el prefijo 02 (Y par) o 03 (Y impar) + solo la coordenada X. Ahorra 32 bytes respecto a la forma no comprimida.</span>'
  )}

  <div class="phase-header" data-step-start="20">
    <h2 data-i18n="phase-4-title">⚪ Fase 4 — Construir Dirección</h2>
    <p data-i18n="phase-4-desc">Pasos 20–25</p>
  </div>

  ${makeCard(20, 4, 's20-title', 's20-desc',
    `<div class="value-label" data-i18n="s20-val-label">Pubkey hija (entrada)</div>
     <div id="v-pub20" class="hl" style="font-size:12px;word-break:break-all">${d.childPubKey||'—'}</div>`,
    '<span data-i18n="tech-20">La clave pública del nodo hoja es el punto de partida para construir la dirección. Es el único dato que se comparte públicamente de todo el árbol.</span>'
  )}

  ${makeCard(21, 4, 's21-title', 's21-desc',
    `<div class="value-label" data-i18n="s21-val-label">Pubkey comprimida (33 bytes)</div>
     <div id="v-pub21" class="hl" style="font-size:12px;word-break:break-all">${d.childPubKey||'—'}</div>
     <div style="margin-top:6px;font-size:12px">
       <span class="hl">${(d.childPubKey||'02').slice(0,2)}</span><span style="color:var(--text2)"> prefijo · </span>
       <span class="hl">${(d.childPubKey||'').slice(2,66)||'X...'}</span><span style="color:var(--text2)"> coordenada X</span>
     </div>`,
    '<span data-i18n="tech-21">La curva secp256k1 es simétrica respecto al eje X. El prefijo 02/03 indica cuál de los dos puntos Y posibles es el correcto. El receptor puede reconstruir Y completo si lo necesita.</span>'
  )}

  ${makeCard(22, 4, 's22-title', 's22-desc',
    `<div class="value-label">SHA256(pubkey) → RIPEMD160 = HASH160</div>
     <div id="v-hash160" class="hl" style="font-size:12px;word-break:break-all">${d.hash160||'—'}</div>
     <small style="color:var(--text2)">→ 20 bytes (160 bits)</small>`,
    '<span data-i18n="tech-22">SHA256 produce 32 bytes. RIPEMD160 reduce a 20 bytes. Este doble hash fue elegido para resistir ataques a SHA256 y a RIPEMD160 por separado. También protege contra vulnerabilidades de criptografía cuántica.</span>'
  )}

  ${makeCard(23, 4, 's23-title', 's23-desc',
    `<div class="value-label">scriptPubKey (P2WPKH — BIP84)</div>
     <div class="hl">OP_0 <span id="v-script-hash" style="font-size:12px">${d.hash160||'<hash160>'}</span></div>
     <small style="color:var(--text2)">Script que define las condiciones para gastar los fondos</small>`,
    '<span data-i18n="tech-23">OP_0 indica versión 0 de witness (SegWit v0). Para P2WPKH el script es exactamente: OP_0 + PUSH20 + hash160. Para gastar, se necesita firma + pubkey en el campo witness de la transacción.</span>'
  )}

  ${makeCard(24, 4, 's24-title', 's24-desc',
    `<div style="display:grid;grid-template-columns:repeat(3,1fr);gap:8px">
       ${[['Bech32','bc1q...','BIP84 SegWit ✓'],['Bech32m','bc1p...','BIP86 Taproot'],['Base58Check','1... / 3...','BIP44/49 Legacy']].map(([e,a,d2])=>`
       <div style="border:1px solid ${e==='Bech32'?'var(--blanco)':'var(--border)'};border-radius:6px;padding:8px;text-align:center">
         <div class="hl" style="font-size:12px">${e}</div>
         <div style="font-family:monospace;font-size:12px">${a}</div>
         <div style="font-size:11px;color:var(--text2)">${d2}</div>
       </div>`).join('')}
     </div>`,
    '<span data-i18n="tech-24">Bech32 detecta errores de tipeo mejor que Base58. Bech32m corrige un bug de Bech32 para versiones de witness mayores a 0 (Taproot usa witness v1). Base58Check usa checksum de 4 bytes al final.</span>'
  )}

  ${makeCard(25, 4, 's25-title', 's25-desc',
    `<div class="address-result">
       <div class="addr-label" data-i18n="s25-val-label">Dirección Bitcoin — m/84'/0'/0'/0/0</div>
       <div id="v-address" class="addr">${d.address||'bc1q...'}</div>
     </div>`,
    '<span data-i18n="tech-25">La dirección es segura de compartir. No revela la clave privada, el mnemónico, ni el camino de derivación. Cada vez que necesites recibir fondos en una cuenta nueva, incrementá el índice (0→1→2...).</span>'
  )}
  `.trim();

  container.innerHTML = html;
  applyLang();
}
```

- [ ] **Step 3: Call renderSteps on load**

In the main `<script>`, add at the end:
```javascript
document.addEventListener('DOMContentLoaded', () => {
  renderSteps();
});
```

- [ ] **Step 4: Open in browser, verify all 25 cards render**

Expected: All 25 numbered step cards visible, color-coded by phase (celeste/amarillo/naranja/blanco borders), value boxes showing EXAMPLE data or `—` for null fields, `+ técnico` buttons visible.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: all 25 step cards rendered with EXAMPLE data"
git push
```

---

## Task 6: Live computation engine (computeAll)

**Files:**
- Modify: `index.html`

This task implements the `computeAll()` function that calculates all 25 step values from a given entropy, and also populates the null fields in `EXAMPLE` on page load.

- [ ] **Step 1: Add computeAll function**

Add a new `<script>` block after the encoders:

```javascript
// Utility: bytes ↔ hex
function toHex(bytes) {
  return Array.from(bytes).map(b => b.toString(16).padStart(2,'0')).join('');
}
function fromHex(hex) {
  const bytes = new Uint8Array(hex.length / 2);
  for (let i = 0; i < bytes.length; i++) bytes[i] = parseInt(hex.slice(i*2, i*2+2), 16);
  return bytes;
}

// BIP32 serialization
function serializeXkey(version, depth, parentFp, childIndex, chainCode, key33) {
  const buf = new Uint8Array(78);
  const dv = new DataView(buf.buffer);
  dv.setUint32(0, version, false);
  buf[4] = depth;
  dv.setUint32(5, parentFp, false);
  dv.setUint32(9, childIndex, false);
  buf.set(chainCode, 13);
  buf.set(key33, 45);
  return buf;
}

function fingerprint(pubkey33) {
  const h = nobleHashes.ripemd160(nobleHashes.sha256(pubkey33));
  return (h[0]<<24)|(h[1]<<16)|(h[2]<<8)|h[3];
}

// CKD (Child Key Derivation) — one level
// hmacFn and sha512Fn are passed in to avoid global name dependency on the vendored bundle
function ckdPriv(parentPriv, parentPub, parentChain, index, hmacFn, sha512Fn) {
  const indexBytes = new Uint8Array(4);
  new DataView(indexBytes.buffer).setUint32(0, index, false);
  let data;
  if (index >= 0x80000000) {
    // hardened — 0x00 || privkey || index
    data = new Uint8Array(37);
    data[0] = 0x00;
    data.set(parentPriv, 1);
    data.set(indexBytes, 33);
  } else {
    // normal — compressedPubkey (33 bytes) || index
    data = new Uint8Array(37);
    data.set(parentPub, 0);
    data.set(indexBytes, 33);
  }
  const I = hmacFn(sha512Fn, parentChain, data);
  const IL = I.slice(0, 32);
  const IR = I.slice(32);
  // child privkey = (IL + parent_privkey) mod n
  const n = BigInt('0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141');
  const childPrivBig = (BigInt('0x' + toHex(IL)) + BigInt('0x' + toHex(parentPriv))) % n;
  const childPriv = fromHex(childPrivBig.toString(16).padStart(64, '0'));
  return { priv: childPriv, chain: IR };
}

async function computeAll(entropyHex, passphrase = '') {
  const result = {};
  const { sha256, ripemd160, hmac, sha512, pbkdf2 } = nobleHashes;

  // Phase 1
  result.entropy = entropyHex;
  const entropyBytes = fromHex(entropyHex);

  // Checksum: first 4 bits of SHA256(entropy)
  const hash = sha256(entropyBytes);
  const checksumByte = hash[0];
  const checksumBits = (checksumByte >> 4).toString(2).padStart(4, '0');
  result.checksumBit = checksumBits;

  // 132 bits as binary string
  let allBits = '';
  for (const b of entropyBytes) allBits += b.toString(2).padStart(8, '0');
  allBits += checksumBits;
  result.bits132 = Array.from({length:12}, (_,i) => allBits.slice(i*11, i*11+11)).join(' ');

  // Indices and words
  const indices = Array.from({length:12}, (_,i) => parseInt(allBits.slice(i*11, i*11+11), 2));
  result.indices = indices;
  const words = indices.map(i => BIP39_WORDLIST[i]);
  result.words = words;
  result.mnemonic = words.join(' ');

  // NFKD normalization
  const mnemonicNFKD = result.mnemonic.normalize('NFKD');
  const passphraseNFKD = passphrase.normalize('NFKD');
  result.mnemonicNFKD = mnemonicNFKD;
  result.passphrase = passphrase;

  // PBKDF2 → root seed
  const mnemonicBytes = new TextEncoder().encode(mnemonicNFKD);
  const saltBytes = new TextEncoder().encode('mnemonic' + passphraseNFKD);
  const rootSeedBytes = await pbkdf2(sha512, mnemonicBytes, saltBytes, { c: 2048, dkLen: 64 });
  result.rootSeed = toHex(rootSeedBytes);

  // Phase 2
  const hmacKey = new TextEncoder().encode('Bitcoin seed');
  const I = hmac(sha512, hmacKey, rootSeedBytes);
  result.hmacOutput = toHex(I);

  const IL = I.slice(0, 32);
  const IR = I.slice(32);
  result.masterPrivKey = toHex(IL);
  result.masterChainCode = toHex(IR);

  // Master public key
  const masterPubBytes = secp.getPublicKey(IL, true);
  result.masterPubKey = toHex(masterPubBytes);

  // xprv / xpub (depth=0, parent fingerprint=0, index=0)
  const xprvPayload = serializeXkey(0x0488ADE4, 0, 0, 0, IR, new Uint8Array([0x00, ...IL]));
  result.xprv = base58CheckEncode(xprvPayload);
  const xpubPayload = serializeXkey(0x0488B21E, 0, 0, 0, IR, masterPubBytes);
  result.xpub = base58CheckEncode(xpubPayload);

  // Phase 3 — derive m/84'/0'/0'/0/0
  const path = [0x80000054, 0x80000000, 0x80000000, 0, 0]; // 84'+0'+0'+0+0
  let privKey = IL;
  let chainCode = IR;
  let pubKey = masterPubBytes;

  for (const index of path) {
    const child = ckdPriv(privKey, pubKey, chainCode, index, hmac, sha512);
    privKey = child.priv;
    chainCode = child.chain;
    pubKey = secp.getPublicKey(privKey, true);
  }

  result.childPrivKey = toHex(privKey);
  result.childPubKey = toHex(pubKey);

  // Phase 4
  const hash160Bytes = ripemd160(sha256(pubKey));
  result.hash160 = toHex(hash160Bytes);
  result.address = p2wpkhAddress(hash160Bytes);

  return result;
}
```

- [ ] **Step 2: Populate EXAMPLE on load and wire live mode**

Update the `DOMContentLoaded` handler:

```javascript
document.addEventListener('DOMContentLoaded', async () => {
  // Compute and populate EXAMPLE null fields
  const computed = await computeAll('00000000000000000000000000000000', '');
  Object.assign(EXAMPLE, computed);

  // Initialize LIVE as a copy
  window.LIVE = { ...EXAMPLE };

  renderSteps();
});
```

Update `onPassphraseChange`:
```javascript
async function onPassphraseChange() {
  if (currentMode === 'b' && window.LIVE) {
    const passphrase = document.getElementById('passphrase-input').value;
    window.LIVE = await computeAll(window.LIVE.entropy, passphrase);
    renderSteps();
  }
}
```

- [ ] **Step 3: Add "Generate entropy" button to step 1 in Mode B**

In `renderSteps()`, after the step 1 card creation, add a button that's only visible in Mode B. Modify the step 1 valueHtml to include:

```javascript
const step1Extra = currentMode === 'b'
  ? `<button onclick="generateEntropy()" style="margin-top:8px;padding:6px 16px;background:var(--celeste);color:#000;border:none;border-radius:6px;font-weight:700;cursor:pointer">🎲 Nueva entropía</button>`
  : '';
```

Add `generateEntropy()` function:
```javascript
async function generateEntropy() {
  const bytes = new Uint8Array(16);
  window.crypto.getRandomValues(bytes);
  const entropyHex = toHex(bytes);
  const passphrase = document.getElementById('passphrase-input').value;
  window.LIVE = await computeAll(entropyHex, passphrase);
  renderSteps();
}
```

- [ ] **Step 4: Console verification — check against BIP39 test vector**

Open browser console and run:
```javascript
computeAll('00000000000000000000000000000000', '').then(r => {
  console.log('mnemonic:', r.mnemonic);
  // Expected: 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about'

  console.log('rootSeed starts with:', r.rootSeed.slice(0, 20));
  // Expected: '5eb00bbddcf069084889'

  console.log('address starts with:', r.address.slice(0, 4));
  // Expected: 'bc1q'

  console.log('xprv starts with:', r.xprv.slice(0, 4));
  // Expected: 'xprv'
});
```

If mnemonic is correct but rootSeed differs: verify PBKDF2 salt is `'mnemonic'` (not NFKD normalized) + NFKD(passphrase).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: computeAll engine — entropy to address, BIP39/32/84, live mode"
git push
```

---

## Task 7: Mode B wiring — generate button + security banner

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Show generate button only in Mode B**

In `renderSteps()`, the step 1 card already includes the conditional button from Task 6. Verify it shows in Mode B and hides in Mode A by clicking the toggle buttons.

- [ ] **Step 2: Security banner text (inline for now)**

In the existing `<span data-i18n="security-text">` in the HTML, add the full ES text:

```html
<span data-i18n="security-text">⚠️ Solo educativo. No uses esta herramienta para guardar fondos reales. Las claves generadas son visibles en la memoria del browser y en DevTools. Si copiás palabras o claves, el portapapeles puede ser leído por otras aplicaciones. Nada se envía a ningún servidor — pero extensiones del browser pueden leer el contenido de la página. Para máxima seguridad, usá un dispositivo air-gapped sin extensiones.</span>
```

- [ ] **Step 3: Open in browser, test Mode B flow**

1. Click `⚡ Generar`
2. Verify: security banner appears (orange-bordered box)
3. Click `🎲 Nueva entropía` (visible in Step 1)
4. Verify: all step values update (new entropy hex, new mnemonic words, new address)
5. Click `📖 Ejemplo`
6. Verify: security banner hides, test vector values restore

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: Mode B generate button, security banner, passphrase recalc"
git push
```

---

## Task 8: i18n (ES/EN translations)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add TRANSLATIONS object and applyLang()**

Replace the empty `applyLang()` stub with:

```javascript
const TRANSLATIONS = {
  es: {
    'mode-a': '📖 Ejemplo',
    'mode-b': '⚡ Generar',
    'phase-1': '🔵 Seed BIP39',
    'phase-2': '🟡 Árbol HD',
    'phase-3': '🟠 Derivación',
    'phase-4': '⚪ Dirección',
    'phase-1-title': '🔵 Fase 1 — Crear Seed (BIP39)',
    'phase-1-desc': 'Pasos 1–7: de bits aleatorios a 12 palabras + root seed',
    'phase-2-title': '🟡 Fase 2 — Árbol HD (BIP32)',
    'phase-2-desc': 'Pasos 8–13: de root seed a master node + xprv/xpub',
    'phase-3-title': '🟠 Fase 3 — Derivar Claves',
    'phase-3-desc': 'Pasos 14–19: de xprv a clave pública hija vía m/84\'/0\'/0\'/0/0',
    'phase-4-title': '⚪ Fase 4 — Construir Dirección',
    'phase-4-desc': 'Pasos 20–25: de pubkey a dirección Bitcoin bc1q...',
    'passphrase-label': 'Passphrase opcional (BIP39)',
    'passphrase-placeholder': 'Dejar vacío para el estándar',
    'tech-btn': '+ técnico',
    'security-text': '⚠️ Solo educativo. No uses esta herramienta para guardar fondos reales. Las claves generadas son visibles en la memoria del browser y en DevTools. Si copiás palabras o claves, el portapapeles puede ser leído por otras aplicaciones. Nada se envía a ningún servidor — pero extensiones del browser pueden leer el contenido de la página. Para máxima seguridad, usá un dispositivo air-gapped sin extensiones.',
    // Step titles
    's1-title': 'Entropía Aleatoria', 's2-title': 'Checksum SHA256',
    's3-title': 'Segmentos de 11 Bits', 's4-title': 'Diccionario BIP39',
    's5-title': 'Mnemónico', 's6-title': 'Passphrase + NFKD',
    's7-title': 'PBKDF2 → Root Seed', 's8-title': 'Root Seed Como Entrada',
    's9-title': 'HMAC-SHA512', 's10-title': 'División IL + IR',
    's11-title': '3 Claves Maestras', 's12-title': 'Master Node (m)',
    's13-title': 'xprv / xpub', 's14-title': 'xprv del Master Node',
    's15-title': 'Ruta de Derivación', 's16-title': 'Normal vs Hardened',
    's17-title': '4 Estándares de Derivación', 's18-title': 'Derivación Por Nivel',
    's19-title': 'Clave Pública Hija', 's20-title': 'Pubkey Como Entrada',
    's21-title': 'Pubkey Comprimida (33 bytes)', 's22-title': 'SHA256 → RIPEMD160',
    's23-title': 'scriptPubKey', 's24-title': 'Encoding',
    's25-title': 'Dirección Bitcoin — Resultado Final',
    // Step descriptions (short, 1-2 sentences)
    's1-desc': 'Un generador criptográficamente seguro produce 128 bits aleatorios. Estos bits son la semilla de todo el proceso.',
    's2-desc': 'Se aplica SHA256 a la entropía. Los primeros 4 bits del resultado forman el checksum que permite detectar errores.',
    's3-desc': '128 bits de entropía + 4 bits de checksum = 132 bits totales. Se dividen en 12 segmentos de 11 bits cada uno.',
    's4-desc': 'Cada segmento (0–2047) apunta a una palabra del diccionario oficial BIP39 de 2048 palabras.',
    's5-desc': 'Las 12 palabras forman el mnemónico, también llamado seed phrase o recovery phrase. Guardarlo en papel o metal.',
    's6-desc': 'El mnemónico y la passphrase se normalizan con NFKD para garantizar consistencia entre dispositivos y teclados.',
    's7-desc': 'PBKDF2-HMAC-SHA512 itera 2048 veces para derivar los 64 bytes (512 bits) del root seed. Este proceso lento es intencional.',
    's8-desc': 'Los 64 bytes del root seed entran al proceso BIP32 para construir el árbol HD.',
    's9-desc': 'HMAC-SHA512 con la clave literal "Bitcoin seed" transforma el root seed en 64 bytes de material de clave.',
    's10-desc': 'Los 64 bytes se dividen: los primeros 32 son IL (clave privada maestra), los últimos 32 son IR (chain code).',
    's11-desc': 'De IL se deriva la clave privada maestra. De ella, la clave pública comprimida (IL·G). IR es el chain code.',
    's12-desc': 'El Master Node (m) combina clave privada, pública y chain code. Es la raíz del árbol HD determinista.',
    's13-desc': 'xprv y xpub son las claves extendidas serializadas en 78 bytes y codificadas en Base58Check para portabilidad.',
    's14-desc': 'La derivación parte siempre del xprv del master node. Con la misma seed se obtiene siempre el mismo árbol.',
    's15-desc': 'La ruta m/84\'/0\'/0\'/0/0 selecciona: BIP84, Bitcoin mainnet, primera cuenta, recepción, primer índice.',
    's16-desc': 'La derivación hardened (apóstrofe) usa la clave privada del padre — más segura. La normal usa la pública.',
    's17-desc': 'Cuatro estándares producen distintos tipos de dirección desde la misma seed. BIP84 es el más usado hoy.',
    's18-desc': 'Se aplica CKD (Child Key Derivation) 5 veces para recorrer los 5 niveles de la ruta hasta el nodo hoja.',
    's19-desc': 'La clave pública hija se obtiene multiplicando la clave privada hija por el punto G de la curva secp256k1.',
    's20-desc': 'La clave pública comprimida del nodo hoja es el único input para construir la dirección Bitcoin.',
    's21-desc': 'La pubkey comprimida usa 33 bytes: prefijo 02/03 + coordenada X. El prefijo indica la paridad de Y.',
    's22-desc': 'SHA256 seguido de RIPEMD160 produce HASH160: una huella digital de 20 bytes de la clave pública.',
    's23-desc': 'El scriptPubKey define las condiciones para gastar los fondos. Para P2WPKH es: OP_0 + HASH160.',
    's24-desc': 'El scriptPubKey se codifica en formato legible. BIP84 usa Bech32 (bc1q), BIP86 usa Bech32m (bc1p).',
    's25-desc': 'La dirección Bitcoin es el resultado final. Es segura de compartir — no revela ningún dato sensible.',
    // Value labels
    's1-val-label': 'Entropía (hex)',
    's1-note': '128 bits · 16 bytes · solo modo Generar',
    's2-val-label': 'SHA256(entropía) → checksum (4 bits)',
    's3-val-label': '132 bits → 12 segmentos de 11 bits',
    's4-val-label': 'Índices → Palabras BIP39',
    's5-val-label': 'Mnemónico (Recovery Code)',
    's5-note': 'Guardar en papel o metal. Nunca en digital.',
    's6-val-label': 'NFKD(mnemónico) + NFKD(passphrase)',
    's8-val-label': 'Root Seed (64 bytes)',
    's15-val-label': 'Ruta de Derivación (BIP84)',
    's18-val-label': 'Child Key Derivation — nivel por nivel',
    's19-val-label': 'Clave Pública Hija (comprimida, 33 bytes)',
    's20-val-label': 'Pubkey hija (entrada)',
    's21-val-label': 'Pubkey comprimida (33 bytes)',
    's25-val-label': "Dirección Bitcoin — m/84'/0'/0'/0/0",
    // Errors
    'err-webcrypto': '⚠️ WebCrypto no disponible. Usá un browser moderno (Chrome, Firefox, Safari).',
    'err-compute': '⚠️ Error en el cálculo. Verificá la consola para más detalles.',
    'err-passphrase': '⚠️ La passphrase contiene caracteres que no pueden normalizarse correctamente.',
  },
  en: {
    'mode-a': '📖 Example',
    'mode-b': '⚡ Generate',
    'phase-1': '🔵 BIP39 Seed',
    'phase-2': '🟡 HD Tree',
    'phase-3': '🟠 Derivation',
    'phase-4': '⚪ Address',
    'phase-1-title': '🔵 Phase 1 — Create Seed (BIP39)',
    'phase-1-desc': 'Steps 1–7: from random bits to 12 words + root seed',
    'phase-2-title': '🟡 Phase 2 — HD Tree (BIP32)',
    'phase-2-desc': 'Steps 8–13: from root seed to master node + xprv/xpub',
    'phase-3-title': '🟠 Phase 3 — Derive Keys',
    'phase-3-desc': "Steps 14–19: from xprv to child public key via m/84'/0'/0'/0/0",
    'phase-4-title': '⚪ Phase 4 — Build Address',
    'phase-4-desc': 'Steps 20–25: from pubkey to Bitcoin address bc1q...',
    'passphrase-label': 'Optional passphrase (BIP39)',
    'passphrase-placeholder': 'Leave empty for standard',
    'tech-btn': '+ technical',
    'security-text': '⚠️ Educational only. Do not use this tool to store real funds. Generated keys are visible in browser memory and DevTools. If you copy words or keys, your clipboard can be read by other applications. Nothing is sent to any server — but browser extensions can read page content. For maximum security, use an air-gapped device without extensions.',
    's1-title': 'Random Entropy', 's2-title': 'SHA256 Checksum',
    's3-title': '11-Bit Segments', 's4-title': 'BIP39 Dictionary',
    's5-title': 'Mnemonic', 's6-title': 'Passphrase + NFKD',
    's7-title': 'PBKDF2 → Root Seed', 's8-title': 'Root Seed Input',
    's9-title': 'HMAC-SHA512', 's10-title': 'IL + IR Split',
    's11-title': '3 Master Keys', 's12-title': 'Master Node (m)',
    's13-title': 'xprv / xpub', 's14-title': 'xprv from Master Node',
    's15-title': 'Derivation Path', 's16-title': 'Normal vs Hardened',
    's17-title': '4 Derivation Standards', 's18-title': 'Per-Level Derivation',
    's19-title': 'Child Public Key', 's20-title': 'Pubkey as Input',
    's21-title': 'Compressed Pubkey (33 bytes)', 's22-title': 'SHA256 → RIPEMD160',
    's23-title': 'scriptPubKey', 's24-title': 'Encoding',
    's25-title': 'Bitcoin Address — Final Result',
    's1-desc': 'A cryptographically secure generator produces 128 random bits. These bits are the seed of the entire process.',
    's2-desc': 'SHA256 is applied to the entropy. The first 4 bits of the result form a checksum to detect transcription errors.',
    's3-desc': '128 entropy bits + 4 checksum bits = 132 total bits, split into 12 segments of 11 bits each.',
    's4-desc': 'Each segment (0–2047) maps to a word in the official BIP39 dictionary of 2048 words.',
    's5-desc': 'The 12 words form the mnemonic, also called seed phrase or recovery phrase. Write it on paper or metal.',
    's6-desc': 'The mnemonic and passphrase are NFKD-normalized to ensure consistency across devices and keyboards.',
    's7-desc': 'PBKDF2-HMAC-SHA512 iterates 2048 times to derive 64 bytes (512 bits) of root seed. The slowness is intentional.',
    's8-desc': 'The 64-byte root seed enters the BIP32 process to build the HD tree.',
    's9-desc': 'HMAC-SHA512 with the literal key "Bitcoin seed" transforms the root seed into 64 bytes of key material.',
    's10-desc': 'The 64 bytes are split: the first 32 are IL (master private key), the last 32 are IR (chain code).',
    's11-desc': 'IL becomes the master private key. From it, the compressed public key (IL·G). IR is the chain code.',
    's12-desc': 'The Master Node (m) combines private key, public key, and chain code. It\'s the root of the deterministic HD tree.',
    's13-desc': 'xprv and xpub are the extended keys, serialized in 78 bytes and Base58Check-encoded for portability.',
    's14-desc': 'Derivation always starts from the master node xprv. The same seed always produces the same tree.',
    's15-desc': "Path m/84'/0'/0'/0/0 selects: BIP84, Bitcoin mainnet, first account, receive, first index.",
    's16-desc': 'Hardened derivation (apostrophe) uses the parent private key — more secure. Normal uses the public key.',
    's17-desc': 'Four standards produce different address types from the same seed. BIP84 is the most widely used today.',
    's18-desc': 'CKD (Child Key Derivation) is applied 5 times to traverse the 5 path levels to the leaf node.',
    's19-desc': 'The child public key is obtained by multiplying the child private key by point G on the secp256k1 curve.',
    's20-desc': 'The compressed public key of the leaf node is the only input for building the Bitcoin address.',
    's21-desc': 'The compressed pubkey uses 33 bytes: prefix 02/03 + X coordinate. The prefix indicates Y parity.',
    's22-desc': 'SHA256 followed by RIPEMD160 produces HASH160: a 20-byte fingerprint of the public key.',
    's23-desc': 'The scriptPubKey defines the spending conditions. For P2WPKH it is: OP_0 + HASH160.',
    's24-desc': 'The scriptPubKey is encoded in human-readable format. BIP84 uses Bech32 (bc1q), BIP86 uses Bech32m (bc1p).',
    's25-desc': 'The Bitcoin address is the final result. Safe to share — it reveals no sensitive information.',
    's1-val-label': 'Entropy (hex)',
    's1-note': '128 bits · 16 bytes · Generate mode only',
    's2-val-label': 'SHA256(entropy) → checksum (4 bits)',
    's3-val-label': '132 bits → 12 segments of 11 bits',
    's4-val-label': 'Indices → BIP39 Words',
    's5-val-label': 'Mnemonic (Recovery Code)',
    's5-note': 'Write on paper or metal. Never digital.',
    's6-val-label': 'NFKD(mnemonic) + NFKD(passphrase)',
    's8-val-label': 'Root Seed (64 bytes)',
    's15-val-label': 'Derivation Path (BIP84)',
    's18-val-label': 'Child Key Derivation — level by level',
    's19-val-label': 'Child Public Key (compressed, 33 bytes)',
    's20-val-label': 'Child pubkey (input)',
    's21-val-label': 'Compressed pubkey (33 bytes)',
    's25-val-label': "Bitcoin Address — m/84'/0'/0'/0/0",
    'err-webcrypto': '⚠️ WebCrypto not available. Use a modern browser (Chrome, Firefox, Safari).',
    'err-compute': '⚠️ Computation error. Check the console for details.',
    'err-passphrase': '⚠️ The passphrase contains characters that cannot be correctly normalized.',
  }
};

function applyLang() {
  document.querySelectorAll('[data-i18n]').forEach(el => {
    const key = el.dataset.i18n;
    if (TRANSLATIONS[currentLang]?.[key]) el.textContent = TRANSLATIONS[currentLang][key];
  });
  document.querySelectorAll('[data-i18n-placeholder]').forEach(el => {
    const key = el.dataset.i18nPlaceholder;
    if (TRANSLATIONS[currentLang]?.[key]) el.placeholder = TRANSLATIONS[currentLang][key];
  });
  // Update tech-toggle buttons text
  document.querySelectorAll('.tech-toggle').forEach(btn => {
    if (!btn.nextElementSibling?.classList.contains('open')) {
      btn.textContent = currentLang === 'es' ? '+ técnico' : '+ technical';
    }
  });
}
```

- [ ] **Step 2: Open in browser, click EN toggle**

Expected: All labels, titles, descriptions, button text switch to English. Technical terms (SHA256, PBKDF2, xprv, etc.) remain unchanged. Mnemonic words stay in English.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: ES/EN i18n — full translations for all 25 steps + errors"
git push
```

---

## Task 9: Scroll phase progress bar

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add scroll observer**

Add to the main `<script>` (after `renderSteps` is called):

```javascript
function initScrollObserver() {
  const phases = [
    { tab: 'ptab-1', steps: [1,2,3,4,5,6,7] },
    { tab: 'ptab-2', steps: [8,9,10,11,12,13] },
    { tab: 'ptab-3', steps: [14,15,16,17,18,19] },
    { tab: 'ptab-4', steps: [20,21,22,23,24,25] },
  ];

  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const stepNum = parseInt(entry.target.dataset.step);
        const phase = phases.find(p => p.steps.includes(stepNum));
        if (phase) {
          phases.forEach(p => document.getElementById(p.tab)?.classList.remove('active'));
          document.getElementById(phase.tab)?.classList.add('active');
        }
      }
    });
  }, { threshold: 0.3, rootMargin: '-80px 0px 0px 0px' });

  document.querySelectorAll('.step-card').forEach(card => observer.observe(card));
}
```

- [ ] **Step 2: Call initScrollObserver after renderSteps**

Update `DOMContentLoaded`:
```javascript
document.addEventListener('DOMContentLoaded', async () => {
  const computed = await computeAll('00000000000000000000000000000000', '');
  Object.assign(EXAMPLE, computed);
  window.LIVE = { ...EXAMPLE };
  renderSteps();
  initScrollObserver();
});
```

Also call `initScrollObserver()` at the end of `renderSteps()` (since cards are re-rendered on mode change).

- [ ] **Step 3: Test scroll behavior**

Scroll through all 25 steps. Expected: phase tabs highlight in the header as you pass into each phase (Seed → HD → Derivación → Dirección).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: scroll phase progress bar with IntersectionObserver"
git push
```

---

## Task 10: Final polish + mobile + push

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Mobile responsive tweaks**

Add to the CSS:
```css
@media (max-width: 600px) {
  .header { padding: 8px 12px; }
  .header-links { display: none; }
  .main { padding: 16px 12px 60px; }
  .phase-bar { overflow-x: auto; }
  .phase-tab { padding: 8px 10px; font-size: 11px; }
  .step-card { padding: 14px; }
  .value-box { font-size: 11px; }
}
```

- [ ] **Step 2: WebCrypto availability check**

At the start of `DOMContentLoaded`, add:
```javascript
if (!window.crypto?.subtle && !window.crypto?.getRandomValues) {
  document.getElementById('steps-container').innerHTML =
    `<div style="color:var(--red);padding:20px" data-i18n="err-webcrypto">${TRANSLATIONS.es['err-webcrypto']}</div>`;
  return;
}
```

- [ ] **Step 3: Error handling in computeAll**

Wrap the `DOMContentLoaded` async call in try/catch:
```javascript
try {
  const computed = await computeAll('00000000000000000000000000000000', '');
  Object.assign(EXAMPLE, computed);
  window.LIVE = { ...EXAMPLE };
  renderSteps();
  initScrollObserver();
} catch(e) {
  console.error('computeAll failed:', e);
  document.getElementById('steps-container').innerHTML =
    `<div style="color:var(--red);padding:20px">${TRANSLATIONS[currentLang]['err-compute']}</div>`;
}
```

- [ ] **Step 4: Final browser test checklist**

- [ ] Mode A loads instantly with test vector values visible
- [ ] Mode B shows security banner, `🎲 Nueva entropía` button visible in step 1
- [ ] Click `🎲 Nueva entropía` → all 25 values update with new entropy
- [ ] Type a passphrase in Mode B → values update from step 6 onward
- [ ] Click EN → all labels translate (mnemonic words stay English)
- [ ] Scroll → phase tab highlights correctly
- [ ] `+ técnico` buttons expand/collapse
- [ ] Mobile: page renders on narrow viewport
- [ ] Console: no errors, no 404s

- [ ] **Step 5: Update MEMORY.md**

Update `D:/claude/Projects/MatheyBTC-seed-BTC/MEMORY.md` (or create if missing) with:
- Project live at `https://matheybtc.github.io/MatheyBTC-seed-step/`
- 25 steps implemented, two modes, ES/EN
- Uses BIP39 test vector for Mode A

- [ ] **Step 6: Final commit and push**

```bash
git add index.html MEMORY.md
git commit -m "feat: final polish — mobile responsive, error handling, WebCrypto check"
git push
```

---

## Summary

| Task | Deliverable | Verifiable by |
|---|---|---|
| 1 | HTML skeleton + CSS + header | Visual in browser |
| 2 | @noble libs vendored inline | Console: secp.getPublicKey works |
| 3 | BIP39 wordlist + encoders | Console: wordlist[0]==='abandon' |
| 4 | EXAMPLE data object | Console: EXAMPLE.mnemonic correct |
| 5 | All 25 step cards rendered | Visual: all cards visible |
| 6 | computeAll + live mode | Console: rootSeed matches test vector |
| 7 | Mode B wiring + security banner | Click Generar → banner shows |
| 8 | ES/EN i18n | Click EN → labels translate |
| 9 | Scroll phase progress bar | Scroll → tabs highlight |
| 10 | Mobile + error handling + final test | Full checklist passes |
