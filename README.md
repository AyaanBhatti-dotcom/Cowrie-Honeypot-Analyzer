## Cowrie Honeypot — SSH Attack Analysis

![Honeypot Analysis](https://github.com/user-attachments/assets/ca70f613-1e6b-446a-bada-eb8530cde14d)

![Platform](https://img.shields.io/badge/Honeypot-Cowrie-blue)
![Cloud](https://img.shields.io/badge/Cloud-DigitalOcean-0080FF?logo=digitalocean)
![Analysis](https://img.shields.io/badge/Analysis-Claude%20Code-black?logo=anthropic)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

---

I deployed a Cowrie honeypot on a DigitalOcean VPS to watch what actually happens when
you leave an SSH port exposed. Cowrie simulates an SSH server, logs everything, and lets
you observe intrusion attempts without risking a real system.

It got hit within minutes. Over 17 hours, 60 unique IPs ran credential sprays and
brute-force attempts. Most were fully automated scanners cycling through common passwords.
A smaller cluster was more deliberate and they logged in, stripped file protections with
`chattr`, and injected a backdoor SSH key. Same key, same script, 28 sessions across
20+ IPs. That is not coincidence.

One of those campaigns was mdrfckr which is a botnet that has been showing up in honeypot logs
since at least 2018. It gets its name from the comment field left in the SSH public key
it injects into `authorized_keys`. The key itself is identical across hundreds of attacks
reported by researchers worldwide. It has been linked to Monero cryptomining operations
and is one of the most consistently observed SSH backdoor campaigns in public honeypot
data. The fact that it showed up here within hours of the VPS going live says a lot about
how aggressive and widespread it is.

To analyze the logs, I used Claude Code with a custom forensic prompt against the raw
`cowrie.txt` output. It grouped sessions by IP, mapped the two attacker clusters,
extracted every IOC, and correlated the shared tooling across hosts automatically.
The full report is in [`analysis.md`](./analysis.md).

---
