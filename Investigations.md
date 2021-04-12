# Tests

## Data Experiments

## SQL Standard
```quote
  Quoting the standard
  (SQL:2003, Part 2 Foundations, aka ISO/IEC 9075-2:2003)
4.4.2 Characteristics of numbers, page 27:
  An exact numeric type has a precision P and a scale S. P is a positive
  integer that determines the number of significant digits in a
  particular radix R, where R is either 2 or 10. S is a non-negative
  integer. Every value of an exact numeric type of scale S is of the
  form n*10^{-S}, where n is an integer such that ­-R^P <= n <= R^P.
  [...]
  If an assignment of some number would result in a loss of its most
  significant digit, an exception condition is raised. If least
  significant digits are lost, implementation-defined rounding or
  truncating occurs, with no exception condition being raised.
  [...]
  Whenever an exact or approximate numeric value is assigned to an exact
  numeric value site, an approximation of its value that preserves
  leading significant digits after rounding or truncating is represented
  in the declared type of the target. The value is converted to have the
  precision and scale of the target. The choice of whether to truncate
  or round is implementation-defined.
  [...]
  All numeric values between the smallest and the largest value,
  inclusive, in a given exact numeric type have an approximation
  obtained by rounding or truncation for that type; it is
  implementation-defined which other numeric values have such
  approximations.
5.3 <literal>, page 143
  <exact numeric literal> ::=
    <unsigned integer> [ <period> [ <unsigned integer> ] ]
  | <period> <unsigned integer>
6.1 <data type>, page 165:
  19) The <scale> of an <exact numeric type> shall not be greater than
      the <precision> of the <exact numeric type>.
  20) For the <exact numeric type>s DECIMAL and NUMERIC:
    a) The maximum value of <precision> is implementation-defined.
       <precision> shall not be greater than this value.
    b) The maximum value of <scale> is implementation-defined. <scale>
       shall not be greater than this maximum value.
  21) NUMERIC specifies the data type exact numeric, with the decimal
      precision and scale specified by the <precision> and <scale>.
  22) DECIMAL specifies the data type exact numeric, with the decimal
      scale specified by the <scale> and the implementation-defined
      decimal precision equal to or greater than the value of the
      specified <precision>.
6.26 <numeric value expression>, page 241:
  1) If the declared type of both operands of a dyadic arithmetic
     operator is exact numeric, then the declared type of the result is
     an implementation-defined exact numeric type, with precision and
     scale determined as follows:
   a) Let S1 and S2 be the scale of the first and second operands
      respectively.
   b) The precision of the result of addition and subtraction is
      implementation-defined, and the scale is the maximum of S1 and S2.
   c) The precision of the result of multiplication is
      implementation-defined, and the scale is S1 + S2.
   d) The precision and scale of the result of division are
      implementation-defined.
```

Note that while the Specification above treats Numeric and Decimal as different things, they can be considered identical for each database unless otherwise stated.

## Tests
n.b. Columns are left empty when they cannot hold the value (attempting to insert causes a failure). Each of these values is added to the table:

## PostgreSQL

### Notes
[Documentation](https://www.postgresql.org/docs/current/datatype-numeric.html)

### Description
According to the docs, they have have ENORMOUS capacity ("up to 131072 digits before the decimal point; up to 16383 digits after the decimal point") but trying to create a column that size in the database gives errors:
```sql
CREATE TABLE numeric_test (col Numeric(131072, 16383));
ERROR:  NUMERIC precision 131072 must be between 1 and 1000
LINE 1: CREATE TABLE numeric_test (col Numeric(131072, 16383));
```


However, the following does work. Here I am inserting a number with 114688 digits before and 16383 digits after the decimal point into a column with no defined precision and scale (note that `114072+16383` gives `131071`; the full `131071` failed, perhaps to allow for displaying a decimal point?). Pardon the slightly ugly functions; nobody wants to see 130k '1's here:
```sql
CREATE TABLE numeric_test (col Numeric);
INSERT INTO numeric_test VALUES (TO_NUMBER(CONCAT(REPEAT('1', 114688), '.', REPEAT('1', 16383)), CONCAT(REPEAT('9', 114688), '.', REPEAT('9', 16383))));
INSERT 0 1
```
Therefore, in order to accurately store very large numbers in a PostgreSQL Numeric column you have to leave the precision and scale undefined. This seems like an odd decision given that the SQL standard does not define a limit on precision or scale, only their relationship.

Not shown, but inserting a number with exactly `131072` digits before the decimal point AND `16383` digits after it causes an error. The *total* number of digits must be less than `131072` which is not clear from the docs.

### Table Creation
I created a simple table with a variety of Numeric columns. `col1` shows no default will be used if precision is unset, `col2` shows that specifying a precision and no scale causes scale to be set to 0.
```sql
CREATE TABLE numeric_test (
	col_val varchar,
	col_both_def Numeric,
	col_scale_def Numeric(5),
	col_five_two Numeric(5, 2),
	col_five_five Numeric(5, 5)
);
```

This table is then inspected using the `information_schema.columns` view, as well as `\d` command:
```sql
SELECT 
	column_name, 
	data_type, 
	udt_name, 
	character_maximum_length, 
	numeric_precision, 
	numeric_scale, 
	datetime_precision 
FROM
	information_schema.columns
WHERE
	table_name = 'numeric_test';
  column_name  |     data_type     | udt_name | character_maximum_length | numeric_precision | numeric_scale | datetime_precision 
---------------+-------------------+----------+--------------------------+-------------------+---------------+--------------------
 col_val       | character varying | varchar  |                          |                   |               |                   
 col_both_def  | numeric           | numeric  |                          |                   |               |                   
 col_scale_def | numeric           | numeric  |                          |                 5 |             0 |                   
 col_five_two  | numeric           | numeric  |                          |                 5 |             2 |                   
 col_five_five | numeric           | numeric  |                          |                 5 |             5 |                   
(5 rows)
```

```sql
\d numeric_test 
          Table "public.numeric_test"
    Column     |       Type        | Modifiers 
---------------+-------------------+-----------
 col_val       | character varying | 
 col_both_def  | numeric           | 
 col_scale_def | numeric(5,0)      | 
 col_five_two  | numeric(5,2)      | 
 col_five_five | numeric(5,5)      | 

```
Both of these agree.

### Data Experiments
n.b. Columns are left empty when they cannot hold the value (attempting to insert causes a failure). Each of these values is added to the table:

```sql
SELECT * FROM numeric_test;
     col_val      |   col_both_def   | col_scale_def | col_five_two | col_five_five 
------------------+------------------+---------------+--------------+---------------
 0.54321          |          0.54321 |             1 |         0.54 |       0.54321
 54321.0          |          54321.0 |         54321 |              |              
 0.10987654321    |    0.10987654321 |             0 |         0.11 |       0.10988
 5432.12345678910 | 5432.12345678910 |          5432 |              |              
 10.5             |             10.5 |            11 |        10.50 |              
 10.445           |           10.445 |            10 |        10.45 |              
 10.444           |           10.444 |            10 |        10.44 |              
(7 rows)

```

### Summary
Postgres will error rather than lose the most significant digit(s), and otherwise follows normal rounding rules (5 up, one place only). (Almost) any value can be stored accurately in a `Numeric` Column, but a `Numeric(p[, s])` column follows the SQL standard.


## MySQL

### Notes

MySQL does not support Numeric or Decimal columns without a precision or scale. On creation, a column declared as `Numeric`, will be created as `Numeric(10, 0)`

### Table and Data
```sql
CREATE TABLE numeric_test (col_val VARCHAR(50), col_both_def Numeric, col_scale_def Numeric(5), col_five_two Numeric(5, 2), col_five_five Numeric(5, 5));

DESCRIBE numeric_test;
+---------------+---------------+------+-----+---------+-------+
| Field         | Type          | Null | Key | Default | Extra |
+---------------+---------------+------+-----+---------+-------+
| col_val       | varchar(50)   | YES  |     | NULL    |       |
| col_both_def  | decimal(10,0) | YES  |     | NULL    |       |
| col_scale_def | decimal(5,0)  | YES  |     | NULL    |       |
| col_five_two  | decimal(5,2)  | YES  |     | NULL    |       |
| col_five_five | decimal(5,5)  | YES  |     | NULL    |       |
+---------------+---------------+------+-----+---------+-------+
5 rows in set (0.01 sec)

SHOW FULL COLUMNS FROM numeric_test;
+---------------+---------------+-------------------+------+-----+---------+-------+---------------------------------+---------+
| Field         | Type          | Collation         | Null | Key | Default | Extra | Privileges                      | Comment |
+---------------+---------------+-------------------+------+-----+---------+-------+---------------------------------+---------+
| col_val       | varchar(50)   | latin1_swedish_ci | YES  |     | NULL    |       | select,insert,update,references |         |
| col_both_def  | decimal(10,0) | NULL              | YES  |     | NULL    |       | select,insert,update,references |         |
| col_scale_def | decimal(5,0)  | NULL              | YES  |     | NULL    |       | select,insert,update,references |         |
| col_five_two  | decimal(5,2)  | NULL              | YES  |     | NULL    |       | select,insert,update,references |         |
| col_five_five | decimal(5,5)  | NULL              | YES  |     | NULL    |       | select,insert,update,references |         |
+---------------+---------------+-------------------+------+-----+---------+-------+---------------------------------+---------+
5 rows in set (0.00 sec)
```

```sql
INSERT INTO numeric_test(
	col_val, col_both_def, col_scale_def, col_five_two, col_five_five)
VALUES 
	('0.54321', 0.54321, 0.54321, 0.54321, 0.54321),
	('54321.0', 54321.0, 54321.0, 54321.0, 54321.0),
	('0.10987654321', 0.10987654321, 0.10987654321, 0.10987654321, 0.10987654321),
	('5432.12345678910', 5432.12345678910, 5432.12345678910, 5432.12345678910, 5432.12345678910),
	('10.5', 10.5, 10.5, 10.5, 10.5),
	('10.445', 10.445, 10.445, 10.445, 10.445),
	('10.444', 10.444, 10.444, 10.444, 10.444);

SELECT * FROM numeric_test;
+------------------+--------------+---------------+--------------+---------------+
| col_val          | col_both_def | col_scale_def | col_five_two | col_five_five |
+------------------+--------------+---------------+--------------+---------------+
| 0.54321          |            1 |             1 |         0.54 |       0.54321 |
| 54321.0          |        54321 |         54321 |       999.99 |       0.99999 |
| 0.10987654321    |            0 |             0 |         0.11 |       0.10988 |
| 5432.12345678910 |         5432 |          5432 |       999.99 |       0.99999 |
| 10.5             |           11 |            11 |        10.50 |       0.99999 |
| 10.445           |           10 |            10 |        10.45 |       0.99999 |
| 10.444           |           10 |            10 |        10.44 |       0.99999 |
+------------------+--------------+---------------+--------------+---------------+
7 rows in set (0.00 sec)
```

### Summary
MySQL will accept any number, and store the value that the column can accomodate that is closest to the value of that number For example, attempting to store 1000 in a column with a maximum possible value of 100 would result in the value 100 being stored. It raises a warning for this potential data loss rather than the SQL-defined exception, but otherwise accepts any number (I expect this can be made more strict by configuration).

In MySQL, no column can have a `null` for `precision` or `scale`.


## SQLITE 3

In SQLite3, ["any column in an SQLite version 3 database, except an INTEGER PRIMARY KEY column, may be used to store a value of any storage class.""](https://www.sqlite.org/datatype3.html) which means that the properties of a value, e.g. precision and scale, are related to the value itself, not the column it's stored in. The typing of a column only sets the preferred type class for that column, but it's not required to match.

As expected, this means the values are stored unchanged regardless of the column definition:

```sql
sqlite> .header on
sqlite> .mode column

sqlite> CREATE TABLE numeric_test (col_val VARCHAR(50), col_both_def Numeric, col_scale_def Numeric(5), col_five_two Numeric(5, 2), col_five_five Numeric(5, 5));

sqlite> .schema numeric_test
CREATE TABLE numeric_test (col_val VARCHAR(50), col_both_def Numeric, col_scale_def Numeric(5), col_five_two Numeric(5, 2), col_five_five Numeric(5, 5));

sqlite> pragma table_info('numeric_test');
cid         name        type         notnull     dflt_value  pk        
----------  ----------  -----------  ----------  ----------  ----------
0           col_val     VARCHAR(50)  0                       0         
1           col_both_d  Numeric      0                       0         
2           col_scale_  Numeric(5)   0                       0         
3           col_five_t  Numeric(5,   0                       0         
4           col_five_f  Numeric(5,   0                       0         


sqlite> INSERT INTO numeric_test(
   ...> col_val, col_both_def, col_scale_def, col_five_two, col_five_five)
   ...> VALUES 
   ...> ('0.54321', 0.54321, 0.54321, 0.54321, 0.54321),
   ...> ('54321.0', 54321.0, 54321.0, 54321.0, 54321.0),
   ...> ('0.10987654321', 0.10987654321, 0.10987654321, 0.10987654321, 0.10987654321),
   ...> ('5432.12345678910', 5432.12345678910, 5432.12345678910, 5432.12345678910, 5432.12345678910),
   ...> ('10.5', 10.5, 10.5, 10.5, 10.5),
   ...> ('10.445', 10.445, 10.445, 10.445, 10.445),
   ...> ('10.444', 10.444, 10.444, 10.444, 10.444);

sqlite> SELECT * FROM numeric_test;
col_val     col_both_def  col_scale_def  col_five_two  col_five_five
----------  ------------  -------------  ------------  -------------
0.54321     0.54321       0.54321        0.54321       0.54321      
54321.0     54321         54321          54321         54321        
0.10987654  0.1098765432  0.10987654321  0.1098765432  0.10987654321
5432.12345  5432.1234567  5432.12345678  5432.1234567  5432.12345678
10.5        10.5          10.5           10.5          10.5         
10.445      10.445        10.445         10.445        10.445       
10.444      10.444        10.444         10.444        10.444   

```

### Summary
SQLite3 is not even close to SQL. The behaviour we are interested in, rounding and sizes, is going to be entirely in the driver/client, and likely to be severely constrained by the clients interface.


## MS SQL Server

### Notes

The maximum precision is 38, default is 18.  The default scale is 0 and scale must be 0 <= s <= p.

### Table Creation
```sql
master> CREATE TABLE numeric_test (col_val VARCHAR(50), col_both_def Numeric, col_scale_def Numeric(5), col_five_two Numeric(5, 2), col_five_five Numeric(5, 5));

master> EXEC sp_columns numeric_test
Time: 1.007s (a second)
+-------------------+---------------+--------------+---------------+-------------+-------------+-------------+----------+---------+---------+------------+-----------+--------------+-----------------+--------------------+---------------------+--------------------+---------------+----------------+
| TABLE_QUALIFIER   | TABLE_OWNER   | TABLE_NAME   | COLUMN_NAME   | DATA_TYPE   | TYPE_NAME   | PRECISION   | LENGTH   | SCALE   | RADIX   | NULLABLE   | REMARKS   | COLUMN_DEF   | SQL_DATA_TYPE   | SQL_DATETIME_SUB   | CHAR_OCTET_LENGTH   | ORDINAL_POSITION   | IS_NULLABLE   | SS_DATA_TYPE   |
|-------------------+---------------+--------------+---------------+-------------+-------------+-------------+----------+---------+---------+------------+-----------+--------------+-----------------+--------------------+---------------------+--------------------+---------------+----------------|
| master            | dbo           | numeric_test | col_val       | 12          | varchar     | 50          | 50       | NULL    | NULL    | 1          | NULL      | NULL         | 12              | NULL               | 50                  | 1                  | YES           | 39             |
| master            | dbo           | numeric_test | col_both_def  | 2           | numeric     | 18          | 20       | 0       | 10      | 1          | NULL      | NULL         | 2               | NULL               | NULL                | 2                  | YES           | 108            |
| master            | dbo           | numeric_test | col_scale_def | 2           | numeric     | 5           | 7        | 0       | 10      | 1          | NULL      | NULL         | 2               | NULL               | NULL                | 3                  | YES           | 108            |
| master            | dbo           | numeric_test | col_five_two  | 2           | numeric     | 5           | 7        | 2       | 10      | 1          | NULL      | NULL         | 2               | NULL               | NULL                | 4                  | YES           | 108            |
| master            | dbo           | numeric_test | col_five_five | 2           | numeric     | 5           | 7        | 5       | 10      | 1          | NULL      | NULL         | 2               | NULL               | NULL                | 5                  | YES           | 108            |
+-------------------+---------------+--------------+---------------+-------------+-------------+-------------+----------+---------+---------+------------+-----------+--------------+-----------------+--------------------+---------------------+--------------------+---------------+----------------+
(5 rows affected)
```

### Data Handling
```
master> SELECT * FROM numeric_test;
Time: 0.610s
+------------------+----------------+-----------------+----------------+-----------------+
| col_val          | col_both_def   | col_scale_def   | col_five_two   | col_five_five   |
|------------------+----------------+-----------------+----------------+-----------------|
| 54321.0          | 54321          | 54321           | NULL           | NULL            |
| 0.54321          | 1              | 1               | 0.54           | 0.54321         |
| 0.10987654321    | 0              | 0               | 0.11           | 0.10988         |
| 5432.12345678910 | 5432           | 5432            | NULL           | NULL            |
| 10.5             | 11             | 11              | 10.50          | NULL            |
| 10.445           | 10             | 10              | 10.45          | NULL            |
| 10.444           | 10             | 10              | 10.44          | NULL            |
+------------------+----------------+-----------------+----------------+-----------------+
(7 rows affected)

```

### Summary
Like PostgreSQL, MSSQLServer rejects values it cannot represent, but with much more restrictive precision and scale, more akin to that of MySQL, and the requirement that every column must have a precision and scale. It follows normal rounding rules.




## Oracle
### Notes

Oracle allows a much wider range of values for precision and scale than the SQL standard permits, allowing scale to be as low as -84, or up to 127, while precision is limited to between 1 and 38, inclusive. Note that the specification defines the relationship between the two values, rather than the absolute values of them. This is the way that Oracle deviates from the spec.

### Table Creation
```
SQL> CREATE TABLE numeric_test (col_val VARCHAR(50), col_both_def Numeric, col_scale_def Numeric(5), col_five_two Numeric(5, 2), col_five_five Numeric(5, 5));
Table created.

SQL> SET LINESIZE 1000

SQL> DESCRIBE numeric_test;
 Name																																			   Null?    Type
 ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- -------- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 COL_VAL																																		    VARCHAR2(50)
 COL_BOTH_DEF																																		    NUMBER(38)
 COL_SCALE_DEF																																		    NUMBER(5)
 COL_FIVE_TWO																																		    NUMBER(5,2)
 COL_FIVE_FIVE																																		    NUMBER(5,5)

SQL> select COLUMN_NAME, DATA_LENGTH, DATA_PRECISION, DATA_SCALE from USER_TAB_COLS;
COLUMN_NAME																  DATA_LENGTH	    DATA_PRECISION	     DATA_SCALE
-------------------------------------------------------------------------------------------------------------------------------- -------------------- -------------------- --------------------
COL_VAL 																	   50
COL_BOTH_DEF																	   22					      0
COL_SCALE_DEF																	   22			 5		      0
COL_FIVE_TWO																	   22			 5		      2
COL_FIVE_FIVE																	   22			 5		      5

```

### Data Behaviour
After inserting the data (elided since multiple row insertions are not nice in Oracle), the table looks like this:
```sql

SQL> SELECT * FROM numeric_test;
COL_VAL 					   COL_BOTH_DEF COL_SCALE_DEF COL_FIVE_TWO COL_FIVE_FIVE
-------------------------------------------------- ------------ ------------- ------------ -------------
0.54321 						      1 	    1	       .54	  .54321
54321.0 						  54321 	54321
0.10987654321						      0 	    0	       .11	  .10988
5432.12345678910					   5432 	 5432
10.5							     11 	   11	      10.5
10.445							     10 	   10	     10.45
10.444							     10 	   10	     10.44

7 rows selected.
```

Note the empty values where that value caused an error in that column.



### Extra `scale` tests
Since Oracle allows scale outside the normal range relative to , I did some additional tests with columns where scale is outside the usual range (greater than precision, or less than 0)
```
SQL> CREATE TABLE test (col_1 Numeric(2, 5), col_2 Numeric(2, -5));
Table created.

SQL> select COLUMN_NAME, DATA_LENGTH, DATA_PRECISION, DATA_SCALE from USER_TAB_COLS;
COLUMN_NAME																  DATA_LENGTH	    DATA_PRECISION	     DATA_SCALE
-------------------------------------------------------------------------------------------------------------------------------- -------------------- -------------------- --------------------
COL_1																		   22			 2		      5
COL_2																		   22			 2		     -5

SQL> INSERT INTO test VALUES (0.0001, 100000);
1 row created.

SQL> INSERT INTO test VALUES (0.000001, 100);
1 row created.

SQL> SELECT * FROM test;
	       COL_1		    COL_2
-------------------- --------------------
	       .0001		   100000
		   0			0

```
### Summary
Oracle allows scale to be null on the column definition, but treats values as though scale were set to 0 in that situation. Depending on how that column is inspected, it may report 0 scale, or null scale. Rounding behaviour is normal.
Scale outside the range usually allowed makes sense mathematically, but goes against the SQL standard.



## Finally

All of these databases handle Numerics differently in a variety of ways. The only one that even appears to follow the standard for everything I've tested is MS SQL Server.

MySQL breaks the standard in that it does NOT raise an error when losing the most significant digit to rounding. PostgresQL breaks the standard in that it allows columns with null precision and scale, Oracle breaks the standard in that it allows scales less than 0 or greater than precision, and SQLite3 is not really SQL.

* All databases round using the normal 5-up rule, considering only one digit beyond what fits in the column. MySQL will also adjust (round?) any value then outside the columns allowable range to the value inside the range closest to it.

* Postgres and SQLite3 both allow `null` for precision and/or scale. In Postgres this has a special meaning, it's unclear what it actually means in SQLite3, probably nothing. Oracle sometimes reports no scale for a column that was created without one, but sometimes reports 0 for the same column depending on how it's inspected, but behaves as though it's 0.

```
          Database            | Precision Min | Precision Max | Precision Default| Scale Min | Scale Max | Scale Default | Notes
------------------------------+---------------+---------------+------------------+-----------+-----------+---------------+----------------------
 PostgreSQL - Numeric(p[, s]) |             1 |          1000 | n/a see next row |         0 |      1000 |             0 | If neither value set, see next row
 PostgresQL - Numeric         |             1 |        131072 | per row          |         0 |     16383 | per row       | Need to store the number of atoms in the observable universe? Many times over?
 MySQL                        |             1 |            65 |               10 |         0 |        30 |             0 | 
 SQLite3                      |           n/a |           n/a | n/a              |       n/a |       n/a | n/a           | Dynamic typing. Will store anything and it's up to the client to figure it out.
 SQL Server                   |             1 |            38 |               38 |         0 |        38 |             0 | 
 Oracle                       |             1 |            38 | n/a              |       -84 |       127 |             0 | Thinking about these is a bit confusing after all the others.
```

There's two basic approaches:
1. Roughly follow the standard (Postgres w/ precision, MySQL, SQL Server)
2. Ignore the standard (Postgres w/o precision, SQLite3, Oracle)
