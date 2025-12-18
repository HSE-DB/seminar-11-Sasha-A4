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
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=1.031..1.032 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=1.024..1.024 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 2.686 ms
   Execution Time: 2.307 ms
   ```
   
   *Объясните результат:*
   Запрос использует Bitmap Index Scan по BRIN индексу для поиска NULL значений. BRIN индекс эффективен для больших таблиц, так как хранит только минимальные и максимальные значения для блоков страниц. В данном случае найдено 0 строк, что означает отсутствие NULL значений в колонке category. Планировщик правильно выбрал использование BRIN индекса благодаря его компактности.

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
   Bitmap Heap Scan on t_books  (cost=12.16..2377.14 rows=1 width=33) (actual time=15.199..15.202 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1224
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=76065 width=0) (actual time=0.127..0.128 rows=12240 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.449 ms
   Execution Time: 15.244 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Запрос использует Bitmap Heap Scan с Bitmap Index Scan по BRIN индексу. Планировщик использует только индекс по category, а фильтрацию по author выполняет на уровне таблицы (Filter). Это происходит потому, что BRIN индексы работают на уровне блоков страниц и не могут эффективно комбинировать условия для разных колонок. Видно, что "Rows Removed by Index Recheck: 150000" - это означает, что BRIN индекс вернул много ложных срабатываний, и все строки пришлось перепроверять. Результат: 0 строк, так как таких комбинаций нет в данных.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3099.14..3099.15 rows=6 width=7) (actual time=27.205..27.208 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3099.00..3099.06 rows=6 width=7) (actual time=27.090..27.092 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.009..7.806 rows=150000 loops=1)
   Planning Time: 0.131 ms
   Execution Time: 27.266 ms
   ```
   
   *Объясните результат:*
   Запрос выполняет последовательное сканирование всей таблицы (Seq Scan), так как BRIN индекс не подходит для операций DISTINCT и ORDER BY. Планировщик использует HashAggregate для группировки уникальных категорий, а затем Sort для сортировки. BRIN индексы эффективны только для фильтрации по диапазонам значений, но не для операций агрегации и сортировки.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=8.098..8.100 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=8.092..8.092 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 3.758 ms
   Execution Time: 8.145 ms
   ```
   
   *Объясните результат:*
   Запрос выполняет последовательное сканирование всей таблицы (Seq Scan), так как BRIN индекс не может использоваться для поиска по шаблону LIKE 'S%'. BRIN индексы работают только с операторами сравнения (=, <, >, <=, >=), но не поддерживают операции поиска по шаблону. Результат: 0 строк, так как в данных нет авторов, начинающихся с 'S' (все авторы имеют формат 'Author_N').

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
   Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=44.324..44.326 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=44.311..44.314 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.304 ms
   Execution Time: 44.354 ms
   ```
   
   *Объясните результат:*
   Несмотря на наличие индекса t_books_lower_title_idx, планировщик все равно использует Seq Scan. Это может происходить из-за того, что статистика не обновлена после создания индекса, или планировщик считает последовательное сканирование более эффективным для данного запроса. Индекс по функции LOWER(title) должен был использоваться, но для этого может потребоваться обновление статистики (ANALYZE) или настройка параметров планировщика. Найдена 1 книга (вероятно, "Oracle Core").

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
   Bitmap Heap Scan on t_books  (cost=12.16..2377.14 rows=1 width=33) (actual time=0.853..0.854 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8817
     Heap Blocks: lossy=72
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=76065 width=0) (actual time=0.034..0.035 rows=720 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.656 ms
   Execution Time: 0.900 ms
   ```
   
   *Объясните результат:*
   Составной BRIN индекс позволяет планировщику использовать оба условия в Index Cond, что более эффективно, чем использование одного индекса с последующей фильтрацией. Видно, что "Rows Removed by Index Recheck: 8817" значительно меньше, чем в предыдущем случае (150000), что говорит о лучшей селективности составного индекса. Время выполнения также улучшилось с 15.244 ms до 0.900 ms. Это демонстрирует преимущество составных BRIN индексов для запросов с несколькими условиями.