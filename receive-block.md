
#  Mail Server Troubleshooting Summary – Inbound Mail Issue

##  Objective

Investigate why **your mail server isn't receiving incoming mail** despite being configured properly.

---

##  1. Checked if Mail Services Are Running

**Command:**

```bash
sudo ss -lntp | egrep ':(25|587|110|143|993|995)'
```

 **Result:**  
All relevant services are listening:
- Postfix on ports **25** and **587**
- Dovecot on ports **110**, **143**, **993**, **995**

---

##  2. Verified Local Firewall on mail server

**Command:**

```bash
sudo ufw status
# (Or checked that firewalld is not in use)
```

 **Result:**  
No firewall active on the mail server — not blocking any ports.

---

##  3. Tested Port Reachability from Outside

**Command:**

```bash
nc -zv -w 3 aristo.fcoos.in 25 587 443 993 995 143 110
```

 **Result:**
- Only port **443** was reachable
- All mail-related ports (**25**, **587**, etc.) **timed out**

---

##  4. Verified pfSense Interfaces

**Command:**

```bash
ifconfig
```

 **Result:**
- Public IP (`148.113.12.50`) is on `em0` (WAN)
- Internal network uses `em1` (LAN)
- Network interfaces are set up correctly

---

##  5. Packet Capture on pfSense

**Command:**

```bash
tcpdump -ni em0 port 25      # WAN
tcpdump -ni em1 port 25      # LAN
```

 **Result:**
- **No inbound traffic on port 25** seen on `em0`
- Confirms the **cloud provider blocks inbound port 25**

---

##  6. Outbound Mail Test

 You confirmed:
- Your server **can send mail successfully**
- Sending via Postfix and port 25 or 587 works

---

##  7. MX Record Check

**Command:**

```bash
dig MX aristo.fcoos.in
```

 **Result:**

```text
aristo.fcoos.in.  300  IN MX  0 aristo.fcoos.in.
```

- MX points to itself: `aristo.fcoos.in`
- External servers will try to connect to port 25 — but that is **blocked**

---

##  Final Conclusion

| Working                    | Not Working                            |
|----------------------------|----------------------------------------|
| Mail server services       | Inbound email (port 25 is blocked)     |
| Sending mail (outbound)    | Receiving mail from other domains      |
| NAT and pfSense rules      | Public mail delivery on port 25        |

---

## 🛠 Next Steps

To receive mail despite provider restrictions:

### Option 1: **Ask Provider to Unblock Port 25**
- May require support ticket or identity verification

### Option 2: **Use a Mail Relay (Smart Host)**
- Set MX to relay service (e.g., Mailgun, SMTP2GO)
- Relay forwards mail to your server on port 587
- Update DNS and Postfix accordingly

Let me know if you'd like a detailed setup for a specific relay provider.
