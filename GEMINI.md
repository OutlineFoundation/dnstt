# dnstt - DNS Tunnel

`dnstt` is a userspace DNS tunnel that supports DNS over HTTPS (DoH), DNS over TLS (DoT), and plaintext UDP DNS. It allows tunneling TCP connections through a DNS resolver, effectively bypassing network restrictions or hiding traffic.

## Project Overview

*   **Language:** Go (1.21+)
*   **Key Features:**
    *   **Transport:** DoH, DoT, UDP DNS.
    *   **Reliability:** Uses KCP and smux for reliable stream management over unreliable datagrams.
    *   **Security:** End-to-end encryption using the Noise protocol (Noise_NK_25519_ChaChaPoly_BLAKE2s). Authenticates the server.
    *   **Covertness:** Client supports uTLS to mimic browser TLS fingerprints.

## Directory Structure

*   `dnstt-client/`: Source code for the tunnel client application.
*   `dnstt-server/`: Source code for the tunnel server application.
*   `dns/`: DNS packet handling and construction logic.
*   `noise/`: Implementation of the Noise protocol for encryption.
*   `turbotunnel/`: Core tunneling logic, including reliability and session management.
*   `man/`: Man pages for the client and server.

## Building and Running

### Building

The project contains two main applications: the client and the server. Both are built using standard Go commands.

**Build Client:**
```bash
cd dnstt-client
go build
```

**Build Server:**
```bash
cd dnstt-server
go build
```

### Running

**Server:**
The server requires a domain name and a private key.
1.  Generate keys: `./dnstt-server -gen-key -privkey-file server.key -pubkey-file server.pub`
2.  Run:
    ```bash
    ./dnstt-server -udp :5300 -privkey-file server.key t.example.com 127.0.0.1:8000
    ```
    (Note: You typically forward external port 53 to 5300 or run as root).

**Client:**
The client connects to the server via a recursive resolver (DoH/DoT) or directly (UDP).
1.  Run (DoH example):
    ```bash
    ./dnstt-client -doh https://doh.example/dns-query -pubkey-file server.pub t.example.com 127.0.0.1:7000
    ```

## Development Conventions

*   **Code Style:** Standard Go formatting (`gofmt`).
*   **Dependencies:** Managed via `go.mod`.
*   **Testing:** Go tests are present in `_test.go` files (e.g., `dns/dns_test.go`). Run with `go test ./...`.
