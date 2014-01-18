# Emacsql

Emacsql is a high-level Emacs Lisp front-end for SQLite. It's
currently a work in progress.

It works by keeping a `sqlite3` inferior process running (a
"connection") for interacting with the back-end database. Connections
are automatically cleaned up if they are garbage collected. All
requests are synchronous.

Any [readable lisp value][readable] can be stored as a value in
Emacsql, including numbers, strings, symbols, lists, vectors, and
closures. Emacsql has no concept of "TEXT" values; it's all just lisp
objects.

Requires Emacs 24 or later.

## Usage

```el
(defvar db (emacsql-connect "company.db"))

;; Create a table. A table identifier can be any kind of lisp value.
(emacsql-create db 'employees [name id salary])

;; Or optionally provide column constraints.
(emacsql-create db 'employees [name (id integer :unique) (salary float)])

;; Insert some data:
(emacsql-insert db 'employees ["Jeff"  1000 60000.0]
                              ["Susan" 1001 64000.0])

;; Query the database for results:
(emacsql db [:select [name id] :from employees :where (> salary 62000)])
;; => (("Susan" 1001))

;; Queries can be templates, using $1, $2, etc.:
(emacsql db
         [:select [name id] :from employees :where (> salary $1)]
         50000)
;; => (("Jeff" 1000) ("Susan" 1001))
```

## Operators

Emacsql currently supports the following expression operators, named
exactly like so in a structured Emacsql statement.

    *     /     %     +     -     <<    >>    &
    |     <     <=    >     >=    =     !=
    is    like  glob  and   or

The `<=` and `>=` operators accept 2 or 3 operands, transforming into
a SQL `_ BETWEEN _ AND _` operator as appropriate.

With `glob` and `like` keep in mind that they're matching the
*printed* representations of these values, even if the value is a
string.

The `||` concatenation operator is unsupported because concatenating
printed representations breaks an important constraint: all values must
remain readable within SQLite.

## Structured Statements

The database is interacted with via structured s-expression
statements. You won't be concatenating strings on your own. (And it
leaves out any possibility of a SQL injection!) See the "Usage"
section above for examples. A statement is a vector of keywords and
other lisp object.

Structured Emacsql statements are compiled into SQL statements. The
statement compiler is memoized so that using the same statement
multiple times is fast. To assist in this, the statement can act as a
template -- using `$1`, `$2`, etc. -- working like the Elisp `format`
function.

### Keywords

Rather than the typical uppercase SQL keywords, keywords in a
structured Emacsql statement are literally just that: lisp keywords.
When multiple keywords appear in sequence, Emacsql will generally
concatenate them with a dash, e.g. `CREATE TABLE` becomes
`:create-table`.

 * `:create-table <ident> <schema>`

Provides `CREATE TABLE`.

    ex. [:create-table employees [name (id integer :primary) (salary float)]]

 * `:drop-table <ident>`

Provides `DROP TABLE`.

    ex. [:drop-table employees]

 * `:select <column-spec>`

Provides `SELECT`. `column-spec` can be a `*` symbol or a vector of
column identifiers, optionally as expressions.

    ex. [:select [name (/ salary 52)] ...]

 * `:from <ident>`

Provides `FROM`.

    ex. [... :from employees]

### Templates

To make statement compilation faster, and to avoid making you build up
statements dynamically, you can insert `$n` "variables" in place of
identifiers and values. These refer to argument positions after the
statement in the `emacsql` function, 1-indexed.

    (emacsql db [:select * :from $1 :where (> salary $2)] 'employees 50000)

## Limitations

Emacsql is *not* intended to play well with other programs accessing
the SQLite database. Non-numeric values are are stored encoded as
s-expressions TEXT values. This avoids ambiguities in parsing output
from the command line and allows for storage of Emacs richer data
types. This is a high-performance database specifically for Emacs.


[readable]: http://nullprogram.com/blog/2013/12/30/#almost_everything_prints_readably
