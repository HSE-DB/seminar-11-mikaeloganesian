## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.032..0.032 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.022..0.022 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 1.100 ms
    Execution Time: 0.080 ms
    ```
    
    *Объясните результат:*
    GIN индекс эффективно используется для полнотекстового поиска. Bitmap Index Scan по GIN индексу быстро находит соответствующие записи, затем Bitmap Heap Scan извлекает полные строки. Время выполнения очень мало благодаря эффективности GIN индекса для текстового поиска.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.026..0.026 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.398 ms
     Execution Time: 0.069 ms
     ```
     
     *Объясните результат:*
     Используется Index Scan по первичному ключу. Поиск выполняется быстро благодаря B-tree индексу на первичном ключе. Данные в таблице не кластеризованы, поэтому после поиска по индексу требуется дополнительный переход к данным в куче.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.033..0.034 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.260 ms
     Execution Time: 0.052 ms
     ```
     
     *Объясните результат:*
     В кластеризованной таблице данные физически упорядочены по первичному ключу. Это означает, что после поиска по индексу данные находятся в последовательных блоках, что может улучшить производительность при последовательном чтении. Время выполнения немного меньше, чем в обычной таблице.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.019..0.020 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.262 ms
     Execution Time: 0.038 ms
     ```
     
     *Объясните результат:*
     Используется Index Scan по индексу на item_value. Поиск выполняется быстро, но после нахождения ключа в индексе требуется переход к данным в куче. Поскольку таблица не кластеризована по этому индексу, данные могут быть разбросаны по разным блокам.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.029..0.029 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.423 ms
     Execution Time: 0.055 ms
     ```
     
     *Объясните результат:*
     Поиск по значению в кластеризованной таблице использует тот же механизм Index Scan. Однако таблица кластеризована по первичному ключу, а не по item_value, поэтому преимущества кластеризации не проявляются при поиске по значению. Время выполнения сопоставимо с обычной таблицей.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
    
     *Сравнение:*
     При поиске по значению производительность в обеих таблицах практически одинаковая. Обычная таблица показала время выполнения 0.038 мс, кластеризованная 0.055 мс. Это объясняется тем, что кластеризация выполнена по первичному ключу item_key, а поиск происходит по item_value. Для получения преимуществ от кластеризации при поиске по значению необходимо было бы кластеризовать таблицу по индексу на item_value. При поиске по первичному ключу кластеризованная таблица показала немного лучшее время выполнения благодаря физической упорядоченности данных.