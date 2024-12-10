# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.019..0.020 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.017..0.017 rows=0 loops=1)
   Index Cond: (category IS NULL)
   Planning Time: 0.319 ms
   Execution Time: 0.065 ms
   ```
   
   *Объясните результат:*
   Используется индекс, ускоряет выполнение запроса тем, что сразу смотрит на строки, где category NULL

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=35.107..35.108 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1224
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.101..0.102 rows=12240 loops=1)
        Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.291 ms
   Execution Time: 35.149 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Сначала индекс не использовался, потом после фильтрации по автору для поиска нужных строк по категории индекс используется

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3099.11..3099.12 rows=5 width=7) (actual time=72.602..72.604 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3099.00..3099.05 rows=5 width=7) (actual time=72.579..72.582 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.014..17.803 rows=150000 loops=1)
   Planning Time: 0.152 ms
   Execution Time: 72.649 ms
   ```
   
   *Объясните результат:*
   Индекс не отсортировал данные, поэтому пришлось тратить время на сортироку. Затем просто используется Seq Scan.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3099.03..3099.05 rows=1 width=8) (actual time=31.320..31.322 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=14 width=0) (actual time=31.315..31.315 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 0.243 ms
   Execution Time: 31.411 ms
   ```
   
   *Объясните результат:*
   Использован просто Seq Scan, потому что созданный индекс не позволяет выполнить поиск строк с нужным вхождением.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=102.388..102.389 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=102.374..102.378 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.623 ms
   Execution Time: 102.444 ms
   ```
   
   *Объясните результат:*
   Аналогично предыдущему пункту не используется индекс

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=2.447..2.448 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8811
   Heap Blocks: lossy=72
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.037..0.038 rows=720 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.256 ms
   Execution Time: 2.488 ms
   ```

   *Объясните результат:*
   Тут индекс используется, поскольку мы сделали его по тем колонкам, по которым он используется в запросе