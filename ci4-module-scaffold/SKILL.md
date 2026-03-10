---
name: ci4-module-scaffold
description: >
  Créer un module HMVC complet dans CodeIgniter 4 selon les standards CE SOFT.
  Utiliser ce skill pour scaffolder : la structure de répertoires d'un module,
  le contrôleur (avec _arrData et _display), les routes, les vues Smarty (.tpl),
  les fichiers de langue (fr/en), les modèles et entités.
  Mots-clés déclencheurs : créer module, nouveau module, scaffolding, CRUD,
  contrôleur module, vue module, routes module, langue module, Language fr en,
  _display, _arrData, .tpl module.
---

# Scaffolding d'un module HMVC — CE SOFT

> Ce skill suppose que le BaseController est configuré (voir **ci4-basecontroller**)
> et que le HMVC est en place (voir **ci4-setup**).

---

## Structure complète d'un module

```
app/Modules/NomModule/
├── Config/
│   └── Routes.php
├── Controllers/
│   └── NomModuleController.php
├── Models/
│   └── NomObjetModel.php
├── Entities/
│   └── NomObjet.php
├── Language/
│   ├── fr/
│   │   └── NomModule.php
│   └── en/
│       └── NomModule.php
└── Views/
    ├── index.tpl
    └── form.tpl
```

> **Note :** les vues des modules se trouvent dans `app/Modules/NomModule/Views/`
> ET doivent être accessibles depuis le `setTemplateDir(APPPATH . 'Views/')`.
> Pour les modules, passer le chemin complet à `_display()` :
> `'Modules\NomModule\Views\index'`

---

## 1. Routes — `Config/Routes.php`

```php
<?php

// Namespace réel du module — avec antislash
$routes->group('module1', ['namespace' => 'Modules\Module1\Controllers'], function ($routes) {
    $routes->get('/', 'Module1Controller::index');
    $routes->get('nouveau', 'Module1Controller::create');
    $routes->post('nouveau', 'Module1Controller::store');
    $routes->get('(:num)', 'Module1Controller::show/$1');
    $routes->get('(:num)/modifier', 'Module1Controller::edit/$1');
    $routes->post('(:num)/modifier', 'Module1Controller::update/$1');
    $routes->post('(:num)/supprimer', 'Module1Controller::delete/$1');
});
```

> Le groupe `'auth'` comme filtre peut être ajouté si Shield est installé :
> `['namespace' => '...', 'filter' => 'auth']`

---

## 2. Contrôleur — `Controllers/Module1Controller.php`

```php
<?php

namespace Modules\Module1\Controllers;

use App\Controllers\BaseController;
use Modules\Module1\Models\ContactsModel;

class Module1Controller extends BaseController
{
    private ContactsModel $objModel;

    /**
     * Initialise le modèle — appelé après initController().
     */
    public function initController(
        \CodeIgniter\HTTP\RequestInterface  $request,
        \CodeIgniter\HTTP\ResponseInterface $response,
        \Psr\Log\LoggerInterface            $logger
    ): void {
        parent::initController($request, $response, $logger);
        $this->objModel = new ContactsModel();
    }

    // -----------------------------------------------------------------------
    // Liste
    // -----------------------------------------------------------------------
    public function index(): void
    {
        $this->_arrData['strTitle']   = lang('Module1.title');
        $this->_arrData['arrItems']   = $this->objModel->findAll();
        $this->_display('Modules\Module1\Views\index');
    }

    // -----------------------------------------------------------------------
    // Formulaire de création
    // -----------------------------------------------------------------------
    public function create(): void
    {
        $this->_arrData['strTitle'] = lang('Module1.add');
        $this->_display('Modules\Module1\Views\form');
    }

    // -----------------------------------------------------------------------
    // Enregistrement
    // -----------------------------------------------------------------------
    public function store(): \CodeIgniter\HTTP\RedirectResponse
    {
        $arrData = $this->request->getPost();

        if (! $this->objModel->save($arrData)) {
            return redirect()->back()
                ->withInput()
                ->with('arrErrors', $this->objModel->errors());
        }

        return redirect()->to('/module1')
            ->with('strSuccess', lang('Module1.saved'));
    }

    // -----------------------------------------------------------------------
    // Détail
    // -----------------------------------------------------------------------
    public function show(int $intId): void
    {
        $this->_arrData['strTitle'] = lang('Module1.view');
        $this->_arrData['objItem']  = $this->objModel->findOrFail($intId);
        $this->_display('Modules\Module1\Views\show');
    }

    // -----------------------------------------------------------------------
    // Formulaire de modification
    // -----------------------------------------------------------------------
    public function edit(int $intId): void
    {
        $this->_arrData['strTitle'] = lang('Module1.edit');
        $this->_arrData['objItem']  = $this->objModel->findOrFail($intId);
        $this->_display('Modules\Module1\Views\form');
    }

    // -----------------------------------------------------------------------
    // Mise à jour
    // -----------------------------------------------------------------------
    public function update(int $intId): \CodeIgniter\HTTP\RedirectResponse
    {
        $arrData = $this->request->getPost();

        if (! $this->objModel->update($intId, $arrData)) {
            return redirect()->back()
                ->withInput()
                ->with('arrErrors', $this->objModel->errors());
        }

        return redirect()->to('/module1')
            ->with('strSuccess', lang('Module1.updated'));
    }

    // -----------------------------------------------------------------------
    // Suppression
    // -----------------------------------------------------------------------
    public function delete(int $intId): \CodeIgniter\HTTP\RedirectResponse
    {
        $this->objModel->delete($intId);

        return redirect()->to('/module1')
            ->with('strSuccess', lang('Module1.deleted'));
    }
}
```

---

## 3. Modèle — `Models/ContactsModel.php`

```php
<?php

namespace Modules\Module1\Models;

use CodeIgniter\Model;

class ContactsModel extends Model
{
    protected $table         = 'module1_contacts';
    protected $primaryKey    = 'contacts_id';
    protected $returnType    = 'object';
    protected $useSoftDeletes = true;

    protected $allowedFields = [
        'contacts_last_name',
        'contacts_first_name',
        'contacts_email',
    ];

    protected $useTimestamps = true;
    protected $createdField  = 'contacts_created_at';
    protected $updatedField  = 'contacts_updated_at';
    protected $deletedField  = 'contacts_deleted_at';

    protected $validationRules = [
        'contacts_last_name' => 'required|max_length[100]',
        'contacts_email'     => 'required|valid_email',
    ];

    /**
     * Trouve un enregistrement ou lève PageNotFoundException.
     */
    public function findOrFail(int $intId): object
    {
        $objResult = $this->find($intId);

        if ($objResult === null) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException(
                lang('Module1.not_found', [$intId])
            );
        }

        return $objResult;
    }
}
```

---

## 4. Fichiers de langue

### `Language/fr/Module1.php`

```php
<?php

return [
    'title'      => 'Liste des éléments',
    'add'        => 'Ajouter',
    'edit'       => 'Modifier',
    'view'       => 'Détail',
    'saved'      => 'Enregistré avec succès.',
    'updated'    => 'Mis à jour.',
    'deleted'    => 'Supprimé.',
    'not_found'  => 'Élément #{0} introuvable.',
    'confirm_delete' => 'Confirmer la suppression ?',
];
```

### `Language/en/Module1.php`

```php
<?php

return [
    'title'      => 'Items list',
    'add'        => 'Add',
    'edit'       => 'Edit',
    'view'       => 'View',
    'saved'      => 'Saved successfully.',
    'updated'    => 'Updated.',
    'deleted'    => 'Deleted.',
    'not_found'  => 'Item #{0} not found.',
    'confirm_delete' => 'Confirm deletion?',
];
```

---

## 5. Vues Smarty

### `Views/index.tpl` — Liste

```smarty
{extends file='layout.tpl'}

{block name='title' nocache}{$strTitle}{/block}

{block name='content' nocache}
<div class="container mt-4">

    <div class="d-flex justify-content-between align-items-center mb-3">
        <h1>{$strTitle|escape}</h1>
        <a href="/module1/nouveau" class="btn btn-primary">
            {* lang() disponible via plugin ou variable passée *}
            Ajouter
        </a>
    </div>

    {* Messages flash *}
    {if isset($smarty.session.strSuccess)}
        <div class="alert alert-success" role="alert">
            {$smarty.session.strSuccess|escape}
        </div>
    {/if}

    {if isset($smarty.session.arrErrors)}
        <div class="alert alert-danger" role="alert">
            <ul class="mb-0">
                {foreach $smarty.session.arrErrors as $strError}
                    <li>{$strError|escape}</li>
                {/foreach}
            </ul>
        </div>
    {/if}

    {if $arrItems|@count > 0}
        <div class="table-responsive">
            <table class="table table-striped table-hover">
                <caption class="visually-hidden">{$strTitle|escape}</caption>
                <thead class="table-dark">
                    <tr>
                        <th scope="col">Nom</th>
                        <th scope="col">Prénom</th>
                        <th scope="col">Email</th>
                        <th scope="col"><span class="visually-hidden">Actions</span></th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $arrItems as $objItem}
                    <tr>
                        <td>{$objItem->contacts_last_name|escape}</td>
                        <td>{$objItem->contacts_first_name|escape}</td>
                        <td>
                            <a href="mailto:{$objItem->contacts_email|escape}">
                                {$objItem->contacts_email|escape}
                            </a>
                        </td>
                        <td>
                            <a href="/module1/{$objItem->contacts_id}/modifier"
                               class="btn btn-sm btn-outline-secondary"
                               aria-label="Modifier {$objItem->contacts_last_name|escape}">
                                Modifier
                            </a>
                            <form method="post"
                                  action="/module1/{$objItem->contacts_id}/supprimer"
                                  class="d-inline"
                                  onsubmit="return confirm('Confirmer la suppression ?')">
                                {csrf_field()}
                                <button type="submit"
                                        class="btn btn-sm btn-outline-danger"
                                        aria-label="Supprimer {$objItem->contacts_last_name|escape}">
                                    Supprimer
                                </button>
                            </form>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    {else}
        <p class="text-muted">Aucun résultat.</p>
    {/if}

</div>
{/block}
```

### `Views/form.tpl` — Formulaire création / modification

```smarty
{extends file='layout.tpl'}

{block name='title' nocache}{$strTitle}{/block}

{block name='content' nocache}
<div class="container mt-4">

    <h1>{$strTitle|escape}</h1>

    {* Erreurs de validation *}
    {if isset($smarty.session.arrErrors)}
        <div class="alert alert-danger" role="alert">
            <ul class="mb-0">
                {foreach $smarty.session.arrErrors as $strError}
                    <li>{$strError|escape}</li>
                {/foreach}
            </ul>
        </div>
    {/if}

    <form method="post"
          action="{if isset($objItem) && $objItem->contacts_id}/module1/{$objItem->contacts_id}/modifier{else}/module1/nouveau{/if}"
          novalidate>

        {csrf_field()}

        <div class="mb-3">
            <label for="contacts_last_name" class="form-label">
                Nom <span class="text-danger" aria-hidden="true">*</span>
            </label>
            <input type="text"
                   class="form-control"
                   id="contacts_last_name"
                   name="contacts_last_name"
                   value="{if isset($objItem)}{$objItem->contacts_last_name|escape}{/if}"
                   required
                   aria-required="true"
                   autocomplete="family-name">
        </div>

        <div class="mb-3">
            <label for="contacts_first_name" class="form-label">Prénom</label>
            <input type="text"
                   class="form-control"
                   id="contacts_first_name"
                   name="contacts_first_name"
                   value="{if isset($objItem)}{$objItem->contacts_first_name|escape}{/if}"
                   autocomplete="given-name">
        </div>

        <div class="mb-3">
            <label for="contacts_email" class="form-label">
                Email <span class="text-danger" aria-hidden="true">*</span>
            </label>
            <input type="email"
                   class="form-control"
                   id="contacts_email"
                   name="contacts_email"
                   value="{if isset($objItem)}{$objItem->contacts_email|escape}{/if}"
                   required
                   aria-required="true"
                   autocomplete="email">
        </div>

        <div class="d-flex gap-2">
            <button type="submit" class="btn btn-primary">Enregistrer</button>
            <a href="/module1" class="btn btn-secondary">Annuler</a>
        </div>

    </form>

</div>
{/block}
```

---

## Rappels de syntaxe

| Besoin | Code |
|--------|------|
| Passer une variable à la vue | `$this->_arrData['strTitle'] = 'Valeur';` |
| Afficher la vue | `$this->_display('Modules\Module1\Views\index');` |
| Clé de traduction | `lang('Module1.title')` dans PHP |
| Échappement Smarty | `{$variable\|escape}` |
| Nocache | `{block name='content' nocache}` |
| Variable de langue | `{$lang}` (assignée automatiquement) |
| CSRF dans formulaire | `{csrf_field()}` |

---

## Checklist module

- [ ] Répertoire `app/Modules/NomModule/` créé avec tous les sous-dossiers
- [ ] `Config/Routes.php` — namespace `Modules\NomModule\Controllers`
- [ ] Contrôleur hérite de `BaseController`, utilise `_arrData` et `_display()`
- [ ] Variables dans `_arrData` préfixées par type (`str`, `arr`, `obj`, `int`, `bool`…)
- [ ] Chemin dans `_display()` avec antislash : `'Modules\NomModule\Views\nom'`
- [ ] Modèle : table `{module}_{objet}`, PK `{objet}_id`, champs `{objet}_*`
- [ ] Fichiers de langue créés en `fr/` ET `en/`
- [ ] Templates `.tpl` étendent `layout.tpl` avec blocs `nocache`
- [ ] `|escape` sur toutes les sorties utilisateur dans les templates
- [ ] `{csrf_field()}` dans chaque formulaire POST
- [ ] Attributs RGAA : `aria-label`, `aria-required`, `role="alert"`, `scope="col"`
