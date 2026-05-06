# Practical 6 – Part A: Securing Redis
### DBS302 – NoSQL Database Management

---

## Table of Contents
- [Aim](#aim)
- [Theory](#theory)
- [Step 0: Confirm Redis is Running](#step-0-confirm-redis-is-running)
- [Step 1: Enable ACL and Create Users](#step-1-enable-acl-and-create-users)
  - [What is ACL?](#what-is-acl)
  - [Default User Behavior](#default-user-behavior)
  - [Configure ACL Users in redis.conf](#configure-acl-users-in-redisconf)
  - [Testing ACL Users](#testing-acl-users)
- [Step 2: Enable TLS Encryption](#step-2-enable-tls-encryption)
  - [Generate Self-Signed Certificates](#generate-self-signed-certificates)
  - [Update redis.conf for TLS](#update-redisconf-for-tls)
  - [Connect Using TLS](#connect-using-tls)
- [Observations](#observations)
- [Conclusion](#conclusion)

---

## Aim

To configure and verify **authentication**, **role-based access control (RBAC)**, and **TLS encryption** for a Redis instance, and to demonstrate that access restrictions are correctly enforced per user.

---

## Theory

Before configuring Redis, it is important to understand the three core security mechanisms applied in this practical:

| Concept | Description | Example |
|--------|-------------|---------|
| **Authentication** | Verifies the identity of a client connecting to the database | Username and password login |
| **RBAC (Role-Based Access Control)** | Grants each user only the minimum permissions they need | An `app_user` can only access `session:*` keys |
| **TLS Encryption** | Encrypts data travelling between the client and the server | Like HTTPS, prevents eavesdropping on the network |

Redis implements RBAC through its **Access Control List (ACL)** system, which allows administrators to define per-user passwords, key patterns, and command permissions.

---

## Step 0: Confirm Redis is Running

Check the installed Redis version:

```bash
redis-server --version
```

![Redis version check](./images/1.png)

Start the Redis server:

```bash
redis-server
```

![Redis server starting](./images/2.png)

Open a new terminal and confirm Redis is accepting connections:

```bash
redis-cli
ping
```

**Expected output:** `PONG`

![Redis ping test](./images/3.png)

---

## Step 1: Enable ACL and Create Users

### What is ACL?

An **Access Control List (ACL)** is a set of rules that defines who can access a resource and what actions they are permitted to perform.

In Redis, ACL works as a security layer on top of the database. When a client connects, it must authenticate with a **username and password**. Once authenticated, that client can only execute commands and access keys that are explicitly allowed for their user account.

### Default User Behavior

Run the following command to see existing users:

```bash
ACL USERS
```

![Default ACL users](./images/4.png)

By default, Redis has a built-in `default` user that allows **all connections without a password**. This means any client that can reach Redis can read and write any data — a significant security risk. Disabling or restricting this default user is the first step toward securing Redis.

### Configure ACL Users in redis.conf

ACL users can be defined either via the command line or directly in `redis.conf`. Using the configuration file is preferred for persistent setup across restarts.

Open the Redis configuration file:

```bash
sudo nano /etc/redis/redis.conf
```

Add the following ACL definitions in the `SECURITY` section (or at the end of the file):

```conf
# Disable the anonymous default user
user default off

# Admin user: full access (DBA / Instructor only)
user admin on >adminStrongPwd ~* +@all

# Application user: can only read/write keys starting with "session:"
user app_user on >appStrongPwd ~session:* +get +set +del +expire +ttl +@connection

# Read-only monitoring user: can read all keys and run INFO
user monitoring on >monitorPwd ~* +@read +info +dbsize +lastsave +@connection
```

![ACL configuration in redis.conf](./images/5.png)

**Understanding the ACL syntax:**

| Symbol | Meaning |
|--------|---------|
| `on` | The user account is active |
| `>password` | Sets the user's login password |
| `~pattern` | Key name patterns the user can access (`~*` means all keys) |
| `+command` | Allows a specific command (e.g., `+get`) |
| `+@category` | Allows all commands in a category (e.g., `+@read`, `+@all`) |

Save the file and restart Redis to apply the changes:

```bash
sudo systemctl restart redis
```

![Redis restarted](./images/6.png)

**After this configuration:**
- The `default` user is **disabled** — anonymous connections are blocked
- `admin` has **full access** to all commands and keys
- `app_user` can only access keys matching `session:*` and run a limited set of commands
- `monitoring` can read all keys and run monitoring commands, but **cannot modify any data**

---

### Testing ACL Users

Since the `default` user is now disabled, all connections must authenticate. Attempting to connect without credentials will be rejected:

![Anonymous connection rejected](./images/7.png)

---

#### Testing the `admin` User

Connect as `admin`:

```bash
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379
```

![Admin connection](./images/8.png)

Verify identity and test full access:

```bash
ACL WHOAMI
set mykey "hello"
get mykey
```

![Admin commands](./images/9.png)

**Observation:** The `admin` user can set and get any key without restriction. ✅

---

#### Testing the `app_user`

Connect as `app_user`:

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

![app_user connection](./images/10.png)

Check identity:

```bash
ACL WHOAMI
```

![app_user whoami](./images/11.png)

> **Note:** `ACL WHOAMI` fails for `app_user` because the user is only permitted to run `get`, `set`, `del`, `expire`, `ttl`, and connection commands. The `ACL` command category is not included in their permissions — this is expected and demonstrates RBAC working correctly.

Test key access:

```bash
set session:user123 "data"
get session:user123
set otherkey "oops"
```

![app_user key access test](./images/12.png)

**Observation:**
- `set session:user123 "data"` → **OK** ✅ (key matches `session:*` pattern)
- `get session:user123` → **returns value** ✅
- `set otherkey "oops"` → **NOPERM error** ✅ (key does not match `session:*`)

![app_user NOPERM error](./images/13.png)

This confirms that RBAC is correctly enforced — `app_user` is confined to their designated key pattern.

---

## Step 2: Enable TLS Encryption

TLS (Transport Layer Security) encrypts all data transmitted between Redis and its clients, preventing attackers from reading or tampering with data on the network.

### Generate Self-Signed Certificates

Create a directory for the TLS certificates and generate them using OpenSSL:

```bash
mkdir -p /etc/redis/tls
cd /etc/redis/tls

# Step 1: Create a private key for the Certificate Authority (CA)
openssl genrsa -out ca.key 4096

# Step 2: Create a self-signed CA certificate
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 \
  -out ca.crt \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=redis-lab-ca"

# Step 3: Create a private key for the Redis server
openssl genrsa -out redis.key 4096

# Step 4: Create a Certificate Signing Request (CSR) for the Redis server
openssl req -new -key redis.key -out redis.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"

# Step 5: Sign the server certificate with the CA
openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out redis.crt -days 365 -sha256
```

**Files created in `/etc/redis/tls/`:**

| File | Purpose |
|------|---------|
| `ca.key` | Certificate Authority private key |
| `ca.crt` | Certificate Authority certificate (used by clients to verify Redis) |
| `redis.key` | Redis server private key |
| `redis.crt` | Redis server certificate (signed by the CA) |

### Update redis.conf for TLS

Add the following lines at the end of `redis.conf`:

```conf
# Disable plain TCP (no unencrypted connections)
port 0

# Enable TLS on port 6379
tls-port 6379

# TLS certificate files
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key

# Do not require clients to present their own certificate
tls-auth-clients no
```

Restart Redis with the updated configuration:

```bash
redis-server /etc/redis/redis.conf
```

![Redis restarted with TLS](./images/14.png)

### Connect Using TLS

Use `redis-cli` with TLS flags and the `rediss://` scheme (double `s` indicates TLS):

```bash
redis-cli --tls \
  --cacert /etc/redis/tls/ca.crt \
  -u rediss://app_user:appStrongPwd@127.0.0.1:6379
```

Inside the Redis CLI, test the connection:

```bash
whoami
set session:user456 "secure"
get session:user456
```

![TLS connection and commands](./images/15.png)

**Observation:** The connection succeeds over TLS, and `app_user` can still only access `session:*` keys — both encryption and RBAC are enforced simultaneously. ✅

---

## Observations

| Test | Command | Expected Result | Result |
|------|---------|----------------|--------|
| Anonymous connection | `redis-cli ping` | NOAUTH error | ✅ Blocked |
| Admin sets any key | `set mykey "hello"` | OK | ✅ Allowed |
| app_user sets session key | `set session:user123 "data"` | OK | ✅ Allowed |
| app_user sets non-session key | `set otherkey "oops"` | NOPERM error | ✅ Blocked |
| app_user runs ACL WHOAMI | `ACL WHOAMI` | NOPERM error | ✅ Blocked |
| TLS connection with correct CA | `redis-cli --tls --cacert ...` | Connected | ✅ Connected |
| Plain TCP connection (no TLS) | `redis-cli` without `--tls` | Connection refused | ✅ Blocked |

---

## Conclusion

This practical demonstrated how to secure a Redis instance using three complementary security mechanisms. By disabling the default user and defining ACL-based users, Redis now requires authentication for every connection. The per-user key pattern and command restrictions enforce the **principle of least privilege**, ensuring that each user can only perform operations relevant to their role. Enabling TLS further protects data in transit, preventing any network-level eavesdropping. Together, these three layers — **authentication, RBAC, and encryption** — form a strong security foundation for Redis in a real-world deployment.
