<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# Design

## Scope

The deck sends StatsD™ metrics over UDP to a StatsD, DogStatsD™ or Telegraf™
agent: counters (`count`, `increment`, `decrement`, `countFloat`), gauges
(`gauge`, `gaugeFloat`), timers (`timing`) and sets (`set`), with the
optional sample-rate (`countRate`, `timingRate`) and DogStatsD tag
(`countTagged` and the other `*Tagged` verbs) extensions, and a `Batch` that
packs several metrics into as few datagrams as fit.

It deliberately leaves out:

- **Delivery guarantees.** StatsD is fire-and-forget: no reply, no error when
  no agent listens. That is the point of the protocol (a metric costs one
  datagram and never blocks the program), so the deck adds no
  acknowledgement or retry.
- **Client-side sampling and aggregation.** The `*Rate` verbs only annotate
  the line; the caller decides which events to send. Aggregation is the
  agent's job.
- **TCP and Unix-socket transports**, and the DogStatsD events, service
  checks and distribution / histogram types.

## Decisions

- **A deck, not a bundled module.** A client for one family of agents belongs
  in the deck ecosystem. The deck is one of the bundled modules that
  `jennifer` moves out (M30.1 in its milestones, formerly M28.32) and keeps
  the module's surface, so migrating changes only the import.
- **A `Client` holds an open socket.** `clientWith` binds a sending UDP
  socket once; every verb reuses it, and copies of the `Client` share it.
  `close` releases it.
- **Resolve once.** `clientWith` checks the `host:port` address and resolves
  a host name when the client is made, preferring an IPv4 address (agents
  often listen on IPv4 only) and bracketing an IPv6 one, so a send never
  waits for DNS. A client made before the agent moves keeps the old address.
- **Gauges are always absolute.** StatsD reads a leading sign on a gauge as a
  change, so a negative value is sent as `name:0|g` then `name:-5|g`, both in
  one datagram so they cannot arrive apart, out of order or with another
  sender's update between. A float gauge of zero (including `-0.0`) is sent
  as `0`.
- **Strict lines.** A metric name or prefix is letters, digits, `_`, `-` and
  `.`-separated segments, none empty; a value holds no control character,
  `:` or `|`; a tag key holds no control character, `|`, `,` or `:`, a tag
  value none of the first three; a tag key is not empty (an empty value
  sends the bare key). A sample rate must be greater than 0 (the agent
  divides by it); a rate of 1 or more sends no suffix. StatsD packs metrics
  separated by newlines and fields by `:` and `|`, so without these checks
  request data put into a name (`"http.ua." + $userAgent`) could forge
  extra metrics.
- **Datagrams that are not fragmented.** `flush` splits a batch, between
  lines, into datagrams of at most 1432 bytes: what fits an Ethernet frame
  with room for tunnels, the size Etsy™ StatsD recommends. A fragmented
  datagram is often dropped whole. A single longer line goes alone.
- **Value-semantic `Batch`.** Each `add*` returns a new batch with the line
  appended, validated as it is added, and the prefix is captured from the
  client at `batch` time. Copying makes building a batch quadratic in its
  length, so batches are meant to be flushed every few hundred metrics.

## Errors

The deck's own checks throw an `Error` of kind `"statsd"`: a bad name,
prefix, value, tag or rate, an address without a port from 1 to 65535, and
a host name that resolves to no address. These are caller input, not a
server's answer, so there is no error struct (as recorded for the
`@jennifer` decks in M28.57: statsd never hears back).

A host name that cannot be resolved is `net`'s `net.NetError` (kind `"net"`,
`op` `"lookup"`, `reason` `"dns"`), passed on as it is so its fields stay
readable; on interpreters without transport error kinds (M28.65) the lookup
failure is wrapped as a `"statsd"` error. Sends never report a missing
agent; a local socket failure comes from `net` unchanged.

## Security and limits

- The line checks above make every verb safe for untrusted names, values and
  tags: no input can add a line, a field or a tag.
- Datagrams are capped at 1432 bytes in `flush`; the single-metric verbs
  send one line each.
- The deck sends plain UDP with no authentication, as the protocol does. Keep
  the agent on the local host or a trusted network.

## Testing and validation

`src/statsd_test.j` covers the pure line building (prefix, rate and tag
suffixes, float formatting, negative gauges), every refused input, and the
wire: sends to a UDP socket on a loopback port, read back datagram by
datagram, including the gauge pairs, host-name resolution to IPv4, the IPv6
form and the splitting of a large batch at the packet cap. A host name that
does not resolve (under `.invalid`) is shown to arrive as a `net.NetError`.

The module's rate, tag, float and batch work (M23.7) was validated against
an independent reference parser when it was added. `examples/push.j` sends
to an agent named on its command line (default `127.0.0.1:8125`) and
succeeds whether or not one listens, so CI runs it offline.

## Known limitations and open questions

- A host name is resolved once; a long-running program whose agent moves
  needs a new client.
- `Batch` building copies on every step; flushing often keeps it cheap.
- No DogStatsD distributions, events or service checks.

## Dependencies on the interpreter

- The floor is `>=0.27.0`, for the UTF-8 codecs of `encoding.encode` /
  `decode`; the bundled module is deprecated from 0.26.0.
- The `net` capability (UDP sockets, `net.lookup`), so the deck runs on the
  default `jennifer` binary only (`pragma-jennifer-capability: net`).
- A failed lookup as `net.NetError` needs transport error kinds (M28.65).
