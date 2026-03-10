---
name: ci4-view
description: >
  Créer ou modifier des templates Smarty (.tpl) pour l'application CodeIgniter 4.
  Utiliser ce skill pour tout fichier de vue : liste, formulaire, fiche détail,
  layout, composant partiel. Intègre les règles RGAA, responsive et les conventions
  de nommage des variables Smarty.
  Mots-clés déclencheurs : template, vue, Smarty, .tpl, formulaire, liste, tableau,
  RGAA, accessibilité, aria, responsive, layout, partial.
---

# Templates Smarty — CodeIgniter 4

> Conventions issues de **ci4-architecture**. Les variables dans les templates
> conservent leur préfixe de type (`$arrUsers`, `$objContact`, `$strTitle`…).

---

## Organisation des templates

```
app/Views/
├── layouts/
│   ├── base.tpl          ← layout principal
│   └── auth.tpl          ← layout pages de connexion
├── partials/
│   ├── _header.tpl
│   ├── _nav.tpl
│   ├── _footer.tpl
│   ├── _flash.tpl        ← messages flash
│   └── _pagination.tpl
└── modules/
    └── crm/
        ├── contacts_index.tpl
        ├── contacts_form.tpl
        └── contacts_show.tpl
```

---

## Layout de base — `layouts/base.tpl`

```smarty
<!DOCTYPE html>
<html lang="{$strLang|default:'fr'}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$strPageTitle|escape} — {$strAppName|escape}</title>
    <link rel="stylesheet" href="{$strBaseUrl}public/themes/default/css/app.css">
</head>
<body>
    <a href="#main-content" class="skip-link">{lang key='Layout.skip_to_content'}</a>

    {include file='partials/_header.tpl'}
    {include file='partials/_nav.tpl'}

    <main id="main-content" role="main">
        {include file='partials/_flash.tpl'}
        {block name='content'}{/block}
    </main>

    {include file='partials/_footer.tpl'}

    <script src="{$strBaseUrl}public/themes/default/js/app.js" defer></script>
    {block name='scripts'}{/block}
</body>
</html>
```

---

## Messages flash — `partials/_flash.tpl`

```smarty
{if session()->has('strSuccess')}
    <div role="alert" class="alert alert--success" aria-live="polite">
        <p>{session('strSuccess')|escape}</p>
    </div>
{/if}

{if session()->has('arrErrors')}
    <div role="alert" class="alert alert--error" aria-live="assertive">
        <ul>
            {foreach session('arrErrors') as $strError}
                <li>{$strError|escape}</li>
            {/foreach}
        </ul>
    </div>
{/if}
```

---

## Template liste — `contacts_index.tpl`

```smarty
{extends file='layouts/base.tpl'}

{block name='content'}
<section class="section" aria-labelledby="page-heading">

    <header class="section__header">
        <h1 id="page-heading">{$strPageTitle|escape}</h1>
        <a href="/crm/contacts/new" class="btn btn--primary">
            {lang key='Contacts.add'}
        </a>
    </header>

    {if $arrContacts|@count > 0}
        <div class="table-responsive" role="region" aria-label="{lang key='Contacts.title'}" tabindex="0">
            <table class="table" aria-describedby="table-desc">
                <caption id="table-desc" class="sr-only">
                    {lang key='Contacts.table_caption'}
                </caption>
                <thead>
                    <tr>
                        <th scope="col">{lang key='Contacts.fields.last_name'}</th>
                        <th scope="col">{lang key='Contacts.fields.first_name'}</th>
                        <th scope="col">{lang key='Contacts.fields.email'}</th>
                        <th scope="col">{lang key='Contacts.fields.is_active'}</th>
                        <th scope="col">
                            <span class="sr-only">{lang key='Layout.actions'}</span>
                        </th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $arrContacts as $objContact}
                    <tr>
                        <td>{$objContact->contacts_last_name|escape}</td>
                        <td>{$objContact->contacts_first_name|escape}</td>
                        <td>
                            <a href="mailto:{$objContact->contacts_email|escape}">
                                {$objContact->contacts_email|escape}
                            </a>
                        </td>
                        <td>
                            {if $objContact->contacts_is_active}
                                <span class="badge badge--active" aria-label="{lang key='Layout.active'}">✓</span>
                            {else}
                                <span class="badge badge--inactive" aria-label="{lang key='Layout.inactive'}">✗</span>
                            {/if}
                        </td>
                        <td class="table__actions">
                            <a href="/crm/contacts/{$objContact->contacts_id}"
                               aria-label="{lang key='Contacts.view'} {$objContact->getFullName()|escape}">
                                {lang key='Layout.view'}
                            </a>
                            <a href="/crm/contacts/{$objContact->contacts_id}/edit"
                               aria-label="{lang key='Contacts.edit'} {$objContact->getFullName()|escape}">
                                {lang key='Layout.edit'}
                            </a>
                            <button type="button"
                                    class="btn btn--danger btn--sm"
                                    aria-label="{lang key='Contacts.confirm_delete'} {$objContact->getFullName()|escape}"
                                    data-confirm="{lang key='Contacts.confirm_delete'}"
                                    data-action="/crm/contacts/{$objContact->contacts_id}">
                                {lang key='Layout.delete'}
                            </button>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>

        {include file='partials/_pagination.tpl' objPager=$objPager}

    {else}
        <p class="empty-state">{lang key='Contacts.no_results'}</p>
    {/if}

</section>
{/block}
```

---

## Template formulaire — `contacts_form.tpl`

```smarty
{extends file='layouts/base.tpl'}

{block name='content'}
<section class="section" aria-labelledby="page-heading">

    <h1 id="page-heading">{$strPageTitle|escape}</h1>

    <form method="post"
          action="{if $objContact->contacts_id}/crm/contacts/{$objContact->contacts_id}{else}/crm/contacts{/if}"
          novalidate
          aria-label="{$strPageTitle|escape}">

        {csrf_field}

        <div class="form-group">
            <label for="contacts_last_name" class="form-label">
                {lang key='Contacts.fields.last_name'}
                <span class="required" aria-hidden="true">*</span>
            </label>
            <input type="text"
                   id="contacts_last_name"
                   name="contacts_last_name"
                   class="form-control{if isset($arrErrors.contacts_last_name)} form-control--error{/if}"
                   value="{$objContact->contacts_last_name|escape}"
                   required
                   aria-required="true"
                   {if isset($arrErrors.contacts_last_name)}aria-describedby="err-last-name"{/if}
                   autocomplete="family-name">
            {if isset($arrErrors.contacts_last_name)}
                <p id="err-last-name" class="form-error" role="alert">
                    {$arrErrors.contacts_last_name|escape}
                </p>
            {/if}
        </div>

        <div class="form-group">
            <label for="contacts_first_name" class="form-label">
                {lang key='Contacts.fields.first_name'}
            </label>
            <input type="text"
                   id="contacts_first_name"
                   name="contacts_first_name"
                   class="form-control"
                   value="{$objContact->contacts_first_name|escape}"
                   autocomplete="given-name">
        </div>

        <div class="form-group">
            <label for="contacts_email" class="form-label">
                {lang key='Contacts.fields.email'}
                <span class="required" aria-hidden="true">*</span>
            </label>
            <input type="email"
                   id="contacts_email"
                   name="contacts_email"
                   class="form-control{if isset($arrErrors.contacts_email)} form-control--error{/if}"
                   value="{$objContact->contacts_email|escape}"
                   required
                   aria-required="true"
                   {if isset($arrErrors.contacts_email)}aria-describedby="err-email"{/if}
                   autocomplete="email">
            {if isset($arrErrors.contacts_email)}
                <p id="err-email" class="form-error" role="alert">
                    {$arrErrors.contacts_email|escape}
                </p>
            {/if}
        </div>

        <div class="form-group">
            <label for="contacts_phone" class="form-label">
                {lang key='Contacts.fields.phone'}
            </label>
            <input type="tel"
                   id="contacts_phone"
                   name="contacts_phone"
                   class="form-control"
                   value="{$objContact->contacts_phone|escape}"
                   autocomplete="tel">
        </div>

        <div class="form-group form-group--checkbox">
            <input type="checkbox"
                   id="contacts_is_active"
                   name="contacts_is_active"
                   value="1"
                   class="form-checkbox"
                   {if $objContact->contacts_is_active}checked{/if}>
            <label for="contacts_is_active" class="form-label">
                {lang key='Contacts.fields.is_active'}
            </label>
        </div>

        <div class="form-actions">
            <button type="submit" class="btn btn--primary">
                {lang key='Layout.save'}
            </button>
            <a href="/crm/contacts" class="btn btn--secondary">
                {lang key='Layout.cancel'}
            </a>
        </div>

    </form>

</section>
{/block}
```

---

## Règles RGAA à respecter dans chaque template

| Critère | Implémentation |
|---------|---------------|
| Images | `alt=""` si décoratif, `alt="description"` si informatif |
| Champs de formulaire | `<label for="id">` + `id` sur chaque input |
| Champs obligatoires | `required` + `aria-required="true"` + mention visuelle |
| Erreurs de formulaire | `role="alert"` + `aria-describedby` sur l'input |
| Liens | Texte explicite ou `aria-label` |
| Boutons icônes seuls | `aria-label` obligatoire |
| Tableaux | `<caption>` ou `aria-label`, `scope` sur les `<th>` |
| Navigation clavier | Lien d'évitement en début de page |
| Alertes dynamiques | `aria-live="polite"` ou `aria-live="assertive"` |
| Couleurs | Jamais la couleur seule comme unique information |
| Langue | `lang=` sur `<html>` |

---

## Checklist template

- [ ] `|escape` sur toutes les variables utilisateur
- [ ] `{lang key='...'}` pour tous les textes affichés
- [ ] Variables avec préfixe de type (`$arrItems`, `$objItem`, `$strTitle`)
- [ ] Règles RGAA appliquées
- [ ] Responsive testé (pas de largeur fixe en px)
- [ ] `{csrf_field}` dans chaque formulaire POST
