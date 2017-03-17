# sqltrace

This package is a database/sql/driver wrapper. It adds support for tracing
database calls by using the OpenTracing API. When driver methods are called
with a context containing an OpenTracing span, sub-spans are created for all
database operations.

Furthermore, transactions and prepared statements save the OpenTracing span
passed to them through the context. If such a span exists, all operations on
that transaction or prepared statement are traced even if a context is not
passed to them explicitly.

## Usage

```go
import (
    "context"
    "database/sql"

    "github.com/lib/pq"
    "github.com/rrraaazzz/sqltrace"
    opentracing "github.com/opentracing/opentracing-go"
)

func main() {
    // wrap the Postgres driver
    driver := sqltrace.New(pq.Open)
    sql.Register("traced_pq", driver)

    // open the database using the name you just registered,
    // and the normal connection string.
    db, err := sql.Open("traced_pq", "dbname=pqgotest")

    // initialize an OpenTracing tracer 
    ...

    // create a parent span and store it in the context
    span := opentracing.StartSpan("main")
    defer span.Finish()
    ctx := opentracing.ContextWithSpan(context.Background(), span)

    // use the database normally, but pass the context to enable tracing
    tx, err := db.BeginTx(ctx, &sql.TxOptions{})

    // this is traced using the parent span passed to BeginTx
    rows, err := tx.Query(...)

    // this is also traced, since the parent span passes down from
    // the transaction
    done := rows.Next()

    ...

    tx.Commit()
   
    // this is not traced, since a context is not given, and there
    // is no other way for a span to propagate to the call
    db.Exec("...")
}

```

## What data is stored in spans 

The spans are created with the following names:

- transactions: begin\_transaction, commit, rollback
- prepared statements: prepare\_statement, close\_statement
- query, exec: query, exec
- rows: next\_row, next\_result\_set, close\_rows

The spans can contain the following tags:

- component: sqltrace
- db.type: sql
- span.kind: client
- db.statement: for prepared statements, query, exec contains the sql string before parameter substitution
- db.last\_insert\_id, db.rows\_affected: correspond to sql.Result data, populated for exec calls
- error: true, for any traced call that fails
