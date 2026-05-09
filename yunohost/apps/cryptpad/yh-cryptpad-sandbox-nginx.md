# CryptPad on YunoHost: Sandbox Domain Fix

**Version:** Mai 2026
**YunoHost:** 12.1.39 (stable)
**Cryptpad:** 2026.2.2 ~ynh1
**Symptoms:** Certificate renewal fails, documents stuck on "Loading", backups fail

---

## Table of Contents

1. [Problem Description](#problem-description)
2. [Root Cause](#root-cause)
3. [Guide for Users](#guide-for-users)
4. [Technical Explanation for Developers](#technical-explanation-for-developers)

---

## Problem Description

After installing CryptPad on YunoHost, the following issues occur:

### Symptom 1: Certificate Renewal Fails

When manually or automatically renewing the Let's Encrypt certificate, the following error appears:

```
Verifying sandbox.cryptpad.example.com...
Wrote file to /var/www/.well-known/acme-challenge-public/..., 
but couldn't download http://sandbox.cryptpad.example.com/.well-known/acme-challenge/...: 
Error: Response Code: None
Certificate renewing for cryptpad.example.com failed!
```

Or with an SSL error:

```
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: 
unable to get local issuer certificate
```

### Symptom 2: Documents Stuck on "Loading"

In the CryptPad interface, opening any document (spreadsheets, presentations, code editor, polls, etc.) results in an endless loading spinner. Only the "Write" editor (plain text) works — all other document types hang at "Loading".

### Symptom 3: Backups Fail

YunoHost backups for the CryptPad app fail because CryptPad services are not fully available.

---

## Root Cause

CryptPad requires a sandbox domain (`sandbox.cryptpad.example.com`) through which all document editors are loaded in an isolated iframe. This isolation is a core security feature of CryptPad.

The YunoHost installation package does create a nginx template file for this sandbox domain (`nginx-sandbox.conf`), but it is **never deployed** as an active nginx configuration. As a result:

- `sandbox.cryptpad.example.com` has **no HTTPS server block** → documents cannot be loaded
- `sandbox.cryptpad.example.com` has **no port 80 server block** with ACME challenge support → Let's Encrypt cannot verify the domain → certificate renewal fails

---

## Guide for Users

### Prerequisites

- Root access to the server (SSH)
- CryptPad is installed on YunoHost
- DNS entry for `sandbox.cryptpad.example.com` exists (CNAME pointing to main domain)

### Step 1: Verify DNS Entry

In your DNS management panel, make sure the following entry exists:

| Host | Type | Target |
|------|------|--------|
| `sandbox.cryptpad` | `CNAME` | `cryptpad.example.com` |

> **Important:** A CNAME record (not an A record) is correct and sufficient — the IP address is automatically resolved from the main domain entry.

Check DNS propagation — wait until an IP address is returned:

```bash
dig sandbox.cryptpad.example.com
```

Expected output (under `ANSWER SECTION`):
```
sandbox.cryptpad.example.com.  300  IN  A  123.456.789.0
```

Once an IP address appears, the DNS entry is active.

---

### Step 2: Retrieve Your Configuration Values

The following values are needed for the configuration. Read them from the YunoHost app settings:

```bash
cat /etc/yunohost/apps/cryptpad/settings.yml | grep -E "domain|port"
```

Example output:
```
domain: cryptpad.example.com
port: 3000
port_socket: 3004
```

Note these three values — they will be used in the following steps.

---

### Step 3: Deploy the Sandbox nginx Configuration

Fill the app's internal template file with the real values and save it as an active nginx configuration:

```bash
sed -e 's/__DOMAIN__/cryptpad.example.com/g' \
    -e 's/__APP__/cryptpad/g' \
    -e 's/__PORT__/3000/g' \
    -e 's/__PORT_SOCKET__/3004/g' \
    /etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf \
    > /etc/nginx/conf.d/sandbox.cryptpad.example.com.conf
```

> **Note:** Replace `cryptpad.example.com`, `3000` and `3004` with your own values from Step 2 if they differ.

---

### Step 4: Test and Reload nginx

Check the nginx configuration syntax:

```bash
nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

> Warnings about `ssl_stapling` and `variables_hash_bucket_size` can be safely ignored — they do not affect functionality.

Reload nginx:

```bash
systemctl reload nginx
```

---

### Step 5: Test HTTPS Accessibility of the Sandbox

```bash
curl -I https://sandbox.cryptpad.example.com
```

Expected output (excerpt):
```
HTTP/2 200
server: nginx
content-type: text/html; charset=UTF-8
```

An `HTTP/2 200` confirms that the sandbox domain is correctly reachable.

---

### Step 6: Renew the Certificate

Since the certificate has not yet expired, the renewal must be forced:

```bash
yunohost domain cert renew cryptpad.example.com --force
```

Expected output at the end:
```
Success! Let's Encrypt certificate renewed for the domain 'cryptpad.example.com'
```

---

### Step 7: Make the Fix Permanent with a Hook

YunoHost overwrites nginx configurations during updates and `regen-conf` calls, which would delete the file created in Step 3. To prevent this, create a hook that automatically recreates the file after every `regen-conf`:

```bash
nano /usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad
```

Insert the following content:

```bash
#!/usr/bin/env bash
set -e

do_pre_regen() { true; }
do_init_regen() { true; }

do_post_regen() {
    sed -e 's/__DOMAIN__/cryptpad.example.com/g' \
        -e 's/__APP__/cryptpad/g' \
        -e 's/__PORT__/3000/g' \
        -e 's/__PORT_SOCKET__/3004/g' \
        /etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf \
        > /etc/nginx/conf.d/sandbox.cryptpad.example.com.conf
    nginx -t && systemctl reload nginx
}

"do_$1_regen" "$(echo "${*:2}" | xargs)"
```

> **Important:** Replace `cryptpad.example.com`, `3000` and `3004` with your own values.

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Make the hook executable:

```bash
chmod +x /usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad
```

---

### Step 8: Test the Hook

Verify the hook works correctly:

```bash
yunohost tools regen-conf nginx
```

Then test again:

```bash
curl -I https://sandbox.cryptpad.example.com
```

If `HTTP/2 200` is returned, everything is correctly configured. ✅

The message `expected to be deleted but was kept back` is normal and can be ignored — it simply means YunoHost does not manage this file itself, but the hook recreates it immediately after every deletion attempt.

---

### Step 9: Verify CryptPad Functionality

Open `https://cryptpad.example.com` in your browser and verify:

- [ ] Home page loads correctly
- [ ] Create a new document (spreadsheet, presentation, code) — no more endless loading
- [ ] Run a backup via the YunoHost web interface

---

## Technical Explanation for Developers

### Background

CryptPad uses a sandbox domain as a security mechanism. All document editors are loaded in an `<iframe>` whose origin is the sandbox domain. This prevents any code running inside the editor from accessing cookies, LocalStorage, or other resources belonging to the main domain — a targeted XSS attack cannot steal user data from the main domain.

### The Problem in the YunoHost Package

The YunoHost package for CryptPad contains the file:

```
/etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf
```

This file contains placeholders (`__DOMAIN__`, `__APP__`, `__PORT__`, `__PORT_SOCKET__`) and defines both a port 80 block (with ACME challenge support) and a port 443 block (HTTPS proxy to the CryptPad instance).

**The problem:** This template file is never deployed to `/etc/nginx/conf.d/` during the installation process. The app's install script calls `ynh_add_nginx_config`, but apparently only for the main domain configuration — not for `nginx-sandbox.conf`.

### Consequences

1. **No port 80 block for `sandbox.*`:** Let's Encrypt writes the ACME challenge file to `/var/www/.well-known/acme-challenge-public/`, then tries to retrieve it via HTTP from `sandbox.cryptpad.example.com`. Since nginx has no server block for this domain on port 80, it falls back to the default behavior and redirects to HTTPS. Since the HTTPS certificate does not yet include the sandbox domain, the SSL connection fails (`CERTIFICATE_VERIFY_FAILED`), and Let's Encrypt receives `Response Code: None`.

2. **No port 443 block for `sandbox.*`:** CryptPad loads all editors via `https://sandbox.cryptpad.example.com`. Without an nginx configuration for this domain on port 443, there is no connection → endless loading in the UI.

### Workaround

The workaround described in this guide consists of two parts:

**Part 1:** Manually deploy the template file:
```bash
sed -e 's/__DOMAIN__/cryptpad.example.com/g' \
    -e 's/__APP__/cryptpad/g' \
    -e 's/__PORT__/3000/g' \
    -e 's/__PORT_SOCKET__/3004/g' \
    /etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf \
    > /etc/nginx/conf.d/sandbox.cryptpad.example.com.conf
```

**Part 2:** A `conf_regen` hook at `/usr/share/yunohost/hooks/conf_regen/99-nginx_sandbox_cryptpad` ensures the file is automatically recreated after every `yunohost tools regen-conf nginx` call (e.g. during YunoHost updates).

The hook follows the YunoHost hook convention with `do_pre_regen`, `do_init_regen`, and `do_post_regen` functions, called via `"do_$1_regen"`.

### Recommendation for Package Maintainers

The CryptPad app install script should deploy `nginx-sandbox.conf` analogously to the main domain configuration. Specifically, the `install` script should include an additional call after `ynh_add_nginx_config` to handle the sandbox configuration, or register the sandbox domain as a full YunoHost domain so that `regen-conf` manages it automatically.

Alternatively, a `conf_regen` hook could be shipped directly within the app package to handle this automatically on every system.

**Relevant files in the package:**
- `/etc/yunohost/apps/cryptpad/conf/nginx-sandbox.conf` — template exists but is never deployed
- `/etc/yunohost/apps/cryptpad/scripts/install` — install script, missing the deployment of the sandbox config

### Related Issues

This bug means that every fresh installation of CryptPad on YunoHost will have:
- Non-functional document editors (all types except "Write")
- Failing certificate renewals (potentially breaking HTTPS after 90 days)
- Potentially failing backups

This affects all users and is not a misconfiguration on the user's side — it is a packaging bug that should be fixed upstream.
