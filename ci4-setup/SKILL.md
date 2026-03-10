---
name: ci4-setup
description: >
  Installer et configurer un nouveau projet CodeIgniter 4 selon les standards CE SOFT.
  Utiliser ce skill pour : créer un projet CI4 from scratch, configurer .env et App.php,
  installer Shield, configurer le HMVC (Autoload + Modules), installer Smarty,
  installer Bootstrap, configurer les traductions.
  Mots-clés déclencheurs : nouveau projet, installer CI4, setup, configuration,
  Shield setup, HMVC setup, Smarty install, Bootstrap install, .env, Autoload.
---

# Installer un projet CodeIgniter 4 — CE SOFT

## Prérequis
Compétences en PHP, Smarty, MVC, Composer.

---

## Étape 1 — Créer le projet via Composer

```bash
composer create-project codeigniter4/appstarter nom_du_projet
```

---

## Étape 2 — Configurer `.env`

```ini
# Base de données
database.default.hostname = localhost
database.default.database = nom_de_la_base_de_donnees
database.default.username = nom_utilisateur
database.default.password = mot_de_passe
database.default.DBDriver = MySQLi

# URL de base
app.baseURL = 'http://localhost/nom_du_projet/public'

# Environnement
CI_ENVIRONMENT = development
```

---

## Étape 3 — Configurer `app/Config/App.php`

```php
public $baseURL = 'http://localhost/nom_du_projet/public';
public $defaultLocale = 'fr';   // langue par défaut
```

---

## Étape 4 — Installer les dépendances

```bash
# Traductions officielles CI4
composer require codeigniter4/translations

# Shield (authentification)
composer require codeigniter4/shield

# Smarty
composer require smarty/smarty

# Bootstrap
composer require twbs/bootstrap
```

---

## Étape 5 — Configurer Shield

```bash
php spark shield:setup
```

Dans `app/Config/Routes.php`, ajouter :
```php
service('auth')->routes($routes);
```

**Erreur CSRF fréquente** — si le message `csrfProtection is set to 'cookie'` apparaît,
ouvrir `app/Config/Security.php` et modifier :
```php
public $csrfProtection = 'session';
```

---

## Étape 6 — Configurer le HMVC (Autoload + Modules)

### 6.1 Modifier `app/Config/Autoload.php`

Ajouter `'Modules'` dans `$psr4`, le constructeur et la méthode `loadModules()` :

```php
<?php

namespace Config;

use CodeIgniter\Config\AutoloadConfig;

class Autoload extends AutoloadConfig
{
    public $psr4 = [
        APP_NAMESPACE => APPPATH,
        'Modules'     => APPPATH . 'Modules',
    ];

    public $classmap = [];
    public $files    = [];

    /**
     * Autoload constructor.
     */
    public function __construct()
    {
        parent::__construct();
        $this->loadModules();
    }

    /**
     * Charge automatiquement tous les modules présents dans app/Modules/
     */
    protected function loadModules(): void
    {
        $strScan = APPPATH . 'Modules';
        $strDir  = scandir($strScan);

        foreach ($strDir as $strDirName) {
            if ($strDirName !== '.' && $strDirName !== '..') {
                $this->psr4[$strDirName] = APPPATH . 'Modules/' . $strDirName;
            }
        }
    }
}
```

### 6.2 Créer le répertoire `app/Modules/`

```bash
mkdir app/Modules
```

---

## Étape 7 — Configurer Smarty dans `app/Controllers/BaseController.php`

Voir le skill **ci4-basecontroller** pour le code complet du BaseController.

---

## Étape 8 — Layout Smarty de base avec Bootstrap et i18n

Créer `app/Views/layout.tpl` :

```smarty
<!DOCTYPE html>
<html lang="{nocache}{$lang}{/nocache}">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{block name='title' nocache}{/block}</title>
    {block name='stylesheet'}
        <link rel="stylesheet" href="../vendor/twbs/bootstrap/dist/css/bootstrap.min.css">
    {/block}
</head>
<body>
    {block name='content' nocache}
    {/block}
    {block name='javascript'}
        <script src="../vendor/twbs/bootstrap/dist/js/bootstrap.min.js"></script>
    {/block}
</body>
</html>
```

---

## Checklist installation complète

- [ ] `composer create-project` exécuté
- [ ] `.env` configuré (DB, baseURL, CI_ENVIRONMENT)
- [ ] `app/Config/App.php` — baseURL et defaultLocale configurés
- [ ] `composer require` : translations, shield, smarty, bootstrap
- [ ] `php spark shield:setup` exécuté
- [ ] `service('auth')->routes($routes)` ajouté dans Routes.php
- [ ] `$csrfProtection = 'session'` dans Security.php
- [ ] `app/Config/Autoload.php` modifié (psr4 + loadModules)
- [ ] Répertoire `app/Modules/` créé
- [ ] `app/Controllers/BaseController.php` configuré avec Smarty
- [ ] `app/Views/layout.tpl` créé
- [ ] Dossiers Smarty cache créés : `writable/cache/templates_c/`, `writable/cache/configs/`, `writable/cache/cache/`
