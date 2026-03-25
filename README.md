# MobiTrack — نظام إدارة بزنس الموبايلات

A mobile phone shop management system built as a single-page Arabic RTL web app wrapped in an Android APK using Capacitor 8.

---

## Features

### Core Operations
- **المشتريات (Purchases)** — Log stock purchases with brand, model, quantity, unit price, supplier, payment method, and serial numbers (IMEI). Weighted-average cost is calculated automatically when buying the same model at different prices.
- **المبيعات (Sales)** — Log sales with model selection from live stock, customer name, sale price, payment method, and serial number selection (checkboxes show available IMEIs per model).
- **الاستوك (Stock)** — Live inventory view. Each model shows current quantity, average cost, total value, and individual serial number chips. Filterable by brand, supplier, and availability status.
- **المصاريف (Expenses)** — Log business expenses by category (rent, fuel, salaries, etc.) with amounts and payment method.

### Financial Management
- **الديون والتسوية (Debts & Settlement)** — Tracks deferred payments ("آجل") automatically from purchases and sales. Shows remaining balances per supplier and customer with progress bars. Settle debts with a payment entry that reduces the outstanding balance.
- **طرق الدفع (Payment Methods)** — Breakdown of transactions by payment type (cash, Instapay, Vodafone Cash, deferred).
- **التقارير (Reports)** — Full P&L: revenue, COGS, gross profit, expenses, net profit, units sold, top models, expense breakdown, and current stock valuation.
- **التقرير اليومي (Daily Report)** — Pick any date to see all sales, purchases, and expenses for that day with summary cards.

### Edit & Delete
All operations support full edit and delete:
- Edit purchase → adjusts stock via `rebuildStock()`
- Edit sale → re-computes stock
- Edit expense → updates totals
- Delete any record → stock and financials recompute cleanly

### Settings (Opening Balances for New Users)
New users can enter their situation before starting to use the app:
- **معلومات المحل** — Shop name and opening cash balance
- **الاستوك الافتتاحي** — Pre-existing inventory (model, brand, qty, avg cost). Added as the base layer in stock calculations before purchases.
- **الديون الافتتاحية** — Pre-existing debts owed to suppliers or receivable from customers. Show in the debts page tagged "افتتاحي" and count toward dashboard totals.

### Data & Backup
- **تصدير JSON** — Export all data as a JSON file for backup.
- **تصدير SQLite** — Export the raw `.db` file directly from the device.
- **استيراد JSON** — Import a previously exported JSON backup (replaces current data).

### Serial Numbers (IMEI Tracking)
- Serials are entered as line-separated values in the purchase form.
- When selling, available IMEIs for the selected model appear as checkboxes.
- Sold serials are removed from stock. Remaining serials show as chips in the stock view.
- Serials persist through edits and are stored as JSON arrays in the database.

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Single HTML file, Arabic RTL, Cairo font |
| Native wrapper | Capacitor 8.2.0 |
| Database | `@capacitor-community/sqlite` v8.0.1 (SQLite on Android) |
| Fallback storage | localStorage (browser/dev mode) |
| Build tool | Gradle + Android SDK API 34 |
| JDK | Amazon Corretto 21 |

---

## Project Structure

```
MobiTrack/
├── www/
│   └── index.html          # Entire app (HTML + CSS + JS, single file)
├── android/                # Capacitor Android project
│   ├── app/src/main/
│   │   ├── assets/public/  # Built web assets (synced from www/)
│   │   └── java/com/mobitrack/app/
│   │       └── MainActivity.java
│   ├── app/build.gradle
│   └── build.gradle
├── capacitor.config.json
└── package.json
```

---

## Database Schema

```sql
CREATE TABLE purchases (
  id INTEGER PRIMARY KEY,
  date TEXT, brand TEXT, model TEXT,
  qty INTEGER, unitPrice REAL,
  supplier TEXT, payment TEXT, notes TEXT,
  serials TEXT DEFAULT '[]'        -- JSON array of IMEI strings
);

CREATE TABLE sales (
  id INTEGER PRIMARY KEY,
  date TEXT, model TEXT,
  qty INTEGER, salePrice REAL, costPerUnit REAL,
  customer TEXT, payment TEXT,
  serials TEXT DEFAULT '[]'        -- JSON array of sold IMEIs
);

CREATE TABLE expenses (
  id INTEGER PRIMARY KEY,
  date TEXT, category TEXT, desc TEXT,
  amount REAL, payment TEXT
);

CREATE TABLE settlements (
  id INTEGER PRIMARY KEY,
  date TEXT, name TEXT, type TEXT,  -- type: 'owe' | 'owedme'
  amount REAL, payment TEXT, notes TEXT
);

CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT
);

CREATE TABLE opening_stock (
  id INTEGER PRIMARY KEY,
  model TEXT, brand TEXT,
  qty INTEGER, avgCost REAL
);

CREATE TABLE opening_debts (
  id INTEGER PRIMARY KEY,
  name TEXT, type TEXT,             -- type: 'owe' | 'owedme'
  amount REAL, notes TEXT
);
```

---

## Stock Calculation (`rebuildStock`)

Stock is always computed from scratch (never stored directly):

1. Apply **opening stock** entries as the base layer
2. Add all **purchases** (weighted-average cost recalculated per batch)
3. Subtract all **sales** (quantity and individual serials)

This means editing or deleting any purchase or sale immediately produces correct stock numbers. The function is called after every write operation.

---

## Building the APK

### Prerequisites
- **JDK 21** (Amazon Corretto recommended)
- **Android SDK** with API 34, build-tools 34.0.0, platform-tools
- `android/local.properties` pointing to your SDK:
  ```
  sdk.dir=D\:\\android-sdk
  ```

### Steps

```bash
# 1. Install dependencies
npm install

# 2. Sync web assets to Android project
npx cap sync android

# 3. Build release APK
cd android
./gradlew assembleRelease

# 4. Sign with debug keystore (for sideloading)
apksigner sign \
  --ks ~/.android/debug.keystore \
  --ks-pass pass:android \
  --ks-key-alias androiddebugkey \
  --key-pass pass:android \
  --out MobiTrack-release.apk \
  app/build/outputs/apk/release/app-release-unsigned.apk
```

The signed APK can be installed on any Android device via sideloading (enable "Install from unknown sources").

---

## App Details

| Field | Value |
|---|---|
| App ID | `com.mobitrack.app` |
| App Name | MobiTrack |
| Min SDK | Android 7.0 (API 24) |
| Target SDK | API 34 |
| Language | Arabic (RTL) |
| Orientation | Portrait + Landscape |
