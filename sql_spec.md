# SQL Specification
For reference, the SQL Specification defines a Numeric as:
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

# Actual Behaviour

The following is based on observed behaviour of a variety of databases.

```
All databases implement a defined subset of what we could think of as a "generalised SQL standard" in which:

    precision is nonnegative or NULL
    scale is an integer (that may be negative) or NULL
    The constraint scale <= precision does not apply

With this generalised interpretation:

    PostgreSQL imposes an additional constraint that either:
        scale and precision are both NULL, or
        scale and precision are both NOT NULL, and scale <= precision
    MySQL imposes additional constraints that both scale and precision are NOT NULL, that scale >= 0, and that scale <= precision
    SQLite imposes no additional constraints
    SQL Server imposes additional constraints that both scale and precision are NOT NULL, that scale >= 0, and that scale <= precision
    Oracle imposes additional constraints that both scale and precision are NOT NULL. (Note that Oracle's NULL reported scale should be treated as scale = 0)

The important point is that all of these constraints produce strict subsets of the generalised behaviour. We therefore have a single unified model of how databases behave.
```
Credit to Michael Brown <mcb30@ipxe.org> for the wording.

