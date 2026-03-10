---
name: ci4-basecontroller
description: >
  Référence complète du BaseController CE SOFT pour CodeIgniter 4.
  Utiliser ce skill pour : modifier ou recréer le BaseController, comprendre
  comment Smarty est initialisé, comment utiliser _arrData et _display(),
  comment la langue est transmise aux templates.
  Mots-clés déclencheurs : BaseController, _smarty, _arrData, _display,
  initController, Smarty init, assign, display, lang Smarty, cache Smarty.
---

# BaseController — CE SOFT (CodeIgniter 4 + Smarty)

Fichier : `app/Controllers/BaseController.php`

---

## Code complet du BaseController

```php
<?php

namespace App\Controllers;

use CodeIgniter\Controller;
use CodeIgniter\HTTP\CLIRequest;
use CodeIgniter\HTTP\IncomingRequest;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use Psr\Log\LoggerInterface;
use Smarty\Smarty;

/**
 * Class BaseController
 *
 * BaseController fournit des méthodes pratiques qui sont
 * disponibles dans tous les contrôleurs enfants.
 * Étendre ce contrôleur dans chaque module.
 */
abstract class BaseController extends Controller
{
    /**
     * Instance de Smarty — moteur de templates.
     */
    protected Smarty $_smarty;

    /**
     * Tableau de données passées aux vues Smarty.
     * Toutes les variables à afficher dans les templates
     * doivent être stockées ici avant l'appel à _display().
     *
     * @var array<string, mixed>
     */
    protected array $_arrData = [];

    /**
     * Requête courante (CLI ou HTTP).
     */
    protected CLIRequest|IncomingRequest $request;

    /**
     * Helpers chargés automatiquement dans tous les contrôleurs.
     *
     * @var array<string>
     */
    protected $helpers = [];

    /**
     * Initialise le contrôleur — appelé automatiquement par CI4.
     * NE PAS modifier la ligne parent::initController().
     */
    public function initController(
        RequestInterface  $request,
        ResponseInterface $response,
        LoggerInterface   $logger
    ): void {
        // Ne pas modifier cette ligne
        parent::initController($request, $response, $logger);

        // --- Initialisation de Smarty ---
        $this->_smarty = new Smarty();
        $this->_smarty->setCaching(Smarty::CACHING_LIFETIME_CURRENT);

        // Répertoire des templates (app/Views/)
        $this->_smarty->setTemplateDir(APPPATH . 'Views/');

        // Répertoires de cache (dans writable/)
        $this->_smarty->setCompileDir(WRITEPATH . 'cache/templates_c/');
        $this->_smarty->setConfigDir(WRITEPATH . 'cache/configs/');
        $this->_smarty->setCacheDir(WRITEPATH . 'cache/cache/');

        // Passer la locale courante à tous les templates ({$lang})
        $this->_smarty->assign('lang', $this->request->getLocale());
    }

    /**
     * Affiche un template Smarty en injectant les données de $_arrData.
     *
     * Usage dans un contrôleur enfant :
     *   $this->_arrData['strTitle'] = 'Ma page';
     *   $this->_display('Modules\Module1\Views\index');
     *
     * @param string $view Chemin du template relatif à APPPATH, sans extension.
     *                     Utiliser des backslashes : 'Modules\Module1\Views\index'
     */
    protected function _display(string $view): void
    {
        foreach ($this->_arrData as $strKey => $mixValue) {
            $this->_smarty->assign($strKey, $mixValue);
        }

        $this->_smarty->display(APPPATH . $view . '.tpl');
    }
}
```

---

## Règles d'utilisation dans les contrôleurs enfants

### Passer des données à la vue
Toutes les variables sont stockées dans `$this->_arrData` **avant** l'appel à `_display()`.
Les clés respectent le préfixe de type :

```php
// Chaîne de caractères
$this->_arrData['strTitle']   = lang('Module1.title');

// Tableau
$this->_arrData['arrUsers']   = $this->objModel->findAll();

// Objet/entité
$this->_arrData['objContact'] = $this->objModel->find($intId);

// Booléen
$this->_arrData['boolAdmin']  = auth()->user()->inGroup('admin');

// Entier
$this->_arrData['intTotal']   = $this->objModel->countAll();
```

### Afficher la vue
```php
// Chemin relatif à app/, sans .tpl, avec antislash
$this->_display('Modules\Module1\Views\index');
$this->_display('Modules\Crm\Views\contacts\list');
```

### Variable `$lang` automatique
La variable `{$lang}` est disponible dans **tous** les templates sans `assign()` explicite.
Elle contient la locale courante (ex. `fr`, `en`).

---

## Répertoires Smarty à créer

Si les dossiers cache n'existent pas, les créer :

```bash
mkdir -p writable/cache/templates_c
mkdir -p writable/cache/configs
mkdir -p writable/cache/cache
```

Et vérifier les droits d'écriture :
```bash
chmod -R 775 writable/
```

---

## Variables globales disponibles dans tous les templates

| Variable Smarty | Source | Exemple de valeur |
|-----------------|--------|-------------------|
| `{$lang}` | `$this->request->getLocale()` | `fr`, `en` |

Toute variable supplémentaire globale (ex. utilisateur connecté, nom du site)
doit être ajoutée dans `initController()` avec `$this->_smarty->assign(...)`.

---

## Checklist

- [ ] `use Smarty\Smarty;` présent en haut du fichier
- [ ] `protected Smarty $_smarty;` déclaré
- [ ] `protected array $_arrData = [];` déclaré
- [ ] `initController()` contient l'initialisation Smarty complète
- [ ] `setTemplateDir(APPPATH . 'Views/')` — noter le slash final
- [ ] Les 3 répertoires `setCompileDir`, `setConfigDir`, `setCacheDir` pointent dans `WRITEPATH`
- [ ] `$lang` assigné depuis `$this->request->getLocale()`
- [ ] `_display()` boucle sur `$_arrData` avant `display()`
