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
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=8.247..59.398 rows=500296 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=8334
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=7.269..7.269 rows=500296 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.561 ms
    Execution Time: 68.929 ms
    ```
    
    *Объясните результат:*
    Выполняется Bitmap Heap Scan с использованием индекса на category. Индекс эффективно находит все строки с category = 'A', но данные в куче разбросаны по 8334 блокам, что требует множественных обращений к диску. Время выполнения составляет 68.929 мс.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    CLUSTER

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=59.17..7668.56 rows=5000 width=68) (actual time=7.604..37.789 rows=500296 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=4170
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=7.202..7.202 rows=500296 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.339 ms
    Execution Time: 47.840 ms
    ```
    
    *Объясните результат:*
    После кластеризации данные с одинаковым значением category физически расположены рядом. Количество блоков, которые нужно прочитать, уменьшилось с 8334 до 4170, что почти в два раза. Время выполнения снизилось с 68.929 мс до 47.840 мс, что составляет улучшение примерно на 30%. Это происходит благодаря тому, что последовательное чтение блоков более эффективно, чем случайные обращения.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    Кластеризация показала значительное улучшение производительности. Время выполнения запроса уменьшилось с 68.929 мс до 47.840 мс, что составляет улучшение на 30.6%. Количество блоков, которые необходимо прочитать, сократилось с 8334 до 4170, то есть в два раза. Это объясняется тем, что после кластеризации все строки с category = 'A' физически расположены в последовательных блоках, что позволяет эффективно использовать последовательное чтение с диска вместо множественных случайных обращений. Кластеризация особенно эффективна для запросов, которые выбирают большие объемы данных по индексированному столбцу.