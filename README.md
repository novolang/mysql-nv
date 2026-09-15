# mysql-nv

MySQL is a relational database server, and MariaDB is the fork of it
that most Linux distributions ship. Both speak the MySQL
client/server protocol, which is documented in full in the
[Client/Server Protocol](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html)
chapter of the MySQL server manual. This package speaks that protocol
in novo-lang, with no `libmysqlclient` underneath it: the packet
framing, the two authentication plugins, the text and binary query
protocols, the column types, a connection pool and a `std.sql` driver.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the protocol is

A client opens a socket, the server sends a **handshake** naming its
version, a 20-byte random **scramble** and the features it supports,
and the client answers with a **handshake response** naming the user,
the features it wants and a value derived from the password and the
scramble. From then on both ends send **packets**.

A packet is a 3-byte payload length, a 1-byte **sequence id**, and the
payload. The sequence id resets to 0 at the start of each command and
increments by one per packet, wrapping at 256. The server checks it
and answers `Packets out of order` when it does not match.

The features the two ends agreed are the **capabilities**, a 32-bit
flag word. **The capabilities decide what the bytes mean.** With
`CLIENT_DEPRECATE_EOF` the server sends an OK packet where it used to
send an EOF packet. With `CLIENT_PROTOCOL_41` an error packet carries
a five-character **SQLSTATE** and without it does not. With
`CLIENT_FOUND_ROWS` the affected-row count means rows matched rather
than rows changed. So every decode in this package takes the
negotiated capabilities as an argument.

A statement is sent in one of two protocols. The **text protocol** is
`COM_QUERY`: the statement as a string, and every value in the answer
as characters, so the integer 42 arrives as the two bytes `4` and `2`.
It has no parameters at all. The **binary protocol** is a **prepared
statement**: the text is sent once with `?` placeholders, the values
are sent separately, and the answer arrives in each column's own
layout with a **NULL bitmap** at the front. A parameterised query is
therefore always a prepared one.

An exchange ends with an **OK packet** or an **error packet**. The OK
packet carries the server's **status flags**, and the transaction's
state is one of them. There is no separate end-of-exchange message.

`LOAD DATA LOCAL INFILE` is a statement that makes the server ask the
client for a file. Mid-exchange the server sends a packet beginning
`0xFB` and carrying a path, and a client that honours it sends that
file's contents. **The path is the server's choice.** This package
never reads a file: the request arrives as a value, and the caller
decides.

The numbers every packet is measured against are these.

| Quantity | Value |
| --- | --- |
| Packet header | 4 bytes: a 3-byte length and a 1-byte sequence id |
| Largest payload in one packet | 16,777,215 bytes (`0xFFFFFF`) |
| Sequence id | 0 to 255, wrapping |
| Service port | 3306 |
| Server scramble | 20 bytes |
| SQLSTATE | 5 characters, and absent without `CLIENT_PROTOCOL_41` |
| Largest EOF packet | 9 bytes |
| `TIME` range | −838:59:59 to 838:59:59 |

Three error numbers decide what a caller does next.

| Number | SQLSTATE | Meaning | What to do |
| --- | --- | --- | --- |
| 1062 | 23000 | Duplicate key | Not retryable |
| 1213 | 40001 | Deadlock, and the transaction was rolled back | Re-run the whole transaction |
| 1205 | HY000 | Lock wait timeout | Not retryable |

## Install

```
novo pkg add mysql-nv
```

## Example

```novo
use myconn
use myerror
use myquery
use mytype

fn main() [io, net, time]
    // Where to connect and as whom. `plain_transport` is the standard
    // library's socket; a program that wants TLS supplies its own.
    let opts = myconn.default_options("app")

    match myconn.connect(opts, myconn.plain_transport())
        Err(f) => println("could not connect")
        Ok(c)  =>
            // The value goes as a parameter and never into the string.
            // MySQL's text protocol has no parameters, so this prepares
            // the statement, executes it and closes it again.
            match myquery.query(c, "SELECT name FROM users WHERE id > ?", [MyInt(v: 100)])
                Err(f) => println("the statement failed")
                Ok(r)  =>
                    match r.outcome
                        // Rows came back. Read one cell by column name.
                        MyRows(rows) =>
                            match myquery.value_named(rows, 0, "name")
                                Some(v) => println(mytype.value_text(v))
                                None    => println("no such row or column")
                        // No rows: an INSERT, an UPDATE, a CREATE TABLE.
                        MyAffected(ok) => println("${myquery.affected_rows(r.outcome)} rows changed")
                        // The server is asking this client for a file.
                        // Refusing is the default answer.
                        MyLocalFileWanted(path) =>
                            match myquery.refuse_local_file(r.conn)
                                Ok(_)  => println("refused ${path}")
                                Err(_) => println("refused, and the connection went with it")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: mysql-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mypacket` | The frame, the sequence number, the capability flags, the length-encoded integer and string, and the OK packet with its status flags. |
| `myauth` | The handshake, the two authentication plugins, the full-authentication decision, and the password encryption that goes with it. |
| `mytype` | A column definition, the two row encodings, the NULL bitmap, and the value type that keeps a decimal a decimal. |
| `myerror` | The server's error packet, its number and its SQLSTATE, and the predicates that say whether a caller may retry. |
| `myconn` | The socket, the connect and the login, the TLS hook, the two safety policies, and the command round trip that keeps the sequence number right. |
| `myquery` | Sending statements in both protocols, reading result sets, the prepared-statement lifetime, and the local-file request. |
| `mypool` | A pool of connections as a value, its policy, and the session reset that makes a returned connection safe to lend again. |
| `mydriver` | The `std.sql` `Database` implementation, so a program can hold a database without naming an engine. |

The first four modules perform no input or output at all. Every
function in them is arithmetic over bytes the caller already holds.
The last four declare `[net]`, and `[time]` where a deadline is
consulted. No module in this package declares `[fs]`.

## How to choose an entry point

**`myquery.exec` sends a statement with no parameters**, in the text
protocol. It is the right call for DDL and for a statement whose text
is entirely the program's own.

**`myquery.query` sends a statement with parameters**, in one call. It
prepares the statement, executes it in the binary protocol, reads the
answer and closes the statement again. Anything carrying a value from
outside the program goes through this.

**`myquery.prepare` and `execute` keep the statement.** Use them when
the same statement runs many times, which is where the round trip
saved is worth holding server state for. `close_statement` gives it
back, and `reset_statement` clears the long data sent into it.

**`mypool` is for a program serving more than one request at a time.**
`acquire` takes a connection, `release` gives it back after resetting
the session.

**`mydriver.MyDatabase` is for a program that must not name an
engine.** It implements the standard library's `Database` trait, so
the same code runs over this client, over postgres-nv and over
sqlite-nv. The trait has no row surface, so a caller that wants rows
calls `mydriver.query` and has chosen MySQL by then.

**`mypacket`, `myauth`, `mytype` and `myerror` together are the whole
protocol with no socket.** That is the path for a proxy, a capture
reader and a test, and none of it performs anything.

## The rules a user needs

1. **Every decode takes the negotiated capabilities.** The same bytes
   are different messages under different flags. `myconn.MyConn`
   carries them and `myconn.capabilities` publishes them for a caller
   reading packets itself. A decoder given the wrong value produces
   plausible output rather than an error.
2. **The sequence number is the client's to keep.** `MySeq` is the
   value, `mypacket.command_seq` is the reset at the start of a
   command, and `mypacket.next_seq` advances it.
   `myconn.send_command` does both, so a caller using it cannot
   forget. `Packets out of order` is this rule broken, and it is
   neither a network fault nor a query fault.
3. **A payload of exactly 16,777,215 bytes is continued by another
   packet**, and a payload that is an exact multiple of that number is
   followed by a packet of length zero to say it ended.
   `mypacket.is_continued` is the rule. A reader that takes the length
   at face value truncates a large row; one that stops at the first
   short packet waits forever for the terminator.
4. **`0xFE` is three different things.** It begins an EOF packet, it
   prefixes an 8-byte length-encoded integer, and it is a legal first
   byte of row data. Which one it is depends on the packet's length
   and on the capabilities. `mypacket.looks_like_eof` carries the
   whole rule, and a parser that branches on the byte alone misreads
   any row starting with it.
5. **Bound the packet size before the first read.**
   `MyConnectOptions.max_packet_bytes` is the ceiling, and
   `mypacket.frame_length` takes it. Without one, a length field on
   the wire is an allocation the wire asked for.
6. **The row format is an argument to every row decode.** A text row
   sends every value as characters; a binary row sends each value in
   its column's own layout. `mytype.MyRowFormat` says which, and it is
   never remembered from a previous decode.
7. **Read the NULL bitmap before the values of a binary row.** A NULL
   in a binary row is a bit in the bitmap and the value is absent from
   the payload entirely, so a decoder that skipped the bitmap reads
   every later column at the wrong offset. The bitmap in a result row
   is offset by two bits, which `mytype.binary_null_at` applies.
8. **A NULL is not an empty string.** In a text row NULL is the length
   prefix `0xFB` and an empty string is a length of zero. `MyNull` is
   a variant of `MyValue` and never an empty buffer.
9. **A `DECIMAL` stays text.** The server sends it as characters, and
   a driver that parses it into a `Float` has rounded somebody's
   money. `MyDecimal` carries the digits.
10. **Read the column flags, not only the type code.** `UNSIGNED_FLAG`
    turns a `BIGINT` into a range no signed 64-bit integer holds, and
    `MyUnsignedText` carries those digits. `BINARY_FLAG` turns a
    `VARCHAR` into bytes rather than text.
11. **`TIME` is a duration, not a clock reading.** It runs from
    −838:59:59 to 838:59:59, and a driver that modelled it as a time
    of day cannot hold half its range.
12. **`0000-00-00` is a value MySQL stores.** No calendar has it, and
    a driver that refused it could not read tables that exist. It is
    its own variant, `MyZeroDate`.
13. **Ask for `utf8mb4`, not `utf8`.** MySQL's `utf8` holds three
    bytes per character and truncates at the first four-byte one.
    `mytype.utf8mb4_collation` is the collation to connect with.
14. **Check `SERVER_MORE_RESULTS_EXISTS` after every statement.** A
    stored procedure answers several result sets in a row, and that
    flag is the only thing that says so. A client that ignores it
    leaves packets on the socket, and the next query reads the
    previous statement's rows. `mypacket.more_results` reads the flag
    and `myquery.next_result` reads the next set.
15. **The transaction's state rides the OK packet's status flags.**
    This protocol has no end-of-exchange message.
    `mypacket.in_transaction` reads an OK packet and
    `myconn.in_transaction` reads the connection's last one. A pool
    checks it before taking a connection back.
16. **`affected_rows` counts rows changed, not rows matched**, unless
    `CLIENT_FOUND_ROWS` was negotiated. An update that set a column to
    the value it already held counts zero.
17. **A deadlock rolls the whole transaction back.**
    `myerror.rolled_back` says so, and re-running from the failing
    statement commits half a transaction. `myerror.is_retryable` is
    the list of error numbers a retry is correct for.
18. **Branch on both the error number and the SQLSTATE.** The number
    is MySQL's own and precise; the SQLSTATE is the ANSI class, which
    is portable and vague. A caller reading only the SQLSTATE cannot
    tell a deadlock from a lock wait timeout. Both are on
    `MyServerError`.
19. **Cancelling a statement needs a second connection.** MySQL has no
    out-of-band cancel: a caller sends `KILL QUERY <id>` from
    somewhere else. `MyConn` carries the server's thread id from the
    handshake for that reason, because looking it up would need the
    connection that is busy. `myconn.kill_query` is the call.
20. **A connection returned to the pool is reset.** MySQL keeps
    temporary tables, user variables, prepared statements, the current
    database, `SET` variables and the transaction across a release.
    `mypool.release` runs `COM_RESET_CONNECTION`, which clears all of
    it in one round trip. A server too old for that command gets a
    `SET`-based fallback, and `mypool.resets_on_release` says which is
    in use.

## Authentication

`myauth` performs nothing: no socket, no clock and no randomness. The
server's scramble comes in as an argument, so a whole handshake
reproduces byte for byte from a captured exchange.

**`caching_sha2_password` has a fast path and a slow path.** The fast
path is a scramble the server checks against its cache, and it is safe
on an unencrypted socket. When the cache misses, which happens on a
fresh server, a restarted one and a user's first connection, the
server asks for **full authentication**, and the client must send the
password where the server can read it: in the clear over TLS, or
encrypted under the server's RSA public key. `MyFullAuth` is that
decision, and it is handed back to the caller rather than taken.

Two things are refused, because in each case the plausible behaviour
is the unsafe one.

| Refused | Unless | Why |
| --- | --- | --- |
| Sending the password in the clear over an unencrypted socket | `allow_cleartext` is set | One cache miss would otherwise downgrade the connection silently |
| Fetching the server's RSA public key over that same socket | a key is pinned in `pinned_public_key` | A key substituted in the middle encrypts the password to the attacker |

**`mysql_native_password` is SHA-1 and is implemented rather than
refused**, because refusing it means refusing to connect to MariaDB
and to every MySQL 5.7. It is weak: the stored verifier is
password-equivalent, so anyone who can read the user table can log in
as that user. `myauth.is_weak_plugin` is the predicate a caller with a
policy refuses on.

**An empty password sends a zero-length response**, under both
plugins. A client that scrambled the empty string sends 20 bytes the
server rejects with "access denied", which sends everyone looking at
the password.

## TLS and the two safety policies

**TLS is a hook, not a dependency.** A driver that chose a TLS library
would choose it for every program that links the driver.
`myconn.MyTransport` is the seam: three named functions that send,
receive and close. `myconn.plain_transport` is the `std.net` pair.
MySQL's own STARTTLS is a short packet carrying the capability flags
and nothing else, so `myconn.negotiate_tls` is a separate call with
the caller's TLS handshake between it and the login. A client that put
its username in that first packet has sent it in the clear.

`MySslMode` has five values and no default.

| Mode | What it does |
| --- | --- |
| `MySslDisable` | Never asks. The honest mode for a Unix socket |
| `MySslPrefer` | Asks, and carries on in the clear if the server says no |
| `MySslRequire` | Asks, and refuses the connection if the server says no |
| `MySslVerifyCa` | As require, and the certificate must chain to a trusted root |
| `MySslVerifyIdentity` | As verify-ca, and the certificate must name the host asked for |

A server that does not offer TLS has not failed. It has said no, and a
client that carried on has downgraded a connection somebody asked to
encrypt. Only `MySslVerifyIdentity` resists an active attacker.

**`LOAD DATA LOCAL INFILE` is off.** `MyLocalInfileOff` does not
negotiate `CLIENT_LOCAL_FILES` at all, so the server cannot ask. A
caller that genuinely bulk-loads sets `MyLocalInfileUnder(directory)`,
and `myquery.local_path_allowed` checks the request after resolving
`..` and symbolic links, because a comparison on the unresolved string
lets `/data/../etc/passwd` through. Either way the request arrives as
`MyLocalFileWanted(path)`, and the caller reads the bytes and calls
`myquery.send_local_file`, or calls `myquery.refuse_local_file`. The
`[fs]` that reading a file costs is spent in the caller's program.

## What is not included

- **Reading any file.** See the section above. No module here declares
  `[fs]`.
- **A TLS implementation.** `MyTransport` is where one goes.
- **Replication and the binary log.** `COM_BINLOG_DUMP` and the
  row-based event format are a package of their own, and a client that
  half-implemented them would silently skip events.
- **Server-side cursors.** `COM_STMT_FETCH` and the cursor flag on
  `COM_STMT_EXECUTE`. A report over a table larger than memory is what
  wants them.
- **Protocol compression.** `CLIENT_COMPRESS` wraps every packet in a
  second framing layer with its own length and sequence number. It is
  a second decoder, and a measurable win only over a slow link.
- **`COM_CHANGE_USER` and `COM_PROCESS_INFO`.** Administrative
  commands whose behaviour differs between MySQL and MariaDB.
- **Asynchronous calls.** Every call blocks its task. `std.net` has
  `recv_async` and this package does not use it yet.
- **A pool that synchronises.** `mypool.MyPool` is a value, so two
  tasks sharing one share it the way they share any other value in
  this language. A pool per cell with no sharing is the shape that
  runs.

## Related packages

- [postgres-nv](https://novo-lang.org/packages/postgres-nv) is the
  same shape for PostgreSQL: a codec half that performs nothing, a
  client half, a transport hook for TLS, a pool and a `std.sql`
  driver. The protocols differ in five places worth knowing about.

| | postgres-nv | mysql-nv |
| --- | --- | --- |
| Message shape | Fixed by the type byte | Negotiated: every decode takes the capabilities |
| Framing | A 4-byte length, one message per frame | A 3-byte length and a sequence id, split at 16 MB |
| End of exchange | A `ReadyForQuery` message | Nothing: the status rides every OK packet's flags |
| Parameters | The extended query protocol, either format | Only a prepared statement has parameters |
| Cancelling | An out-of-band request on a fresh socket | `KILL QUERY` on a second connection |

- [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) reads and
  writes an SQLite database file directly. No server and no socket.
- [migrate](https://novo-lang.org/packages/migrate) plans schema
  migrations and produces their SQL. It runs nothing, so a caller
  hands its steps to this package.
- [query-builder-nv](https://novo-lang.org/packages/query-builder-nv)
  builds statements as values and renders them for MySQL's dialect,
  including its backtick quoting and its `ON DUPLICATE KEY UPDATE`.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is the SHA-1
  and SHA-256 both authentication plugins are built on.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the
  civil date and time `mytype.to_civil` converts a `DATETIME` into.
- `std.sql` in the standard library opens an SQLite file by shelling
  out to the `sqlite3` binary. It is the `Database` contract this
  package implements, not a MySQL client.

## Tests

```bash
novo test tests/packet_tests.nv   # 10 tests: the frame, the sequence, the split
novo test tests/auth_tests.nv     #  7 tests: both plugins, against captured scrambles
novo test tests/type_tests.nv     # 10 tests: the two row formats and the values
novo test tests/host_tests.nv     # 14 tests: the connection, the queries and the pool
```

The wire is MySQL's own Client/Server Protocol chapter, which is the
normative document. MariaDB's protocol documentation is the reference
where the two forks disagree, which is mostly authentication and the
EOF deprecation. `go-sql-driver/mysql` and `mysql_async` are the
reference implementations for the shape of the API.

No test opens a socket. The scramble, the password and the server's
bytes are all arguments, so a handshake is a value the test writes out
and the same bytes produce the same response on every run. The suite
checks that a packet of exactly `0xFFFFFF` bytes is read as continued,
that a zero-length packet terminates the run that precedes it, that
`0xFE` at the front of a long packet is row data and not an EOF, that
an empty password sends nothing, that a binary row's NULL bitmap is
read before its values, that a `DECIMAL` stays text, and that a
connection is reset before it goes back into the pool.

The tests compile today and fail at run, each on the
`not implemented: mysql-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

Nothing is implemented. Every function below is a `todo()`.

| Module | Public surface |
| --- | --- |
| `mypacket` | The capability constructors and readers, `negotiate`, the five capability flag accessors, `frame_length`, `read_packet`, `is_continued`, `max_payload_bytes`, `kind_of`, `looks_like_eof`, `read_ok`, `in_transaction`, `more_results`, the length-encoded readers and writer, `write_packet`, `command_seq`, `next_seq` |
| `myauth` | `read_handshake`, `plugin_of_name`, `plugin_name`, `is_weak_plugin`, `native_response`, `caching_sha2_response`, `read_auth_more_data`, `request_public_key`, `encrypt_password`, `cleartext_password`, `response`, `write_response`, `write_ssl_request` |
| `mytype` | `read_column`, `column_name`, `read_row`, `binary_null_at`, `null_bitmap_bytes`, `is_unsigned`, `is_binary`, `type_name`, `encode_parameter`, `value_text`, `is_null`, `utf8mb4_collation`, `to_civil`, `of_civil` |
| `myerror` | `read_error`, `is_retryable`, `rolled_back`, `is_constraint_violation`, `is_connection_error`, the three error-number constants, `protocol_fault`, `server_error` |
| `myconn` | `plain_transport`, `transport`, `default_options`, `is_unix_socket`, `connect`, `negotiate_tls`, `next_packet`, `send_command`, `ping`, `reset_session`, `select_database`, `close`, `kill_query`, `connection_id`, `capabilities`, `in_transaction`, `is_broken` |
| `myquery` | `exec`, `query`, `prepare`, `execute`, `close_statement`, `reset_statement`, `send_long_data`, `next_result`, `send_local_file`, `refuse_local_file`, `local_path_allowed`, `row_count`, `value_at`, `value_named`, `last_insert_id`, `affected_rows` |
| `mypool` | `default_policy`, `pool`, `acquire`, `release`, `discard`, `prune`, `close_all`, `idle_count`, `borrowed_count`, `idle_is_usable`, `resets_on_release` |
| `mydriver` | `open`, `of_conn`, `borrow`, `give_back`, `last_fault`, `conn_of`, `query`, `to_db_error`, and the `Database` members `close`, `exec` and `query_count` |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
