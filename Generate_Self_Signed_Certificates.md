
# 🛡️ Create Your Own Certificate Authority (CA) and SSL Certificates for Local HTTPS

## 🧠 Understanding SSL/TLS and Certificates

### 📌 What is SSL/TLS?
SSL (Secure Sockets Layer) and TLS (Transport Layer Security) are cryptographic protocols that provide secure communication over a network.

### 📌 What is a Certificate Authority (CA)?
A Certificate Authority is a trusted entity that issues digital certificates. These certificates verify that a public key belongs to an individual, organization, or device.

### 📌 Self-Signed vs CA-Signed Certificates
- **Self-Signed Certificate**: Signed with your own private key. Suitable for development.
- **CA-Signed Certificate**: Signed by a trusted CA. Suitable for production environments.

### 📌 Diagram: SSL Certificate Flow (Client ↔️ Server Communication)
![SSL Padlock](images/ssl-workflow.png)

## ⚙️ Prerequisites

1. **OpenSSL** must be installed:
   - [Install OpenSSL (StackOverflow guide)](https://stackoverflow.com/a/51757939/1273882)
   - On Windows, add `C:\Program Files\Git\usr\bin` to your PATH environment variable

---

## 🔐 Key Concepts

### 📁 File Types
- **`.key`**: Contains the private key (used to sign or decrypt data).
- **`.csr`**: Certificate Signing Request (contains public key + metadata).
- **`.crt` / `.pem`**: Final certificate, signed using CA.
- **`.pfx`**: Combined file (Private Key + Certificate), usually for web servers.

### 🛠️ Useful Commands
| Command | Purpose |
|--------|---------|
| `openssl genrsa` | Generate RSA private key |
| `openssl rsa -pubout` | Extract public key from private key |
| `openssl req` | Generate CSR or self-signed cert |
| `openssl x509` | Sign CSR using CA private key |
| `openssl pkcs12` | Create a PFX bundle |

![SSL Padlock](images/key_component.png)

---

## 🏗️ Step-by-Step Guide to Set Up Local Certificate Authority

### ✅ Step 1: Create Your Own Root Certificate Authority (CA)

1. **Set a name for your CA:**
```bash
# Linux
CANAME=MyOrg-RootCA

# Windows
set CANAME=MyOrg-RootCA
```

2. **Generate a private key for the CA:**
```bash
openssl genrsa -aes256 -out $CANAME.key 4096
```
> 💡 This will ask you for a passphrase to secure your private key.

3. **Create a Root Certificate (self-signed):**
```bash
openssl req -x509 -new -nodes -key $CANAME.key -sha256 -days 1826 -out $CANAME.crt
```
> 📌 `-x509` creates a self-signed cert, valid for ~5 years.

### 🧩 Add the Root Certificate to Trusted Store

#### ✅ Windows
1. Double-click `$CANAME.crt`
2. Click **Install Certificate**
3. Select **Local Machine** → **Place all certificates in** → **Trusted Root Certification Authorities**
4. Verify with `certmgr.msc`

#### ✅ Ubuntu Linux
```bash
sudo apt install -y ca-certificates
sudo cp $CANAME.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

#### ✅ CentOS/Fedora
```bash
sudo cp $CANAME.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

![napkin](https://napkin.ai/api/v1/c/yErFy0uGDX?style=sketch)

---

### ✅ Step 2: Create Server SSL Certificate Signed by Your CA

1. **Set a name for your server certificate:**
```bash
# Linux
MYCERT=MyServer

# Windows
set MYCERT=MyServer
```

2. **Generate a Private Key + CSR (Certificate Signing Request):**
```bash
openssl req -new -nodes -out $MYCERT.csr -newkey rsa:4096 -keyout $MYCERT.key
```
> 📌 This creates a new key + CSR containing public key + metadata.

3. **Create SAN (Subject Alternative Name) file:**
```ini
# Save as MyServer.v3.ext

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = myserver.local
DNS.2 = myserver1.local
IP.1 = 192.168.1.1
IP.2 = 192.168.2.1
```
> ✅ SAN is required by modern browsers.

4. **Sign the CSR using your CA’s key:**
```bash
openssl x509 -req -in $MYCERT.csr -CA $CANAME.crt -CAkey $CANAME.key \
  -CAcreateserial -out $MYCERT.crt -days 730 -sha256 -extfile $MYCERT.v3.ext
```
> 🔐 This produces a `.crt` file signed by your root CA.

![napkin](https://napkin.ai/api/v1/c/M7eTKNB7Or?style=sketch)

---

### ✅ Step 3: Run a Local Website with HTTPS (ASP.NET Example)

1. **Create PFX bundle (private key + certificate)**
```bash
openssl pkcs12 -export -out $MYCERT.pfx -inkey $MYCERT.key -in $MYCERT.crt
```
> Set a password when prompted. You’ll use it in your server config.

2. **Configure ASP.NET Core Application**
```csharp
builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.ConfigureEndpointDefaults(listenOptions =>
    {
        listenOptions.UseHttps("MyServer.pfx", "Password");
    });
});
```

3. **Test in browser**:
- Navigate to https://myserver.local
- If CA is trusted, the lock icon should appear

---

## 📘 Full Command Summary

```bash
# Create Root CA
openssl genrsa -aes256 -out MyOrg-RootCA.key 4096
openssl req -x509 -new -nodes -key MyOrg-RootCA.key -sha256 -days 1826 -out MyOrg-RootCA.crt

# Generate Server Key + CSR
openssl req -new -nodes -out MyServer.csr -newkey rsa:4096 -keyout MyServer.key

# Create SAN file (MyServer.v3.ext)
# [See example above]

# Sign the CSR
openssl x509 -req -in MyServer.csr -CA MyOrg-RootCA.crt -CAkey MyOrg-RootCA.key \
  -CAcreateserial -out MyServer.crt -days 730 -sha256 -extfile MyServer.v3.ext

# Generate PFX bundle
openssl pkcs12 -export -out MyServer.pfx -inkey MyServer.key -in MyServer.crt
```

---

## 🧪 Example Use Case: Local Development on `https://myserver.local`

1. Add `127.0.0.1 myserver.local` to your `hosts` file
2. Create and trust your Root CA
3. Create and sign SSL cert for `myserver.local`
4. Run your server (ASP.NET, Node.js, etc.) using the cert
5. Test HTTPS connection in the browser

![napkin](https://napkin.ai/api/v1/c/i5hbgZYrQm?style=sketch)

---

## 🔗 References

1. [OpenSSL genrsa](https://www.openssl.org/docs/man1.1.1/man1/openssl-genrsa.html)
2. [OpenSSL req](https://www.openssl.org/docs/man1.0.2/man1/openssl-req.html)
3. [Extract public key from private](https://security.stackexchange.com/a/172277)
