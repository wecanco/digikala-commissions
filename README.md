# راهنمای دیتابیس محصولات دیجیکالا

## معرفی
این دیتابیس شامل اطلاعات کامل محصولات و دسته‌بندی‌های دیجیکالا به همراه داده‌های کارمزد و ساختار سلسله‌مراتبی دسته‌بندی‌ها می‌باشد.

## ساختار دیتابیس

### جدول categories
- `id`: شناسه یکتای دسته‌بندی
- `title_fa`: عنوان فارسی دسته‌بندی
- `title_en`: عنوان انگلیسی دسته‌بندی
- `code`: کد دسته‌بندی
- `parent_id`: شناسه دسته‌بندی والد
- `created_at`: زمان ایجاد رکورد
- `updated_at`: زمان آخرین به‌روزرسانی

### جدول products
- `id`: شناسه یکتای محصول
- `title`: عنوان محصول
- `status`: وضعیت محصول
- `min_price`: حداقل قیمت
- `category_id`: شناسه دسته‌بندی محصول
- `category_name`: نام دسته‌بندی
- `brand`: برند محصول
- `market_price`: قیمت بازار
- `image_src`: آدرس تصویر محصول
- `commission_cancel`: اطلاعات لغو کارمزد
- `commission_rate`: نرخ کارمزد
- `recommended_price`: قیمت پیشنهادی
- `number_of_sellers`: تعداد فروشندگان
- `is_selling`: وضعیت فروش
- `order_limit_minimum`: حداقل تعداد سفارش
- `order_limit_maximum`: حداکثر تعداد سفارش
- `created_at`: زمان ایجاد رکورد
- `updated_at`: زمان آخرین به‌روزرسانی

## راه‌اندازی

1. فایل زیپ را از گیتهاب دانلود کنید
2. محتوای فایل زیپ را استخراج کنید
3. دیتابیس PostgreSQL را ایجاد کنید
4. اسکریپت‌های SQL را اجرا کنید:
   ```bash
   psql -U postgres -d your_database_name -f categories.sql
   psql -U postgres -d your_database_name -f products.sql
   ```

## نمونه کوئری‌ها

### 1. یافتن دسته‌بندی‌های بدون محصول
```sql
SELECT categories.id
FROM categories
LEFT JOIN products p ON categories.id = p.category_id
GROUP BY categories.id 
HAVING COUNT(p.id) = 0
ORDER BY 1 DESC;
```

### 2. تحلیل کارمزد بر اساس دسته‌بندی و برند
```sql
WITH RECURSIVE category_level AS (
    SELECT id, parent_id, 0 AS lvl
    FROM categories
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, cl.lvl + 1
    FROM categories c
    JOIN category_level cl ON c.parent_id = cl.id
),

category_tree AS (
    SELECT id AS root_id, id AS cat_id
    FROM categories
    UNION ALL
    SELECT ct.root_id, c.id
    FROM category_tree ct
    JOIN categories c ON c.parent_id = ct.cat_id
),

category_brand_commission AS (
    SELECT ct.root_id AS category_id,
           p.brand,
           p.commission_rate * 100 AS commission_pct
    FROM category_tree ct
    JOIN products p ON p.category_id = ct.cat_id
    WHERE p.commission_rate IS NOT NULL
      AND p.commission_rate <> 0
)

SELECT
    c.id,
    c.title_fa,
    cl.lvl AS category_level,
    cbc.brand,
    COUNT(*) AS product_cnt,
    MIN(cbc.commission_pct)::numeric(10,2) AS min_commission_pct,
    MAX(cbc.commission_pct)::numeric(10,2) AS max_commission_pct,
    AVG(cbc.commission_pct)::numeric(10,2) AS avg_commission_pct
FROM categories c
JOIN category_level cl ON cl.id = c.id
JOIN category_brand_commission cbc ON cbc.category_id = c.id
GROUP BY c.id, c.title_fa, cl.lvl, cbc.brand
ORDER BY c.id, cbc.brand;
```

### 3. محصولات پرطرفدار بر اساس تعداد فروشندگان
```sql
SELECT 
    id,
    title,
    brand,
    category_name,
    number_of_sellers,
    min_price
FROM products
WHERE is_selling = true
ORDER BY number_of_sellers DESC
LIMIT 50;
```

### 4. ساختار درختی دسته‌بندی‌ها
```sql
WITH RECURSIVE category_tree AS (
    SELECT id, title_fa, parent_id, 0 AS level
    FROM categories
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.title_fa, c.parent_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT 
    id,
    LPAD('', level * 2, ' ') || title_fa AS hierarchical_title,
    level
FROM category_tree
ORDER BY level, title_fa;
```

### 5. تحلیل قیمت‌ها در هر دسته‌بندی
```sql
SELECT 
    c.title_fa AS category_name,
    COUNT(p.id) AS product_count,
    AVG(p.min_price)::numeric(10,2) AS avg_price,
    MIN(p.min_price)::numeric(10,2) AS min_price,
    MAX(p.min_price)::numeric(10,2) AS max_price
FROM categories c
JOIN products p ON c.id = p.category_id
WHERE p.is_selling = true
GROUP BY c.id, c.title_fa
HAVING COUNT(p.id) > 10
ORDER BY avg_price DESC;
```

### 6. جستجوی محصولات بر اساس عنوان
```sql
SELECT 
    id,
    title,
    brand,
    category_name,
    min_price,
    image_src
FROM products
WHERE to_tsvector('simple', title) @@ to_tsquery('simple', 'موبایل & سامسونگ')
AND is_selling = true
ORDER BY min_price ASC;
```

## نکات مهم

- از ایندکس‌های ایجاد شده برای بهینه‌سازی عملکرد کوئری‌ها استفاده کنید
- برای جستجوی متنی از عملگر `@@` و توابع `to_tsvector` و `to_tsquery` استفاده کنید
- از CTEها برای کوئری‌های پیچیده و بازگشتی استفاده کنید
- برای تحلیل داده‌ها از توابع تجمیعی مانند `COUNT`, `AVG`, `MIN`, `MAX` استفاده کنید

## پشتیبانی

برای سوالات و مشکلات مربوط به دیتابیس، لطفاً از طریق گیتهاب اقدام کنید.
