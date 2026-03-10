---
name: ci4-module
description: >
  Créer ou scaffolder un nouveau module HMVC dans l'application CodeIgniter 4.
  Utiliser ce skill quand on demande de créer un module, un CRUD, un contrôleur,
  un modèle ou une entité dans un module existant ou nouveau.
  Mots-clés déclencheurs : nouveau module, créer module, scaffolding, CRUD,
  contrôleur module, modèle module, entité, migration, routes module.
---

# Créer un module HMVC — CodeIgniter 4

> Ce skill s'appuie sur **ci4-architecture** pour les conventions globales.
> Lire ci4-architecture en premier si les conventions de base ne sont pas en mémoire.

---

## Structure complète d'un module

```
app/Modules/{NomModule}/
├── Config/
│   └── Routes.php
├── Controllers/
│   └── {NomModule}Controller.php
├── Models/
│   └── {Objet}Model.php
├── Entities/
│   └── {Objet}.php
├── Language/
│   ├── fr/
│   │   └── {Objet}.php
│   └── en/
│       └── {Objet}.php
└── Database/
    └── Migrations/
        └── {timestamp}_{NomModule}_create_{table}.php
```

---

## 1. Routes — `Config/Routes.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Config;

$routes->group('crm', ['namespace' => 'App\Modules\Crm\Controllers', 'filter' => 'auth'], function ($routes) {
    $routes->get('contacts',          'ContactsController::index');
    $routes->get('contacts/new',      'ContactsController::create');
    $routes->post('contacts',         'ContactsController::store');
    $routes->get('contacts/(:num)',   'ContactsController::show/$1');
    $routes->get('contacts/(:num)/edit', 'ContactsController::edit/$1');
    $routes->post('contacts/(:num)',  'ContactsController::update/$1');
    $routes->delete('contacts/(:num)', 'ContactsController::delete/$1');
});
```

---

## 2. Contrôleur — `Controllers/{Objet}Controller.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Controllers;

use App\Controllers\BaseController;
use App\Modules\Crm\Models\ContactsModel;
use App\Modules\Crm\Entities\Contact;

class ContactsController extends BaseController
{
    private ContactsModel $objModel;

    public function __construct()
    {
        $this->objModel = new ContactsModel();
    }

    public function index(): string
    {
        $arrContacts = $this->objModel->findAll();

        $objSmarty = service('smarty');
        $objSmarty->assign('arrContacts', $arrContacts);
        $objSmarty->assign('strPageTitle', lang('Contacts.title'));

        return $objSmarty->fetch('modules/crm/contacts_index.tpl');
    }

    public function create(): string
    {
        $objSmarty = service('smarty');
        $objSmarty->assign('strPageTitle', lang('Contacts.add'));
        $objSmarty->assign('objContact', new Contact());

        return $objSmarty->fetch('modules/crm/contacts_form.tpl');
    }

    public function store(): \CodeIgniter\HTTP\RedirectResponse
    {
        $arrData = $this->request->getPost();

        if (! $this->objModel->save($arrData)) {
            return redirect()->back()
                ->withInput()
                ->with('arrErrors', $this->objModel->errors());
        }

        return redirect()->to('/crm/contacts')
            ->with('strSuccess', lang('Contacts.saved'));
    }

    public function show(int $intId): string
    {
        $objContact = $this->objModel->findOrFail($intId);

        $objSmarty = service('smarty');
        $objSmarty->assign('objContact', $objContact);
        $objSmarty->assign('strPageTitle', lang('Contacts.view'));

        return $objSmarty->fetch('modules/crm/contacts_show.tpl');
    }

    public function edit(int $intId): string
    {
        $objContact = $this->objModel->findOrFail($intId);

        $objSmarty = service('smarty');
        $objSmarty->assign('objContact', $objContact);
        $objSmarty->assign('strPageTitle', lang('Contacts.edit'));

        return $objSmarty->fetch('modules/crm/contacts_form.tpl');
    }

    public function update(int $intId): \CodeIgniter\HTTP\RedirectResponse
    {
        $objContact = $this->objModel->findOrFail($intId);
        $arrData    = $this->request->getPost();

        if (! $this->objModel->update($intId, $arrData)) {
            return redirect()->back()
                ->withInput()
                ->with('arrErrors', $this->objModel->errors());
        }

        return redirect()->to('/crm/contacts')
            ->with('strSuccess', lang('Contacts.updated'));
    }

    public function delete(int $intId): \CodeIgniter\HTTP\RedirectResponse
    {
        $this->objModel->delete($intId);

        return redirect()->to('/crm/contacts')
            ->with('strSuccess', lang('Contacts.deleted'));
    }
}
```

---

## 3. Modèle — `Models/{Objet}Model.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Models;

use App\Models\BaseModel;
use App\Modules\Crm\Entities\Contact;

class ContactsModel extends BaseModel
{
    protected $table         = 'crm_contacts';
    protected $primaryKey    = 'contacts_id';
    protected $returnType    = Contact::class;
    protected $useSoftDeletes = true;

    protected $allowedFields = [
        'contacts_tenant_id',
        'contacts_first_name',
        'contacts_last_name',
        'contacts_email',
        'contacts_phone',
        'contacts_is_active',
    ];

    protected $useTimestamps  = true;
    protected $createdField   = 'contacts_created_at';
    protected $updatedField   = 'contacts_updated_at';
    protected $deletedField   = 'contacts_deleted_at';

    // Validation
    protected $validationRules = [
        'contacts_last_name' => 'required|max_length[100]',
        'contacts_email'     => 'required|valid_email|is_unique[crm_contacts.contacts_email,contacts_id,{contacts_id}]',
    ];

    protected $validationMessages = [];

    /**
     * Surcharge : injection automatique du tenant_id à la création.
     */
    protected function beforeInsert(array $arrData): array
    {
        $arrData['data']['contacts_tenant_id'] = service('tenant')->getId();
        return $arrData;
    }

    /**
     * Surcharge : filtre automatique par tenant sur toutes les requêtes.
     */
    protected function initialize(): void
    {
        $this->where('contacts_tenant_id', service('tenant')->getId());
    }

    /**
     * Trouve un enregistrement ou lève une exception 404.
     */
    public function findOrFail(int $intId): Contact
    {
        $objContact = $this->find($intId);

        if ($objContact === null) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException(
                lang('Contacts.not_found', [$intId])
            );
        }

        return $objContact;
    }
}
```

---

## 4. Entité — `Entities/{Objet}.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Entities;

use CodeIgniter\Entity\Entity;

class Contact extends Entity
{
    protected $attributes = [
        'contacts_id'         => null,
        'contacts_tenant_id'  => null,
        'contacts_first_name' => null,
        'contacts_last_name'  => null,
        'contacts_email'      => null,
        'contacts_phone'      => null,
        'contacts_is_active'  => true,
        'contacts_created_at' => null,
        'contacts_updated_at' => null,
        'contacts_deleted_at' => null,
    ];

    protected $casts = [
        'contacts_id'        => 'integer',
        'contacts_tenant_id' => 'integer',
        'contacts_is_active' => 'boolean',
    ];

    /**
     * Retourne le nom complet du contact.
     */
    public function getFullName(): string
    {
        return trim(
            ($this->attributes['contacts_first_name'] ?? '') . ' ' .
            ($this->attributes['contacts_last_name'] ?? '')
        );
    }
}
```

---

## 5. Migration — `Database/Migrations/`

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateCrmContacts extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'contacts_id'         => ['type' => 'INT', 'constraint' => 10, 'unsigned' => true, 'auto_increment' => true],
            'contacts_tenant_id'  => ['type' => 'INT', 'constraint' => 10, 'unsigned' => true],
            'contacts_first_name' => ['type' => 'VARCHAR', 'constraint' => 100, 'null' => true],
            'contacts_last_name'  => ['type' => 'VARCHAR', 'constraint' => 100],
            'contacts_email'      => ['type' => 'VARCHAR', 'constraint' => 255, 'null' => true],
            'contacts_phone'      => ['type' => 'VARCHAR', 'constraint' => 30, 'null' => true],
            'contacts_is_active'  => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 1],
            'contacts_created_at' => ['type' => 'DATETIME', 'null' => true],
            'contacts_updated_at' => ['type' => 'DATETIME', 'null' => true],
            'contacts_deleted_at' => ['type' => 'DATETIME', 'null' => true],
        ]);

        $this->forge->addPrimaryKey('contacts_id');
        $this->forge->addKey('contacts_tenant_id');
        $this->forge->addKey('contacts_email');

        $this->forge->createTable('crm_contacts');
    }

    public function down(): void
    {
        $this->forge->dropTable('crm_contacts');
    }
}
```

---

## 6. Fichiers de langue — `Language/fr/{Objet}.php`

```php
<?php

declare(strict_types=1);

// app/Modules/Crm/Language/fr/Contacts.php
return [
    'title'       => 'Contacts',
    'add'         => 'Ajouter un contact',
    'edit'        => 'Modifier le contact',
    'view'        => 'Fiche contact',
    'saved'       => 'Contact enregistré avec succès.',
    'updated'     => 'Contact mis à jour.',
    'deleted'     => 'Contact supprimé.',
    'not_found'   => 'Contact #{0} introuvable.',
    'confirm_delete' => 'Supprimer ce contact ?',
    'fields' => [
        'first_name' => 'Prénom',
        'last_name'  => 'Nom',
        'email'      => 'Adresse e-mail',
        'phone'      => 'Téléphone',
        'is_active'  => 'Actif',
    ],
];
```

---

## Checklist création d'un module

- [ ] Répertoire `app/Modules/{NomModule}/` créé avec tous les sous-dossiers
- [ ] `Config/Routes.php` — routes préfixées par le nom du module, filtre `auth`
- [ ] Contrôleur — namespace correct, variables préfixées
- [ ] Modèle — `$table`, `$primaryKey`, `$returnType`, filtre tenant dans `initialize()`
- [ ] Entité — `$attributes` exhaustif, `$casts` pour les types
- [ ] Migration — tous les champs systèmes inclus (tenant_id, timestamps, deleted_at)
- [ ] Fichiers de langue `fr/` et `en/`
- [ ] Templates Smarty dans `app/Views/modules/{module}/`
- [ ] Module enregistré dans `app/Config/Modules.php` (si auto-discover désactivé)
