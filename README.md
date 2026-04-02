# ERPNext ↔ OpenCart Product Sync System

A Python-based integration system that synchronizes products from ERPNext to OpenCart. Supports **bulk syncing** (all products at once) and **real-time webhook-based syncing** (add / update / delete individual products triggered by ERPNext events).

Communication with the OpenCart database happens through a lightweight PHP bridge file (`db_bridge.php`) uploaded to your OpenCart server. The Python app sends SQL queries to the PHP bridge via HTTP, which executes them on the OpenCart MySQL database.

---

## Table of Contents

- [Features](#features)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [ERPNext Webhook Setup](#erpnext-webhook-setup)
- [API Endpoints](#api-endpoints)
- [How It Works](#how-it-works)
- [Important Notes](#important-notes)
- [Troubleshooting](#troubleshooting)

---

## Features

- **Automated image sync** — Automatically downloads product images from ERPNext and uploads them to your OpenCart server via the PHP bridge.
- **Telegram notifications** — Get instant logs and success/error reports directly to your Telegram account.
- **Retry mechanism** — Each operation retries up to 4 times on failure for maximum reliability.
- **Logging** — All operations logged to `logger.log` and console with colored output.

---

## Architecture Overview

```
┌─────────────┐       HTTP/API        ┌──────────────────┐
│   ERPNext    │ ───────────────────►  │  Python Sync App │
│   (Source)   │                       │  (This Project)  │
└─────────────┘                       └────────┬─────────┘
                                               │
                                        HTTP POST (SQL)
                                               │
                                               ▼
                                      ┌─────────────────┐
                                      │  db_bridge.php   │
                                      │ (On OpenCart Web  │
                                      │      Server)     │
                                      └────────┬────────┘
                                               │
                                         MySQL Queries
                                               │
                                               ▼
                                      ┌─────────────────┐
                                      │  OpenCart MySQL   │
                                      │    Database       │
                                      └─────────────────┘
```

---

## Project Structure

```
ERP_Opencart_sync_system/
│
├── .env                          # Environment variables (API keys, URLs, secrets)
├── main.py                       # Starts the FastAPI webhook server (real-time sync)
├── initiate_sync.py              # Runs bulk sync for all configured categories
├── main_manager.py               # Orchestrates add / update / delete operations
├── requirements.txt              # Python dependencies
├── db_bridge.php                 # PHP bridge file (upload to your OpenCart server)
│
├── app/
│   ├── config/
│   │   ├── settings.py           # ERP stores, OpenCart languages, tax & stock settings
│   │   └── category_configs.py   # ERPNext → OpenCart category name mapping
│   │
│   ├── controller/
│   │   ├── endpoints.py          # FastAPI routes for webhook-based sync
│   │   └── schema.py             # Pydantic request schemas
│   │
│   ├── mapper/
│   │   └── erpnext_to_opencart.py  # Maps ERPNext item fields to OpenCart product format
│   │
│   ├── services/
│   │   ├── ERPNext_services/
│   │   │   └── item_service.py     # Fetches items, prices, quantities from ERPNext API
│   │   │
│   │   └── opencart_services/
│   │       ├── product_addition.py   # Inserts new products into OpenCart
│   │       ├── prodcut_updating.py   # Updates existing products in OpenCart
│   │       ├── product_deleting.py   # Deletes products from OpenCart
│   │       ├── category_addition.py  # Links categories to products
│   │       └── category_updating.py  # Updates category-product links
│   │
│   └── utils/
│       ├── db_utils.py           # Sends SQL queries to the PHP bridge
│       ├── image_sync.py         # Downloads images and uploads via PHP bridge
│       ├── telegram_notifier.py  # Sends logs and alerts to Telegram
│       └── logger.py             # Logging configuration
```

---

## Prerequisites

- Python 3.10+
- ERPNext instance with API access enabled
- OpenCart store with admin access (to find language IDs, tax class IDs, stock status IDs)
- Server access (FTP/SSH) to upload the PHP bridge file
- PHP + MySQL on the OpenCart server (already present if OpenCart is running)

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/your-username/ERP_Opencart_sync_system.git
cd ERP_Opencart_sync_system
```

### 2. Create a Virtual Environment

```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Linux / macOS
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Set Up the PHP Bridge on Your OpenCart Server

The `db_bridge.php` file acts as a secure middleman between this Python app and your OpenCart MySQL database.

1. Open `db_bridge.php` and fill in your OpenCart database credentials:

```php
define('SECRET', 'your_strong_secret_key_here');

$host     = 'localhost';
$port     = 3306;
$user     = 'your_opencart_db_username';
$password = 'your_opencart_db_password';
$database = 'your_opencart_db_name';
```

> **Tip:** You can find the database credentials in your OpenCart's `config.php` file on the server.

2. Upload `db_bridge.php` to the **root directory** of your OpenCart installation (e.g., `public_html/db_bridge.php`).

3. Test the bridge by visiting `https://your-opencart-domain.com/db_bridge.php` — you should see `{"error":"Forbidden"}` (this means it's working and rejecting unauthenticated requests).

---

## Configuration

### 1. Environment Variables (`.env`)

Create a `.env` file in the project root (a template already exists). Fill in all variables:

```env
# API Secret Key (Webhook Authentication)
API_SECRET_KEY="your_api_secret_key_here"

# ERPNext Configuration
ERP_BASE_URL=https://your-erpnext-site.com/api/resource
ERP_API_KEY="your_erp_api_key"
ERP_API_SECRET="your_erp_api_secret"

# OpenCart Bridge Configuration
BRIDGE_URL="https://your-opencart-domain.com/db_bridge.php"
SECRET="your_bridge_secret_key_here"
DB_PREFIX="oc_"
```

| Variable | Description | Example |
|---|---|---|
| `API_SECRET_KEY` | Secret key to authenticate webhook requests. ERPNext sends this in the `Authorization` header. | `"mY_s3cret_w3bhook_key"` |
| `ERP_BASE_URL` | ERPNext API base URL. Must end with `/api/resource`. | `https://myerp.frappe.cloud/api/resource` |
| `ERP_API_KEY` | API key from ERPNext (Settings → API Access). | `"8d4a583c6091756"` |
| `ERP_API_SECRET` | API secret from ERPNext (paired with the API key). | `"8549b3a0f8f8c7a"` |
| `BRIDGE_URL` | Full URL to `db_bridge.php` on your OpenCart server. | `"https://myshop.com/db_bridge.php"` |
| `SECRET` | Must exactly match the `SECRET` inside `db_bridge.php`. | `"mYs3cr3tKey_ch4ngeMe"` |
| `DB_PREFIX` | OpenCart database table prefix. Usually `oc_`. | `"oc_"` |

### 2. Telegram Configuration (`.env`)

To receive logs and error reports directly on your Telegram, add these variables to your `.env` file. You can get your `API_ID` and `API_HASH` from [my.telegram.org](https://my.telegram.org/).

| Variable | Description | Example |
|---|---|---|
| `TELEGRAM_API_ID` | Your Telegram API ID from my.telegram.org. | `24958371` |
| `TELEGRAM_API_HASH` | Your Telegram API Hash from my.telegram.org. | `"abc123d4e5f6g7h8i9j"` |
| `TELEGRAM_PHONE_NUMBER`| The phone number associated with your Telegram. | `"+1234567890"` |
| `TELEGRAM_SESSION_NAME`| The name of the session file (stored in project root).| `"log_session"` |
| `TELEGRAM_RECEIPT_EMAIL`| The Telegram username or phone number to send logs to. | `"@MyUsername"` |

> **Note:** The first time you run the system, it will ask you for a **login code** in the terminal to authorize the Telegram session. This only happens once.

### 3. ERP & OpenCart Settings (`app/config/settings.py`)

#### ERPNext Store Settings

```python
erp_settig = {
    'store': ['Store Name 1 - ABBR', 'Store Name 2 - ABBR']
}
```

- `store` — List of ERPNext warehouse names to calculate stock quantity.
- Formula: `Available Qty = SUM(actual_qty - reserved_qty)` across all listed warehouses.
- Find names at: ERPNext → Stock → Warehouse (include abbreviation, e.g., `"Stores - MGD"`).

#### OpenCart Settings

```python
opencart_settings = {
    'languages': [1, 2],
    "tax_class_id": "9",
    "stock_status_id": "7",
}
```

| Setting | Description | How to Find |
|---|---|---|
| `languages` | Language IDs in your OpenCart store. Descriptions inserted for each. | Admin → System → Localisation → Languages |
| `tax_class_id` | Tax class for synced products. `"0"` = no tax. | Admin → System → Localisation → Taxes → Tax Classes |
| `stock_status_id` | Display label when qty = 0 (cosmetic only). | Admin → System → Localisation → Stock Statuses |

Default stock status IDs: `5` = In Stock, `6` = 2-3 Days, `7` = Out of Stock, `8` = Pre-Order. Verify in your admin panel.

### 4. Category Mapping (`app/config/category_configs.py`)

Defines which ERPNext item groups are synced and maps them to OpenCart category names.

```python
categoris_names_convertion = {
    'ERPNext Category Name': 'OpenCart Category Name',
}
```

- **Key** = Exact Item Group name in ERPNext
- **Value** = Exact Category name in OpenCart (any language)
- Only listed categories are synced; unlisted ones are skipped

> ⚠️ **Important:** Categories must already exist in OpenCart (the system does NOT create them). Names must match exactly (case-sensitive).

**Example:**

```python
categoris_names_convertion = {
    'JVC'                   : 'JVC',
    'Magic Unbreakable'     : 'Magic Unbreakable',
    'Electronics - Home'    : 'Home Electronics',      # different names OK
    'Air Conditioners'      : 'تكييفات',               # Arabic name OK
}
```

---

## Usage

### Bulk Sync — Sync All Products

Fetches ALL products from every category in `category_configs.py` and syncs them to OpenCart. Use for initial import or full re-sync.

```bash
python initiate_sync.py
```

The script iterates through each mapped category, fetches item data/price/quantity from ERPNext, and adds products to OpenCart (including SEO URL and category linking). Retries up to 4 times on failure.

Console output: 🔵 Cyan = Fetching, 🟢 Green = Success, 🔴 Red = Error, 🟣 Magenta = Timing.

### Real-Time Sync — Webhook API Server

Starts a FastAPI server that listens for ERPNext webhook requests.

```bash
Server starts on `http://localhost:8000`. Then configure ERPNext webhooks (see [ERPNext Webhook Setup](#erpnext-webhook-setup) below).

> **Tip:** If you are running this on a remote server, use `nohup` to keep the process running after you close the terminal:
> ```bash
> nohup python main.py > server.log 2>&1 &
> ```

---

## ERPNext Webhook Setup

You need to create **7 webhooks** in ERPNext to enable real-time sync.
Go to: **Home → Integrations → Webhook** and create each one as described below.

### Common Settings for ALL Webhooks

All 7 webhooks share these **headers**:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Authorization` | `Your_api_secret_key` |

> Replace `Your_api_secret_key` with the actual `API_SECRET_KEY` from your `.env` file.
> Replace `{your-server}` in the URLs below with your actual server IP/domain and port.

---

### Webhook 1 — Insert Item (Add Product)

| Setting | Value |
|---|---|
| **DocType** | `Item` |
| **Event** | After Insert |
| **Request URL** | `http://{your-server}:8000/sync/add-product` |
| **Request Method** | POST |
| **Request Structure** | JSON |

**Request Body:**
```json
{
  "data": {{ doc | tojson | safe }}
}
```

---

### Webhook 2 — Update Item (Update Product)

| Setting | Value |
|---|---|
| **DocType** | `Item` |
| **Event** | on Update |
| **Request URL** | `http://{your-server}:8000/sync/update-product` |
| **Request Method** | PUT |
| **Request Structure** | JSON |

**Request Body:**
```json
{
  "data": {{ doc | tojson | safe }}
}
```

---

### Webhook 3 — Delete Item (Delete Product)

| Setting | Value |
|---|---|
| **DocType** | `Item` |
| **Event** | on Trash |
| **Request URL** | `http://{your-server}:8000/sync/delete-product` |
| **Request Method** | DELETE |
| **Request Structure** | JSON |

**Request Body:**
```json
{
  "data": {{ doc | tojson | safe }}
}
```

---

### Webhook 4 — Update Price

| Setting | Value |
|---|---|
| **DocType** | `Item Price` |
| **Event** | on Update |
| **Request URL** | `http://{your-server}:8000/sync/update-product-price` |
| **Request Method** | PUT |

**Webhook Data (Fields):**

| Fieldname | Key |
|---|---|
| `price_list_rate` | `price_list_rate` |
| `price_list` | `price_list` |
| `item_code` | `item_code` |

---

### Webhook 5 — Update Quantity (Stock Ledger Entry)

| Setting | Value |
|---|---|
| **DocType** | `Stock Ledger Entry` |
| **Event** | After Insert |
| **Request URL** | `http://{your-server}:8000/sync/update-product-quantity-sle` |
| **Request Method** | PUT |

**Webhook Data (Fields):**

| Fieldname | Key |
|---|---|
| `item_code` | `item_code` |

---

### Webhook 6 — Update Quantity on Sales Order Submit

| Setting | Value |
|---|---|
| **DocType** | `Sales Order` |
| **Event** | on Submit |
| **Request URL** | `http://{your-server}:8000/sync/update-product-quantity-so` |
| **Request Method** | PUT |

**Webhook Data (Fields):**

| Fieldname | Key |
|---|---|
| `items` | `items` |

---

### Webhook 7 — Update Quantity on Sales Order Cancel

| Setting | Value |
|---|---|
| **DocType** | `Sales Order` |
| **Event** | on Cancel |
| **Request URL** | `http://{your-server}:8000/sync/update-product-quantity-so` |
| **Request Method** | PUT |

**Webhook Data (Fields):**

| Fieldname | Key |
|---|---|
| `items` | `items` |

---

## API Endpoints

All endpoints require an `Authorization` header matching your `API_SECRET_KEY`.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/sync/add-product` | Add a new product to OpenCart |
| `PUT` | `/sync/update-product` | Update an existing product in OpenCart |
| `DELETE` | `/sync/delete-product` | Delete a product from OpenCart |
| `PUT` | `/sync/update-product-price` | Update only the price of a product |
| `PUT` | `/sync/update-product-quantity-sle` | Update product quantity (Stock Ledger Entry) |
| `PUT` | `/sync/update-product-quantity-so` | Update quantity for items in a Sales Order |
| `GET` | `/` | Test endpoint (verify server is running) |

---

## How It Works

1. **Data Fetching** — The Item service fetches item data, prices (from "Standard Selling" price list), and available quantities from ERPNext REST API.

2. **Data Mapping** — The ERPToOpencart mapper transforms ERPNext fields into OpenCart product format, including SEO-friendly URL slugs.

3. **Database Operations** — OpenCart services build SQL queries and send them to `db_bridge.php` via HTTP POST. The bridge executes them on the OpenCart MySQL database.

4. **Category Linking** — The system maps ERPNext item groups to OpenCart categories via `categoris_names_convertion` and inserts the product-category relationship.

---

## Important Notes

- ⚠️ Do not change file paths within the project — imports depend on the directory structure.
- ⚠️ The `.env` file must be in the project root (same level as `main.py`).
- ⚠️ Categories must exist in OpenCart before syncing — the system does NOT create them.
- ⚠️ The `SECRET` in `.env` must exactly match the `SECRET` in `db_bridge.php`.
- ⚠️ The `image/` directory on your OpenCart server must be **writable** (permissions `755` or `777`).
- ⚠️ Secure your `db_bridge.php` — it executes raw SQL and handles file uploads. Use a strong secret.
- Price synced is from the "Standard Selling" price list in ERPNext.
- Product SKU in OpenCart = Item Code in ERPNext (unique identifier to match products).

---

## Technical Details: Image Syncing

The system automatically detects product images in ERPNext. Here is how the sync works:
1.  **Download**: The Python app downloads the image from ERPNext (or an external URL).
2.  **Upload**: The image is sent via the `db_bridge.php` to your OpenCart server.
3.  **Storage**: Images are saved in the `image/catalog/erp_sync/` directory.
4.  **Database**: The product record in OpenCart is updated with the path `catalog/erp_sync/filename.jpg`.

> **Note:** If an image with the same name already exists in the `erp_sync` folder, it will be overwritten.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| "DB connection failed" error | Verify database credentials in `db_bridge.php` match your OpenCart `config.php`. |
| "Forbidden" error from bridge | `SECRET` in `.env` must exactly match `SECRET` in `db_bridge.php`. |
| 401 / 403 on webhook endpoints | Ensure ERPNext sends the correct `API_SECRET_KEY` in the `Authorization` header. |
| Products not syncing for a category | Verify category is listed in `category_configs.py` with exact ERPNext item group name. |
| Quantity showing 0 | Check warehouse names in `erp_settig['store']` match ERPNext exactly (incl. abbreviation). |
| Price showing 0 | Ensure item has a price entry in the "Standard Selling" price list in ERPNext. |
| "No item codes found" warning | Item group name in `category_configs.py` doesn't match ERPNext. Check spelling/case. |
| Connection timeout errors | System retries up to 4 times. Check network connectivity to ERPNext and OpenCart. |

---

## License

This project is provided as-is for personal and commercial use.
