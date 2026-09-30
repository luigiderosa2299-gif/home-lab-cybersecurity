# Evidenze raccolte — Home Lab Setup

## Scan 1 — Nmap SYN scan (default), risultato iniziale ambiguo

```
$ nmap -sV -Pn 192.168.1.95 -p 8080,22

PORT     STATE    SERVICE    VERSION
22/tcp   open     ssh        OpenSSH 10.5p1 Debian 1 (protocol 2.0)
8080/tcp filtered http-proxy
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Scan 2 — Nmap TCP connect scan, conferma servizio attivo

```
$ nmap -sT -sV -p 8080,22 192.168.1.95

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.5p1 Debian 1 (protocol 2.0)
8080/tcp open  http    Apache httpd 2.4.68 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Verifica applicativa DVWA (locale, dentro la VM Kali)

```
$ curl -sI http://127.0.0.1:8080/setup.php

HTTP/1.1 200 OK
Server: Apache/2.4.68 (Debian)
X-Powered-By: PHP/8.5.10
Set-Cookie: security=impossible; path=/; HttpOnly
Content-Type: text/html;charset=utf-8
```

## Verifica applicativa DVWA (dalla rete, dal Mac host)

```
$ curl -sI http://192.168.1.95:8080/setup.php

HTTP/1.1 200 OK
Server: Apache/2.4.68 (Debian)
X-Powered-By: PHP/8.5.10
Set-Cookie: security=impossible; path=/; HttpOnly
Content-Type: text/html;charset=utf-8
```

## Stato host Kali al momento del setup

```
$ hostnamectl
Static hostname: kali
Operating System: Kali GNU/Linux Rolling
Kernel: Linux 7.1.5+kali-arm64
Architecture: arm64
Virtualization: qemu

$ nproc
4

$ free -h
              total   used   free   shared  buff/cache  available
Mem:          3.8Gi   601Mi  2.7Gi  13Mi     692Mi       3.2Gi
Swap:         2.1Gi   0B     2.1Gi

$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda3        37G   21G   15G  59% /
```
