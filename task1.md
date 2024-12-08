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
   "Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.044..0.045 rows=0 loops=1)"
   "Recheck Cond: (category IS NULL)"
   "->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.037..0.038 rows=0 loops=1)"
   "Index Cond: (category IS NULL)"
   "Planning Time: 0.519 ms"
   "Execution Time: 0.140 ms"
   
   *Объясните результат:*
   В данном случае, индекс используется для фильтрации строк, где category is NULL. Видно, что для начала происходит создания битмапов, которые указывают, какие строки потенциально удовлетворяют условию. Далее происходит перепроверка по условию category is null. После создания карты битов, оптимизатор проходит по ним для определения тех строк, которые нужно извлечь, с помощью индекса (индекс BRIN является упрощенным вариантом индекса, он разделяет данные на блоки, устанавливая для каждого минимальные и максимальные значения, а далее выполняет фильтрацию по этим блокам). Потом происходит еще одна перепроверка данных. Можно заметить, что было возвращено 0 строк, а на выполнение запроса было затрачено немного времени, что свидетельствует об эффективности использования данного индекса.

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
   "Bitmap Heap Scan on t_books  (cost=12.16..2345.54 rows=1 width=33) (actual time=19.181..19.182 rows=0 loops=1)"
   "Recheck Cond: ((category)::text = 'INDEX'::text)"
   "Rows Removed by Index Recheck: 150000"
   "Filter: ((author)::text = 'SYSTEM'::text)"
   "Heap Blocks: lossy=1224"
   "->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=73959 width=0) (actual time=0.152..0.152 rows=12240 loops=1)"
   "Index Cond: ((category)::text = 'INDEX'::text)"
   "Planning Time: 0.487 ms"
   "Execution Time: 19.292 ms"
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Вначале происходит сканирование с помощью битмап скана. Происходит это так же, как и в прошлом пункте (по категории). С помощью фильтра по второму условию (автор) убираются неподходящие строки. Далее, оптимизатор сканирует индекс и создает битовую карту по данным условиям. По ней отбираются уже значения по атрибуту category. Таким образом, выводятся значения, которые соответствуют данным условиям.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   "Sort  (cost=3099.14..3099.15 rows=6 width=7) (actual time=49.243..49.244 rows=6 loops=1)"
   "Sort Key: category"
   "Sort Method: quicksort  Memory: 25kB"
   "  ->  HashAggregate  (cost=3099.00..3099.06 rows=6 width=7) (actual time=49.178..49.179 rows=6 loops=1)"
   "Group Key: category"
   "Batches: 1  Memory Usage: 24kB"
   "->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.044..17.510 rows=150000 loops=1)"
   "Planning Time: 0.652 ms"
   "Execution Time: 49.281 ms"
   
   *Объясните результат:*
   Так как BRIN-индекс не умеет упорядочивать данные, то в данном случае оптимизатор его не использует. Вначале оптимизатор прибегает к сортировке с помощью алгоритма "Quicksort", который заключается в определении опорного элемента (середины), от которого отталкивается дальнейшее решение. К сортированным данным применяется проверка по фильтру, в результате которой убираются ненужные строки. Потом используется метод агрегации HashAggregate по категориям, в результате которого категории группируются по названиям (так как в запросе было DISTINCT). После чего проводится последовательное сканирование по полученным данным, в результате которого выводятся нужные строки.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   "Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=23.309..23.310 rows=1 loops=1)"
   "->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=23.305..23.305 rows=0 loops=1)"
   "Filter: ((author)::text ~~ 'S%'::text)"
   "Rows Removed by Filter: 150000"
   "Planning Time: 1.570 ms"
   "Execution Time: 23.454 ms"
   ```
   
   *Объясните результат:*
   Так как никаких индексов не было создано, то оптимизатор для начала использует агрегирование, так как требуется вывести результат по функции COUNT. После чего проходит по всей таблице последовательным поиском и считает, что подходит под условие, и убирает неподходящие строки по фильтру. В данном случае эффективно было бы использовать префиксный индекс для оптимизации запроса.

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
   "Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=44.216..44.217 rows=1 loops=1)"
   "->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=44.207..44.209 rows=1 loops=1)"
   "Filter: (lower((title)::text) ~~ 'o%'::text)"
   "Rows Removed by Filter: 149999"
   "Planning Time: 1.024 ms"
   "Execution Time: 44.279 ms"
   ```
   
   *Объясните результат:*
   Так как индекс не является оптимальным для поиска по оператору LIKE, то оптимизатор решил применить то же, что и в прошлом пункте.

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
   "Bitmap Heap Scan on t_books  (cost=12.16..2345.54 rows=1 width=33) (actual time=2.274..2.275 rows=0 loops=1)"
   "Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "Rows Removed by Index Recheck: 8822"
   "Heap Blocks: lossy=72"
   "->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=73959 width=0) (actual time=0.269..0.269 rows=720 loops=1)"
   "Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "Planning Time: 0.970 ms"
   "Execution Time: 2.389 ms"
   ```

   
   *Объясните результат:*
   Для начала оптимизатор делает битмап сканирование по таблице с перепроверкой по двум условиям, в результате которой с помощью индекса были убраны неподходящие строки.
   Далее, с помощью индекса, создается битовая карта, в результате которой (по index cond) отбираются строки при соблюдении необходимых условий из запроса. Можно заметить, что скорость запроса является быстрой относительно запроса без использования индекса.