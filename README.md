## Intro

This is my *first* Jupyter Notebook in VSCodium, I'm excited to see what it'll look like!

I'll be going through Python's [sqlite3 tutorial](https://docs.python.org/3/library/sqlite3.html) within this notebook.


## Tutorial Prerequisites

- A **database cursor** allows access to rows within a "results set" (one or more rows).  The cursor operates on a single row at a time. ([Wikipedia](https://en.wikipedia.org/wiki/Cursor_(databases)))

- A **database transaction** is a "complete unit of work" consisting of one or more operations. ([Wikipedia](https://en.wikipedia.org/wiki/Database_transaction))
    - ex, from wiki: Transferring money from one bank acct to another is a single transaction with two operations:
        1. Remove money from sender acct
        2. Add money to recipient acct
        
    - **ACID** ([link](https://en.wikipedia.org/wiki/ACID))
        - A = *atomic*; must finish completely or, if failure, have no effect on the DB state/data
        - C = *consistent*; DB state must maintain consistency - any data written must meet all defined rules (constraints, cascades, triggers, etc.)
            - Cascade = sequence of transactions that are rolled back due to a later transaction being rolled back/undone ([link](https://en.wikipedia.org/wiki/Rollback_(data_management)))
            - Trigger = code that is automatically executed in response to certain events on a particular table/view ([link](https://en.wikipedia.org/wiki/Database_trigger))
        - I = *isolated*; concurrent transactions leave database in same final state as same set of transactions, executed sequentially
        - D = *durable*; once completed, a transaction will remain committed even if system fails (power outage/crash).  Aka, store info to non-volatile memory (hard drive), not RAM.

## Tutorial


```python
import sqlite3
con = sqlite3.connect('tutorial.db')  # open connection to DB (either existing or implicitly created).  Returns Connection object.
```


```python
cur = con.cursor()  # we need a "cursor" to execute SQL queries!

# Making table called "movie" with three columns ("title", "year", "score").
# Specifying the per-column data type is optional in SQLite, and if specified is just the "preferred" datatype (advisory, not mandatory).
#   SQLite has flexible typing: https://www.sqlite.org/flextypegood.html
cur.execute("CREATE TABLE movie(title, year, score)")

# Verify that the table was correctly created by looking at the Schema Table (https://www.sqlite.org/schematab.html)
#   Every SQLite DB has one - it stores the schema (a description of all other tables, indices, triggers, views (one row per aforementioned entity/object))
res = cur.execute("SELECT name FROM sqlite_master")
res.fetchone()
```




    ('movie',)




```python
# We are allowed to query tables for non-existent entries -- for example, trying to find a table we haven't created in the sqlite_master schema table.
res = cur.execute("SELECT name FROM sqlite_master WHERE name='spam'")
res.fetchone() is None
```




    True




```python
# Adding data to our movie table.
# Apparently, the INSERT statement implicitely opens a transaction, but this behavior can be modified by changing sqlite3 module's isolation_level attribute.
#   https://docs.python.org/3/library/sqlite3.html#sqlite3-controlling-transactions 
cur.execute("""
    INSERT INTO movie VALUES
        ('Monty Python and the Holy Grail', 1975, 8.2),
        ('And Now for Something Completely Different', 1971, 7.5)
""")

# The transaction has been implicitly opened, we've made changes, and it's pending until we commit it
# (note that it's the Connection, not the Cursor, committing)
con.commit()
```


```python
# Verify that the data was correctly inserted
res = cur.execute("SELECT score FROM movie")
res.fetchall()  # return all resulting rows
```




    [(8.2,), (7.5,)]




```python
# Let's insert more rows!
data = [
    ("Monty Python Live at the Hollywood Bowl", 1982, 7.9),
    ("Monty Python's The Meaning of Life", 1983, 7.5),
    ("Monty Python's Life of Brian", 1979, 8.0),
]
# 1. executemany allows iteration over a list of values to be inserted into the statement
# 2. "?" is a placeholder -- use this instead of python string formatting/interpolation to avoid SQL injection attacks :O
cur.executemany("INSERT INTO movie VALUES(?, ?, ?)", data)
con.commit()

# Now, verify insertion.  Cur.execute supports iteration over the result -- is this the same as calling .fetchall() on the result and iterating over that?
for row in cur.execute("SELECT year, title FROM movie ORDER BY year"):
    print(row)
```

    (1971, 'And Now for Something Completely Different')
    (1975, 'Monty Python and the Holy Grail')
    (1979, "Monty Python's Life of Brian")
    (1982, 'Monty Python Live at the Hollywood Bowl')
    (1983, "Monty Python's The Meaning of Life")



```python
# Finally -- let's verify that the data was written to the disk!
# 1. Close our connection
con.close()

# 2. Open a new connection / cursor and make a query
new_con = sqlite3.connect('tutorial.db')
new_cur = new_con.cursor()
res = new_cur.execute("SELECT title, year FROM movie ORDER BY score DESC")
title, year = res.fetchone()
print(f'Highest scoring MP movie is {title!r} ({year}).')
new_con.close()
```

    Highest scoring MP movie is 'Monty Python and the Holy Grail' (1975).

