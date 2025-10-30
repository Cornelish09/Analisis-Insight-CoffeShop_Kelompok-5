# Coffee Shop Insights — Analisis & Kode (Siap Tempel)

Repo ini berisi **6 insight** dari dataset Coffee Shop. Setiap bagian punya: **Tujuan**, **Manfaat Bisnis**, **Langkah (pipeline)**, **Kode Python (Pandas/SQLite)**, dan **Output**.

> Catatan eksekusi: ⚠️ Database `CoffeeShop_Dataset.db` tidak ditemukan di lingkungan ini, sehingga output di bawah adalah **contoh (DEMO)** dari data sintetis. Saat dijalankan pada dataset Anda, hasil akan menyesuaikan data sebenarnya.


---

## 1) Generasi Z lebih suka teh atau kopi?

**Tujuan**  
- Mengetahui preferensi minuman Gen Z (Coffee vs Tea) berdasarkan transaksi.

**Manfaat Bisnis**  
- Menentukan stok & promo spesifik segmen Gen Z; optimasi menu dan bundling.

**Langkah (Pipeline)**  
1) Gabungkan sales–product–customer
2) Labelkan `bev_type` (Coffee/Tea)
3) Derivasi generasi dari `birth_year`
4) Agregasi kuantitas Gen Z per `bev_type` dan bandingkan.

**Kode (Python)**
```python
import sqlite3, pandas as pd, numpy as np
con = sqlite3.connect('CoffeeShop_Dataset.db')

# --- Muat tabel (otomatis deteksi nama yang mirip) ---
tables = pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table';", con)['name'].str.lower().tolist()
def pick(patterns):
    for name in tables:
        if any(p in name for p in patterns): return name
    return None

sales_tbl = pick(["sales","receipt","transaction"])
prod_tbl  = pick(["product","item","sku"])
cust_tbl  = pick(["customer","member","client"])

sales = pd.read_sql_query(f"SELECT * FROM '{sales_tbl}'", con)
product = pd.read_sql_query(f"SELECT * FROM '{prod_tbl}'", con)
customer = pd.read_sql_query(f"SELECT * FROM '{cust_tbl}'", con)

# --- Normalisasi kolom ---
for df in (sales, product, customer):
    df.columns = [c.strip().lower().replace(" ","_") for c in df.columns]

# --- Tanggal, jam ---
dt_col = next((c for c in ["transaction_datetime","transaction_time","transaction_date","created_at","datetime"] if c in sales.columns), None)
sales["__dt"] = pd.to_datetime(sales[dt_col])
sales["date"] = sales["__dt"].dt.date
sales["hour"] = sales["__dt"].dt.hour

# --- Klasifikasi minuman Coffee vs Tea ---
def beverage_type(row):
    text = " ".join([str(row.get(c,"")) for c in ["product_group","product_category","product_type","product"]]).lower()
    if "coffee" in text or any(k in text for k in ["espresso","latte","americano","cappuccino"]): return "Coffee"
    if "tea" in text or "teh" in text: return "Tea"
    return "Other"

product["bev_type"] = product.apply(beverage_type, axis=1)

# --- Label generasi ---
def generation_from_year(y):
    try: y = int(y)
    except: return None
    if 1997 <= y <= 2012: return "Gen Z"
    if 1981 <= y <= 1996: return "Millennial"
    if 1965 <= y <= 1980: return "Gen X"
    if 1946 <= y <= 1964: return "Baby Boomer"
    return "Other"

customer["generation"] = customer.get("birth_year", pd.Series([None]*len(customer))).apply(generation_from_year)

# --- Hitung preferensi ---
df = (sales.merge(product[["product_id","bev_type"]], on="product_id", how="left")
           .merge(customer[["customer_id","generation"]], on="customer_id", how="left"))
genz = df[df["generation"]=="Gen Z"]
ins1_counts = genz.groupby("bev_type")["quantity"].sum().reset_index().sort_values("quantity", ascending=False)
ins1_counts
```

**Output**
| bev_type   |   quantity |
|:-----------|-----------:|
| Other      |        530 |
| Coffee     |        517 |
| Tea        |        381 |


---

## 2) Total Transaksi & Total Revenue

**Tujuan**  
- Mengukur besaran aktivitas penjualan dan pendapatan periode data.

**Manfaat Bisnis**  
- Monitoring kinerja harian/mingguan; dasar target & forecast; ukuran campaign.

**Langkah (Pipeline)**  
1) Hitung unik `transaction_id` sebagai total transaksi
2) Jumlahkan `line_item_amount` atau `quantity*unit_price` sebagai revenue.

**Kode (Python)**
```python
import sqlite3, pandas as pd, numpy as np
con = sqlite3.connect('CoffeeShop_Dataset.db')

# --- Muat tabel (otomatis deteksi nama yang mirip) ---
tables = pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table';", con)['name'].str.lower().tolist()
def pick(patterns):
    for name in tables:
        if any(p in name for p in patterns): return name
    return None

sales_tbl = pick(["sales","receipt","transaction"])
prod_tbl  = pick(["product","item","sku"])
cust_tbl  = pick(["customer","member","client"])

sales = pd.read_sql_query(f"SELECT * FROM '{sales_tbl}'", con)
product = pd.read_sql_query(f"SELECT * FROM '{prod_tbl}'", con)
customer = pd.read_sql_query(f"SELECT * FROM '{cust_tbl}'", con)

# --- Normalisasi kolom ---
for df in (sales, product, customer):
    df.columns = [c.strip().lower().replace(" ","_") for c in df.columns]

# --- Tanggal, jam ---
dt_col = next((c for c in ["transaction_datetime","transaction_time","transaction_date","created_at","datetime"] if c in sales.columns), None)
sales["__dt"] = pd.to_datetime(sales[dt_col])
sales["date"] = sales["__dt"].dt.date
sales["hour"] = sales["__dt"].dt.hour

# --- Total transaksi & revenue ---
if "line_item_amount" in sales.columns:
    total_revenue = sales["line_item_amount"].sum()
else:
    total_revenue = (sales["quantity"] * sales["unit_price"]).sum()

total_txn = sales["transaction_id"].nunique()
pd.DataFrame({"metric":["Total Transactions","Total Revenue"], "value":[total_txn, total_revenue]})
```

**Output**
| metric             |    value |
|:-------------------|---------:|
| Total Transactions |      993 |
| Total Revenue      | 55171000 |


---

## 3) Top-5 Produk Terlaris by Quantity

**Tujuan**  
- Mengidentifikasi 5 produk paling laku berdasarkan unit terjual.

**Manfaat Bisnis**  
- Fokus replenishment stok; highlight produk di kampanye; optimasi display.

**Langkah (Pipeline)**  
1) Join sales–product untuk dapatkan nama produk
2) Agregasi `sum(quantity)` per produk
3) Urutkan desc dan ambil Top-5.

**Kode (Python)**
```python
import sqlite3, pandas as pd, numpy as np
con = sqlite3.connect('CoffeeShop_Dataset.db')

# --- Muat tabel (otomatis deteksi nama yang mirip) ---
tables = pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table';", con)['name'].str.lower().tolist()
def pick(patterns):
    for name in tables:
        if any(p in name for p in patterns): return name
    return None

sales_tbl = pick(["sales","receipt","transaction"])
prod_tbl  = pick(["product","item","sku"])
cust_tbl  = pick(["customer","member","client"])

sales = pd.read_sql_query(f"SELECT * FROM '{sales_tbl}'", con)
product = pd.read_sql_query(f"SELECT * FROM '{prod_tbl}'", con)
customer = pd.read_sql_query(f"SELECT * FROM '{cust_tbl}'", con)

# --- Normalisasi kolom ---
for df in (sales, product, customer):
    df.columns = [c.strip().lower().replace(" ","_") for c in df.columns]

# --- Tanggal, jam ---
dt_col = next((c for c in ["transaction_datetime","transaction_time","transaction_date","created_at","datetime"] if c in sales.columns), None)
sales["__dt"] = pd.to_datetime(sales[dt_col])
sales["date"] = sales["__dt"].dt.date
sales["hour"] = sales["__dt"].dt.hour

# --- Top-5 produk terlaris (berdasarkan quantity) ---
df = sales.merge(product[["product_id","product"]], on="product_id", how="left")
top5_products = (df.groupby("product")["quantity"].sum()
                   .sort_values(ascending=False).head(5).reset_index()
                   .rename(columns={"quantity":"total_quantity"}))
top5_products
```

**Output**
| product   |   total_quantity |
|:----------|-----------------:|
| Black Tea |              322 |
| Green Tea |              315 |
| Americano |              300 |
| Muffin    |              278 |
| Latte     |              272 |


---

## 4) Jumlah Transaksi per Tanggal

**Tujuan**  
- Melihat tren transaksi harian untuk mendeteksi pola & outlier.

**Manfaat Bisnis**  
- Penjadwalan staf; evaluasi promo harian; deteksi hari sibuk/lesu.

**Langkah (Pipeline)**  
1) Ekstrak `date` dari timestamp
2) Hitung unik transaksi per `date`
3) Sortir kronologis dan plot bila perlu.

**Kode (Python)**
```python
import sqlite3, pandas as pd, numpy as np
con = sqlite3.connect('CoffeeShop_Dataset.db')

# --- Muat tabel (otomatis deteksi nama yang mirip) ---
tables = pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table';", con)['name'].str.lower().tolist()
def pick(patterns):
    for name in tables:
        if any(p in name for p in patterns): return name
    return None

sales_tbl = pick(["sales","receipt","transaction"])
prod_tbl  = pick(["product","item","sku"])
cust_tbl  = pick(["customer","member","client"])

sales = pd.read_sql_query(f"SELECT * FROM '{sales_tbl}'", con)
product = pd.read_sql_query(f"SELECT * FROM '{prod_tbl}'", con)
customer = pd.read_sql_query(f"SELECT * FROM '{cust_tbl}'", con)

# --- Normalisasi kolom ---
for df in (sales, product, customer):
    df.columns = [c.strip().lower().replace(" ","_") for c in df.columns]

# --- Tanggal, jam ---
dt_col = next((c for c in ["transaction_datetime","transaction_time","transaction_date","created_at","datetime"] if c in sales.columns), None)
sales["__dt"] = pd.to_datetime(sales[dt_col])
sales["date"] = sales["__dt"].dt.date
sales["hour"] = sales["__dt"].dt.hour

# --- Jumlah transaksi per tanggal ---
txn_per_date = (sales.groupby("date")["transaction_id"]
                    .nunique().reset_index(name="transactions")
                    .sort_values("date"))
txn_per_date
```

**Output**
| date       |   transactions |
|:-----------|---------------:|
| 2025-10-01 |            117 |
| 2025-10-02 |             85 |
| 2025-10-03 |            111 |
| 2025-10-04 |            119 |
| 2025-10-05 |             73 |
| 2025-10-06 |            103 |
| 2025-10-07 |             79 |
| 2025-10-08 |            108 |
| 2025-10-09 |            113 |
| 2025-10-10 |             85 |


---

## 5) Jam Puncak (Peak Hour) per Outlet

**Tujuan**  
- Mengetahui jam tersibuk tiap outlet berdasarkan jumlah order.

**Manfaat Bisnis**  
- Shift & staffing tepat; penentuan jam promo; optimasi kapasitas kasir/barista.

**Langkah (Pipeline)**  
1) Ekstrak `hour` dari timestamp
2) Hitung unik transaksi per `(outlet,hour)`
3) Ambil baris dengan `orders` maksimum per outlet.

**Kode (Python)**
```python
import sqlite3, pandas as pd, numpy as np
con = sqlite3.connect('CoffeeShop_Dataset.db')

# --- Muat tabel (otomatis deteksi nama yang mirip) ---
tables = pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table';", con)['name'].str.lower().tolist()
def pick(patterns):
    for name in tables:
        if any(p in name for p in patterns): return name
    return None

sales_tbl = pick(["sales","receipt","transaction"])
prod_tbl  = pick(["product","item","sku"])
cust_tbl  = pick(["customer","member","client"])

sales = pd.read_sql_query(f"SELECT * FROM '{sales_tbl}'", con)
product = pd.read_sql_query(f"SELECT * FROM '{prod_tbl}'", con)
customer = pd.read_sql_query(f"SELECT * FROM '{cust_tbl}'", con)

# --- Normalisasi kolom ---
for df in (sales, product, customer):
    df.columns = [c.strip().lower().replace(" ","_") for c in df.columns]

# --- Tanggal, jam ---
dt_col = next((c for c in ["transaction_datetime","transaction_time","transaction_date","created_at","datetime"] if c in sales.columns), None)
sales["__dt"] = pd.to_datetime(sales[dt_col])
sales["date"] = sales["__dt"].dt.date
sales["hour"] = sales["__dt"].dt.hour

# --- Jam puncak (peak hour) per outlet ---
orders = (sales.groupby(["sales_outlet_id","hour"])["transaction_id"]
               .nunique().reset_index(name="orders"))
peak_per_outlet = (orders.sort_values(["sales_outlet_id","orders"], ascending=[True,False])
                          .groupby("sales_outlet_id").head(1)
                          .sort_values("sales_outlet_id").reset_index(drop=True))
peak_per_outlet
```

**Output**
|   sales_outlet_id |   hour |   orders |
|------------------:|-------:|---------:|
|               101 |     13 |       26 |
|               102 |     17 |       31 |
|               103 |     11 |       32 |


---

## 6) Top-5 Pastry Terlaris

**Tujuan**  
- Mengetahui pastry paling laku untuk pengelolaan bakery counter.

**Manfaat Bisnis**  
- Atur produksi harian; penempatan display; bundling minuman + pastry favorit.

**Langkah (Pipeline)**  
1) Tandai item pastry via kata kunci pada metadata produk
2) Filter transaksi pastry
3) Agregasi `sum(quantity)` per pastry dan ambil Top-5.

**Kode (Python)**
```python
import sqlite3, pandas as pd, numpy as np
con = sqlite3.connect('CoffeeShop_Dataset.db')

# --- Muat tabel (otomatis deteksi nama yang mirip) ---
tables = pd.read_sql_query("SELECT name FROM sqlite_master WHERE type='table';", con)['name'].str.lower().tolist()
def pick(patterns):
    for name in tables:
        if any(p in name for p in patterns): return name
    return None

sales_tbl = pick(["sales","receipt","transaction"])
prod_tbl  = pick(["product","item","sku"])
cust_tbl  = pick(["customer","member","client"])

sales = pd.read_sql_query(f"SELECT * FROM '{sales_tbl}'", con)
product = pd.read_sql_query(f"SELECT * FROM '{prod_tbl}'", con)
customer = pd.read_sql_query(f"SELECT * FROM '{cust_tbl}'", con)

# --- Normalisasi kolom ---
for df in (sales, product, customer):
    df.columns = [c.strip().lower().replace(" ","_") for c in df.columns]

# --- Tanggal, jam ---
dt_col = next((c for c in ["transaction_datetime","transaction_time","transaction_date","created_at","datetime"] if c in sales.columns), None)
sales["__dt"] = pd.to_datetime(sales[dt_col])
sales["date"] = sales["__dt"].dt.date
sales["hour"] = sales["__dt"].dt.hour

# --- Top-5 Pastry terlaris ---
def is_pastry(row):
    text = " ".join([str(row.get(c,"")) for c in ["product_group","product_category","product_type","product"]]).lower()
    return any(k in text for k in ["pastry","bakery","croissant","scone","muffin","cookie","cake","bread","biscotti"])

product["is_pastry"] = product.apply(is_pastry, axis=1)
df = sales.merge(product[["product_id","product","is_pastry"]], on="product_id", how="left")
top5_pastry = (df[df["is_pastry"]==True].groupby("product")["quantity"].sum()
                 .sort_values(ascending=False).head(5).reset_index()
                 .rename(columns={"quantity":"total_quantity"}))
top5_pastry
```

**Output**
| product    |   total_quantity |
|:-----------|-----------------:|
| Muffin     |              278 |
| Croissant  |              222 |
| Cookie     |              179 |
| Cheesecake |              110 |
| Scone      |              103 |

