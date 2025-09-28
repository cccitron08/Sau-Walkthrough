<img width="247" height="118" alt="sau-icon" src="https://github.com/user-attachments/assets/0b97f676-7a1b-494b-a843-c770115029e5" />


Machine Info

* **Name:** `Sau`
* **OS:** `Windows`
* **Category:** `Web`
* **Difficulty:** `Easy`
* **Source:** https://app.hackthebox.com/machines/551
* **Date solved:** `2025-09-28`

---

## Summary

Request-Baskets SSRF → Maltrail RCE → systemd pager LPE

Found SSRF in Request-Baskets API parameter and used it to make the app fetch the host’s internal web service.

Discovered Maltrail on the internal service and exploited a public Maltrail RCE PoC to get a user shell.

Checked sudo privileges, found systemctl status trail.service allowed.

Abused systemd’s pager behavior (systemd < 247) to escape to a root shell.

Root obtained.

## Open services (important ones)

55555 request-baskets v1.2.1
8338 Maltrail v0.53

---

## Credits

* **Lab author:** `sau123`
* **Author link:** https://app.hackthebox.com/users/201596 
