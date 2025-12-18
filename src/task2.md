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
    Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.036..0.037 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.020..0.020 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 0.542 ms
    Execution Time: 0.098 ms
    ```
    
    *Объясните результат:*
    Запрос использует GIN индекс для полнотекстового поиска. Планировщик применяет Bitmap Index Scan по индексу t_books_fts_idx, что позволяет быстро найти все документы, содержащие слово 'expert'. GIN (Generalized Inverted Index) индекс идеально подходит для полнотекстового поиска, так как хранит обратный список токенов. Найдена 1 книга ("Expert PostgreSQL Architecture"). Время выполнения очень быстрое - 0.098 ms, что демонстрирует эффективность GIN индексов для полнотекстового поиска.

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
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.017..0.017 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.223 ms
     Execution Time: 0.035 ms
     ```
     
     *Объясните результат:*
     Запрос использует Index Scan по первичному ключу t_lookup_pk. Поиск по первичному ключу очень эффективен благодаря B-tree индексу, который автоматически создается при определении PRIMARY KEY. Время выполнения составляет всего 0.035 ms, что говорит о высокой эффективности индекса для точечного поиска. Найдена 1 строка с ключом '0000000455'.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.094..0.098 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.257 ms
     Execution Time: 0.122 ms
     ```
     
     *Объясните результат:*
     Запрос также использует Index Scan по первичному ключу. Время выполнения немного больше (0.122 ms против 0.035 ms), что может быть связано с тем, что данные в кластеризованной таблице физически упорядочены по первичному ключу, но для точечного поиска это не дает значительного преимущества. Кластеризация более полезна для диапазонных запросов и последовательного чтения данных.

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
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.025..0.025 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.237 ms
     Execution Time: 0.045 ms
     ```
     
     *Объясните результат:*
     Запрос использует Index Scan по индексу t_lookup_value_idx. Индекс эффективно используется для поиска по значению. Время выполнения составляет 0.045 ms. Результат: 0 строк, так как в таблице нет значения 'T_BOOKS' (все значения имеют формат 'Value_N', где N - число от 1 до 150000).

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.044..0.044 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.228 ms
     Execution Time: 0.068 ms
     ```
     
     *Объясните результат:*
     Запрос использует Index Scan по индексу t_lookup_clustered_value_idx. Время выполнения составляет 0.068 ms, что немного больше, чем в обычной таблице (0.045 ms). Это может быть связано с тем, что кластеризация по первичному ключу не влияет на производительность поиска по вторичному индексу (item_value), так как данные физически упорядочены по другому ключу. Результат: 0 строк.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     При поиске по значению (item_value) производительность обеих таблиц практически одинакова:
     - Обычная таблица: 0.045 ms
     - Кластеризованная таблица: 0.068 ms
     
     Кластеризация по первичному ключу (item_key) не дает преимущества при поиске по вторичному индексу (item_value), так как данные физически упорядочены по другому ключу. Кластеризация эффективна только для запросов, которые используют тот же ключ, по которому выполнена кластеризация, или для последовательного чтения данных. Для поиска по вторичному индексу кластеризация может даже немного замедлить выполнение из-за дополнительных операций чтения разрозненных страниц данных.