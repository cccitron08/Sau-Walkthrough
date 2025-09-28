<img width="247" height="118" alt="sau-icon" src="https://github.com/user-attachments/assets/0195af54-95ff-42c4-8050-a7a0d23b722b" />


Walkthrough — `Request‑Baskets SSRF` → `Maltrail RCE` → `systemd LPE` 

`Short summary`
A public Request‑Baskets instance (v1.2.1) exposed an API SSRF `(CVE‑2023‑27163)`. By abusing the API parameter on /api/baskets/{name} we made the app connect back to the box and discovered an internal Maltrail instance (v0.53). Maltrail was exploited for unauthenticated RCE (public PoCs exist), giving a user shell. A sudo/systemctl misconfiguration combined with systemd < 247 allowed a pager escape to escalate to root `(CVE‑2023‑26604)`. Links to the CVEs and public PoCs are listed below for reference.

`Basic facts`

Open ports observed: `20`, `80`, `8338`, `55555`

Service on `55555`: `Request‑Baskets` `(v1.2.1)`

`Vulnerability`: `SSRF` via the API component `/api/baskets/{name}` `(CVE‑2023‑27163)`. 

What `SSRF` means (simple): the server itself is tricked into making network requests you control — so you can reach internal services that you can’t normally access from your machine. 

Internal service discovered via `SSRF`: Maltrail on port `80` `(v0.53)`, which is known to have an unauthenticated command injection / `RCE`. Public exploits and writeups exist. 

`Privilege escalation`: systemd pager escape (systemd before 247) when `systemctl` status is allowed by `sudo`; tracked as `CVE‑2023‑26604`. 

`How the chain actually worked`

Scan → discover: a quick scan revealed a web app on port `55555`. Opening it showed `Request‑Baskets v1.2.1` and an API endpoint that can be told to forward/fetch other URLs on the server side. The problem is in the API parameter for /api/baskets/{name}, which accepts a target URL — that’s the `SSRF` vector. 

<img width="1426" height="891" alt="nmap" src="https://github.com/user-attachments/assets/631f9dcb-4cae-4a00-9bd6-22db430ac7d4" />

Abuse `SSRF` to reach inside: by instructing the vulnerable API parameter to request `http://127.0.0.1:80` the server fetched the internal web service and revealed what was running there. This is classic `SSRF`: you make the server do the network request for you. 

<img width="1914" height="995" alt="web1" src="https://github.com/user-attachments/assets/b5e1e616-285a-485f-96d6-d851e001016c" />

<img width="256" height="24" alt="1 2 1" src="https://github.com/user-attachments/assets/4ea2d339-adc9-4f04-a59d-d9f6b4ad896f" />

<img width="1006" height="315" alt="cve-api" src="https://github.com/user-attachments/assets/789be94f-4905-47a3-a7bd-bdbef49e6cda" />

<img width="1238" height="440" alt="basket1" src="https://github.com/user-attachments/assets/f0294da3-b379-44cf-8ab9-53377dfdf4bf" />

<img width="824" height="451" alt="127" src="https://github.com/user-attachments/assets/72b5048f-147b-4956-8774-156df6b68c30" />


Find `Maltrail` on port `80`: the internal site exposed was `Maltrail v0.53`. Maltrail v0.53 has a known unauthenticated command‑injection / RCE in its web component; several community PoCs and exploit scripts demonstrate how to trigger it. 

<img width="1654" height="956" alt="80-1" src="https://github.com/user-attachments/assets/a435d452-72a9-4189-a21b-ab3a543a1200" />


Exploit Maltrail → user shell: an available public exploit (PoC) was used to trigger the Maltrail issue and obtain a shell running under a normal user account. The exploit artifacts are public `(exploit-db / GitHub mirrors)`.

<img width="610" height="150" alt="python" src="https://github.com/user-attachments/assets/79f21bd9-cc76-4f7b-82ae-f07fc0b5b45f" />

<img width="259" height="79" alt="8000" src="https://github.com/user-attachments/assets/248f4206-2a43-441a-a6d2-41c8471c2959" />

`Privilege escalation` → root: sudo -l on the compromised account showed a sudo rule permitting systemctl status trail.service. On systems with systemd < 247, systemctl status can invoke the less pager when output is long; because LESSSECURE wasn’t set, less could be used to escape to a shell running as root. This weakness is documented as `CVE‑2023‑26604` and vendor advisories. Using that sequence produced a `root shell`. 

<img width="1010" height="221" alt="cve-system" src="https://github.com/user-attachments/assets/8d2146c4-cbde-4640-9ce5-bdba55226716" />

<img width="793" height="337" alt="systemd" src="https://github.com/user-attachments/assets/fd3f9ec1-982b-48d8-add9-660bf0ff240b" />

<img width="535" height="212" alt="sudo-bin" src="https://github.com/user-attachments/assets/3d0c5353-1e78-41cf-b9d4-01d22f6e49ef" />

Why this was effective (short)

Request‑Baskets let an attacker control an API parameter that triggers server‑side HTTP requests → SSRF. That’s how an attacker pivoted from the externally reachable app into internal services. 
NVD

Maltrail v0.53 accepted attacker‑controlled input in a way that allowed command injection (unauthenticated RCE), so code executed on the server. 
Rapid7

systemd versions before 247 did not force the pager into a safe mode when systemctl was invoked via sudo, enabling a local pager escape to root if sudo allowed systemctl status. 
NVD

Impact 

This chain allows an unauthenticated remote attacker to pivot into internal services, execute code on the host, and escalate to root — a full compromise.

`Links & references (read these for details / PoCs)`

`CVE‑2023‑27163` (Request‑Baskets SSRF) — NVD entry. 

https://nvd.nist.gov/vuln/detail/CVE-2023-27163

PoC / community repos for `CVE‑2023‑27163` (examples): GitHub PoCs.

(example) https://github.com/entr0pie/CVE-2023-27163

`Maltrail v0.53 RCE` — public PoC / exploit listings (Exploit‑DB / GitHub). 

(example) https://github.com/spookier/Maltrail-v0.53-Exploit

`CVE‑2023‑26604` (systemd pager LPE) — NVD and vendor advisories. 

https://nvd.nist.gov/vuln/detail/CVE-2023-26604

Quick remediation checklist

Patch request‑baskets to a fixed release; if not possible, disable any feature that lets untrusted input control server‑side requests. Whitelist allowed targets and enforce URL validation. 

Update or harden Maltrail (or take it off public networks), apply vendor patches, and protect admin interfaces behind authentication and network ACLs. 
GitHub

Update systemd to a patched version (>= 247 or vendor backport) and avoid giving systemctl status or other dangerous commands to untrusted users via sudo. Review sudoers for overly permissive entries.


<img width="699" height="621" alt="finish" src="https://github.com/user-attachments/assets/456e92c2-d870-43f8-835a-522546411bce" />

