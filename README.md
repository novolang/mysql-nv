# mysql-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A MySQL and MariaDB client, in novo-lang, with no libmysqlclient.

The protocol is documented, stable and thirty years old, and every
language that talks to MySQL without C has ported it — Go, Rust, Java,
Python.  This is that port: the packet framing, the two authentication
plugins, the text and binary query protocols, the column types, `LOAD
DATA LOCAL` as a value the caller answers, TLS as a hook, a pool and a
`std.sql` driver.

Eight modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **frame** | `mypacket` | anything. Start here |
| the **login** | `myauth` | you are debugging a handshake |
| the **values** | `mytype` | you are reading or binding a column |
| the **faults** | `myerror` | something went wrong |
| the **connection** | `myconn` | you want the socket, or TLS |
| the **queries** | `myquery` | you want rows |
| the **pool** | `mypool` | you have more than one request at a time |
| the **contract** | `mydriver` | you want a `dyn Database` |

113 public functions and seven trait members, every body a `todo()`.

## Adding it, and checking it

```bash
novo pkg add mysql-nv             # into your novo.toml
novo pkg build                    # type- and effect-check the package
novo test --isolate tests/packet_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: mysql-nv.<module>.<fn>`.  Four
suites, 41 tests.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use myconn
use myquery
use mytype

// Ask a parameterised question, in the binary protocol.
fn count_after(id: Int) -> Int [net, time]
    match myconn.connect(myconn.default_options("app"), myconn.plain_transport())
        Err(f) => -1
        Ok(c) =>
            match myquery.query(c, "SELECT count(*) FROM t WHERE id > ?", [MyInt(v: id)])
                Err(f) => -1
                Ok(r)  => 1
```

The value goes as a parameter, not into the string.  MySQL's text
protocol has no parameters at all, so a parameterised query is a
prepared one — and `query` does that round trip for a caller that does
not want to hold a statement.

## The load-bearing interface

`MyCapabilities`, and the rule that **the same bytes are different
messages under different flags.**

```novo norun:pseudo
pub fn kind_of(buf: Bytes, p: MyPacket, c: MyCapabilities) -> MyPacketKind []
pub fn looks_like_eof(buf: Bytes, p: MyPacket, c: MyCapabilities) -> Bool []
pub fn read_ok(buf: Bytes, p: MyPacket, c: MyCapabilities) -> Result<MyOk, MyPacketFault> []
pub fn read_error(buf: Bytes, p: MyPacket, c: MyCapabilities) -> MyServerError []
```

**This is the difference from postgres-nv, and it is the whole shape of
the package.**  In PostgreSQL a message's type byte determines its shape
for all time, so `pgmsg.decode_backend` takes bytes and nothing else.
In MySQL the wire format is *negotiated*: with `CLIENT_DEPRECATE_EOF`
the server sends an OK packet where it used to send an EOF, with
`CLIENT_PROTOCOL_41` an error packet carries a SQLSTATE and without it
does not, and `CLIENT_FOUND_ROWS` changes what `affected_rows` means.

So every decode here takes the negotiated capabilities, `myconn.MyConn`
carries them, and `myconn.capabilities` publishes them for a caller
reading packets itself.  A decoder that cached the wrong value produces
plausible garbage rather than an error — which is why the parameter is
required at every call rather than remembered in a global.

Three more rules come with the frame, each a public function because
each one is a silent bug elsewhere.

**One: the sequence number is the client's to keep.**  Every packet
carries a one-byte counter that resets to 0 at each command and
increments per packet.  The server checks it and answers `Packets out of
order`, which is not a network problem and not a query problem.
`MySeq` is a value the connection threads, `mypacket.command_seq` is the
reset, and `myconn.send_command` is where it happens so no call site can
forget.

**Two: a 16 MB payload is split, and the last piece may be empty.**  The
length field is three bytes, so a payload of exactly `0xFFFFFF` is
continued — and one that is an exact multiple of it is followed by a
packet of length ZERO.  A reader that took the length at face value
truncates a large row; one that stopped at the first short packet hangs
on the terminator.  `mypacket.is_continued`.

**Three: `0xFE` is three different things.**  It begins an EOF packet,
it prefixes an 8-byte length-encoded integer, and it is a legal first
byte of row data.  Which one depends on the packet's LENGTH and on the
capabilities.  `mypacket.looks_like_eof` carries the whole rule; a
parser that branched on the byte alone misreads any row beginning with
it.

## Where this differs from postgres-nv, and why

The two packages are deliberately the same shape — a `[]` codec half and
a `[net]` client half in one `host` package, a transport hook for TLS, a
pool that is a value, a `std.sql` driver — and they differ in six places
because the protocols do.

| | postgres-nv | mysql-nv |
| --- | --- | --- |
| **message shape** | fixed by the type byte | negotiated: every decode takes `MyCapabilities` |
| **framing** | a 4-byte length, one message per frame | a 3-byte length and a sequence id, with a 16 MB split |
| **end of exchange** | `ReadyForQuery`, carrying the transaction status | nothing: the status rides every OK packet's flags |
| **parameters** | the extended query protocol, either format | only a prepared statement has parameters at all |
| **cancellation** | an out-of-band `CancelRequest` on a fresh socket | `KILL QUERY <id>`, an ordinary statement on a second connection |
| **the dangerous feature** | none in the client | `LOAD DATA LOCAL INFILE` — see below |

The one that changes the API most is the third.  postgres-nv's
load-bearing rule is "only `ReadyForQuery` ends an exchange", and there
is no such message here: MySQL's transaction status is a bit in the
status flags of whatever OK packet last arrived, so `mypacket.in_transaction`
takes an `MyOk` and `myconn.in_transaction` reads the connection's last
one.  A caller that wants the same guarantee has to look at the flags
after every statement, which is what the pool does before it takes a
connection back.

The second-biggest is cancellation.  PostgreSQL's cancel is a designed
out-of-band request; MySQL's is `KILL QUERY` sent from somewhere else,
so a caller that wants to cancel must be able to open a second
connection at the moment it is needed.  That is why `MyConn` carries the
server's thread id from the handshake rather than looking it up —
looking it up needs the connection that is busy.

## `LOAD DATA LOCAL INFILE`, and why this package has no `[fs]`

Mid-exchange, a MySQL server can send a `0xFB` packet carrying a path
and expect the client to send that file's contents.  The path is the
SERVER's choice.  A compromised server, a hostile one, or one reached
through a man in the middle can therefore ask a client for any file the
client process can read — `~/.ssh/id_rsa`, a credentials file, the
application's own configuration — and a client that honours the request
sends it.

This package's answer is that **it never reads a file**:

- the capability is not negotiated by default
  (`MyLocalInfileOff`), so the server cannot ask at all;
- a caller that genuinely bulk-loads sets
  `MyLocalInfileUnder(directory)`, and `myquery.local_path_allowed`
  checks the request AFTER resolving `..` and symbolic links, because a
  comparison on the unresolved string lets `/data/../etc/passwd`
  through;
- the request itself arrives as a VALUE — `MyLocalFileWanted(path)` —
  and the caller reads the bytes and passes them to
  `myquery.send_local_file`, or refuses with
  `myquery.refuse_local_file`.

So the `[fs]` that the feature costs is spent in the caller's program,
where a reader of its manifest can see it, and this package's rows stay
`[net]` and `[time]`.

## Authentication

`myauth` is `[]` — no socket, no clock, no randomness — and the server's
nonce is the caller's argument, which is what makes the handshake
testable against a captured exchange.

**`caching_sha2_password` has a fast path and a slow path, and the slow
path is where the password goes.**  The fast path is a scramble the
server checks against its cache and is safe on a plain socket.  When the
cache misses — a fresh server, a restarted one, a user's first
connection — the server asks for full authentication, and the client
must send the password where the server can read it: in the clear over
TLS, or RSA-encrypted under the server's public key.  Two refusals,
because in each case the plausible behaviour is the insecure one:

- **sending the password in the clear over an unencrypted socket** —
  refused unless `allow_cleartext`, because otherwise a cache miss
  silently downgrades every connection that ever hits one;
- **fetching the server's RSA public key over that same unencrypted
  socket** — refused unless the caller pinned one, because a key an
  attacker in the middle substituted encrypts the password to the
  attacker.

`mysql_native_password` is SHA-1 and is implemented rather than refused,
because refusing it means refusing to connect to MariaDB and to every
MySQL 5.7.  It is weak — the stored verifier is password-equivalent, so
anybody who can read the user table can log in as that user — and
`myauth.is_weak_plugin` is the predicate a caller with a policy can
refuse on.

**An empty password sends a zero-length response**, under both plugins.
A client that scrambled the empty string sends 20 bytes the server
rejects with "access denied", which sends everybody looking at the
password.

## TLS is a hook, not a dependency

A driver that chose a TLS implementation would choose it for every
program that links the driver.  `myconn.MyTransport` is the seam: three
named functions that move bytes, with `myconn.plain_transport` the
`std.net` pair.  MySQL's STARTTLS is one short packet carrying the
capability flags and nothing else, so `negotiate_tls` is a separate call
with the caller's handshake between it and the login — and a client that
put its username in that first packet has sent it in the clear.

`MySslMode` has five values and no default, because **a server that does
not offer TLS has not failed.**  It has said no, and a client that
carried on has silently downgraded a connection the user asked to
encrypt.  `MySslRequire` and above refuse.

## The `std.sql` driver

`mydriver.MyDatabase` implements the prelude's `Database` trait, so a
program written against `dyn Database` runs over this client, over
postgres-nv and over sqlite-nv with nothing in it naming any of them.

**What the effect row costs, stated rather than discovered.**  The
trait's members declare `[io]`; this impl declares `[net, time]`, which
is legal — a trait with no effect parameter does not pin its impls'
rows.  The bill is that **a `dyn Database` call is charged the UNION
over every impl in the program** (SPEC § 5.6), so a program that links
this package makes every `dyn Database` call in it cost `[net, time]`,
including the ones that only ever hold a file.  Concrete receivers are
charged their own rows, so the union is only paid where the engine
really is unknown.

This is the same obstruction postgres-nv's `pgdriver` records, and it
has the same fix: an effect parameter on the trait.  It is a **contract
change** and belongs in a feature file; this package names it rather
than working around it, and it is the widening this lane found.

## What the wire gets wrong quietly, and where each one has a name

| the mistake | what it costs | where it is named |
| --- | --- | --- |
| one socket read treated as one packet | works on localhost, fails under load | `mypacket.frame_length` |
| the length field trusted | an allocation the wire asked for | `frame_length`'s `max_bytes` |
| the sequence number not tracked | `Packets out of order` on the second statement | `mypacket.next_seq` |
| a 16 MB row read as one packet | a truncated value, or a hang | `mypacket.is_continued` |
| `0xFE` branched on as a byte | a row beginning with it read as end-of-set | `mypacket.looks_like_eof` |
| the NULL bitmap's two-bit offset | every prepared column shifted by one | `mytype.binary_null_at` |
| `DECIMAL` converted to `Float` | money, rounded | `MyDecimal` |
| a NULL read as an empty string | two different values collapsed | `MyNull` |
| `TIME` modelled as a clock time | half its range unrepresentable | `MyTime` |
| `0000-00-00` refused | tables that exist cannot be read | `MyZeroDate` |
| `utf8` asked for instead of `utf8mb4` | truncated at the first emoji | `mytype.utf8mb4_collation` |
| `SERVER_MORE_RESULTS_EXISTS` ignored | the next query reads the last one's rows | `mypacket.more_results` |
| a connection returned mid-transaction | the next borrower inside somebody else's | `mypool.release` |
| `affected_rows` read as "matched" | an update that changed nothing looks like a failure | `myquery.affected_rows` |
| a deadlock retried from the failing statement | half a transaction committed | `myerror.rolled_back` |

## What is out of scope, out loud

**Replication and binlog.**  `COM_BINLOG_DUMP` and the row-based event
format are a package of their own, and a client that half-implemented
them would be a client that silently skips events.

**The old `COM_CHANGE_USER` and `COM_PROCESS_INFO`.**  Administrative
commands with no portable behaviour across MySQL and MariaDB.

**Server-side cursors.**  `COM_STMT_FETCH` and the cursor flag on
`COM_STMT_EXECUTE`.  Worth having and not needed for the first
implementation; the row that wants it is a report over a table larger
than memory.

**Compression.**  `CLIENT_COMPRESS` wraps every packet in a second
framing layer with its own length and its own sequence number, and it is
a measurable win only over a slow link.  It is a second decoder, and the
first one should work first.

**`async`.**  Every call blocks its task.  `std.net` has `recv_async`
and this package does not use it yet; the row that wants it is a server
handling many connections per cell, and the change is an effect row and
a second set of entry points rather than a redesign.

**A pool that synchronises.**  `mypool.MyPool` is a VALUE, so it does
not synchronise anything and cannot: two tasks sharing one share it the
way they share any other value in this language.  `cell.pool` is how
novo-lang programs own shared state, and a pool per cell with no sharing
is the shape that actually runs.

## The reference implementations

`go-sql-driver/mysql` and `mysql_async`, for the API shape; MySQL's own
[Client/Server Protocol](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html)
chapter for the wire, which is the normative document and what the
module headers transcribe; MariaDB's own protocol documentation for the
places the two forks disagree, which is mostly authentication and the
EOF deprecation.

The implementation lane's gate is a real server — a `mysqld`, a socket,
a corpus of statements, and the same queries through `mysql` for
comparison — with **MariaDB beside it**, because a client tested against
only one of the two forks passes and then fails to log in.

## Status

Interface only.  Eight modules, 113 public functions and seven trait
members, every body a `todo()`.

- `novo pkg build` — clean, 8 modules checked.
- `novo test` — four suites, all red, every failure `not implemented`.
- `scripts/shard_audit.sh --strict` — `effect-budget`, `dep-layer`,
  `no-discharge-in-core`, `doc-examples` and `docs-pub` green; `test`
  red by design.

## Licence

Apache-2.0.
