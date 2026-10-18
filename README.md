# goit-rdb-hw-05

ДЗ 5: CTE, recursive CTE і Materialized View на датасеті Olist.

## Датасет

Olist Brazilian E-commerce Public Dataset, ~100 тис. замовлень з бразильського маркетплейсу Olist за 2016-2018 роки.

Джерело: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

Використовую 5 файлів, вони лежать в `data/olist/`:

- `olist_customers_dataset.csv` (99441 рядків)
- `olist_orders_dataset.csv` (99441)
- `olist_order_items_dataset.csv` (112650)
- `olist_products_dataset.csv` (32951)
- `olist_sellers_dataset.csv` (3095)

## Як отримати файли

CSV вже є в репозиторії в папці `data/olist/` (~43 MB), ноутбук бере їх звідти.

Якщо папки нема (наприклад, ноутбук відкрили в Colab без клонування репо), то датасет завантажується з Kaggle через `kagglehub.dataset_download('olistbr/brazilian-ecommerce')`. Датасет публічний, тому Kaggle credentials не потрібні.

## Raw і typed шар

- raw (`olist_*_raw`) - CSV як є, завантажені через `pandas.to_sql`. Це тільки проміжний шар.
- typed (`olist_customers`, `olist_orders`, `olist_order_items`, `olist_products`, `olist_sellers`) - таблиці з PRIMARY KEY, FOREIGN KEY, NOT NULL і CHECK. Дані переносяться з raw через `INSERT ... SELECT` з приведенням типів. В CSV колонки `product_name_lenght` і `product_description_lenght` написані з помилкою, в typed-таблиці перейменував їх на `product_name_length` і `product_description_length`.
- `hw5_referrals` - синтетична таблиця рефералів (ланцюжок з 10 клієнтів) для recursive CTE, бо в Olist такої нема.

## CTE stages

- stage 0 `reference_date` - дата відліку MAX(order_purchase_timestamp) + 1 день (2018-10-18)
- stage 1 `stage1_orders_total` - сума кожного замовлення (price + freight_value)
- stage 2 `stage2_rfm` - recency_days, frequency, monetary, first_order_date
- stage 3 `stage3_cohort` - cohort_quarter
- stage 4 `stage4_rolling_aov` - rolling AVG за 30 днів через window function
- stage 5 `stage5_cancel_rate` - частка скасованих замовлень
- stage 6 `stage6_category_revenue` - виручка клієнта по категоріях
- stage 7 `stage7_preferred_category` - топ-1 категорія через ROW_NUMBER()

Весь pipeline збережений в MV `mv_olist_customer_features`, на ній UNIQUE INDEX по customer_id і оновлення через `REFRESH MATERIALIZED VIEW CONCURRENTLY`.

## Запуск

В Colab: відкрити `dz_topic_5_Skalskyi.ipynb` і зробити Runtime -> Restart and run all. PostgreSQL піднімається прямо в ноутбуці через pgserver, бібліотеки ставить перша клітинка.

Локально:

```bash
python -m venv .venv
.venv\Scripts\activate          # на Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook dz_topic_5_Skalskyi.ipynb
```

На Windows venv треба створювати в папці без кирилиці в шляху, інакше `initdb` з pgserver падає з помилкою `invalid byte sequence for encoding "UTF8"`.
