# Changelog

All notable changes to socks5-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

First release.

### Added

- The messages of RFC 1928: the greeting, the method selection, the
  request and the reply in the IPv4, domain name and IPv6 forms, and the
  UDP request header.
- The username and password sub-negotiation of RFC 1929.
- The reply codes, the commands and the methods as values with names.
- A client and a server as state machines that a caller drives with the
  bytes it reads. CONNECT, BIND and UDP ASSOCIATE are supported.

### Not included

- GSS-API authentication (RFC 1961), SOCKS4 and SOCKS4a, UDP fragment
  reassembly, and any socket.

This release requires novo 0.24.0 or later.
