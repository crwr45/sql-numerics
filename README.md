# Numeric Types in SQL

This is a collection of changes for enabling sane handling of `Numeric` columns
in SQL, JDBC (and Kafka). It has potential wider applicability for anyone
needing to handle `Numeric` data types.


# The Problem

## SQL

The underlying SQL Standard for Numeric and Decimal columns is restrictive and
flawed, to the point it is ignored by around half of databases examined, and not
fully followed by (m)any other databases.

Leading the charge of those discarding the standard are PostgreSQL, Oracle, and
SQLite. Many others, including MySQL, ignore parts of the standard. The only
database that seems to follow the standard precisely is MS SQL Server!

For the databases that do not follow the standard, they tend to ignore some of
the restrictions on `scale`, though that is not the only way that databases
deviate from the specification.

More generally the result of this is that standard interfaces to databases are
unnecessarily difficult.

There is a potential equivalence between the way strings with defined or
undefined lengths are stored and represented and the way `Numeric`s could be
represented.


## JDBC

JDBC specifies two methods by which a client can get information about the
`precision` and `scale` of a `Numeric` column.

First, using `DatabaseMetadata.getColumns()` to get a `ResultSet` that
summarises data about a table. This approach is ambiguous as to how undefined
`precision` and `scale` should be represented. The columns in the `ResultSet`
are `int` but are not explicitly `NOT NULL`.

Second, using `ResultSetMetadata.getPrecision()` and
`ResultSetMetadata.getScale()` to get an `int` for the scale of a column in the
`ResultSet`. This is explicitly an `int` and there is no mechanism by which a
`NULL` can be returned.


The combination of these means that JDBC *requires* that information about
`scale` (and potentially `precision`) is lost when it is used to interact with
databases.


## Mathematical Definitions

The meaning of `precision` and `scale` are NOT directly equivalent to the
mathematical concepts of "significant digits" and "decimal places", though
similar. In maths, ["significant digits" are reasonably clearly defined](https://en.wikipedia.org/wiki/Significant_figures#Significant_figures_rules_explained)
but do not entirely map onto the SQL standard `precision`. Similarly, "decimal
places" are not the same as `scale`.

`precision` is the max digits, including ones that are
*mathematically insignificant*. Therefore the concepts of `precision` and
"significant digits" are NOT the same.

<!-- In SQL, a `precision` of `0` means that the column can only meaningfully
store `0`, but that means it must have at least one digit to display. Depending
on the situation may be considered accurate down to many decimal places. -->

When expressing a number as `N * R^+-E` where `N` is the number as an integer
and `R` is the radix (usually 2 or 10), `scale` is NOT equivalent to the
exponent `E`.

In SQL the `scale` is restricted to *positive* values less than or equal to
`precision`. In those cases, the scale is the number of digits of the integer
number that are after the decimal point. This restriction to only positive
values, and restriction relative to `precision` means the Mathematical and SQL
definitions are not quite equivalent.

Oracle allows `scale` outside the standard range, and allows (some) negative
values, and so is more like the mathematical definition of an exponent (but not
the same).


## Display

While in most cases the `precision` and `scale` directly inform the display
formatting of a value from a column, this muddies the waters and complicates the
relationship between the mathematical definitions and the SQL definitions. I am
ignoring the display implications of the values and leaving that to each
implementation. The focus is therefore on the mathematical value of what is
stored.


# Solution

The solution is a multi-layered set of clarifications and changes to the meaning
of `precision` and `scale` for `Numeric` (and `Decimal`) columns in JDBC and a
generalisation of the SQL specification. 

For the approach reconcile these differences, please see the individual
documents for each area.
