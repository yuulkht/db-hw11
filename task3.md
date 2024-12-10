## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=31.222..258.014 rows=500562 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=28.790..28.791 rows=500562 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.480 ms
    Execution Time: 289.981 ms
    ```
    
    *Объясните результат:*
    Индекс используется, но данных много, поэтому фактическое время исполнения большое

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
    workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
    [2024-12-10 23:16:41] completed in 2 s 210 ms
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=5549.34..20104.18 rows=497667 width=39) (actual time=29.809..154.790 rows=500562 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=4172
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5424.93 rows=497667 width=0) (actual time=28.556..28.557 rows=500562 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.309 ms
    Execution Time: 184.881 ms
    ```
    
    *Объясните результат:*
    Используется созданный индекс, теперь время фактического исполнения меньше в примерно в 1.5 раза за счет кластеризации

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    После кластеризации время планирования не сильно изменилось (хотя тоже снизилось), зато очень сильно уменьшилось время фактического выполнения, что говорит о том, что кластеризация имеет положительное влияние на производительность.