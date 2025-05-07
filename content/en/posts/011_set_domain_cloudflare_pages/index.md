+++
date = '2025-05-07T19:24:45+09:00'
draft = false
title = 'Apply a Custom Domain to Cloudflare'
+++

## Purpose

Apply a custom domain to your blog.

## Summary of the Setup

- Domain registrar: SAKURA Internet
- Blog hosting: Cloudflare (DNS management, CDN/security, and hosting)

## Procedure (Overview)

1. Add the domain to Cloudflare

- Log in to Cloudflare
- Proceed with the “Free” plan (no need for a paid plan)
- Use “Add Domain” to add the newly purchased domain to your site
  (The domain status will initially be *Invalid nameservers*)
- Set up DNS records (A record, CNAME, etc., if needed; in our case, this was automatic)

2. Check the nameservers provided by Cloudflare

Example: `clark.ns.cloudflare.com`, `dina.ns.cloudflare.com`

3. Log in to the SAKURA Internet control panel

- Remove the existing nameservers under domain settings
- Add the nameservers specified by Cloudflare

4. Wait for propagation

Once complete, the domain status in the Cloudflare dashboard will display **"Status: Active"**.

---

## ✅ Configuration Notes

- **DNS Record Settings**
In the “DNS” tab of the Cloudflare dashboard, verify the CNAME record::

| Type  | Name        | Content                |
|-------|-------------|------------------------|
| CNAME | yourdomain.com | yourblog.pages.dev   |


This configuration allows content hosted on Cloudflare Pages (yourblog.pages.dev) to be displayed when the domain name (yourdomain.com) is entered.

---

💡 **About A Records**

An A record points a domain directly to an IP address.
If you're hosting on your own server or using a traditional web host, an A record is typically used to map to the server's IP.

However, for services like **Cloudflare Pages**, a static site hosting platform, a **CNAME record** is usually sufficient.

Therefore, in this case, as long as the CNAME is set properly, you don't need to add an A record.

---

- **SSL/TLS Configuration**
In Cloudflare’s **SSL/TLS** settings, set the mode to **Full** or **Full (Strict)** to enable HTTPS automatically.

- **Caching & Optimization**
You can fine-tune page caching and compression under the **Speed** and **Caching** tabs in Cloudflare.

---

5. Access the domain from your browser to verify that it works
<p></p>

6. Update the `baseURL` in your `config.toml` (ideally, this should be done earlier)

- Change:
  `baseURL = "https://yourdomain.com/"`
- Rebuild with the `hugo` command
- Push to GitHub → This triggers a redeploy on Cloudflare Pages