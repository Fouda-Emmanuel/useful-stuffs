# **📘 Domain, Email & Server Setup Guide**

*Spaceship + Brevo + AWS EC2*

---

## **⚡ Quick Checklist**

* [ ] Buy domain in Spaceship (e.g., `foudadev.site`)
* [ ] Get EC2 public IP & DNS (`<EC2_PUBLIC_IP>` / `<EC2_PUBLIC_DNS>`)
* [ ] Add **A records** in Spaceship for `@`, `api`, `portainer`, `flower` → EC2 IP
* [ ] Add domain in Brevo: `mg.foudadev.site` → choose **manual auth**
* [ ] Add Brevo DNS records (TXT, CNAME, DMARC) in Spaceship → Authenticate
* [ ] Add sender in Brevo: `noreply@mg.foudadev.site` → verify
* [ ] Generate Brevo SMTP key → add to `.env.production`
* [ ] Optional: Set `EMAIL_REPLY_TO` for user replies
* [ ] Test: DNS, Brevo domain + sender, Django email sending
* [ ] **Dynamic EC2 IP management**: Update A records if IP changes

---

## **Overview**

This guide explains how to:

1. Configure DNS records in **Spaceship** for a domain and subdomains.
2. Authenticate a domain in **Brevo** for transactional emails.
3. Create a verified **sender email** in Brevo.
4. Generate a Brevo **SMTP key** and connect it to Django.
5. Connect subdomains to an **AWS EC2** instance.
6. Handle **dynamic EC2 IP changes**.

**Tools used:**

* Domain Registrar / DNS: **Spaceship**
* Email / SMTP Provider: **Brevo**
* Server: **AWS EC2 (Free Tier)**

---

## **Step 1 — Prepare Your Domain**

1. Buy a domain in **Spaceship** (or any registrar). Example: `foudadev.site`.
2. Keep track of your **domain name** and **nameservers** (Spaceship default: `launch1.spaceship.net`, `launch2.spaceship.net`).

---

## **Step 2 — Set Up A Records for EC2**

1. Get your EC2 **public IPv4 address** and **DNS** from AWS:

```
<EC2_PUBLIC_IP>
<EC2_PUBLIC_DNS>
```

2. Go to **Spaceship → Domain → Advanced DNS → Add Record**.

Add these **A records**:

| Host      | Type | Value           | Notes         |
| --------- | ---- | --------------- | ------------- |
| @         | A    | <EC2_PUBLIC_IP> | Main domain   |
| api       | A    | <EC2_PUBLIC_IP> | API subdomain |
| portainer | A    | <EC2_PUBLIC_IP> | Portainer UI  |
| flower    | A    | <EC2_PUBLIC_IP> | Celery Flower |

> `@` points to the root domain (`foudadev.site`).
> `api.foudadev.site`, `portainer.foudadev.site`, `flower.foudadev.site` will point to the same EC2 instance.

---

## **Step 3 — Add Domain in Brevo**

1. Go to **Brevo → Senders, Domains & Dedicated IPs → Add Domain**
2. Enter your domain/subdomain:

```
mg.foudadev.site
```

3. Choose: **“Authenticate the domain yourself – Set up your domain records manually in your domain provider account.”**
4. Click **Continue**.

---

## **Step 4 — Add DNS Records in Spaceship (after Brevo)**

After Brevo shows you the records:

1. Go to **Spaceship → Advanced DNS → Add Record**
2. Add the **Brevo authentication records** exactly as given:

| Host / Name          | Type  | Value / Target                                   |
| -------------------- | ----- | ------------------------------------------------ |
| mg                   | TXT   | brevo-code:<BREVO_CODE>                          |
| brevo1._domainkey.mg | CNAME | b1.mg-foudadev-site.dkim.brevo.com               |
| brevo2._domainkey.mg | CNAME | b2.mg-foudadev-site.dkim.brevo.com               |
| _dmarc.mg            | TXT   | v=DMARC1; p=none; rua=mailto:rua@dmarc.brevo.com |

3. Click **Save all**
4. Wait **10–30 minutes** for DNS propagation.
5. Go back to Brevo → Click **Authenticate**. ✅ Status should change to **Authenticated**.

---

## **Step 5 — Add and Verify Sender in Brevo**

1. Go to **Brevo → Senders → Add Sender**
2. Fill the fields:

| Field         | Example                                                     |
| ------------- | ----------------------------------------------------------- |
| From Name     | FoudaWire                                                   |
| From Email    | [noreply@mg.foudadev.site](mailto:noreply@mg.foudadev.site) |
| Phone Overlay | (optional, leave empty)                                     |

3. Click **Add / Save**
4. Status will show **Verified** once Brevo confirms the sender (even if mailbox doesn’t exist).

---

## **Step 6 — Generate Brevo SMTP Key**

1. Go to **Brevo → SMTP & API → SMTP → Your SMTP Keys**
2. Click **“Click here to generate an SMTP key”**
3. Copy the key immediately — it will only be shown once.
4. Keep it secure (password manager or secrets manager).

---

## **Step 7 — Configure Django Email Settings**

Add these to `.env.production`:

```env
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp-relay.brevo.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=<BREVO_LOGIN_EMAIL>
EMAIL_HOST_PASSWORD=<BREVO_SMTP_KEY>
DEFAULT_FROM_EMAIL="FoudaWire <noreply@mg.foudadev.site>"
```

> Replace `<BREVO_LOGIN_EMAIL>` and `<BREVO_SMTP_KEY>` with your actual Brevo values.

---

### **Optional — Receiving Replies**

* If you want users to reply to emails sent from `noreply@mg.foudadev.site`, create a mailbox in Spaceship.
* Otherwise, set a `reply-to` in Django:

```python
EMAIL_REPLY_TO = "support@foudadev.site"
```

---

## **Step 8 — Dynamic EC2 IP Management**

* If your EC2 instance stops/starts, the **public IP may change** (unless using Elastic IP).
* When this happens:

1. Check the new EC2 public IP.
2. Go to **Spaceship → Advanced DNS → Update A records** for `@`, `api`, `portainer`, `flower` with the new IP.
3. Wait a few minutes for DNS propagation.
4. Your domain and subdomains will point to the updated instance automatically.

> Optional: Use an **Elastic IP** in AWS to avoid changing DNS every time.

---

## **Step 9 — Verify Everything**

1. **Spaceship DNS**: All records should show green / online.
2. **Brevo**: Domain authenticated, sender verified.
3. **Django**: Test sending an email via the SMTP setup:

```python
from django.core.mail import send_mail
send_mail("Test Email", "Brevo SMTP works!", "noreply@mg.foudadev.site", ["yourpersonal@gmail.com"])
```

---

## **Step 10 — Domain & Email Flow Diagram**

```
                   +------------------+
                   |   Spaceship DNS   |
                   | (Domain & Subdomains)
                   +--------+---------+
                            |
                            | A records
                            v
                   +------------------+
                   |     AWS EC2       |
                   | <EC2_PUBLIC_IP>   |
                   +--------+---------+
                            |
                            | Hosts web apps & API
                            v
                   +------------------+
                   |      Brevo        |
                   | mg.foudadev.site |
                   +--------+---------+
                            |
                            | Authenticated sender
                            v
                   +------------------+
                   |     Django App    |
                   | Uses SMTP to send |
                   | emails via Brevo  |
                   +------------------+
```

---

## ✅ **Summary**

* **Domain setup**: Main + subdomains → EC2 IP placeholder
* **Brevo setup**: Authenticated subdomain `mg.foudadev.site`
* **Sender setup**: `noreply@mg.foudadev.site` verified
* **SMTP setup**: Key generated, added to Django `.env.production`
* **EC2 deployment**: DNS points to your instance; update A records if public IP changes

---
