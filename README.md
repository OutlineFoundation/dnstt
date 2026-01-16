# Userspace DNS tunnel with support for DoH and DoT

This is a fork of the [original `dnstt` repository](https://www.bamsoftware.com/software/dnstt/)
for exploration and investigation on how to make it work with the Outline SDK. See the [original README](README).

## How the client works

The `dnstt-client` acts as a bridge between local TCP applications and the `dnstt-server` through a DNS tunnel. It works by encapsulating TCP streams into encrypted, reliable sessions that are transported inside standard DNS queries (DoH, DoT, or UDP).

Here is the step-by-step breakdown of its operation:

### 1. Initialization and Configuration

*   **Startup:** The client starts by parsing command-line arguments to determine the transport mode (DoH, DoT, or UDP), the tunnel domain (e.g., `t.example.com`), the server's public key, and the local listening address (e.g., `127.0.0.1:7000`).
*   **uTLS:** If configured, it initializes a specific uTLS fingerprint (e.g., Firefox, Chrome) to mask its TLS handshake signatures when using DoH or DoT.

### 2. Transport Layer Abstraction

The client establishes a connection to a recursive resolver using one of three implementations of `net.PacketConn`:
*   **DoH (`HTTPPacketConn`):** Sends DNS queries as HTTP POST requests to a DoH resolver (e.g., `https://doh.example/dns-query`). It handles HTTP rate limiting (`Retry-After`).
*   **DoT (`TLSPacketConn`):** Maintains a persistent TLS connection to a DoT resolver (e.g., `8.8.8.8:853`) and sends length-prefixed DNS messages. It automatically reconnects if the connection drops.
*   **UDP:** Uses a standard UDP socket to send DNS packets directly to a resolver or the tunnel server.

### 3. DNS Encapsulation (`DNSPacketConn`)

This is the core "tunneling" logic. The `DNSPacketConn` wraps the chosen transport and handles the conversion between raw bytes and DNS messages:
*   **Encoding (Sending):**
    1.  Takes a chunk of data (from the upper layers).
    2.  Adds a length prefix and random padding.
    3.  Prefixes a unique `ClientID`.
    4.  Base32-encodes the entire payload.
    5.  Splits the encoded string into 63-byte labels and appends the tunnel domain (e.g., `...base32data... .t.example.com`).
    6.  Constructs a DNS `TXT` query for this name and sends it via the transport.
*   **Decoding (Receiving):**
    1.  Receives a DNS response.
    2.  Extracts the `TXT` record from the Answer section.
    3.  Decodes the Base32 data to retrieve the inner payload.

### 4. Protocol Stack (The "Tunnel" Inside)

Inside the DNS packets, `dnstt` runs a full networking stack to ensure the connection is useful:
1.  **KCP (Reliability):** Since DNS is unreliable and unordered, KCP is used to provide a reliable, ordered stream of data (like TCP) over the packet-based DNS transport.
2.  **Noise (Encryption):** The KCP stream is encrypted using the Noise protocol (Noise_NK_25519_ChaChaPoly_BLAKE2s). This ensures that the recursive resolver (and anyone watching the DNS traffic) cannot read the data, only the tunnel server can.
3.  **smux (Multiplexing):** On top of the encrypted stream, `smux` allows multiple concurrent logical streams. This enables you to handle multiple TCP connections (e.g., multiple web requests) over the single tunnel session.

### 5. Main Loop (`run`)

*   **Listening:** The client listens on the local TCP port (e.g., `127.0.0.1:7000`).
*   **Forwarding:** When a user application (like `nc` or `ssh`) connects to this port:
    1.  The client accepts the connection.
    2.  It opens a new `smux` stream within the existing tunnel session.
    3.  It pipes data bidirectionally between the local TCP connection and the `smux` stream.
*   **Polling:** Since DNS is a request-response protocol, the server cannot push data to the client spontaneously. The client's `sendLoop` actively sends "polling" queries (empty queries) to the server to ask "Do you have data for me?" so the server can reply with data in the TXT record.

## Running end-to-end locally

It's possible to run the system end-to-end locally without registering a domain name. This is helpful for development.

### 1. Generate the keys:

```console
$ go run www.bamsoftware.com/git/dnstt.git/dnstt-server@latest -gen-key
privkey 04be6acdc398b0779fe538a4e042b7309b143006ae6d883a1ba808469f5b0f85
pubkey  370d43f47ce12c1361c19a68c335729030a57065b4c8dc762422d1ec5b628d32
```

### 2. Run the remote port forwarder (server)

You need to point to the backend you want to talk to. In this example, `example.com`.

```sh
go run www.bamsoftware.com/git/dnstt.git/dnstt-server@latest -udp :5300 -privkey 04be6acdc398b0779fe538a4e042b7309b143006ae6d883a1ba808469f5b0f85 t.example.com example.com:80
```

### 3. Run the local port forwarder (client)

You need to pass the server address and a local address that is forwarded to the server. In this example, `127.0.0.1:8001`.

```sh
go run www.bamsoftware.com/git/dnstt.git/dnstt-client@latest -pubkey 370d43f47ce12c1361c19a68c335729030a57065b4c8dc762422d1ec5b628d32 -udp 127.0.0.1:5300 t.example.com 127.0.0.1:8001
```

### 4. Test

Connect to the local address to observe the end-to-end connection.

```console
$ curl --connect-to ::127.0.0.1:8001 http://example.com
<!doctype html><html lang="en"><head><title>Example Domain</title><meta name="viewport" content="width=device-width, initial-scale=1"><style>body{background:#eee;width:60vw;margin:15vh auto;font-family:system-ui,sans-serif}h1{font-size:1.5em}div{opacity:0.8}a:link,a:visited{color:#348}</style><body><div><h1>Example Domain</h1><p>This domain is for use in documentation examples without needing permission. Avoid use in operations.<p><a href="https://iana.org/domains/example">Learn more</a></div></body></html>
```
