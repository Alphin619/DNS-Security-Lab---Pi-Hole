# :strawberry: DNS Security Lab: Pi-hole on a Raspberry Pi 4

A hardened, headless Raspberry Pi running Pi-hole as the DNS filter and DHCP server for my home network, with threat-intelligence blocklists, per-device logging and a set of short investigation notes written the way I would write them in a SOC.
 
**Status:** running since 14 October 2026. This write-up covers the build and the first eight hours of data. I'll add a one-month review in mid-November 2026.
 
> **How to read this:** sections 1 to 6 describe what I built and how I checked it. Sections 7 to 9 are the analyst side: results, three investigation notes and the limits of what I can claim. Section 10 is what went wrong and what I would change.
 
---
 
## 1. Objective and SOC relevance
 
DNS is one of the most useful telemetry sources a SOC analyst works with. Almost everything that talks to the internet asks DNS first, so lookups show up in investigations, in threat hunting and in detections for malware beaconing and phishing. I built this lab to get hands-on with those ideas on a network I control.
 
What the project demonstrates:
 
| Skill | Where it shows up |
|---|---|
| Log analysis and triage | Section 8: three investigation notes |
| Threat intelligence feeds (IOCs) | Section 5.6: URLhaus and Phishing Army, with attribution of each block |
| Hardening and verification | Section 5.4: key-only SSH, tested positively and negatively |
| Asset inventory and identification | Sections 4 and 6: mapping IPs to named devices through DHCP |
| Baselines | Listening ports and auth logs captured before and after the install |
| Change management | Section 6: before and after state, risk, rollback |
| Integrity checking | Section 5.2: SHA256 verification of the OS image |
| Honest reporting of limits | Sections 9 and 10 |
 
This project is deliberately standalone. Forwarding the logs to Microsoft Sentinel for a longer simulated-SOC exercise is planned and described in the roadmap (section 11), not built.
 
---
 
## 2. Architecture
 
```
  Phone / PC / TV / other devices
              |
              |  DHCP: gets an address, gateway 192.168.0.1, DNS 192.168.0.205
              |  DNS:  asks 192.168.0.205 for every lookup
              v
   +------------------------------+
   |  Raspberry Pi 4 (Pi-hole)    |   blocklists: ads/trackers, malware hosts, phishing
   |  192.168.0.205 (static)      |   per-client query log, DHCP lease table
   +------------------------------+
              |
              |  allowed lookups only, forwarded as plain DNS (port 53)
              v
     Quad9 (filtered, DNSSEC)  --->  internet
 
   ISP-provided (192.168.0.1): routing only, its DHCP server is disabled
```

 
---
 
## 3. Hardware and software
 
| Item | Detail |
|---|---|
| Board | Raspberry Pi 4 Model B, 2GB RAM |
| Storage | Raspberry Pi 32GB microSD (V30 / A2) |
| Power | Official Raspberry Pi USB-C power supply |
| Case | Raspberry Pi 4 case |
| Flashing | Beikell USB card reader, Raspberry Pi Imager v2.0.12 |
| Network | Wired Ethernet only, no Wi-Fi configured |
| OS | Raspberry Pi OS Lite (64-bit), Debian 13 (trixie), release of 6 Oct 2026, kernel 6.18 |
| DNS filter | Pi-hole v6 (settings live in `pihole.toml`) |
| Upstream resolver | Quad9, filtered with DNSSEC |
| Blocklists | StevenBlack hosts, abuse.ch URLhaus, Phishing Army (extended) |
 
 
<img width="3000" height="4000" alt="1" src="https://github.com/user-attachments/assets/8db1f5f1-4b9e-4c11-bd9f-be13dbe80dc7" />

<img width="3000" height="4000" alt="2" src="https://github.com/user-attachments/assets/c2eae5d8-6711-448d-9015-5c16be44a33d" />

<img width="3000" height="4000" alt="3" src="https://github.com/user-attachments/assets/991c8720-df13-4f0c-b17a-b8e86e3e9ae2" />
 
---
 
## 4. Asset inventory
 
| Hostname | Role | Address | OS | Notes |
|---|---|---|---|---|
| Alphinspihole-dns | DNS filter and DHCP server | 192.168.0.205 (static, set on the host) | Raspberry Pi OS Lite 64-bit, Debian 13, kernel 6.18.50+rpt-rpi-v8 | Ethernet only, key-only SSH |
| (ISP-provided) | Router and gateway | 192.168.0.1 | n/a | DHCP server disabled on 14 Oct 2026 |
 
Client devices (a PC, phones, a streaming device and others) appear in Pi-hole's lease table by hostname once DHCP moved to the Pi. I have left other people's devices out of this write-up.
 
---
 
## 5. Build log
 
Each phase says what I did, why it matters from a SOC point of view, and how I checked it.
 
### 5.1 Hardware and flashing
 
**What I did:** downloaded Raspberry Pi Imager from the official site, selected Raspberry Pi 4, Raspberry Pi OS Lite (64-bit) and the 32GB card (31.7 GB, shown as `E:\`), then set the customisations: hostname, localisation, a non-default user and SSH on. I left Wi-Fi blank and Raspberry Pi Connect off.
 
**Why:** Lite has no desktop, so there are fewer services and less attack surface. A non-default username avoids the default-credential problem, and a single wired network path keeps the topology simple and documentable. Leaving Raspberry Pi Connect off means no extra remote-access route that I would have to monitor.
 
**Check:** the Imager summary lists exactly the customisations I intended, and the write completed and verified.
 
<img width="1269" height="707" alt="4" src="https://github.com/user-attachments/assets/a17b0c82-8e0b-4490-959b-1bc209c99842" />
<img width="681" height="482" alt="6" src="https://github.com/user-attachments/assets/7cb59059-e692-454e-b351-bb8889bc82da" />
<img width="680" height="479" alt="8" src="https://github.com/user-attachments/assets/e26e839d-8264-4c1f-8696-69d29b5e658c" />
<img width="684" height="479" alt="9" src="https://github.com/user-attachments/assets/718d55a6-af66-4a7d-98b9-67658d68003a" />
<img width="679" height="476" alt="10" src="https://github.com/user-attachments/assets/50b6c0c4-53fc-4ea6-9962-de475b907552" />
<img width="686" height="482" alt="11" src="https://github.com/user-attachments/assets/f76cf28b-f2fe-4d65-8fbe-d15dc646e963" />
<img width="680" height="476" alt="12" src="https://github.com/user-attachments/assets/f5973723-1c71-49dc-9d23-70a6b12418bd" />
 
### 5.2 Integrity check of the OS image
 
**What I did:** ran `Get-FileHash` (SHA256) on the downloaded `.img.xz` and compared the result with the hash published on the Raspberry Pi download page.
 
**Result:** the two values match.
 
```
483db18a48da399b5b7022ffae9a07bc6d99daab30e593aedac09b69ff422843
```
 
**Limits, stated plainly:**
- I ran this check after flashing, not before. Imager downloads and verifies its own copy, so my check covers the file I downloaded separately and not necessarily the exact copy written to the card.
- A matching hash proves the file is intact and matches what the site published. It does not prove authenticity if the site itself were compromised, because the hash and the download come from the same place.
**Why it matters:** analysts compare file hashes constantly, against threat-intel feeds and when verifying software. The habit I would keep is checking before use, not after.
 
<img width="1111" height="627" alt="13" src="https://github.com/user-attachments/assets/49e73770-8f22-4138-8956-0ee5161784d0" />
<img width="776" height="451" alt="14" src="https://github.com/user-attachments/assets/d93d71e2-b90c-47c2-a950-55cb1f8ae846" />
 
### 5.3 First access and baseline
 
**What I did:** connected over SSH using the `.local` name. The first connection shows a host-key fingerprint that has to be trusted (trust on first use), and I accepted it. I then recorded the address, the interface details, the listening ports and the SSH log.
 
**Baseline captured before Pi-hole was installed:**
- Listening ports: `sshd` on port 22 and nothing else
- SSH log: one successful password login from my PC, no failed attempts
- OS: aarch64, kernel 6.18.50+rpt-rpi-v8
**Network identity:** the Pi took a DHCP address of 192.168.0.205 and I created a reservation for it on the Hub. I rebooted and confirmed the address persisted.

<img width="713" height="432" alt="17" src="https://github.com/user-attachments/assets/3bc537b2-7715-4eeb-9c34-0422bb8df562" />
<img width="700" height="142" alt="18" src="https://github.com/user-attachments/assets/a933902c-b552-477d-a693-7cbbe6c43531" />
<img width="699" height="372" alt="21" src="https://github.com/user-attachments/assets/69b4407d-103b-469a-93c3-8bcbace89718" />
<img width="693" height="135" alt="23" src="https://github.com/user-attachments/assets/ea5e073b-2487-4e3d-99a9-b66884dabd54" />
<img width="687" height="266" alt="24" src="https://github.com/user-attachments/assets/721dabe0-bf1f-4595-9756-f475abc399f0" />

 
### I also updated the OS with `sudo apt update && sudo apt full-upgrade -y` before putting any services on it, because patching comes before exposure.
 
 <img width="702" height="408" alt="22" src="https://github.com/user-attachments/assets/25ecc880-cbbd-4e7f-bcce-ee90c2fbadae" />

### 5.4 SSH hardening
 
**Goal:** remove password logins so SSH brute-forcing is no longer possible, and prove the change worked in both directions.
 
**Steps:**
1. Generated an ed25519 key pair on my PC with a passphrase.
2. Added the public key to `~/.ssh/authorized_keys` on the Pi. I checked the file first: it was empty (0 bytes), owned by my user, and dated 6 Oct, which matches the image build date.
3. Checked permissions: `~/.ssh` is 700 and `authorized_keys` is 600.
4. Logged in with the key (using the IP address) before changing anything.
5. Added a drop-in file, `/etc/ssh/sshd_config.d/01-hardening.conf`:
```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```
 
6. Validated the syntax with `sshd -t`, checked the **effective** settings with `sshd -T`, then restarted the service.
**Verification:**
 
| Test | Result |
|---|---|
| Effective settings (`sshd -T`) | Password and keyboard-interactive off, root login off, public key on |
| Forced password login | `Permission denied (publickey)` |
| Key login after the restart | `Accepted publickey` at 21:53 from my PC, same key fingerprint as before the change |
 
The key fingerprint printed when I generated the key matches the fingerprint recorded in the SSH log, so a login can be attributed to that specific key.
 
**Why it matters:** brute-forcing exposed SSH is one of the most common patterns analysts see in authentication logs (MITRE ATT&CK T1110). Keeping `authorized_keys` clean and tightly permissioned also matters because adding a key is a known persistence technique (T1098.004).
 
<img width="687" height="416" alt="25" src="https://github.com/user-attachments/assets/9d8a619f-6180-4591-adb3-268b6e43c41f" />
<img width="927" height="344" alt="27" src="https://github.com/user-attachments/assets/be507e23-d35c-4968-b32d-fa8bd3d15634" />
<img width="858" height="526" alt="28" src="https://github.com/user-attachments/assets/cfc1aad9-a0f1-4796-95ca-eb7dac0366a9" />
<img width="845" height="171" alt="29" src="https://github.com/user-attachments/assets/1462be3e-67e2-467e-ab57-413adfffb03d" />
<img width="738" height="69" alt="30" src="https://github.com/user-attachments/assets/a3c0793e-ba89-4ac9-b359-047c792b36a8" />

 
### 5.5 Pi-hole install
 
**What I did:** downloaded the install script to a file first, then ran it with `sudo bash`, rather than piping it straight into a shell. I chose **Quad9 (filtered, DNSSEC)** as the upstream resolver, set a dashboard password with `sudo pihole setpassword`, and checked service health.
 
**Why Quad9:** it refuses to resolve some known-malicious domains and validates DNSSEC, which gives a second filtering layer behind Pi-hole's own lists. The trade-off is that upstream queries leave the Pi as plain DNS on port 53 (see section 9).
 
**Baseline versus after:**
 
| | Before | After |
|---|---|---|
| Listening ports | 22 (sshd) | 22 (sshd), 53 (DNS), 80 and 443 (web dashboard), all three new ones owned by `pihole-FTL` |
 
The `sshd` process ID changed from 770 to 980 between the two captures. That is the restart from the hardening change in 5.4, which is a useful reminder that a different PID needs an explanation.
 
**First functional test (from my PC, aimed at the Pi):** `google.com` resolved normally, and `doubleclick.net` returned `0.0.0.0` and `::`. The Query Log showed the first as forwarded to Quad9 (9.9.9.9) and the second as blocked by gravity.
 
<img width="848" height="555" alt="32" src="https://github.com/user-attachments/assets/2464fa08-7e3a-44d0-9155-25de4c1ff4f2" />
<img width="636" height="373" alt="33" src="https://github.com/user-attachments/assets/5950b135-4f47-4d3e-be7c-820ec33a899f" />
<img width="1057" height="552" alt="34" src="https://github.com/user-attachments/assets/cc964d72-e4d5-430a-a233-9e4a326044c6" />
<img width="613" height="467" alt="35" src="https://github.com/user-attachments/assets/4ab07acc-41af-4ee1-94ac-b3f741e2b4d5" />
<img width="1003" height="788" alt="36" src="https://github.com/user-attachments/assets/01980bf0-f851-41a5-8a9f-cee9190edd46" />

 
### 5.6 Threat-intelligence feeds
 
The default list (StevenBlack hosts) handles ads and trackers. I added two security-focused feeds:
 
| Feed | Covers | Notes |
|---|---|---|
| abuse.ch URLhaus (hostfile) | Hosts currently serving malware | 427 domains at the time of the first gravity update, changes constantly |
| Phishing Army (extended) | Phishing domains | About 158,800 entries in the file header on the day I added it |
 
Total domains on lists afterwards: **218,013**.
 
**Loading a feed:** adding a list in the web interface does nothing until gravity is updated. After adding URLhaus its status icon was a grey question mark, and after `sudo pihole -g` it showed a green tick. Pi-hole also keeps the previous database available as a fallback if an update fails.
 
**Why two security feeds:** they cover different threat types, and lists overlap less than people expect. `pihole -q <domain>` shows which list matched, so each block can be attributed to a source.
 
<img width="1242" height="706" alt="37" src="https://github.com/user-attachments/assets/c890675b-811d-4e5f-9acb-aaf7a0b39924" />
<img width="906" height="734" alt="38" src="https://github.com/user-attachments/assets/103a65b2-96a2-4614-9a27-10897ba6a671" />
<img width="1557" height="321" alt="43 green tick" src="https://github.com/user-attachments/assets/97c4f9f1-0797-4b3c-9de4-a3c6e9aebb95" />

---
 
## 6. Change record: moving DHCP to the Pi
 
**Reason:** until now only my PC used the Pi, because I had set its DNS by hand. The router hands out its own DNS servers to every other device and, on this model, does not let me change them. Making Pi-hole the DHCP server means every device gets the Pi as its resolver automatically, and the log shows devices by name.
 
**Before:**
- Router DHCP server enabled, range starting at 192.168.0.10, 245 addresses, 24-hour lease
- Pi on a Hub-reserved address (192.168.0.205)
**Preparation:**
1. Gave the Pi its own static address (NetworkManager, manual method, 192.168.0.205/24, gateway 192.168.0.1), because a DHCP server cannot depend on getting its own address from DHCP. The Pi's own DNS points at Quad9, so it does not depend on itself to resolve names.
2. Verified the address with `ip -4 addr` and `ip route`, rebooted, and confirmed the static address persisted and Pi-hole restarted by itself.
3. Chose a Pi-hole DHCP range of 192.168.0.10 to 192.168.0.200, which keeps `.205` outside the pool, with the gateway set to 192.168.0.1 and a 24-hour lease. IPv6 support, rapid commit, repeated DNS advertisement and "ignore unknown clients" all stayed off.
4. Checked the Hub's device list: nothing other than the Pi sat above `.200`.
**Sequence:** applying the change on the Hub restarts it, and the Hub warned that this takes about five minutes. That changed my plan, so I saved and enabled DHCP on Pi-hole first and then disabled the Hub's DHCP server, which meant only one DHCP server was live when the Hub came back.
 
**After:**
- Hub DHCPv4 server: Disabled
- Pi-hole's lease table lists devices with hostnames and expiry (about a day), including my PC, my phone and a streaming device
- My PC, reset to automatic DNS and renewed, reports the Pi as its DNS server and `nslookup` answers from `pi.hole`
**Open point:** the PC's `DHCP Server` line still showed 192.168.0.1 after the renewal, while the lease table showed Pi-hole issuing leases. I believe this is a display or caching quirk, but I did not prove it.
 
**Risks accepted:**
- The Pi is now a single point of failure for DNS and addressing. Devices with a current lease keep working for up to 24 hours if it goes down, but new devices get no address.
- Rollback means re-enabling DHCP on the Hub (another five-minute restart) and switching Pi-hole's DHCP off.
**Evidence gap:** the change window arrived faster than planned, so I did not capture screenshots during the restart. The sequence above is reconstructed from the before and after states and the timestamps.
 
<img width="990" height="753" alt="42" src="https://github.com/user-attachments/assets/242b5b2f-9cb3-496d-8098-5fe56fd9e636" />
<img width="297" height="292" alt="Screenshot 2026-10-15 084246" src="https://github.com/user-attachments/assets/c5ebe26c-fb3e-4571-bb37-9738791cc3b9" />
 
---
 
## 7. Results from the first eight hours
 
Captured on the morning of 15 October 2026.
 
| Metric | Value |
|---|---|
| Total queries | 13,979 |
| Blocked | 9,047 (64.7%) |
| Domains on lists | 218,013 |
| Active clients | 8 |
 
**The headline percentage is misleading.** Two AppLovin domains account for about 7,990 of the 9,047 blocked queries (5,309 for `ms.applovin.com` and 2,677 for `ms.applvn.com`), and one phone made 8,697 of all blocked queries. Without those two domains, roughly 1,060 of about 5,990 queries were blocked, around 18%.
 
The lesson is that raw totals are dominated by whichever device or app is noisiest, so I break numbers down by client and domain before drawing conclusions.
 
<img width="1000" height="797" alt="44" src="https://github.com/user-attachments/assets/85c2e837-939a-4cbd-ae4f-8975f6bb22db" />
<img width="816" height="765" alt="45" src="https://github.com/user-attachments/assets/4e2f56b2-54bb-49d3-94fa-e9ef70861ce4" />

---
 
## 8. Investigation notes
 
Each note follows the same format: summary, timeline, evidence, analysis, verdict, action. Times are as shown in the Pi's logs.
 
### 8.1 Known-bad domain blocked and attributed (`6sim.vip`)
 
**Summary:** a lookup for a domain taken from the URLhaus feed was blocked, and the block was attributed to that feed.
 
**Timeline (14 Oct 2026):**
- 22:29:39: A and AAAA lookups for `6sim.vip` from my PC, both blocked (gravity)
- 22:43:54: the same lookups again after my PC began using the Pi as its default DNS, both blocked
**Evidence:** `nslookup` returned `0.0.0.0` and `::`. `pihole -q 6sim.vip` found zero manual domain rules and exactly one matching list, the URLhaus hostfile.
 
**Analysis:** this was a deliberate test, so the lookup was expected. The block came from the threat feed, not from the ad list, and the answer was a null address, so no connection to the real host was made. A DNS lookup alone does not contact the destination.
 
**Verdict:** true positive on a deliberate test. The control works end to end: feed, gravity update, DNS answer and log.
 
**If it had been unprompted:** I would have identified the client from the lease table, found out which process or app made the request, and checked what else that device had looked up around the same time.
 
<img width="1021" height="381" alt="40" src="https://github.com/user-attachments/assets/ce1450f2-f927-4b58-ab91-3985a11d3bdf" />
<img width="588" height="167" alt="41" src="https://github.com/user-attachments/assets/52d7da5c-df4b-4635-8c86-41b6122cc02d" />

### 8.2 Safe test indicator used to validate a feed (`amiprotected.it`)
 
**Summary:** I validated the Phishing Army feed without touching a real phishing site.
 
**Evidence:** the feed's own header says it deliberately includes two harmless test domains for checking that filtering works. `nslookup amiprotected.it` returned `0.0.0.0` and `::`, and `pihole -q` matched exactly one list, Phishing Army. The Lists page showed all three lists enabled with green ticks, and total domains on lists rose to 218,013.
 
**Analysis:** this is the same idea as the EICAR test file for antivirus: an indicator that is safe to trigger and exists to prove the detection path works.
 
**Verdict:** control validated. Expected result, no action needed.

 <img width="1557" height="702" alt="43" src="https://github.com/user-attachments/assets/99b8df9b-7d00-4fad-8231-6caab0496fbc" />
 
### 8.3 Regular lookups to an ad network (benign beaconing-like pattern)
 
**Summary:** one device sent thousands of lookups to a single domain at a steady rhythm. That pattern deserves attention, and the context shows it is benign.
 
**Timeline:**
- About 04:00 to 05:30 on 15 Oct: a large red spike of blocked queries on the dashboard
- 05:44:55 to 05:46:05: roughly 35 lookups for `ms.applovin.com` from my phone, spaced about two seconds apart, every one blocked
**Evidence:**
- Dashboard: 5,309 hits for `ms.applovin.com`, 2,677 for `ms.applvn.com`, plus smaller counts for other AppLovin and Unity analytics domains. The phone made 8,697 of the blocked queries.
- The filtered Query Log (client and domain selected) shows evenly spaced rows and runs to about 11 pages of 500 entries, which agrees with the dashboard count.
- I was playing a mobile game at the time, and I saw no ads in it while the phone was on home Wi-Fi.
- Switching the phone to mobile data brought the ads back, and switching back to Wi-Fi removed them again. I repeated it both ways.
**Analysis:** fixed-interval requests from one source to one destination are the shape analysts look for when hunting malware beaconing. What separates this case from malware:
 
| Question | This case | Malware would more likely |
|---|---|---|
| Domain | Established ad network | Be new, obscure or algorithm-generated |
| Timing | Almost no variation | Often add random jitter (not always) |
| Tied to user activity | Only while the game was running | Continue regardless of what the user is doing |
| Result | Blocked, no connection made | Succeed and carry data |
 
My conclusion is that the game's ad code kept retrying after each refusal, with no backoff.
 
**Caveats:**
- DNS logs show what was requested and when, not which app asked. I inferred the game from the timing and from what I was doing.
- Switching to mobile data changes more than the DNS server (network, address, location), so it is strong evidence but not airtight. Disabling blocking for a few minutes would isolate the variable more cleanly, and I did not run that test.
**Verdict:** benign, an ad SDK retry loop. High confidence it is not malicious, moderate confidence that the game is the source.
 
**Action:** no allow rule added, because allowing the domain would only bring the ads back. If the retry loop mattered, limiting the app's background activity is the better fix.
 
<img width="804" height="763" alt="Screenshot 2026-10-15 085038" src="https://github.com/user-attachments/assets/6a0138fa-15e5-4081-9489-0892793a069b" />
<img width="1000" height="797" alt="44" src="https://github.com/user-attachments/assets/43f724ba-690c-4a08-a8ce-4a37d215730a" />
<img width="816" height="765" alt="45" src="https://github.com/user-attachments/assets/c57a3e9f-4011-4d7f-ba43-8c48d07544ca" />
 
---
 
## 9. Findings and limitations
 
**What DNS filtering does well here**
- Blocks known ad, tracker, malware and phishing domains for every device that uses it
- Gives per-device, named logs once Pi-hole is the DHCP server
- Provides a cheap, safe way to practise reading real DNS telemetry
**What it cannot do, or where I have not checked**
- **Ads served from the same domain as the content.** YouTube ads are the clearest example. I first credited Pi-hole for ad-free YouTube on my PC, but my browser's own built-in blocking was responsible. The phone, which only has DNS filtering, still sees YouTube ads.
- **Coverage ends at the network edge.** My phone is only filtered on home Wi-Fi. On mobile data it bypasses the Pi entirely.
- **Single point of failure.** One Pi serves DNS and DHCP.
- **Upstream DNS is unencrypted.** Allowed lookups reach Quad9 on port 53, so my ISP can still see the domains.
- **Bypass routes I have not tested:** browser or OS secure-DNS settings (DNS over HTTPS), Android's Private DNS, and devices with hard-coded resolvers. I also have not confirmed whether the router advertises its own IPv6 DNS, although my PC showed no IPv6 DNS server.
- **Dashboard certificate.** The web interface uses a self-signed certificate, so browsers warn about it.
- **List quality.** Feeds age and can contain false positives. Pi-hole refreshes lists weekly by default, which is slow for feeds that change within hours.
- **Local, short-lived logs.** Queries are stored on the Pi only, and I have not verified how long the system logs survive a reboot.
**Not yet investigated**
- An unnamed client at 192.168.0.180
- 57 lookups for `wpad.lan`, which should return "not found" on a clean network
- Whether the Fire TV-style device bypasses the Pi for some lookups
**Smaller checks that passed**
- Pi-hole's diagnosis page raised one NTP warning. `timedatectl` showed the system clock synchronised, so I treated it as transient.
- `pihole status` run without `sudo` warns that it cannot read `pihole.toml`. That is a permissions detail, and I left the file locked down.
---
 
## 10. Lessons learned
 
1. **Verify before use, not after.** I hashed the OS image after flashing. A pre-write check is what actually prevents a bad image from being used.
2. **Know which machine you are typing on.** I ran my first key-generation commands inside the SSH session, which created the key pair on the Pi, the wrong place for a private key. The prompt and the file path gave it away, I deleted the pair and redid it on my PC.
3. **A setting is not proof.** My PC's DNS change had not applied to the active adapter (probably saved against the disconnected Wi-Fi adapter), and `ipconfig /all` showed it. When I then tried the PowerShell route, it failed with a permissions error because the window was not elevated. Effective-state checks caught both.
4. **Test your assumption about which control acted.** I credited Pi-hole for something my browser did. Two overlapping controls make it hard to attribute an outcome without a test.
5. **Break headline numbers down.** One app inflated the blocked percentage from about 18% to 64.7%.
6. **Capture the baseline first.** The listening-port baseline taken before the install made the after state meaningful.
7. **Prepare a screenshot checklist before a change window.** The router restart arrived faster than I planned and I missed evidence during it.
8. **Use safe test indicators.** `amiprotected.it` validated a feed without visiting anything dangerous.
---
 
## 11. Roadmap
 
- **Forward logs to Microsoft Sentinel** and build a month-long simulated SOC exercise on top of this lab, with a shift-style review routine and a triage journal
- **Detections:** a rule that flags clients sending many queries to one domain at near-constant intervals, a first-seen-device alert from DHCP leases, and a failed-SSH-login rule
- **Encrypted upstream DNS** (DNS over TLS or HTTPS) to remove the plain-DNS limitation
- **Resilience:** a second resolver, and a written recovery runbook for the DHCP rollback
- **Coverage:** a VPN back to the home network so phones stay filtered away from home
- **Feed hygiene:** daily refresh for the threat feeds, and an allowlist register with a reason and date for every exception
---
 
## Appendix
 
### Key commands
 
```
Get-FileHash <file> -Algorithm SHA256          # integrity check (Windows)
ssh-keygen -t ed25519                          # key pair, run on the client
sudo sshd -t                                   # validate SSH config syntax
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|pubkeyauthentication"
sudo ss -tlnp                                  # listening services (baseline)
sudo journalctl -u ssh --no-pager | tail -20   # SSH authentication log
sudo pihole -g                                 # update gravity (rebuild blocklists)
sudo pihole -q <domain>                        # which list matched a domain
pihole status                                  # service health
nslookup <domain> <pi-address>                 # test a lookup against the Pi
```
 
### Sources and licences
 
- Raspberry Pi OS and Raspberry Pi Imager: raspberrypi.com
- Pi-hole: pi-hole.net
- Quad9: quad9.net
- StevenBlack hosts: github.com/StevenBlack/hosts
- abuse.ch URLhaus: urlhaus.abuse.ch (check abuse.ch's terms of use)
- Phishing Army: phishing.army (free for personal use, see the licence page linked in the list header)


  
