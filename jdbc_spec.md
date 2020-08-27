# JDBC Interface

This is concerned with the two parts of the JDBC interface used to get `precision` and `scale`:

## [`getColumns()`](https://docs.oracle.com/javase/8/docs/api/java/sql/DatabaseMetaData.html#getColumns-java.lang.String-java.lang.String-java.lang.String-java.lang.String-)
```
ResultSet getColumns(String catalog,
                     String schemaPattern,
                     String tableNamePattern,
                     String columnNamePattern)

Retrieves a description of table columns available in the specified catalog.

Only column descriptions matching the catalog, schema, table and column name criteria are returned. They are ordered by TABLE_CAT,TABLE_SCHEM, TABLE_NAME, and ORDINAL_POSITION.

Each column description has the following columns: <snipped for brevity>

<snip>
    COLUMN_SIZE int => column size.
    BUFFER_LENGTH is not used.
    DECIMAL_DIGITS int => the number of fractional digits. Null is returned for data types where DECIMAL_DIGITS is not applicable.
    NUM_PREC_RADIX int => Radix (typically either 10 or 2)
    NULLABLE int => is NULL allowed.
        columnNoNulls - might not allow NULL values
        columnNullable - definitely allows NULL values
        columnNullableUnknown - nullability unknown
<snip>

The COLUMN_SIZE column specifies the column size for the given column. For numeric data, this is the maximum precision. For character data, this is the length in characters. For datetime datatypes, this is the length in characters of the String representation (assuming the maximum allowed precision of the fractional seconds component). For binary data, this is the length in bytes. For the ROWID datatype, this is the length in bytes. Null is returned for data types where the column size is not applicable.

Parameters:
    catalog - a catalog name; must match the catalog name as it is stored in the database; "" retrieves those without a catalog; null means that the catalog name should not be used to narrow the search
    schemaPattern - a schema name pattern; must match the schema name as it is stored in the database; "" retrieves those without a schema; null means that the schema name should not be used to narrow the search
    tableNamePattern - a table name pattern; must match the table name as it is stored in the database
    columnNamePattern - a column name pattern; must match the column name as it is stored in the database
Returns:
    ResultSet - each row is a column description
```

Here, the `precision` is the `COLUMN_SIZE` column, and the `scale` is the `DECIMAL_DIGITS`.

## [`getPrecision`](https://docs.oracle.com/javase/8/docs/api/java/sql/ResultSetMetaData.html#getPrecision-int-)
```
    int getPrecision(int column)
              throws SQLException

    Get the designated column's specified column size. For numeric data, this is the maximum precision. For character data, this is the length in characters. For datetime datatypes, this is the length in characters of the String representation (assuming the maximum allowed precision of the fractional seconds component). For binary data, this is the length in bytes. For the ROWID datatype, this is the length in bytes. 0 is returned for data types where the column size is not applicable.

    Parameters:
        column - the first column is 1, the second is 2, ...
    Returns:
        precision
```


## [`getScale()`](https://docs.oracle.com/javase/8/docs/api/java/sql/ResultSetMetaData.html#getScale-int-)

```
int getScale(int column)

Gets the designated column's number of digits to right of the decimal point. 0 is returned for data types where the `scale` is not applicable.

Parameters:
    column - the first column is 1, the second is 2, ...
Returns:
    scale
```

# Suggested Changes

Within the existing definition, `getColumns` can use `NULL` to mean an undefined `precision` anmd `scale`, and `NULL`-ness can be established using `getInt()` followed by `wasNull()`. It may be necessary to change some drivers to have a `NULL` in the appropriate column, such as [this change to pgjdbc](https://github.com/pgjdbc/pgjdbc/commit/30843e45edc1e2bac499df2d1576c2db4d3d3309). So the method becomes:


```diff
-    COLUMN_SIZE int => column size.
-    DECIMAL_DIGITS int => the number of fractional digits. Null is returned for data types where DECIMAL_DIGITS is not applicable.
+    COLUMN_SIZE int => column size. Null is returned if COLUMN_SIZE is not specified.
+    DECIMAL_DIGITS int => the number of fractional digits. Null is returned when DECIMAL_DIGITS is not set or not applicable to the current type.
```
This could be considered a change to the meaning of `NULL` for `DECIMAL_DIGITS`, but the previous meaning was already explicitly only relevent to types where `DECIMAL_DIGITS` was not applicable, so this change is filling in the gap in the meaning.


Neither `getPrecision()` nor `getScale()` can be so cleanly updated.
I propose using the maximum allowed value for `precision` in the underlying storage when `getPrecision` is called on a column with no defined `precision`. e.g. in PostgreSQL, the maximum `precision` when undefined is 131072, so calling `getPrecision` on a `Numeric` column with no defined `precision` should return `131072`. Looking at the defintion above, this does not require a change, it is an allowable implementation choice. However, it does have implications for any system downstream.

I suggest using a value of `-127` to mean an undefined `scale`. `-1` cannot reasonably be used generally as Oracle allows `scale` in the range `-84` to `127`. `-127` is also used to mean an undefined `scale` internally for Oracle and Kafka-Connect-JDBC already. This change requires a small change to the definition of `getScale`:
```diff
int getScale(int column)

- Gets the designated column's number of digits to right of the decimal point. 0 is returned for data types where the scale is not applicable.
+ Gets the designated column's number of digits to right of the decimal point. -127 is returned for an unset scale, and 0 is returned for data types where the scale is not applicable.

Parameters:
    column - the first column is 1, the second is 2, ...
Returns:
    scale
```

The change needs further consideration as not every DBMS has been considered and some may be incompatible with `-127`
