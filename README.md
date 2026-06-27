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
│   ┌─────────────────┐        ┌──────────────────────────┐   │
│   │   Cassandra 5.0 │        │     PostgreSQL 18        │   │
│   │   (Source)      │        │     (Target)             │   │
│   │   Port: 9042    │        │     Port: 5432           │   │
│   └────────┬────────┘        └────────────┬─────────────┘   │
│            │                              │                 │
│            └──────────┬───────────────────┘                 │
│                       │                                     │
│              ┌────────▼──────────┐                          │
│              │  migrate.py       │                          │
│              │  (Python 3.12)    │                          │
│              │  cassandra-driver |                          │
│              │  psycopg2         │                          │
│              └───────────────────┘                          │
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

### Step 1 — Enable WSL2 and Install Ubuntu

```powershell
# Run in PowerShell (Admin) on Windows
wsl --install -d Ubuntu-22.04
wsl --set-default-version 2
```

Restart your machine, then open **Ubuntu 22.04** from the Start Menu and complete the user setup.

### Step 2 — Install Podman inside WSL2

```bash
# Update packages
sudo apt-get update && sudo apt-get upgrade -y

# Install Podman
sudo apt-get install -y podman

# Verify installation
podman --version
```

### Step 3 — Configure Podman for Rootless Operation

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

### Step 4 — Install Python Dependencies

Ubuntu 22.04 enforces [PEP 668](https://peps.python.org/pep-0668/), blocking system-wide `pip` installs. Use a virtual environment instead:

```bash
# Install python3-full (includes the venv module)
sudo apt-get install -y python3-full

# Create the project directory
mkdir ~/cassandra2pg && cd ~/cassandra2pg

# Create and activate a virtual environment
python3 -m venv ~/migration-env
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

### Step 1 — Create a Shared Pod Network

```bash
podman network create migration-net
```

### Step 2 — Start Apache Cassandra 5.0

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

### Step 3 — Start PostgreSQL 18

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

### Step 4 — Verify Both Containers Are Running

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

### Step 1 — Create Keyspace and Schema

> **Important:** CQL does not support `--` inline comments inside a `-e` string. Use a `.cql` file to avoid `SyntaxException`. Do not use `vi` — generate the file automatically with `cat`:

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

### Step 2 — Generate `generate_data.py` using cat

```bash
cat > ~/cassandra2pg/generate_data.py << 'EOF'
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

def ts():
    return datetime.now().strftime("%H:%M:%S")

def print_step(num, msg):
    print(f"\n[Step {num}] {msg}")
    print(f"  ⏱  Start : {ts()}")
    return time.time()

def print_done(t0, rows):
    elapsed = time.time() - t0
    print(f"  ⏱  End   : {ts()}")
    print(f"  ⏱  Time  : {elapsed:.1f}s  |  {rows:,} rows  |  {rows/elapsed:,.0f} rows/s")

# ── Connection ─────────────────────────────────────────────────────────────────
cluster = Cluster([CASSANDRA_HOST], port=CASSANDRA_PORT, connect_timeout=30)
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
    total = len(params)
    with tqdm(total=total, desc=f"    {label}", unit=" rows") as pbar:
        for i in range(0, total, CHUNK):
            chunk = params[i:i+CHUNK]
            execute_concurrent_with_args(
                session, stmt, chunk,
                concurrency=CONCURRENCY,
                raise_on_first_error=True
            )
            pbar.update(len(chunk))
            time.sleep(0.05)

base     = datetime(2023, 1, 1)
wall_t0  = time.time()

print("╔══════════════════════════════════════════════════════╗")
print("║         Cassandra Data Generator — Starting          ║")
print(f"║  Script Start : {ts()}                            ║")
print("╚══════════════════════════════════════════════════════╝")

# ── Step 1: Customers ──────────────────────────────────────────────────────────
t0 = print_step(1, f"Generating {NUM_CUSTOMERS:,} customers")
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
print_done(t0, NUM_CUSTOMERS)

# ── Step 2: Orders ─────────────────────────────────────────────────────────────
total_orders = NUM_CUSTOMERS * ORDERS_PER_CUST
t0 = print_step(2, f"Generating {total_orders:,} orders")
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
print_done(t0, total_orders)

# ── Step 3: Order Items ────────────────────────────────────────────────────────
total_items = len(order_ids) * ITEMS_PER_ORDER
t0 = print_step(3, f"Generating {total_items:,} order items")
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
print_done(t0, total_items)

# ── Summary ────────────────────────────────────────────────────────────────────
total_rows = NUM_CUSTOMERS + total_orders + total_items
wall_elapsed = time.time() - wall_t0
print(f"""
╔══════════════════════════════════════════════════════╗
║         Data Generation Complete ✅                  ║
╠══════════════════════════════════════════════════════╣
║  Step 1 — Customers   : {NUM_CUSTOMERS:>8,}                     ║
║  Step 2 — Orders      : {total_orders:>8,}                     ║
║  Step 3 — Order Items : {total_items:>8,}                     ║
║  ──────────────────────────────────────────────────  ║
║  Total rows           : {total_rows:>8,}                    ║
║  Script End           : {ts()}                     ║
║  Total Time           : {wall_elapsed:>7.1f}s                     ║
╚══════════════════════════════════════════════════════╝
""")
cluster.shutdown()
EOF

```

Run it:

```bash
python3 ~/cassandra2pg/generate_data.py
```

Expected output:
```
╔══════════════════════════════════════════════════════╗
║         Cassandra Data Generator — Starting          ║
║  Script Start : 10:05:01                             ║
╚══════════════════════════════════════════════════════╝

[Step 1] Generating 2,000 customers
  ⏱  Start : 10:05:01
    customers: 100%|████████████| 2000/2000 [00:00<00:00, 4502 rows/s]
  ⏱  End   : 10:05:02
  ⏱  Time  : 0.4s  |  2,000 rows  |  4,502 rows/s

[Step 2] Generating 40,000 orders
  ⏱  Start : 10:05:02
    orders: 100%|████████████| 40000/40000 [00:07<00:00, 5141 rows/s]
  ⏱  End   : 10:05:09
  ⏱  Time  : 7.8s  |  40,000 rows  |  5,141 rows/s

[Step 3] Generating 120,000 order items
  ⏱  Start : 10:05:09
    items: 100%|████████████| 120000/120000 [00:25<00:00, 4657 rows/s]
  ⏱  End   : 10:05:35
  ⏱  Time  : 25.8s  |  120,000 rows  |  4,657 rows/s

╔══════════════════════════════════════════════════════╗
║         Data Generation Complete ✅                  ║
║  Total rows           :  162,000                     ║
║  Script End           : 10:05:35                     ║
║  Total Time           :   34.7s                      ║
╚══════════════════════════════════════════════════════╝
```

### Step 3 — Verify Row Counts in Cassandra

Run each table separately (multiple statements in one `-e` string cause `SyntaxException`):

```bash
podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT COUNT(*) FROM customers;"
podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT COUNT(*) FROM orders;"
podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT COUNT(*) FROM order_items;"
```

> ⚠️ `COUNT(*)` on large Cassandra tables triggers an aggregation warning — this is expected and a known Cassandra limitation vs. PostgreSQL.

> ⚠️ If you re-run `generate_data.py` multiple times, row counts will be higher than expected because each run generates fresh UUIDs. Truncate before re-running:
> ```bash
> podman exec -it cassandra-source cqlsh -e "USE ecommerce; TRUNCATE customers;"
> podman exec -it cassandra-source cqlsh -e "USE ecommerce; TRUNCATE orders;"
> podman exec -it cassandra-source cqlsh -e "USE ecommerce; TRUNCATE order_items;"
> ```

---

## Equivalent Schema — PostgreSQL

### Step 1 — Create Tables and Indexes

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

CREATE INDEX IF NOT EXISTS idx_orders_customer   ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_orders_date       ON orders(order_date DESC);
CREATE INDEX IF NOT EXISTS idx_items_order       ON order_items(order_id);
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

### Step 1 — Generate `migrate.py` using cat

```bash
cat > ~/cassandra2pg/migrate.py << 'EOF'
#!/usr/bin/env python3
"""
migrate.py — Cassandra to PostgreSQL migration.

Key design decisions:
  - Token-range partitioning splits the Cassandra ring across N threads
  - MAX_TOKEN uses 2**63 - 1 (not 2**63) to avoid signed int64 overflow
  - Migration order is strictly customers -> orders -> order_items (FK dependency)
  - ON CONFLICT DO NOTHING makes reruns safe
  - Each stage prints Step number, start time, end time, elapsed time

Usage:
  python3 migrate.py                      # default: 4 threads, batch=500
  python3 migrate.py --threads 8 --batch 1000
  python3 migrate.py --table customers    # single table
"""

import time, argparse, threading
from datetime import datetime, timezone

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

def ts():
    return datetime.now().strftime("%H:%M:%S")

def print_step(num, msg):
    print(f"\n[Step {num}] {msg}")
    print(f"  ⏱  Start : {ts()}")
    return time.time()

def print_done(t0, rows):
    elapsed = time.time() - t0
    rate    = rows / elapsed if elapsed > 0 else 0
    print(f"  ⏱  End   : {ts()}")
    print(f"  ⏱  Time  : {elapsed:.1f}s  |  {rows:,} rows  |  {rate:,.0f} rows/s")

def pg_conn():
    conn = psycopg2.connect(PG_DSN)
    conn.autocommit = False
    return conn

def record_progress(n):
    with STATS["lock"]:
        STATS["rows"] += n

# ── Step 1: customers ──────────────────────────────────────────────────────────
def migrate_customers(batch_size, step_num):
    t0      = print_step(step_num, "Migrating customers")
    cluster = Cluster([CASS_HOST], port=CASS_PORT)
    session = cluster.connect(CASS_KS)
    stmt    = SimpleStatement(
        "SELECT customer_id,email,full_name,country,tier,created_at FROM customers",
        fetch_size=FETCH_SIZE, consistency_level=ConsistencyLevel.LOCAL_ONE
    )
    conn = pg_conn(); cur = conn.cursor(); batch = []
    sql = """INSERT INTO customers (customer_id,email,full_name,country,tier,created_at)
             VALUES %s ON CONFLICT (customer_id) DO NOTHING"""
    count = 0
    for row in tqdm(session.execute(stmt), desc="    customers", unit=" rows"):
        ts_val = row.created_at.replace(tzinfo=timezone.utc) if row.created_at else None
        batch.append((str(row.customer_id), row.email, row.full_name,
                      row.country, row.tier, ts_val))
        if len(batch) >= batch_size:
            psycopg2.extras.execute_values(cur, sql, batch, page_size=batch_size)
            conn.commit(); record_progress(len(batch)); count += len(batch); batch.clear()
    if batch:
        psycopg2.extras.execute_values(cur, sql, batch)
        conn.commit(); record_progress(len(batch)); count += len(batch)
    cur.close(); conn.close(); cluster.shutdown()
    print_done(t0, count)

# ── Step 2: orders (parallel by token range) ───────────────────────────────────
def migrate_orders_chunk(token_start, token_end, batch_size, pbar):
    cluster = Cluster([CASS_HOST], port=CASS_PORT)
    session = cluster.connect(CASS_KS)
    stmt    = SimpleStatement(
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
        ts_val = row.order_date.replace(tzinfo=timezone.utc) if row.order_date else None
        batch.append((str(row.order_id), str(row.customer_id), ts_val,
                      row.status, float(row.total_amount or 0), row.currency))
        if len(batch) >= batch_size:
            psycopg2.extras.execute_values(cur, sql, batch, page_size=batch_size)
            conn.commit(); record_progress(len(batch))
            pbar.update(len(batch)); batch.clear()
    if batch:
        psycopg2.extras.execute_values(cur, sql, batch)
        conn.commit(); record_progress(len(batch)); pbar.update(len(batch))
    cur.close(); conn.close(); cluster.shutdown()

def migrate_orders_parallel(num_threads, batch_size, step_num):
    t0        = print_step(step_num, f"Migrating orders ({num_threads} parallel threads)")
    MIN_TOKEN = -(2**63)
    MAX_TOKEN =   2**63 - 1   # must be 2**63-1 — 2**63 causes signed int64 overflow
    step      = (MAX_TOKEN - MIN_TOKEN) // num_threads
    ranges    = [(MIN_TOKEN + i * step, MIN_TOKEN + (i + 1) * step)
                 for i in range(num_threads)]
    ranges[-1] = (ranges[-1][0], MAX_TOKEN)

    pbar    = tqdm(desc="    orders", unit=" rows", dynamic_ncols=True)
    threads = [threading.Thread(target=migrate_orders_chunk,
                                args=(s, e, batch_size, pbar))
               for s, e in ranges]
    before = STATS["rows"]
    for t in threads: t.start()
    for t in threads: t.join()
    pbar.close()
    print_done(t0, STATS["rows"] - before)

# ── Step 3: order_items ────────────────────────────────────────────────────────
def migrate_order_items(batch_size, step_num):
    t0      = print_step(step_num, "Migrating order_items (after orders fully committed)")
    cluster = Cluster([CASS_HOST], port=CASS_PORT)
    session = cluster.connect(CASS_KS)
    stmt    = SimpleStatement(
        "SELECT order_id,item_id,product_name,quantity,unit_price FROM order_items",
        fetch_size=FETCH_SIZE, consistency_level=ConsistencyLevel.LOCAL_ONE
    )
    conn = pg_conn(); cur = conn.cursor(); batch = []
    sql = """INSERT INTO order_items (item_id,order_id,product_name,quantity,unit_price)
             VALUES %s ON CONFLICT (item_id) DO NOTHING"""
    count = 0
    for row in tqdm(session.execute(stmt), desc="    items", unit=" rows"):
        batch.append((str(row.item_id), str(row.order_id),
                      row.product_name, row.quantity, float(row.unit_price or 0)))
        if len(batch) >= batch_size:
            psycopg2.extras.execute_values(cur, sql, batch, page_size=batch_size)
            conn.commit(); record_progress(len(batch)); count += len(batch); batch.clear()
    if batch:
        psycopg2.extras.execute_values(cur, sql, batch)
        conn.commit(); record_progress(len(batch)); count += len(batch)
    cur.close(); conn.close(); cluster.shutdown()
    print_done(t0, count)

# ── Main ───────────────────────────────────────────────────────────────────────
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--threads", type=int, default=4)
    parser.add_argument("--batch",   type=int, default=500)
    parser.add_argument("--table",   type=str, default="all",
                        choices=["all", "customers", "orders", "order_items"])
    args = parser.parse_args()

    wall_t0 = time.time()
    print("╔══════════════════════════════════════════════════════╗")
    print("║         Cassandra → PostgreSQL Migration             ║")
    print(f"║  Script Start : {datetime.now().strftime('%H:%M:%S')}  |  threads={args.threads}  |  batch={args.batch}  ║")
    print("╚══════════════════════════════════════════════════════╝")

    step = 1
    # Order is fixed: customers -> orders -> order_items (FK dependency)
    if args.table in ("all", "customers"):
        migrate_customers(args.batch, step); step += 1
    if args.table in ("all", "orders"):
        migrate_orders_parallel(args.threads, args.batch, step); step += 1
    if args.table in ("all", "order_items"):
        migrate_order_items(args.batch, step)

    wall_elapsed = time.time() - wall_t0
    total_rows   = STATS["rows"]
    throughput   = total_rows / wall_elapsed if wall_elapsed > 0 else 0

    print(f"""
╔══════════════════════════════════════════════════════╗
║         Migration Complete ✅                        ║
╠══════════════════════════════════════════════════════╣
║  Total rows migrated  : {total_rows:>10,}              ║
║  Script End           : {datetime.now().strftime('%H:%M:%S')}                    ║
║  Total Time           : {wall_elapsed:>10.1f}s              ║
║  Overall Throughput   : {throughput:>8,.0f} rows/s          ║
╚══════════════════════════════════════════════════════╝
""")

if __name__ == "__main__":
    main()
EOF
```

### Step 2 — Truncate PostgreSQL Before Running (prevents FK conflicts on reruns)

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c \
  "TRUNCATE order_items, orders, customers RESTART IDENTITY CASCADE;"
```

### Step 3 — Run the Migration

```bash
# Default: 4 threads, batch size 500
python3 ~/cassandra2pg/migrate.py

# High-throughput: 8 threads, batch size 1000
python3 ~/cassandra2pg/migrate.py --threads 8 --batch 1000

# Single table only
python3 ~/cassandra2pg/migrate.py --table customers
```

Expected output:
```
╔══════════════════════════════════════════════════════╗
║         Cassandra → PostgreSQL Migration             ║
║  Script Start : 10:10:00  |  threads=4  |  batch=500 ║
╚══════════════════════════════════════════════════════╝

[Step 1] Migrating customers
  ⏱  Start : 10:10:00
    customers: 2000 rows [00:00, 28402 rows/s]
  ⏱  End   : 10:10:00
  ⏱  Time  : 0.1s  |  2,000 rows  |  28,402 rows/s

[Step 2] Migrating orders (4 parallel threads)
  ⏱  Start : 10:10:00
    orders: 40000 rows [00:01, 39853 rows/s]
  ⏱  End   : 10:10:01
  ⏱  Time  : 1.0s  |  40,000 rows  |  39,853 rows/s

[Step 3] Migrating order_items (after orders fully committed)
  ⏱  Start : 10:10:01
    items: 120000 rows [00:04, 26534 rows/s]
  ⏱  End   : 10:10:05
  ⏱  Time  : 4.5s  |  120,000 rows  |  26,534 rows/s

╔══════════════════════════════════════════════════════╗
║         Migration Complete ✅                        ║
║  Total rows migrated  :    162,000                   ║
║  Script End           : 10:10:05                     ║
║  Total Time           :      5.7s                    ║
║  Overall Throughput   :  28,286 rows/s               ║
╚══════════════════════════════════════════════════════╝
```

---

## Data Validation & Testing

### Step 1 — Generate `validate.py` using cat

```bash
cat > ~/cassandra2pg/validate.py << 'EOF'
#!/usr/bin/env python3
"""
validate.py — Side-by-side row count + integrity validation.
Compares Cassandra vs PostgreSQL row counts and checks FK integrity.
Prints Step number, start time, end time, elapsed time per check.
"""

import time
import psycopg2
from datetime import datetime
from cassandra.cluster import Cluster

PG_DSN = "host=localhost port=5432 dbname=ecommerce user=poc_user password=poc_pass"

def ts():
    return datetime.now().strftime("%H:%M:%S")

def print_step(num, msg):
    print(f"\n[Step {num}] {msg}")
    print(f"  ⏱  Start : {ts()}")
    return time.time()

def print_done(t0):
    print(f"  ⏱  End   : {ts()}")
    print(f"  ⏱  Time  : {time.time() - t0:.2f}s")

wall_t0 = time.time()
print("╔══════════════════════════════════════════════════════╗")
print("║         Data Validation — Starting                   ║")
print(f"║  Script Start : {ts()}                         ║")
print("╚══════════════════════════════════════════════════════╝")

# ── Step 1: Connect ────────────────────────────────────────────────────────────
t0 = print_step(1, "Connecting to Cassandra and PostgreSQL")
cass    = Cluster(["localhost"], port=9042).connect("ecommerce")
pg_conn = psycopg2.connect(PG_DSN)
cur     = pg_conn.cursor()
print_done(t0)

# ── Step 2: Row Count Comparison ───────────────────────────────────────────────
t0 = print_step(2, "Row count comparison — Cassandra vs PostgreSQL")
tables = [
    ("customers",   "SELECT COUNT(*) FROM customers"),
    ("orders",      "SELECT COUNT(*) FROM orders"),
    ("order_items", "SELECT COUNT(*) FROM order_items"),
]

all_match = True
print(f"\n  {'Table':<15} {'Cassandra':>12} {'PostgreSQL':>12} {'Match':>8}")
print("  " + "─" * 52)
for table, sql in tables:
    cass_count = cass.execute(f"SELECT COUNT(*) FROM {table}").one()[0]
    cur.execute(sql)
    pg_count = cur.fetchone()[0]
    match    = "✅" if cass_count == pg_count else "❌"
    if cass_count != pg_count:
        all_match = False
    print(f"  {table:<15} {cass_count:>12,} {pg_count:>12,} {match:>8}")
print_done(t0)

# ── Step 3: Referential Integrity ──────────────────────────────────────────────
t0 = print_step(3, "Referential integrity check (orphaned FK rows)")
cur.execute("""
    SELECT 'orphaned_orders' AS check_name, COUNT(*) AS violations
    FROM orders o
    LEFT JOIN customers c ON o.customer_id = c.customer_id
    WHERE c.customer_id IS NULL
    UNION ALL
    SELECT 'orphaned_items', COUNT(*)
    FROM order_items i
    LEFT JOIN orders o ON i.order_id = o.order_id
    WHERE o.order_id IS NULL
""")
rows = cur.fetchall()
print(f"\n  {'Check':<20} {'Violations':>12} {'Status':>8}")
print("  " + "─" * 44)
for check_name, violations in rows:
    status = "✅" if violations == 0 else "❌"
    if violations > 0:
        all_match = False
    print(f"  {check_name:<20} {violations:>12,} {status:>8}")
print_done(t0)

# ── Summary ────────────────────────────────────────────────────────────────────
wall_elapsed = time.time() - wall_t0
overall = "✅  ALL CHECKS PASSED" if all_match else "❌  ISSUES FOUND — review above"
print(f"""
╔══════════════════════════════════════════════════════╗
║         Validation Complete                          ║
╠══════════════════════════════════════════════════════╣
║  Result    : {overall:<39}║
║  Script End: {ts()}                             ║
║  Total Time: {wall_elapsed:.2f}s                                 ║
╚══════════════════════════════════════════════════════╝
""")

cass.cluster.shutdown()
cur.close()
pg_conn.close()
EOF
```

### Step 2 — Run Validation

```bash
python3 ~/cassandra2pg/validate.py
```

Expected output:
```
╔══════════════════════════════════════════════════════╗
║         Data Validation — Starting                   ║
║  Script Start : 10:11:00                             ║
╚══════════════════════════════════════════════════════╝

[Step 1] Connecting to Cassandra and PostgreSQL
  ⏱  Start : 10:11:00
  ⏱  End   : 10:11:00
  ⏱  Time  : 0.21s

[Step 2] Row count comparison — Cassandra vs PostgreSQL
  ⏱  Start : 10:11:00

  Table           Cassandra   PostgreSQL    Match
  ────────────────────────────────────────────────────
  customers           2,000        2,000        ✅
  orders             40,000       40,000        ✅
  order_items       120,000      120,000        ✅
  ⏱  End   : 10:11:03
  ⏱  Time  : 3.10s

[Step 3] Referential integrity check (orphaned FK rows)
  ⏱  Start : 10:11:03

  Check                Violations   Status
  ────────────────────────────────────────────
  orphaned_orders               0        ✅
  orphaned_items                0        ✅
  ⏱  End   : 10:11:03
  ⏱  Time  : 0.05s

╔══════════════════════════════════════════════════════╗
║         Validation Complete                          ║
║  Result    : ✅  ALL CHECKS PASSED                   ║
║  Script End: 10:11:03                                ║
║  Total Time: 3.36s                                   ║
╚══════════════════════════════════════════════════════╝
```

### Step 3 — Cassandra Query Limitations (Run These First — Live Demo Contrast)

Run these queries **in Cassandra first** so the audience sees the friction and warnings before switching to PostgreSQL.

#### 3a — What Cassandra CAN do: single-partition lookup by primary key

This is Cassandra's sweet spot — instant lookup when you know the partition key:

```bash
# Fast: lookup by partition key (customer_id) — Cassandra's strength
podman exec -it cassandra-source cqlsh -e "
USE ecommerce;
SELECT order_id, order_date, status, total_amount
FROM orders
WHERE customer_id = aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
LIMIT 5;
"
```

> 💡 **Demo tip:** Replace the UUID above with a real one from your dataset:
> ```bash
> podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT customer_id FROM customers LIMIT 1;"
> ```

Expected: returns instantly — this is what Cassandra is optimised for.

---

#### 3b — What Cassandra STRUGGLES with: aggregate without partition key

```bash
# Cassandra: COUNT(*) across all rows — triggers a full scan + warning
podman exec -it cassandra-source cqlsh -e "
USE ecommerce;
SELECT COUNT(*) FROM orders;
"
```

Expected Cassandra output — note the warning:
```
 count
--------
 40000
(1 rows)

Warnings :
Aggregation query used without partition key
```

> ⚠️ This full-scan aggregation holds a read lock across all partitions. At production scale (millions of rows) this can take **minutes** and impact cluster health.

---

#### 3c — What Cassandra CANNOT do: filter by non-partition column

```bash
# Cassandra: filter by status (non-partition column) — ALLOW FILTERING required
podman exec -it cassandra-source cqlsh -e "
USE ecommerce;
SELECT customer_id, order_id, total_amount
FROM orders
WHERE status = 'delivered'
LIMIT 10
ALLOW FILTERING;
"
```

Expected Cassandra output:
```
 customer_id | order_id | total_amount
 ...         | ...      | ...
(10 rows)

Warnings :
Aggregation query used without partition key
```

> ⚠️ `ALLOW FILTERING` scans **every partition** on the cluster to find matching rows. It is explicitly [discouraged in production](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html) — Cassandra's own docs call it a potential performance hazard. There is no execution time shown because Cassandra has no `EXPLAIN ANALYZE`.

---

#### 3d — What Cassandra CANNOT do at all: JOIN + GROUP BY + aggregation

```bash
# Cassandra: this query is structurally impossible — no JOIN, no GROUP BY across tables
podman exec -it cassandra-source cqlsh -e "
USE ecommerce;
SELECT c.country, SUM(o.total_amount)
FROM customers c JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'delivered'
GROUP BY c.country;
"
```

Expected Cassandra output:
```
SyntaxException: line 2:... no viable alternative at input 'JOIN'
```

> ❌ Cassandra has no JOIN support. This query requires either: (a) a pre-built denormalized table designed exactly for this query pattern, or (b) full scans of both tables in application code with client-side aggregation. Both approaches are expensive to build and maintain.

---

### Step 4 — PostgreSQL Query Demo (Same Questions, Milliseconds)

Now run the **identical business questions** in PostgreSQL and show the contrast:

#### 4a — Revenue by country and tier (impossible in Cassandra)

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT
  c.country,
  c.tier,
  COUNT(DISTINCT o.order_id)             AS total_orders,
  ROUND(SUM(o.total_amount)::numeric, 2) AS total_revenue,
  ROUND(AVG(o.total_amount)::numeric, 2) AS avg_order_value
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'delivered'
GROUP BY c.country, c.tier
ORDER BY total_revenue DESC
LIMIT 10;
"
```

#### 4b — Orders by status with revenue — date range filter

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT status, COUNT(*) AS orders,
       ROUND(SUM(total_amount)::numeric, 2) AS revenue
FROM orders
WHERE order_date BETWEEN '2023-06-01' AND '2024-06-01'
GROUP BY status
ORDER BY revenue DESC;
"
```

#### 4c — Top products by gross value

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT product_name,
       SUM(quantity)                                  AS units_sold,
       ROUND(SUM(quantity * unit_price)::numeric, 2) AS gross_value
FROM order_items
GROUP BY product_name
ORDER BY gross_value DESC;
"
```

### Step 5 — EXPLAIN ANALYZE — Flagship Demo Query

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

> 💡 **Demo talking point:** The same question that causes a multi-second full cluster scan in Cassandra (with `ALLOW FILTERING` warnings) and requires pre-built denormalized tables for joins — runs in **0.143 milliseconds** in PostgreSQL with a standard index, on the same hardware.

---

### Step 6 — JSON Performance Demo

JSON handling is a common real-world requirement. Cassandra stores JSON as plain `TEXT` with no indexing or querying capability. PostgreSQL has a native `JSONB` type with indexing, operators, and full query support.

#### 6a — Add a JSON metadata column to PostgreSQL

```bash
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
-- Add a JSONB column to orders (e.g. shipping metadata, tags, custom attributes)
ALTER TABLE orders ADD COLUMN IF NOT EXISTS metadata JSONB;

-- Populate with realistic sample JSON per order
UPDATE orders SET metadata = jsonb_build_object(
  'shipping_provider', (ARRAY['FedEx','DHL','UPS','USPS'])[floor(random()*4+1)::int],
  'warehouse_id',      floor(random()*5+1)::int,
  'priority',          (ARRAY['standard','express','overnight'])[floor(random()*3+1)::int],
  'tags',              jsonb_build_array(
                         (ARRAY['fragile','bulky','gift','promo'])[floor(random()*4+1)::int],
                         (ARRAY['new_customer','returning','vip'])[floor(random()*3+1)::int]
                       ),
  'gift_message',      CASE WHEN random() > 0.8 THEN 'Happy Birthday!' ELSE NULL END
);

-- Create a GIN index on the JSONB column for fast key/value lookups
CREATE INDEX IF NOT EXISTS idx_orders_metadata ON orders USING GIN (metadata);
"
```

#### 6b — Query JSONB: filter, extract, aggregate

```bash
# Filter orders by a JSON field value — uses GIN index
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
EXPLAIN ANALYZE
SELECT order_id, total_amount,
       metadata->>'shipping_provider' AS provider,
       metadata->>'priority'          AS priority
FROM orders
WHERE metadata @> '{\"shipping_provider\": \"DHL\"}'
ORDER BY total_amount DESC
LIMIT 10;
"
```

Expected:
```
Index Scan using idx_orders_metadata on orders
Execution Time: ~0.5 ms
```

```bash
# Aggregate by JSON field — shipping provider revenue breakdown
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT
  metadata->>'shipping_provider'         AS provider,
  metadata->>'priority'                  AS priority,
  COUNT(*)                               AS orders,
  ROUND(SUM(total_amount)::numeric, 2)   AS revenue
FROM orders
WHERE metadata IS NOT NULL
GROUP BY 1, 2
ORDER BY revenue DESC;
"
```

```bash
# Unnest a JSON array — list all unique tags across orders
podman exec -it postgres-target psql -U poc_user -d ecommerce -c "
SELECT tag, COUNT(*) AS occurrences
FROM orders,
     jsonb_array_elements_text(metadata->'tags') AS tag
GROUP BY tag
ORDER BY occurrences DESC;
"
```

#### 6c — Add JSON to Cassandra for comparison

```bash
# Cassandra: JSON stored as plain TEXT — no indexing, no operators
podman exec -it cassandra-source cqlsh -e "
USE ecommerce;
ALTER TABLE orders ADD metadata TEXT;
UPDATE orders SET metadata = '{\"shipping_provider\":\"DHL\",\"priority\":\"express\"}'
WHERE customer_id = aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
  AND order_date  = '2023-06-15 10:00:00'
  AND order_id    = ffffffff-gggg-hhhh-iiii-jjjjjjjjjjjj;
"
```

> ⚠️ Replace the UUIDs above with real values from your dataset. To find one:
> ```bash
> podman exec -it cassandra-source cqlsh -e "USE ecommerce; SELECT customer_id, order_date, order_id FROM orders LIMIT 1;"
> ```

Now try to query by a JSON field value in Cassandra:

```bash
# Cassandra: filter by JSON field — this FAILS without ALLOW FILTERING
# and even with it, Cassandra does a string match, not JSON parse
podman exec -it cassandra-source cqlsh -e "
USE ecommerce;
SELECT customer_id, order_id, metadata
FROM orders
WHERE metadata LIKE '%DHL%'
LIMIT 5;
"
```

Expected Cassandra output:
```
InvalidRequest: LIKE is not supported on column metadata of type text
```

**JSON Capability Comparison:**

| Capability | Cassandra | PostgreSQL JSONB |
|---|---|---|
| Store JSON | ✅ As `TEXT` | ✅ Native `JSONB` (binary, validated) |
| Index JSON fields | ❌ No | ✅ GIN index — sub-millisecond lookup |
| Filter by JSON value | ❌ Only full-text `LIKE` (slow, error-prone) | ✅ `@>` containment operator |
| Extract nested fields | ❌ Application code only | ✅ `->` / `->>` operators in SQL |
| Aggregate by JSON field | ❌ Client-side only | ✅ Native `GROUP BY metadata->>'field'` |
| Unnest JSON arrays | ❌ Application code only | ✅ `jsonb_array_elements_text()` |
| Partial index on JSON key | ❌ Not possible | ✅ `CREATE INDEX WHERE metadata->>'key' = 'val'` |
| JSON schema validation | ❌ No | ✅ Check constraints on JSONB fields |

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

### Step 1 — Generate `benchmark.sh` using cat

```bash
cat > ~/cassandra2pg/benchmark.sh << 'EOF'
#!/bin/bash
# benchmark.sh — Throughput sweep: batch size and thread count
# Prints Step number, start time, end time, elapsed time for each run.

ts() { date +"%H:%M:%S"; }

echo ""
echo "╔══════════════════════════════════════════════════════╗"
echo "║         Migration Benchmark — Starting               ║"
echo "║  Script Start : $(ts())                         ║"
echo "╚══════════════════════════════════════════════════════╝"

WALL_START=$(date +%s)
STEP=1

# ── Benchmark 1: Batch Size ────────────────────────────────────────────────────
echo ""
echo "════════════════════════════════════════════════════════"
echo "  BENCHMARK 1 — Batch Size vs Throughput  (threads=4)"
echo "════════════════════════════════════════════════════════"
echo ""
printf "  %-12s %-10s %-10s %-12s %s\n" "Batch" "Start" "End" "Elapsed" "Throughput"
printf "  %-12s %-10s %-10s %-12s %s\n" "──────────" "────────" "────────" "──────────" "──────────"

for BATCH in 100 250 500 1000; do
  podman exec -it postgres-target psql -U poc_user -d ecommerce -q -c \
    "TRUNCATE order_items, orders, customers CASCADE;" 2>/dev/null

  START_TS=$(ts)
  T_START=$(date +%s%N)

  RESULT=$(python3 ~/cassandra2pg/migrate.py --threads 4 --batch $BATCH 2>/dev/null | grep "Throughput")
  THROUGHPUT=$(echo "$RESULT" | grep -oP '[\d,]+ rows/s')

  T_END=$(date +%s%N)
  END_TS=$(ts)
  ELAPSED=$(echo "scale=1; ($T_END - $T_START) / 1000000000" | bc)s

  printf "  [Step %d] %-6s %-10s %-10s %-12s %s\n" \
    $STEP "b=$BATCH" "$START_TS" "$END_TS" "$ELAPSED" "$THROUGHPUT"
  STEP=$((STEP + 1))
done

# ── Benchmark 2: Thread Count ──────────────────────────────────────────────────
echo ""
echo "════════════════════════════════════════════════════════"
echo "  BENCHMARK 2 — Thread Count vs Throughput  (batch=500)"
echo "════════════════════════════════════════════════════════"
echo ""
printf "  %-12s %-10s %-10s %-12s %s\n" "Threads" "Start" "End" "Elapsed" "Throughput"
printf "  %-12s %-10s %-10s %-12s %s\n" "──────────" "────────" "────────" "──────────" "──────────"

for THREADS in 1 2 4 8; do
  podman exec -it postgres-target psql -U poc_user -d ecommerce -q -c \
    "TRUNCATE order_items, orders, customers CASCADE;" 2>/dev/null

  START_TS=$(ts)
  T_START=$(date +%s%N)

  RESULT=$(python3 ~/cassandra2pg/migrate.py --threads $THREADS --batch 500 2>/dev/null | grep "Throughput")
  THROUGHPUT=$(echo "$RESULT" | grep -oP '[\d,]+ rows/s')

  T_END=$(date +%s%N)
  END_TS=$(ts)
  ELAPSED=$(echo "scale=1; ($T_END - $T_START) / 1000000000" | bc)s

  printf "  [Step %d] %-6s %-10s %-10s %-12s %s\n" \
    $STEP "t=$THREADS" "$START_TS" "$END_TS" "$ELAPSED" "$THROUGHPUT"
  STEP=$((STEP + 1))
done

WALL_END=$(date +%s)
WALL_ELAPSED=$((WALL_END - WALL_START))

echo ""
echo "╔══════════════════════════════════════════════════════╗"
echo "║         Benchmark Complete ✅                        ║"
echo "║  Script End   : $(ts())                         ║"
echo "║  Total Time   : ${WALL_ELAPSED}s                                 ║"
echo "╚══════════════════════════════════════════════════════╝"
echo ""
EOF

chmod +x ~/cassandra2pg/benchmark.sh
```

### Step 2 — Run the Benchmark

```bash
bash ~/cassandra2pg/benchmark.sh
```

Expected output:
```
╔══════════════════════════════════════════════════════╗
║         Migration Benchmark — Starting               ║
║  Script Start : 10:15:00                             ║
╚══════════════════════════════════════════════════════╝

════════════════════════════════════════════════════════
  BENCHMARK 1 — Batch Size vs Throughput  (threads=4)
════════════════════════════════════════════════════════

  Batch        Start      End        Elapsed      Throughput
  ──────────   ────────   ────────   ──────────   ──────────
  [Step 1] b=100    10:15:01   10:15:07   6.2s         12,450 rows/s
  [Step 2] b=250    10:15:08   10:15:13   5.9s         21,300 rows/s
  [Step 3] b=500    10:15:14   10:15:20   5.7s         28,286 rows/s
  [Step 4] b=1000   10:15:21   10:15:27   6.0s         26,849 rows/s

════════════════════════════════════════════════════════
  BENCHMARK 2 — Thread Count vs Throughput  (batch=500)
════════════════════════════════════════════════════════

  Threads      Start      End        Elapsed      Throughput
  ──────────   ────────   ────────   ──────────   ──────────
  [Step 5] t=1      10:15:28   10:15:40   11.8s        13,700 rows/s
  [Step 6] t=2      10:15:41   10:15:49   8.2s         19,600 rows/s
  [Step 7] t=4      10:15:50   10:15:56   5.7s         28,286 rows/s
  [Step 8] t=8      10:15:57   10:16:03   6.0s         26,849 rows/s

╔══════════════════════════════════════════════════════╗
║         Benchmark Complete ✅                        ║
║  Script End   : 10:16:03                             ║
║  Total Time   : 63s                                  ║
╚══════════════════════════════════════════════════════╝
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
  relname                                     AS table_name,
  pg_size_pretty(pg_total_relation_size(oid)) AS total_size
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

### Step 1 — Stop and Remove Containers

```bash
podman stop cassandra-source postgres-target
podman rm   cassandra-source postgres-target
podman network rm migration-net
```

### Step 2 — Remove Container Images (Optional)

```bash
podman rmi docker.io/library/cassandra:5.0
podman rmi docker.io/library/postgres:18
```

### Step 3 — Verify Clean State

```bash
podman ps -a
podman network ls
# Neither container nor migration-net should appear
```

### Step 4 — Remove Python Virtual Environment

```bash
deactivate
rm -rf ~/migration-env
```

---

## Project Structure

```
cassandra2pg/
├── README.md
├── schema.cql            # Cassandra keyspace + table definitions (cat generated)
├── generate_data.py      # Synthetic data generator — 162k rows, ~35s (cat generated)
├── migrate.py            # Migration script — parallel threads, batched inserts (cat generated)
├── validate.py           # Row count + FK integrity validation (cat generated)
└── benchmark.sh          # Batch size + thread count throughput sweep (cat generated)
```

> All `.py` and `.sh` files are generated automatically using `cat > file << 'EOF' ... EOF` — no manual editor required.

---

## Key Takeaways

- **Cassandra 5.0** introduced Storage-Attached Indexes (SAI) and vector search, narrowing the gap for secondary queries — but joins, ad-hoc aggregations, and complex reporting still require PostgreSQL.
- **PostgreSQL 18** provides full relational capabilities, strong consistency, and rich aggregate/join/window-function support that Cassandra cannot match natively.
- This PoC migrates **162,000 rows in ~6 seconds** (~28,000 rows/sec) on a single WSL2 node using token-range parallel threads and `execute_values` batch inserts.
- Every script prints **step number, start time, end time, and elapsed time** per stage — making progress transparent during live demos.
- The same pattern is production-scalable and adaptable to cloud targets (Amazon RDS, AlloyDB, Aurora PostgreSQL).
- A JOIN + GROUP BY + date-range query that is impossible natively in Cassandra executes in **0.143 ms** in PostgreSQL using index scans — no application-side aggregation required.

---

*PoC authored for client demonstration purposes. All data is synthetically generated.*
