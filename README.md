# socks5-nv

SOCKS5 is a protocol for asking a proxy server to open a connection on
the caller's behalf. It is specified in
[RFC 1928](https://www.rfc-editor.org/rfc/rfc1928), and the user name
and password sub-negotiation it can use is
[RFC 1929](https://www.rfc-editor.org/rfc/rfc1929). This package is the
protocol without a socket: the messages as values and as bytes, and a
client and a server as state machines that the caller drives with the
bytes it reads.

## What SOCKS5 is

A client connects to the proxy and speaks in four steps.

1. **The greeting.** The client lists the authentication methods it can
   use. The server answers with the one it chose, or with 255 when it
   accepts none.
2. **The authentication,** if the method needs one. Method 2 is a user
   name and a password, sent in one message and answered with a status.
3. **The request.** The client names a command and a destination. The
   destination is an IPv4 address, a domain name or an IPv6 address.
4. **The reply.** The server answers with a reply code and an address
   and port of its own. After a successful reply, every byte on the
   connection belongs to the application.

There are three commands. **CONNECT** opens a TCP connection to the
destination. **BIND** asks the server to listen for one inbound TCP
connection and answers twice: once with the address it listens on, and
again when the peer connects. **UDP ASSOCIATE** asks the server for a
UDP relay. The client then sends each datagram to the relay behind a
header that names the destination, and the relay returns datagrams with
a header that names the sender.

| Quantity | Value |
| --- | --- |
| Version byte of every message | 5 |
| Version byte of the user name and password sub-negotiation | 1 |
| Method: no authentication | 0 |
| Method: GSS-API | 1 |
| Method: user name and password | 2 |
| Method: no acceptable method | 255 |
| Commands | 1 CONNECT, 2 BIND, 3 UDP ASSOCIATE |
| Address types | 1 IPv4, 3 domain name, 4 IPv6 |
| Reply codes assigned | 0 to 8 |
| Longest domain name, user name or password | 255 bytes |
| Methods in one greeting | 1 to 255 |

## Install

```
novo pkg add socks5-nv
```

## Example

The program below builds a client for a CONNECT to `example.com`
port 443 and feeds it the first answer a proxy can give. It does not
open a socket. The bytes in `step.output` are what a program writes to
the proxy, and the bytes it reads from the proxy go to `client_feed`.

```novo
use sockscode
use socksaddr
use socksclient

fn main() [io]
    let config = socksclient.client_config(SocksConnect, SocksAddrDomain("example.com"), 443)
    match socksclient.client_start(config)
        Err(e)   => println(e.message())
        Ok(step) =>
            // The greeting: version 5, one method, method 0.
            println("write ${list.len(step.output)} bytes")

            // The proxy chose method 0, so the request goes out next.
            let chose_none: [u8] = [5 as u8, 0 as u8]
            let next = socksclient.client_feed(step.client, chose_none)
            println("write ${list.len(next.output)} bytes")
```

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `sockscode` | The authentication methods, the three commands and the nine reply codes, each as a number with a name. |
| `sockserr` | The refusals a message can meet, and the check that tells a message cut short from a message that is wrong. |
| `socksaddr` | An IPv4 address, a domain name or an IPv6 address as one value, its bytes in both directions, and its text. |
| `sockswire` | The greeting, the method selection, the user name and password messages, the request, the reply and the UDP header, each with an encoder and a decoder. |
| `socksclient` | The client as a state machine: the configuration, the greeting to send, and the events a read completes. |
| `socksserver` | The server as a state machine: it picks a method, hands the caller the credentials and the request, and writes the caller's answers. |

## How to choose an entry point

**`socksclient` is for a program that connects through a proxy.** It
takes the destination and the credentials once, and every read from the
proxy goes to `client_feed`.

**`socksserver` is for a program that is a proxy.** It reads the
greeting, the credentials and the request, and asks the caller to
decide. The caller opens the outbound connection and relays the data.

**`sockswire` is for a program that handles single messages.** A tool
that records a handshake, a test that builds a request, and a UDP relay
that reads the header of each datagram all use the encoders and decoders
directly.

## The rules a user needs

1. **A message cut short is not a fault.** Every decoder answers
   `SocksTruncated` when the bytes stop before the message does, with the
   number of bytes it needs and the number it has. On a stream the
   caller keeps what it has and reads more. `sockserr.is_incomplete`
   tells this case from the others. The state machines do this
   themselves: they keep the bytes of a partial message.
2. **A domain name goes to the proxy as text.** The proxy resolves it.
   A client that resolves the name itself and sends an address gives the
   resolver, not the proxy, the name of the host it reaches.
3. **A reply other than 0 ends the exchange.** RFC 1928 section 6
   requires the connection to be closed soon after a failure. The client
   reports `SocksClientRefused` and does not continue.
4. **The bytes after the reply are the tunnel's.** A server may send
   data directly after its reply. `SocksClientEstablished` carries those
   bytes as `leftover`, so that a read which holds both loses neither.
5. **BIND has two replies.** The first, reported as
   `SocksClientListening`, names where the server listens. The second,
   reported as `SocksClientEstablished`, names the peer that connected.
   A server calls `server_reply` twice.
6. **The user name and password are sent unencrypted.** RFC 1929
   section 2 says so. The exchange protects nothing against an observer
   of the connection to the proxy.
7. **The server decides who may pass.** `socksserver` reads the
   credentials and reports them as `SocksServerCredentialsRead`. The
   server writes a status only when the caller calls
   `server_authenticate`. A rejection writes status 1 and ends the
   exchange.
8. **An unknown command is answered with reply 7 and an unknown address
   type with reply 8.** The decoders read a command number the RFC does
   not define as `SocksCommandOther`, so that the server can name it in
   its refusal. The server does this itself (RFC 1928 section 6).
9. **The reserved bytes must be zero.** A request, a reply or a UDP
   header with a non-zero reserved field is refused.
10. **A UDP datagram is whole when its FRAG field is 0.** The header is
    read and the field is reported; `udp_is_standalone` tells the two
    apart. A relay that does not implement fragmentation drops every
    other datagram (RFC 1928 section 7). This package does not
    reassemble fragments.
11. **The association lasts as long as the TCP connection that created
    it.** For UDP ASSOCIATE the reply names the relay's address and port.
    The caller keeps the TCP connection open for as long as it uses the
    relay (RFC 1928 section 6).
12. **Lengths are checked in both directions.** A domain name, a user
    name and a password are 1 to 255 bytes. An encoder refuses anything
    else and writes nothing.

## What is not included

- **Sockets.** The caller opens the connection and moves the bytes.
- **GSS-API authentication (RFC 1961).** Method 1 can be offered, but no
  exchange for it is implemented, and a server that chooses it is
  reported as `SocksNoAcceptableMethod`.
- **SOCKS4 and SOCKS4a.** Their messages have a different layout.
- **Resolving names.** A domain name is passed to the proxy.
- **Reassembling UDP fragments.** Fragments are read and reported.
- **A relay.** The package parses and builds the UDP header. Opening the
  UDP socket and forwarding the datagrams is the caller's.
- **The embedded tier.** Messages are lists of bytes and strings, which
  allocate.

## Related packages

- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) is a
  protocol codec of the same shape: bytes in, values out, no socket.
- [url-nv](https://novo-lang.org/packages/url-nv) reads the
  `socks5://user:password@host:port` form that names a proxy.

## Tests

```bash
novo test tests/socks_wire_tests.nv      # the messages, byte by byte
novo test tests/socks_machine_tests.nv   # the client and the server, driven with bytes
novo test tests/socks_code_tests.nv      # the numbers, their names and the refusals
bash tests/coverage.sh                   # line coverage over src/, merged across the suites
```

The vectors in `tests/socks_wire_tests.nv` are written out by hand from
the diagrams of RFC 1928 sections 3 to 7 and RFC 1929 section 2, with the
field names beside each byte. The machine tests drive a client and a
server against each other, feed an exchange one byte at a time, and
check each refusal.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
