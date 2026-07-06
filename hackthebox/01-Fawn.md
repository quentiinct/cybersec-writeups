# SQLi 01 — Retrieve Hidden Data

**Plateforme :** Hack The Box
**Catégorie :** Network
**Niveau :** Very Easy
**Date :** 05/07/2026  

---

## Objectif

La machine expose un service réseau mal configuré. L'objectif est d'identifier
le service, d'exploiter une mauvaise configuration d'authentification et de
récupérer le flag.

---

## Vulnérabilité identifiée

Le service FTP est configuré pour accepter les connexions anonymes. N'importe qui
peut se connecter sans credentials valides et accéder aux fichiers du serveur.

---

## Reconnaissance

**Scan nmap :**

```bash
nmap -sV -sC 10.129.X.X
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3

- Port 21 ouvert — FTP (File Transfer Protocol)
- Version : vsftpd 3.0.3
- Le script nmap `ftp-anon` indique : `Anonymous FTP login allowed`

---

## Exploitation

FTP accepte un login anonyme avec username `anonymous` et un mot de passe vide.

```bash
ftp 10.129.X.X
```

Listage des fichiers disponibles :

```bash
ls
```
flag.txt

Lecture du flag :

```bash
cat flag.txt
```

---

## Résultat

Flag récupéré.

---

## Impact

En conditions réelles, un FTP anonyme exposé permettrait de :

- Lire des fichiers sensibles accessibles publiquement.
- Déposer des fichiers malveillants si les droits d'écriture sont activés
- Utiliser le serveur comme point d'appui pour pivoter sur le réseau interne

---

## Remédiation

- Désactiver le login anonyme dans la configuration vsftpd :
- Si le FTP anonyme est nécessaire : restreindre les droits en lecture seule
  sur un répertoire dédié et isolé
- Préférer SFTP ou FTPS qui chiffrent les échanges

---

## Références

- [HackTheBox — Starting Point](https://app.hackthebox.com/starting-point)
- [vsftpd documentation](https://security.appspot.com/vsftpd.html)