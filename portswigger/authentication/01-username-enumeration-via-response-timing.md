# Auth 01 — Username enumeration via response timing

**Plateforme :** PortSwigger Academy  
**Catégorie :** Authentication  
**Niveau :** Practitioner  

---

## Objectif

Le formulaire de connexion laisse fuiter des informations par le **temps de réponse**.
L'objectif est d'énumérer un nom d'utilisateur valide en mesurant ce temps, puis de
brute-forcer son mot de passe pour se connecter au compte. Le tout en contournant une
protection anti-brute-force basée sur l'adresse IP.

---

## Vulnérabilité identifiée

Deux faiblesses se combinent :

1. **Canal auxiliaire temporel (timing side-channel).** Quand le nom d'utilisateur
   est invalide, l'application rejette immédiatement la requête. Quand il est valide,
   elle vérifie réellement le mot de passe (opération de hachage coûteuse). En envoyant
   un mot de passe **très long**, cette vérification prend sensiblement plus de temps :
   le temps de réponse trahit donc les usernames valides.

2. **Protection anti-brute-force contournable.** Le blocage est appliqué par adresse IP,
   mais l'application fait confiance à l'en-tête **`X-Forwarded-For`** pour déterminer
   l'IP du client. En changeant cette valeur à chaque requête, on repart d'un compteur
   neuf et le blocage ne se déclenche jamais.

---

## Reconnaissance

Interception d'une tentative de connexion dans Burp Suite :

```http
POST /login HTTP/1.1
Host: 0a12008903c4c59b8063a30a00c90062.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

En répétant plusieurs connexions échouées depuis la même IP, l'application finit par
répondre :

```
You have made too many incorrect login attempts. Please try again in 1 minute(s).
```

→ Le blocage est bien basé sur l'IP. On teste l'ajout de l'en-tête `X-Forwarded-For`
avec une valeur différente à chaque envoi : le message de blocage disparaît. Le compteur
se base donc sur cet en-tête, qui est contrôlé par le client.

---

## Exploitation

### 1. Énumération du username par le temps de réponse

On envoie la requête vers **Burp Intruder** et on choisit un type d'attaque **Pitchfork**
(les deux listes avancent en parallèle), avec deux positions :

- `X-Forwarded-For: §1§` → valeur incrémentée à chaque requête (contourne le blocage IP)
- `username=§candidat§` → liste des usernames fournie par le lab
- `password=` → un mot de passe **très long** et fixe (plusieurs centaines de caractères)

```http
POST /login HTTP/1.1
Host: 0a12008903c4c59b8063a30a00c90062.web-security-academy.net
X-Forwarded-For: §1§
Content-Type: application/x-www-form-urlencoded

username=§candidat§&password=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

On lance l'attaque et on affiche la colonne **« Response received »** (temps de réponse).
Un username ressort avec un temps nettement supérieur aux autres : c'est le compte valide,
car c'est le seul pour lequel le serveur a réellement haché le mot de passe.

> Username identifié : **`alaska`**

### 2. Brute-force du mot de passe

On relance une attaque **Pitchfork**, cette fois avec le username figé et la liste de
mots de passe :

- `X-Forwarded-For: §1§` → toujours incrémenté pour éviter le blocage
- `username=` → le username trouvé à l'étape précédente
- `password=§candidat§` → liste des mots de passe fournie par le lab

```http
POST /login HTTP/1.1
Host: 0a12008903c4c59b8063a30a00c90062.web-security-academy.net
X-Forwarded-For: §1§
Content-Type: application/x-www-form-urlencoded

username=alaska&password=§candidat§
```

On trie les résultats par **code de statut** : la bonne combinaison renvoie un **302**
(redirection après connexion réussie) au lieu du **200** des échecs.

> Mot de passe identifié : **`nicole`**

### 3. Connexion

On se connecte avec les identifiants récupérés via le formulaire de login. Le lab passe
en **Solved**.

---

## Résultat

Username énuméré via le canal temporel, mot de passe brute-forcé, et accès au compte
obtenu — malgré la protection anti-brute-force, contournée grâce à `X-Forwarded-For`.

---

## Impact

En conditions réelles, cette vulnérabilité permettrait de :

- Confirmer la validité de comptes (emails, identifiants) sans jamais connaître le mot de
  passe, uniquement via une différence de temps de réponse.
- Réduire drastiquement l'espace de recherche d'une attaque par force brute en ciblant
  uniquement des comptes existants.
- Rendre inefficace une protection anti-brute-force mal conçue (basée sur un en-tête que
  le client contrôle), ouvrant la voie au credential stuffing.

---

## Remédiation

- **Uniformiser les temps de réponse** : effectuer une opération de hachage factice même
  quand le username est invalide, afin que le temps de traitement soit constant quel que
  soit le résultat.
- **Ne jamais faire confiance à `X-Forwarded-For`** pour appliquer un rate-limiting :
  s'appuyer sur l'IP réelle de la connexion (au niveau du reverse-proxy de confiance).
- **Messages d'erreur génériques** et identiques pour « username inconnu » et « mot de
  passe incorrect ».
- Mettre en place une limitation de débit robuste, un CAPTCHA après N échecs, et idéalement
  une authentification multifacteur (MFA).

---

## Références

- [PortSwigger — Username enumeration via response timing](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)
- [PortSwigger — Vulnerabilities in password-based login](https://portswigger.net/web-security/authentication/password-based)
