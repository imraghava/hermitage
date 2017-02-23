```
CREATE TABLE emp (
  name varchar(25),
  dept varchar(25),
  salary int
);
CREATE TABLE salaries (dept varchar(25), sumsalary int);

insert into emp values ('one','sales',1000),('two','sales',1000);
insert into salaries values ('sales',2000);

== phantom 1
T1: set session transaction isolation level serializable; begin;
T2: set session transaction isolation level serializable; begin;
T1: SELECT * FROM emp WHERE dept='sales';
T2: SELECT sumsalary FROM salaries WHERE dept='sales';
T2: insert into emp values ('third','sales',1000);
T2: UPDATE salaries SET sumsalary = 3000 WHERE dept='sales'; 
T2: COMMIT;
T1: SELECT * FROM emp WHERE dept='sales';


== phantom 2
T1: set session transaction isolation level serializable; begin;
T1: SELECT sum(salary) FROM emp WHERE dept='sales';
T2: set session transaction isolation level serializable; begin;
T2: SELECT sum(salary) FROM emp WHERE dept='sales';
T2: insert into emp values ('third','sales',1000);
T2: COMMIT;
T1: UPDATE emp SET salary=1100 WHERE dept='sales';
T1: COMMIT;
```
