# Practical 6: Securing Redis and MongoDB
### DBS302 – NoSQL Database Management | Complete Step-by-Step Guide

---

## Table of Contents

1. [Overview & Aim](#overview)
2. [Pre-Requisites & Setup](#prerequisites)
3. [Part A – Securing Redis](#part-a-redis)
   - A1. Verify Redis is Running
   - A2. Enable ACL Users and Permissions
   - A3. Test ACL (RBAC-like behavior)
   - A4. Enable TLS Encryption (Optional but Recommended)
   - A5. Python Demo Script (Optional)
4. [Part B – Securing MongoDB](#part-b-mongodb)
   - B1. Start MongoDB Without Auth (Initial Setup)
   - B2. Create the First Admin User
   - B3. Enable Authentication in mongod.conf
   - B4. Test Authentication
   - B5. Create Application Role and User (RBAC)
   - B6. Enable TLS Encryption
   - B7. Node.js Demo Script (Optional)
5. [Part C – Security Audit](#part-c-audit)
   - C1. Redis Security Audit Checklist
   - C2. MongoDB Security Audit Checklist
6. [Lab Report Structure](#report)
7. [Common Mistakes to Avoid](#mistakes)
8. [Quick Revision Notes](#revision)

---

## 1. Overview & Aim {#overview}

This practical teaches you to **secure Redis and MongoDB** using three core security mechanisms:

| Mechanism | What it does | Example |
|-----------|-------------|---------|
| **Authentication** | Verifies who the client is | Username + Password login |
| **Encryption (TLS)** | Protects data traveling over the network | Like HTTPS for databases |
| **RBAC** | Grants only the minimum needed permissions | A "reader" cannot delete data |

**By end of practical, you will be able to:**
- Enable password-based ACL authentication in Redis
- Enable TLS encryption for Redis connections
- Enable authentication and RBAC in MongoDB
- Enable TLS for MongoDB
- Perform a basic security audit for both databases

---

## 2. Pre-Requisites & Setup {#prerequisites}

### Software Required

- **OS**: Ubuntu Linux (or WSL2 on Windows)
- **Redis** 7.x or later
- **MongoDB Community Server** 7.0 or similar
- **OpenSSL** (for generating certificates)
- **Python 3** (for optional Redis demo)
- **Node.js** (for optional MongoDB demo)

### Verify your installations

```bash
# Check Redis
redis-server --version

# Check MongoDB
mongod --version

# Check OpenSSL
openssl version
```

> **Tip:** If Redis or MongoDB is not installed, ask your lab instructor for the pre-configured lab machine access.

---

## 3. Part A – Securing Redis {#part-a-redis}

### A1. Step 0 – Verify Redis is Running

Open **Terminal 1** and start Redis:

```bash
redis-server
```

Open **Terminal 2** and test the connection:

```bash
redis-cli
ping
```

**Expected output:**
```
PONG
```

If you see `PONG`, Redis is running. If your lab uses a config file:

```bash
redis-server /etc/redis/redis.conf
```

---

### A2. Step 1 – Enable ACL Users in `redis.conf`

ACL (Access Control List) lets you create named users with specific passwords, key access patterns, and command permissions.

#### Open the Redis config file:

```bash
sudo nano /etc/redis/redis.conf
```

#### Add the following ACL user definitions:

```conf
# Disable the default anonymous user (security best practice)
user default off

# Admin user: full access (DBA / Instructor only)
user admin on >adminStrongPwd ~* +@all

# Application user: can only read/write keys starting with "session:"
user app_user on >appStrongPwd ~session:* +get +set +del +expire +ttl +@connection

# Read-only monitoring user: can read all keys and run INFO
user monitoring on >monitorPwd ~* +@read +info +dbsize +lastsave +@connection
```

**Understanding the ACL syntax:**

| Symbol | Meaning |
|--------|---------|
| `on` | User is active/enabled |
| `>password` | Sets the user's password |
| `~pattern` | Key patterns the user can access (`~*` = all keys) |
| `+command` | Allow this specific command |
| `+@category` | Allow all commands in this category (e.g., `@read`, `@all`) |

#### Save and restart Redis:

```bash
sudo systemctl restart redis-server
# OR (manual):
pkill redis-server
redis-server /etc/redis/redis.conf
```

---

### A3. Step 2 – Test ACL Users (RBAC Behavior)

#### Connect as `admin` (full access):

```bash
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379
```

Inside `redis-cli`, run:

```bash
whoami
set mykey "hello"
get mykey
```

**Expected output:**
```
"admin"
OK
"hello"
```

#### Connect as `app_user` (limited access):

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

Inside `redis-cli`, run:

```bash
whoami
set session:user123 "data"
get session:user123
set otherkey "oops"
```

**Expected output (observe the permission denial):**
```
"app_user"
OK
"data"
(error) NOPERM this user has no permissions to access one of the keys used as arguments
```

The error on `set otherkey` confirms RBAC is working — `app_user` can only touch keys matching `session:*`.

#### Connect as `monitoring` (read-only):

```bash
redis-cli -u redis://monitoring:monitorPwd@127.0.0.1:6379
```

Inside `redis-cli`, run:

```bash
whoami
info server
set somekey "attempt"
```

**Expected output:**
```
"monitoring"
# Server
redis_version:7.x.x
...
(error) NOPERM this user has no permissions to run the 'set' command
```

#### View all ACL users:

```bash
# As admin:
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379 ACL LIST
```

Record all outputs in your **Observation Table** for the lab report.

---

### A4. Step 3 – Enable TLS Encryption for Redis

> **Note:** This section is optional but strongly recommended for full marks. TLS ensures all data between the client and Redis is encrypted.

#### A4.1 – Generate Self-Signed Certificates

Run these commands to create the TLS certificates:

```bash
# Create a directory for the certificates
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

# Step 4: Create a Certificate Signing Request (CSR) for Redis
openssl req -new -key redis.key -out redis.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"

# Step 5: Sign the Redis server certificate with the CA
openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out redis.crt -days 365 -sha256
```

**Files created:**
- `ca.crt` – Certificate Authority certificate
- `redis.crt` – Redis server certificate
- `redis.key` – Redis server private key

#### A4.2 – Update `redis.conf` for TLS

Open the config file and add/edit these lines:

```conf
# Disable plain TCP port (use 0 to disable, or keep 6379 for demo)
port 0

# Enable TLS on port 6379
tls-port 6379

# TLS certificate files
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key

# Require clients to use TLS
tls-auth-clients yes
```

Restart Redis:

```bash
redis-server /etc/redis/redis.conf
```

#### A4.3 – Connect Using TLS

```bash
redis-cli --tls \
  --cacert /etc/redis/tls/ca.crt \
  -u rediss://app_user:appStrongPwd@127.0.0.1:6379
```

> **Note:** `rediss://` (with double `ss`) indicates a TLS connection.

Inside `redis-cli`, test:

```bash
whoami
set session:user456 "secure"
get session:user456
```

**Expected output:**
```
"app_user"
OK
"secure"
```

Also verify that a non-TLS connection **fails**:

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

This should return a connection error — record this in your audit report.

---

### A5. (Optional) Python Application Demo for Redis

Create a file `redis_secure_demo.py`:

```python
# file: redis_secure_demo.py
import redis
import ssl

def create_redis_client():
    # Create SSL context for TLS
    ssl_ctx = ssl.create_default_context(
        purpose=ssl.Purpose.SERVER_AUTH,
        cafile="/etc/redis/tls/ca.crt",
    )
    ssl_ctx.check_hostname = False   # for lab demo only
    ssl_ctx.verify_mode = ssl.CERT_REQUIRED

    # Connect as app_user over TLS
    client = redis.Redis(
        host="127.0.0.1",
        port=6379,
        username="app_user",
        password="appStrongPwd",
        ssl=True,
        ssl_context=ssl_ctx,
        decode_responses=True,
    )
    return client

if __name__ == "__main__":
    r = create_redis_client()
    print("Connected as:", r.acl_whoami())
    r.set("session:python_demo", "hello from python")
    print("Value:", r.get("session:python_demo"))
```

Install the Redis Python client and run:

```bash
pip install redis
python3 redis_secure_demo.py
```

**Expected output:**
```
Connected as: app_user
Value: hello from python
```

---

## 4. Part B – Securing MongoDB {#part-b-mongodb}

### B1. Step 0 – Start MongoDB Without Auth (Initial Setup Only)

> **Important:** This step is only for the very first setup to create the admin user. After that, auth will always be required.

**Terminal 1** – Start MongoDB without authentication:

```bash
mongod --dbpath /data/db --bind_ip 127.0.0.1 --port 27017
```

> If `/data/db` does not exist: `sudo mkdir -p /data/db && sudo chown $USER /data/db`

**Terminal 2** – Connect using mongosh:

```bash
mongosh --host 127.0.0.1 --port 27017
```

---

### B2. Step 1 – Create the First Admin User

Inside `mongosh`, run:

```javascript
// Switch to admin database
use admin;

// Create an admin user with full privileges
db.createUser({
  user: "rootAdmin",
  pwd: "rootStrongPwd",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
});
```

**Expected output:**
```
{ ok: 1 }
```

Exit mongosh:

```javascript
exit
```

Stop the MongoDB instance (Ctrl+C in Terminal 1).

---

### B3. Step 2 – Enable Authentication in `mongod.conf`

Open the MongoDB config file:

```bash
sudo nano /etc/mongod.conf
```

Add or edit the security section:

```yaml
security:
  authorization: "enabled"
```

Restart MongoDB:

```bash
sudo systemctl restart mongod
# OR manually:
mongod --config /etc/mongod.conf
```

---

### B4. Step 3 – Test Authentication

#### Connect WITH credentials (should succeed):

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u rootAdmin -p rootStrongPwd \
  --authenticationDatabase admin
```

Inside mongosh, verify:

```javascript
db.runCommand({ connectionStatus: 1 });
```

Look for the `authenticatedUsers` field in the output — it should show `rootAdmin` with the assigned roles.

#### Connect WITHOUT credentials (should fail):

```bash
mongosh --host 127.0.0.1 --port 27017
```

Inside mongosh, run:

```javascript
show dbs;
```

**Expected output:**
```
MongoServerError: Command listDatabases requires authentication
```

Record this result — it confirms authentication is enforced. ✅

---

### B5. Step 4 – Create Application Role and User (RBAC)

Reconnect as `rootAdmin`:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u rootAdmin -p rootStrongPwd \
  --authenticationDatabase admin
```

Inside mongosh, run:

```javascript
// Switch to the application database
use myapp;

// Step 1: Create a custom limited role
db.runCommand({
  createRole: "myAppRole",
  privileges: [
    {
      resource: { db: "myapp", collection: "customers" },
      actions: ["find", "insert", "update", "remove"]
    }
  ],
  roles: []  // no inherited roles - principle of least privilege
});

// Step 2: Create an application user with this limited role
db.createUser({
  user: "appUser",
  pwd: "appStrongPwd",
  roles: [
    { role: "myAppRole", db: "myapp" }
  ]
});
```

#### Test `appUser` Permissions:

Open a new terminal and connect as `appUser`:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

Inside mongosh, run these tests:

```javascript
// Test 1: Insert into allowed collection (should SUCCEED)
use myapp;
db.customers.insertOne({ name: "Student One", city: "Phuntsholing" });

// Test 2: Query the allowed collection (should SUCCEED)
db.customers.find();

// Test 3: Access admin database (should FAIL)
use admin;
db.system.users.find();
```

**Expected output for Test 3:**
```
MongoServerError: not authorized on admin to execute command
```

This confirms RBAC is working — `appUser` is confined to `myapp.customers` only. ✅

#### Verify user roles (as rootAdmin):

```javascript
use admin;
db.system.users.find({ user: "appUser" }).pretty();
```

This shows the exact roles and privileges assigned to `appUser`.

---

### B6. Step 5 – Enable TLS Encryption for MongoDB

#### B6.1 – Generate Self-Signed Certificates

```bash
# Create certificate directory
mkdir -p /etc/mongo/tls
cd /etc/mongo/tls

# Step 1: CA key and certificate
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 \
  -out ca.pem \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=mongo-lab-ca"

# Step 2: MongoDB server key and certificate
openssl genrsa -out mongo.key 4096
openssl req -new -key mongo.key -out mongo.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"
openssl x509 -req -in mongo.csr -CA ca.pem -CAkey ca.key -CAcreateserial \
  -out mongo.crt -days 365 -sha256

# Step 3: Combine key and cert into a single PEM file (required by MongoDB)
cat mongo.key mongo.crt > mongo.pem
```

**Files created:**
- `ca.pem` – Certificate Authority
- `mongo.pem` – MongoDB server certificate + key (combined)

#### B6.2 – Update `mongod.conf` for TLS

Edit `/etc/mongod.conf`:

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/tls/mongo.pem
    CAFile: /etc/mongo/tls/ca.pem
    allowConnectionsWithoutCertificates: true  # for lab; set false in production

security:
  authorization: "enabled"
```

Restart MongoDB:

```bash
sudo systemctl restart mongod
```

#### B6.3 – Connect WITH TLS and Auth

```bash
mongosh \
  --host 127.0.0.1 \
  --port 27017 \
  --tls \
  --tlsCAFile /etc/mongo/tls/ca.pem \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

Inside mongosh, verify it works:

```javascript
use myapp;
db.customers.insertOne({ name: "TLS Test", city: "Thimphu" });
db.customers.find();
```

#### Verify TLS is enforced (try without `--tls`):

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

**Expected:** Connection error (TLS required). Record this in your audit. ✅

---

### B7. (Optional) Node.js Application Demo for MongoDB

Create a file `mongo_secure_demo.js`:

```javascript
// file: mongo_secure_demo.js
const { MongoClient } = require("mongodb");

async function main() {
  const uri = "mongodb://appUser:appStrongPwd@127.0.0.1:27017/myapp?tls=true";

  const client = new MongoClient(uri, {
    tlsCAFile: "/etc/mongo/tls/ca.pem",
  });

  try {
    await client.connect();
    console.log("Connected to MongoDB with TLS and Auth");

    const db = client.db("myapp");
    const customers = db.collection("customers");

    await customers.insertOne({ name: "Node Client", city: "Phuntsholing" });

    const docs = await customers.find({}).toArray();
    console.log("Customers:", docs);
  } finally {
    await client.close();
  }
}

main().catch(console.error);
```

Install and run:

```bash
npm install mongodb
node mongo_secure_demo.js
```

**Expected output:**
```
Connected to MongoDB with TLS and Auth
Customers: [ { _id: ..., name: 'Node Client', city: 'Phuntsholing' }, ... ]
```

---

## 5. Part C – Security Audit {#part-c-audit}

### C1. Redis Security Audit Checklist

Use this checklist and document your results for the lab report:

| # | Check | Command | Pass / Fail | Observation |
|---|-------|---------|-------------|-------------|
| 1 | Anonymous connection denied | Connect without credentials; try `INFO` | ✅ / ❌ | |
| 2 | `app_user` denied on wrong key | `set otherkey "x"` as `app_user` | ✅ / ❌ | |
| 3 | `monitoring` denied write | `set k v` as `monitoring` | ✅ / ❌ | |
| 4 | Admin commands restricted for non-admin | `FLUSHALL` as `app_user` | ✅ / ❌ | |
| 5 | ACL list shows all users | `ACL LIST` as admin | ✅ / ❌ | |
| 6 | TLS connection succeeds | `redis-cli --tls ...` | ✅ / ❌ | |
| 7 | Non-TLS connection fails | `redis-cli` without `--tls` | ✅ / ❌ | |

**Commands for audit:**

```bash
# Check 1: Anonymous connection
redis-cli -h 127.0.0.1 ping
# Should fail if default user is off

# Check 5: View all ACL users
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379 ACL LIST

# Check 4: Try FLUSHALL as app_user
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379 FLUSHALL
```

---

### C2. MongoDB Security Audit Checklist

| # | Check | Command | Pass / Fail | Observation |
|---|-------|---------|-------------|-------------|
| 1 | Auth enforced (no credentials) | `mongosh` without `-u`; try `show dbs` | ✅ / ❌ | |
| 2 | `appUser` confined to `myapp.customers` | Access `admin` db as `appUser` | ✅ / ❌ | |
| 3 | `appUser` roles visible | `db.system.users.find({user:"appUser"})` as admin | ✅ / ❌ | |
| 4 | TLS connection succeeds | `mongosh --tls ...` | ✅ / ❌ | |
| 5 | Non-TLS connection fails | `mongosh` without `--tls` | ✅ / ❌ | |
| 6 | Network exposure noted | Review `bindIp` in `mongod.conf` | Note | |

---

## 6. Lab Report Structure {#report}

Your lab report should follow this structure:

### 1. Aim and Objectives
Copy and adapt from Section 2 of this guide.

### 2. Theory (Short, 2–3 paragraphs)
- Explain **Authentication** in Redis and MongoDB
- Explain **Encryption (TLS)** and why it matters
- Explain **RBAC** and the principle of least privilege

### 3. Procedure
Step-by-step commands you executed, organized by section (Part A, B, C). Include screenshots or console logs where possible.

### 4. Observations Table

For Redis:

| Test | Command Run | Expected Result | Actual Result |
|------|-------------|----------------|---------------|
| app_user sets session key | `set session:x "y"` | OK | OK |
| app_user sets other key | `set otherkey "y"` | NOPERM error | NOPERM error |
| monitoring writes | `set k v` | NOPERM error | NOPERM error |
| TLS connection | `redis-cli --tls ...` | Connected | Connected |
| Non-TLS connection | `redis-cli ...` | Failed | Failed |

For MongoDB:

| Test | Command Run | Expected Result | Actual Result |
|------|-------------|----------------|---------------|
| No-auth connection | `mongosh` without creds | Auth error | Auth error |
| appUser insert customers | `db.customers.insertOne(...)` | Inserted | Inserted |
| appUser access admin | `use admin; db.system.users.find()` | Auth error | Auth error |
| TLS connection | `mongosh --tls ...` | Connected | Connected |
| Non-TLS connection | `mongosh` without `--tls` | Failed | Failed |

### 5. Security Audit Summary

**What is now secure:**
- Redis requires username and password (ACL enabled)
- Redis users are limited by key patterns and allowed commands
- MongoDB authentication is enforced for all clients
- MongoDB users have minimal roles (principle of least privilege)
- TLS encrypts all data in transit for both databases

**What still needs improvement:**
- `bindIp` is set to `0.0.0.0` in the lab; in production, restrict to trusted IPs
- Self-signed certificates used; production should use CA-signed certificates
- Password strength could be improved (consider using a password manager)
- Audit logging is not yet enabled (MongoDB has an `auditLog` option for Enterprise)
- No automated intrusion detection is set up

### 6. Conclusion (3–4 lines)
Reflect on how authentication prevents unauthorized access, TLS prevents eavesdropping, and RBAC limits damage from compromised credentials. These three layers together provide defense-in-depth for NoSQL databases.

---

## 7. Common Mistakes to Avoid {#mistakes}

| Mistake | Why it's a problem | How to avoid it |
|---------|-------------------|----------------|
| Using weak passwords like `1234` | Easily guessed by attackers | Use 12+ char passwords with letters, numbers, symbols |
| Forgetting to restart services | Config changes don't apply | Always restart and then re-test |
| Leaving `bindIp: 0.0.0.0` in production | Exposes DB to the entire internet | Restrict to localhost or internal IP |
| Giving `readWriteAnyDatabase` to app users | Violates least privilege | Create focused roles per database/collection |
| Only testing successful operations | Doesn't verify security | Always test denied cases too |

---

## 8. Quick Revision Notes {#revision}

### Redis Security Summary

```
Authentication  → ACL users with username + password
Key Restriction → ~pattern limits which keys a user can touch
Command Limit   → +@category or +command restricts what user can do
TLS             → tls-port + tls-cert-file + tls-key-file in redis.conf
Test RBAC       → Try forbidden key/command; expect NOPERM error
```

### MongoDB Security Summary

```
Authentication  → security.authorization: "enabled" in mongod.conf
First user      → Create rootAdmin BEFORE enabling auth
RBAC            → createRole + createUser with specific privileges
TLS             → net.tls.mode: requireTLS + certificateKeyFile in mongod.conf
Test auth       → Connect without creds; expect authorization error
Test RBAC       → appUser tries admin db; expect auth error
Test TLS        → Connect without --tls; expect connection error
```

### Security Audit Summary

Always test **both** the successful case AND the denied case. Security is only confirmed when:
- Authorized access **works**
- Unauthorized access **fails**

---

*Guide compiled for DBS302 – NoSQL Database Management, Practical 6*
*Bhutan | Phuntsholing | Lab Environment: Ubuntu Linux*
