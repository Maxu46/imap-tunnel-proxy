# IMAP Tunnel - Technical Documentation

This document provides in-depth technical details about the IMAP Tunnel Proxy, including protocol design, DPI evasion techniques, security analysis, and implementation details.

> For basic setup and usage, see [README.md](README.md).

---

## Table of Contents

- [Why IMAP?](#-why-imap)
- [How It Bypasses DPI](#-how-it-bypasses-dpi)
- [Why It's Fast](#-why-its-fast)
- [Architecture](#-architecture)
- [Protocol Design](#-protocol-design)
- [Component Details](#-component-details)
- [Security Analysis](#-security-analysis)
- [Domain vs IP Address](#-domain-name-vs-ip-address-security-implications)
- [Advanced Configuration](#-advanced-configuration)

---

## Why IMAP?

IMAP (Internet Message Access Protocol) is used for accessing emails from a mail server. It's an excellent choice for tunneling because:

### 1. Ubiquitous Traffic
- Email access is essential infrastructure - blocking it breaks legitimate services
- IMAP traffic on port 993 (IMAPS) is expected and normal
- Millions of users access their email via IMAP every second

### 2. Expected to be Encrypted
- Port 993 (IMAPS) implies TLS-encrypted connections
- DPI systems expect to see TLS-encrypted IMAP traffic
- No red flags for encrypted content

### 3. Flexible Protocol
- IMAP connections are typically long-lived (IDLE command)
- Session-based access patterns are normal
- Multiple commands per session is expected

### 4. Hard to Block
- Blocking port 993 would break email access for everyone
- Can't easily distinguish tunnel from real mail access after TLS
- Would require blocking all encrypted email access

---

## How It Bypasses DPI

Deep Packet Inspection (DPI) systems analyze network traffic to identify and block certain protocols or content. Here's how IMAP Tunnel evades detection:

### Phase 1: The Deception (Plaintext)

```
+--------------------------------------------------------------+
|                    DPI CAN SEE THIS                          |
+--------------------------------------------------------------+
|                                                              |
|  Server: * OK imap.example.com IMAP4rev1 Service Ready       |
|  Client: A001 CAPABILITY                                     |
|  Server: * CAPABILITY IMAP4rev1 STARTTLS LOGINDISABLED       |
|          A001 OK CAPABILITY completed                        |
|  Client: A002 STARTTLS                                       |
|  Server: A002 OK Begin TLS negotiation now                   |
|                                                              |
|  DPI Analysis: "This is a normal email access connection"    |
|                                                              |
+--------------------------------------------------------------+
```

**What DPI sees:**
- Standard IMAP greeting from mail server
- Normal capability negotiation
- STARTTLS upgrade (expected for secure email access)

**What makes it convincing:**
- Greeting matches real IMAP servers
- Capabilities list is realistic
- Proper RFC 3501 compliance
- Port 993 is standard IMAPS port

### Phase 2: TLS Handshake

```
+--------------------------------------------------------------+
|                    DPI CAN SEE THIS                          |
+--------------------------------------------------------------+
|                                                              |
|  [TLS 1.2/1.3 Handshake]                                     |
|  - Client Hello                                              |
|  - Server Hello                                              |
|  - Certificate Exchange                                      |
|  - Key Exchange                                              |
|  - Finished                                                  |
|                                                              |
|  DPI Analysis: "Normal TLS for email access encryption"      |
|                                                              |
+--------------------------------------------------------------+
```

**What DPI sees:**
- Standard TLS handshake
- Server certificate for mail domain
- Normal cipher negotiation

### Phase 3: Encrypted Tunnel (Invisible)

```
+--------------------------------------------------------------+
|                   DPI CANNOT SEE THIS                        |
|                   (Encrypted with TLS)                       |
+--------------------------------------------------------------+
|                                                              |
|  Client: A003 CAPABILITY                                     |
|  Server: * CAPABILITY IMAP4rev1 AUTH=PLAIN IDLE ...          |
|          A003 OK CAPABILITY completed                        |
|  Client: A004 AUTHENTICATE PLAIN <token>                     |
|  Server: A004 OK Logged in                                   |
|  Client: BINARY                                              |
|  Server: * 299 Binary mode activated                         |
|                                                              |
|  [Binary streaming begins - raw TCP tunnel]                  |
|                                                              |
|  DPI Analysis: "Encrypted email session, cannot inspect"     |
|                                                              |
+--------------------------------------------------------------+
```

**What DPI sees:**
- Encrypted TLS traffic
- Packet sizes and timing consistent with email access
- Cannot inspect content

**What actually happens:**
- Authentication with pre-shared key
- Switch to binary streaming mode
- Full-speed TCP tunneling

### Why DPI Can't Detect It

| DPI Technique | Why It Fails |
|---------------|--------------|
| **Port Analysis** | Uses standard IMAP port 993 |
| **Protocol Detection** | Initial handshake is valid IMAP |
| **TLS Fingerprinting** | Standard Python SSL library |
| **Packet Size Analysis** | Variable sizes, similar to email |
| **Timing Analysis** | No distinctive patterns |
| **Deep Inspection** | Content encrypted with TLS |

---

## Why It's Fast

### The Approach: Protocol Upgrade

```
+-------------------------------------------------------------+
|                    HANDSHAKE PHASE                          |
|                    (One time only)                          |
+-------------------------------------------------------------+
|  CAPABILITY -> STARTTLS -> TLS -> CAPABILITY -> AUTH -> BINARY |
|                                                             |
|  Time: ~200-500ms (network latency dependent)               |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                    STREAMING PHASE                          |
|                    (Rest of session)                        |
+-------------------------------------------------------------+
|                                                             |
|  +---------+------------+------------+-------------+        |
|  |  Type   | Channel ID |   Length   |   Payload   |        |
|  | 1 byte  |  2 bytes   |  2 bytes   |  N bytes    |        |
|  +---------+------------+------------+-------------+        |
|                                                             |
|  - Full duplex - send and receive simultaneously            |
|  - No waiting for responses                                 |
|  - 5 bytes overhead per frame                               |
|  - Raw binary - no base64 encoding                          |
|  - Speed limited only by network bandwidth                  |
|                                                             |
+-------------------------------------------------------------+
```

### Performance

| Metric | Binary Streaming |
|--------|------------------|
| **Overhead per packet** | 5 bytes |
| **Round trips per send** | 0 (streaming) |
| **Encoding overhead** | 0% |
| **Duplex mode** | Full-duplex |
| **Effective speed** | Limited by bandwidth |

---

## Architecture

### System Components

```
YOUR COMPUTER                           YOUR VPS                        INTERNET
+--------------------+                  +--------------------+          +---------+
|                    |                  |                    |          |         |
|  +--------------+  |                  |  +--------------+  |          | Website |
|  |   Browser    |  |                  |  |    Server    |  |          |   API   |
|  |   or App     |  |                  |  |   server.py  |  |          | Service |
|  +------+-------+  |                  |  +------+-------+  |          |         |
|         |          |                  |         |          |          +----+----+
|         | SOCKS5   |                  |         | TCP      |               |
|         v          |                  |         v          |               |
|  +--------------+  |   TLS Tunnel     |  +--------------+  |               |
|  |    Client    |<-+----------------->|->|   Outbound   |<-+---------------+
|  |   client.py  |  |   Port 993       |  |  Connector   |  |
|  +--------------+  |                  |  +--------------+  |
|                    |                  |                    |
+--------------------+                  +--------------------+
     Censored Network                      Free Internet
```

### Data Flow

```
1. Browser wants to access https://example.com

2. Browser -> SOCKS5 (client.py:1080)
   "CONNECT example.com:443"

3. Client -> Server (port 993, looks like IMAP)
   [FRAME: CONNECT, channel=1, "example.com:443"]

4. Server -> example.com:443
   [Opens real TCP connection]

5. Server -> Client
   [FRAME: CONNECT_OK, channel=1]

6. Browser <-> Client <-> Server <-> example.com
   [Bidirectional data streaming]
```

---

## Protocol Design

### Frame Format (Binary Mode)

All communication after handshake uses this simple binary frame format:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |          Channel ID           |    Length     |
+---------------+-------------------------------+---------------+
|    Length     |            Payload...                         |
+---------------+-----------------------------------------------+
|                        Payload (continued)                    |
+---------------------------------------------------------------+

Type (1 byte):
  0x01 = DATA         - Tunnel data
  0x02 = CONNECT      - Open new channel
  0x03 = CONNECT_OK   - Connection successful
  0x04 = CONNECT_FAIL - Connection failed
  0x05 = CLOSE        - Close channel

Channel ID (2 bytes): Identifies the connection (supports 65535 simultaneous connections)
Length (2 bytes): Payload size (max 65535 bytes)
Payload (variable): The actual data
```

### CONNECT Payload Format

```
+---------------+-------------------------+---------------+
|  Host Length  |         Host            |     Port      |
|   (1 byte)    |    (variable, UTF-8)    |   (2 bytes)   |
+---------------+-------------------------+---------------+
```

### Session State Machine

```
                    +---------+
                    |  START  |
                    +----+----+
                         |
                         v
              +---------------------+
              |   TCP Connected     |
              +----------+----------+
                         | * OK greeting
                         v
              +---------------------+
              |   CAPABILITY        |
              +----------+----------+
                         | TAG OK
                         v
              +---------------------+
              |     STARTTLS        |
              +----------+----------+
                         | TAG OK
                         v
              +---------------------+
              |   TLS Handshake     |
              +----------+----------+
                         | Success
                         v
              +---------------------+
              |   CAPABILITY        |
              |   (post-TLS)        |
              +----------+----------+
                         | TAG OK
                         v
              +---------------------+
              |   AUTHENTICATE      |
              +----------+----------+
                         | TAG OK
                         v
              +---------------------+
              |   BINARY Command    |
              +----------+----------+
                         | * 299 OK
                         v
              +---------------------+
              |   Binary Streaming  |<------+
              |   (Full Duplex)     |-------+
              +---------------------+
```

---

## Component Details

### server.py - Server Component

**Purpose:** Runs on your VPS in an uncensored network. Accepts tunnel connections and forwards traffic to the real internet.

**What it does:**
- Listens on port 993 (IMAPS)
- Presents itself as an IMAP mail server
- Handles IMAP handshake (CAPABILITY, STARTTLS, AUTHENTICATE)
- Switches to binary streaming mode after authentication
- Manages multiple tunnel channels
- Forwards data to destination servers
- Sends responses back through the tunnel

**Key Classes:**
| Class | Description |
|-------|-------------|
| `TunnelServer` | Main server, accepts connections |
| `TunnelSession` | Handles one client connection |
| `Channel` | Represents one tunneled TCP connection |

### client.py - Client Component

**Purpose:** Runs on your local computer. Provides a SOCKS5 proxy interface and tunnels traffic through the server.

**What it does:**
- Runs SOCKS5 proxy server on localhost:1080
- Connects to tunnel server on port 993
- Performs IMAP handshake to look legitimate
- Switches to binary streaming mode
- Multiplexes multiple connections over single tunnel
- Handles SOCKS5 CONNECT requests from applications

**Key Classes:**
| Class | Description |
|-------|-------------|
| `TunnelClient` | Manages connection to server |
| `SOCKS5Server` | Local SOCKS5 proxy |
| `Channel` | One proxied connection |

### common.py - Shared Utilities

**Purpose:** Code shared between client and server.

**What it contains:**
| Component | Description |
|-----------|-------------|
| `TunnelCrypto` | Handles authentication tokens |
| `TrafficShaper` | Padding and timing (optional stealth) |
| `IMAPMessageGenerator` | Generates realistic content (legacy) |
| `FrameBuffer` | Parses binary frames from stream |
| `load_config()` | YAML configuration loader |
| `ServerConfig` | Server configuration dataclass |
| `ClientConfig` | Client configuration dataclass |

### generate_certs.py - Certificate Generator

**Purpose:** Creates TLS certificates for the tunnel.

**What it generates:**
| File | Description |
|------|-------------|
| `ca.key` | Certificate Authority private key |
| `ca.crt` | Certificate Authority certificate |
| `server.key` | Server private key |
| `server.crt` | Server certificate (signed by CA) |

**Features:**
- Customizable hostname in certificate
- Configurable key size (default 2048-bit RSA)
- Configurable validity period
- Includes proper extensions for TLS server auth

---

## Security Analysis

### Authentication Flow

```
+-------------------------------------------------------------+
|                  Authentication Flow                         |
+-------------------------------------------------------------+
|                                                             |
|  1. Client generates timestamp                              |
|                                                             |
|  2. Client computes:                                        |
|     HMAC-SHA256(secret, "imap-tunnel-auth:" + timestamp)    |
|                                                             |
|  3. Client sends: AUTHENTICATE PLAIN base64(timestamp:hmac) |
|                                                             |
|  4. Server verifies:                                        |
|     - Timestamp within 5 minutes (prevents replay)          |
|     - HMAC matches (proves knowledge of secret)             |
|                                                             |
|  5. Server responds: TAG OK Logged in                       |
|                                                             |
+-------------------------------------------------------------+
```

### Encryption Layers

| Layer | Protection |
|-------|------------|
| **TLS 1.2+** | All traffic after STARTTLS |
| **Pre-shared Key** | Authentication |
| **HMAC-SHA256** | Token integrity |

### Threat Model

| Threat | Mitigation |
|--------|------------|
| Passive eavesdropping | TLS encryption |
| Active MITM | Certificate verification (requires domain) |
| Replay attacks | Timestamp validation (5-minute window) |
| Unauthorized access | Pre-shared key authentication |
| Protocol detection | IMAP mimicry during handshake |

### Security Recommendations

1. **Use a strong secret:** Generate with `python -c "import secrets; print(secrets.token_urlsafe(32))"`

2. **Keep secret secure:** Never commit to version control, share securely

3. **Use certificate verification:** Copy `ca.crt` to client and set `ca_cert` in config

4. **Restrict server access:** Use whitelist to limit source IPs if possible

5. **Monitor logs:** Watch for failed authentication attempts

6. **Update regularly:** Keep Python and dependencies updated

---

## Domain Name vs IP Address: Security Implications

### Understanding TLS Certificate Verification

TLS certificates are digital documents that prove a server's identity. When your client connects to a server, it can verify:

1. **The certificate is signed by a trusted authority** (in our case, your own CA)
2. **The certificate matches who you're connecting to** (hostname/IP verification)

### The IP Address Problem

TLS certificates store identifiers in specific fields within the **Subject Alternative Name (SAN)** extension:

| Identifier Type | SAN Field Type | Example |
|-----------------|----------------|---------|
| Domain name | `DNSName` | `imap.example.com` |
| IP address | `IPAddress` | `192.168.1.100` |

**These are different field types.** A certificate generated with `--hostname 192.168.1.100` creates a `DNSName` entry, not an `IPAddress` entry, causing verification to fail.

### Man-in-the-Middle Attack

When certificate verification is disabled, an attacker can intercept your connection:

```
WITHOUT Certificate Verification (ca_cert not set):

  +--------+       +------------+       +------------+       +--------+
  | Client |------>|  Attacker  |------>|  Firewall  |------>| Server |
  |        |<------|  (MITM)    |<------|   (DPI)    |<------|        |
  +--------+       +------------+       +------------+       +--------+

  Attacker presents their own certificate, client accepts it,
  YOUR TRAFFIC IS COMPLETELY EXPOSED TO THE ATTACKER

WITH Certificate Verification (ca_cert set + domain name):

  +--------+       +------------+
  | Client |------>|  Attacker  |
  |        |   X   |  (MITM)    |
  +--------+       +------------+

  Client checks certificate: "This isn't signed by my CA!"
  CONNECTION REFUSED - Attack blocked!
```

### Security Options Comparison

| Configuration | MITM Protected? | Works? | Recommended? |
|---------------|-----------------|--------|--------------|
| Domain + `ca_cert` set | **YES** | YES | **BEST** |
| Domain + no `ca_cert` | NO | YES | Not ideal |
| IP address + `ca_cert` set | — | NO | Won't work |
| IP address + no `ca_cert` | NO | YES | Vulnerable |

---

## Advanced Configuration

### Full Configuration Reference

```yaml
# ============================================================================
# Server Configuration (for server.py on VPS)
# ============================================================================
server:
  # Interface to listen on
  host: "0.0.0.0"

  # Port to listen on (993 = IMAPS, recommended)
  port: 993

  # Hostname for IMAP greeting and TLS certificate
  hostname: "imap.example.com"

  # TLS certificate files
  cert_file: "server.crt"
  key_file: "server.key"

  # Users configuration file
  users_file: "users.yaml"

  # Global logging setting
  log_users: true

# ============================================================================
# Client Configuration (for client.py on local machine)
# ============================================================================
client:
  # Server domain name (FQDN required for certificate verification)
  server_host: "yourdomain.duckdns.org"

  # Server port (must match server config)
  server_port: 993

  # Local SOCKS5 proxy port
  socks_port: 1080

  # Local SOCKS5 bind address
  socks_host: "127.0.0.1"

  # Username for authentication
  username: "your-username"

  # Pre-shared secret (from server admin)
  secret: "your-secret"

  # CA certificate for server verification (RECOMMENDED)
  ca_cert: "ca.crt"
```

### IMAP Protocol Compliance

The tunnel implements these IMAP RFCs during handshake:
- **RFC 3501** - Internet Message Access Protocol - Version 4rev1
- **RFC 2595** - Using TLS with IMAP, POP3 and ACAP
- **RFC 4959** - IMAP Extension for SASL Initial Client Response

### Multiplexing

Multiple TCP connections are multiplexed over a single tunnel:

```
+-------------------------------------------------------------+
|                    Single TLS Connection                     |
+-------------------------------------------------------------+
|                                                             |
|  Channel 1: Browser Tab 1 -> google.com:443                 |
|  Channel 2: Browser Tab 2 -> github.com:443                 |
|  Channel 3: curl -> ifconfig.me:443                         |
|  Channel 4: SSH -> remote-server:22                         |
|  ...                                                        |
|  Channel 65535: Maximum concurrent connections              |
|                                                             |
+-------------------------------------------------------------+
```

### Memory Usage

- **Server:** ~50MB base + ~1MB per active connection
- **Client:** ~30MB base + ~0.5MB per active channel

### Concurrency Model

Both client and server use Python's `asyncio` for efficient handling of multiple simultaneous connections without threads.

---

## Version Information

- **Current Version:** 1.0.0
- **Protocol Version:** Binary streaming v1
- **Minimum Python:** 3.8
