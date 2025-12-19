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
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.022..0.022 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.016..0.017 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 0.613 ms
   Execution Time: 0.074 ms
   
   *Объясните результат:*
   BRIN индекс используется для поиска NULL, но строк с NULL нет (rows=0). Bitmap scan эффективен для такого запроса.

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
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=34.617..34.618 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.234..0.234 rows=12250 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.468 ms
   Execution Time: 34.664 ms
   
   *Объясните результат (обратите внимание на bitmap scan):*
   BRIN индекс создаёт lossy bitmap (1225 блоков), поэтому все 150000 строк проверяются при recheck. Индекс по author не используется, фильтрация - на уровне heap.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=71.006..71.008 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=70.962..70.965 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.012..16.447 rows=150000 loops=1)
   Planning Time: 0.174 ms
   Execution Time: 71.105 ms
   
   *Объясните результат:*
   BRIN индекс не подходит для DISTINCT, используется Seq Scan. HashAggregate группирует все строки, затем Sort упорядочивает результат.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=27.185..27.186 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=27.176..27.177 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 0.403 ms
   Execution Time: 27.222 ms
   
   *Объясните результат:*
   BRIN индекс не поддерживает префиксный поиск (LIKE 'S%'), используется Seq Scan с фильтрацией всех строк.

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
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=90.470..90.472 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=90.456..90.460 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.569 ms
   Execution Time: 90.509 ms
   
   *Объясните результат:*
   Индекс по LOWER(title) не используется для LIKE 'o%', т.к. префиксный поиск требует полного сканирования. Используется Seq Scan.

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
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=2.270..2.271 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8854
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.054..0.055 rows=730 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.573 ms
   Execution Time: 2.314 ms
   
   *Объясните результат:*
   Составной BRIN индекс эффективнее: проверяется меньше блоков (73 vs 1225), удалено меньше строк при recheck (8854 vs 150000) => уменьшилось время