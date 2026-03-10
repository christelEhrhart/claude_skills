---
name: owasp-secure-code
description: >
  Patterns de code sécurisé en PHP / CodeIgniter 4 pour chaque risque OWASP Top 10 2021.
  Utiliser ce skill pour écrire du code PHP ou CI4 sécurisé, corriger une vulnérabilité
  existante, implémenter une protection (XSS, CSRF, injection SQL, auth, headers, logs...),
  ou répondre à des questions pratiques sur la sécurisation d'une application PHP.
  Mots-clés déclencheurs : code sécurisé, PHP sécurité, CI4 sécurité, protection XSS,
  protection CSRF, requête paramétrée, hachage mot de passe, headers sécurité,
  Shield sécurité, validation CI4, rate limiting, logs sécurité.
---

# Patterns de code sécurisé — PHP / CodeIgniter 4

> Basé sur OWASP Top 10 2021. Voir le skill **owasp-overview** pour les descriptions des risques.
> Voir **ci4-architecture** pour les conventions de nommage du projet.

---

## A01 — Contrôles d'accès (Broken Access Control)

### Filtre d'authentification Shield sur les routes
```php
// app/Modules/Crm/Config/Routes.php
$routes->group('crm', [
    'namespace' => 'Modules\Crm\Controllers',
    'filter'    => 'auth',           // ← bloque les non-connectés
], function ($routes) {
    $routes->get('contacts', 'ContactsController::index');
});
```

### Vérification de rôle dans un contrôleur
```php
// Vérification fine des droits
public function delete(int $intId): \CodeIgniter\HTTP\RedirectResponse
{
    // ✅ Vérifier le rôle AVANT toute action
    if (! auth()->user()->inGroup('admin', 'manager')) {
        throw new \CodeIgniter\Exceptions\PageNotFoundException();
        // Ne pas révéler l'existence de la ressource → 404, pas 403
    }

    $this->objModel->delete($intId);
    return redirect()->to('/crm/contacts');
}
```

### Isolation multi-tenant dans les modèles
```php
// BaseModel — filtre automatique par tenant sur TOUTES les requêtes
protected function initialize(): void
{
    $intTenantId = service('tenant')->getId();
    // ✅ Un tenant ne peut jamais voir les données d'un autre
    $this->where($this->table . '_tenant_id', $intTenantId);
}
```

### Ne jamais exposer les IDs bruts — UUID optionnel
```php
// ✅ Utiliser des références non-séquentielles si la ressource est sensible
$strUuid = bin2hex(random_bytes(16));
```

---

## A02 — Cryptographie (Cryptographic Failures)

### Hachage de mot de passe — CI4 Shield (bcrypt par défaut)
```php
// Shield gère le hachage automatiquement. Ne jamais faire :
// ❌ $strHash = md5($strPassword);
// ❌ $strHash = sha1($strPassword);
// ❌ $strHash = password_hash($strPassword, PASSWORD_MD5);

// ✅ Laisser Shield gérer l'enregistrement et la vérification
$boolAuthenticated = auth()->attempt([
    'email'    => $strEmail,
    'password' => $strPassword,
]);
```

### Chiffrement de données sensibles au repos
```php
// Utiliser le service Encryption de CI4
$objEncrypter = \Config\Services::encrypter();

// Chiffrer avant stockage
$strEncrypted = base64_encode($objEncrypter->encrypt($strSensitiveData));

// Déchiffrer à la lecture
$strDecrypted = $objEncrypter->decrypt(base64_decode($strEncrypted));
```

### Configuration HTTPS dans `.env`
```ini
# Forcer HTTPS
app.forceGlobalSecureRequests = true
```

### Cookie sécurisé
```php
// app/Config/Cookie.php
public bool   $secure   = true;   // HTTPS uniquement
public bool   $httponly  = true;   // Non accessible en JavaScript
public string $samesite  = 'Lax'; // Protection CSRF
```

---

## A03 — Injection

### Requêtes paramétrées — Query Builder CI4
```php
// ✅ Query Builder — paramètres liés automatiquement
$arrUsers = $this->db->table('core_users')
    ->where('users_email', $strEmail)   // ← jamais de concaténation ici
    ->where('users_tenant_id', $intTenantId)
    ->get()->getResultObject();

// ✅ Prepared statement direct si besoin
$objQuery = $this->db->query(
    'SELECT * FROM crm_contacts WHERE contacts_email = ? AND contacts_tenant_id = ?',
    [$strEmail, $intTenantId]
);

// ❌ À ne JAMAIS faire
$arrUsers = $this->db->query("SELECT * FROM users WHERE email = '$strEmail'");
```

### Échappement des sorties HTML — Smarty
```smarty
{* ✅ Toujours échapper les variables utilisateur *}
{$objContact->contacts_last_name|escape}
{$strUserInput|escape:'html'}

{* ❌ Ne jamais afficher en brut *}
{$strUserInput}         {* dangereux si contient du HTML *}
{$strUserInput|nofilter} {* dangereux — désactive l'échappement *}
```

### Validation des entrées — CI4
```php
// Dans le contrôleur
$arrRules = [
    'contacts_email'     => 'required|valid_email|max_length[255]',
    'contacts_last_name' => 'required|alpha_space|max_length[100]',
    'contacts_phone'     => 'permit_empty|regex_match[/^[0-9\s\+\-\(\)]{7,20}$/]',
];

if (! $this->validate($arrRules)) {
    return redirect()->back()
        ->withInput()
        ->with('arrErrors', $this->validator->getErrors());
}
```

### Content Security Policy (CSP)
```php
// app/Config/ContentSecurityPolicy.php  ou dans les headers
header("Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
```

---

## A05 — Mauvaise configuration (Security Misconfiguration)

### Configuration `.env` — production
```ini
# ✅ Toujours en production
CI_ENVIRONMENT = production

# ✅ Clé de chiffrement forte (générer avec : php spark key:generate)
encryption.key = hex2bin:VOTRE_CLE_64_CHARS_HEX

# ✅ Session sécurisée
session.cookieSecure   = true
session.cookieHTTPOnly = true
session.cookieSameSite = Lax
```

### Headers de sécurité HTTP — Filtre CI4
```php
// app/Filters/SecurityHeaders.php
namespace App\Filters;

use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use CodeIgniter\Filters\FilterInterface;

class SecurityHeaders implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null): void {}

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null): void
    {
        $response->setHeader('X-Frame-Options', 'SAMEORIGIN');
        $response->setHeader('X-Content-Type-Options', 'nosniff');
        $response->setHeader('X-XSS-Protection', '1; mode=block');
        $response->setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
        $response->setHeader(
            'Content-Security-Policy',
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:"
        );
        // HSTS — uniquement si HTTPS configuré
        $response->setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    }
}
```

Enregistrer dans `app/Config/Filters.php` :
```php
public array $globals = [
    'after' => ['SecurityHeaders'],
];
```

---

## A06 — Composants vulnérables

### Vérification des dépendances
```bash
# Auditer les dépendances Composer
composer audit

# Lister les packages obsolètes
composer show --outdated

# Mettre à jour les patches de sécurité uniquement
composer update --prefer-lowest
```

### Automatisation (GitHub Actions)
```yaml
# .github/workflows/security.yml
name: Security Audit
on:
  schedule:
    - cron: '0 6 * * 1'   # chaque lundi à 6h
  push:
    branches: [main]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - run: composer install --no-dev
      - run: composer audit --no-dev
```

---

## A07 — Authentification (Identification and Authentication Failures)

### Configuration Shield — mots de passe forts
```php
// app/Config/Auth.php
public int $minimumPasswordLength = 12;

// Activer le contrôle des mots de passe compromis (Have I Been Pwned)
public bool $passwordValidators = true;
```

### Rate limiting sur le login
```php
// app/Config/Filters.php
public array $aliases = [
    'auth'       => \CodeIgniter\Shield\Filters\SessionAuth::class,
    'rate-limit'  => \App\Filters\RateLimitFilter::class,
];

// Appliquer sur les routes sensibles
$routes->post('login', 'Auth::loginAction', ['filter' => 'rate-limit:5,1']);
// → max 5 tentatives par minute
```

### Invalidation de session après déconnexion
```php
// Shield gère ça automatiquement via auth()->logout()
// ✅ Forcer l'invalidation côté serveur
auth()->logout();
session()->destroy();
```

### Token de réinitialisation sécurisé
```php
// ✅ Token cryptographiquement aléatoire, à usage unique, expirant
$strToken     = bin2hex(random_bytes(32));   // 64 chars hex
$intExpiry    = time() + 900;                // expire dans 15 minutes
$boolIsUsed   = false;

// Stocker en BDD avec timestamp d'expiration
// Invalider après utilisation (boolIsUsed = true)
```

---

## A08 — Intégrité des données (Software and Data Integrity Failures)

### Ne jamais désérialiser des données non fiables
```php
// ❌ Dangereux — injection d'objet PHP possible
$objData = unserialize($_COOKIE['cart']);
$objData = unserialize(base64_decode($strInput));

// ✅ Utiliser JSON à la place
$arrData = json_decode($strInput, true);

// ✅ Si unserialize est indispensable, utiliser allowed_classes
$arrData = unserialize($strInput, ['allowed_classes' => false]);
```

### Vérification d'intégrité des fichiers uploadés
```php
// ✅ Vérifier le type MIME réel, pas l'extension déclarée
$objFile = $this->request->getFile('document');

if (! $objFile->isValid() || $objFile->hasMoved()) {
    throw new \RuntimeException('Fichier invalide');
}

// Vérifier le type MIME réel
$strMime = $objFile->getMimeType();
$arrAllowed = ['application/pdf', 'image/jpeg', 'image/png'];

if (! in_array($strMime, $arrAllowed, true)) {
    return redirect()->back()->with('strError', 'Type de fichier non autorisé.');
}

// Renommer le fichier — ne jamais utiliser le nom original
$strNewName = $objFile->getRandomName();
$objFile->move(WRITEPATH . 'uploads', $strNewName);
```

---

## A09 — Journalisation (Security Logging and Monitoring Failures)

### Logger les événements de sécurité
```php
// app/Libraries/SecurityLogger.php
namespace App\Libraries;

use CodeIgniter\Log\Logger;

class SecurityLogger
{
    private Logger $objLogger;

    public function __construct()
    {
        $this->objLogger = service('logger');
    }

    /**
     * Log un événement de sécurité avec contexte.
     */
    public function log(string $strEvent, string $strLevel = 'warning', array $arrContext = []): void
    {
        $arrContext = array_merge([
            'ip'        => service('request')->getIPAddress(),
            'user_id'   => auth()->id() ?? 'anonymous',
            'tenant_id' => service('tenant')->getId() ?? 'none',
            'url'       => current_url(),
            'timestamp' => date('Y-m-d H:i:s'),
        ], $arrContext);

        $this->objLogger->$strLevel(
            '[SECURITY] ' . $strEvent . ' | ' . json_encode($arrContext)
        );
    }
}

// Utilisation dans un contrôleur
$objSecLog = new \App\Libraries\SecurityLogger();
$objSecLog->log('LOGIN_FAILED', 'warning', ['email' => $strEmail]);
$objSecLog->log('ACCESS_DENIED', 'warning', ['resource' => '/admin']);
$objSecLog->log('PASSWORD_CHANGED', 'info');
```

### Événements à logger systématiquement
```
✅ Connexion réussie      → info
✅ Connexion échouée      → warning  (avec email/IP)
✅ Déconnexion            → info
✅ Accès refusé (403/404) → warning
✅ Modification de droits → critical
✅ Suppression de données → warning
✅ Changement de mot de passe → info
✅ Upload de fichier       → info
✅ Erreur 500              → error
```

---

## A10 — SSRF (Server-Side Request Forgery)

### Valider les URLs avant appel HTTP
```php
// ❌ Dangereux — appel de n'importe quelle URL utilisateur
$strUrl = $this->request->getPost('webhook_url');
$strResponse = file_get_contents($strUrl);   // SSRF !

// ✅ Liste blanche de domaines autorisés
protected array $arrAllowedDomains = [
    'api.stripe.com',
    'hooks.slack.com',
    'api.sendgrid.com',
];

protected function isUrlAllowed(string $strUrl): bool
{
    $arrParsed = parse_url($strUrl);

    if (empty($arrParsed['host'])) {
        return false;
    }

    $strHost = strtolower($arrParsed['host']);

    // Bloquer les IPs privées et localhost
    if (filter_var($strHost, FILTER_VALIDATE_IP)) {
        if (filter_var($strHost, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false) {
            return false;   // IP privée ou réservée
        }
    }

    // Bloquer localhost sous toutes ses formes
    if (in_array($strHost, ['localhost', '127.0.0.1', '::1', '0.0.0.0'], true)) {
        return false;
    }

    // Vérifier la liste blanche
    return in_array($strHost, $this->arrAllowedDomains, true);
}
```

---

## Checklist de sécurité — avant mise en production

### Configuration
- [ ] `CI_ENVIRONMENT = production` dans `.env`
- [ ] Clé de chiffrement générée (`php spark key:generate`)
- [ ] Debug/stack traces désactivés
- [ ] Tous les mots de passe par défaut changés

### Authentification & Accès
- [ ] Shield installé et configuré
- [ ] Toutes les routes sensibles avec filtre `auth`
- [ ] Vérification des droits à chaque requête (pas seulement sur les routes)
- [ ] Rate limiting sur login, reset de mot de passe, API

### Données
- [ ] Toutes les requêtes SQL via Query Builder (pas de concaténation)
- [ ] Validation des entrées sur tous les formulaires
- [ ] Sorties Smarty avec `|escape`
- [ ] `{csrf_field()}` dans tous les formulaires POST

### Fichiers & Uploads
- [ ] Vérification du type MIME réel (pas de l'extension)
- [ ] Renommage des fichiers uploadés
- [ ] Stockage hors du répertoire public

### Headers & Transport
- [ ] HTTPS forcé (`forceGlobalSecureRequests = true`)
- [ ] Headers de sécurité configurés (filtre SecurityHeaders)
- [ ] Cookies `Secure`, `HttpOnly`, `SameSite`

### Dépendances & Logs
- [ ] `composer audit` sans vulnérabilités critiques
- [ ] Événements de sécurité loggés
- [ ] Logs stockés hors de la portée de l'application
