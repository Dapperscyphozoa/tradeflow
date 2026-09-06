# TradeFlow

Quotes, jobs and invoices for Australian trades. Native macOS, one binary, one SQLite file.
No Electron, no browser, no account, no server required. Subscription enforced by a signed
licence the app verifies offline.

---

## What it does

| | |
|---|---|
| **Customers** | Contacts, site addresses, ABN, per-customer GST eligibility, archive, money owed |
| **Quotes** | Numbered per business, per-line GST toggle, PDF, one click to a job |
| **Jobs** | Diary by scheduled date, status through to invoiced, one click to an invoice |
| **Invoices** | Tax invoices at 10% GST, part payments, overdue tracking, PDF |
| **Payments** | Status is derived from money received — never typed in |
| **Dashboard** | Outstanding, overdue, received, quotes out, twelve-month receipts, what needs attention |
| **Settings** | Business details, logo, bank details, numbering, subscription, backup |

Quote → job → invoice is a single click at each step and each conversion is refused twice, so
the same job cannot be invoiced two ways.

### GST

10%, rounded to cents **at the line and then summed** — never on the total. The two disagree by
a cent often enough to cost you an argument, and the ATO expects the former. A business not
registered for GST charges none, whatever a line says. `is_tax_invoice` and the ABN are frozen
on the invoice at issue, so registering for GST later never changes the face of an invoice
already sent.

### Your data

One SQLite file at `~/Library/Application Support/TradeFlow/tradeflow.sqlite`. Copy it and you
have copied the business. Nothing is uploaded. The only network call the app can make is an
optional licence check, and only if you set a server address.

---

## The subscription kill switch

The app carries the **public** half of an Ed25519 key. `tfkey` holds the private half. A licence
is a small signed JSON payload:

```
{"v":1,"business_id":"…","name":"…","plan":"pro","device":"*",
 "issued":"…","expires":"…"}    →  base64url(payload).base64url(signature)
```

Three ways a subscription ends, in increasing severity:

| State | When | Effect |
|---|---|---|
| `valid` | before `expires` | runs |
| `grace` | up to 7 days after `expires` | runs, warns on every screen |
| `lapsed` | after grace | **locked** — data untouched on disk |
| `revoked` | server says so | **locked** at the next check, no grace |
| `invalid` | signature does not verify | **locked** |

Offline is neither a free pass nor a lockout: the signed expiry is the offline authority, so a
copied app stops working on its own, and a tradesman on a site with no signal keeps working
until the licence genuinely runs out. Winding the Mac's clock back does not revive a lapsed
licence — the last date seen is recorded and trusted over a backwards jump.

Nothing is deleted when a licence lapses. The gate goes back up; the data stays.

### Issuing licences — `tfkey`

```bash
build/tfkey pubkey                                # the key baked into the app
build/tfkey new "Coastal Plumbing & Gas" pro 365  # mint — prints the key to give the customer
build/tfkey new "Trial Co" trial 14               # a 14-day trial
build/tfkey list                                  # everything minted
build/tfkey check <key>                           # what the app will make of it
build/tfkey revoke <business-id>                  # kill it
build/tfkey restore <business-id>
build/tfkey serve 8799                            # phone-home endpoint for revocation
```

The private key lives at `~/.tradeflow/issuer.key` (0600) and the register at
`~/.tradeflow/register.json`. **Anyone holding that key can mint licences** — back it up
somewhere you control, and do not ship it.

The everyday kill is a short-dated key: sell monthly, mint 31 days, stop minting when they stop
paying. `serve` exists for when you want to cut someone off mid-term — run it anywhere the
customer's Mac can reach, put the address in Settings → Subscription, and `revoke` takes effect
the next time they bring the app to the front.

---

## Building

Requires only the Command Line Tools — no Xcode, no SwiftPM, no third-party dependency.

```bash
./test.sh     # 62 checks: GST, licence forgery, quote-to-cash, tenant isolation, PDF
./build.sh    # → build/TradeFlow.app, ad-hoc signed
```

`build.sh` works around a stale `module.modulemap` the Command Line Tools leave behind
(both it and `bridging.modulemap` define `SwiftBridging`, which is a hard error) by shadowing
the stale file with a VFS overlay. Nothing under `/Library` is touched.

### Layout

```
Sources/
  TradeFlowApp.swift    @main, app delegate, menu commands
  RootView.swift        sidebar, licence gate, grace banner
  LicenceGateView.swift the gate, and the app mark
  DashboardView.swift   stat tiles, receipts chart, attention list
  CustomersView.swift   table + editor + shared sheet/search chrome
  QuotesView.swift      table + editor
  JobsView.swift        table + editor
  InvoicesView.swift    table + editor + payment sheet
  SettingsView.swift    business, payment, subscription, data
  LineItemsEditor.swift the line grid shared by quotes and invoices
  Store.swift           every query; business_id on every row
  DB.swift              SQLite, schema, dates
  Models.swift          the row types
  Money.swift           GST — a port of the server's gst.py, to the cent
  Licence.swift         Ed25519 verification, states, the vault
  PDFExport.swift       quote and invoice PDFs, drawn with CoreGraphics
  DemoData.swift        the worked example, loaded on request only
  Theme.swift           brand, formatting, cards, pills, page scaffold
tools/tfkey.swift       the licence authority
tests/main.swift        the harness
icon/make-icon.swift    the app icon, drawn not imported
```

---

## Signing for distribution

The build is ad-hoc signed, which is enough to run on this Mac. To send it to a customer without
Gatekeeper stopping them you need a Developer ID certificate:

```bash
codesign --force --deep --options runtime --timestamp \
  --sign "Developer ID Application: <your name> (<team id>)" build/TradeFlow.app
xcrun notarytool submit build/TradeFlow.app --keychain-profile <profile> --wait
xcrun stapler staple build/TradeFlow.app
```

Until then a customer opens it the first time with right-click → Open.
