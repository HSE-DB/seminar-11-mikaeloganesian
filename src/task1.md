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
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.019..0.019 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.016..0.016 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.448 ms
   Execution Time: 0.087 ms
   ```
   
   *Объясните результат:*
   Используется Bitmap Index Scan по BRIN индексу для поиска NULL значений. Запрос выполняется быстро, так как BRIN индекс эффективно фильтрует блоки данных. Результат пустой, так как в таблице нет записей с NULL категорией.

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
   Bitmap Heap Scan on t_books  (cost=12.16..2387.26 rows=1 width=33) (actual time=8.390..8.390 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1224
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=76740 width=0) (actual time=0.045..0.045 rows=12240 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.356 ms
   Execution Time: 8.422 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Выполняется Bitmap Index Scan по BRIN индексу на category, затем Bitmap Heap Scan с фильтрацией по author. BRIN индекс возвращает lossy блоки, поэтому требуется Recheck для точной проверки условий. После проверки индекса по category применяется фильтр по author на уровне кучи, что приводит к проверке всех 150000 строк.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3099.14..3099.15 rows=6 width=7) (actual time=21.302..21.304 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3099.00..3099.06 rows=6 width=7) (actual time=21.265..21.266 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.005..7.136 rows=150000 loops=1)
   Planning Time: 0.519 ms
   Execution Time: 21.403 ms
   ```
   
   *Объясните результат:*
   Выполняется последовательное сканирование всей таблицы, так как BRIN индекс не эффективен для операций DISTINCT. Все строки читаются, затем применяется HashAggregate для группировки и Sort для упорядочивания результата.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3099.03..3099.05 rows=1 width=8) (actual time=7.849..7.850 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=14 width=0) (actual time=7.846..7.846 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 0.538 ms
   Execution Time: 7.904 ms
   ```
   
   *Объясните результат:*
   BRIN индекс не используется для префиксного поиска с LIKE, так как он не поддерживает такие операции. Выполняется последовательное сканирование всей таблицы с фильтрацией по условию LIKE.

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
   Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=20.491..20.491 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=20.485..20.487 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.451 ms
   Execution Time: 20.527 ms
   ```
   
   *Объясните результат:*
   Несмотря на наличие индекса на LOWER(title), планировщик выбирает последовательное сканирование, так как префиксный поиск с LIKE не может эффективно использовать B-tree индекс. Все строки проверяются последовательно.

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
   Bitmap Heap Scan on t_books  (cost=12.16..2387.26 rows=1 width=33) (actual time=0.510..0.510 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8816
     Heap Blocks: lossy=72
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=76740 width=0) (actual time=0.038..0.038 rows=720 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.273 ms
   Execution Time: 0.538 ms
   ```
   
   *Объясните результат:*
   Составной BRIN индекс позволяет эффективно фильтровать по обоим условиям одновременно. Bitmap Index Scan использует оба условия в Index Cond, что значительно сокращает количество проверяемых блоков. Время выполнения уменьшилось с 8.422 мс до 0.538 мс по сравнению с предыдущим запросом, так как теперь не требуется дополнительная фильтрация по author на уровне кучи.