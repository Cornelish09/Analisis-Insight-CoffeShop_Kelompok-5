# Coffee Shop Insights — Output dari Data Asli (Pakai Kode Anda)

Berikut 6 insight dari dataset **CoffeeShop_Dataset.db** menggunakan **pendekatan & variabel** yang selaras dengan kode Anda
(`sales reciepts`, join ke `product`, join ke `customer` + `generations`, `sales_prod`, dll.).
Setiap insight berisi: **Tujuan**, **Manfaat**, **Langkah**, **Kode**, **Output (riil)**.
### Setup & Preprocessing (sesuai kode Anda)
```python

import sqlite3, pandas as pd, numpy as np

con = sqlite3.connect('CoffeeShop_Dataset.db')
sales_rec   = pd.read_sql_query('SELECT * FROM "sales reciepts";', con)
product     = pd.read_sql_query('SELECT * FROM "product";', con)
customer    = pd.read_sql_query('SELECT * FROM "customer";', con)
generations = pd.read_sql_query('SELECT * FROM "generations";', con)
sales_outlet= pd.read_sql_query('SELECT * FROM "sales outlet";', con)
con.close()

# Normalisasi kolom
def norm(df):
    df = df.copy()
    df.columns = [str(c).strip().lower().replace(' ','_').replace('-','_') for c in df.columns]
    return df

sales_rec   = norm(sales_rec)
product     = norm(product)
customer    = norm(customer)
generations = norm(generations)
sales_outlet= norm(sales_outlet)

# Datetime, date, hour
sales_rec['transaction_datetime'] = pd.to_datetime(sales_rec['transaction_date'] + ' ' + sales_rec['transaction_time'])
sales_rec['date'] = sales_rec['transaction_datetime'].dt.date
sales_rec['hour'] = sales_rec['transaction_datetime'].dt.hour

# Klasifikasi minuman
def beverage_type(r):
    text = ' '.join([str(r.get(c,'')) for c in ['product_group','product_category','product_type','product']]).lower()
    if 'tea' in text or 'teh' in text: return 'Tea'
    if ('coffee' in text) or any(k in text for k in ['espresso','latte','americano','cappuccino','mocha']): return 'Coffee'
    return 'Other'

product['bev_type'] = product.apply(beverage_type, axis=1)

# Join sesuai alur Anda
sales_prod   = sales_rec.merge(product[['product_id','product','product_group','product_category','product_type','bev_type']], on='product_id', how='left')
customer_gen = customer.merge(generations, on='birth_year', how='left')

```

---

## 1) Generasi Z lebih suka teh atau kopi?

**Tujuan**  
- Mengetahui preferensi minuman Gen Z (Coffee vs Tea).

**Manfaat**  
- Bahan keputusan stok & promo spesifik segmen Gen Z.

**Langkah**  
1) Join sales→product→customer+generations  2) Filter Gen Z  3) Agregasi quantity per `bev_type`.

**Kode (potongan relevan)**
```python
sp1 = sales_prod.merge(customer_gen[['customer_id','generation']], on='customer_id', how='left')
genz = sp1[sp1['generation'].str.lower().str.contains('gen z', na=False)]
ins1_counts = (genz.groupby('bev_type')['quantity'].sum()
                    .reset_index().sort_values('quantity', ascending=False))
ins1_counts
```

**Output (riil)**
| bev_type   |   quantity |
|:-----------|-----------:|
| Coffee     |       2565 |
| Tea        |       2186 |
| Other      |       1445 |

---

## 2) Total Transaksi & Total Revenue

**Tujuan**  
- Mengukur total aktivitas penjualan dan pendapatan.

**Manfaat**  
- Monitoring kinerja, baseline target/forecast.

**Langkah**  
Hitung unik `transaction_id` dan jumlahkan `line_item_amount`.

**Kode (potongan relevan)**
```python
total_txn = sales_rec['transaction_id'].nunique()
total_rev = sales_rec['line_item_amount'].sum()
pd.DataFrame({'metric':['Total Transactions','Total Revenue'],
              'value':[total_txn, total_rev]})
```

**Output (riil)**
| metric             |   value |
|:-------------------|--------:|
| Total Transactions |    4203 |
| Total Revenue      |  233636 |

---

## 3) Top-5 Produk Terlaris by Quantity

**Tujuan**  
- Mengidentifikasi 5 produk paling laku (unit terjual).

**Manfaat**  
- Fokus replenishment & kampanye produk unggulan.

**Langkah**  
Join `sales_prod`, agregasi `sum(quantity)` per `product`, sort desc & ambil 5.

**Kode (potongan relevan)**
```python
ins3_top5 = (sales_prod.groupby('product')['quantity'].sum()
              .sort_values(ascending=False).head(5).reset_index()
              .rename(columns={'quantity':'total_quantity'}))
ins3_top5
```

**Output (riil)**
| product                 |   total_quantity |
|:------------------------|-----------------:|
| Earl Grey Rg            |             1558 |
| Dark chocolate Lg       |             1546 |
| Latte                   |             1531 |
| Morning Sunrise Chai Rg |             1513 |
| Ethiopia Rg             |             1506 |

---

## 4) Jumlah Transaksi per Tanggal

**Tujuan**  
- Melihat tren harian & deteksi hari sibuk/lesu.

**Manfaat**  
- Penjadwalan staf & evaluasi promo harian.

**Langkah**  
Ekstrak `date`, hitung unik transaksi per tanggal, urutkan kronologis.

**Kode (potongan relevan)**
```python
ins4_per_date = (sales_rec.groupby('date')['transaction_id'].nunique()
                 .reset_index(name='transactions').sort_values('date'))
ins4_per_date
```

**Output (riil)**
| date       |   transactions |
|:-----------|---------------:|
| 2019-04-01 |           1193 |
| 2019-04-02 |           1171 |
| 2019-04-03 |           1224 |
| 2019-04-04 |           1159 |
| 2019-04-05 |           1172 |
| 2019-04-06 |           1077 |
| 2019-04-07 |            514 |
| 2019-04-08 |            480 |
| 2019-04-09 |            505 |
| 2019-04-10 |            487 |
| 2019-04-11 |            452 |
| 2019-04-12 |            489 |
| 2019-04-13 |            509 |
| 2019-04-14 |            477 |
| 2019-04-15 |           1132 |
| 2019-04-16 |           1153 |
| 2019-04-17 |           1087 |
| 2019-04-18 |           1098 |
| 2019-04-19 |           1116 |
| 2019-04-20 |            751 |
| 2019-04-21 |            768 |
| 2019-04-22 |           1223 |
| 2019-04-23 |           1234 |
| 2019-04-24 |           1303 |
| 2019-04-25 |           1236 |
| 2019-04-26 |           1256 |
| 2019-04-27 |            971 |
| 2019-04-28 |           1033 |
| 2019-04-29 |            961 |

---

## 5) Mencari Jam Puncak (Peak Hour) untuk Setiap Outlet

**Tujuan**  
- Menentukan jam tersibuk per outlet (berdasarkan jumlah order unik).

**Manfaat**  
- Optimasi shift & jam promo.

**Langkah**  
Groupby `(sales_outlet_id,hour)` → `nunique(transaction_id)` → ambil baris dengan `orders` maksimum per outlet.

**Kode (potongan relevan)**
```python
orders = (sales_rec.groupby(['sales_outlet_id','hour'])['transaction_id']
          .nunique().reset_index(name='orders'))
peak_per_outlet = (orders.sort_values(['sales_outlet_id','orders'], ascending=[True,False])
                          .groupby('sales_outlet_id').head(1)
                          .sort_values('sales_outlet_id').reset_index(drop=True))
peak_per_outlet
```

**Output (riil)**
|   sales_outlet_id |   hour |   orders |
|------------------:|-------:|---------:|
|                 3 |     16 |      832 |
|                 5 |      7 |      859 |
|                 8 |      8 |      982 |

---

## 6) Tampilkan 5 Pastry Terlaris

**Tujuan**  
- Mengetahui pastry paling laku.

**Manfaat**  
- Atur produksi harian & bundling pastry + minuman.

**Langkah**  
Tag pastry via kata kunci, filter transaksi pastry, agregasi quantity, ambil Top-5.

**Kode (potongan relevan)**
```python
def is_pastry(r):
    text = ' '.join([str(r.get(c,'')) for c in ['product_group','product_category','product_type','product']]).lower()
    return any(k in text for k in ['pastry','bakery','croissant','scone','muffin','cookie','cake','bread','biscotti'])

product['is_pastry'] = product.apply(is_pastry, axis=1)
sp_pastry = sales_rec.merge(product[['product_id','product','is_pastry']], on='product_id', how='left')
ins6_top5 = (sp_pastry[sp_pastry['is_pastry']==True]
             .groupby('product')['quantity'].sum()
             .sort_values(ascending=False).head(5).reset_index()
             .rename(columns={'quantity':'total_quantity'}))
ins6_top5
```

**Output (riil)**
| product             |   total_quantity |
|:--------------------|-----------------:|
| Chocolate Croissant |             1040 |
| Ginger Scone        |              872 |
| Jumbo Savory Scone  |              685 |
| Cranberry Scone     |              684 |
| Hazelnut Biscotti   |              676 |
