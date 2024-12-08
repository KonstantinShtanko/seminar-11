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
     "Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.132..0.133 rows=1 loops=1)"
     "Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
       Heap Blocks: exact=1"
     "->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.115..0.115 rows=1 loops=1)"
     "Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
     "Planning Time: 1.389 ms"
     "Execution Time: 0.223 ms"
     ```
    
    *Объясните результат:*
    GIN индекс помогает эффективнее искать строки, соответствующие поисковому запросу, без необходимости сканировать всю таблицу. Тип данных tsvector помогает отформатировать данные в вид, доступный для поиска по значению: в русском языке, например, у слова отрезаются окончания (то есть, происходит нормализация слов). Происходит битмап сканирование по всей таблице, а потом происходит перепроверка по соответствующим в запросе критериям. В результате, было зафиксировано 1 обращение к таблице. После чего происходит битмап индекс сканирование, в результате которого происходит полнотекстный поиск по критерию сходства title слову expert.

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
     "Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.098..0.100 rows=1 loops=1)"
     "Index Cond: ((item_key)::text = '0000000455'::text)"
     "Planning Time: 0.453 ms"
     "Execution Time: 0.142 ms" 
     ```   
     
     *Объясните результат:*
     В данном запросе используется индекс, который автоматически создается для первичного ключа. В результате запроса инициализируется поиск по b-tree, в результате которого по index cond выводятся необходимые строки. Скорость запроса благодаря использованию индекса - быстрая.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     "Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.033..0.034 rows=1 loops=1)"
     "Index Cond: ((item_key)::text = '0000000455'::text)"
     "Planning Time: 0.076 ms"
     "Execution Time: 0.051 ms"
     ```
     
     *Объясните результат:*
     Проводя поиск по кластеризованной таблице, скорость запроса становится немного более быстрой. Это объясняется тем, что кластеризация меняет физическое положение данных в таблице, которые помогают оптимизировать запрос. Головка диска, которая ездит по диску и собирает нужные строки, в случае некластеризованных данных, ездит по диску и собирает необходимые данные. Процесс кластеризации оптимизирует данный процесс, что и позволяет ускорить его относительно предыдущего запроса.

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
     "Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.026..0.026 rows=0 loops=1)"
     "Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
     "Planning Time: 0.096 ms"
     "Execution Time: 0.044 ms"
     ```
     
     *Объясните результат:*
     Запрос использует индекс b-tree, который мы создали, в результате которого происходит проверка на соответствие строки операции равенства. Index cond проверяет строку на соответствие условию запроса. Благодаря данному индексу, скорость запроса является быстрой. В результате запроса было выведено 0 строк.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     "Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.029..0.030 rows=0 loops=1)"
     "Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
     "Planning Time: 0.090 ms"
     "Execution Time: 0.049 ms"
     ```

     
     *Объясните результат:*
     Аналогично запросу с ключами, индекс по кластеризованным данным является оптимизированным ввиду физического изменения расположения строк. Таким образом, время запроса некритично, но уменьшается по сравнению с предыдущим.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Так как в кластеризованной таблице данные упорядочиваются в соответствии с указанным индексом. В результате кластеризации создается временная таблица, которая в результате заменяет оригинальную, также в конце происходит обновление статистики, что помогает улучшить производительность запросов.
     Помимо этого, кластеризация данных делается для оптимизации времени выполнения запросов к диску, на котором ездит головка и считывает данные. Он упорядочивает данные так, чтобы движение данной головки было оптимальным и более быстрым, чем по неупорядоченным данным. Таким образом, использование индекса по кластеризованной таблице помогает оптимизировать время запроса и улучшить производительность.