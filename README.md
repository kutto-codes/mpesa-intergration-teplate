# Club POS (Django)

A point of sale for a nightclub or bar (also fine for a shop): open tabs, waiters and tables, bar and kitchen
order screens, shift cash-up, bottle and tot stock, happy hour pricing, manager-PIN approvals, an audit log,
barcode scanning, cash / M-Pesa / card, 80mm receipts, reports. Django + plain JS. No door/cover charge.

## Run it
```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py seed_demo          # demo menu, tables, users manager / cashier / waiter (password pos12345, PIN 1234)
python manage.py runserver
```
Open http://127.0.0.1:8000 . **Change the demo passwords** (Admin -> Users) before real use.
Create your own admin with `python manage.py createsuperuser`.

## Roles (Admin -> Users -> Groups)
- **Waiter**: opens tabs, adds rounds, sees the bar/kitchen screens. Can never take payment.
- **Cashier**: everything a waiter does, plus takes payment, opens/closes their own shift, direct sales.
- **Manager** (group `Manager`, or superuser): everything, plus Reports, Inventory, voids, Audit log, Admin.
  A manager sets their approval PIN under **My PIN**.

## A night at the club
1. **Cashier opens a shift** (Shift page) and enters the float counted into the till. No shift, no payments.
2. **Waiter opens a tab** (table and/or customer name) and taps drinks and food, then **Send order**. Stock is
   taken immediately and one ticket per station (Bar, Kitchen) is created. Add more rounds any time.
3. **Bar and Kitchen screens** (nav links) show new tickets within seconds. Tick *Auto-print* on the bar PC to
   print each new ticket automatically (see printing note below). Press **Done** when served.
4. **Removing an item** from a tab asks for a **manager PIN** and a reason. The removal is logged with who asked
   and who approved, stock goes back, and a cancelled item shows struck through on the bar screen.
   5 wrong PINs in 10 minutes locks that user out of PIN entry.
5. **Split / merge / move table** from the tab screen. Tick items and move them to a new or existing tab, then
   pay each tab separately. Moving never touches stock.
6. **Cashier closes and pays the tab** (cash, M-Pesa or card). Receipt prints. Voids stay manager-only.
7. **Cash-up**: the cashier counts the till and submits the amount **before seeing** what was expected (blind
   count). The report then shows expected (float + cash sales), counted and the short/over difference.
   Shortages and overages are written to the audit log.

## Bottles and tots
In Admin -> Products set **Tots per bottle** (e.g. 25) and **Tot price** on a spirit. The sell screens then show
the bottle and a "(tot)" version. Tots come from an open bottle; a new bottle is opened (stock minus 1) when it
runs out, and tots returned by a removal or void refill the open bottle. Stock shows in tots for the tot button.
Cost per tot = bottle cost / tots per bottle, so profit reports stay right.

## Happy hour
Admin -> Happy hours: start and end time, the days it starts on (0=Mon ... 6=Sun), discount %, and optional
categories (empty = everything). Windows can pass midnight (22:00 to 02:00). The discounted price is saved on each
tab item and receipt line with a label, so changing the offer later never rewrites old bills.

## Printing the bar and kitchen tickets
Browsers cannot silently choose a printer. For automatic printing, open the Bar (or Kitchen) screen in Chrome on the
computer next to that printer, set the thermal printer as default, and start Chrome with `--kiosk-printing`.
Without that flag the print dialog appears for each ticket. The screens also work as a plain on-screen queue.

## Daily use
1. Set shop name, address, phone, KRA PIN, VAT rate and receipt footer in Admin -> Shop settings.
2. Add products in Admin -> Products (price, cost, barcode, stock). Tick off "track stock" for services.
3. Sell: type or scan a barcode (scanners act like a keyboard and press Enter), tap items, pick payment, Charge.
4. The receipt opens and prints automatically (set your thermal printer as the browser's default,
   paper size 80mm). Use Print again for reprints (reprints are counted).
5. Restock under Inventory. Mistakes: open the receipt and **Void** (returns stock, keeps the record).

## Design rules worth keeping
- Completed sales are never edited or deleted. Corrections are voids. (Admin shows sales read-only.)
- Each sale item stores the product name, price and cost at the time of sale.
- Checkout is one database transaction: stock, sale and movements all succeed or all roll back.
- Money is `Decimal` on the server and integer cents in the browser. The server recalculates everything.
- VAT is treated as already included in prices; the receipt shows the VAT portion.

## M-Pesa (Daraja STK Push)
Cashiers can either send an STK prompt or type the confirmation code from the customer's SMS.
STK needs configuration through environment variables:
```bash
export MPESA_ENV=sandbox                      # or production
export MPESA_CONSUMER_KEY=...
export MPESA_CONSUMER_SECRET=...
export MPESA_SHORTCODE=174379                 # your Paybill/Till (sandbox: 174379)
export MPESA_PASSKEY=...                      # from the Daraja portal
export MPESA_TRANSACTION_TYPE=CustomerPayBillOnline   # Till numbers: CustomerBuyGoodsOnline
export MPESA_CALLBACK_BASE_URL=https://your-public-url   # ngrok URL while developing
export MPESA_CALLBACK_SECRET=$(python -c "import secrets;print(secrets.token_urlsafe(24))")
```
The callback URL registered with each request is `<base>/mpesa/callback/<secret>/`. The secret reduces
unsolicited requests but does not cryptographically authenticate the callback; correlate it to a pending
payment and reconcile with Safaricom when appropriate. Safaricom must be able to reach it over public HTTPS.
Without these settings the Send prompt button is disabled and manual codes still work.

**Status:** the STK code follows Safaricom's documented request and callback format and the callback
handling is unit tested, but it has NOT been run against the real Daraja sandbox from here.
Test it with your own sandbox keys before relying on it, and compare against the current Daraja docs.
Manual codes are only checked for length (not verified with Safaricom).

For a reusable Django/Daraja starting point with example client, callback view,
and production caveats, see [the M-Pesa integration template](docs/mpesa-daraja-django-template.md).

## Production checklist
- Set `DJANGO_DEBUG=0`, a long random `DJANGO_SECRET_KEY`, `DJANGO_ALLOWED_HOSTS=yourdomain.com`,
  and `CSRF_TRUSTED_ORIGINS=https://yourdomain.com`.
- Serve over HTTPS (needed for M-Pesa callbacks anyway). Run with gunicorn behind nginx or Caddy,
  and `python manage.py collectstatic`.
- Move from SQLite to PostgreSQL if you have several tills; back up the database daily.

## Not included yet (good next steps)
- **Offline mode** (queue orders when the internet drops and sync later). A club on a local network can run the
  server on a small PC in the venue, which avoids most internet problems.
- **KRA eTIMS** fiscal invoicing. If the club is VAT-registered, check current KRA rules early.
- Tips and cash payouts in the cash-up, a "no sale" drawer log, per-item refunds on closed sales.
- Customer accounts and credit, multiple branches, supplier purchases, CSV export, stock counts.
- Tickets are plain text lists (no modifiers like "no ice").

## Layout
- `shop/models.py` data model, `shop/services.py` checkout/void logic, `shop/mpesa.py` Daraja client,
  `shop/views.py` pages and JSON endpoints, `shop/static/shop/pos.js` the sales screen,
  `shop/tests.py` and `shop/tests_club.py` 63 tests (`python manage.py test`).
  `shop/views_club.py` tabs, stations, shifts, audit, PIN.
