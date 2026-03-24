# MatheyBTC Seed BTC — Interactive Explainer Design Spec

**Date:** 2026-03-24
**Project:** `MatheyBTC-seed-BTC`
**Deployed:** GitHub Pages (static, no backend)

---

## Goal

A single-page interactive explainer showing the complete 25-step process from entropy to Bitcoin address. Two audiences served: beginners (simple explanations) and technical users (expandable detail per step). Two modes: fixed educational example (default) and live generator using real WebCrypto calculations.

---

## Architecture

Single `index.html` with all JS/CSS inline — no build step, no backend, works offline after first load. Matches MatheyBTC project conventions (invest-BTC, HW-list).

**Libraries — vendored as inline JS (no CDN calls at runtime):**
Embedding libraries directly into `index.html` avoids CDN supply-chain risk and eliminates the need for SRI hashes. Pinned versions copied from npm builds.
- `@noble/hashes` — SHA256, RIPEMD160, HMAC-SHA512, PBKDF2 (WebCrypto lacks RIPEMD160)
- `@noble/secp256k1` — secp256k1 elliptic curve operations for pubkey derivation
- BIP39 English wordlist (2048 words) embedded as inline JS array
- Base58Check encoder — implemented inline (~30 lines)
- Bech32 / Bech32m encoder — implemented inline (~60 lines)

---

## Modes

### Mode A — Ejemplo Educativo (default)
Pre-calculated values from the official BIP39 test vector are hardcoded. Page loads instantly, no computation. All 25 steps show real values derived from the test vector mnemonic.

Test vector mnemonic:
```
abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about
```

### Mode B — Generar Mi Seed
User clicks `🎲 Nueva entropía` at step 1. The app generates 128 bits of cryptographically secure random entropy via `window.crypto.getRandomValues()`, then computes all 25 steps in sequence, updating each step's value display in place.

**Optional passphrase field** visible in both modes (step 6). In Mode A shows `""`. In Mode B user can type their own — triggers full recalculation from step 6 onward.

**Security banner** shown in Mode B (translated in both languages):
> "⚠️ Solo educativo. No uses esta herramienta para guardar fondos reales. Las claves generadas son visibles en la memoria del browser y en DevTools. Nada se envía a ningún servidor — pero extensiones del browser pueden leer el contenido de la página. Para máxima seguridad, usá un dispositivo air-gapped sin extensiones."

Constraint: **128-bit entropy only (12 palabras)**. BIP39 soporta 128–256 bits pero esta herramienta muestra solo el caso de 12 palabras por claridad educativa. Documentado explícitamente en el paso 1.

---

## Visual Design

**Theme:** Dark background (`#0a0a0a`), matching MatheyBTC dark palette.

**Header:** MatheyBTC nav bar with logo ₿, links to other projects, ES/EN language toggle, mode toggle (Ejemplo / Generar).

**Progress bar:** Fixed at top while scrolling — shows current phase highlighted.

**4 phases with distinct accent colors:**

| Phase | Color | Hex | Steps |
|---|---|---|---|
| Crear Seed (BIP39) | Celeste | `#67e8f9` | 1–7 |
| Árbol HD (BIP32) | Amarillo | `#facc15` | 8–13 |
| Derivar Claves (BIP44/49/84/86) | Naranja | `#fb923c` | 14–19 |
| Construir Dirección | Blanco | `#f1f5f9` | 20–25 |

---

## Step Structure

Each of the 25 steps renders as a card with:

```
┌─[N]──────────────────────────────────────────┐
│  ▌  TITLE                                     │
│     Short explanation (2-3 lines, ES or EN)   │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │  value / data / hex / words displayed   │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  [ + técnico / + technical ]  (expandable)    │
└───────────────────────────────────────────────┘
                    ↓
```

- **Badge `[N]`**: step number, background = phase color
- **Left border**: 4px solid line in phase color
- **Value box**: monospace font, dark bg, highlighted portions in phase color
- **Expandable technical section**: hidden by default, click to reveal detailed explanation with byte-level breakdown

---

## The 25 Steps

### Phase 1 — Crear Seed / BIP39 (Celeste)

| # | Title ES | Title EN | Key computation |
|---|---|---|---|
| 1 | Entropía Aleatoria | Random Entropy | `crypto.getRandomValues(16 bytes)` → 128 bits hex |
| 2 | Checksum SHA256 | SHA256 Checksum | `SHA256(entropy)[0]` → first 4 bits appended |
| 3 | Segmentos de 11 Bits | 11-Bit Segments | 132 bits ÷ 11 = 12 segments |
| 4 | Diccionario BIP39 | BIP39 Dictionary | Each segment → index 0–2047 → word |
| 5 | Mnemónico | Mnemonic | 12 words joined |
| 6 | Passphrase + NFKD | Passphrase + NFKD | NFKD normalization applied independently to mnemonic AND passphrase before combining |
| 7 | PBKDF2 → Root Seed | PBKDF2 → Root Seed | `PBKDF2-HMAC-SHA512(password=NFKD(mnemonic), salt="mnemonic"+NFKD(passphrase), iterations=2048, dkLen=64)` → 64 bytes |

### Phase 2 — Árbol HD / BIP32 (Amarillo)

| # | Title ES | Title EN | Key computation |
|---|---|---|---|
| 8 | Root Seed Como Entrada | Root Seed Input | 64 bytes from step 7 |
| 9 | HMAC-SHA512 | HMAC-SHA512 | `HMAC-SHA512(key="Bitcoin seed", data=root_seed)` → 64 bytes |
| 10 | División IL + IR | IL + IR Split | First 32 bytes = IL (private key), last 32 = IR (chain code) |
| 11 | 3 Claves Maestras | 3 Master Keys | Master Private Key (IL), Master Public Key (IL·G), Chain Code (IR) |
| 12 | Master Node (m) | Master Node (m) | MPK + Chain Code = root of HD tree |
| 13 | xprv / xpub | xprv / xpub | 78-byte serialization: version(4) + depth(1) + parent_fingerprint(4) + child_index(4) + chain_code(32) + key(33) → Base58Check |

### Phase 3 — Derivar Claves (Naranja)

| # | Title ES | Title EN | Key computation |
|---|---|---|---|
| 14 | xprv Del Master Node | xprv From Master Node | Starting point for derivation |
| 15 | Ruta De Derivación | Derivation Path | `m/84'/0'/0'/0/0` (BIP84 default) |
| 16 | Normal vs Hardened | Normal vs Hardened | Hardened: uses private key; Normal: uses public key |
| 17 | 4 Estándares | 4 Standards | BIP44 (1xxx Legacy), BIP49 (3xxx P2SH), BIP84 (bc1q SegWit), BIP86 (bc1p Taproot) |
| 18 | Derivación Por Nivel | Per-Level Derivation | Hardened: `HMAC-SHA512(key=chain_code, data=0x00‖privkey‖index)`; Normal: `HMAC-SHA512(key=chain_code, data=pubkey‖index)` → child privkey + child chain code |
| 19 | Clave Pública Hija | Child Public Key | `child_privkey · G` on secp256k1 → compressed 33-byte pubkey (prefix `02`/`03` + X coordinate) |

### Phase 4 — Construir Dirección (Blanco)

| # | Title ES | Title EN | Key computation |
|---|---|---|---|
| 20 | Pubkey Hija Como Entrada | Child Pubkey Input | 33-byte compressed pubkey from step 19 |
| 21 | Pubkey Comprimida (33 bytes) | Compressed Pubkey | Already compressed — step shows byte breakdown: prefix byte + 32-byte X coordinate |
| 22 | SHA256 → RIPEMD160 | SHA256 → RIPEMD160 | `RIPEMD160(SHA256(compressed_pubkey))` = HASH160 → 20 bytes |
| 23 | scriptPubKey | scriptPubKey | BIP84: `OP_0 <hash160>` (P2WPKH) |
| 24 | Encoding | Encoding | Bech32 for BIP84 (`bc1q...`); Base58Check for BIP44/49 |
| 25 | Dirección Bitcoin | Bitcoin Address | Final result — safe to share publicly |

---

## Derivation Default

BIP84 (`m/84'/0'/0'/0/0`) is the default shown in steps 15–25, producing a Native SegWit `bc1q...` address. Step 17 shows all 4 standards (BIP44/49/84/86) as reference with their address prefixes.

---

## i18n

ES/EN toggle in header. All labels, titles, explanations and button text translated. Technical terms (SHA256, PBKDF2, xprv, etc.) remain untranslated in both languages. BIP39 mnemonic words always shown in English (BIP39 standard).

Translation keys stored in a `const TRANSLATIONS = { es: {...}, en: {...} }` object, applied via `data-i18n` attributes and a `applyLang()` function — same pattern as invest-BTC.

Translations must cover:
- All static labels, titles, step explanations, button text
- Security banner (full text)
- Error states: WebCrypto unavailable, computation failure
- Passphrase input placeholder text
- Dynamically injected content (step value labels, mode toggle text)

---

## File Structure

```
MatheyBTC-seed-BTC/
├── index.html          ← entire app (HTML + CSS + JS inline)
├── bitcoin_01_seed_bip39.png     ← reference images
├── bitcoin_02_arbol_bip32.png
├── bitcoin_03_derivacion_bips.png
├── bitcoin_04_direccion.png
└── docs/superpowers/specs/
    └── 2026-03-24-seed-btc-interactivo-design.md
```

No separate `.js` or `.css` files — all inline in `index.html`.

---

## Out of Scope

- Actual wallet generation for real funds (this is educational only)
- QR code display of addresses
- Multiple accounts or address index navigation
- Signet / Testnet toggle
- Mobile-specific native app features
