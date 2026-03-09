# F5 Big-IP Configuration Standards

This is a practical guide to designing, naming, and maintaining F5 Big-IP load balancer configurations that scale reliably over years. It captures operational lessons learned—the failure modes that documentation doesn't warn you about, the subtle configuration choices that compound into chaos or stability, and the disciplined approaches that keep environments maintainable when you're managing hundreds of objects.

This is not a replacement for F5 documentation. It is a companion to it: documentation explains *what* the tools do; this guide explains *when and why* you should use them, and what goes wrong when you don't.

---

## Philosophy

Load balancer configuration is infrastructure policy made concrete. Small inconsistencies and unclear choices propagate across years. A pool named `backend_prod_web` versus `prod-web-pool` versus `app01_app02_app03_nodes` costs your team seconds of cognitive load on every change, and hours across years. Poor health monitor design doesn't just fail to detect problems—it *creates* them, triggering failover cascades when the application is actually healthy.

These standards exist because consistency and clarity are force multipliers. A team that can read a virtual server configuration and immediately understand what it does, why it exists, which application owns it, and which environment it runs in is a team that makes fewer mistakes, changes faster, and responds to incidents more effectively.

---

## Naming Conventions

Names are the primary interface between intention and implementation. A good name encodes purpose, environment, and ownership. It must remain legible at a glance across decade-old configurations and brand-new ones.

### Three-Part Structure

Follow this structure for all user-created objects except monitors (which have conventions below):

```
APPLICATION-ENVIRONMENT-OBJECT_TYPE
```
The source of truth that these objects are created against is DNS:
a VIP is service.domain.tld at precisely the moment that DNS resolution drives traffic to it. 
By unwaveringly indexing service names against DNS, you have the first of many self-documenting features of your network that cost nother other than consistency. The VIP includes both the UQDN and the IP so even if the records change over time, you know how it was originally deployed and to the extent you keep your config to match the change, anyone will instantly know what the intent of the configuration is. 

These conventions are easily expressed in automation interfaces whether a home grown script or an IaC interface. 

Examples:
- Pool Members: <slc>-pm-<uqdn> ie: prd-pm-usfld1svcpr01
- Pool: <slc>-pl-<svc> example: prd-pl-svc
- Virtual Server: <slc>-vs-<svc>-<ipv4/6.addr>-<t/u>-<listenport>
- iRule: <slc>-<svc>-description<yyyymmdd>-<ordinal>
- TCP Profile: <tcp|udp>-<svc>-<feature>

### Why This Matters

- **Consistency compounds**: A shop with 20 pools follows one naming convention; a shop with 500 pools across different eras follows seven. The operational cost of the second grows faster than linearly.
- **Readability across time**: You will not remember what `pool_15` does when you encounter it in a change request three years from now. `dev-pl-service` tells you immediately. iRules are especially prone to iteration and as such have a timestamp and ordinals in the object name to identify the direction of change. 
- **Ownership clarity**: When a rule fails, it's obvious which team owns it. Ambiguous names require detective work for every incident. indexing to DNS allows you to validate the entire config with a small script. 

### Specific Guidelines

**Pools and Nodes**
- Pool: `<slc>-pl-<svc>` (e.g., `prd-pl-service`, `dev-pl-service2`)
- the first such pool is named with no ordinal and each subsequent pool is incremented. 
- Nodes match the DNS A and PTR records of the Node IP at the time of creation.
- Key feature of this config is that the pool matches the VIP in log parsing and the node name can be reversed to an IP via DNS lookup. 

**Virtual Servers**
- Format: `<slc>-vs-<svc>-<ip.addr>-<listenport>` (e.g., `prd-vs-service-10.1.1.150-443`)
- VIPs for the same service sort naturally and the next VIP changes only the port.
- Now the VIP when it appears in logs will immediately provide positive identification of the service in a single line.
- Service is DNS centric still and the A and PTR records are what drive client traffic to the VIP.

**Health Monitors**
- Monitors are the exception: they often serve multiple pools so they omit port info
- Format: `<slc>-mon-<svc>-feature` (e.g., `prd-mon-service-sni`, `stg-mon-service-radius`, `dev-mon-service-tcphalfopen`)

**iRules and Profiles**
- iRule: `<slc>-<svc>-feature<yyyymmdd>-<ordinal>` (e.g., `prd-service-hdr-rewrite20260606-7`, `dev-service-root-redir20260303-4`)
- Profile: `<protocol>-<svc>-feature` (e.g., `http-portal-xff`, `http-service-64khdr`)

**SSL Certificates and Keys**
- Follow application/environment structure: `<fqdn>-<yyyymmdd>` with -crt and -key if necessary however we will specifiy pkcs12/pfx from upstream for leaf certs for uniformity. 
- Use FQDN in description field for clarity and the date to ID.
- Chain certs additionally contain the strings '-chain-' and '-<root>-' or '-<intmdt>- if necessary.

---

## Load Balancing Algorithm Selection

F5 offers multiple load balancing algorithms. Documentation describes them accurately. What it doesn't always explain is when each choice is actually correct, and what failure looks like when you choose poorly.

### Least Connections (node) (Problematic Default)

**How it works**: Directs new connections to the pool member with the fewest active connections.

**Where it fails**: When your backend connections have *widely varying lifetimes* or processing times.

**Failure mode**: 
- A backend processes quick requests (100ms): takes 100 connections.
- Another backend processes slow operations (30s): takes 10 connections.
- Least Connections will still route the next connection to the slow backend because it's "least busy."
- Your slow backend gets slower. More connections back up. The system develops hot spots instead of balancing.

**Use this when**: 
- Backend response times are consistent.
- Connections are genuinely equivalent in cost.
- You've tested that load distributes evenly in production.

**Exactly the wrong algorithm when**:
- You have background job processing and quick API calls on the same backends.
- You have geographic distribution with variable latency.
- Your application has heavyweight and lightweight request types.

### Round Robin (Safe Default)

**How it works**: Rotates through pool members in order.

**Advantages**:
- Predictable
- Doesn't require tracking connection state
- Fair across homogeneous backends
- Clearly documents that you *chose* fairness over optimization

**Use this when**:
- You're uncertain about backend behavior
- Connections are reasonably equivalent
- You want debuggable, reproducible load distribution
- You value simplicity and transparency over micro-optimization

**Round Robin is not a compromise—it's often the correct choice.**

### Ratio (Intentional Imbalance)

**How it works**: Allocates connections in specified ratios across pool members.

**Use this when**:
- You have heterogeneous backends with known performance differences.
- You're scaling up/down and want to gradually shift load.
- You're testing a new backend and want controlled traffic (e.g., 90/10 split).

**Document the ratios explicitly in pool description**: why is one backend getting 40% and another 60%?

### Fastest (Usually Wrong)

**How it works**: Routes to the backend with the lowest latency.

**Problem**: This is observed latency to the load balancer, not to your actual user. It optimizes for network proximity, not application performance. For most use cases, this creates hot spots rather than balance.

**Skip this one unless you have a very specific reason.**

---

## Health Monitor Design

A health monitor does not monitor health. It monitors *availability*. You can be available but unhealthy. You can pass health checks and be broken. Misunderstanding this distinction is the root cause of every production incident attributed to "flaky monitoring."

### The Core Problem

A typical production failure looks like:
1. Your application has a bug that only affects certain request types.
2. Your monitor sends generic requests that don't trigger the bug.
3. Health checks pass.
4. Real clients fail.
5. Traffic stays on the broken backend.
6. Incident escalates and is blamed on the load balancer.

Alternatively:
1. Your backend hangs on shutdown.
2. Health checks time out slowly.
3. The monitor marks it down, but by then it's already picked up 10 slow requests.
4. Those clients experience 30-second hangs instead of fast failover.
5. Incident escalates and is blamed on the load balancer.

### Designing a Monitor That Actually Works

**Interval and Timeout: The Essential Ratio**

Your timeout *must* be significantly shorter than your interval. A typical mistake:
```
interval: 10 seconds
timeout: 5 seconds
```

This means: "Check every 10 seconds, wait 5 seconds for a response."

Why this fails:
- One slow response consumes the entire timeout window.
- The monitor is already checking again before the first response returns.
- You have concurrent checks piling up.
- False positives become common.

**Correct approach**:
```
interval: 10 seconds
timeout: 2 seconds
```

Now: "Check every 10 seconds, wait 2 seconds for a response."
- Slow responses fail fast and clearly.
- False positives are rare.
- Failure detection is responsive.

**General rule**: `timeout = interval / 5` or tighter.

**Fall-off count** (how many consecutive failures before marking down):
- Default is 3. This is reasonable and conservative.
- If you set to 1, a single packet loss marks down a healthy backend.
- If you set to 5 or higher, you're waiting for real failure to be obvious, not just suspicious.
- 3 is almost always correct: it's resilient to transient issues but responsive to real problems.

### HTTP and HTTPS Monitors: Detecting Actual Health

The difference between a monitor that detects *availability* versus *health*:

**Availability monitor** (insufficient):
```
GET /
Expect status: 200
```

This confirms: "The web server responds." It does not confirm: "Your application works."

**Health monitor** (better):
```
GET /health
Expect status: 200
Expect body contains: "ok"
```

Send requests that exercise the parts of your application that matter. If your app has a database, have the health endpoint test the database. If it has dependencies, test them.

**Application-specific monitors**:
- Checkout service → Monitor endpoint that actually attempts a transaction (without committing).
- Authentication service → Monitor that validates a test token.
- Database connection pool → Monitor that runs a test query.

The cost of implementing a real health endpoint is trivial compared to the cost of spending 3 hours on a production incident because the monitor didn't catch degradation early.

### Avoid These Monitor Mistakes

**TCP monitors on HTTP services**: A TCP port responding doesn't mean your application is healthy. The overhead of an HTTP health check is negligible.

**Monitors that cause load**: A monitor that runs expensive queries or initiates actual transactions creates cascading load during incidents. Keep monitors lightweight and separate from business logic.

**Shared monitors across different applications**: A monitor designed for database availability is not correct for API health. When you need 12 different monitors because each application is different, that's not fragmentation—that's accuracy. Create them.

**Not testing monitor behavior under load**: Test your monitors at production scale. A monitor that works fine at 100 requests per second might become the bottleneck at 10,000 requests per second.

---

## Persistence Profile Choices

Persistence profiles ensure that a single client's requests go to the same backend. Methods differ fundamentally in how they work and what breaks under different network topologies.

### Source IP (Simplest, Often Wrong)

**How it works**: All requests from a single IP address go to the same pool member.

**Works perfectly when**:
- Clients are on stable, individual IPs.
- All traffic is direct (no upstream load balancer, no NAT).
- Sessions are IP-scoped (ATM era applications).

**Breaks when**:
- Your load balancer sits behind *another* load balancer (everything from Big-IP to Big-IP to cloud load balancer). All traffic appears to come from one IP.
- Clients are on mobile networks that hand off towers (all traffic appears to come from the carrier's IP).
- You have a corporate proxy or VPN backend (hundreds of users share one IP).
- Your clients are cloud instances without stable public IPs.

**In production**: Source IP often appears to work until a client connects through new infrastructure, then creates a mysterious sticky session that won't load-balance properly.

**Use source IP only if you've specifically verified that your client base has stable, non-shared IPs.**

### Cookie-Based Persistence (More Robust)

**How it works**: Load balancer sets a cookie that encodes the pool member ID. Subsequent requests include the cookie and return to that member.

**Advantages**:
- Works through upstream load balancers (the cookie is in the request, not the IP).
- Works with mobile clients (the cookie persists across network changes).
- Works with shared IPs (each client has their own cookie).

**Implementation details**:
- HTTP Set-Cookie header → Load balancer sets `BigIPCookie=member_id`
- Client includes cookie in subsequent requests
- Load balancer reads cookie and routes accordingly

**Breaks when**:
- Client doesn't support cookies (legacy applications, some APIs).
- Client clears cookies between requests.
- You have strict HTTPS requirements and the cookie isn't marked Secure/HttpOnly properly.

**Use this for**: Web browsers, mobile apps, any client you control or that's modern HTTP.

**Cookie persistence in F5**: Create a cookie profile that matches your application's session timeout. F5 can set the cookie automatically or rewrite it.

This feature is often marked as a security vulnerability by web securitiy scans because the simple algorithm used to encode the cookie data can theoretically provide an attacker information about the infrastructure that didn't already know. 

Cookies are an active measure and as such do create overhead, especially when configured with encryption and other features for compliance reasons. 

Some applications are already pushing the boundaries of header length so any additional items are going to be viewed as counter to application health. 

### SSL Session ID Persistence (Reference, Rarely Needed)

**How it works**: Routes subsequent HTTPS requests that reuse the same SSL session to the same backend.

**Advantages**:
- No application awareness required.
- Automatic if your backends negotiate SSL sessions with clients.

**Breaks when**:
- Client doesn't reuse SSL sessions (many modern clients don't by default for security).
- SSL session expires, clients need a new session.
- You have multiple SSL termination points; sessions don't map cleanly.

**In practice**: SSL session persistence is rarely necessary. Use cookie-based or source IP (with caveats). Don't rely on it unless you understand the specific constraint.

### Choosing in Practice

- **Web applications with browsers**: Cookie-based persistence.
- **Legacy applications that require session stickiness**: Cookie-based persistence (with fallback to source IP if cookies fail).
- **Stateless services without session requirements**: No persistence (let each request round-robin).
- **Behind another load balancer**: Never source IP; use cookie-based.
- **Mobile app or client behind residential NAT**: Cookie-based or HTTP-based custom persistence.

---

## iRule Conventions

iRules are F5's scripting language. They are powerful, but their power is also their danger. A poorly written iRule can do more damage than good. Conventions matter.

### Structure and Header

Every iRule should open with a clear header:

```tcl
# ============================================================================
# iRule: prd-portal-jwt20260301
# Purpose: Validate JWT tokens for authentication service requests
# Environment: Production
# SVCOwner: SECOPS
# Last Updated: 2026-03-01
# Last Updated By: network services
# 
# Events:
#   - HTTP_REQUEST: Validate JWT, inject headers
#
# Dependencies:
#   - /Common/jwt-validation-library (external iRule)
#   - auth-prod-https-profile (client-side SSL)
#
# Change Log:
#   2026-03-01: Added retry logic for token validation service failures
#   2026-02-15: Initial implementation
# ============================================================================
```

This takes 30 seconds to write and saves hours when someone needs to understand this rule in production on Sunday at 2 AM.

### Priority: What Event, When?

iRules execute in specific event contexts. The cost of doing work at the wrong time is significant.

**Event Priority (early to late)**:
1. `CLIENT_ACCEPTED` - Before the TCP connection is established
2. `CLIENT_HELLO` - SSL handshake beginning (TLS only)
3. `HTTP_REQUEST` - Full HTTP request received
4. `HTTP_RESPONSE` - Full HTTP response received
5. `SERVER_CONNECTED` - Connection to backend established
6. `SERVER_RESPONSE` - Response from backend received

**Cost differences**:
- Work in `CLIENT_ACCEPTED`: Low cost, affects routing decisions.
- Work in `HTTP_REQUEST`: Moderate cost, you have request data.
- Work in `HTTP_RESPONSE`: Higher cost, you're delaying responses back to client.
- Work in `CLIENT_HELLO`: Very low cost, perfect for SSL routing decisions.

**Common mistakes**:
- Doing HTTP parsing in `CLIENT_ACCEPTED` (you don't have the request yet).
- Rewriting responses in iRule when profiles would be more efficient.
- Making external API calls in `HTTP_RESPONSE` (you're delaying the response; use a separate work queue).

### Performance Implications

iRules are evaluated per-request or per-connection. Their performance matters at scale.

- Events with heavy iRule logic (`HTTP_REQUEST` with external API calls) can become the bottleneck.
- Simple URL routing in iRules is fine; complex business logic starts to hurt at 10K+ RPS.
- External API calls (especially failures) in iRules cause cascading timeouts.


**Comments in iRules**:

```tcl
# ✓ Good: explains the why
if { [HTTP::header exists X-Custom-Auth] } {
    # Custom auth header present; route to dedicated pool for client validation
    pool custom-auth-prod-pool
}

# ✗ Bad: explains the what (obvious from code)
if { [HTTP::header exists X-Custom-Auth] } {
    # Check if the header exists
    pool custom-auth-prod-pool  # Route to pool
}
```

---

## SSL Profile Organization

SSL configuration sprawls. Without discipline, you end up with 50 custom profiles that are mostly duplicates. With discipline, you have 5 profiles that are consistently used and understood.

### Client vs. Server Profiles (Critical Distinction)

**Client Profile** (certificate, protocols, ciphers *facing your clients*):
- Determines what TLS versions clients must support.
- Determines which ciphers are acceptable from clients.
- Usually more permissive (supporting browsers requires compatibility).
- Example: `service.domain.tld-client` supports TLS 1.2+ and modern ciphers, but also some legacy support for IE 11.

**Server Profile** (certificate, protocols, ciphers *facing your backends*):
- Determines what TLS versions you'll use to backends.
- Determines which ciphers you'll accept from backends.
- Usually more restrictive (you control backends, so enforce strict standards).
- Example: `service.domain.tld-server` requires TLS 1.3, no deprecated ciphers.

**Both are required** if you're terminating and re-establishing SSL to backends.

### Cipher String Rationale

F5 cipher configurations come as pre-built strings or custom concatenations. Understand what you're enabling.

**Mozilla "Intermediate" (reasonable default)**:
- Supports TLS 1.2+
- Doesn't support legacy browsers (IE 8/9/10)
- Doesn't require exotic ciphers
- Well-tested across the internet

**Mozilla "Modern"** (strict):
- TLS 1.3+ only
- Requires modern clients
- Smallest attack surface
- Use if your client base allows it (usually internal APIs, not public web)

**Mozilla "Old"** (compatibility):
- Supports ancient browsers and devices
- Should be rare in 2026
- If you're using this, understand what legacy clients need it and whether you've tested it

**Custom ciphers** (often wrong):
```
AES256-GCM-SHA384:AES128-GCM-SHA256:...
```
Custom ciphers *feel* optimized, but in practice they're often misconfigurations. Stick with Mozilla's tested suites unless your security team has a specific requirement.

### Standard Cipher Policy Object vs. Per-Virtual-Server Settings

**Single policy object** (preferred):
- Create `corporate-ssl-profile-client` and `corporate-ssl-profile-server`.
- Apply to all virtual servers.
- Change corporate cipher policy in one place; applies everywhere.
- Consistency is automatic.

**Per-virtual-server custom settings** (red flag):
- One VS uses `Mozilla Modern`, another uses custom ciphers, a third is missing TLS 1.3.
- Operational nightmare.
- Security review becomes 50-item checklist.
- Hard to change.

**The right answer**: One client profile, one server profile. Exceptions are documented and justified.

### Default Profiles: The Hidden Risk

F5 ships with default `/Common/clientssl` and `/Common/serverssl` profiles. They are:
- Backward-compatibility focused
- Not optimal for modern deployments
- Often missing recent TLS versions
- Fine for testing, not for production

**Never modify the default profiles.** Create your own.

Why it matters:
- Every virtual server you create has access to your customizations
- Future virtual servers inherit them too
- When you need to change requirements, entire deployments automatically change
- You have no way back—the default isn't default anymore

Example of the lesson learned the hard way:
1. Shop modifies `/Common/clientssl` to remove TLS 1.0 (good security decision).
2. Six months later, a legacy partner needs to integrate and requires TLS 1.0.
3. They ask: "Just enable it on our VS."
4. You can't—it's in the default profile, affecting everything.
5. You spend a week un-tangling which virtual servers actually need TLS 1.0 vs. which were just inheriting the broken default.

**Create named profiles** for every deployment. Keep defaults untouched.

---

## Summary

This is a living document. Configuration standards should evolve as your infrastructure grows, as threats change, and as you learn from production incidents. Document why each standard exists, not just what it is. "We don't use Least Connections" is a rule; "We don't use Least Connections because our backend processing times vary by 100x and it created hot spots that took three hours to debug" is a standard that your team will remember and respect.

---

## Repository Structure

As this grows, separate detailed sections into their own files:

```
/docs
  /naming-conventions.md
  /load-balancing-algorithms.md
  /health-monitors.md
  /persistence-profiles.md
  /irules.md
  /ssl-profiles.md
  /examples/
    /pool-examples.md
    /virtual-server-examples.md
```

For now, this README is the canonical source. Link to it, reference it, challenge it, and update it when you find a better way.
