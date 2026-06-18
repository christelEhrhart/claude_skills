---
name: owasp-secure-code
description: >
  Patterns de code sécurisé en PHP / CodeIgniter 4 pour chaque risque OWASP Top 10 2025.
  Utiliser ce skill pour écrire du code PHP ou CI4 sécurisé, corriger une vulnérabilité
  existante, implémenter une protection (XSS, CSRF, injection SQL, auth, headers, logs...),
  ou répondre à des questions pratiques sur la sécurisation d'une application PHP.
  Mots-clés déclencheurs : code sécurisé, PHP sécurité, CI4 sécurité, protection XSS,
  protection CSRF, requête paramétrée, hachage mot de passe, headers sécurité,
  Shield sécurité, validation CI4, rate limiting, logs sécurité.
---

# Patterns de code sécurisé — PHP / CodeIgniter 4

> Basé sur OWASP Top 10 2025. Voir le skill **owasp-overview** pour les descriptions des risques.
> Voir **ci4-architecture** pour les conventions de nommage du projet.

---

## A01 — Contrôles d'accès (Broken Access Control) + SSRF

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

### Validation d'URL sortante — protection SSRF
```php
// ✅ Liste blanche de domaines autorisés pour les appels HTTP sortants
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
            return false;
        }
    }

    if (in_array($strHost, ['localhost', '127.0.0.1', '::1', '0.0.0.0'], true)) {
        return false;
    }

    return in_array($strHost, $this->arrAllowedDomains, true);
}

// Utilisation
$strWebhookUrl = $this->request->getPost('webhook_url');
if (! $this->isUrlAllowed($strWebhookUrl)) {
    return redirect()->back()->with('strError', 'URL non autorisée.');
}
$strResponse = \Config\Services::curlrequest()->get($strWebhookUrl);
```

---

## A02 — Mauvaise configuration (Security Misconfiguration)

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

## A03 — Chaîne d'approvisionnement (Software Supply Chain Failures)

### Vérification des dépendances
```bash
# Auditer les dépendances Composer
composer audit

# Lister les packages obsolètes
composer show --outdated

# Installer sans les dépendances de dev en production
composer install --no-dev --optimize-autoloader
```

### Automatisation — GitHub Actions sécurisé
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
    # ✅ Permissions minimales
    permissions:
      contents: read
    steps:
      # ✅ Pinner les actions sur leur SHA exact (pas seulement le tag)
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - run: composer install --no-dev
      - run: composer audit --no-dev
      # ✅ Ne jamais logger les secrets
      - name: Verify no secrets in code
        run: grep -rn "password\|secret\|api_key" --include="*.php" app/ | grep -v "\.env" || true
```

### Vérification d'intégrité des packages
```bash
# Vérifier que composer.lock est versionné et non modifié
git show HEAD:composer.lock | md5sum
md5sum composer.lock

# Forcer la vérification des checksums
composer install --verify-no-file-permissions
```

---

## A04 — Cryptographie (Cryptographic Failures)

### Hachage de mot de passe — CI4 Shield (bcrypt par défaut)
```php
// Shield gère le hachage automatiquement. Ne jamais faire :
// ❌ $strHash = md5($strPassword);
// ❌ $strHash = sha1($strPassword);

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

## A05 — Injection

### Requêtes paramétrées — Query Builder CI4
```php
// ✅ Query Builder — paramètres liés automatiquement
$arrUsers = $this->db->table('core_users')
    ->where('users_email', $strEmail)
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
{$strUserInput}
{$strUserInput|nofilter}
```

### Validation des entrées — CI4
```php
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

---

## A06 — Conception (Insecure Design)

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

### Messages d'erreur génériques côté utilisateur
```php
// ✅ Message générique → pas de révélation d'information
catch (\Exception $e) {
    log_message('error', '[APP] ' . $e->getMessage());
    return redirect()->back()->with('strError', 'Une erreur est survenue. Veuillez réessayer.');
}
```

---

## A07 — Authentification (Authentication Failures)

### Configuration Shield — mots de passe forts
```php
// app/Config/Auth.php
public int $minimumPasswordLength = 12;
public bool $passwordValidators   = true;  // vérifie Have I Been Pwned
```

### Invalidation de session après déconnexion
```php
// ✅ Shield + destruction complète de session
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

## A08 — Intégrité des données (Software or Data Integrity Failures)

### Ne jamais désérialiser des données non fiables
```php
// ❌ Dangereux
$objData = unserialize($_COOKIE['cart']);

// ✅ Utiliser JSON à la place
$arrData = json_decode($strInput, true);

// ✅ Si unserialize est indispensable
$arrData = unserialize($strInput, ['allowed_classes' => false]);
```

### Vérification d'intégrité des fichiers uploadés
```php
$objFile = $this->request->getFile('document');

if (! $objFile->isValid() || $objFile->hasMoved()) {
    throw new \RuntimeException('Fichier invalide');
}

// ✅ Vérifier le type MIME réel
$strMime   = $objFile->getMimeType();
$arrAllowed = ['application/pdf', 'image/jpeg', 'image/png'];

if (! in_array($strMime, $arrAllowed, true)) {
    return redirect()->back()->with('strError', 'Type de fichier non autorisé.');
}

// ✅ Renommer le fichier — ne jamais utiliser le nom original
$strNewName = $objFile->getRandomName();
$objFile->move(WRITEPATH . 'uploads', $strNewName);
```

---

## A09 — Journalisation et alerte (Security Logging and Alerting Failures)

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

## A10 — Gestion des conditions exceptionnelles (Mishandling of Exceptional Conditions)

### Gestionnaire d'erreurs global — CI4
```php
// app/Config/Exceptions.php
// En production, CI4 affiche une page d'erreur générique si CI_ENVIRONMENT = production
// Vérifier que les erreurs ne sont pas affichées en clair
```

### Pattern catch sécurisé
```php
// ❌ Dangereux — révèle la structure interne
catch (\Exception $e) {
    echo $e->getMessage();
}

// ❌ Dangereux — exception silencieuse
catch (\Exception $e) {
    // rien
}

// ✅ Correct — log interne, message générique à l'utilisateur
catch (\Exception $e) {
    log_message('error', '[APP] ' . get_class($e) . ': ' . $e->getMessage() . ' in ' . $e->getFile() . ':' . $e->getLine());
    return redirect()->back()->with('strError', 'Une erreur est survenue. Veuillez réessayer.');
}

// ✅ Pour les API JSON
catch (\Exception $e) {
    log_message('error', '[API] ' . $e->getMessage());
    return $this->response
        ->setStatusCode(500)
        ->setJSON(['error' => 'Internal server error']);
    // Jamais : ['error' => $e->getMessage()]
}
```

### Validation des états de transition
```php
// ✅ Vérifier la cohérence de l'état avant toute action critique
public function confirmOrder(int $intOrderId): \CodeIgniter\HTTP\RedirectResponse
{
    $objOrder = $this->objOrderModel->find($intOrderId);

    if ($objOrder === null) {
        throw new \CodeIgniter\Exceptions\PageNotFoundException();
    }

    // ✅ Vérifier que le paiement est validé avant confirmation
    if ($objOrder->order_payment_status !== 'paid') {
        log_message('warning', '[SECURITY] Tentative de confirmation sans paiement, order_id=' . $intOrderId);
        return redirect()->back()->with('strError', 'Le paiement doit être validé avant confirmation.');
    }

    // ✅ Vérifier l'appartenance au tenant
    if ($objOrder->order_tenant_id !== service('tenant')->getId()) {
        throw new \CodeIgniter\Exceptions\PageNotFoundException();
    }

    $this->objOrderModel->update($intOrderId, ['order_status' => 'confirmed']);
    return redirect()->to('/orders')->with('strSuccess', 'Commande confirmée.');
}
```

### Gestion des cas limites
```php
// ✅ Toujours vérifier les valeurs avant opérations critiques
public function calculateDiscount(float $fltTotal, float $fltRate): float
{
    if ($fltTotal <= 0 || $fltRate < 0 || $fltRate > 100) {
        log_message('warning', '[APP] Valeurs invalides calculateDiscount: total=' . $fltTotal . ', rate=' . $fltRate);
        return 0.0;
    }

    return $fltTotal * ($fltRate / 100);
}

// ✅ Fail-safe : en cas de doute, refuser l'accès
public function hasPermission(string $strAction): bool
{
    try {
        return auth()->user()->inGroup('admin') || $this->checkAcl($strAction);
    } catch (\Exception $e) {
        log_message('error', '[SECURITY] Permission check failed: ' . $e->getMessage());
        return false;   // fail-safe : refuser plutôt qu'accorder
    }
}
```

---

## Checklist de sécurité — avant mise en production

### Configuration
- [ ] `CI_ENVIRONMENT = production` dans `.env`
- [ ] Clé de chiffrement générée (`php spark key:generate`)
- [ ] Debug/stack traces désactivés
- [ ] Tous les mots de passe par défaut changés
- [ ] Headers de sécurité configurés (filtre SecurityHeaders)

### Chaîne d'approvisionnement
- [ ] `composer audit` sans vulnérabilités critiques
- [ ] `composer.lock` versionné dans git
- [ ] Packages de dev exclus en production (`--no-dev`)
- [ ] Actions CI/CD pinnées sur des SHA précis
- [ ] Secrets CI/CD dans des variables d'environnement chiffrées

### Authentification & Accès
- [ ] Shield installé et configuré
- [ ] Toutes les routes sensibles avec filtre `auth`
- [ ] Vérification des droits à chaque requête (pas seulement sur les routes)
- [ ] Rate limiting sur login, reset de mot de passe, API
- [ ] URLs sortantes validées contre une liste blanche (SSRF)

### Données
- [ ] Toutes les requêtes SQL via Query Builder (pas de concaténation)
- [ ] Validation des entrées sur tous les formulaires
- [ ] Sorties Smarty avec `|escape`
- [ ] `{csrf_field()}` dans tous les formulaires POST

### Fichiers & Uploads
- [ ] Vérification du type MIME réel (pas de l'extension)
- [ ] Renommage des fichiers uploadés
- [ ] Stockage hors du répertoire public

### Gestion des erreurs
- [ ] Aucun `echo $e->getMessage()` en production
- [ ] Aucun bloc catch vide
- [ ] États de transition vérifiés côté serveur (paiement, validation)
- [ ] Cas limites testés (null, 0, chaînes vides, valeurs négatives)

### Logs & Alertes
- [ ] Événements de sécurité loggés (connexions, accès refusés, erreurs)
- [ ] Logs stockés hors de la portée de l'application
- [ ] Alertes configurées sur les anomalies
