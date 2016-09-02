Testing MySQL (TokuDB) transaction isolation levels
==========================================

These tests were run with Percona Server 5.7.14.

Setup (before every test case):

```sql
create table test (id int primary key, value int) engine=TokuDB;
insert into test (id, value) values (1, 10), (2, 20);
```

To see the current isolation level:

```sql
select @@tx_isolation;
```


Predicate-Many-Preceders (PMP)
------------------------------

TokuDB "repeatable read" prevents Predicate-Many-Preceders (PMP) for read predicates:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing
commit; -- T1
```

TokuDB "repeatable read" doesprevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
update test set value = value + 10; -- T1
select * from test where value = 20; -- T2. Returns 2 => 20
delete from test where value = 20;  -- T2, ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

```

Lost Update (P4)
----------------

TokuDB "repeatable read" prevents Lost Update (P4):

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```



Read Skew (G-single)
--------------------

TokuDB "repeatable read" prevents Read Skew (G-single) on a read-only transaction:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

TokuDB "repeatable read" prevents Read Skew (G-single) -- test using predicate dependencies:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

TokuDB "repeatable read" does not prevent Read Skew (G-single) on a write predicate:

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Doesn't delete anything
select * from test where id = 2;   -- T1. Shows 2 => 20
commit; -- T1
```


Write Skew (G2-item)
--------------------

TokuDB "repeatable read" does not prevent Write Skew (G2-item):

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
```


Anti-Dependency Cycles (G2)
---------------------------

TokuDB "repeatable read" does not prevent Anti-Dependency Cycles (G2):

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```

