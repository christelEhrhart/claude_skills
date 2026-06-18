---
name: owasp-audit
description: >
  Auditer un fichier, un module ou un bout de code PHP / CI4 selon les critères OWASP Top 10 2025.
  Utiliser ce skill quand on demande de relire du code pour trouver des failles de sécurité,
  faire un audit de sécurité, chercher des vulnérabilités, vérifier si du code est sécurisé,
  ou produire un rapport d'audit.
  Mots-clés déclencheurs : audit sécurité, relire le code sécurité, vulnérabilité dans ce code,
  ce code est-il sécurisé, faille, pentest, revue de sécurité, rapport sécurité.
---

# Audit de sécurité — PHP / CodeIgniter 4

> Méthode d'analyse basée sur OWASP Top 10 2025.
> Pour les corrections, consulter le skill **owasp-secure-code**.

---

## Méthode d'audit

Quand du code est soumis pour audit, l'analyser dans cet ordre :

### 1. Identifier le périmètre
- Type de fichier : contrôleur / modèle / vue / config / migration / CI/CD ?
- Données manipulées : utilisateur, paiement, fichier, session, API ?
- Exposition : endpoint public, privé (auth), admin ?

### 2. Appliquer la grille OWASP

Pour **chaque point de la grille**, indiquer :
- ✅ **Conforme** — avec la ligne de code concernée
- ⚠️ **À améliorer** — avec explication et correction proposée
- ❌ **Vulnérabilité** — avec niveau de criticité (Critique / Élevé / Moyen / Faible) et PoC

### 3. Produire le rapport

---

## Grille d'analyse — 10 axes OWASP 2025

### A01 — Contrôles d'accès (+ SSRF)
```
□ Les routes sont-elles protégées par un filtre auth ?
□ Le contrôleur vérifie-t-il le rôle de l'utilisateur avant l'action ?
□ L'isolation tenant est-elle appliquée dans les requêtes ?
□ Les IDs en URL ne permettent pas d'accéder aux données d'un autre utilisateur ?
□ Les actions destructives (delete, update) vérifient-elles l'appartenance de la ressource ?
□ L'application effectue-t-elle des requêtes HTTP vers des URLs fournies par l'utilisateur ?
□ Ces URLs sont-elles validées contre une liste blanche ? Les IPs privées sont-elles bloquées ?
```

**Patterns à rechercher :**
```php
// ❌ Vulnérable — pas de vérification d'appartenance
public function delete(int $intId): RedirectResponse
{
    $this->objModel->delete($intId);   // n'importe quel utilisateur peut supprimer !
}

// ❌ Vulnérable — vérification côté client uniquement
if ($_POST['is_admin'] == 1) { ... }

// ✅ Correct
$objItem = $this->objModel->where('items_user_id', auth()->id())->find($intId);
if ($objItem === null) throw new PageNotFoundException();

// ❌ SSRF — appel d'URL utilisateur sans validation
file_get_contents($_POST['url'])
curl_setopt($ch, CURLOPT_URL, $strUserUrl)   // sans liste blanche
```

---

### A02 — Configuration de sécurité
```
□ CI_ENVIRONMENT est-il en 'production' (pas 'development') ?
□ Les headers de sécurité sont-ils configurés ?
□ Le mode debug est-il désactivé ?
□ Les credentials sont-ils dans .env (pas dans le code) ?
□ Y a-t-il des tokens, clés API ou mots de passe en dur dans le code ?
□ CORS est-il correctement restreint ?
□ Les fonctionnalités inutilisées sont-elles désactivées ?
```

**Patterns à rechercher :**
```php
// ❌ Critique — credentials en dur
$strPassword = 'admin123';
$strApiKey   = 'sk-abc123def456';
define('DB_PASS', 'root');

// ❌ Élevé — debug en production
CI_ENVIRONMENT = development

// ❌ Élevé — CORS trop permissif
header('Access-Control-Allow-Origin: *');
```

---

### A03 — Chaîne d'approvisionnement (Supply Chain)
```
□ Le composer.lock est-il présent, versionné et à jour ?
□ Des versions de packages avec CVE connues sont-elles utilisées ?
□ La version de PHP est-elle maintenue (>= 8.2) ?
□ Les packages de dev sont-ils exclus en production (--no-dev) ?
□ Les noms de packages sont-ils vérifiés (typosquatting) ?
□ Le pipeline CI/CD est-il sécurisé (accès restreint, audit des actions) ?
□ Les secrets CI/CD sont-ils dans des variables d'environnement chiffrées ?
```

**Points à vérifier :**
```bash
# Audit des dépendances
composer audit
composer show --outdated

# Vérifier les permissions sur le pipeline CI/CD
# Vérifier que les secrets ne sont pas loggés en clair
# Vérifier l'intégrité des actions GitHub utilisées (pin sur commit SHA)
```

---

### A04 — Cryptographie
```
□ Les mots de passe sont-ils hachés avec bcrypt/Argon2 (Shield) ?
□ Les données sensibles sont-elles stockées en clair en BDD ?
□ Des algorithmes obsolètes sont-ils utilisés (md5, sha1, base64 pour mots de passe) ?
□ Les cookies sont-ils Secure + HttpOnly ?
□ Les communications utilisent-elles HTTPS ?
□ TLS 1.0/1.1 sont-ils désactivés ?
```

**Patterns à rechercher :**
```php
// ❌ Critique
md5($strPassword)
sha1($strPassword)
base64_encode($strPassword)

// ❌ Élevé
INSERT INTO users SET password='$strPassword'  // en clair
```

---

### A05 — Injection
```
□ Toutes les requêtes SQL utilisent-elles le Query Builder ou des prepared statements ?
□ Y a-t-il des concaténations de variables dans des requêtes SQL ?
□ Les sorties dans les templates sont-elles échappées (|escape) ?
□ Les entrées utilisateur sont-elles validées avant traitement ?
□ Des commandes shell sont-elles exécutées avec des données utilisateur ?
```

**Patterns à rechercher :**
```php
// ❌ Critique — SQL Injection
"SELECT * FROM users WHERE id = $intId"
"WHERE name = '" . $strName . "'"
$this->db->query("SELECT * FROM $strTable")

// ❌ Élevé — XSS
echo $_GET['name'];
echo $strUserInput;  // sans htmlspecialchars

// ❌ Critique — Command Injection
exec("convert " . $strFilename);
shell_exec($_POST['cmd']);
```

---

### A06 — Conception
```
□ Y a-t-il un rate limiting sur les endpoints sensibles (login, reset, API) ?
□ Les limites métier sont-elles vérifiées côté serveur ?
□ Des données sensibles sont-elles exposées dans les messages d'erreur ?
□ Les messages d'erreur révèlent-ils la structure interne (noms de tables, chemins) ?
```

**Patterns à rechercher :**
```php
// ❌ Révèle la structure interne
catch (\Exception $e) {
    echo $e->getMessage();   // peut révéler chemin, requête SQL, etc.
}

// ❌ Pas de limite de tentatives
public function login() { /* aucun rate limiting */ }
```

---

### A07 — Authentification
```
□ Shield est-il utilisé (pas d'authentification maison) ?
□ La longueur minimale des mots de passe est-elle définie (>= 12) ?
□ Les sessions sont-elles invalidées après déconnexion ?
□ Les tokens de reset sont-ils aléatoires, uniques et à expiration courte ?
□ Le MFA est-il disponible pour les comptes sensibles ?
```

**Patterns à rechercher :**
```php
// ❌ Authentification maison fragile
if ($strPassword == $_POST['password']) { ... }
if (md5($_POST['password']) == $strStoredHash) { ... }

// ❌ Session non invalidée
public function logout() {
    session()->remove('user_id');  // insuffisant
    // manque : session()->destroy() + regenerate()
}
```

---

### A08 — Intégrité
```
□ La fonction unserialize() est-elle utilisée avec des données non fiables ?
□ Les fichiers uploadés sont-ils vérifiés par type MIME réel ?
□ Les fichiers uploadés sont-ils stockés hors du répertoire public ?
□ Les noms de fichiers uploadés sont-ils renommés ?
□ Les dépendances ont-elles des checksums vérifiés (composer.lock) ?
```

**Patterns à rechercher :**
```php
// ❌ Critique
unserialize($_COOKIE['data'])
unserialize(base64_decode($_POST['object']))

// ❌ Élevé — vérification par extension seulement
$strExt = pathinfo($strFilename, PATHINFO_EXTENSION);
if ($strExt === 'pdf') { ... }   // contournable

// ❌ Élevé — upload dans le répertoire public
move_uploaded_file($strTmp, FCPATH . 'uploads/' . $strName);
```

---

### A09 — Journalisation et alerte
```
□ Les connexions échouées sont-elles loggées avec IP + timestamp ?
□ Les accès refusés sont-ils loggés ?
□ Les actions sensibles (suppression, changement de rôle) sont-elles tracées ?
□ Les logs contiennent-ils assez de contexte (IP, user, action) ?
□ Les logs sont-ils stockés de manière sécurisée ?
□ Des alertes automatiques sont-elles configurées sur les anomalies ?
```

---

### A10 — Gestion des conditions exceptionnelles
```
□ Les blocs catch affichent-ils des messages d'erreur génériques à l'utilisateur ?
□ Les exceptions sont-elles loggées en interne (pas en clair en sortie) ?
□ Y a-t-il des blocs catch vides (exception silencieuse) ?
□ Les états de transition (paiement, validation) sont-ils vérifiés côté serveur ?
□ Les cas limites (null, 0, chaîne vide, valeur négative) sont-ils gérés ?
□ Le comportement en cas d'erreur est-il fail-safe (refus par défaut) ?
```

**Patterns à rechercher :**
```php
// ❌ Critique — révèle la structure interne
catch (\Exception $e) {
    echo $e->getMessage();   // stack trace, nom de tables, chemins
    // ou
    return $this->response->setJSON(['error' => $e->getMessage()]);
}

// ❌ Élevé — exception silencieuse
try {
    $this->verifyPermission($intUserId);
} catch (\Exception $e) {
    // rien — la vérification est ignorée silencieusement
}

// ❌ Élevé — pas de vérification d'état cohérent
public function confirmOrder(int $intOrderId): void
{
    // manque : vérifier que le paiement est bien validé avant confirmation
    $this->objOrderModel->update($intOrderId, ['status' => 'confirmed']);
}

// ❌ Moyen — cas limite non géré
$fltPrixUnitaire = $this->getPrix($intProduitId);
$fltTotal = $fltMontant / $fltPrixUnitaire;   // division par zéro si prix = 0

// ✅ Correct
catch (\Exception $e) {
    log_message('error', '[SECURITY] ' . $e->getMessage());
    return redirect()->back()->with('strError', 'Une erreur est survenue.');
}
```

---

## Template de rapport d'audit

```markdown
# Rapport d'audit de sécurité
**Fichier :** `app/Modules/Crm/Controllers/ContactsController.php`
**Date :** 2025-01-15
**Référentiel :** OWASP Top 10 2025

## Résumé
| Niveau      | Nombre |
|-------------|--------|
| 🔴 Critique  | 0      |
| 🟠 Élevé     | 1      |
| 🟡 Moyen     | 2      |
| 🟢 Faible    | 1      |

## Vulnérabilités détectées

### 🟠 [A05] Injection SQL — Élevé
**Ligne :** 45
**Code vulnérable :**
```php
$arrData = $this->db->query("SELECT * FROM crm_contacts WHERE contacts_name = '$strName'");
```
**Risque :** Injection SQL permettant de lire, modifier ou supprimer des données.
**Correction :**
```php
$arrData = $this->db->table('crm_contacts')->where('contacts_name', $strName)->get()->getResult();
```

### 🟡 [A10] Exception silencieuse — Moyen
**Ligne :** 78
**Code vulnérable :**
```php
try {
    $this->verifyOwnership($intId);
} catch (\Exception $e) { }
```
**Risque :** La vérification d'appartenance peut être contournée silencieusement.
**Correction :** Logger l'exception et interrompre le traitement.

## Points conformes
- ✅ A01 : Filtre auth présent sur toutes les routes
- ✅ A07 : Shield utilisé pour l'authentification
- ✅ A04 : Pas d'algorithme de hachage obsolète
- ✅ A03 : composer.lock présent et composer audit sans CVE critique
```
