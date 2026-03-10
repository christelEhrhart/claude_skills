---
name: ci4-model
description: >
  Créer ou modifier des modèles CodeIgniter 4 avec gestion multi-tenant, relations,
  requêtes complexes et migrations. Utiliser ce skill pour tout ce qui concerne
  la base de données : requêtes SQL, Query Builder, scopes, jointures, agrégats,
  seeds, ou pour concevoir un schéma de table.
  Mots-clés déclencheurs : modèle, model, migration, requête, SQL, Query Builder,
  jointure, relation, tenant, schéma, table, champs, seed, données de test.
---

# Modèles CI4 — Base de données multi-tenant

> Ce skill s'appuie sur **ci4-architecture** pour les conventions de nommage.
> Tables préfixées `{module}_{objet}`, champs préfixés `{objet}_`.

---

## BaseModel — socle commun

Tous les modèles héritent de `App\Models\BaseModel` :

```php
<?php

declare(strict_types=1);

namespace App\Models;

use CodeIgniter\Model;

abstract class BaseModel extends Model
{
    protected bool $boolApplyTenantScope = true;

    protected function initialize(): void
    {
        if ($this->boolApplyTenantScope) {
            $intTenantId = service('tenant')->getId();
            $strField    = $this->table . '_tenant_id';
            $this->where($strField, $intTenantId);
        }
    }

    /**
     * Trouve un enregistrement ou lève PageNotFoundException.
     */
    public function findOrFail(int $intId): mixed
    {
        $objResult = $this->find($intId);

        if ($objResult === null) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException(
                "Enregistrement #$intId introuvable dans {$this->table}."
            );
        }

        return $objResult;
    }

    /**
     * Injecte automatiquement tenant_id avant insertion.
     */
    protected function beforeInsert(array $arrData): array
    {
        if ($this->boolApplyTenantScope) {
            $strField = $this->table . '_tenant_id';
            $arrData['data'][$strField] = service('tenant')->getId();
        }
        return $arrData;
    }
}
```

---

## Modèle standard

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Models;

use App\Models\BaseModel;
use App\Modules\Crm\Entities\Contact;

class ContactsModel extends BaseModel
{
    protected $table          = 'crm_contacts';
    protected $primaryKey     = 'contacts_id';
    protected $returnType     = Contact::class;
    protected $useSoftDeletes = true;

    protected $allowedFields  = [
        'contacts_tenant_id',
        'contacts_first_name',
        'contacts_last_name',
        'contacts_email',
        'contacts_phone',
        'contacts_company_id',
        'contacts_is_active',
    ];

    protected $useTimestamps = true;
    protected $createdField  = 'contacts_created_at';
    protected $updatedField  = 'contacts_updated_at';
    protected $deletedField  = 'contacts_deleted_at';

    protected $validationRules = [
        'contacts_last_name' => 'required|max_length[100]',
        'contacts_email'     => [
            'rules' => 'required|valid_email|is_unique[crm_contacts.contacts_email,contacts_id,{contacts_id}]',
            'errors' => [
                'is_unique' => 'Cette adresse e-mail est déjà utilisée.',
            ],
        ],
    ];

    // -----------------------------------------------------------------------
    // Scopes / méthodes métier
    // -----------------------------------------------------------------------

    /** Retourne uniquement les contacts actifs. */
    public function actifs(): static
    {
        return $this->where('contacts_is_active', 1);
    }

    /** Recherche full-text sur nom, prénom, email. */
    public function recherche(string $strTerme): static
    {
        $strTerme = '%' . $strTerme . '%';
        return $this->groupStart()
            ->like('contacts_last_name', $strTerme, 'both', false, true)
            ->orLike('contacts_first_name', $strTerme, 'both', false, true)
            ->orLike('contacts_email', $strTerme, 'both', false, true)
            ->groupEnd();
    }

    /**
     * Liste paginée avec jointure société.
     *
     * @return array{arrContacts: Contact[], objPager: \CodeIgniter\Pager\Pager}
     */
    public function listeAvecSociete(int $intPerPage = 20): array
    {
        $arrContacts = $this
            ->select('crm_contacts.*, crm_companies.companies_name')
            ->join('crm_companies', 'companies_id = contacts_company_id', 'left')
            ->orderBy('contacts_last_name', 'ASC')
            ->paginate($intPerPage);

        return [
            'arrContacts' => $arrContacts,
            'objPager'    => $this->pager,
        ];
    }
}
```

---

## Relations courantes

### Appartient à (belongsTo) — manuel
```php
public function getSociete(): ?Company
{
    if (empty($this->attributes['contacts_company_id'])) {
        return null;
    }
    $objModel = new CompaniesModel();
    return $objModel->find($this->attributes['contacts_company_id']);
}
```

### A plusieurs (hasMany) — dans le modèle parent
```php
public function getContacts(int $intCompanyId): array
{
    return $this->where('contacts_company_id', $intCompanyId)->findAll();
}
```

---

## Requêtes Query Builder — exemples

```php
// Comptage avec filtre tenant (automatique via BaseModel)
$intCount = $this->where('contacts_is_active', 1)->countAllResults();

// Jointure multiple
$arrResultats = $this->db->table('crm_contacts c')
    ->select('c.contacts_id, c.contacts_last_name, co.companies_name, u.users_email')
    ->join('crm_companies co', 'co.companies_id = c.contacts_company_id', 'left')
    ->join('core_users u', 'u.users_id = c.contacts_user_id', 'left')
    ->where('c.contacts_tenant_id', service('tenant')->getId())
    ->whereNull('c.contacts_deleted_at')
    ->orderBy('c.contacts_last_name')
    ->get()->getResultObject();

// Sous-requête
$objSubQuery = $this->db->table('crm_contacts')
    ->select('contacts_company_id')
    ->where('contacts_is_active', 1)
    ->groupBy('contacts_company_id');

$arrSocietesActives = $this->db->table('crm_companies')
    ->whereIn('companies_id', $objSubQuery)
    ->get()->getResultObject();

// Transaction
$this->db->transStart();
$this->save($arrContact);
$objInteractionsModel->save($arrInteraction);
$boolOk = $this->db->transComplete();
```

---

## Migration complète avec index et clés étrangères

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
            'contacts_id' => [
                'type'           => 'INT',
                'constraint'     => 10,
                'unsigned'       => true,
                'auto_increment' => true,
            ],
            'contacts_tenant_id' => [
                'type'       => 'INT',
                'constraint' => 10,
                'unsigned'   => true,
            ],
            'contacts_company_id' => [
                'type'       => 'INT',
                'constraint' => 10,
                'unsigned'   => true,
                'null'       => true,
            ],
            'contacts_first_name' => ['type' => 'VARCHAR', 'constraint' => 100, 'null' => true],
            'contacts_last_name'  => ['type' => 'VARCHAR', 'constraint' => 100],
            'contacts_email'      => ['type' => 'VARCHAR', 'constraint' => 255, 'null' => true],
            'contacts_phone'      => ['type' => 'VARCHAR', 'constraint' => 30,  'null' => true],
            'contacts_is_active'  => ['type' => 'TINYINT', 'constraint' => 1, 'default' => 1],
            'contacts_created_at' => ['type' => 'DATETIME', 'null' => true],
            'contacts_updated_at' => ['type' => 'DATETIME', 'null' => true],
            'contacts_deleted_at' => ['type' => 'DATETIME', 'null' => true],
        ]);

        $this->forge->addPrimaryKey('contacts_id');
        $this->forge->addKey(['contacts_tenant_id', 'contacts_deleted_at']);
        $this->forge->addKey('contacts_email');
        $this->forge->addForeignKey('contacts_tenant_id', 'core_tenants', 'tenants_id', 'RESTRICT', 'CASCADE');
        $this->forge->addForeignKey('contacts_company_id', 'crm_companies', 'companies_id', 'SET NULL', 'CASCADE');

        $this->forge->createTable('crm_contacts', true, ['ENGINE' => 'InnoDB', 'CHARSET' => 'utf8mb4']);
    }

    public function down(): void
    {
        $this->forge->dropTable('crm_contacts', true);
    }
}
```

---

## Seeder — données de test

```php
<?php

declare(strict_types=1);

namespace App\Modules\Crm\Database\Seeds;

use CodeIgniter\Database\Seeder;

class ContactsSeeder extends Seeder
{
    public function run(): void
    {
        $intTenantId = 1; // tenant de démo

        $arrContacts = [
            [
                'contacts_tenant_id'  => $intTenantId,
                'contacts_first_name' => 'Alice',
                'contacts_last_name'  => 'Dupont',
                'contacts_email'      => 'alice.dupont@example.com',
                'contacts_is_active'  => 1,
            ],
            [
                'contacts_tenant_id'  => $intTenantId,
                'contacts_first_name' => 'Bob',
                'contacts_last_name'  => 'Martin',
                'contacts_email'      => 'bob.martin@example.com',
                'contacts_is_active'  => 1,
            ],
        ];

        $this->db->table('crm_contacts')->insertBatch($arrContacts);
    }
}
```

---

## Checklist modèle

- [ ] Hérite de `BaseModel` (pas directement de `Model`)
- [ ] `$table` en `{module}_{objet}` minuscule
- [ ] `$primaryKey` en `{objet}_id`
- [ ] `$returnType` pointe vers l'Entité du module
- [ ] `$allowedFields` liste **tous** les champs modifiables
- [ ] `$useSoftDeletes = true` + `$deletedField` défini
- [ ] Règles de validation dans `$validationRules`
- [ ] `beforeInsert()` hérité de BaseModel injecte `tenant_id`
- [ ] Scopes/méthodes métier nommées en camelCase français si pertinent
- [ ] Migration avec tous les index et FK nécessaires
