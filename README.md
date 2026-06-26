# 🔄 Cassandra → PostgreSQL Migration PoC

> **Proof of Concept** | Apache Cassandra to PostgreSQL migration using Podman on WSL2 (Windows 11)
> Demonstrates schema translation, data migration via Python scripting, and performance benchmarking.

> **Versions:** Apache Cassandra **5.0.8** (Apr 2026) → PostgreSQL **18.4** (May 2026) | cassandra-driver **3.30.0** | psycopg2-binary **2.9.12** | tqdm **4.68.3**

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Environment Setup — Podman on WSL2](#environment-setup--podman-on-wsl2)
4. [Running the Containers](#running-the-containers)
5. [Sample Dataset — Cassandra](#sample-dataset--cassandra)
6. [Equivalent Schema — PostgreSQL](#equivalent-schema--postgresql)
7. [Data Migration Script](#data-migration-script)
8. [Data Validation & Testing](#data-validation--testing)
9. [Performance Strategy & Benchmarking](#performance-strategy--benchmarking)
10. [Cleanup](#cleanup)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      WSL2 (Ubuntu 22.04)                    │
│                                                             │
│   ┌─────────────────┐        ┌──────────────────────────┐  │
│   │   Cassandra 5.0 │        │     PostgreSQL 18         │  │
│   │   (Source)      │        │     (Target)              │  │
│   │   Port: 9042    │        │     Port: 5432            │  │
│   └────────┬────────┘        └────────────┬─────────────┘  │
│            │                              │                 │
│            └──────────┬───────────────────┘                 │
│                       │                                     │
│              ┌────────▼────────┐                            │
│              │  migrate.py     │                            │
│              │  (Python 3.12)  │                            │
│              │  cassandra-driver                            │
│              │  psycopg2       │                            │
│              └─────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

**Domain:** E-commerce order management system
**Dataset:** 2,000 customers · 40,000 orders · 120,000 order items = **162,000 rows**
**Migration time:** ~6 seconds at ~28,000 rows/sec on a single WSL2 node

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Windows 11 | 22H2+ | WSL2 feature enabled |
| WSL2 (Ubuntu) | 22.04 LTS | Default distribution |
| Podman | 4.x+ | Installed inside WSL2 |
| Python | 3.12 | Ships with Ubuntu 22.04 |
| pip packages | — | `cassandra-driver>=3.30.0`, `psycopg2-binary`, `tqdm` |

---

## Environment Setup — Podman on WSL2

### 1. Enable WSL2 and Install Ubuntu

```powershell
# Run in PowerShell (Admin) on Windows
wsl --install -d Ubuntu-22.04
wsl --set-default-version 2
```

Restart your machine, then open **Ubuntu 22.04** from the Start Menu and complete the user setup.

### 2. Install Podman inside WSL2

```bash
# Update packages
sudo apt-get update && sudo apt-get upgrade -y

# Install Podman
sudo apt-get install -y podman

# Verify installation
podman --version
```

### 3. Configure Podman for Rootless Operation

```bash
# Enable lingering for your user (keeps containers alive after logout)
loginctl enable-linger $USER

# Verify subuid/subgid mappings exist
cat /etc/subuid
cat /etc/subgid
# Should show entries like: youruser:100000:65536

# If missing, add them:
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
```

### 4. Install Python Dependencies

Ubuntu 22.04 enforces [PEP 668](https://peps.python.org/pep-0668/), blocking system-wide `pip` installs. Use a virtual environment instead:

```bash
# Install python3-full (includes the venv module)
sudo apt-get install -y python3-full

# Create the project directory and virtual environment
mkdir ~/cassandra2pg && cd ~/cassandra2pg
python3 -m venv ~/migration-env

# Activate it (prompt changes to "(migration-env) ...")
source ~/migration-env/bin/activate

# Install packages inside the venv
pip install "cassandra-driver>=3.30.0" psycopg2-binary tqdm

# Verify
pip list | grep -E "cassandra|psycopg2|tqdm"
```

Expected output:
```
cassandra-driver  3.30.0
psycopg2-binary   2.9.12
tqdm              4.68.3
```

> **Note:** Run `source ~/migration-env/bin/activate` at the start of every new terminal session before executing any project scripts.

---

## Running the Containers

### 1. Create a Shared Pod Network

```bash
podman network create migration-net
```

### 2. Start Apache Cassandra 5.0

```bash
podman run -d \
  --name cassandra-source \
  --network migration-net \
  -p 9042:9042 \
  -e CASSANDRA_CLUSTER_NAME=poc-cluster \
  -e CASSANDRA_DC=datacenter1 \
  -e CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch \
  docker.io/library/cassandra:5.0

# Wait for Cassandra to be ready (~45-60 seconds)
echo "Waiting for Cassandra to start..."
until podman exec cassandra-source cqlsh -e "DESCRIBE KEYSPACES" &>/dev/null; do
  sleep 5
  echo "  ...still waiting"
done
echo "✅ Cassandra is ready."
```

### 3. Start PostgreSQL 18

```bash
podman run -d \
  --name postgres-target \
  --network migration-net \
  -p 5432:5432 \
  -e POSTGRES_USER=poc_user \
  -e POSTGRES_PASSWORD=poc_pass \
  -e POSTGRES_DB=ecommerce \
  docker.io/library/postgres:18

# Wait for PostgreSQL to be ready
echo "Waiting for PostgreSQL to start..."
until podman exec postgres-target pg_isready -U poc_user &>/dev/null; do
  sleep 3
  echo "  ...still waiting"
done
echo "✅ PostgreSQL is ready."
```

### 4. Verify Both Containers Are Running

```bash
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected output:
```
NAMES               STATUS              PORTS
cassandra-source    Up 2 minutes        0.0.0.0:9042->9042/tcp
postgres-target     Up 1 minute         0.0.0.0:5432->5432/tcp
```

---

## Sample Dataset — Cassandra

### 1. Create Keyspace and Schema

> **Important:** CQL does not support `--` inline comments inside a `-e` string. Use a `.cql` file to avoid `SyntaxException`.

```bash
cat > ~/cassandra2pg/schema.cql << 'EOF'
CREATE KEYSPACE IF NOT EXISTS ecommerce
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};

USE ecommerce;

CREATE TABLE IF NOT EXISTS customers (
  customer_id   UUID PRIMARY KEY,
  email         TEXT,
  full_name     TEXT,
  country       TEXT,
  tier          TEXT,
  created_at    TIMESTAMP
);

CREATE TABLE IF NOT EXISTS orders (
  customer_id   UUID,
  order_id      UUID,
  order_date    TIMESTAMP,
  status        TEXT,
  total_amount  DECIMAL,
  currency      TEXT,
  PRIMARY KEY (customer_id, order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);

CREATE TABLE IF NOT EXISTS order_items (
  order_id      UUID,
  item_id       UUID,
  product_name  TEXT,
  quantity      INT,
  unit_price    DECIMAL,
  PRIMARY KEY (order_id, item_id)
);
EOF

podman cp ~/cassandra2pg/schema.cql cassandra-source:/tmp/schema.cql
podman exec -it cassandra-source cqlsh -f /tmp/schema.cql
```

Verify tables were created:

```bash
podman exec -it cassandra-source cqlsh -e "USE ecommerce; DESCRIBE TABLES;"
```

Expected output:
```
customers   orders   order_items
```

### 2. Generate Synthetic Data

Save as `~/cassandra2pg/generate_data.py`:

```python
#!/usr/bin/env python3
"""
generate_data.py — Fast demo data generator for Cassandra.
Uses execute_concurrent_with_args for high throughput.
Target: ~35 seconds total, 162,000 rows.

Tuning notes (single-node Podman container):
  CONCURRENCY=50  — prevents CRC/connection overload on localhost
  CHUNK=2000      — keeps memory pressure low per batch
"""

import uuid, random, time
from datetime import datetime, timedelta
from cassandra.cluster import Cluster
from cassandra.policies import RetryPolicy
from cassandra.concurrent import execute_concurrent_with_args
from tqdm import tqdm

# ── Config ─────────────────────────────────────────────────────────────────────
CASSANDRA_HOST  = "localhost"
CASSANDRA_PORT  = 9042
KEYSPACE        = "ecommerce"
NUM_CUSTOMERS   = 2_000
ORDERS_PER_CUST = 20
ITEMS_PER_ORDER = 3
CONCURRENCY     = 50      # safe for single-node container (200+ causes CRC errors)
CHUNK           = 2_000   # rows per batch

COUNTRIES = ["US", "UK", "DE", "FR", "IN", "BR", "CA", "AU"]
TIERS     = ["bronze", "silver", "gold"]
STATUSES  = ["pending", "shipped", "delivered", "cancelled"]
PRODUCTS  = [
    "Laptop Pro 15", "Wireless Headphones", "USB-C Hub", "Mechanical Keyboard",
    "4K Monitor", "Webcam HD", "SSD 1TB", "Gaming Mouse", "Desk Lamp",
    "Phone Stand", "Cable Organizer", "Laptop Sleeve"
]

# ── Connection ─────────────────────────────────────────────────────────────────
cluster = Cluster(
    [CASSANDRA_HOST],
    port=CASSANDRA_PORT,
    connect_timeout=30
)
session = cluster.connect(KEYSPACE)
session.default_timeout = 120

ins_cust = session.prepare(
    "INSERT INTO customers (customer_id,email,full_name,country,tier,created_at) VALUES (?,?,?,?,?,?)"
)
ins_ord = session.prepare(
    "INSERT INTO orders (customer_id,order_id,order_date,status,total_amount,currency) VALUES (?,?,?,?,?,?)"
)
ins_item = session.prepare(
    "INSERT INTO order_items (order_id,item_id,product_name,quantity,unit_price) VALUES (?,?,?,?,?)"
)

def run_concurrent(session, stmt, params, label):
    """Run inserts in chunks with progress bar and brief pause between chunks."""
    total = len(params)
    with tqdm(total=total, desc=f"  {label}", unit=" rows") as pbar:
        for i in range(0, total, CHUNK):
            chunk = params[i:i+CHUNK]
            execute_concurrent_with_args(
                session, stmt, chunk,
                concurrency=CONCURRENCY,
                raise_on_first_error=True
            )
            pbar.update(len(chunk))
            time.sleep(0.05)  # brief pause — lets Cassandra flush buffers

base = datetime(2023, 1, 1)
t0   = time.time()

# ── Customers ──────────────────────────────────────────────────────────────────
print(f"\n🚀 Generating {NUM_CUSTOMERS:,} customers...")
cust_params  = []
customer_ids = []
for i in range(NUM_CUSTOMERS):
    cid = uuid.uuid4()
    customer_ids.append(cid)
    cust_params.append((
        cid, f"user{i}@example.com", f"Customer {i}",
        random.choice(COUNTRIES), random.choice(TIERS),
        base + timedelta(days=random.randint(0, 500))
    ))
run_concurrent(session, ins_cust, cust_params, "customers")
print(f"   ✅ {NUM_CUSTOMERS:,} customers inserted")

# ── Orders ─────────────────────────────────────────────────────────────────────
total_orders = NUM_CUSTOMERS * ORDERS_PER_CUST
print(f"\n📦 Generating {total_orders:,} orders...")
ord_params = []
order_ids  = []
for cid in customer_ids:
    for _ in range(ORDERS_PER_CUST):
        oid = uuid.uuid4()
        order_ids.append(oid)
        ord_params.append((
            cid, oid,
            base + timedelta(days=random.randint(0, 500), hours=random.randint(0, 23)),
            random.choice(STATUSES),
            round(random.uniform(10.0, 2000.0), 2),
            "USD"
        ))
run_concurrent(session, ins_ord, ord_params, "orders")
print(f"   ✅ {total_orders:,} orders inserted")

# ── Order Items ────────────────────────────────────────────────────────────────
total_items = len(order_ids) * ITEMS_PER_ORDER
print(f"\n🛒 Generating {total_items:,} order items...")
item_params = []
for oid in order_ids:
    for _ in range(ITEMS_PER_ORDER):
        item_params.append((
            oid, uuid.uuid4(),
            random.choice(PRODUCTS),
            random.randint(1, 5),
            round(random.uniform(5.0, 500.0), 2)
        ))
run_concurrent(session, ins_item, item_params, "items")
print(f"   ✅ {total_items:,} items inserted")

# ── Summary ────────────────────────────────────────────────────────────────────
elapsed = time.time() - t0
total   = NUM_CUSTOMERS + total_orders + total_items
print(f"""
╔══════════════════════════════════════════╗
║      Data Generation Complete ✅         ║
╠══════════════════════════════════════════╣
║  Customers   : {NUM_CUSTOMERS:>8,}               ║
║  Orders      : {total_orders:>8,}               ║
║  Order Items : {total_items:>8,}               ║
║  Total rows  : {total:>8,}               ║
║  Time        : {elapsed:>7.1f}s               ║
╚══════════════════════════════════════════╝
""")
cluster.shutdown()
```

```bash
python3 generate_data.py
```

Expected output:
```
🚀 Generating 2,000 customers...
   customers: 100%|████████████| 2000/2000 [00:00<00:00, 4502 rows/s]
   ✅ 2,000 customers inserted
📦 Generating 40,000 orders...
   orders: 100%|████████████| 40000/40000 [00:07<00:00, 5141 rows/s]
   ✅ 40,000 orders inserted
🛒 Generating 120,000 order items...
   items: 100%|████████████| 120000/120000 [00:25<00:00, 4657 rows/s]
   ✅ 120,000 items inserted
╔══════════════════════════════════════════╗
║      Data Generation Complete ✅         ║
║  Total rows  :  162,000               ║
║  Time        :    34.7s               ║
╚══════════════════════════════════════════╝
```

### 3. Verify Row Counts in Cassandra

Run each table separately (multiple statements in a single `-e` string cause `SyntaxException`):

```bash
podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT COUNT(*) FROM customers;"
podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT COUNT(*) FROM orders;"
podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT COUNT(*) FROM order_items;"
```

> ⚠️ `COUNT(*)` on large Cassandra tables triggers an aggregation warning — this is expected and a known Cassandra limitation vs. PostgreSQL.

> ⚠️ If you re-run `generate_data.py` multiple times, row counts will be **higher than expected** because each run generates fresh UUIDs — Cassandra's `INSERT` is an upsert but duplicates are never created, so rows accumulate. Truncate before re-running: `podman exec -it cassandra-source cqlsh -e "USE ecommerce; TRUNCATE customers;"` (repeat for orders and order_items).

---

## Equivalent Schema — PostgreSQL

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
CREATE TABLE IF NOT EXISTS customers (
  customer_id   UUID        PRIMARY KEY,
  email         TEXT        NOT NULL,
  full_name     TEXT,
  country       CHAR(2),
  tier          TEXT        CHECK (tier IN ('bronze','silver','gold')),
  created_at    TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS orders (
  order_id      UUID        PRIMARY KEY,
  customer_id   UUID        NOT NULL REFERENCES customers(customer_id),
  order_date    TIMESTAMPTZ,
  status        TEXT        CHECK (status IN ('pending','shipped','delivered','cancelled')),
  total_amount  NUMERIC(12,2),
  currency      CHAR(3)     DEFAULT 'USD'
);

CREATE TABLE IF NOT EXISTS order_items (
  item_id       UUID        PRIMARY KEY,
  order_id      UUID        NOT NULL REFERENCES orders(order_id),
  product_name  TEXT,
  quantity      INT         CHECK (quantity > 0),
  unit_price    NUMERIC(10,2)
);

CREATE INDEX IF NOT EXISTS idx_orders_customer  ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_orders_date      ON orders(order_date DESC);
CREATE INDEX IF NOT EXISTS idx_items_order      ON order_items(order_id);
CREATE INDEX IF NOT EXISTS idx_customers_country ON customers(country);
"
```

**Key Schema Differences:**

| Concept | Cassandra | PostgreSQL |
|---|---|---|
| Primary Key | Partition + Clustering key | Single/composite PK |
| Relationships | Denormalized (duplicated data) | Foreign keys enforced |
| Aggregates | No native `COUNT(*)` at scale | Full aggregate support |
| Consistency | Tunable (eventual by default) | ACID, strong consistency |
| Query flexibility | Limited to partition key | Full SQL, arbitrary WHERE |

---

## Data Migration Script

Save as `~/cassandra2pg/migrate.py`:

```python
#!/usr/bin/env python3
"""
migrate.py — Cassandra → PostgreSQL migration.

Key design decisions:
  - Token-range partitioning splits the Cassandra ring across N threads
  - MAX_TOKEN uses 2**63 - 1 (not 2**63) to avoid signed int64 overflow
  - Migration order is strictly customers → orders → order_items (FK dependency)
  - ON CONFLICT DO NOTHING makes reruns safe

Usage:
  python3 migrate.py                      # default: 4 threads, batch=500
  python3 migrate.py --threads 8 --batch 1000
  python3 migrate.py --table customers    # single table
"""

import time
import argparse
import threading
from datetime import timezone

import psycopg2
import psycopg2.extras
from cassandra.cluster import Cluster
from cassandra.query import SimpleStatement, ConsistencyLevel
from tqdm import tqdm

# ── Config ─────────────────────────────────────────────────────────────────────
CASS_HOST  = "localhost"
CASS_PORT  = 9042
CASS_KS    = "ecommerce"
PG_DSN     = "host=localhost port=5432 dbname=ecommerce user=poc_user password=poc_pass"
FETCH_SIZE = 1000
STATS      = {"rows": 0, "lock": threading.Lock()}

def pg_conn():
    conn = psycopg2.connect(PG_DSN)
    conn.autocommit = False
    return conn

def record_progress(n):
    with STATS["lock"]:
        STATS["rows"] += n

# ── customers ──────────────────────────────────────────────────────────────────
def migrate_customers(batch_size):
    cluster = Cluster([CASS_HOST], port=CASS_PORT)
    session = cluster.connect(CASS_KS)
    stmt = SimpleStatement(
        "SELECT customer_id,email,full_name,country,tier,created_at FROM customers",
        fetch_size=FETCH_SIZE, consistency_level=ConsistencyLevel.LOCAL_ONE
    )
    conn = pg_conn(); cur = conn.cursor(); batch = []
    sql = """INSERT INTO customers (customer_id,email,full_name,country,tier,created_at)
             VALUES %s ON CONFLICT (customer_id) DO NOTHING"""
    for row in tqdm(session.execute(stmt), desc="  customers", unit=" rows"):
        ts = row.created_at.replace(tzinfo=timezone.utc) if row.created_at else None
        batch.append((str(row.customer_id), row.email, row.full_name,
                      row.country, row.tier, ts))
        if len(batch) >= batch_size:
            psycopg2.extras.execute_values(cur, sql, batch, page_size=batch_size)
            conn.commit(); record_progress(len(batch)); batch.clear()
    if batch:
        psycopg2.extras.execute_values(cur, sql, batch)
        conn.commit(); record_progress(len(batch))
    cur.close(); conn.close(); cluster.shutdown()

# ── orders (parallel by token range) ──────────────────────────────────────────
def migrate_orders_chunk(token_start, token_end, batch_size, pbar):
    cluster = Cluster([CASS_HOST], port=CASS_PORT)
    session = cluster.connect(CASS_KS)
    stmt = SimpleStatement(
        f"""SELECT customer_id,order_id,order_date,status,total_amount,currency
            FROM orders
            WHERE token(customer_id) >= {token_start}
              AND token(customer_id) <  {token_end}""",
        fetch_size=FETCH_SIZE, consistency_level=ConsistencyLevel.LOCAL_ONE
    )
    conn = pg_conn(); cur = conn.cursor(); batch = []
    sql = """INSERT INTO orders (order_id,customer_id,order_date,status,total_amount,currency)
             VALUES %s ON CONFLICT (order_id) DO NOTHING"""
    for row in session.execute(stmt):
        ts = row.order_date.replace(tzinfo=timezone.utc) if row.order_date else None
        batch.append((str(row.order_id), str(row.customer_id), ts,
                      row.status, float(row.total_amount or 0), row.currency))
        if len(batch) >= batch_size:
            psycopg2.extras.execute_values(cur, sql, batch, page_size=batch_size)
            conn.commit(); record_progress(len(batch))
            pbar.update(len(batch)); batch.clear()
    if batch:
        psycopg2.extras.execute_values(cur, sql, batch)
        conn.commit(); record_progress(len(batch)); pbar.update(len(batch))
    cur.close(); conn.close(); cluster.shutdown()

def migrate_orders_parallel(num_threads, batch_size):
    MIN_TOKEN = -(2**63)
    MAX_TOKEN =   2**63 - 1   # must be 2**63-1, not 2**63 (signed int64 overflow)
    step      = (MAX_TOKEN - MIN_TOKEN) // num_threads

    ranges = [(MIN_TOKEN + i * step, MIN_TOKEN + (i + 1) * step)
              for i in range(num_threads)]
    ranges[-1] = (ranges[-1][0], MAX_TOKEN)

    pbar    = tqdm(desc="  orders", unit=" rows", dynamic_ncols=True)
    threads = [threading.Thread(target=migrate_orders_chunk,
                                args=(s, e, batch_size, pbar))
               for s, e in ranges]
    for t in threads: t.start()
    for t in threads: t.join()
    pbar.close()

# ── order_items ────────────────────────────────────────────────────────────────
def migrate_order_items(batch_size):
    cluster = Cluster([CASS_HOST], port=CASS_PORT)
    session = cluster.connect(CASS_KS)
    stmt = SimpleStatement(
        "SELECT order_id,item_id,product_name,quantity,unit_price FROM order_items",
        fetch_size=FETCH_SIZE, consistency_level=ConsistencyLevel.LOCAL_ONE
    )
    conn = pg_conn(); cur = conn.cursor(); batch = []
    sql = """INSERT INTO order_items (item_id,order_id,product_name,quantity,unit_price)
             VALUES %s ON CONFLICT (item_id) DO NOTHING"""
    for row in tqdm(session.execute(stmt), desc="  items", unit=" rows"):
        batch.append((str(row.item_id), str(row.order_id),
                      row.product_name, row.quantity, float(row.unit_price or 0)))
        if len(batch) >= batch_size:
            psycopg2.extras.execute_values(cur, sql, batch, page_size=batch_size)
            conn.commit(); record_progress(len(batch)); batch.clear()
    if batch:
        psycopg2.extras.execute_values(cur, sql, batch)
        conn.commit(); record_progress(len(batch))
    cur.close(); conn.close(); cluster.shutdown()

# ── main ───────────────────────────────────────────────────────────────────────
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--threads", type=int, default=4)
    parser.add_argument("--batch",   type=int, default=500)
    parser.add_argument("--table",   type=str, default="all",
                        choices=["all", "customers", "orders", "order_items"])
    args = parser.parse_args()

    print(f"\n🔄 Starting migration | threads={args.threads} | batch={args.batch}\n")
    t0 = time.time()

    # Migration order is fixed: customers → orders → order_items (FK dependency)
    if args.table in ("all", "customers"):
        print("📋 Migrating customers...")
        migrate_customers(args.batch)

    if args.table in ("all", "orders"):
        print("📋 Migrating orders (parallel)...")
        migrate_orders_parallel(args.threads, args.batch)

    if args.table in ("all", "order_items"):
        print("📋 Migrating order_items (after orders committed)...")
        migrate_order_items(args.batch)

    elapsed    = time.time() - t0
    total_rows = STATS["rows"]
    throughput = total_rows / elapsed if elapsed > 0 else 0

    print(f"""
╔══════════════════════════════════════════╗
║         Migration Complete ✅            ║
╠══════════════════════════════════════════╣
║  Total rows migrated : {total_rows:>10,}      ║
║  Elapsed time        : {elapsed:>10.1f}s      ║
║  Throughput          : {throughput:>8,.0f} rows/s  ║
╚══════════════════════════════════════════╝
""")

if __name__ == "__main__":
    main()
```

### Run the Migration

If retesting, always truncate PostgreSQL first to avoid FK conflicts from partial previous runs:

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c \
  "TRUNCATE order_items, orders, customers RESTART IDENTITY CASCADE;"
```

Then run:

```bash
# Default: 4 threads, batch size 500
python3 migrate.py

# High-throughput: 8 threads, batch size 1000
python3 migrate.py --threads 8 --batch 1000

# Single table only
python3 migrate.py --table customers
```

Expected output:
```
🔄 Starting migration | threads=4 | batch=500

📋 Migrating customers...
  customers: 2000 rows [00:00, 28402 rows/s]
📋 Migrating orders (parallel)...
  orders: 40000 rows [00:01, 39853 rows/s]
📋 Migrating order_items (after orders committed)...
  items: 120000 rows [00:04, 26534 rows/s]

╔══════════════════════════════════════════╗
║         Migration Complete ✅            ║
║  Total rows migrated :    162,000      ║
║  Elapsed time        :        5.7s      ║
║  Throughput          :   28,286 rows/s  ║
╚══════════════════════════════════════════╝
```

---

## Data Validation & Testing

### 1. Row Count Comparison (validate.py)

Save as `~/cassandra2pg/validate.py`:

```python
#!/usr/bin/env python3
"""validate.py — Side-by-side row count comparison: Cassandra vs PostgreSQL."""

import psycopg2
from cassandra.cluster import Cluster

cass = Cluster(["localhost"], port=9042).connect("ecommerce")
pg   = psycopg2.connect("host=localhost port=5432 dbname=ecommerce user=poc_user password=poc_pass")
cur  = pg.cursor()

checks = [
    ("customers",   "SELECT COUNT(*) FROM customers"),
    ("orders",      "SELECT COUNT(*) FROM orders"),
    ("order_items", "SELECT COUNT(*) FROM order_items"),
]

print(f"\n{'Table':<15} {'Cassandra':>12} {'PostgreSQL':>12} {'Match':>8}")
print("─" * 52)

for table, sql in checks:
    cass_count = cass.execute(f"SELECT COUNT(*) FROM {table}").one()[0]
    cur.execute(sql)
    pg_count   = cur.fetchone()[0]
    match      = "✅" if cass_count == pg_count else "❌"
    print(f"{table:<15} {cass_count:>12,} {pg_count:>12,} {match:>8}")

print()
cass.cluster.shutdown()
cur.close()
pg.close()
```

```bash
python3 validate.py
```

Expected output:
```
Table           Cassandra   PostgreSQL    Match
────────────────────────────────────────────────────
customers           2,000        2,000        ✅
orders             40,000       40,000        ✅
order_items       120,000      120,000        ✅
```

### 2. Referential Integrity Check

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT 'orphaned_orders' AS check_name, COUNT(*) AS violations
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL
UNION ALL
SELECT 'orphaned_items', COUNT(*)
FROM order_items i
LEFT JOIN orders o ON i.order_id = o.order_id
WHERE o.order_id IS NULL;
"
```

Expected: both rows return `0`.

### 3. Business Query Demo

These queries are **impossible in Cassandra** without full table scans and client-side aggregation. In PostgreSQL they run in milliseconds:

```bash
# Revenue by country and tier
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT
  c.country,
  c.tier,
  COUNT(DISTINCT o.order_id)          AS total_orders,
  ROUND(SUM(o.total_amount)::numeric, 2) AS total_revenue,
  ROUND(AVG(o.total_amount)::numeric, 2) AS avg_order_value
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'delivered'
GROUP BY c.country, c.tier
ORDER BY total_revenue DESC
LIMIT 10;
"

# Orders by status with revenue — date range filter
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT status, COUNT(*) AS orders, ROUND(SUM(total_amount)::numeric, 2) AS revenue
FROM orders
WHERE order_date BETWEEN '2023-06-01' AND '2024-06-01'
GROUP BY status
ORDER BY revenue DESC;
"

# Top products by gross value
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT product_name,
       SUM(quantity) AS units_sold,
       ROUND(SUM(quantity * unit_price)::numeric, 2) AS gross_value
FROM order_items
GROUP BY product_name
ORDER BY gross_value DESC;
"
```

### 4. EXPLAIN ANALYZE — Flagship Demo Query

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
EXPLAIN ANALYZE
SELECT
  DATE_TRUNC('month', o.order_date) AS month,
  c.tier,
  COUNT(*)                          AS orders,
  SUM(o.total_amount)               AS revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= NOW() - INTERVAL '12 months'
  AND o.status = 'delivered'
GROUP BY 1, 2
ORDER BY 1 DESC, revenue DESC;
"
```

Verified result on this PoC:
```
Execution Time: 0.143 ms   ← JOIN + GROUP BY + DATE_TRUNC + index scan
Planning Time:  0.939 ms
Index used:     idx_orders_date  ✅
```

---

## Performance Strategy & Benchmarking

### Overview

| Technique | Implementation | Impact |
|---|---|---|
| **Batch inserts** | `execute_values()` with page_size | 10–20× vs row-by-row |
| **Parallel threads** | Token-range partitioning of Cassandra ring | Near-linear scaling |
| **Prepared statements** | `session.prepare()` in Cassandra | Reduced parse overhead |
| **Connection pooling** | psycopg2 persistent connections per thread | Eliminates reconnect latency |
| **Tunable consistency** | `LOCAL_ONE` on reads | Lower Cassandra read cost |
| **Concurrent inserts** | `execute_concurrent_with_args()` in data gen | 5,000+ rows/s per table |

---

### Benchmark Script

Save as `~/cassandra2pg/benchmark.sh`:

```bash
#!/bin/bash
echo ""
echo "════════════════════════════════════════════════════"
echo "  BENCHMARK 1: Batch Size vs Throughput"
echo "════════════════════════════════════════════════════"
for BATCH in 100 250 500 1000; do
  podman exec -it postgres-target psql -U poc_user -d ecommerce -q -c \
    "TRUNCATE order_items, orders, customers CASCADE;" 2>/dev/null
  RESULT=$(python3 migrate.py --threads 4 --batch $BATCH 2>/dev/null | grep "Throughput")
  echo "  batch=$BATCH   $RESULT"
done

echo ""
echo "════════════════════════════════════════════════════"
echo "  BENCHMARK 2: Thread Count vs Throughput"
echo "════════════════════════════════════════════════════"
for THREADS in 1 2 4 8; do
  podman exec -it postgres-target psql -U poc_user -d ecommerce -q -c \
    "TRUNCATE order_items, orders, customers CASCADE;" 2>/dev/null
  RESULT=$(python3 migrate.py --threads $THREADS --batch 500 2>/dev/null | grep "Throughput")
  echo "  threads=$THREADS   $RESULT"
done
echo ""
```

```bash
chmod +x ~/cassandra2pg/benchmark.sh
bash ~/cassandra2pg/benchmark.sh
```

---

### Optimization Checklist

```bash
# Refresh query planner statistics after migration
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "ANALYZE;"

# Check index usage (PostgreSQL 18 uses indexrelname, not indexname)
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT
  relname          AS table_name,
  indexrelname     AS index_name,
  idx_scan         AS times_used,
  idx_tup_read     AS rows_read
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
"

# Check table sizes on disk
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT
  relname                                         AS table_name,
  pg_size_pretty(pg_total_relation_size(oid))     AS total_size
FROM pg_class
WHERE relkind = 'r'
  AND relname IN ('customers','orders','order_items')
ORDER BY pg_total_relation_size(oid) DESC;
"
```

Verified results on this PoC:
```
table_name   | total_size
-------------+-----------
order_items  | 18 MB
orders       | 6472 kB
customers    | 368 kB
```

---

### Client Demo: Side-by-Side Capability Comparison

| Query Type | Cassandra 5.0 | PostgreSQL 18 (post-migration) |
|---|---|---|
| Lookup by primary key | ✅ Fast | ✅ Fast |
| Filter by non-partition column | ⚠️ SAI index (5.0+) or ALLOW FILTERING | ✅ Index scan on any column |
| Multi-table JOIN | ❌ Not supported | ✅ Native |
| Aggregation (SUM, AVG, COUNT) | ❌ Client-side only | ✅ Native |
| Window functions | ❌ Not supported | ✅ Native |
| Vector / ANN search | ✅ New in 5.0 (HNSW via SAI) | ✅ via pgvector extension |
| Ad hoc reporting | ❌ Pre-modeled tables only | ✅ Flexible SQL |
| `COUNT(*)` on large table | ⚠️ Minutes (full scan + warning) | ✅ Milliseconds (planner stats) |

---

## Cleanup

### Stop and Remove Containers

```bash
# Stop containers
podman stop cassandra-source postgres-target

# Remove containers
podman rm cassandra-source postgres-target

# Remove the shared network
podman network rm migration-net

# Optional: remove pulled images to free disk space
podman rmi docker.io/library/cassandra:5.0
podman rmi docker.io/library/postgres:18
```

### Verify Clean State

```bash
podman ps -a
podman network ls
# Neither container nor migration-net should appear
```

### Remove Python Virtual Environment

```bash
# Deactivate first if currently active
deactivate

# Remove the entire venv directory
rm -rf ~/migration-env
```

---

## Project Structure

```
cassandra2pg/
├── README.md
├── schema.cql            # Cassandra keyspace + table definitions
├── generate_data.py      # Synthetic data generator (162k rows, ~35s)
├── migrate.py            # Migration script (parallel threads, batched inserts)
├── validate.py           # Side-by-side row count validation
└── benchmark.sh          # Batch size + thread count throughput sweep
```

---

## Key Takeaways

- **Cassandra 5.0** introduced Storage-Attached Indexes (SAI) and vector search, narrowing the gap for secondary queries — but joins, ad-hoc aggregations, and complex reporting still require PostgreSQL.
- **PostgreSQL 18** provides full relational capabilities, strong consistency, and rich aggregate/join/window-function support that Cassandra cannot match natively.
- This PoC migrates **162,000 rows in ~6 seconds** (~28,000 rows/sec) on a single WSL2 node using token-range parallel threads and `execute_values` batch inserts.
- The same pattern is production-scalable and adaptable to cloud targets (Amazon RDS, AlloyDB, Aurora PostgreSQL).
- A JOIN + GROUP BY + date-range query that is impossible natively in Cassandra executes in **0.143 ms** in PostgreSQL using index scans — no application-side aggregation required.

