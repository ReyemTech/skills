---
name: qbo-cleanup
description: Audit QBO expenses for duplicates, missing FX rates, and unassigned vendors. Supports 90-day or full fiscal year mode.
disable-model-invocation: true
compatibility: Requires network access and valid Maton API key
allowed-tools: Bash(*), Read, Grep
metadata:
  author: reyemtech
  version: "2.0"
  clawdbot:
    emoji: "\U0001F9F9"
    requires:
      env:
        - MATON_API_KEY
---

# QBO Cleanup

Audit expense transactions for duplicates (with smart merge), missing/incorrect foreign exchange rates, and missing vendor assignments. Designed for QBO instances with a home currency and foreign currency transactions.

**Configuration:** Set your home currency (e.g., CAD) and foreign currency (e.g., USD) in the instructions below. The FX rate source defaults to Bank of Canada (FXUSDCAD).

**Two modes:**
- `/qbo-cleanup` (no args) -- **Regular mode** -- last 90 days for ongoing hygiene
- `/qbo-cleanup <year>` -- **Fiscal year mode** -- full year (e.g., FY2025 = Dec 1 2024 - Nov 30 2025)

## Instructions

When the user invokes `/qbo-cleanup [year]`, execute the following phases in order. Use `python3 <<'EOF'` blocks via Bash for all API calls and data processing.

**Execution style:** Present each item to the user for a decision (merge/delete/skip/override), but once they decide, execute the bash immediately -- do not ask permission to run the script. Source `MATON_API_KEY` from `.env` for every bash call: `export $(grep MATON_API_KEY .env) && python3 <<'EOF'`

---

## Phase 1: Setup

1. Parse the argument:
   - **No argument** -- regular mode, last 90 days from today
   - **Number (e.g. 2025)** -- fiscal year mode
2. Calculate the date range:
   - **Regular:** `today - 90 days` to `today`
   - **Fiscal:** `{year-1}-12-01` to `{year}-11-30`
3. Verify the QBO connection:

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json, datetime

# --- Parse mode ---
# Replace ARGS with actual argument or empty string
ARGS = ""  # e.g., "2025" or ""

today = datetime.date.today()

if ARGS.strip().isdigit():
    year = int(ARGS.strip())
    mode = "fiscal"
    start_date = f"{year - 1}-12-01"
    end_date = f"{year}-11-30"
    label = f"FY{year}"
else:
    mode = "regular"
    start_date = (today - datetime.timedelta(days=90)).isoformat()
    end_date = today.isoformat()
    label = "90-day"

print(f"Mode: {label}")
print(f"Date range: {start_date} to {end_date}")

# --- Verify QBO connection ---
req = urllib.request.Request('https://ctrl.maton.ai/connections?app=quickbooks&status=ACTIVE')
req.add_header('Authorization', f'Bearer {os.environ["MATON_API_KEY"]}')
result = json.load(urllib.request.urlopen(req))
connections = result.get('connections', [])
if not connections:
    print("ERROR: No active QuickBooks connection found. Visit https://ctrl.maton.ai to connect.")
else:
    print(f"Connected: {len(connections)} active QBO connection(s)")
    for c in connections:
        print(f"  - {c['connection_id']} (last updated: {c.get('last_updated_time', 'unknown')})")

print(f"\nStarting QBO cleanup ({label}: {start_date} to {end_date})")
EOF
```

---

## Phase 2: Data Collection

### 2a. Fetch Expense Transactions

Query **all 5 entity types** with pagination. For each type, use STARTPOSITION-based pagination (1-based, increment by 1000, stop when fewer than 1000 returned).

**Entity types:** `Purchase`, `Bill`, `BillPayment`, `VendorCredit`, `JournalEntry`

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json, time, sys

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"
# Replace these with the actual dates from Phase 1
START_DATE = "YYYY-MM-DD"
END_DATE = "YYYY-MM-DD"

ENTITY_TYPES = ["Purchase", "Bill", "BillPayment", "VendorCredit", "JournalEntry"]
PAGE_SIZE = 1000

all_transactions = {}

for entity in ENTITY_TYPES:
    all_transactions[entity] = []
    start_pos = 1
    while True:
        query = (
            f"SELECT * FROM {entity} "
            f"WHERE TxnDate >= '{START_DATE}' AND TxnDate <= '{END_DATE}' "
            f"STARTPOSITION {start_pos} MAXRESULTS {PAGE_SIZE}"
        )
        url = f"{BASE}/query?query={urllib.parse.quote(query)}"
        req = urllib.request.Request(url)
        req.add_header('Authorization', f'Bearer {API_KEY}')

        try:
            resp = json.load(urllib.request.urlopen(req))
        except Exception as e:
            print(f"  ERROR querying {entity}: {e}", file=sys.stderr)
            break

        qr = resp.get("QueryResponse", {})
        rows = qr.get(entity, [])
        all_transactions[entity].extend(rows)
        print(f"  {entity}: fetched {len(rows)} (pos {start_pos})")

        if len(rows) < PAGE_SIZE:
            break
        start_pos += PAGE_SIZE
        time.sleep(0.15)

    print(f"  {entity} total: {len(all_transactions[entity])}")

# Save to temp file for subsequent phases
with open("/tmp/qbo_cleanup_transactions.json", "w") as f:
    json.dump(all_transactions, f)

total = sum(len(v) for v in all_transactions.values())
print(f"\nSaved all transactions to /tmp/qbo_cleanup_transactions.json")
print(f"Total transactions fetched: {total}")
EOF
```

### 2b. Fetch FX Rates

Fetch exchange rates from your FX rate source. The example below uses Bank of Canada (FXUSDCAD). Adjust the URL and currency pair for your setup.

```bash
python3 <<'EOF'
import urllib.request, json, datetime

# Replace with actual dates from Phase 1
START_DATE = "YYYY-MM-DD"
END_DATE = "YYYY-MM-DD"

url = (
    f"https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json"
    f"?start_date={START_DATE}&end_date={END_DATE}"
)
resp = json.load(urllib.request.urlopen(url))
observations = resp.get("observations", [])

# Build date -> rate lookup
fx_rates = {}
for obs in observations:
    date = obs["d"]
    rate_val = obs["FXUSDCAD"].get("v")
    if rate_val:
        fx_rates[date] = float(rate_val)

print(f"Loaded {len(fx_rates)} FX rates from {START_DATE} to {END_DATE}")

# Save rates
with open("/tmp/qbo_cleanup_fx_rates.json", "w") as f:
    json.dump(fx_rates, f)
print("Saved FX rates to /tmp/qbo_cleanup_fx_rates.json")

# Spot check
sample_dates = list(fx_rates.items())[:3]
for d, r in sample_dates:
    print(f"  {d}: 1 USD = {r} CAD")
EOF
```

### 2c. Fetch Attachments

Query attachments to know which transactions have files attached (used for keeper selection in duplicate merging).

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json, time

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"
# Replace with actual start date from Phase 1
START_DATE = "YYYY-MM-DD"
PAGE_SIZE = 1000

attachments = {}  # {(entity_type, entity_id): [filenames]}
start_pos = 1
total_fetched = 0

while True:
    query = (
        f"SELECT * FROM Attachable "
        f"WHERE MetaData.LastUpdatedTime >= '{START_DATE}' "
        f"STARTPOSITION {start_pos} MAXRESULTS {PAGE_SIZE}"
    )
    url = f"{BASE}/query?query={urllib.parse.quote(query)}"
    req = urllib.request.Request(url)
    req.add_header('Authorization', f'Bearer {API_KEY}')

    try:
        resp = json.load(urllib.request.urlopen(req))
    except Exception as e:
        print(f"  ERROR querying Attachable: {e}")
        break

    qr = resp.get("QueryResponse", {})
    rows = qr.get("Attachable", [])
    total_fetched += len(rows)
    print(f"  Attachable: fetched {len(rows)} (pos {start_pos})")

    for att in rows:
        filename = att.get("FileName", "unknown")
        refs = att.get("AttachableRef", [])
        for ref in refs:
            etype = ref.get("EntityRef", {}).get("type", "")
            eid = ref.get("EntityRef", {}).get("value", "")
            if etype and eid:
                key = f"{etype}:{eid}"
                if key not in attachments:
                    attachments[key] = []
                attachments[key].append(filename)

    if len(rows) < PAGE_SIZE:
        break
    start_pos += PAGE_SIZE
    time.sleep(0.15)

with open("/tmp/qbo_cleanup_attachments.json", "w") as f:
    json.dump(attachments, f)

print(f"\nTotal attachables fetched: {total_fetched}")
print(f"Transactions with attachments: {len(attachments)}")
EOF
```

### 2d. Fetch All Vendors

Fetch the full vendor list for fuzzy matching in Phase 5 (missing vendor detection).

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json, time

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"
PAGE_SIZE = 1000

vendors = {}  # {display_name_lower: {id, name}}
all_vendors_raw = []
start_pos = 1

while True:
    query = f"SELECT * FROM Vendor STARTPOSITION {start_pos} MAXRESULTS {PAGE_SIZE}"
    url = f"{BASE}/query?query={urllib.parse.quote(query)}"
    req = urllib.request.Request(url)
    req.add_header('Authorization', f'Bearer {API_KEY}')

    try:
        resp = json.load(urllib.request.urlopen(req))
    except Exception as e:
        print(f"  ERROR querying Vendor: {e}")
        break

    qr = resp.get("QueryResponse", {})
    rows = qr.get("Vendor", [])
    all_vendors_raw.extend(rows)
    print(f"  Vendor: fetched {len(rows)} (pos {start_pos})")

    for v in rows:
        name = v.get("DisplayName", "")
        vid = v.get("Id", "")
        if name:
            vendors[name.lower()] = {"id": vid, "name": name}

    if len(rows) < PAGE_SIZE:
        break
    start_pos += PAGE_SIZE
    time.sleep(0.15)

with open("/tmp/qbo_cleanup_vendors.json", "w") as f:
    json.dump(vendors, f)

print(f"\nTotal vendors fetched: {len(vendors)}")
for name_lower, info in list(vendors.items())[:5]:
    print(f"  {info['name']} (#{info['id']})")
EOF
```

---

## Phase 3: Duplicate Detection -- Smart Merge

Load the transactions from Phase 2 and detect duplicates, then merge data from donor into keeper before deleting.

**Duplicate criteria -- ALL must match:**
- Same entity type
- Same vendor (`VendorRef.value` or `EntityRef.value`) -- including both-blank as a match
- Same total amount (`TotalAmt`)
- Same date OR within 2 calendar days

### 3a. Detect Duplicates

```bash
python3 <<'EOF'
import json, datetime
from collections import defaultdict

with open("/tmp/qbo_cleanup_transactions.json") as f:
    all_txns = json.load(f)
with open("/tmp/qbo_cleanup_attachments.json") as f:
    attachments = json.load(f)

def is_reconciled(txn):
    """Check if transaction is reconciled -- NEVER modify these."""
    for line in txn.get("Line", []):
        detail = line.get("AccountBasedExpenseLineDetail", {})
        if detail.get("BillableStatus") == "Reconciled":
            return True
    # Also check LinkedTxn for bank-feed reconciliation
    if txn.get("LinkedTxn"):
        return True
    return False

def get_vendor(txn):
    for key in ("VendorRef", "EntityRef"):
        ref = txn.get(key)
        if ref:
            return ref.get("value", "")
    return ""

def get_vendor_name(txn):
    for key in ("VendorRef", "EntityRef"):
        ref = txn.get(key)
        if ref:
            return ref.get("name", ref.get("value", "unknown"))
    return "(none)"

def has_attachment(entity_type, entity_id):
    key = f"{entity_type}:{entity_id}"
    return key in attachments

def get_account(txn):
    """Get the primary account from line items."""
    lines = txn.get("Line", [])
    for line in lines:
        detail = line.get("AccountBasedExpenseLineDetail", {})
        acct = detail.get("AccountRef", {})
        if acct.get("name"):
            return acct["name"]
        # Also check JournalEntry lines
        je_detail = line.get("JournalEntryLineDetail", {})
        acct = je_detail.get("AccountRef", {})
        if acct.get("name"):
            return acct["name"]
    return ""

def get_created(txn):
    """Get creation date from MetaData."""
    meta = txn.get("MetaData", {})
    created = meta.get("CreateTime", "")
    if created:
        return created[:10]  # just the date part
    return ""

duplicate_groups = []

for entity_type, txns in all_txns.items():
    buckets = defaultdict(list)
    for txn in txns:
        if is_reconciled(txn):
            continue  # NEVER touch reconciled transactions
        if "DUPOK" in (txn.get("PrivateNote", "") or ""):
            continue  # Previously reviewed and marked as not-a-duplicate
        vendor = get_vendor(txn)
        amount = str(txn.get("TotalAmt", ""))
        key = (vendor, amount)
        buckets[key].append(txn)

    for key, group in buckets.items():
        if len(group) < 2:
            continue
        sorted_group = sorted(group, key=lambda t: t.get("TxnDate", ""))
        clusters = []
        current_cluster = [sorted_group[0]]

        for txn in sorted_group[1:]:
            prev_date = datetime.date.fromisoformat(current_cluster[-1].get("TxnDate", "2000-01-01"))
            curr_date = datetime.date.fromisoformat(txn.get("TxnDate", "2000-01-01"))
            if abs((curr_date - prev_date).days) <= 2:
                current_cluster.append(txn)
            else:
                if len(current_cluster) >= 2:
                    clusters.append(current_cluster)
                current_cluster = [txn]

        if len(current_cluster) >= 2:
            clusters.append(current_cluster)

        for cluster in clusters:
            # Pick recommended keeper: auto-imported wins, then attachments, then earliest created
            def keeper_score(t):
                """Higher score = better keeper candidate."""
                score = 0
                # Auto-imported transactions get highest priority
                if (t.get("PrivateNote") or "").startswith("Auto-imported: "):
                    score += 100
                # Attachments get second priority
                if has_attachment(entity_type, t.get("Id", "")):
                    score += 10
                # Earlier created as tiebreaker (negate so earlier = higher)
                return (score, -len(get_created(t) or "9999"))

            best_idx = 0
            for i, t in enumerate(cluster):
                if keeper_score(t) > keeper_score(cluster[best_idx]):
                    best_idx = i

            duplicate_groups.append({
                "entity_type": entity_type,
                "recommended_keeper_idx": best_idx,
                "transactions": [
                    {
                        "Id": t.get("Id"),
                        "SyncToken": t.get("SyncToken"),
                        "TxnDate": t.get("TxnDate"),
                        "TotalAmt": t.get("TotalAmt"),
                        "CurrencyRef": t.get("CurrencyRef", {}).get("value", "HOME"),
                        "VendorName": get_vendor_name(t),
                        "VendorRef": t.get("VendorRef"),
                        "EntityRef": t.get("EntityRef"),
                        "DocNumber": t.get("DocNumber", ""),
                        "PrivateNote": t.get("PrivateNote", ""),
                        "ExchangeRate": t.get("ExchangeRate"),
                        "PaymentType": t.get("PaymentType"),
                        "Account": get_account(t),
                        "HasAttachment": has_attachment(entity_type, t.get("Id", "")),
                        "IsAutoImported": (t.get("PrivateNote") or "").startswith("Auto-imported: "),
                        "Created": get_created(t),
                        "Line": t.get("Line", []),
                    }
                    for t in cluster
                ]
            })

with open("/tmp/qbo_cleanup_duplicates.json", "w") as f:
    json.dump(duplicate_groups, f, indent=2)

print(f"Found {len(duplicate_groups)} potential duplicate group(s)\n")
for i, grp in enumerate(duplicate_groups):
    keeper_idx = grp["recommended_keeper_idx"]
    print(f"Group {i+1} ({grp['entity_type']}):")
    print(f"  {'ID':<10} {'Date':<12} {'Vendor':<20} {'Amount':>10} {'Curr':<5} {'DocNum':<25} {'Account':<25} {'Attach':<6} {'Auto':<4} {'Created':<10}")
    for j, t in enumerate(grp["transactions"]):
        att = "Y" if t["HasAttachment"] else ""
        auto = "Y" if t.get("IsAutoImported") else ""
        marker = " <- keeper" if j == keeper_idx else ""
        print(f"  {t['Id']:<10} {t['TxnDate']:<12} {t['VendorName']:<20} {t['TotalAmt']:>10} {t['CurrencyRef']:<5} {t['DocNumber']:<25} {t['Account']:<25} {att:<6} {auto:<4} {t['Created']:<10}{marker}")
    print()
EOF
```

### 3b. Merge-then-Delete

Present each duplicate group to the user one at a time. Once they decide, execute immediately:
1. **Merge + delete** -- merge donor data into keeper, then delete donor (recommended)
2. **Keep all** -- append "DUPOK" to `PrivateNote` on all transactions in the group (so they're excluded from future runs)
3. **Override keeper** -- user picks a different keeper
4. **Skip** -- move to next group (does NOT tag -- group will reappear next run)

**DUPOK tagging:** When user says "keep all", do a sparse update on each transaction to append " DUPOK" to `PrivateNote` (or set it to "DUPOK" if blank). Always include `PaymentType` for Purchase updates. This prevents the group from being flagged again on subsequent runs.

**Merge logic** -- copy from donor to keeper (only fill blanks, never overwrite existing):
- `EntityRef`/`VendorRef` -- if keeper has none, take donor's
- `DocNumber` -- if keeper is blank, take donor's
- `PrivateNote` -- if keeper is blank, take donor's; if both have notes, append donor's note
- Line-level `AccountRef` -- if keeper's account is "Uncategorized Expense", take donor's categorized account
- `ExchangeRate` -- if keeper's is missing/1.0 and donor has a real rate, take donor's

**Show proposed merge to user before executing:**
```
Merge: copy VendorRef=Stripe, DocNumber=xyz from #1234 -> #5678, then delete #1234
```

**To execute merge + delete:**

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json, time

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"

ENTITY_TYPE = "Purchase"   # replace with actual type
KEEPER_ID = "5678"         # replace with keeper ID
DONOR_ID = "1234"          # replace with donor ID

# --- Merge fields to apply to keeper (computed from donor) ---
# Only include fields that need updating. Example:
MERGE_FIELDS = {
    # "DocNumber": "ch_3SPTXh...",
    # "VendorRef": {"value": "123", "name": "Stripe"},
    # "PrivateNote": "Appended note from donor",
    # "ExchangeRate": 1.4390,
}

# Step 1: Fetch fresh keeper
url = f"{BASE}/{ENTITY_TYPE.lower()}/{KEEPER_ID}"
req = urllib.request.Request(url)
req.add_header('Authorization', f'Bearer {API_KEY}')
resp = json.load(urllib.request.urlopen(req))
keeper_data = resp.get(ENTITY_TYPE, resp)

# Step 2: Apply merge fields (sparse update)
if MERGE_FIELDS:
    update_payload = {
        "Id": keeper_data["Id"],
        "SyncToken": keeper_data["SyncToken"],
        "sparse": True,
    }
    # Always include PaymentType for Purchase sparse updates
    if ENTITY_TYPE == "Purchase" and keeper_data.get("PaymentType"):
        update_payload["PaymentType"] = keeper_data["PaymentType"]

    update_payload.update(MERGE_FIELDS)

    update_url = f"{BASE}/{ENTITY_TYPE.lower()}"
    data = json.dumps(update_payload).encode()
    req = urllib.request.Request(update_url, data=data, method='POST')
    req.add_header('Authorization', f'Bearer {API_KEY}')
    req.add_header('Content-Type', 'application/json')

    try:
        result = json.load(urllib.request.urlopen(req))
        updated = result.get(ENTITY_TYPE, result)
        print(f"Merged fields into {ENTITY_TYPE} #{KEEPER_ID} (SyncToken: {updated.get('SyncToken')})")
    except urllib.error.HTTPError as e:
        error_body = e.read().decode()
        print(f"ERROR merging into {ENTITY_TYPE} #{KEEPER_ID}: {e.code} {error_body}")
        # Stop -- do not delete donor if merge failed
        raise SystemExit(1)

time.sleep(0.15)

# Step 3: Fetch fresh donor and delete
url = f"{BASE}/{ENTITY_TYPE.lower()}/{DONOR_ID}"
req = urllib.request.Request(url)
req.add_header('Authorization', f'Bearer {API_KEY}')
donor = json.load(urllib.request.urlopen(req))
donor_data = donor.get(ENTITY_TYPE, donor)

delete_url = f"{BASE}/{ENTITY_TYPE.lower()}?operation=delete"
data = json.dumps({"Id": donor_data["Id"], "SyncToken": donor_data["SyncToken"]}).encode()
req = urllib.request.Request(delete_url, data=data, method='POST')
req.add_header('Authorization', f'Bearer {API_KEY}')
req.add_header('Content-Type', 'application/json')

try:
    result = json.load(urllib.request.urlopen(req))
    print(f"Deleted {ENTITY_TYPE} #{DONOR_ID}")
except urllib.error.HTTPError as e:
    error_body = e.read().decode()
    if "6480" in error_body:
        print(f"SKIPPED {ENTITY_TYPE} #{DONOR_ID}: bank-feed-matched (error 6480).")
        print(f"  -> Manually unmatch this transaction in QBO before it can be deleted.")
    else:
        print(f"ERROR deleting {ENTITY_TYPE} #{DONOR_ID}: {e.code} {error_body}")
EOF
```

**Notes on merge + delete:**
- Purchase sparse updates **require `PaymentType`** in the payload -- always include it
- Bank-feed-matched transactions return **error 6480** on delete -- catch gracefully and tell the user to unmatch manually in QBO
- `BillPayment` and `VendorCredit` may not support delete -- inform the user if deletion fails
- For JournalEntry, the vendor field may be on individual lines -- check `Line[].JournalEntryLineDetail.Entity` as well

---

## Phase 4: Missing FX Rates

Identify foreign currency transactions with missing or default exchange rates, and offer to fix them with official FX rates.

**Filter criteria:**
- `CurrencyRef.value` != your home currency (e.g., `"CAD"`)
- AND `ExchangeRate` is missing, null, 0, or exactly 1.0

```bash
python3 <<'EOF'
import json, datetime

with open("/tmp/qbo_cleanup_transactions.json") as f:
    all_txns = json.load(f)
with open("/tmp/qbo_cleanup_fx_rates.json") as f:
    fx_rates = json.load(f)

HOME_CURRENCY = "CAD"  # Configure for your QBO home currency

def get_rate(date_str):
    """Get FX rate, falling back to most recent prior business day."""
    d = datetime.date.fromisoformat(date_str)
    for i in range(7):
        key = (d - datetime.timedelta(days=i)).isoformat()
        if key in fx_rates:
            return fx_rates[key]
    return None

def get_vendor_name(txn):
    for key in ("VendorRef", "EntityRef"):
        ref = txn.get(key)
        if ref:
            return ref.get("name", ref.get("value", "unknown"))
    return "(none)"

missing_fx = []

for entity_type, txns in all_txns.items():
    for txn in txns:
        currency = txn.get("CurrencyRef", {}).get("value", HOME_CURRENCY)
        if currency == HOME_CURRENCY:
            continue

        exchange_rate = txn.get("ExchangeRate")
        if exchange_rate is not None and exchange_rate not in (0, 1.0, 1):
            continue

        txn_date = txn.get("TxnDate", "")
        rate = get_rate(txn_date)

        missing_fx.append({
            "entity_type": entity_type,
            "Id": txn.get("Id"),
            "SyncToken": txn.get("SyncToken"),
            "TxnDate": txn_date,
            "TotalAmt": txn.get("TotalAmt"),
            "CurrencyRef": currency,
            "ExchangeRate": exchange_rate,
            "OfficialRate": rate,
            "VendorName": get_vendor_name(txn),
            "PaymentType": txn.get("PaymentType"),
        })

with open("/tmp/qbo_cleanup_missing_fx.json", "w") as f:
    json.dump(missing_fx, f, indent=2)

print(f"Found {len(missing_fx)} transaction(s) with missing/default FX rates\n")

if missing_fx:
    print(f"{'Type':<15} {'Date':<12} {'Vendor':<30} {'Amount':>10} {'Curr':<5} {'Current':>8} {'Official':>9} {'ID':<10}")
    print("-" * 105)
    for t in missing_fx:
        current = t['ExchangeRate'] if t['ExchangeRate'] is not None else "null"
        official = f"{t['OfficialRate']:.4f}" if t['OfficialRate'] else "N/A"
        print(f"{t['entity_type']:<15} {t['TxnDate']:<12} {t['VendorName']:<30} {t['TotalAmt']:>10} {t['CurrencyRef']:<5} {str(current):>8} {official:>9} {t['Id']:<10}")
EOF
```

### Updating Exchange Rates

Present the table to the user. Once they confirm, execute updates immediately:

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"

ENTITY_TYPE = "Purchase"  # replace with actual
ENTITY_ID = "123"         # replace with actual
NEW_RATE = 1.4390         # replace with official rate

# Step 1: Fetch fresh entity
url = f"{BASE}/{ENTITY_TYPE.lower()}/{ENTITY_ID}"
req = urllib.request.Request(url)
req.add_header('Authorization', f'Bearer {API_KEY}')
resp = json.load(urllib.request.urlopen(req))

entity_data = resp.get(ENTITY_TYPE, resp)

# Step 2: Sparse update -- set ExchangeRate
update_payload = {
    "Id": entity_data["Id"],
    "SyncToken": entity_data["SyncToken"],
    "sparse": True,
    "ExchangeRate": NEW_RATE,
}
# Always include PaymentType for Purchase sparse updates
if ENTITY_TYPE == "Purchase" and entity_data.get("PaymentType"):
    update_payload["PaymentType"] = entity_data["PaymentType"]

update_url = f"{BASE}/{ENTITY_TYPE.lower()}"
data = json.dumps(update_payload).encode()
req = urllib.request.Request(update_url, data=data, method='POST')
req.add_header('Authorization', f'Bearer {API_KEY}')
req.add_header('Content-Type', 'application/json')

try:
    result = json.load(urllib.request.urlopen(req))
    updated = result.get(ENTITY_TYPE, result)
    print(f"Updated {ENTITY_TYPE} #{ENTITY_ID}: ExchangeRate = {updated.get('ExchangeRate')}")
except urllib.error.HTTPError as e:
    error_body = e.read().decode()
    print(f"ERROR updating {ENTITY_TYPE} #{ENTITY_ID}: {e.code} {error_body}")
EOF
```

**FX direction:** FXUSDCAD = 1.4390 means 1 USD = 1.4390 CAD. This matches QBO's `ExchangeRate` field when the home currency is CAD. Adjust for your currency pair.

---

## Phase 5: Missing Vendors

Find transactions with no vendor assigned, attempt to auto-match against existing vendors, and offer to create new vendors where no match exists.

### 5a. Detect Missing Vendors

```bash
python3 <<'EOF'
import json, re

with open("/tmp/qbo_cleanup_transactions.json") as f:
    all_txns = json.load(f)
with open("/tmp/qbo_cleanup_vendors.json") as f:
    vendors = json.load(f)  # {display_name_lower: {id, name}}

# Common statement prefixes to strip for matching
STRIP_PREFIXES = [
    r"^FS \*\s*",
    r"^IN \*\s*",
    r"^TST-\s*",
    r"^TST\*\s*",
    r"^WWW\.\s*",
    r"^SER \s*",
    r"^SP \*\s*",
    r"^SQ \*\s*",
    r"^PP\*\s*",
    r"^PAYPAL \*\s*",
]

def clean_description(desc):
    """Normalize a description for fuzzy matching."""
    cleaned = desc.strip()
    for prefix in STRIP_PREFIXES:
        cleaned = re.sub(prefix, "", cleaned, flags=re.IGNORECASE)
    # Remove trailing reference numbers, punctuation
    cleaned = re.sub(r"\s+\d{5,}$", "", cleaned)
    cleaned = cleaned.strip(" .,;:-*")
    return cleaned

def extract_description(txn):
    """Get merchant/description from transaction line items."""
    # Check memo/description fields
    for field in ("PrivateNote", "Memo"):
        val = txn.get(field, "")
        if val:
            return val.strip()
    # Check line item descriptions
    lines = txn.get("Line", [])
    for line in lines:
        desc = line.get("Description", "")
        if desc:
            return desc.strip()
        detail = line.get("AccountBasedExpenseLineDetail", {})
        if detail:
            pass
    return ""

def fuzzy_match_vendor(description, vendors_lookup):
    """Try to match a cleaned description against existing vendors."""
    cleaned = clean_description(description).lower()
    if not cleaned:
        return None
    # Exact match
    if cleaned in vendors_lookup:
        return vendors_lookup[cleaned]
    # Contains match (vendor name appears in description or vice versa)
    for vendor_lower, info in vendors_lookup.items():
        if vendor_lower in cleaned or cleaned in vendor_lower:
            return info
    # Word-level match (first significant word)
    words = cleaned.split()
    if words:
        first_word = words[0].lower()
        if len(first_word) >= 4:  # skip short words
            for vendor_lower, info in vendors_lookup.items():
                if first_word in vendor_lower.split():
                    return info
    return None

def has_vendor(txn):
    for key in ("VendorRef", "EntityRef"):
        ref = txn.get(key)
        if ref and ref.get("value"):
            return True
    return False

# Patterns that indicate bank transfers, not vendor expenses
TRANSFER_PATTERNS = [
    r"(?i)payment to keep",
    r"(?i)e-?tfr",
    r"(?i)tfr-?to",
    r"(?i)send e-",
    r"(?i)^ry\d+\s+tfr",
    r"(?i)interac",
    r"(?i)transfer",
]

def is_transfer(desc):
    """Check if description looks like a bank transfer, not a vendor expense."""
    for pattern in TRANSFER_PATTERNS:
        if re.search(pattern, desc):
            return True
    return False

missing_vendor = []

for entity_type, txns in all_txns.items():
    for txn in txns:
        if has_vendor(txn):
            continue
        desc = extract_description(txn)
        if desc and is_transfer(desc):
            continue  # Skip transfers -- not vendor expenses
        match = fuzzy_match_vendor(desc, vendors) if desc else None
        cleaned = clean_description(desc) if desc else ""

        missing_vendor.append({
            "entity_type": entity_type,
            "Id": txn.get("Id"),
            "SyncToken": txn.get("SyncToken"),
            "TxnDate": txn.get("TxnDate", ""),
            "TotalAmt": txn.get("TotalAmt"),
            "Description": desc,
            "CleanedDescription": cleaned,
            "PaymentType": txn.get("PaymentType"),
            "SuggestedVendor": match["name"] if match else (cleaned.title() if cleaned else ""),
            "SuggestedVendorId": match["id"] if match else None,
            "Action": "Link" if match else ("Create + Link" if cleaned else "Manual"),
        })

with open("/tmp/qbo_cleanup_missing_vendors.json", "w") as f:
    json.dump(missing_vendor, f, indent=2)

print(f"Found {len(missing_vendor)} transaction(s) with no vendor assigned\n")

if missing_vendor:
    print(f"{'ID':<10} {'Date':<12} {'Amount':>10} {'Description':<30} {'Suggested Vendor':<25} {'Action'}")
    print("-" * 100)
    for t in missing_vendor:
        desc_display = (t['Description'][:28] + "..") if len(t['Description']) > 30 else t['Description']
        vendor_display = t['SuggestedVendor'] or "(unknown)"
        existing = " (existing)" if t['SuggestedVendorId'] else " (new)" if t['SuggestedVendor'] else ""
        print(f"{t['Id']:<10} {t['TxnDate']:<12} {t['TotalAmt']:>10} {desc_display:<30} {vendor_display + existing:<25} {t['Action']}")

# Group by suggested vendor for summary
from collections import Counter
vendor_actions = Counter()
for t in missing_vendor:
    vendor_actions[t['Action']] += 1
print(f"\nSummary: {vendor_actions}")
EOF
```

### 5b. Create Vendors & Link Transactions

Present each vendor assignment to the user. Once they decide, execute immediately.

Group multiple transactions with the same suggested vendor into one action.

**To create a new vendor:**

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"

VENDOR_NAME = "Cursor"  # replace with actual name

# Create vendor
create_url = f"{BASE}/vendor"
data = json.dumps({"DisplayName": VENDOR_NAME}).encode()
req = urllib.request.Request(create_url, data=data, method='POST')
req.add_header('Authorization', f'Bearer {API_KEY}')
req.add_header('Content-Type', 'application/json')

try:
    result = json.load(urllib.request.urlopen(req))
    vendor = result.get("Vendor", result)
    print(f"Created vendor: {vendor['DisplayName']} (#{vendor['Id']})")
except urllib.error.HTTPError as e:
    error_body = e.read().decode()
    print(f"ERROR creating vendor: {e.code} {error_body}")
EOF
```

**To link a vendor to a transaction (sparse update):**

```bash
python3 <<'EOF'
import urllib.request, urllib.parse, os, json

API_KEY = os.environ["MATON_API_KEY"]
BASE = "https://gateway.maton.ai/quickbooks/v3/company/:realmId"

ENTITY_TYPE = "Purchase"   # replace with actual type
ENTITY_ID = "4423"         # replace with actual ID
VENDOR_ID = "999"          # replace with vendor ID
VENDOR_NAME = "Cursor"     # replace with vendor name

# Fetch fresh entity
url = f"{BASE}/{ENTITY_TYPE.lower()}/{ENTITY_ID}"
req = urllib.request.Request(url)
req.add_header('Authorization', f'Bearer {API_KEY}')
resp = json.load(urllib.request.urlopen(req))
entity_data = resp.get(ENTITY_TYPE, resp)

# Sparse update with EntityRef
update_payload = {
    "Id": entity_data["Id"],
    "SyncToken": entity_data["SyncToken"],
    "sparse": True,
    "EntityRef": {"value": VENDOR_ID, "name": VENDOR_NAME, "type": "Vendor"},
}
# Always include PaymentType for Purchase sparse updates
if ENTITY_TYPE == "Purchase" and entity_data.get("PaymentType"):
    update_payload["PaymentType"] = entity_data["PaymentType"]

update_url = f"{BASE}/{ENTITY_TYPE.lower()}"
data = json.dumps(update_payload).encode()
req = urllib.request.Request(update_url, data=data, method='POST')
req.add_header('Authorization', f'Bearer {API_KEY}')
req.add_header('Content-Type', 'application/json')

try:
    result = json.load(urllib.request.urlopen(req))
    updated = result.get(ENTITY_TYPE, result)
    print(f"Linked {ENTITY_TYPE} #{ENTITY_ID} to vendor {VENDOR_NAME} (#{VENDOR_ID})")
except urllib.error.HTTPError as e:
    error_body = e.read().decode()
    print(f"ERROR linking vendor to {ENTITY_TYPE} #{ENTITY_ID}: {e.code} {error_body}")
EOF
```

---

## Phase 6: Summary

After all phases complete, print a summary:

```
QBO Cleanup Summary ({label})
=======================================
Date range:            {start_date} to {end_date}
Transactions scanned:  {total}
  - Purchase:          {count}
  - Bill:              {count}
  - BillPayment:       {count}
  - VendorCredit:      {count}
  - JournalEntry:      {count}

Duplicates:            {dup_groups} group(s) found
  Merged + deleted:    {merged_count}
  Skipped:             {skipped_count}
  Failed (bank-matched): {bank_matched_count} -- manual action needed in QBO

Missing FX rates:      {fx_missing} found, {fx_updated} updated, {fx_errors} errors

Missing vendors:       {missing_vendor_count} found
  Linked to existing:  {linked_count}
  New vendors created: {created_count}
  Skipped:             {vendor_skipped}
```

Clean up temp files at `/tmp/qbo_cleanup_*.json` after the session.

---

## API Reference

| Item | Value |
|------|-------|
| **QBO Base URL** | `https://gateway.maton.ai/quickbooks/v3/company/:realmId/` |
| **Auth header** | `Authorization: Bearer $MATON_API_KEY` |
| **FX API (example)** | `https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json?start_date=...&end_date=...` |
| **FX direction** | FXUSDCAD: 1 USD = X CAD (matches QBO ExchangeRate when home=CAD) |
| **Pagination** | STARTPOSITION 1-based, increment by 1000, stop when < 1000 returned |
| **SyncToken** | Always fetch fresh entity before any update or delete |
| **Rate limit** | 10 req/sec via Maton gateway (add `time.sleep(0.15)` between calls) |

### Additional API Endpoints

```
# Attachments query (uses MetaData.LastUpdatedTime, NOT TxnDate)
SELECT * FROM Attachable WHERE MetaData.LastUpdatedTime >= 'YYYY-MM-DD' MAXRESULTS 1000

# Vendor query (for fuzzy matching)
SELECT * FROM Vendor MAXRESULTS 1000

# Create vendor
POST /v3/company/:realmId/vendor
{"DisplayName": "Cursor"}
```

## Important Notes

- **NEVER touch reconciled transactions** -- check `LinkedTxn` and line-level `ClearingStatus`/`MatchStatus`. If a transaction is reconciled (Cleared/Reconciled), skip it entirely. Deleting or modifying a reconciled expense breaks bank reconciliation and requires manual correction in QBO
- **Present each action to the user for decision** -- once they decide, execute immediately without asking to run the script
- **Re-fetch before mutating** -- always GET the entity immediately before POST to get a fresh SyncToken
- Purchase sparse updates **require `PaymentType`** in payload -- always include it
- Bank-feed-matched transactions return **error 6480** on delete -- catch and advise user to unmatch manually in QBO
- `urllib.parse` must be imported at the top alongside `urllib.request`
- `BillPayment` and `VendorCredit` may not support delete operations -- inform the user if deletion fails
- For JournalEntry entities, the vendor field may be on individual lines -- check `Line[].JournalEntryLineDetail.Entity`
- Attachable query uses `MetaData.LastUpdatedTime`, not `TxnDate`
- Common QBO description prefixes to strip for matching: `FS *`, `IN *`, `TST-`, `WWW.`, `SER `, `SP *`, `SQ *`, `PP*`, `PAYPAL *`
- **Auto-imported preference** -- transactions with `PrivateNote` starting with `"Auto-imported: "` are auto-imported from bank feeds and should be preferred as keepers during duplicate merging (priority: auto-imported > has attachment > earliest created)
- **DUPOK** -- transactions with "DUPOK" in PrivateNote are excluded from duplicate detection. Applied when user keeps all in a duplicate group. Ensures idempotent re-runs
- Running the cleanup again should find 0 issues (idempotent)
