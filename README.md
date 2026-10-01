# Home Lab — Cybersecurity Practice Environment

## 📌 Obiettivo
Ambiente di laboratorio personale per esercitarmi in modo pratico sui concetti studiati per la certificazione CompTIA Security+ (SY0-701): network scanning, vulnerability assessment, hardening e — in futuro — incident response e Active Directory security. Questo repository documenta l'infrastruttura del lab e serve da base per i progetti successivi del portfolio.

## 🧩 Contesto / Scenario
Lab virtualizzato su un singolo host macOS (Apple Silicon), con macchine virtuali gestite tramite UTM (QEMU) in bridge sulla rete locale. L'obiettivo è avere un ambiente isolato ma realistico dove poter attaccare/scansionare macchine vulnerabili senza rischi per sistemi reali.

## 🛠 Tecnologie e Strumenti
- **Hypervisor**: UTM (QEMU/HVF) su macOS Apple Silicon (arm64)
- **Attack machine**: Kali Linux Rolling (arm64), accesso SSH con autenticazione a chiave
- **Target vulnerabili** (set ispirato a OWASP Broken Web Applications — OWASP BWA):
  - [DVWA](https://github.com/digininja/DVWA) (Damn Vulnerable Web Application) — porta 8080
  - [WebGoat](https://owasp.org/www-project-webgoat/) — piattaforma didattica OWASP con lezioni guidate passo-passo — porta 8081
  - [Mutillidae II](https://github.com/webpwnized/mutillidae) — copertura quasi completa OWASP Top 10, livelli di difficoltà regolabili — porta 8082
- **Networking**: bridge su interfaccia fisica (vmnet-bridged), IP assegnati via DHCP sulla LAN domestica
- **Tool di analisi**: Nmap, cURL
- **Container runtime**: Docker (`docker.io` da repository Kali) + `qemu-user-static`/`binfmt-support` per emulare immagini amd64 su host arm64

## 🏗 Architettura del Lab

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
│  │  Windows 11 VM   │  (target futuro:         │
│  │  (spenta finché  │   scan, hardening,       │
│  │   non serve)     │   AD lab)                │
│  └─────────────────┘                          │
└─────────────────────────────────────────────┘
```

## ⚙️ Cosa ho fatto

1. **Provisioning VM Kali Linux** su UTM con rete in modalità bridge (`vmnet-bridged`), per ottenere un IP reale sulla LAN e poter simulare traffico realistico tra macchine.
2. **Accesso SSH con chiave pubblica** configurato (niente autenticazione a password), con `sudo` senza password per l'utente di lab — pratica comune in ambienti di test isolati, da NON replicare mai su sistemi in produzione.
3. **Installazione Docker** (`apt-get install docker.io`) su Kali per poter orchestrare target vulnerabili in modo leggero, senza dover creare una VM dedicata per ogni scenario.
4. **Deploy di DVWA** come container Docker (`ghcr.io/digininja/dvwa:latest`), esposto sulla porta 8080. Ho dovuto risolvere un problema di compatibilità architetturale: la prima immagine testata (`vulnerables/web-dvwa`) era compilata solo per `linux/amd64` e falliva con `exec format error` sulla VM Kali arm64 — ho sostituito con l'immagine ufficiale DVWA multi-architettura.
5. **Verifica raggiungibilità** del target sia in locale (`127.0.0.1:8080` dentro la VM) sia dalla rete (`192.168.1.95:8080` dal Mac host), confermando che il container è correttamente esposto.
6. **Primo scan di rete con Nmap** dalla VM Kali verso se stessa, per validare i servizi esposti.

## 📊 Risultati

Scan Nmap (TCP connect + service detection) eseguito dalla VM Kali:

```
$ nmap -sT -sV -p 8080,22 192.168.1.95

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.5p1 Debian 1 (protocol 2.0)
8080/tcp open  http    Apache httpd 2.4.68 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Verifica applicativa (risposta HTTP di DVWA):

```
$ curl -sI http://192.168.1.95:8080/setup.php
HTTP/1.1 200 OK
Server: Apache/2.4.68 (Debian)
X-Powered-By: PHP/8.5.10
Set-Cookie: security=impossible; path=/; HttpOnly
```

**Nota interessante**: un primo scan con `nmap -Pn -sV` (SYN scan di default) segnalava la porta 8080 come `filtered`, mentre il servizio era in realtà pienamente raggiungibile via HTTP. Passando a `-sT` (TCP connect scan) il risultato è coerente (`open`). Questo è un promemoria pratico di quanto studiato nel modulo Vulnerability Scanning del corso Security+: tecniche di scan diverse possono dare risultati diversi sullo stesso target, ed è importante capire il perché prima di trarre conclusioni su un servizio.

## 💡 Cosa ho imparato
- Le immagini Docker non sono automaticamente portabili tra architetture: su hardware ARM (Apple Silicon) serve verificare la compatibilità `arm64`/`amd64` prima di scegliere un'immagine, o si rischia un fallimento silenzioso (`exec format error`).
- La differenza pratica tra SYN scan e TCP connect scan di Nmap non è solo teorica: in ambienti containerizzati/NAT può produrre risultati diversi (`filtered` vs `open`) sullo stesso servizio realmente attivo, e capire la causa è più importante del semplice numero di porte aperte.
- Usare Docker per i target vulnerabili invece di intere VM dedicate riduce drasticamente i tempi di setup e il consumo di risorse, mantenendo comunque un ambiente di attacco realistico.

## 🔜 Prossimi passi
- [x] Estensione del lab con bundle stile OWASP BWA (WebGoat + Mutillidae II accanto a DVWA)
- [ ] Vulnerability assessment completo su DVWA (OWASP Top 10) → repository dedicato
- [ ] Primi laboratori guidati su WebGoat (SQL injection, XSS, autenticazione)
- [ ] Accensione della VM Windows 11 come target di scan/hardening
- [ ] SIEM leggero (Wazuh) per raccogliere i log delle attività del lab

## 💡 Cosa ho imparato (aggiornamento)
- OWASP BWA originale è distribuita come immagine VM (OVA) compilata solo per x86, quindi non è utilizzabile direttamente su un host Apple Silicon: la soluzione più pratica è ricostruire lo stesso set di applicazioni vulnerabili come container Docker indipendenti, ottenendo lo stesso valore didattico senza i problemi di compatibilità.
- Quando un'immagine Docker resta disponibile solo per `amd64` (come Mutillidae II) e non esiste un'alternativa arm64 nativa, installare `qemu-user-static` e `binfmt-support` permette a Docker di emulare l'architettura x86 trasparentemente — più lento della nativa ma pienamente funzionante per un lab didattico.

## 🔗 Collegamenti
- [DVWA — repository ufficiale](https://github.com/digininja/DVWA)
- Percorso di studio CompTIA Security+ (SY0-701) in corso, materiale in vault Obsidian personale
