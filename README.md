# Home Lab — Cybersecurity Practice Environment

## 📌 Objective
A personal lab environment for hands-on practice of the concepts studied for the CompTIA Security+ (SY0-701) certification: network scanning, vulnerability assessment, hardening and — in the future — incident response and Active Directory security. This repository documents the lab infrastructure and serves as the foundation for the later projects in the portfolio.

## 🧩 Context / Scenario
Virtualized lab on a single macOS host (Apple Silicon), with virtual machines managed through UTM (QEMU) bridged onto the local network. The goal is an isolated but realistic environment where I can attack/scan vulnerable machines without risk to real systems.

## 🛠 Technologies and Tools
- **Hypervisor**: UTM (QEMU/HVF) su macOS Apple Silicon (arm64)
- **Attack machine**: Kali Linux Rolling (arm64), SSH access with key-based authentication
- **Vulnerable targets** (set inspired by OWASP Broken Web Applications — OWASP BWA):
  - [DVWA](https://github.com/digininja/DVWA) (Damn Vulnerable Web Application) — port 8080
  - [WebGoat](https://owasp.org/www-project-webgoat/) — OWASP training platform with step-by-step guided lessons — port 8081
  - [Mutillidae II](https://github.com/webpwnized/mutillidae) — near-complete OWASP Top 10 coverage, adjustable difficulty levels — port 8082
- **Networking**: bridge on the physical interface (vmnet-bridged), IPs assigned via DHCP on the home LAN
- **Analysis tools**: Nmap, cURL
- **Container runtime**: Docker (`docker.io` from the Kali repository) + `qemu-user-static`/`binfmt-support` to emulate amd64 images on an arm64 host

## 🏗 Lab Architecture

```
┌─────────────────────────────────────────────┐
│              Rete locale (LAN)               │
│              192.168.1.0/24                  │
│                                               │
│  ┌──────────────┐        ┌─────────────────┐ │
│  │  macOS Host  │        │  Kali Linux VM  │ │
│  │  (UTM/QEMU)  │◄──SSH──┤  192.168.1.95   │ │
│  └──────────────┘        │                 │ │
│                           │  ┌───────────┐  │ │
│                           │  │  Docker   │  │ │
│                           │  │ ┌───────┐ │  │ │
│                           │  │ │ DVWA  │ │  │ │
│                           │  │ │ :8080 │ │  │ │
│                           │  │ └───────┘ │  │ │
│                           │  │ ┌───────┐ │  │ │
│                           │  │ │WebGoat│ │  │ │
│                           │  │ │ :8081 │ │  │ │
│                           │  │ └───────┘ │  │ │
│                           │  │ ┌───────┐ │  │ │
│                           │  │ │Mutilli│ │  │ │
│                           │  │ │dae:8082│ │  │ │
│                           │  │ └───────┘ │  │ │
│                           │  └───────────┘  │ │
│                           └─────────────────┘ │
│                                               │
│  ┌─────────────────┐                          │
│  │  Windows 11 VM   │  (future target:         │
│  │  (powered off    │   scan, hardening,       │
│  │   until needed)  │   AD lab)                │
│  └─────────────────┘                          │
└─────────────────────────────────────────────┘
```

## ⚙️ What I Did

1. **Provisioned the Kali Linux VM** on UTM with bridged networking (`vmnet-bridged`), to get a real IP on the LAN and be able to simulate realistic traffic between machines.
2. **Configured SSH access with a public key** (no password authentication), with passwordless `sudo` for the lab user — a common practice in isolated test environments, to be NEVER replicated on production systems.
3. **Installed Docker** (`apt-get install docker.io`) on Kali to orchestrate vulnerable targets in a lightweight way, without having to create a dedicated VM for each scenario.
4. **Deployed DVWA** as a Docker container (`ghcr.io/digininja/dvwa:latest`), exposed on port 8080. I had to solve an architecture compatibility problem: the first image I tested (`vulnerables/web-dvwa`) was built only for `linux/amd64` and failed with `exec format error` on the arm64 Kali VM — I replaced it with the official multi-architecture DVWA image.
5. **Verified reachability** of the target both locally (`127.0.0.1:8080` inside the VM) and from the network (`192.168.1.95:8080` from the Mac host), confirming the container is correctly exposed.
6. **First network scan with Nmap** from the Kali VM against itself, to validate the exposed services.

## 📊 Results

Nmap scan (TCP connect + service detection) run from the Kali VM:

```
$ nmap -sT -sV -p 8080,22 192.168.1.95

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.5p1 Debian 1 (protocol 2.0)
8080/tcp open  http    Apache httpd 2.4.68 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Application check (DVWA HTTP response):

```
$ curl -sI http://192.168.1.95:8080/setup.php
HTTP/1.1 200 OK
Server: Apache/2.4.68 (Debian)
X-Powered-By: PHP/8.5.10
Set-Cookie: security=impossible; path=/; HttpOnly
```

**Interesting note**: a first scan with `nmap -Pn -sV` (default SYN scan) reported port 8080 as `filtered`, while the service was in fact fully reachable via HTTP. Switching to `-sT` (TCP connect scan) gave a consistent result (`open`). This is a practical reminder of what I studied in the Vulnerability Scanning module of the Security+ course: different scan techniques can give different results on the same target, and it is important to understand why before drawing conclusions about a service.

## 💡 What I Learned
- Docker images are not automatically portable across architectures: on ARM hardware (Apple Silicon) you need to check `arm64`/`amd64` compatibility before choosing an image, or you risk a silent failure (`exec format error`).
- The practical difference between an Nmap SYN scan and a TCP connect scan is not just theoretical: in containerized/NAT environments it can produce different results (`filtered` vs `open`) on the same genuinely running service, and understanding the cause matters more than the raw number of open ports.
- Using Docker for the vulnerable targets instead of entire dedicated VMs drastically reduces setup time and resource consumption, while still keeping a realistic attack environment.


## 💡 What I Learned (update)
- The original OWASP BWA is distributed as a VM image (OVA) built only for x86, so it cannot be used directly on an Apple Silicon host: the most practical solution is to rebuild the same set of vulnerable applications as independent Docker containers, getting the same learning value without the compatibility problems.
- When a Docker image is only available for `amd64` (like Mutillidae II) and there is no native arm64 alternative, installing `qemu-user-static` and `binfmt-support` lets Docker emulate the x86 architecture transparently — slower than native but fully functional for a learning lab.

## 🔗 Links
- [DVWA — official repository](https://github.com/digininja/DVWA)
- CompTIA Security+ (SY0-701) study path in progress, material kept in my personal Obsidian vault
