# Splunk Enterprise Installation Guide (Windows)

This guide documents the step-by-step process of installing and accessing **Splunk Enterprise 10.2.2** on a Windows machine.

---

## Prerequisites

- Windows Server 2019 / 2022 / 2025 (64-bit)
- At least **1 GB** of free disk space for the installer
- A valid Splunk account (free trial available)
- Administrator privileges on the machine

---

## Step 1 — Download Splunk Enterprise

Visit the official Splunk download page and select the Windows `.msi` package.

 **Version:** Splunk Enterprise 10.2.2  
 **Package Size:** ~1026 MB  
 **Supported OS:** Windows Server 2019, 2022, 2025 (64-bit)

![Splunk Enterprise Download Page](s1.png)

**Download steps:**
1. Go to [splunk.com/download](https://www.splunk.com/en_us/download/splunk-enterprise.html)
2. Select the **Windows** tab
3. Click **Download Now** next to the 64-bit `.msi` installer
4. Sign in or create a free Splunk account if prompted

**Note:** The free trial allows indexing up to **500 MB/day**. After 60 days, you can convert to a perpetual free license or upgrade to a full Enterprise license.

---

## Step 2 — Install & Create Admin Credentials

Run the downloaded `.msi` installer and follow the on-screen wizard. During installation, you will be prompted to **create an admin username and password** — these credentials will be used to log into the Splunk Web interface.

**Important:** Remember your credentials. If lost, a password reset requires access to the Splunk server directly.

After installation completes, Splunk Enterprise starts automatically and is accessible at:

```
http://localhost:8000
```

---

## Step 3 — Log In to Splunk Web

Open your browser and navigate to `http://localhost:8000`. Enter the **username and password** you created during installation.

![Splunk Enterprise Login Page](s2.png)

**Login details (default):**

| Field    | Value                                      |
|----------|--------------------------------------------|
| Username | The username you set during installation   |
| Password | The password you set during installation   |

**First time signing in?** Use the credentials you created at installation. If this is a shared instance, contact your Splunk administrator.

---

## Step 4 — Splunk Home Dashboard

After a successful login, you will land on the **Splunk Home Dashboard** — the central hub for monitoring, searching, and auditing logs.

![Splunk Enterprise Home Dashboard](s3.png)

From here you can:

- **Search & Report** — Run SPL (Search Processing Language) queries on indexed data
- **Apps** — Install and manage Splunk apps (e.g., Audit Trail, Splunk Secure Gateway)
- **Dashboards** — View pre-built or custom dashboards for log monitoring and analytics
- **Settings** — Configure data inputs, indexes, users, and roles
- **Activity** — Review job history and triggered alerts

---



