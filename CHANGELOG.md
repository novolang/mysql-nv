# Changelog

## 0.0.1 — interface

The interface, published before anything is implemented: every `pub fn`
body is a `todo()`, and the signatures, the effect rows and the tests are
the design.

- Eight modules.  `mypacket`, `myauth`, `mytype` and `myerror` are `[]`
  throughout — bytes in, messages out, and not one socket; `myconn`,
  `myquery`, `mypool` and `mydriver` declare `[net]` and `[time]` and
  nothing else.
- 113 public functions and seven trait members, every body a
  `todo("mysql-nv.<module>.<fn>")`.
- Four test suites, 41 tests, red on purpose against the protocol's own
  constants.
- `LOAD DATA LOCAL INFILE` is a value the caller answers, and the
  capability is not negotiated by default — which is why this package
  declares no `[fs]` at all.
