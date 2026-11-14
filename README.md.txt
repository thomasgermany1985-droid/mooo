# InvestorsMoo — 100 % free setup guide (no backend, no billing)

Everything runs on **Firebase's free Spark plan** (Authentication + Realtime Database + Hosting) and your **FreeDNS** subdomain.
**No Cloud Functions, no Blaze plan, no credit card.** All security is enforced by the database's own rules.

```
investorsmoo/
├─ public/index.html        ← user website  (the ONLY file you upload to hosting)
├─ admin.html               ← admin panel   (stays on your PC, never uploaded)
├─ database.rules.json      ← security rules (you paste this into the Firebase console)
├─ firebase.json            ← optional: only for deploying with the Firebase CLI
└─ tools/                   ← optional offline tests (need only Node.js, no internet)
```

---

## 1. Free services used (everything in the system)

| Purpose | Service | Free allowance (checked on the official pages) | Needs card? |
|---|---|---|---|
| Login / registration / password reset e-mails | **Firebase Authentication** (Spark) | E-mail + password sign-in; free up to 50,000 monthly active users | No |
| Database | **Firebase Realtime Database** (Spark) | 1 GB stored · 10 GB downloaded / month · 100 simultaneous connections | No |
| Website hosting + HTTPS + custom domain | **Firebase Hosting** (Spark) | 10 GB storage · 360 MB transfer / day · custom domain & SSL included | No |
| Your domain name | **FreeDNS (freedns.afraid.org)** – `investors.mooo.com` | Free subdomain; you add the DNS records Firebase gives you | No |
| Live crypto prices | **CoinGecko public API** (no key) | About 10–30 calls / minute shared per IP; intended for prototypes/demos | No |
| Country flags | **flagcdn.com** | Free flag images | No |
| Font | **Google Fonts (Inter)** | Free | No |
| Firebase JavaScript SDK | **gstatic.com CDN** (`www.gstatic.com/firebasejs`) | Free | No |
| Deploy tool (optional) | **Firebase CLI** (`npm i -g firebase-tools`) + **Node.js** | Free, open-source | No |
| Run `admin.html` locally | **Python 3** (`python -m http.server`) – or VS Code "Live Server" | Free | No |
| E-mail for password resets | Sent by Firebase Authentication itself | Included | No |

**Not used (because they are paid or need Blaze):** Cloud Functions, Cloud Run, Firebase Storage (avatars are stored as small Base64 JPEGs in the database instead), Firebase App Check with paid providers, SMS / phone auth.

> ⚠️ **Stay on Spark.** When you create the project, never click **Upgrade / Blaze**. Without a billing account Google cannot charge you.
>
> Free-tier reality check: the Firebase Console shows your usage under *Realtime Database → Usage* and *Hosting → Usage*. For a demo with a few dozen users you will use a tiny fraction of these limits. CoinGecko's keyless API is shared by IP and can answer "429 Too Many Requests" at busy times — the Crypto page then shows the last saved prices.

Sources: [Firebase pricing](https://firebase.google.com/pricing) · [CoinGecko keyless API](https://docs.coingecko.com/docs/keyless-public-api)

---

## 2. Step-by-step setup (all done by hand in your browser)

### Step 1 — Create the Firebase project
1. Go to <https://console.firebase.google.com> and sign in with a Google account.
2. **Create a project** → name it (e.g. `investorsmoo`) → turn **Google Analytics OFF** → *Create*.
3. Do **not** upgrade the plan. The default is **Spark (free)**.

### Step 2 — Register the web app and get the config
1. Project overview → click the **`</>` Web** icon → nickname `investorsmoo` → *Register app*.
   (Leave "Also set up Firebase Hosting" unticked for now.)
2. You will see a `firebaseConfig = { apiKey: …, authDomain: …, databaseURL: …, … }` block. Copy it.
3. Open **both** files in a text editor and replace the placeholder `firebaseConfig` near the top of the `<script>`:
   * `public/index.html`
   * `admin.html`
4. In `public/index.html` also set your advisor link (used by every *Contact advisor* and *Add Funds* button):
   ```js
   const ADVISOR_URL = "https://your-link-here";
   ```
> The config is not secret. Security comes from Authentication + the database rules below.

### Step 3 — Turn on Authentication
1. **Build → Authentication → Get started → Sign-in method → Email/Password → Enable → Save.**
   (Leave "Email link" off. No e-mail verification is needed for this project.)
2. **Authentication → Settings → Authorized domains**: `localhost` and your `*.web.app` / `*.firebaseapp.com` domains are already there.
   You will add `investors.mooo.com` here later (Step 9).

### Step 4 — Create the Realtime Database
1. **Build → Realtime Database → Create database** → choose a location → **Start in locked mode** → Enable.
2. Look at the URL shown at the top of the Data tab. If it is **not** `https://<project>-default-rtdb.firebaseio.com` (for example it contains `europe-west1` or `asia-southeast1`), make sure the same URL is in `databaseURL` in **both** HTML files.

### Step 5 — Paste the security rules (this is your "backend")
1. **Realtime Database → Rules tab.**
2. Delete everything there, open `database.rules.json` from this project, copy **all** of it (it starts with `{ "rules": {`), paste it, click **Publish**.
3. These rules are what stop users editing balances, guessing PINs, double-withdrawing, or reading other people's data. Do not skip or edit them.

### Step 6 — Make yourself the administrator
1. **Authentication → Users → Add user** → enter *your* admin e-mail and a **long, unique password** → *Add user*.
2. Copy that user's **User UID** (the long code in the table).
3. **Realtime Database → Data tab.** Hover the root (the database URL line) → click **＋** →
   * Name: `admins`  → click **＋** next to it →
   * Name: **your UID**, Value: **`true`** (typed exactly `true` — a boolean, not in quotes).
   * *Add*.
4. This is the only way to become admin: the rules forbid anyone (including you, from the website) from writing to `admins`. Only the Firebase console can.

### Step 7 — Run the admin panel on your PC
`admin.html` must be opened from a tiny local web server (Firebase sign-in does not work from a `file://` address):
```bash
cd investorsmoo
python -m http.server 8080        # Windows: py -m http.server 8080
```
Then open **http://localhost:8080/admin.html** and sign in with the admin e-mail/password.
(`localhost` is already an authorized domain.) Any account that is not in `admins` is refused immediately, and the database refuses its admin reads/writes anyway.

*Alternative:* in VS Code install the free **Live Server** extension → right-click `admin.html` → *Open with Live Server*.

### Step 8 — Deploy the user website for free
**Option A — Firebase Hosting (recommended: free HTTPS + custom domain)**
```bash
npm install -g firebase-tools
firebase login
cd investorsmoo
firebase use --add            # pick your project, give it any alias
firebase deploy --only hosting
```
Your site is now live at `https://<project-id>.web.app`. Only the `public/` folder is uploaded, so **`admin.html` is never published**.

**Option B — no command line:** Firebase Console → *Build → Hosting → Get started* shows the same steps; or use any other free static host (Cloudflare Pages, Netlify Drop, GitHub Pages) by uploading just `public/index.html` — then add that site's domain to *Authentication → Authorized domains*.

> 🔒 **HTTPS is required.** The transaction-PIN feature uses the browser's built-in crypto, which browsers only provide on secure (`https://`) pages. On plain `http://investors.mooo.com` the PIN and withdrawal features will show an error. Firebase Hosting gives you HTTPS automatically.

### Step 9 — Point `investors.mooo.com` (FreeDNS) at Firebase Hosting
1. Firebase Console → **Hosting → Add custom domain** → enter `investors.mooo.com` → *Continue*.
2. Firebase shows DNS records to add (usually a **TXT** verification record and one or two **A** records).
3. Log in to <https://freedns.afraid.org> → **Subdomains** → edit `investors` (or add it) → create the records exactly as Firebase shows (type, host, value). Save.
4. Back in Firebase click *Verify*. DNS and the free SSL certificate can take from a few minutes to several hours.
5. **Authentication → Settings → Authorized domains → Add domain → `investors.mooo.com`.**
6. If the custom-domain verification does not work for a shared `mooo.com` subdomain, just use the free `https://<project-id>.web.app` address — everything works identically.

### Step 10 — Test everything (10 minutes)
1. Open the site → **Create account**. You land on the dashboard with $0.00 everywhere and a *Welcome* notification.
2. **Admin → Users →** open the user → **Account** → set Total Invested to `5000`, Stocks `2000`, etc. → Save. The user's dashboard and chart update instantly.
3. User → **Profile → Create PIN** (4 digits).
4. User → **Home → Withdraw** → add a bank account → enter an amount → enter the PIN → request is **Pending** in *History*.
5. **Admin → Withdrawals → Review →** *Mark as paid*. The user's History shows **Success**, Total Invested drops, Reserved returns to 0.
6. Try **Cancel** on another pending withdrawal; enter a wrong PIN 5 times to see the 15-minute lockout; mark a paid one *Reversed*.

---

## 3. How the money works

| Value | Where | Meaning |
|---|---|---|
| **Total Invested Value** | `stats/{uid}/totalInvested` | The figure on the dashboard; edited only by the admin |
| **Reserved** | `stats/{uid}/reserved` | Sum of Pending / On-Hold withdrawals (held, not yet paid) |
| **Available** | `totalInvested − reserved` | What the user may still request |

| Event | What changes (in one atomic database write) |
|---|---|
| User submits a withdrawal (PIN proven) | request saved as **Pending** · `reserved += amount` |
| User cancels (only while Pending) | status **Cancelled** · `reserved −= amount` |
| Admin **declines** | status **Declined** · `reserved −= amount` |
| Admin marks **paid** | status **Success** · `reserved −= amount` **and** `totalInvested −= amount` |
| Admin **reverses** a paid withdrawal | original kept, marked **Reversed**; a *separate* linked `RV-…` **Reversal** transaction credits `totalInvested += amount` |

The rules reject any write whose numbers don't match that table, so a user can't invent or release funds, a double-click can't deduct or refund twice, balances can't go negative, and reserved can never exceed the total.

**Transaction PIN (no server):**
* The browser turns the PIN into a salted **PBKDF2-SHA-256** hash (150,000 rounds). The plain PIN never leaves the device or is stored.
* The hash lives at `pins/{uid}/hash`, which **no client can read** — the database rules compare the user's proof to it internally.
* Every attempt is counted *before* it is checked. **5 wrong tries → locked for 15 minutes.** A proof is valid for 2 minutes and works once.
* Forgot PIN → advisor/admin presses **Reset PIN** in the admin panel (Users → user → Security).

**Statuses:** Success · Pending · On Hold · Declined · Reversed · Cancelled. Valid moves: Pending ⇄ On Hold, Pending/On Hold → Success or Declined, Success → Reversed. Nothing else is accepted.

---

## 4. Database structure
```
admins/{uid}                 true                     (set by hand in the console only)
profiles/{uid}               firstName lastName email age nationality nationalityCode phone
                             status(active|suspended) deleted deletedAt createdAt
avatars/{uid}                "data:image/jpeg;base64,…"   (256×256, made in the browser)
settings/{uid}/theme         light | dark
stats/{uid}                  totalInvested percentage stocks bonds crypto reserved lastWd updatedAt
statsHistory/{uid}/*         {t, v}     chart points (added whenever the admin saves or a payout happens)
transactions/{uid}/{id}      id type amount status date description createdAt updatedAt deleted [relatedTx]
withdrawals/{uid}/{id}       id uid amount status createdAt updatedAt bankName accountName maskedAccount maskedIban
                             bankId [paidAt declinedAt cancelledAt reversedAt reversalTxId]   (what the user sees)
withdrawalBank/{uid}/{id}    bankId bankName accountName accountNumber [iban]   (full details — ADMIN READ ONLY)
withdrawalAdminNotes/{id}    "internal note"                                   (admin only)
banks/{uid}/{bankId}         bankName accountName accountNumber [iban] createdAt   (owner + admin)
notifications/{uid}/{id}     title message type read deleted createdAt [editedAt deletedAt broadcastId]
gainers/{id}                 name symbol price change color deleted [deletedAt]
adminNotes/{uid}             note updatedAt by                                  (admin only)
auditLog/{id}                at by byEmail action target details                (admin only, append-only)
pins/{uid}                   hash salt [attempts lastAt pending{proof,at}]      (hash is unreadable by every client)
```

## 5. What the rules guarantee (summary)
* **Default deny.** Users read only their own data. Admin (`admins/{uid} === true`) reads collections.
* **Users can write only:** their own first/last name, avatar, theme, saved bank accounts, notification *read* flag, their own registration (profile + zero stats + welcome notice, once), and the exact withdrawal / cancellation / PIN-attempt shapes described above.
* **Users can never write:** balances, transactions, other profile fields (age, nationality, phone, e-mail, status), gainers, notes, audit log, PIN hash.
* **Admins can never hard-delete** profiles, transactions, notifications, gainers or withdrawals — the rules only accept `deleted: true` (soft delete) and allow restoring. Withdrawal amounts cannot be changed by anyone.
* **Suspended / archived accounts** lose all read/write access immediately (they can still sign in to Firebase, but the database shows them nothing).
* The audit log can only be appended, and each entry must carry the real admin's UID and the server time.

## 6. Honest limits of a free, backend-less design
These are the trade-offs of not having a server. For a private demo they are normally acceptable.
* **You (the project owner) are fully trusted.** Anyone who can open your Firebase console can read/edit all data.
* **A 4-digit PIN is weak by nature.** Hashing + the 5-tries/15-minute lock stop online guessing, but someone who could read the hash in the console could brute-force 10,000 values offline. Keep the console account secure (2-step verification on your Google account).
* **No one-time reset *link* generation** (needs a server). Instead the admin panel sends Firebase's standard **password-reset e-mail** to the user.
* **Changing a user's *sign-in* e-mail or password** can't be done from the panel — use Firebase Console → Authentication. The panel edits the profile/contact e-mail shown in the system.
* **Suspension is enforced by the database rules** (the user sees nothing and can change nothing), not by disabling the Firebase login.
* **Registration is open** — anyone with the site link can create an account (e-mail verification was not requested). To block strangers, share the link only with colleagues, or temporarily disable *Email/Password* sign-up in the console.
* **Admin screens download all records** and listen live. This is fine for dozens/hundreds of records inside the free 10 GB/month; for thousands of users you'd want pagination.
* **Browser-side price data:** CoinGecko may rate-limit; the page falls back to the last saved prices.

## 7. Offline tests (optional, need only Node.js — no internet, no installs)
```bash
node tools/test-rules.js          # 132 checks of the rules (money, PIN lockout, privacy, soft delete…)
node tools/integration-test.js    # 57 checks running the real client code against the rules
node tools/build-rules.js         # regenerates database.rules.json if you edit the rule generator
```
The simulator is an approximation of Firebase's rules engine. For extra certainty you can also run Firebase's free local emulator (needs Java) or try single writes in the console's **Rules Playground**.

## 8. Troubleshooting
| Symptom | Fix |
|---|---|
| Page says "Firebase is not configured" | Paste your `firebaseConfig` into the file (Step 2). |
| Admin login says "not authorised" | The `admins/<UID>` node is missing or isn't boolean `true` (Step 6). Sign out and in again. |
| "Permission denied" everywhere | Rules not published (Step 5), or `databaseURL` doesn't match your database region (Step 4). |
| `auth/unauthorized-domain` | Add the domain in Authentication → Settings → Authorized domains. |
| PIN / withdrawal error about secure connection | You're on `http://`. Use the `https://` Hosting address. |
| "The database rejected this change" in admin | Data changed while you were editing (e.g. the user cancelled). Reload and retry. |
| Crypto page shows old prices | CoinGecko rate limit — wait a minute; it retries automatically. |
| New user can't register: "That email is already registered" | Use Sign in, or delete the user in Authentication → Users. |
| Forgot admin password | Authentication → Users → ⋮ → *Reset password*. |

## 9. Keeping it free
* Never press **Upgrade** in Firebase.
* Watch *Realtime Database → Usage* once in a while (stay under 1 GB stored, 10 GB/month).
* Avatars are automatically shrunk to 256×256 JPEG (~20–40 KB each) to protect the 1 GB limit.
* Don't leave `admin.html` open on many tabs/devices — each open tab keeps live connections (limit: 100 simultaneous).
