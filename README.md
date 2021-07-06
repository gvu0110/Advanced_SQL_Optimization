# Advanced_SQL_Optimization

## PostgresSQL tools for tuning
- `EXPLAIN` and `ANALYZE`
```sql
EXPLAIN ANALYZE SELECT * FROM staff;
Seq Scan on staff  (cost=0.00..24.00 rows=1000 width=75) (actual time=0.012..4.878 rows=1000 loops=1)
Planning Time: 0.038 ms
Execution Time: 9.384 ms

EXPLAIN ANALYZE SELECT * FROM staff WHERE salary > 75000;
Seq Scan on staff  (cost=0.00..26.50 rows=715 width=75) (actual time=0.012..3.497 rows=717 loops=1)
  Filter: (salary > 75000)
  Rows Removed by Filter: 283
Planning Time: 0.060 ms
Execution Time: 8.237 ms
```
The cost actually be less when returning more data if there are fewer steps in the execution plan

## Types of indexes
### 1. Indexing
- Purpose of indexes:
    + Speed up access to data
    + Help enforce constraints
    + Indexes are ordered
    + Typically smaller than tables
- Types of indexes:
    + B-tree: balance tree, commonly used
    + Bitmap: used when columns have few distinct values (low cardinality)
    + Hash: used when looking up data in key-value form
    + Other special purpose indexes: designed for specific data types, like geospatial data

### 2. B-tree indexes
- With indexes, we can avoid costly full table scans. 
```sql
-- Create an index on salary column in staff table
CREATE INDEX idx_staff_salary ON staff(salary);

EXPLAIN ANALYZE SELECT * FROM staff WHERE salary > 75000;
Seq Scan on staff  (cost=0.00..26.50 rows=715 width=75) (actual time=0.014..4.211 rows=717 loops=1)
  Filter: (salary > 75000)
  Rows Removed by Filter: 283
Planning Time: 0.228 ms
Execution Time: 8.063 ms
-- The index is not used, still doing full table scan
-- It's because there are so many rows with a salaray greater than 75000, the query execution builder determined that it would be faster to simply scan the full table

EXPLAIN ANALYZE SELECT * FROM staff WHERE salary > 150000;
Index Scan using idx_staff_salary on staff  (cost=0.28..8.29 rows=1 width=75) (actual time=0.012..0.018 rows=0 loops=1)
  Index Cond: (salary > 150000)
Planning Time: 0.210 ms
Execution Time: 0.057 ms
```

### 3. Bitmap indexes
```sql
EXPLAIN ANALYZE SELECT * FROM staff WHERE job_title = 'Operator';
Seq Scan on staff  (cost=0.00..26.50 rows=11 width=75) (actual time=0.017..0.166 rows=11 loops=1)
  Filter: ((job_title)::text = 'Operator'::text)
  Rows Removed by Filter: 989
Planning Time: 0.061 ms
Execution Time: 0.242 ms

CREATE INDEX idx_staff_job_title ON staff(job_title);

EXPLAIN ANALYZE SELECT * FROM staff WHERE job_title = 'Operator';
Bitmap Heap Scan on staff  (cost=4.36..18.36 rows=11 width=75) (actual time=0.042..0.194 rows=11 loops=1)
  Recheck Cond: ((job_title)::text = 'Operator'::text)
  Heap Blocks: exact=9
  ->  Bitmap Index Scan on idx_staff_job_title  (cost=0.00..4.36 rows=11 width=0) (actual time=0.032..0.037 rows=11 loops=1)
        Index Cond: ((job_title)::text = 'Operator'::text)
Planning Time: 0.220 ms
Execution Time: 0.292 ms
```

### 4. Hash indexes
```sql
EXPLAIN ANALYZE SELECT * FROM staff WHERE email = 'bphillips5@time.com';
Seq Scan on staff  (cost=0.00..26.50 rows=1 width=75) (actual time=0.077..0.177 rows=1 loops=1)
  Filter: ((email)::text = 'bphillips5@time.com'::text)
  Rows Removed by Filter: 999
Planning Time: 0.059 ms
Execution Time: 0.219 ms

CREATE INDEX idx_staff_email ON staff USING HASH (email);

EXPLAIN ANALYZE SELECT * FROM staff WHERE email = 'bphillips5@time.com';
Index Scan using idx_staff_email on staff  (cost=0.00..8.02 rows=1 width=75) (actual time=0.016..0.025 rows=1 loops=1)
  Index Cond: ((email)::text = 'bphillips5@time.com'::text)
Planning Time: 0.117 ms
Execution Time: 0.060 ms
```

### 5. Specialized indexes
- Used in Postgres (may not have equivalents in other relational databases):
    + GIST
    + SP-GIST
    + GIN
    + BRIN

## Tuning joins
- Types of joins:
    + INNER JOIN
        + Most common join
        + Return rows from both tables that have corresponding row in other table
        + Performed when joining in WHERE clause
    + LEFT OUTER JOIN
        + Returns all rows from left table
        + Returns rows from right table that have matching key
    + RIGHT OUTER JOIN
        + Returns all rows from right table
        + Returns rows from left table that have matching key
    + FULL OUTER JOIN
        + Returns all rows from both tables

### 2. Nested loop joins
- Outer loop iterates over one table, called the driver table
- Inner loop iterates over other table, called the join table

Pros:
- Works with all join conditions
- Low overhead
- Works well with small tables
- Works well with small driver tables and joined column is indexed

Cons:
- Can be slow
- If tables do not fit in memory, even slower performance
- Indexes can improve the performance of the nested loop joins, especially covered indexes

```sql
-- Force the query plan builder to use the nested loops
set enable_nestloop=true;
set enable_hashjoin=false;
set enable_mergejoin=false;

EXPLAIN SELECT s.id, s.last_name, s.job_title, cr.country FROM staff s INNER JOIN company_regions cr ON s.region_id = cr.region_id;
Nested Loop  (cost=0.15..239.37 rows=1000 width=88)
  ->  Seq Scan on staff s  (cost=0.00..24.00 rows=1000 width=34)
  ->  Index Scan using company_regions_pkey on company_regions cr  (cost=0.15..0.22 rows=1 width=62)
        Index Cond: (region_id = s.region_id)

-- After drop the constraint company_regions_pkey
EXPLAIN SELECT s.id, s.last_name, s.job_title, cr.country FROM staff s INNER JOIN company_regions cr ON s.region_id = cr.region_id;
Nested Loop  (cost=0.00..8290.88 rows=2750 width=88)
  Join Filter: (s.region_id = cr.region_id)
    ->  Seq Scan on staff s  (cost=0.00..24.00 rows=1000 width=34)
    ->  Materialize  (cost=0.00..18.25 rows=550 width=62)
        ->  Seq Scan on company_regions cr  (cost=0.00..15.50 rows=550 width=62)
```
- When using any kind of joins (nested loops in particular), it helps to ensure that the foreign key columns (the column you're matching on) in your clause are indexed

### 3. Hash joins
- Equality only: hash joins used when "=" is used, but not other operators
- Time based on table size: rows in smaller table for build, rows in larger table for probe table
- Fast lookup: hash value is index into array of row identifiers
```sql
set enable_nestloop=false;
set enable_hashjoin=true;
set enable_mergejoin=false;

EXPLAIN SELECT s.id, s.last_name, s.job_title, cr.country from staff s INNER JOIN company_regions cr ON s.region_id = cr.region_id;
Hash Join  (cost=22.38..145.12 rows=2750 width=88)
  Hash Cond: (s.region_id = cr.region_id)
  ->  Seq Scan on staff s  (cost=0.00..24.00 rows=1000 width=34)
  ->  Hash  (cost=15.50..15.50 rows=550 width=62)
        ->  Seq Scan on company_regions cr  (cost=0.00..15.50 rows=550 width=62)
```

### 4. Merge joins
- Also known as sort merge
- First step is sorting tables
- Take advantage of ordering to reduce the number of rows checked
- Equality only: hash joins used when "=" is used, but not other operators
- Time based on table size: time to sort and time to scan
- Large table joins: works well when tables do not fit in memory
```sql
set enable_nestloop=false;
set enable_hashjoin=false;
set enable_mergejoin=true;

EXPLAIN SELECT s.id, s.last_name, s.job_title, cr.country from staff s INNER JOIN company_regions cr ON s.region_id = cr.region_id;
Merge Join  (cost=114.36..158.36 rows=2750 width=88)
  Merge Cond: (cr.region_id = s.region_id)
  ->  Sort  (cost=40.53..41.91 rows=550 width=62)
        Sort Key: cr.region_id
        ->  Seq Scan on company_regions cr  (cost=0.00..15.50 rows=550 width=62)
  ->  Sort  (cost=73.83..76.33 rows=1000 width=34)
        Sort Key: s.region_id
        ->  Seq Scan on staff s  (cost=0.00..24.00 rows=1000 width=34)
```
