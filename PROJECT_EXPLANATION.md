# Kamailio Course - Complete Project Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [What is This Project?](#what-is-this-project)
3. [Architecture](#architecture)
4. [Network Topology](#network-topology)
5. [Docker Services](#docker-services)
6. [Kamailio Configuration Deep Dive](#kamailio-configuration-deep-dive)
7. [Business Requirements](#business-requirements)
8. [Course Structure](#course-structure)
9. [Key Technologies](#key-technologies)
10. [Setup and Installation](#setup-and-installation)
11. [How It All Works Together](#how-it-all-works-together)
12. [Advanced Features](#advanced-features)

---

## Project Overview

This is a **comprehensive educational project** designed to teach SIP (Session Initiation Protocol) through hands-on experimentation with Kamailio, the world's most powerful and flexible open-source SIP server.

### What is Kamailio?
Kamailio is an industrial-strength SIP proxy/router that can handle thousands of call setups per second. It's used by telecom carriers, VoIP providers, and enterprises worldwide to:
- Route SIP calls
- Act as a Session Border Controller (SBC)
- Provide load balancing
- Handle user registration
- Manage media streams
- Provide security features

### What is This Project?

This repository is the companion code for the Udemy course **"Learn SIP Through Kamailio"**. It provides a complete, working SIP infrastructure that demonstrates:

1. **Enterprise-grade SIP proxy/SBC functionality**
2. **Security best practices** (traffic filtering, IP blocking, topology hiding)
3. **Media handling** with RTPEngine for audio/video
4. **Load balancing** across multiple backend servers
5. **Advanced routing** using ENUM, dispatcher, and dynamic routing via REST API
6. **Encrypted communications** (TLS for signaling, SRTP for media)
7. **Call Detail Record (CDR) generation** for billing/analytics

---

## Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PUBLIC INTERNET                              │
│                      (192.168.254.0/24)                             │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                │ SIP Traffic (UDP:5060, TLS:5061)
                                │ RTP Media (UDP:23000-23100)
                                ▼
                    ┌───────────────────────┐
                    │   Kamailio Edge SBC   │ ← Main SIP Proxy
                    │  192.168.254.2 (pub)  │
                    │  172.16.254.2 (core)  │
                    └───────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
        ┌───────────────────┐   ┌───────────────────┐
        │   RTPEngine Edge  │   │   MySQL Database  │
        │ 192.168.254.3     │   │   172.16.254.10   │
        │ 172.16.254.3      │   │                   │
        └───────────────────┘   └───────────────────┘
                    │
┌───────────────────┴────────────────────────────────────────────────┐
│                       CORE NETWORK                                  │
│                    (172.16.254.0/24)                               │
│                                                                     │
│   ┌────────────┐  ┌────────────┐  ┌─────────┐  ┌──────────────┐  │
│   │ B2BUA      │  │ B2BUA      │  │ BIND9   │  │ REST API     │  │
│   │ Internal 1 │  │ Internal 2 │  │ DNS     │  │ (Flask)      │  │
│   │ .100:5060  │  │ .101:5060  │  │ .20:53  │  │ .30:5000     │  │
│   └────────────┘  └────────────┘  └─────────┘  └──────────────┘  │
│                                                                     │
│   ┌────────────┐  ┌────────────┐                                  │
│   │ B2BUA      │  │ B2BUA      │  (External test endpoints)       │
│   │ External 1 │  │ External 2 │                                  │
│   │ (on public)│  │ (on public)│                                  │
│   └────────────┘  └────────────┘                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Design Philosophy

This architecture follows a **defense-in-depth** approach common in carrier-grade VoIP deployments:

1. **Edge Layer**: Kamailio SBC faces the public internet and core network simultaneously
2. **Security Layer**: All traffic is inspected, filtered, and validated
3. **Media Layer**: RTPEngine handles media transcoding, encryption, and NAT traversal
4. **Core Layer**: Backend B2BUA services are isolated from the public internet
5. **Data Layer**: MySQL stores CDRs, user registrations, and routing data
6. **Support Services**: DNS (BIND9) for ENUM routing, REST API for dynamic routing

---

## Network Topology

### Two-Network Design

The project uses **two isolated Docker networks** to simulate real-world enterprise deployments:

#### 1. External Network (192.168.254.0/24)
**Purpose**: Simulates the public internet

**Connected Services**:
- `kamailio-edge`: 192.168.254.2 (public-facing interface)
- `rtpengine-edge`: 192.168.254.3 (public media endpoint)
- `b2bua_external_01`: 192.168.254.100 (test SIP client)
- `b2bua_external_02`: 192.168.254.101 (test SIP client)

**Traffic Allowed**:
- SIP signaling on UDP:5060 and TLS:5061
- RTP media on UDP:23000-23100
- All external traffic MUST go through Kamailio

#### 2. Internal Network (172.16.254.0/24)
**Purpose**: Protected core network where backend services run

**Connected Services**:
- `kamailio-edge`: 172.16.254.2 (core-facing interface)
- `rtpengine-edge`: 172.16.254.3 (core media endpoint)
- `b2bua_internal_01`: 172.16.254.100 (core SIP server 1)
- `b2bua_internal_02`: 172.16.254.101 (core SIP server 2)
- `db`: 172.16.254.10 (MySQL database)
- `bind`: 172.16.254.20 (ENUM DNS server)
- `api`: 172.16.254.30 (REST API for dynamic routing)

**Security Rule**: Direct traffic from external network to internal network is **forbidden**. All traffic must traverse Kamailio.

---

## Docker Services

### 1. Kamailio Edge SBC (`kamailio-edge`)

**Image**: `kamailio/kamailio-ci:5.4-alpine.debug`

**Purpose**: The central SIP proxy that:
- Routes all SIP signaling traffic
- Enforces security policies
- Manages user registrations
- Handles load balancing
- Performs topology hiding
- Generates call detail records

**Configuration**:
- Dual-homed: Connected to both external and internal networks
- Listens on:
  - Public side: UDP:5060, TLS:5061 (192.168.254.2)
  - Core side: UDP:5060 (172.16.254.2)
- Shared memory: 256MB
- 2 child processes for SIP routing

**Environment Variables**:
```yaml
COREIP=172.16.254.2          # Core network IP
CORESUBNET=172.16.254.1/24   # Core network subnet
PUBLICIP=192.168.254.2       # Public network IP
UDPPORT=5060                 # SIP UDP port
TLSPORT=5061                 # SIP TLS port
DBUSERNAME=kamailio          # Database username
DBPASSWORD=kamailiorw        # Database password
DBHOST=172.16.254.10         # Database host
DBNAME=kamailio              # Database name
```

**Key Files**:
- `/etc/kamailio/kamailio.cfg` - Main configuration entry point
- `/etc/kamailio/globals.cfg` - Global parameters (IPs, ports, DB connection)
- `/etc/kamailio/modules.cfg` - Module loading order
- `/etc/kamailio/routes.cfg` - Routing logic includes
- `/etc/kamailio/routes/*.cfg` - Modular route handlers

---

### 2. RTPEngine Edge (`rtpengine-edge`)

**Purpose**: Media proxy that handles all audio/video streams

**What RTPEngine Does**:
1. **NAT Traversal**: Bridges media between endpoints behind NAT
2. **Media Transcoding**: Converts between different codecs (G.711, Opus, etc.)
3. **SRTP/RTP Conversion**: Encrypts/decrypts media streams
4. **Media Recording**: Can record calls for compliance/quality
5. **DTLS-SRTP**: Handles WebRTC-style encrypted media

**Network Interfaces**:
- Public: 192.168.254.3 (media from/to internet)
- Core: 172.16.254.3 (media from/to internal servers)

**Media Port Range**: UDP 23000-23100 (101 simultaneous calls supported)

**How Kamailio Integrates**:
Kamailio sends commands to RTPEngine via NG protocol (network gateway) to:
- Allocate media ports
- Set up media forwarding rules
- Enable/disable encryption
- Control codec transcoding

---

### 3. MySQL Database (`db`)

**Image**: `mysql:5.7`

**Purpose**: Persistent storage for Kamailio

**Databases/Tables Created**:
1. **subscriber**: User credentials for authentication
2. **location**: Current registration locations (AOR bindings)
3. **acc**: Transaction-level call accounting
4. **acc_cdrs**: Dialog-level call detail records
5. **dispatcher**: List of backend servers for load balancing
6. **secfilter**: Blacklist entries for security filtering
7. **rtpengine**: RTPEngine instance configuration

**Credentials**:
- Root password: `rw_password`
- User: `user` / Password: `ro_password`
- Kamailio user: `kamailio` / Password: `kamailiorw`

**Initialization**:
The `initial_setup.sh` script:
1. Creates the database schema using `kamdbctl`
2. Adds test users
3. Populates dispatcher table with backend B2BUA addresses
4. Configures RTPEngine

---

### 4. BIND9 DNS Server (`bind`)

**Image**: `internetsystemsconsortium/bind9:9.20`

**Purpose**: ENUM (E.164 Number Mapping) server for outbound call routing

**What is ENUM?**
ENUM translates phone numbers into SIP URIs using DNS NAPTR records.

**Example**:
- Phone number: `+31 20 123 4567`
- ENUM query: `7.6.5.4.3.2.1.0.2.1.3.e164.arpa`
- DNS returns: `sip:31201234567@192.168.254.101`

**Configuration Files**:
- `/etc/bind/named.conf` - Main BIND configuration
- `/etc/bind/db.e164.arpa` - ENUM zone file with NAPTR records

**Sample NAPTR Record**:
```
7.6.5.4.3.2.1.0.2.1.3.e164.arpa.  IN NAPTR 5 10 "u" "E2U+SIP" "!^.*$!sip:31201234567@192.168.254.200!" .
```

**Kamailio Integration**:
- Kamailio queries this DNS server when routing outbound calls
- If no registration found, ENUM lookup provides external routing
- Supports multiple NAPTR records for failover

---

### 5. REST API (`api`)

**Technology**: Python Flask

**Purpose**: Dynamic routing decisions via HTTP API

**Endpoint**: `POST http://172.16.254.30:5000/api/routing`

**How It Works**:
1. Kamailio suspends the SIP transaction
2. Makes async HTTP POST to API with call details
3. API returns JSON routing instructions (RTJSON format)
4. Kamailio resumes transaction and routes according to API response

**Request Example**:
```json
{
  "to_user": "b2bua_internal_01"
}
```

**Response Example**:
```json
{
  "rtjson": {
    "version": "1.0",
    "routing": "serial",
    "routes": [
      {
        "uri": "sip:b2bua_internal@172.16.254.101:5060",
        "dst_uri": "sip:172.16.254.101:5060",
        "socket": "udp:172.16.254.2:5060",
        "headers": {
          "from": {
            "display": "Alice",
            "uri": "sip:alice@a.example.org"
          },
          "to": {
            "display": "Bob",
            "uri": "sip:bob@b.example.org"
          }
        },
        "fr_timer": 5000,
        "fr_inv_timer": 30000
      }
    ]
  }
}
```

**Use Cases**:
- Customer-specific routing logic
- Time-based routing
- Least-cost routing
- Percentage-based traffic distribution
- A/B testing of routes

**Module**: This feature is controlled by the `RTJSON_INBOUND` define in `kamailio.cfg`

---

### 6. B2BUA Services (Back-to-Back User Agents)

**Image**: `andrius/pjsua:latest` (PJSIP-based SIP client)

**Purpose**: Simulate SIP endpoints and backend servers

#### Internal B2BUAs (Core Network)
1. **b2bua_internal_01** (172.16.254.100)
   - Registered to Kamailio
   - Can receive calls from external sources
   - Part of dispatcher group for load balancing

2. **b2bua_internal_02** (172.16.254.101)
   - Auto-answers with 200 OK
   - No registration required
   - Part of dispatcher group

#### External B2BUAs (Public Network)
1. **b2bua_external_01** (192.168.254.100)
   - Can register to Kamailio from "outside"
   - Tests inbound call flows

2. **b2bua_external_02** (192.168.254.101)
   - Secondary external endpoint
   - Used for ENUM routing tests

**Configuration**: PJSUA config files in `./b2bua/*/pjsua.cfg`

---

### 7. Sngrep (`sngrep`)

**Purpose**: Network packet capture and SIP message analysis

**What is Sngrep?**
Sngrep is a powerful terminal-based tool that displays SIP call flows in real-time, similar to Wireshark but specialized for SIP.

**Features**:
- Live SIP message capture
- Call flow diagrams
- Filter by Call-ID, IP, method, etc.
- Save/export captures

**Usage**: `docker exec -it sngrep sngrep`

**Network Mode**: `host` - Can see all network traffic on the host machine

---

## Kamailio Configuration Deep Dive

### Configuration Architecture

Kamailio uses a **modular configuration approach**:

```
kamailio.cfg (entry point)
├── modules-core.cfg (core modules: tm, sl, rr, etc.)
├── globals.cfg (IPs, ports, DB credentials)
├── modules.cfg (load all feature modules)
└── routes.cfg (includes all route files)
    ├── request_main.cfg (main routing logic)
    ├── reqinit.cfg (request initialization)
    ├── withindialog.cfg (in-dialog request handling)
    ├── relay.cfg (final message relay)
    ├── set_direction_flag.cfg (public vs core detection)
    ├── set_socket.cfg (choose correct network interface)
    ├── set_rtp_request.cfg (RTPEngine engagement for requests)
    ├── set_rtp_reply.cfg (RTPEngine engagement for replies)
    ├── handle_register.cfg (user registration)
    ├── remove_codecs_inbound.cfg (SDP manipulation)
    ├── blacklist.cfg (security filtering)
    ├── set_cdr_vars.cfg (CDR variable setup)
    ├── event_routes.cfg (event handlers)
    ├── manage_branch.cfg (per-branch processing)
    ├── manage_reply.cfg (response handling)
    ├── manage_failure.cfg (failure handling)
    ├── failed_dispatch.cfg (dispatcher failover)
    ├── failed_serial.cfg (ENUM failover)
    ├── handle_http_request.cfg (RTJSON API call)
    └── handle_http_response.cfg (RTJSON API response)
```

### Main Request Flow (request_main.cfg)

**Step-by-step processing of INVITE**:

1. **Pike Check** (`pike_check_req()`)
   - Detects flooding attacks
   - Blocks IPs exceeding rate limits
   - Silent drop if threshold exceeded

2. **Blacklist Check** (`route(BLACKLIST)`)
   - Checks htable blacklist
   - Queries secfilter database
   - Rejects if IP/User-Agent is blacklisted

3. **Request Initialization** (`route(REQINIT)`)
   - Max-Forwards header validation
   - OPTIONS ping response
   - Sanity checks (malformed messages)

4. **Direction Detection** (`route(SET_DIRECTION_FLAG)`)
   - Determines if request is from public or core network
   - Sets `FLT_FROM_PUBLIC` or `FLT_FROM_CORE` flag
   - Critical for security - prevents core bypass

5. **Socket Selection** (`route(SET_SOCKET)`)
   - Chooses correct network interface for outbound
   - Public requests exit via core interface
   - Core requests exit via public interface

6. **CANCEL Handling**
   - Matches transaction
   - Routes to appropriate endpoint

7. **Retransmission Handling**
   - Uses transaction module to detect retransmissions
   - Prevents duplicate processing

8. **Registration Handling** (`route(HANDLE_REGISTER)`)
   - Authenticates user
   - Saves location binding
   - Sends 200 OK

9. **Codec Filtering** (`route(REMOVE_CODECS_INBOUND)`)
   - Parses SDP
   - Removes all codecs except G.711A and G.711U
   - Business requirement for standardization

10. **RTP Setup** (`route(SET_RTP_REQUEST)`)
    - Engages RTPEngine
    - Allocates media ports
    - Sets up encryption if TLS is used

11. **Dialog Handling** (`route(WITHINDLG)`)
    - Processes in-dialog requests (BYE, re-INVITE, etc.)
    - Uses Record-Route headers
    - Bypasses routing logic

12. **Record-Route**
    - Adds Record-Route header for dialog-forming requests
    - Ensures all in-dialog messages traverse this proxy

13. **Call Accounting** (`route(SET_CDR_VARS)`)
    - Sets AVPs for CDR generation
    - Flags transaction for accounting

14. **Routing Decision** (INVITE only)
    - **From Public**:
      - If `RTJSON_INBOUND` defined: API-based routing
      - Otherwise: Dispatcher to core B2BUAs
    - **From Core**:
      - Check user registration
      - If not registered: ENUM lookup
      - If ENUM found: Load contacts and set failover

15. **Final Relay** (`route(RELAY)`)
    - Sends message to next hop
    - Stateful forwarding with transaction

### Key Kamailio Modules Used

#### Core Modules

1. **tm (Transaction Module)**
   - Stateful SIP processing
   - Retransmission handling
   - Failure route invocation

2. **sl (Stateless Module)**
   - Quick responses (100 Trying, etc.)
   - Low-overhead replies

3. **rr (Record Route)**
   - Adds Record-Route headers
   - Ensures in-dialog routing

#### Database Modules

4. **db_mysql**
   - MySQL connectivity
   - All DB operations

5. **usrloc (User Location)**
   - In-memory registration storage
   - Location bindings

6. **registrar**
   - REGISTER method handling
   - Authentication integration

7. **auth_db**
   - Database-backed authentication
   - Digest authentication

#### Routing Modules

8. **dispatcher**
   - Load balancing across multiple destinations
   - Round-robin, hash-based, failover algorithms
   - Active/standby monitoring

9. **enum**
   - ENUM lookups via DNS
   - NAPTR record parsing
   - Failover across multiple records

10. **rtjson**
    - JSON-based routing
    - REST API integration
    - Dynamic routing decisions

#### Security Modules

11. **pike**
    - Flood detection
    - Rate limiting per source IP
    - Automatic blocking

12. **secfilter**
    - Database-driven blacklisting
    - User-Agent filtering
    - Country-based blocking

13. **htable**
    - In-memory hash tables
    - Runtime blacklist management
    - Shared data across processes

#### Media Modules

14. **rtpengine**
    - RTPEngine control
    - Media proxy commands
    - SRTP/RTP conversion

15. **sdpops**
    - SDP parsing and manipulation
    - Codec filtering
    - Media attribute modification

#### Topology Hiding

16. **topoh**
    - Hides topology in headers
    - Obfuscates Via, Record-Route
    - Prevents information disclosure

17. **topos**
    - Dialog-level topology hiding
    - Contact header masking

#### Accounting/CDR

18. **acc (Accounting)**
    - Transaction-level accounting
    - Database CDR storage
    - Flexible attribute selection

19. **dialog**
    - Dialog state tracking
    - Duration calculation
    - Dialog-level CDRs

#### Advanced Routing

20. **uac (User Agent Client)**
    - From/To header manipulation
    - Outbound authentication
    - Dialog parameter restoration

21. **http_async_client**
    - Asynchronous HTTP requests
    - Transaction suspension/resumption
    - REST API integration

22. **jansson**
    - JSON parsing
    - JSON object manipulation
    - RTJSON support

#### TLS/Encryption

23. **tls**
    - TLS transport
    - Certificate validation
    - Encrypted signaling

---

## Business Requirements

The course teaches how to implement **real-world SIP proxy requirements**:

### General Requirements

1. **Network Segmentation**
   - Differentiate public vs core traffic
   - Prevent direct core access from internet
   - Topology hiding (no private IPs exposed)

2. **OPTIONS Handling**
   - Respond to SIP OPTIONS keepalives
   - 200 OK for permitted IPs
   - Used for service monitoring

3. **Call Detail Records (CDRs)**
   - Start time, end time, duration
   - Source number, destination number
   - Remote IP address, Call-ID
   - Duration counting starts at call connect (200 OK)

4. **Protocol Support**
   - UDP on port 5060
   - TLS on port 5061
   - **TCP on 5060 is disabled** (security requirement)

5. **Media Encryption**
   - SRTP support
   - DTLS-SRTP for WebRTC
   - SDES key exchange

### Inbound Requirements

1. **Security Filtering**
   - Block malicious IPs automatically (pike)
   - Blacklist management (secfilter, htable)
   - Rate limiting

2. **User Registration**
   - Accept REGISTER from external devices
   - Digest authentication
   - Location storage

3. **Codec Standardization**
   - Remove all codecs except G.711A and G.711U
   - SDP manipulation before reaching core
   - Ensures compatibility and reduces transcoding load

4. **Load Balancing**
   - Distribute calls to multiple core servers
   - Round-robin algorithm
   - Automatic failover

### Outbound Requirements

1. **ENUM Routing**
   - DNS-based routing for outbound calls
   - Multiple NAPTR record support
   - Failover to next record on failure

2. **Automatic Failover**
   - Try next ENUM record within acceptable timeframe
   - Timeout: 5 seconds per attempt
   - `t_on_failure("FAILED_SERIAL")` route

3. **SRTP Offering**
   - If TLS is used for signaling, offer SRTP for media
   - Automatic encryption upgrade

---

## Course Structure

The repository has multiple branches representing course progression:

### Branches

1. **course/01-basic-config**
   - Initial Kamailio setup
   - Minimal proxy configuration
   - Docker environment setup

2. **course/02-first-refactor**
   - Separate concerns into modules
   - Routing logic organization
   - Configuration file structure

3. **course/03-general-reqs**
   - Implement general business requirements
   - Direction detection
   - Socket management
   - OPTIONS handling

4. **course/04-rtpengine**
   - RTPEngine integration
   - Media proxy setup
   - NAT traversal

5. **course/05-cdrs**
   - Call Detail Record generation
   - Database accounting
   - Dialog tracking

6. **course/06-tls**
   - TLS transport setup
   - Certificate configuration
   - Encrypted signaling

7. **course/07-topo**
   - Topology hiding (topoh/topos)
   - Header obfuscation
   - Privacy protection

8. **course/08-registrar**
   - SIP registrar implementation
   - User location service
   - Authentication

9. **course/09-removing-codecs**
   - SDP manipulation
   - Codec filtering
   - G.711 enforcement

10. **course/10-secfilter**
    - Security filtering
    - Pike flood detection
    - Blacklist management

11. **course/11-dispatcher**
    - Load balancing
    - Multiple backend servers
    - Failover logic

12. **course/12-srtp**
    - SRTP configuration
    - RTP ↔ SRTP conversion
    - Media encryption

13. **course/13-enum**
    - ENUM setup
    - DNS integration
    - Outbound routing

### Main Branch

The **main** branch contains the **complete, final configuration** with all features enabled (except RTJSON which is commented out).

To use main branch directly:
```bash
git clone https://github.com/keithcroxford/kamailio-course.git
cd kamailio-course
chmod +x initial_setup.sh
./initial_setup.sh
```

---

## Key Technologies

### SIP (Session Initiation Protocol)

**What**: Signaling protocol for establishing, modifying, and terminating multimedia sessions

**Key Concepts**:
- **User Agent**: SIP endpoint (phone, softphone, gateway)
- **Proxy**: Routes SIP messages
- **Registrar**: Accepts REGISTER requests
- **B2BUA**: Back-to-Back User Agent (terminates and re-originates calls)

**SIP Methods**:
- `INVITE`: Establish session
- `ACK`: Confirm INVITE
- `BYE`: Terminate session
- `CANCEL`: Cancel pending INVITE
- `REGISTER`: Register location
- `OPTIONS`: Query capabilities
- `UPDATE`: Modify session parameters

**SIP Responses**:
- `1xx`: Provisional (100 Trying, 180 Ringing)
- `2xx`: Success (200 OK)
- `3xx`: Redirection (302 Moved Temporarily)
- `4xx`: Client Error (404 Not Found, 407 Proxy Auth Required)
- `5xx`: Server Error (500 Internal Server Error)
- `6xx`: Global Failure (603 Decline)

### RTP/SRTP (Real-time Transport Protocol)

**RTP**: Carries audio/video data
**RTCP**: Control protocol for RTP (statistics, feedback)
**SRTP**: Secure RTP (encrypted media)

**Key Concepts**:
- **Codec**: Audio/video encoding (G.711, Opus, H.264)
- **DTMF**: Dual-tone multi-frequency (phone keypad tones)
- **Jitter Buffer**: Compensates for network delay variation

### SDP (Session Description Protocol)

**What**: Describes media sessions (codecs, IP addresses, ports)

**Example SDP**:
```
v=0
o=- 3634023641 3634023641 IN IP4 192.168.1.100
s=-
c=IN IP4 192.168.1.100
t=0 0
m=audio 10000 RTP/AVP 0 8
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=sendrecv
```

### ENUM (E.164 Number Mapping)

**What**: Maps phone numbers to SIP URIs using DNS

**Process**:
1. Phone number: +1-202-555-0100
2. Reverse and append domain: 0.0.1.0.5.5.5.2.0.2.1.e164.arpa
3. DNS NAPTR query
4. Returns SIP URI: sip:12025550100@sip.example.com

### RTJSON (Routing JSON)

**What**: Kamailio-specific JSON format for dynamic routing

**Advantages**:
- External routing logic (not hardcoded)
- Easy integration with business logic
- Real-time routing updates
- A/B testing, percentage routing

### Docker & Docker Compose

**Why Docker?**
- Reproducible environment
- Isolated network simulation
- Easy setup/teardown
- Multi-container orchestration

**This Project's Docker Features**:
- Custom networks (external, internal)
- Static IP assignment
- Service dependencies
- Volume mounts for configuration

---

## Setup and Installation

### Prerequisites

- Docker and Docker Compose installed
- Linux, macOS, or Windows with WSL2
- At least 4GB RAM available for Docker
- Ports available: 5060, 5061, 5000, 3306, 23000-23100

### Installation Steps

#### Option 1: Start from Main Branch (Complete Configuration)

```bash
# Clone repository
git clone https://github.com/keithcroxford/kamailio-course.git
cd kamailio-course

# Make setup script executable
chmod +x initial_setup.sh

# Run initial setup (WARNING: Deletes existing DB)
./initial_setup.sh
```

**What initial_setup.sh Does**:
1. Stops any running containers
2. Deletes existing database volume
3. Downloads minimal Kamailio config
4. Starts all Docker services
5. Creates database schema with `kamdbctl`
6. Populates RTPEngine table
7. Creates test users:
   - `b2bua_external@192.168.254.100` (password: `password1`)
   - `b2bua_internal_01@172.16.254.100` (password: `password1`)
8. Adds dispatcher entries for load balancing
9. Downloads final kamailio.cfg from GitHub
10. Restarts Kamailio

#### Option 2: Start from Beginning (Course Branch 01)

```bash
git clone https://github.com/keithcroxford/kamailio-course.git
cd kamailio-course
git checkout course/01-basic-config
docker compose up -d
```

### Verification

```bash
# Check all services are running
docker compose ps

# Should show 9 services running:
# - kamailio-edge
# - rtpengine-edge
# - db
# - bind
# - api
# - b2bua_internal_01
# - b2bua_internal_02
# - b2bua_external_01
# - b2bua_external_02
# - sngrep

# Check Kamailio logs
docker logs kamailio-edge

# Access Kamailio CLI
docker exec -it kamailio-edge kamcmd

# Test database connection
docker exec kamailio-edge kamctl db show
```

### Testing the Setup

#### Test 1: SIP OPTIONS Ping

```bash
# From your machine
docker exec -it b2bua_external_01 pjsua \
  --null-audio \
  --no-tcp \
  --max-calls=1 \
  "sip:192.168.254.2;transport=udp" \
  --duration=1
```

Expected: 200 OK response

#### Test 2: User Registration

```bash
docker exec -it b2bua_external_01 bash
pjsua --config-file=/config/pjsua.cfg
```

Expected: Registration successful

#### Test 3: Inbound Call (External → Core)

From `b2bua_external_01`, dial the number configured in ENUM.

Expected: Call routes through Kamailio to internal B2BUA

#### Test 4: View Live SIP Traffic

```bash
docker exec -it sngrep sngrep
```

Expected: See SIP messages in real-time with call flows

---

## How It All Works Together

### Scenario 1: Inbound Call from Internet

**Goal**: External SIP phone calls a registered user

**Call Flow**:

1. **External Phone** (192.168.254.100) sends INVITE to Kamailio public IP
   ```
   INVITE sip:b2bua_internal_01@192.168.254.2 SIP/2.0
   ```

2. **Kamailio** receives on public interface (192.168.254.2:5060)
   - Pike checks flood detection: ✓ Pass
   - Blacklist check: ✓ Pass
   - Direction flag: `FLT_FROM_PUBLIC` set
   - Socket selection: Will reply from public, forward to core

3. **Codec Filtering**
   - Parses SDP
   - Removes codecs other than G.711A/U
   - Rebuilds SDP

4. **RTPEngine Engagement**
   - Allocates public media port (e.g., 23050)
   - Allocates core media port (e.g., 23051)
   - Rewrites SDP with RTPEngine IPs

5. **Routing Decision** (Public → Core)
   - Checks if `RTJSON_INBOUND` is defined
   - If yes: Calls REST API for routing
   - If no: Uses dispatcher to load-balance to core B2BUAs
   - Selects destination: 172.16.254.100 (b2bua_internal_01)

6. **Dispatcher Selection**
   ```
   ds_select_dst(1, 4);  # Group 1, Round-robin
   ```
   - Chooses one of: 172.16.254.100 or 172.16.254.101
   - Sets `$du` (destination URI)

7. **Topology Hiding**
   - Hides Via headers with topoh
   - Masks Contact with topos
   - External endpoints see only Kamailio IPs

8. **Forward to Core**
   - Kamailio sends INVITE to 172.16.254.100:5060
   - Uses core interface (172.16.254.2)
   - RTPEngine bridges media

9. **B2BUA Responds**
   - Sends 180 Ringing
   - Kamailio forwards to external phone
   - Sends 200 OK
   - Kamailio forwards to external phone

10. **ACK and Media**
    - External phone sends ACK
    - Media flows:
      ```
      External Phone ↔ RTPEngine (public) ↔ RTPEngine (core) ↔ Internal B2BUA
      ```

11. **CDR Generation**
    - Dialog starts: Record connect time
    - On BYE: Calculate duration
    - Store in `acc_cdrs` table

### Scenario 2: Outbound Call with ENUM

**Goal**: Internal B2BUA calls external number via ENUM

**Call Flow**:

1. **Internal B2BUA** (172.16.254.100) sends INVITE to Kamailio
   ```
   INVITE sip:31201234567@172.16.254.2 SIP/2.0
   ```

2. **Kamailio** receives on core interface
   - Direction flag: `FLT_FROM_CORE` set
   - Checks if source is in dispatcher list: ✓ Yes

3. **Registration Lookup**
   ```
   lookup("location")
   ```
   - Checks if 31201234567 is registered: ✗ No

4. **ENUM Lookup**
   ```
   enum_query()
   ```
   - Reverses number: 7.6.5.4.3.2.1.0.2.1.3.e164.arpa
   - DNS query to BIND (172.16.254.20)
   - Returns NAPTR records:
     ```
     Priority 5: sip:31201234567@192.168.254.200
     Priority 100: sip:31201234567@192.168.254.101
     ```

5. **Load Contacts**
   ```
   t_load_contacts()
   ```
   - Stores all ENUM results in branches

6. **Select First Contact**
   ```
   t_next_contacts()
   ```
   - Tries priority 5 first
   - Sets timeout: 5 seconds

7. **RTPEngine Setup**
   - Allocates media ports
   - Rewrites SDP

8. **Forward to External Destination**
   - Sends INVITE to 192.168.254.200
   - Uses public interface

9. **Failure Handling** (if 192.168.254.200 fails)
   ```
   failure_route[FAILED_SERIAL] {
     if (t_next_contacts()) {
       route(RELAY);  # Try next NAPTR record
     }
   }
   ```
   - Automatically tries 192.168.254.101

10. **Success**
    - Call connects
    - Media flows
    - CDR generated

### Scenario 3: Dynamic Routing with RTJSON

**Goal**: Use REST API to determine call routing

**Prerequisites**:
- Uncomment `#!define RTJSON_INBOUND` in kamailio.cfg
- Restart Kamailio

**Call Flow**:

1. **External Phone** sends INVITE
2. **Kamailio** detects `FLT_FROM_PUBLIC`
3. **RTJSON Route Activated**
   ```
   route(HANDLE_HTTP_REQUEST)
   ```

4. **Suspend Transaction**
   ```
   t_suspend()
   ```
   - SIP transaction is suspended
   - Kamailio continues processing other requests

5. **HTTP POST to API**
   ```
   http_async_query("http://172.16.254.30:5000/api/routing", ...)
   ```
   - Sends JSON body: `{"to_user": "b2bua_internal_01"}`

6. **API Processes Request**
   - Flask app receives POST
   - Routing logic executes (could be complex business rules)
   - Returns RTJSON structure

7. **Resume Transaction**
   ```
   route[HANDLE_HTTP_RESPONSE]
   ```
   - Parses JSON response
   - RTJSON module applies routing

8. **RTJSON Routing**
   ```
   rtjson_init_routes($var(rtjson))
   rtjson_push_routes()
   rtjson_next_route()
   ```
   - Sets destination URI
   - Modifies From/To headers (if specified)
   - Applies branch flags, timers

9. **Call Proceeds**
   - Forwarded to destination from RTJSON
   - Normal call flow continues

**Advantages**:
- Routing logic in familiar language (Python, Node.js, etc.)
- Real-time updates without Kamailio restart
- Integration with databases, APIs, ML models
- Complex business rules

---

## Advanced Features

### 1. Topology Hiding (TOPOH + TOPOS)

**Problem**: Default SIP headers expose internal network topology

**Example** (Without Topology Hiding):
```
Via: SIP/2.0/UDP 172.16.254.100:5060;branch=z9hG4bK...
Record-Route: <sip:172.16.254.2:5060;lr>
Contact: <sip:1234@172.16.254.100:5060>
```

External parties can see:
- Internal IP: 172.16.254.100
- Network structure
- Number of hops

**Solution**: TOPOH (header topology hiding) + TOPOS (dialog topology hiding)

**Result** (With Topology Hiding):
```
Via: SIP/2.0/UDP 192.168.254.2:5060;branch=z9hG4bKa1b2c3d4
Record-Route: <sip:192.168.254.2:5060;lr;ftag=abc123>
Contact: <sip:1234@192.168.254.2:5060>
```

External parties see only:
- Public IP: 192.168.254.2
- No internal topology
- Kamailio appears as endpoint

**Configuration**:
```
loadmodule "topoh.so"
loadmodule "topos.so"

modparam("topoh", "mask_key", "random_secret_key")
```

### 2. SRTP/DTLS Media Encryption

**SRTP (Secure RTP)**: Encrypted media streams

**Two Key Exchange Methods**:

#### SDES (Session Description Protocol Security Descriptions)
- Keys exchanged in SDP
- Simpler implementation
- Requires secure signaling (TLS)

**SDP Example**:
```
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:PS1uQCVeeCFCanVmcjkpPywjNWhcYD0mXXtxaVBR
```

#### DTLS-SRTP (Datagram Transport Layer Security)
- Keys negotiated via DTLS handshake
- WebRTC standard
- Fingerprints in SDP

**SDP Example**:
```
a=fingerprint:sha-256 49:27:AF:1D:BA:94:2B:00:E9:33:0A:9E:3A:B5:93:6D:EF:11:E4:55
a=setup:actpass
```

**RTPEngine Configuration**:
Kamailio tells RTPEngine to encrypt/decrypt:
```
rtpengine_offer("RTP/SAVP replace-session-connection ICE=remove");
rtpengine_answer("RTP/SAVP replace-session-connection ICE=remove");
```

**Use Case**:
- External call uses SRTP
- Internal B2BUA uses plain RTP
- RTPEngine bridges: SRTP ↔ RTP

### 3. Pike Flood Detection

**Problem**: SIP flooding attacks (register floods, INVITE floods)

**Solution**: Pike module monitors request rate per IP

**Configuration**:
```
modparam("pike", "sampling_time_unit", 2)     # Sample every 2 seconds
modparam("pike", "reqs_density_per_unit", 30) # Max 30 requests per 2 sec
modparam("pike", "remove_latency", 120)       # Unblock after 120 sec
```

**Logic**:
```
if (!pike_check_req()) {
  xlog("L_ALERT", "PIKE BLOCKED: $si\n");
  exit;  # Silent drop
}
```

**Result**: Automatic IP blocking when threshold exceeded

### 4. Security Filtering (Secfilter + Htable)

**Secfilter**: Database-driven blacklist

**Tables**:
- Blacklist by IP
- Blacklist by User-Agent
- Blacklist by country (GeoIP)

**Htable**: In-memory hash table for runtime blacklist

**Configuration**:
```
modparam("htable", "htable", "blacklist=>size=8;")
```

**Runtime Blacklist Add**:
```
kamcmd htable.seti blacklist 192.168.1.100 1
```

**Check**:
```
route[BLACKLIST] {
  if ($sht(blacklist=>$si) != $null) {
    sl_send_reply("403", "Forbidden");
    exit;
  }
  secf_check_ip();
  secf_check_ua();
}
```

### 5. Dispatcher Load Balancing

**Purpose**: Distribute calls across multiple backend servers

**Database Table**: `dispatcher`

**Algorithms**:
- `0`: Hash over Call-ID
- `4`: Round-robin
- `8`: Weight-based
- `10`: Active calls (least loaded)

**Configuration**:
```
dispatcher table:
+----+-------+---------------------------+-------+
| id | setid | destination               | flags |
+----+-------+---------------------------+-------+
| 1  | 1     | sip:172.16.254.100:5060   | 0     |
| 2  | 1     | sip:172.16.254.101:5060   | 0     |
+----+-------+---------------------------+-------+
```

**Usage**:
```
ds_select_dst(1, 4);  # Set 1, Round-robin
t_on_failure("FAILED_DISPATCH");
route(RELAY);

failure_route[FAILED_DISPATCH] {
  if (ds_next_dst()) {
    route(RELAY);  # Try next server
  }
}
```

**Active Probing**:
```
modparam("dispatcher", "ds_probing_mode", 1)
modparam("dispatcher", "ds_ping_interval", 30)
```
- Sends OPTIONS every 30 seconds
- Marks destination as inactive if no response
- Automatically routes around failed servers

### 6. Dialog-Based CDRs

**Problem**: Transaction accounting only captures attempts, not successful calls

**Solution**: Dialog module tracks call state

**CDR Fields**:
- `start_time`: When 200 OK is received
- `duration`: Time from 200 OK to BYE
- `sip_code`: Final response code
- `sip_reason`: Response reason phrase

**Configuration**:
```
modparam("dialog", "db_url", DBURL)
modparam("dialog", "db_mode", 1)  # Real-time DB writes

modparam("acc", "cdr_enable", 1)
modparam("acc", "cdr_start_on_confirmed", 1)
```

**Usage**:
```
if (is_method("INVITE")) {
  setflag(FLT_DLG);  # Enable dialog tracking
  setflag(FLT_ACC);  # Enable accounting
}
```

**Database**: `acc_cdrs` table with complete call records

### 7. Custom Headers and Variables

**Problem**: Need to pass custom data through the call

**Solution**: AVPs, XAVPs, and custom headers

**AVP (Attribute-Value Pair)**:
```
$avp(caller_id) = $fU;
$avp(original_ruri) = $ru;
```

**XAVP (Extended AVP)**:
```
$xavp(caller=>name) = $fU;
$xavp(caller=>ip) = $si;
```

**Custom SIP Header**:
```
append_hf("X-Customer-ID: 12345\r\n");
append_hf("X-Original-Caller: $fU\r\n");
```

**Usage in CDRs**:
```
modparam("acc", "cdr_extra", "caller_id=$avp(caller_id)")
```

---

## Troubleshooting

### Common Issues

#### 1. Containers Won't Start

**Symptom**: `docker compose up -d` fails

**Checks**:
```bash
# Check port conflicts
sudo netstat -tulpn | grep -E ':(5060|5061|3306|5000|53)'

# Check Docker network conflicts
docker network ls
docker network inspect kamailio-course-experimentation_external

# Check logs
docker compose logs
```

**Solution**:
- Kill processes using conflicting ports
- Remove conflicting Docker networks
- Check firewall rules

#### 2. Kamailio Won't Start

**Symptom**: `kamailio-edge` container exits immediately

**Checks**:
```bash
docker logs kamailio-edge

# Common errors:
# - Syntax error in .cfg file
# - Module dependency missing
# - Database connection failure
```

**Solution**:
```bash
# Test config syntax
docker exec kamailio-edge kamailio -c

# Check module loading
docker exec kamailio-edge kamailio -M

# Test database connection
docker exec kamailio-edge kamctl db show
```

#### 3. No Audio in Calls

**Symptom**: Call connects but no audio

**Checks**:
1. **RTPEngine Running?**
   ```bash
   docker logs rtpengine-edge
   ```

2. **RTPEngine Configured in Kamailio?**
   ```bash
   docker exec kamailio-edge kamcmd rtpengine.show all
   ```

3. **SDP Inspection**
   ```bash
   docker exec -it sngrep sngrep
   # Look at SDP in INVITE and 200 OK
   ```

4. **Firewall Rules**
   - UDP ports 23000-23100 must be open
   - Check Docker port mapping

**Solution**:
```bash
# Restart RTPEngine
docker restart rtpengine-edge

# Check RTPEngine registration in DB
docker exec kamailio-edge kamctl db exec "SELECT * FROM rtpengine"

# Re-add if missing
docker exec kamailio-edge sh -c "query=\$(cat /etc/kamailio/sql/rtpengine.sql) ; kamctl db exec \"\$query\""
```

#### 4. ENUM Not Working

**Symptom**: Outbound calls fail with 404

**Checks**:
```bash
# Test DNS resolution
docker exec kamailio-edge dig @172.16.254.20 NAPTR 7.6.5.4.3.2.1.0.2.1.3.e164.arpa

# Check BIND logs
docker logs bind

# Verify Kamailio can reach DNS
docker exec kamailio-edge ping -c 3 172.16.254.20
```

**Solution**:
- Fix BIND configuration
- Check DNS server IP in Kamailio (globals.cfg)
- Verify NAPTR records format

#### 5. Registration Fails

**Symptom**: 401 Unauthorized loop

**Checks**:
```bash
# Check if user exists
docker exec kamailio-edge kamctl db exec "SELECT * FROM subscriber WHERE username='b2bua_external'"

# Check authentication module
docker logs kamailio-edge | grep -i auth
```

**Solution**:
```bash
# Re-add user
docker exec kamailio-edge kamctl add b2bua_external@192.168.254.100 password1

# Check auth module parameters
docker exec kamailio-edge kamcmd cfg.list auth_db
```

---

## Performance and Scaling

### Resource Requirements

**Minimal Setup** (Development):
- CPU: 2 cores
- RAM: 4GB
- Disk: 10GB

**Production Recommendations**:
- CPU: 8+ cores (Kamailio is multi-process)
- RAM: 16GB+ (shared memory for location, dialog tables)
- Disk: SSD recommended (database I/O)
- Network: Low latency, high bandwidth

### Kamailio Performance Tuning

**Shared Memory**:
```
modparam("tm", "shm_size", 512)  # 512MB for transactions
```

**Child Processes**:
```
children=8  # Increase for higher CPS (calls per second)
```

**Database Connections**:
```
modparam("db_mysql", "max_connections", 10)
```

**UDP Buffer**:
```
# In globals.cfg
udp4_raw_mtu=1500
udp4_raw_ttl=64
```

### Scaling Architecture

**Horizontal Scaling**:

1. **Multiple Kamailio Instances**
   - Load balancer in front (DNS RR, HAProxy)
   - Shared database
   - Stateless where possible

2. **Database Replication**
   - Master-slave MySQL replication
   - Read replicas for lookups
   - Write to master only

3. **Distributed RTPEngine**
   - RTPEngine per region
   - Reduce media latency
   - Use `rtpengine_manage()` with set selection

**Geographic Distribution**:
```
US East    US West    Europe
   │          │          │
Kamailio   Kamailio  Kamailio
   │          │          │
   └──────────┴──────────┘
              │
         Shared DB
```

---

## Security Best Practices

1. **Network Segmentation** ✓
   - Public/core separation
   - No direct core access

2. **Rate Limiting** ✓
   - Pike module active
   - Per-IP request limits

3. **Authentication**
   - Strong passwords
   - Digest authentication
   - Consider SIP identity (RFC 8224)

4. **TLS/SRTP**
   - Enforce TLS for sensitive traffic
   - Use SRTP for media
   - Valid certificates (Let's Encrypt)

5. **Firewall**
   - Restrict source IPs where possible
   - Close unused ports
   - Consider fail2ban integration

6. **Monitoring**
   - Log all authentication failures
   - Alert on high call volume
   - Monitor for scanning activity

7. **Regular Updates**
   - Keep Kamailio updated
   - Patch OS and dependencies
   - Security mailing lists

8. **Least Privilege**
   - Run Kamailio as non-root
   - Database permissions (read-only where possible)
   - Container security

---

## Useful Commands

### Kamailio CLI (kamcmd)

```bash
# Access kamcmd
docker exec -it kamailio-edge kamcmd

# Show statistics
kamcmd stats.get_statistics all

# Show active dialogs
kamcmd dlg.list

# Show registrations
kamcmd ul.dump

# Reload dispatcher
kamcmd dispatcher.reload

# Show RTPEngine instances
kamcmd rtpengine.show all

# Kill specific dialog
kamcmd dlg.end_dlg <call-id>
```

### Database Operations (kamctl)

```bash
# Show database
docker exec kamailio-edge kamctl db show

# Execute query
docker exec kamailio-edge kamctl db exec "SELECT * FROM acc_cdrs LIMIT 10"

# Add user
docker exec kamailio-edge kamctl add username@domain password

# Remove user
docker exec kamailio-edge kamctl rm username@domain

# Show dispatcher
docker exec kamailio-edge kamctl dispatcher show

# Add dispatcher destination
docker exec kamailio-edge kamctl dispatcher add 1 sip:1.2.3.4:5060 0 0 '' 'description'
```

### Debugging

```bash
# Increase log level
docker exec kamailio-edge kamcmd cfg.set_now_int core debug 3

# Tail logs
docker logs -f kamailio-edge

# Sngrep (SIP packet capture)
docker exec -it sngrep sngrep

# Filter by Call-ID
docker exec -it sngrep sngrep -c <call-id>

# tcpdump in container
docker exec kamailio-edge apk add tcpdump
docker exec kamailio-edge tcpdump -i any -n port 5060
```

---

## Conclusion

This project provides a **complete, production-ready SIP proxy/SBC** implementation using Kamailio. It demonstrates:

✓ **Security**: Flood protection, blacklisting, topology hiding
✓ **Scalability**: Load balancing, dispatcher, multiple backends
✓ **Flexibility**: ENUM, RTJSON, dynamic routing
✓ **Media Handling**: RTPEngine, codec filtering, SRTP
✓ **Observability**: CDRs, accounting, logging
✓ **Standards Compliance**: RFC 3261 (SIP), RFC 3264 (SDP), RFC 4566

The modular configuration approach makes it easy to:
- Understand each component
- Modify for specific requirements
- Add new features incrementally
- Debug issues efficiently

This is not just a learning project—it's a **blueprint for building real-world VoIP infrastructure**.

---

## Additional Resources

**Official Kamailio Documentation**:
- https://www.kamailio.org/wikidocs/

**SIP RFCs**:
- RFC 3261: SIP (Session Initiation Protocol)
- RFC 3263: SIP Locating Services
- RFC 3264: Offer/Answer Model with SDP
- RFC 3265: SIP Event Notification
- RFC 3891: Replaces Header
- RFC 4028: Session Timers
- RFC 5626: Managing Client-Initiated Connections

**RTPEngine**:
- https://github.com/sipwise/rtpengine

**PJSIP/PJSUA**:
- https://www.pjsip.org/

**Kamailio Modules**:
- https://www.kamailio.org/docs/modules/stable/

**Community**:
- sr-users@lists.kamailio.org (mailing list)
- kamailio.org/w/ (wiki)
- github.com/kamailio/kamailio (source code)

---

**Course Link**: [Learn SIP Through Kamailio on Udemy](https://www.udemy.com/course/learn-sip-through-kamailio/?referralCode=1091DEB811F7C7329839)

**Repository**: https://github.com/keithcroxford/kamailio-course

**License**: Check repository for license information

---

*This documentation was created to provide a comprehensive understanding of the Kamailio course project. For questions or issues, please refer to the course materials or GitHub repository.*
