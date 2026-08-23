# DNS - How DNS Works

---

## GROUP A: DNS Hierarchy (The Structure)

### What is the DNS Hierarchy?

DNS is organized like an **inverted tree**. The root is at the top, and everything branches downward.

### Visual Representation

```
                    .  (ROOT)
                    │
        ┌───────────┼───────────┬───────────┐
        │           │           │           │
      .com         .org        .io         .net
        │           │           │           │
     google       wiki      kubernetes   cloudflare
        │           │           │           │
    ┌───┴───┐       │       ┌───┴───┐       │
    │       │       │       │       │       │
   mail    www    en      docs   blog   api   www
```

### Read from Right to Left

```
www.google.com.
  ↑    ↑    ↑   ↑
  │    │    │   └── Root (.) - The top
  │    │    └────── TLD (.com) - The category
  │    └─────────── Domain (google) - The registered name
  └──────────────── Subdomain (www) - A specific service
```

---

### TERMS IN GROUP A

#### 1. Root (`.`)

**Definition:** The absolute top of the DNS hierarchy. Every fully qualified domain name ends with a dot (`.`) that represents the root.

**Analogy:** Like "Earth" or "The World" in a mailing address.

**Key Facts:**
- Represented by a dot (`.`) at the end of domain names
- Example: `www.google.com.` (the dot at the end is the root)
- We usually don't type it, but DNS always uses it internally
- The root zone file contains NS records for all TLDs

**What the Root Does:**
- Doesn't know individual domain IPs
- Knows where to find TLD servers (`.com`, `.org`, `.io`, etc.)
- Responds with referrals: "Ask the `.com` servers"

**Who Manages It:** IANA (Internet Assigned Numbers Authority), part of ICANN

---

#### 2. TLD (Top-Level Domain)

**Definition:** The highest level of domains in the DNS hierarchy, right below the root.

**Analogy:** Like a "Country" in a mailing address.

**Examples:**
| TLD | Purpose |
|-----|---------|
| `.com` | Commercial |
| `.org` | Organizations |
| `.edu` | Educational |
| `.io` | Tech companies (originally British Indian Ocean Territory) |
| `.gov` | Government |
| `.net` | Network infrastructure |

**Key Facts:**
- Managed by Domain Registries
- Each TLD has its own set of authoritative nameservers
- The root points to these TLD servers
- TLD servers know which domains are registered under them

**What TLD Servers Do:**
- Don't know individual website IPs
- Know where to find authoritative nameservers for each registered domain
- Example: `.com` knows that `google.com` is handled by Google's nameservers

---

#### 3. Second-Level Domain (SLD)

**Definition:** The part of the domain name immediately to the left of the TLD. This is what you register.

**Analogy:** Like a "City" in a mailing address.

**Examples:**
```
google.com
   ↑
   └── Second-Level Domain

kubernetes.io
   ↑
   └── Second-Level Domain

amazon.com
   ↑
   └── Second-Level Domain
```

**Key Facts:**
- This is what you actually buy/register
- Must be unique within its TLD
- Example: You can have `google.com` and `google.io` (different TLDs)
- You control all subdomains under your SLD

---

#### 4. Subdomain

**Definition:** Any label to the left of the second-level domain. It segments services under the main domain.

**Analogy:** Like a "Street" in a mailing address.

**Examples:**
```
www.google.com
  ↑
  └── Subdomain

docs.kubernetes.io
  ↑
  └── Subdomain

api.amazon.com
  ↑
  └── Subdomain

mail.google.com
  ↑
  └── Subdomain
```

**Key Facts:**
- You can create unlimited subdomains
- No need to register them separately
- Common examples: `www`, `docs`, `api`, `blog`, `mail`
- You can have multiple levels: `a.b.c.example.com`

---

#### 5. Hostname

**Definition:** The specific name of a machine or service, usually the leftmost label.

**Analogy:** Like a "Building Name" in an address.

**Examples:**
```
www.google.com
  ↑
  └── Hostname

docs.kubernetes.io
  ↑
  └── Hostname

mail.google.com
  ↑
  └── Hostname
```

**Key Facts:**
- Often the same as the subdomain
- In practice, "hostname" and "subdomain" are often used interchangeably
- The hostname identifies a specific resource

---

#### 6. FQDN (Fully Qualified Domain Name)

**Definition:** The complete domain name that specifies its exact location in the DNS hierarchy, ending with a dot.

**Analogy:** The complete mailing address including the country.

**Examples:**
```
www.google.com.   ← Includes the trailing dot
docs.kubernetes.io.   ← Includes the trailing dot
mail.amazon.com.   ← Includes the trailing dot
```

**Key Facts:**
- Always ends with a dot (`.`)
- Without the dot, it's a relative name
- Example: `www.google.com` (without dot) is the FQDN `www.google.com.`
- Most browsers/apps add the dot automatically

**Visual Breakdown:**
```
www . google . com .
 ↑      ↑       ↑    ↑
 │      │       │    └── Root
 │      │       └────── TLD
 │      └───────────── Second-Level Domain
 └──────────────────── Hostname/Subdomain
```

---

### Summary Table - Group A

| Term | Definition | Example |
|------|------------|---------|
| **Root (`.`)** | Top of DNS hierarchy | `.` at end of domain |
| **TLD** | Top-level domain category | `.com`, `.org`, `.io` |
| **Second-Level Domain** | The registered domain name | `google` in `google.com` |
| **Subdomain** | Label to the left of SLD | `www` in `www.google.com` |
| **Hostname** | Specific machine/service name | `www` in `www.google.com` |
| **FQDN** | Complete domain with trailing dot | `www.google.com.` |

---

## GROUP B: DNS Infrastructure (The Players)

### The Four Types of DNS Servers

There are exactly **four types** of DNS servers you need to understand. They work together in a chain.

```
          YOUR COMPUTER
               │
               ↓
    1. RESOLVER (Recursive)
               │
               ↓
    2. ROOT SERVER
               │
               ↓
    3. TLD SERVER
               │
               ↓
    4. AUTHORITATIVE SERVER
```

---

### TERMS IN GROUP B

#### 1. Recursive Resolver (Recursive DNS Server)

**Definition:** A DNS server that acts on behalf of clients, performing full DNS resolution by querying multiple servers until it finds the answer.

**Analogy:** A personal assistant who will go to any length to find information for you.

**Key Characteristics:**
- **Does the hard work** - It asks questions up and down the hierarchy
- **Caches answers** - Remembers responses to speed up future queries
- **Responds to clients** - Gives back the final IP address
- **Never has authoritative answers** - It just finds them

**Examples:**
- Google Public DNS: `8.8.8.8`
- Cloudflare DNS: `1.1.1.1`
- Quad9: `9.9.9.9`
- Your ISP's DNS server

**Analogy in Detail:**
You: "Find the address of John's Pizza"
Assistant: Goes to city hall, then county records, then the restaurant itself
Assistant: Returns with the address

**What It Does:**
```
Client → "www.google.com?"
Resolver → "I'll find out..."
Resolver → Root: "Who handles .com?"
Root → "Here are .com servers"
Resolver → .com: "Who handles google.com?"
.com → "Here are google's authoritative servers"
Resolver → google authoritative: "What's www.google.com?"
Authoritative → "142.250.190.14"
Resolver → Client: "142.250.190.14"
```

---

#### 2. Root Server

**Definition:** The top-level DNS servers that know the location of all TLD nameservers. There are 13 named root servers (A-M).

**Analogy:** The global directory that knows where to find every country's records.

**Key Characteristics:**
- **Only 13 names** (A.root-servers.net through M.root-servers.net)
- **Actually hundreds of instances** using Anycast
- **Knows TLD servers** - Only stores NS records for TLDs
- **Doesn't cache** - Always gives authoritative answers from its zone file
- **Doesn't know individual domain IPs**

**What Root Servers Know:**
```
. → .com servers
. → .org servers  
. → .io servers
. → .net servers
... (all TLDs)
```

**Root Server Response:**
Resolver: "Who handles .io?"
Root: ".io is handled by these nameservers: a0.nic.io, b0.nic.io"

---

#### 3. TLD Server (Top-Level Domain Server)

**Definition:** DNS servers responsible for a specific TLD. They know which authoritative nameservers are responsible for each registered domain under that TLD.

**Analogy:** The country's official records office that knows where each city's records are.

**Key Characteristics:**
- **Managed by Domain Registries**
- **Knows registered domains** - Has NS records for domains under its TLD
- **Doesn't cache** - Always gives authoritative answers from its zone file
- **Doesn't know individual website IPs**

**What TLD Servers Know:**
```
.com → google.com → Google's authoritative NS
.com → amazon.com → Amazon's authoritative NS
.com → microsoft.com → Microsoft's authoritative NS
```

**TLD Server Response:**
Resolver: "Who handles google.com?"
.com: "google.com is handled by these nameservers: ns1.google.com, ns2.google.com"

---

#### 4. Authoritative Nameserver

**Definition:** The DNS server that holds the actual DNS records for a specific domain and can provide definitive answers about that domain.

**Analogy:** The actual property records office that has the deed for a specific property.

**Key Characteristics:**
- **Source of truth** - Has the actual DNS records
- **Contains zone files** - Stores A, CNAME, MX, TXT, etc.
- **Can be primary or secondary** - Primary has original zone file, secondary has copies
- **Doesn't cache** - Always gives answers from its own records
- **Can answer any record type** - A, AAAA, CNAME, MX, etc.

**What Authoritative Servers Know:**
```
google.com → A → 142.250.190.14
google.com → MX → mail.google.com
www.google.com → CNAME → google.com
```

**Authoritative Server Response:**
Resolver: "What's the A record for www.google.com?"
Authoritative: "www.google.com. A 142.250.190.14"

---

### Authoritative vs Recursive - The BIG Distinction

This confuses everyone. Here's the simple difference:

| Recursive Resolver | Authoritative Nameserver |
|---------------------|--------------------------|
| **Asks questions** | **Answers questions** |
| "I'll go find it" | "I know the answer" |
| Gets info from others | Is the source of truth |
| Caches answers | Has the actual records |
| Can answer any domain | Only answers for its zone |
| Not authoritative | Is authoritative |

**Analogy:**
- **Recursive Resolver** = Private investigator who finds information
- **Authoritative Nameserver** = Official record-keeper who has the information

---

### Summary Table - Group B

| Term | Definition | Role | Example |
|------|------------|------|---------|
| **Recursive Resolver** | Finds DNS answers for clients | Asks questions, caches | `8.8.8.8` |
| **Root Server** | Knows TLD locations | Points to TLD servers | `a.root-servers.net` |
| **TLD Server** | Knows domain authoritative NS | Points to domain NS | `.com` servers |
| **Authoritative Nameserver** | Has actual DNS records | Provides final answers | `ns1.google.com` |

---

## GROUP C: DNS Zones & Zone Files

### What is a DNS Zone?

**Definition:** A portion of the DNS namespace that is managed by a specific administrative authority.

**Analogy:** Think of it as a department in a company. Each department is responsible for its own area.

### Zone vs Domain - The Distinction

| Zone | Domain |
|------|--------|
| Administrative concept | Naming concept |
| "Who manages this" | "What is this called" |
| Can be a subset of a domain | The complete name |
| Can be delegated | Can have subdomains |

**Example:**
```
google.com (Domain)
    │
    ├── google.com (Zone - managed by Google's DNS)
    │
    └── maps.google.com (Could be separate zone if delegated)
```

### Zone Delegation Example

```
example.com (Parent Zone)
    │
    ├── api.example.com (Managed by parent)
    │
    ├── www.example.com (Managed by parent)
    │
    └── dev.example.com (Delegated to separate zone)
             │
             └── New Zone with its own authoritative NS
```

**Why delegate?**
- Separate teams manage different parts
- Load distribution
- Security boundaries
- Organizational structure

---

### DNS Zone File

**Definition:** The actual file/record that contains all the DNS records for a zone.

**Analogy:** The actual folder with all the papers (records) for a department.

### Zone File Structure

```
$TTL 3600

@       IN  SOA     ns1.example.com. admin.example.com. (
                    2025072401
                    3600
                    1800
                    604800
                    86400 )

@       IN  NS      ns1.example.com.
@       IN  NS      ns2.example.com.

@       IN  A       192.168.1.10
www     IN  CNAME   example.com.
docs    IN  A       192.168.1.20
mail    IN  MX 10   mail.example.com.
mail    IN  A       192.168.1.30
```

---

### TERMS IN GROUP C

#### 1. Zone File

**Definition:** A text file that contains DNS records for a specific zone.

**Analogy:** The actual document that contains all the addresses.

**Key Facts:**
- Stored on authoritative nameservers
- Contains all DNS records for that zone
- Must contain SOA and at least one NS record
- Updated when DNS records change

---

#### 2. SOA (Start of Authority) Record

**Definition:** A record that marks the beginning of a zone and contains administrative metadata.

**Analogy:** The cover page of a department's folder with information about who's in charge and how things work.

**SOA Fields:**
```
@       IN  SOA     ns1.example.com. admin.example.com. (
                    2025072401  ; Serial number
                    3600        ; Refresh interval (seconds)
                    1800        ; Retry interval (seconds)
                    604800      ; Expire time (seconds)
                    86400 )     ; Minimum TTL
```

| Field | Purpose |
|-------|---------|
| **Primary NS** | The main authoritative nameserver |
| **Email** | Administrator's contact (admin@example.com) |
| **Serial** | Version number - increment when zone changes |
| **Refresh** | How often secondary NS checks for updates |
| **Retry** | How often to retry if refresh fails |
| **Expire** | When to stop using stale data |
| **Minimum TTL** | Default TTL for negative responses |

**Why SOA Matters:**
- Tells secondaries when to update
- Identifies zone administrator
- Version control for zone transfers

---

#### 3. NS (Name Server) Record

**Definition:** Identifies the authoritative nameservers for a zone/domain.

**Analogy:** The sign that says "This department's records are kept at this office."

**Example:**
```
@       IN  NS      ns1.digitalocean.com.
@       IN  NS      ns2.digitalocean.com.
```

**Key Facts:**
- Must have at least one NS record in every zone file
- These records must match what the parent zone (TLD) has
- Primary and secondary nameservers should be listed
- Used to find the authoritative server

---

#### 4. A Record

**Definition:** Maps a domain name to an IPv4 address (32-bit).

**Analogy:** The actual street address (in IPv4 format).

**Example:**
```
example.com.    A    192.168.1.10
www.example.com. A   192.168.1.10
```

**Key Facts:**
- Used for IPv4 addresses
- Most common DNS record
- A domain can have multiple A records (load balancing)

---

#### 5. AAAA Record

**Definition:** Maps a domain name to an IPv6 address (128-bit).

**Analogy:** The newer, longer street address format.

**Example:**
```
example.com.    AAAA    2001:db8::10
```

**Key Facts:**
- Used for IPv6 addresses
- Same purpose as A record, but for IPv6
- Both A and AAAA can exist for the same name

---

#### 6. CNAME (Canonical Name) Record

**Definition:** Creates an alias that points one domain name to another.

**Analogy:** A shortcut or nickname that redirects to the real name.

**Example:**
```
www.example.com.    CNAME    example.com.
```

**How it works:**
```
Query: www.example.com
         ↓
CNAME: www.example.com → example.com
         ↓
A Record: example.com → 192.168.1.10
         ↓
Response: 192.168.1.10
```

**Key Facts:**
- Cannot coexist with other record types at the same name
- Can point to another CNAME (but avoid)
- Used for aliasing services

**When to use CNAME:**
- `www.example.com` → `example.com`
- `cdn.example.com` → `cdn.provider.com`
- Multiple hostnames pointing to same server

---

#### 7. MX (Mail Exchange) Record

**Definition:** Specifies the mail server(s) responsible for receiving email for a domain.

**Analogy:** The post office location for a particular area.

**Example:**
```
example.com.    MX 10    mail.example.com.
example.com.    MX 20    backup-mail.example.com.
```

**MX Fields:**
| Field | Purpose |
|-------|---------|
| **Priority** | Lower number = higher priority |
| **Mail Server** | The domain name of the mail server |

**How it works:**
```
Email: user@example.com
         ↓
DNS Lookup: MX record for example.com
         ↓
Priority 10 → mail.example.com
Priority 20 → backup-mail.example.com
         ↓
Connect to mail.example.com first
```

**Key Facts:**
- Must point to a domain name, not an IP address
- Lower priority number = preferred server
- Can have multiple MX records for redundancy

---

#### 8. TXT Record

**Definition:** Stores arbitrary text information for a domain.

**Analogy:** A notes section in a file.

**Examples:**
```
example.com.    TXT    "v=spf1 include:_spf.google.com ~all"
example.com.    TXT    "google-site-verification=abc123"
```

**Common Uses:**
- **SPF** - Sender Policy Framework (email authentication)
- **DKIM** - DomainKeys Identified Mail
- **Domain verification** - Prove you own the domain
- **Security policies** - DMARC, etc.

---

#### 9. Glue Record

**Definition:** A special A/AAAA record that provides the IP address of a nameserver to prevent circular dependencies.

**Analogy:** A cheat sheet that tells you where the next person is even if they're in the same building.

**The Problem:**
```
example.com.    NS    ns1.example.com.
ns1.example.com.    A    ? (We need this to contact ns1.example.com)
```

**The Solution (Glue Record):**
```
example.com.    NS    ns1.example.com.
ns1.example.com.    A    192.168.1.1  ← Glue Record
```

**Key Facts:**
- Only needed when nameserver is within the same domain
- Provided by the parent zone (TLD)
- Prevents circular dependencies in DNS resolution

---

### Summary Table - Group C

| Term | Definition | Example |
|------|------------|---------|
| **Zone** | Managed portion of DNS namespace | `example.com` zone |
| **Zone File** | File containing DNS records | `example.com.zone` |
| **SOA** | Administrative metadata | Start of Authority record |
| **NS** | Authoritative nameserver | `ns1.example.com` |
| **A** | IPv4 address | `192.168.1.10` |
| **AAAA** | IPv6 address | `2001:db8::10` |
| **CNAME** | Alias | `www → example.com` |
| **MX** | Mail server | `mail.example.com` |
| **TXT** | Arbitrary text | SPF records |
| **Glue** | Nameserver IP | `ns1.example.com → 192.168.1.1` |

---

## GROUP D: Domain Registration (Who Does What)

### The Three Key Players

```
YOU (Domain Owner)
    ↓
REGISTRAR (Where you buy)
    ↓
REGISTRY (The official database)
    ↓
DNS HOSTING (Where records live)
```

### TERMS IN GROUP D

#### 1. Domain Registrar

**Definition:** A company accredited by a registry to sell domain names to the public.

**Analogy:** A real estate agent who helps you buy property.

**Key Responsibilities:**
- Sells domain registrations
- Manages your account and payment
- Updates WHOIS information
- Pushes NS records to the registry
- Renews domains before expiration

**Examples:**
- GoDaddy
- Namecheap
- Google Domains
- Cloudflare Registrar

**What They Do:**
```
You → "I want to buy example.com"
Registrar → "It's available. Here's the price."
You → "OK, I'll buy it."
Registrar → "I'll register it with the .com registry"
Registrar → "I'll also set the default nameservers"
```

---

#### 2. Domain Registry

**Definition:** The organization responsible for maintaining the database for a specific TLD.

**Analogy:** The government's official property records office.

**Key Responsibilities:**
- Maintains the TLD zone file
- Keeps track of which domains are registered
- Stores NS delegation records for each domain
- Operates the TLD nameservers

**Examples:**
| TLD | Registry |
|-----|----------|
| `.com` | Verisign |
| `.org` | Public Interest Registry (PIR) |
| `.io` | NIC.IO (Internet Computer Bureau) |
| `.edu` | Educause |

**What They Do:**
```
Registrar → "Please register example.com"
Registry → "OK, I'll add it to the .com database"
Registry → "I'll add the NS records you provided"
Registry → "Now DNS resolvers can find example.com"
```

---

#### 3. DNS Hosting Provider

**Definition:** The service that hosts your DNS zone files and answers queries as the authoritative nameserver.

**Analogy:** The actual filing cabinet where you keep your records.

**Key Responsibilities:**
- Stores your DNS zone files
- Answers DNS queries as authoritative server
- Allows you to manage DNS records
- Provides nameservers

**Examples:**
- Cloudflare DNS
- AWS Route 53
- Google Cloud DNS
- Your registrar's DNS (if they offer it)

**What They Do:**
```
You → "I want www.example.com to point to 192.168.1.10"
DNS Host → "I'll add that A record to the zone file"
DNS Host → "When resolvers ask, I'll give that answer"
```

---

### The Complete Registration Flow

```
1. YOU want example.com
       ↓
2. REGISTRAR (e.g., GoDaddy)
   • Checks if available
   • You pay
   • Sets WHOIS info
       ↓
3. REGISTRY (e.g., Verisign for .com)
   • Records: "example.com is registered"
   • Records: "example.com's NS are ns1.godaddy.com, ns2.godaddy.com"
   • Adds NS records to .com zone file
       ↓
4. DNS HOSTING (e.g., GoDaddy's DNS)
   • Stores zone file with A, CNAME, MX records
   • Answers queries for example.com
       ↓
5. NOW example.com works on the internet
```

### Registrar vs DNS Host - The Critical Distinction

**SAME COMPANY EXAMPLE:**
```
You → GoDaddy → Register domain → GoDaddy also hosts DNS
```
You can do everything in one place.

**DIFFERENT COMPANIES EXAMPLE:**
```
You → GoDaddy → Register domain
You → Route 53 → Host DNS
You → GoDaddy → Update NS records: "Use Route 53 nameservers"
```

Now:
- GoDaddy handles billing and WHOIS
- Route 53 handles DNS resolution
- The .com registry has NS records pointing to Route 53

### Why Separate?

| Reason | Explanation |
|--------|-------------|
| **Flexibility** | Use best DNS hosting features |
| **Security** | Separate registrar and DNS provider |
| **Reliability** | Avoid single point of failure |
| **Cost** | Different pricing for each service |

---

### Summary Table - Group D

| Term | Definition | Example |
|------|------------|---------|
| **Registrar** | Sells domain names | GoDaddy, Namecheap |
| **Registry** | Manages TLD database | Verisign (.com), NIC.IO |
| **DNS Hosting** | Hosts DNS records | Cloudflare, Route 53 |

---

## GROUP E: DNS Resolution (The Complete Flow)

### The Full Journey with All Terms

Let's trace `www.amazon.com` from typing to loading the page, using EVERY term we've learned.

### Step-by-Step

#### 1. User Types URL

```
User types: www.amazon.com
```

#### 2. Local Cache Check

```
Computer checks:
- Browser cache
- OS DNS cache
- hosts file
```

**Terminology:** This is the **client-side caching**.

#### 3. Query to Recursive Resolver

```
Computer → "What's the IP of www.amazon.com?"
Computer → Recursive Resolver (e.g., 8.8.8.8)
```

#### 4. Resolver Cache Check

```
Recursive Resolver checks its cache
If found → Return IP
If not → Continue to root
```

#### 5. Query Root Server

```
Resolver → Root Server: "Who handles .com?"
```

**What's happening:** The resolver asks the root to find the `.com` TLD servers.

#### 6. Root Response

```
Root → "Here are the .com TLD nameservers"
Root → NS Records: a.gtld-servers.net, b.gtld-servers.net
Root → Glue Records: IP addresses for these nameservers
```

**Terminology:**
- **Referral:** The root doesn't know the answer, it points to the next level
- **NS Record:** Tells who handles `.com`
- **Glue Record:** Gives IPs of those nameservers

#### 7. Query TLD Server

```
Resolver → .com TLD Server: "Who handles amazon.com?"
```

**What's happening:** The resolver asks the `.com` server to find Amazon's authoritative nameservers.

#### 8. TLD Response

```
.com → "Here are Amazon's authoritative nameservers"
.com → NS Records: ns1.amazon.com, ns2.amazon.com
.com → Glue Records: IP addresses for these nameservers
```

**Terminology:**
- **Referral:** The TLD points to the domain's authoritative servers
- **Delegation:** The TLD has delegated `amazon.com` to Amazon's NS

#### 9. Query Authoritative Server

```
Resolver → Amazon's Authoritative NS: "What's www.amazon.com?"
```

**What's happening:** The resolver finally asks the authority for the domain.

#### 10. Authoritative Response

```
Amazon's Authoritative NS → "www.amazon.com A 83.224.16.20"
Amazon's Authoritative NS → TTL: 300 seconds
```

**Terminology:**
- **A Record:** The actual IPv4 address
- **TTL:** Time to live - cache for 300 seconds

#### 11. Resolver Caches

```
Resolver stores: www.amazon.com → 83.224.16.20
Resolver stores with TTL: 300 seconds
```

**Terminology:**
- **Caching:** Storing for future queries
- **TTL:** Determines how long to cache

#### 12. Resolver Responds to Client

```
Resolver → Computer: "83.224.16.20"
```

#### 13. Computer Caches

```
Computer stores: www.amazon.com → 83.224.16.20
```

#### 14. Browser Connects

```
Browser → TCP connection to 83.224.16.20:443
Browser → TLS handshake
Browser → HTTP request
Browser → Receives webpage
```

---

### Visual Flow with All Terms

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR COMPUTER                           │
│  Browser → "www.amazon.com"                                │
│     ↓                                                      │
│  Local Cache? (Browser/OS/hosts)                          │
│     ↓                                                      │
│  No cache → Send query                                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────┐
│               RECURSIVE RESOLVER (8.8.8.8)                │
│  Check cache?                                             │
│     ↓                                                     │
│  No cache → Query root                                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                    ROOT SERVER (.)                         │
│  "Who handles .com?"                                      │
│     ↓                                                     │
│  Response: ".com TLD servers"                             │
│  NS Records + Glue Records                                │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                   .COM TLD SERVER                          │
│  "Who handles amazon.com?"                                │
│     ↓                                                     │
│  Response: "Amazon's authoritative NS"                    │
│  NS Records + Glue Records                                │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────┐
│           AMAZON AUTHORITATIVE NAMESERVER                  │
│  "What's www.amazon.com?"                                 │
│     ↓                                                     │
│  Response: "83.224.16.20"                                 │
│  A Record + TTL                                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────┐
│               RECURSIVE RESOLVER (8.8.8.8)                │
│  Cache: www.amazon.com → 83.224.16.20                    │
│     ↓                                                     │
│  Return IP to computer                                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                     YOUR COMPUTER                           │
│  Cache IP                                                │
│     ↓                                                     │
│  Browser connects to 83.224.16.20                         │
│     ↓                                                     │
│  TCP → TLS → HTTP → Webpage loaded                        │
└─────────────────────────────────────────────────────────────┘
```

---

## GROUP F: TTL and Caching

### What is TTL?

**Definition:** Time To Live - the duration (in seconds) that a DNS record can be cached.

**Analogy:** The expiration date on milk. After it expires, you check for fresh milk.

### TTL Values and Their Uses

| TTL | Duration | Use Case |
|-----|----------|----------|
| 60-300 | 1-5 minutes | Dynamic IPs, load balancers |
| 300-600 | 5-10 minutes | Services that might change |
| 600-3600 | 10-60 minutes | Normal production services |
| 3600-86400 | 1-24 hours | Stable services, static content |
| 86400+ | 1+ days | CDN domains, rarely changed |

### Why TTL Matters

**Scenario: Changing IP Address**

```
Current: www.example.com → 192.168.1.10 (TTL: 86400 seconds)
                                  ↓
You want to change to: → 192.168.1.20

Problem: People's resolvers have cached 192.168.1.10 for 24 hours!

Best Practice:
1. Lower TTL to 60 seconds (24 hours before change)
2. Wait 24 hours (old TTL expires)
3. Change IP to 192.168.1.20
4. People start seeing new IP within 60 seconds
5. Increase TTL back to normal after change
```

### Cache Layers

```
1. Application Cache (Browser)
2. Operating System Cache
3. Recursive Resolver Cache
4. ISP or Public DNS Cache
```

### TTL and DNS Propagation

**Definition:** "DNS Propagation" is the time it takes for all cached copies of a DNS record to expire and be updated.

**Myth:** "DNS Propagation" is not a technical process where something "spreads." It's just cached records expiring.

---

## COMPLETE TERM GLOSSARY (Alphabetical)

| Term | Definition |
|------|------------|
| **A Record** | Maps a name to an IPv4 address |
| **AAAA Record** | Maps a name to an IPv6 address |
| **Authoritative Nameserver** | The source of truth for a DNS zone |
| **CNAME Record** | Alias that points one name to another |
| **DNS** | Domain Name System - translates names to IPs |
| **DNS Hosting Provider** | Service that stores DNS zone files |
| **FQDN** | Fully Qualified Domain Name - complete with root dot |
| **Glue Record** | IP address of a nameserver to prevent circular dependencies |
| **Hostname** | The name of a specific machine/service |
| **MX Record** | Mail Exchange - specifies mail servers |
| **NS Record** | Name Server - identifies authoritative servers |
| **Recursive Resolver** | DNS server that finds answers for clients |
| **Registrar** | Company that sells domain registrations |
| **Registry** | Organization managing a TLD's database |
| **Root** | The top of the DNS hierarchy (.) |
| **Root Server** | DNS server that knows TLD locations |
| **Second-Level Domain** | The registered domain name (google in google.com) |
| **SOA Record** | Start of Authority - administrative metadata for a zone |
| **Subdomain** | A label to the left of the main domain |
| **TLD** | Top-Level Domain (.com, .org, .io) |
| **TLD Server** | DNS server responsible for a specific TLD |
| **TTL** | Time To Live - how long to cache a record |
| **TXT Record** | Stores arbitrary text information |
| **Zone** | A managed portion of the DNS namespace |
| **Zone File** | The file containing all records for a zone |

---

## FINAL SUMMARY: The Mental Model

### The Complete Picture

```
THE HIERARCHY:
. (Root) → .com (TLD) → amazon.com (Domain) → www (Subdomain)

THE PLAYERS:
Your Computer → Recursive Resolver → Root → TLD → Authoritative

THE DATA:
Zone Files → Records (A, CNAME, MX, etc.)

THE PROCESS:
Type domain → Check caches → Recursive resolver walks hierarchy → Authoritative returns IP → Connect

THE ROLES:
Registrar sells it → Registry tracks it → DNS Host stores it → Resolvers find it → You use it
```

### The One Sentence Summary

> DNS is a hierarchical, distributed system where recursive resolvers walk from the root through TLD servers to authoritative nameservers to find the IP address for a domain name, using caching (controlled by TTL) to make the process fast.

---
