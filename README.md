# Ribhi Pro (ربحي برو)

A Salla app that shows store owners the real profit on every order, and reads supplier invoices with AI so product costs stay right.

> **Source code is private.** Ribhi Pro is a commercial product. The code is available to hiring teams on request; see the contact details on my CV.

**Project page:** https://claude.ai/artifact/5R7aFd11kmN2GeW7ChftR1

## The problem

Salla shows a merchant what they sold. It doesn't show what they kept after the product cost, shipping, payment fees and VAT. Many products have no cost entered, and supplier prices change with every delivery.

## What it does

- **Profit on every order:** revenue after VAT, minus product cost (per size), shipping and the payment fee for the method used. Figures that rest on an estimate are labelled.
- **Profit dashboard:** today, week and month; a sales-to-profit waterfall; the products that lose money and why.
- **Products page:** each product's cost and where it came from (Salla, invoice, manual or estimate). Costs edited here are written back to Salla.
- **AI invoice reading:** upload a PDF or a phone photo of a supplier invoice. The app reads it, matches each line to a product and size, and the merchant reviews and confirms. The new cost goes to Salla and past orders are recalculated.

## How it works

1. **Install:** Salla sends the store's access token. The app opens inside the merchant's Salla dashboard and checks who's signed in through Salla's token introspection.
2. **First sync:** a background job pulls all products and 90 days of orders, then works out the profit on each order.
3. **Live orders:** signed webhooks (new, updated, cancelled, refunded) trigger a recalculation within seconds.
4. **Invoices:** read → checked → re-read if the numbers don't add up → matched → reviewed → confirmed.

## Invoice reading: checking the AI's arithmetic

Real invoices have handwriting, thermal receipts, Arabic-Indic digits, stamps over numbers, discounts, and "paid" or "change" rows that look like totals. Every reading is checked before the merchant sees it:

- quantity × unit price = line amount
- the lines add up to the subtotal
- the total, after discount, extra costs and VAT, adds up (prices with VAT included are handled too)
- the Saudi VAT number format (15 digits, starting and ending with 3)
- Arabic-Indic digits and Arabic decimal separators are converted

If a check fails, the invoice is read again in two steps: first the text is copied word for word, then JSON is built from the copy. Any number missing from the copy is flagged, and the reading whose totals match better is kept. What still doesn't add up is shown at the top of the review page.

Products are matched by exact SKU, then near SKU (one character off, or parts reversed by right-to-left text), then cleaned name, then what was learned from this supplier before. Discounts and shipping or customs costs are spread across the lines by value.

### Measured results

The method was chosen by scoring it against an answer key: 10 realistic invoices built to reproduce real-world problems, plus 5 clean ones, each read 3 times by each method.

| Method | Hard invoices | Clean invoices | Readings with warnings | Cost / invoice | Time / invoice |
|---|---|---|---|---|---|
| Direct reading | 83.9% | 98.0% | 13 of 30 | $0.0009 | 22 s |
| Two-step reading | 82.6% | 95.9% | 12 of 29 | $0.0010 | 27 s |
| **Direct, then two-step if needed (shipped)** | **85.0%** | **97.7%** | **5 of 30** | $0.0012 | 28 s |

## Engineering notes

- **Exact money:** amounts are stored as `decimal(12,2)`, never floats.
- **Token refresh with a lock:** Salla refresh tokens work only once, so refreshing runs under a per-store lock.
- **Webhook security:** every event's HMAC signature is checked. Events from non-live stores and stale ones are skipped, and a deleted order can't be brought back by a late event.
- **Privacy:** no personal customer data is stored, only financial figures. Invoice files are deleted after 90 days, and everything is removed on uninstall.
- **Native inside Salla:** Salla's tabs, toasts, confirm dialogs and save button. Arabic or English with right-to-left layout, following the merchant's dashboard language.
- **Billing from Salla events:** plans, a 7-day trial and subscriptions, with monthly order counting and a grace period.
- **Pluggable AI readers:** Claude and Z.ai GLM behind one interface, with a fake reader for tests and a command that compares models on accuracy, cost and time.
- **235 automated tests:** PHPUnit, with the AI and Salla calls faked.

## Stack

| | |
|---|---|
| Backend | Laravel 13, PHP 8.3, MySQL, queued jobs, scheduled commands |
| Platform | Salla Admin API, OAuth, webhooks, Embedded SDK |
| AI | Vision LLMs (Anthropic Claude, Z.ai GLM) returning structured JSON |
| Frontend | Blade, Tailwind CSS, Vite, right-to-left first |
| Quality | PHPUnit, Laravel Pint |
