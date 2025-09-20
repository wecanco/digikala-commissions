# دیتابیس محصولات و دسته بندی های دیجیکالا (کارمزدهای دیجیکالا)

## معرفی
این دیتابیس شامل اطلاعات کامل نمونه محصولات و دسته‌بندی‌های دیجیکالا به همراه داده‌های کارمزد و ساختار سلسله‌مراتبی دسته‌بندی‌ها می‌باشد. فایل SQL ارائه شده یک دامپ کامل از دیتابیس است.

## راه‌اندازی سریع

### پیش‌نیازها
- نصب PostgreSQL
- دسترسی به خط فرمان (Command Line)

### مراحل راه‌اندازی

1. **ایجاد دیتابیس جدید**:
```bash
createdb -U postgres digikala_db
```

2. **بارگذاری دامپ کامل**:
```bash
psql -U postgres -d digikala_db -f complete_dump.sql
```

3. **اتصال به دیتابیس**:
```bash
psql -U postgres -d digikala_db
```

### روش جایگزین با استفاده از PGAdmin
1. PGAdmin را باز کنید
2. روی سرور کلیک راست کرده و Create > Database را انتخاب کنید
3. نام دیتابیس را وارد کنید (مثلاً digikala_db)
4. روی دیتابیس جدید کلیک راست کرده و Restore را انتخاب کنید
5. فایل complete_dump.sql را انتخاب و اجرا کنید

## ساختار دیتابیس

### جدول categories
- `id` (integer) - شناسه یکتای دسته‌بندی
- `title_fa` (varchar) - عنوان فارسی
- `title_en` (varchar) - عنوان انگلیسی  
- `code` (varchar) - کد دسته‌بندی
- `parent_id` (integer) - والد دسته‌بندی
- `created_at` (timestamp) - زمان ایجاد
- `updated_at` (timestamp) - زمان به‌روزرسانی

### جدول products
- `id` (bigint) - شناسه محصول
- `title` (text) - عنوان محصول
- `status` (varchar) - وضعیت محصول
- `min_price` (numeric) - حداقل قیمت
- `category_id` (integer) - شناسه دسته‌بندی
- `category_name` (varchar) - نام دسته‌بندی
- `brand` (varchar) - برند محصول
- `market_price` (numeric) - قیمت بازار
- `image_src` (text) - آدرس تصویر
- `commission_cancel` (varchar) - شرایط لغو کارمزد
- `commission_rate` (numeric) - نرخ کارمزد
- `recommended_price` (varchar) - قیمت پیشنهادی
- `number_of_sellers` (integer) - تعداد فروشندگان
- `is_selling` (boolean) - وضعیت فروش
- `order_limit_minimum` (integer) - حداقل سفارش
- `order_limit_maximum` (integer) - حداکثر سفارش
- `created_at` (timestamp) - زمان ایجاد
- `updated_at` (timestamp) - زمان به‌روزرسانی

## ایندکس‌های ایجاد شده
- `idx_products_category_id` - ایندکس برای category_id
- `idx_products_brand` - ایندکس برای برند
- `idx_products_status` - ایندکس برای وضعیت
- `idx_products_is_selling` - ایندکس برای وضعیت فروش  
- `idx_products_title` - ایندکس全文 برای عنوان محصول

## نمونه کوئری‌های مفید

### 1. بررسی اطلاعات پایه
```sql
-- تعداد کل محصولات
SELECT COUNT(*) AS total_products FROM products;

-- تعداد کل دسته‌بندی‌ها
SELECT COUNT(*) AS total_categories FROM categories;

-- محصولات فعال
SELECT COUNT(*) AS active_products FROM products WHERE is_selling = true;
```

### 2. تحلیل سلسله‌مراتب دسته‌بندی‌ها
```sql
-- نمایش ساختار درختی دسته‌بندی‌ها
WITH RECURSIVE category_tree AS (
    SELECT id, title_fa, parent_id, ARRAY[id] AS path, 0 AS level
    FROM categories 
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.title_fa, c.parent_id, ct.path || c.id, ct.level + 1
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT 
    level,
    REPEAT('--', level) || title_fa AS category_tree,
    path
FROM category_tree
ORDER BY path;
```

### 3. تحلیل کارمزد بر اساس دسته‌بندی
```sql
-- میانگین کارمزد به تفکیک دسته‌بندی
SELECT 
    c.title_fa AS category_name,
    COUNT(p.id) AS product_count,
    ROUND(AVG(p.commission_rate * 100)::numeric, 2) AS avg_commission_percent,
    ROUND(MIN(p.commission_rate * 100)::numeric, 2) AS min_commission_percent,
    ROUND(MAX(p.commission_rate * 100)::numeric, 2) AS max_commission_percent
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.commission_rate IS NOT NULL 
  AND p.commission_rate > 0
GROUP BY c.id, c.title_fa
ORDER BY avg_commission_percent DESC;
```

### 4. تحلیل برندها
```sql
-- برترین برندها بر اساس تعداد محصولات
SELECT 
    brand,
    COUNT(*) AS product_count,
    ROUND(AVG(min_price)::numeric, 2) AS avg_price,
    ROUND(MIN(min_price)::numeric, 2) AS min_price,
    ROUND(MAX(min_price)::numeric, 2) AS max_price
FROM products
WHERE brand IS NOT NULL 
  AND is_selling = true
GROUP BY brand
HAVING COUNT(*) > 10
ORDER BY product_count DESC
LIMIT 20;
```

### 5. محصولات پرطرفدار
```sql
-- محصولات با بیشترین تعداد فروشنده
SELECT 
    id,
    title,
    brand,
    category_name,
    number_of_sellers,
    min_price,
    image_src
FROM products
WHERE is_selling = true 
  AND number_of_sellers > 0
ORDER BY number_of_sellers DESC, min_price ASC
LIMIT 30;
```

### 6. جستجوی پیشرفته محصولات
```sql
-- جستجوی محصولات با فیلترهای مختلف
SELECT 
    p.id,
    p.title,
    p.brand,
    p.category_name,
    p.min_price,
    p.number_of_sellers,
    p.commission_rate * 100 AS commission_percent,
    p.image_src
FROM products p
WHERE p.is_selling = true
  AND p.min_price BETWEEN 100000 AND 500000
  AND p.number_of_sellers >= 3
  AND p.brand ILIKE '%سامسونگ%'
  AND to_tsvector('persian', p.title) @@ to_tsquery('persian', 'گوشی & موبایل')
ORDER BY p.min_price ASC, p.number_of_sellers DESC;
```

### 7. تحلیل قیمت‌ها
```sql
-- توزیع قیمتی محصولات به تفکیک دسته‌بندی
SELECT 
    c.title_fa AS category_name,
    COUNT(p.id) AS product_count,
    ROUND(AVG(p.min_price)::numeric, 2) AS average_price,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY p.min_price)::numeric, 2) AS median_price,
    ROUND(MIN(p.min_price)::numeric, 2) AS min_price,
    ROUND(MAX(p.min_price)::numeric, 2) AS max_price
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.is_selling = true 
  AND p.min_price > 0
GROUP BY c.id, c.title_fa
HAVING COUNT(p.id) > 50
ORDER BY average_price DESC;
```

### 8. دسته‌بندی‌های بدون محصول
```sql
-- یافتن دسته‌بندی‌های خالی
SELECT 
    c.id,
    c.title_fa,
    c.title_en,
    c.parent_id
FROM categories c
LEFT JOIN products p ON c.id = p.category_id
WHERE p.id IS NULL
ORDER BY c.id;
```


## نمونه کوئری‌ها 2

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



## نکات فنی مهم

1. **جستجوی متنی**: از ایندکس GIN ایجاد شده برای جستجوی سریع در عنوان محصولات استفاده کنید
2. **بهینه‌سازی**: کوئری‌های پیچیده را با استفاده از CTEها به بخش‌های کوچکتر تقسیم کنید
3. **فیلتر کردن**: همیشه از فیلتر `is_selling = true` برای محصولات فعال استفاده کنید
4. **ایندکس‌ها**: از ایندکس‌های موجود برای فیلتر کردن بر اساس category_id، brand و status استفاده کنید

## عیب‌یابی

اگر با خطا مواجه شدید:

1. مطمئن شوید PostgreSQL در حال اجراست
2. بررسی کنید کاربر postgres دسترسی لازم را دارد
3. حجم دیتابیس و فضای دیسک را بررسی کنید
4. از اجرای مرحله به مرحله اسکریپت استفاده کنید

## منابع مفید

- مستندات PostgreSQL: https://www.postgresql.org/docs/
- آموزش SQL: https://www.w3schools.com/sql/
- ابزارهای مدیریت PostgreSQL: pgAdmin, DBeaver, psql

این دیتابیس کامل بوده و تمامی داده‌های لازم برای تحلیل محصولات دیجیکالا را در اختیار شما قرار می‌دهد.
