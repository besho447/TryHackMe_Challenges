# Recruit (TryHackMe) — Root Cause & Mitigation Analysis

> Offensive/Defensive breakdown of the vulnerabilities chained in the **Recruit** room (TryHackMe).
> Goal: not just "how it was exploited" but **why the code allowed it** and **how to fix it properly**.

**Attack chain:**
```
Nmap → Directory Enumeration → mail.log (Info Disclosure)
     → file.php?cv= (LFI/SSRF via PHP wrappers) → config.php → HR creds
     → HR Login → SQL Injection (Union-based) → users table → Admin creds
     → Admin Access
```

---

## 1. Information Disclosure — `mail.log`

**Root Cause**
- Log file placed inside the web root instead of a private path (e.g. `/var/log`)
- Directory indexing left enabled on the web server (default misconfiguration, never hardened)
- No server-level rule blocking access to sensitive extensions (`.log`, `.env`, `.bak`, `.config`) as a defense-in-depth layer

**Mitigation**
- Store logs outside the web root, or in a directory with no HTTP exposure
- Disable directory indexing (`Options -Indexes` on Apache / `autoindex off` on nginx)
- Deny rules for sensitive file extensions, even as a backup layer in case a file is misplaced

---

## 2. LFI / SSRF via `file://` wrapper — `file.php?cv=`

**Root Cause**
- User-controlled input passed directly into a file-handling function (`file_get_contents`, `include`, `fopen`, etc.) with no restriction on the URI scheme or path
- No whitelist validation — the parameter accepted any scheme (`file://`, `php://`, `http://`, `data://`) instead of only an expected file identifier
- No hardening at the PHP configuration level

**Mitigation**
- Strict whitelist for the input (e.g. a filename/ID resolved server-side, never a raw path or URL)
- Explicitly reject unexpected schemes (`file://`, `php://`, `data://`, …)
- `allow_url_fopen = Off` and `allow_url_include = Off` in `php.ini`
- Use `basename()` + a fixed, server-controlled base path — never let user input define the full path

---

## 3. SQL Injection (Union-based) — Search functionality

**Root Cause**
- Query built via string concatenation instead of parameterized queries/prepared statements
- Verbose/raw database error messages returned to the client, enabling column-count enumeration (`UNION SELECT` tuning) via trial and error

**Mitigation**
- Always use **Prepared Statements** (`mysqli`/PDO with bound parameters)
- **Least privilege** DB account for the application (no access to tables/columns it doesn't need)
- Disable verbose errors in production (`display_errors = Off`), use generic error pages + server-side logging
- WAF as an additional layer — never a substitute for fixing the query logic

---

## Key Takeaways

- **Defense in depth matters**: each bug here could've been stopped at more than one layer (code, server config, DB permissions).
- **Never trust client input for scheme/path/structure** — validate against a whitelist, not a blacklist.
- **Verbose errors are a reconnaissance tool for attackers** — treat error output as sensitive data.
- **Credential hygiene**: storing plaintext creds in `config.php` reachable via LFI is itself a separate anti-pattern (secrets management / env variables should be used instead).

---

*Methodology notes based on hands-on exploitation of the Recruit room — https://tryhackme.com/room/recruitwebchallenge*
