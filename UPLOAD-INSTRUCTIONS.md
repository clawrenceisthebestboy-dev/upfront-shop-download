# How to take the download page live

This folder is a static site. Four files:

```
shop-download-site/
├── index.html                   — the download page
├── latest.json                  — manifest the running app fetches
├── logo.png                     — shop logo used by the page
└── robots.txt                   — "search engines stay out"
```

You pick where to host it. Three options, ranked by fit for this situation.

---

## Option A (recommended) — subdomain on your existing domain

**URL:** `https://shop.upfrontautorepair207.com/`

This keeps the public site (`upfrontautorepair207.com`) completely separate from the internal download page. Customers see nothing. Staff go to `shop.` and get the installer. It's still YOUR domain — you already own it, you already trust it, your DNS records already work.

**Steps:**

1. Log into your domain registrar (wherever `upfrontautorepair207.com` is registered — GoDaddy, Namecheap, Cloudflare, etc.).
2. Add a DNS A or CNAME record: `shop` → wherever you want to host the files. Easiest option is a free static-site host (see below under Option B) and point the CNAME at that.
3. Upload the four files in this folder to the host's root (via cPanel File Manager, FTP, or the host's web UI).
4. Put the installer file `UpFrontShopSetup-1.4.0.exe` in the same folder. (That file is produced by running `build\build.bat` on your Windows build machine — it's not in this zip.)
5. Fill in the two `PASTE_SHA256_HERE_AFTER_BUILDING` placeholders:
   - in `index.html` (line ~127, inside `<div class="hash" id="sha">`)
   - in `latest.json`
6. Sign `latest.json` by running `python tools\sign_manifest.py --key release_signing_key.pem --in latest.json --out latest.json` from the upfront-shop source. That replaces the `PASTE_SIGNATURE_HERE_…` placeholder with the real Ed25519 signature.
7. Re-upload the filled-in `index.html` and signed `latest.json`.

Done. Visit `https://shop.upfrontautorepair207.com/` on any browser — download page appears.

---

## Option B — free static-site host (fastest to get live, no web host needed)

Good fit if you DON'T already have a web host with FTP access for the main site, or if you don't want to deal with DNS at all.

Best picks:

- **Cloudflare Pages** — free, unlimited bandwidth, supports custom domains with a click. Upload the folder via their dashboard or connect a GitHub repo. Can password-protect with Cloudflare Access (also free for up to 50 users).
- **Netlify** — same idea, drag-and-drop the folder onto netlify.com, get a free `*.netlify.app` URL instantly. Add your custom domain in one step.

If you use one of these, your URL will either be:
- The free host subdomain — e.g. `https://upfront-shop.pages.dev/` (Cloudflare) or `https://upfront-shop.netlify.app/` (Netlify). **Works immediately, no DNS needed.**
- A custom domain — e.g. `https://shop.upfrontautorepair207.com/` pointing to the free host.

Either works. If you're using a free host subdomain, change one line in `app/updater.py` where the manifest URL is configured and rebuild the app.

---

## Option C — a hidden folder on the main web host

Put everything in a folder like `upfrontautorepair207.com/internal-shop-portal-xyz/` and just don't link to it. This works, but it's the weakest option:

- Search engines can still find it if anyone ever links to it.
- It'll share TLS / server config with the public site — any mistake on the public site leaks through.
- If you ever switch web hosts for the main site, the download page moves with it.

Use this only if you're time-constrained and already have FTP access to the main host. Upload the four files to a deep path, e.g. `/internal-shop-portal-7b4c/`, and the URL becomes `https://upfrontautorepair207.com/internal-shop-portal-7b4c/`. The `robots.txt` in this folder will keep honest search engines out.

---

## After you pick a host — update the app

Two places in the app code need to match the URL you chose:

**1. The auto-updater's default manifest URL.**

Open `app/db.py` (in the upfront-shop source) and look for the `DEFAULT_SETTINGS` dictionary. After the next edit there'll be a key `update_manifest_url` (it's on my short list to add). Set it to:

```python
"update_manifest_url": "https://<wherever-you-picked>/latest.json",
```

Then rebuild the app (`build\build.bat`). From that build forward, the shop laptop fetches updates from the URL you chose.

**2. The link inside `index.html` that points at the installer.**

The button currently points at `UpFrontShopSetup-1.4.0.exe` (relative — same folder). If you put the installer somewhere else, edit the `<a class="btn" href="…">` line.

---

## Optional: password-protect the page

The page is marked `noindex` and sits on a subdomain customers don't know about — but it's still publicly reachable by anyone who types the URL. If you want belt-and-suspenders:

- **Cloudflare Pages** → Cloudflare Access → add email rule (free).
- **Any cPanel host** → create an `.htaccess` file in the same folder:

```
AuthType Basic
AuthName "Up Front Shop"
AuthUserFile /path/to/.htpasswd
Require valid-user
```

…and an `.htpasswd` with a hashed password you set via `htpasswd -c .htpasswd josh`. Anyone visiting the page gets a browser password prompt first.

I'd skip this for v1 — the page isn't a secret, just private. Revisit if you ever hand the URL to someone outside the shop.

---

## Total time to live

- If you pick Cloudflare Pages and use their free subdomain: **~5 minutes**.
- If you pick the custom subdomain `shop.upfrontautorepair207.com`: **~20 minutes** (DNS propagation).
- If you pick the hidden path on the main site: **~10 minutes**.

If you want me to tailor these instructions to a specific host (cPanel, Cloudflare, Netlify, whoever hosts `upfrontautorepair207.com` today) — tell me which one and I'll cut straight to the step-by-step.
