---
name: ci4-architecture
description: >
  Architecture et conventions globales de l'application CodeIgniter 4 / PHP 8.3.
  Utiliser ce skill pour toute question sur la structure générale du projet,
  les conventions de nommage, la configuration HMVC, Shield, Smarty, SaaS multi-tenant,
  accessibilité RGAA, ou pour créer/modifier n'importe quel fichier du projet.
  Mots-clés déclencheurs : CodeIgniter, CI4, HMVC, module, Shield, Smarty, tenant,
  architecture, conventions, nommage, SAAS, RGAA.
---

# Architecture CodeIgniter 4 — Conventions du projet

## Stack technique

| Composant       | Version / Détail                        |
|-----------------|------------------------------------------|
| PHP             | 8.3 minimum                             |
| Framework       | CodeIgniter 4 (dernière version stable) |
| Auth            | CodeIgniter Shield                      |
| Vues            | Smarty (moteur de templates)            |
| BDD             | Relationnelle (MySQL/MariaDB)           |
| Architecture    | HMVC — modules dans `app/Modules/`      |
| Déploiement     | SaaS multi-domaines (multi-tenant)      |
| Accessibilité   | RGAA (Référentiel Général d'Accessibilité) |
| Responsive      | Oui — mobile-first                      |

---

## Structure des répertoires

```
app/
├── Config/
├── Controllers/        ← contrôleurs globaux (hors modules)
├── Modules/
│   ├── Core/           ← fonctions transversales (tenant, helpers…)
│   │   ├── Controllers/
│   │   ├── Models/
│   │   ├── Entities/
│   │   ├── Language/
│   │   │   ├── fr/
│   │   │   └── en/
│   │   └── Config/
│   ├── Crm/
│   │   ├── Controllers/
│   │   ├── Models/
│   │   ├── Entities/
│   │   ├── Language/
│   │   │   ├── fr/
│   │   │   └── en/
│   │   └── Config/
│   └── [AutreModule]/
├── Views/              ← layouts globaux Smarty (.tpl)
└── ...
public/
├── index.php
└── themes/
    └── default/        ← assets CSS/JS/images
```

---

## Conventions de nommage

### Variables PHP
Toutes les variables sont **préfixées par leur type** :

| Préfixe  | Type                        | Exemple                        |
|----------|-----------------------------|--------------------------------|
| `$str`   | string                      | `$strName`, `$strEmail`        |
| `$int`   | integer                     | `$intId`, `$intCount`          |
| `$bool`  | boolean                     | `$boolIsActive`, `$boolOk`     |
| `$float` | float / decimal             | `$floatPrice`, `$floatTax`     |
| `$arr`   | array                       | `$arrUsers`, `$arrOptions`     |
| `$obj`   | objet / instance de classe  | `$objModel`, `$objUser`        |
| `$mix`   | type mixte                  | `$mixValue`                    |
| `$null`  | nullable                    | utiliser `?type` en typehint   |

```php
// ✅ Correct
$strFirstName = 'Alice';
$intAge       = 30;
$boolActive   = true;
$arrUsers     = $objModel->findAll();

// ❌ Incorrect
$firstName = 'Alice';
$users     = $model->findAll();
```

### Classes et fichiers
- **Controllers** : `NomModuleController.php` → `ContactsController`
- **Models** : `NomModuleModel.php` → `ContactsModel`
- **Entities** : `NomSingulier.php` → `Contact`
- **Namespaces** : `App\Modules\NomModule\Controllers`

### Routes
Les routes sont définies par module dans `app/Modules/NomModule/Config/Routes.php`.

---

## Base de données

### Préfixage des tables
`{module}_{objet}` en **snake_case minuscule** :
- `crm_contacts`
- `crm_companies`
- `billing_invoices`
- `core_tenants`

### Préfixage des champs
Les champs sont préfixés par le **nom de la table sans le module** :

```sql
-- Table : crm_contacts
contacts_id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
contacts_tenant_id    INT UNSIGNED NOT NULL,
contacts_first_name   VARCHAR(100),
contacts_last_name    VARCHAR(100),
contacts_email        VARCHAR(255),
contacts_is_active    TINYINT(1) DEFAULT 1,
contacts_created_at   DATETIME,
contacts_updated_at   DATETIME,
contacts_deleted_at   DATETIME NULL   -- soft delete

-- Table : crm_companies
companies_id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
companies_tenant_id   INT UNSIGNED NOT NULL,
companies_name        VARCHAR(150),
companies_siret       VARCHAR(14),
```

### Champs systèmes obligatoires dans chaque table
```sql
{table}_id          -- clé primaire
{table}_tenant_id   -- isolation multi-tenant
{table}_created_at  -- horodatage création
{table}_updated_at  -- horodatage modification
{table}_deleted_at  -- soft delete (nullable)
```

---

## Multi-tenant (SaaS)

Chaque requête est isolée par `tenant_id` :

```php
// Dans BaseModel — filtre automatique par tenant
protected function applyTenantScope(Builder $builder): Builder
{
    $intTenantId = service('tenant')->getId();
    return $builder->where($this->table . '_tenant_id', $intTenantId);
}
```

- La résolution du tenant se fait par le **domaine** (sous-domaine ou domaine dédié).
- Le service `tenant` est enregistré dans `app/Config/Services.php`.
- Jamais de données cross-tenant dans les requêtes.

---

## Shield — Authentification

- Utiliser CodeIgniter Shield natif.
- Les groupes/rôles sont définis dans `app/Config/AuthGroups.php`.
- Les routes protégées utilisent le filtre `['filter' => 'auth']`.
- L'utilisateur connecté : `auth()->user()`.
- Étendre l'entité User de Shield pour ajouter `tenant_id` :

```php
// app/Entities/User.php
namespace App\Entities;
use CodeIgniter\Shield\Entities\User as ShieldUser;

class User extends ShieldUser
{
    public function getTenantId(): int
    {
        return (int) $this->attributes['tenant_id'];
    }
}
```

---

## Smarty — Vues

- Les templates ont l'extension `.tpl`.
- Le répertoire des templates compilés : `writable/smarty_cache/`.
- Chaque module peut avoir son sous-répertoire de templates dans `app/Views/modules/{module}/`.
- Transmettre les variables au template :

```php
// Dans un contrôleur
$objSmarty = service('smarty');
$objSmarty->assign('arrUsers', $arrUsers);
$objSmarty->assign('strPageTitle', lang('Crm.contacts.title'));
$objSmarty->display('modules/crm/contacts_list.tpl');
```

- Dans les templates Smarty, les variables conservent leur préfixe :
```smarty
{* contacts_list.tpl *}
<h1>{$strPageTitle|escape}</h1>
{foreach $arrUsers as $objUser}
    <p>{$objUser->contacts_first_name|escape}</p>
{/foreach}
```

---

## Traduction (i18n)

- Chaque module possède son répertoire `Language/fr/` et `Language/en/`.
- Les fichiers de langue retournent un tableau PHP :

```php
// app/Modules/Crm/Language/fr/Contacts.php
return [
    'title'        => 'Liste des contacts',
    'add'          => 'Ajouter un contact',
    'edit'         => 'Modifier le contact',
    'delete'       => 'Supprimer',
    'confirm_delete' => 'Confirmer la suppression ?',
];
```

- Appel dans le code : `lang('Contacts.title')`
- Appel dans Smarty : `{lang key='Contacts.title'}`

---

## Accessibilité RGAA

Règles minimales à respecter dans tous les templates :

1. **Images** : attribut `alt` obligatoire — descriptif ou vide si décoratif.
2. **Formulaires** : chaque `<input>` / `<select>` doit avoir un `<label>` associé (`for` + `id`).
3. **Liens** : texte explicite ou `aria-label`.
4. **Couleurs** : contraste minimum 4.5:1 (texte normal), 3:1 (grand texte).
5. **Navigation clavier** : focus visible sur tous les éléments interactifs.
6. **Structure** : un seul `<h1>` par page, hiérarchie `h2`/`h3` cohérente.
7. **Langue** : attribut `lang` sur la balise `<html>`.
8. **Rôles ARIA** : utiliser `role`, `aria-label`, `aria-expanded`, `aria-current` quand nécessaire.

```smarty
{* Exemple bouton accessible *}
<button type="button"
        aria-label="{lang key='Contacts.delete'} {$objContact->contacts_full_name|escape}"
        aria-controls="modal-confirm-delete">
    <span aria-hidden="true">🗑</span>
</button>
```

---

## Responsive

- **Mobile-first** : breakpoints définis dans les variables CSS / SCSS.
- Grille CSS (Grid ou Flexbox) — pas de framework CSS imposé mais les composants doivent fonctionner sans JavaScript.
- Les tableaux de données utilisent un scroll horizontal sur mobile.

---

## Checklist création d'un fichier

Avant de créer tout fichier, vérifier :
- [ ] Namespace correct (`App\Modules\{Module}\...`)
- [ ] Variables préfixées par leur type
- [ ] Champs BDD préfixés correctement
- [ ] Filtre tenant appliqué dans les modèles
- [ ] Textes passés par `lang()`
- [ ] Templates Smarty avec `|escape` sur les sorties utilisateur
- [ ] Attributs d'accessibilité présents dans les vues
