---
title: Basic usage
summary: How to set up your taxonomy relations and fields
---

# Basic usage

The main model you'll be interacting with is [`TaxonomyTerm`](api:SilverStripe\Taxonomy\TaxonomyTerm). This class represents the actual taxonomy terms you will apply to your data.

A taxonomy is of extremely limited use by itself. To make use of it, you need to associate it with
`DataObject` models in your site.

To add the the ability to associate a model with `TaxonomyTerm`, you need to add the many-many relation:

```php
namespace App\Model;

use SilverStripe\Forms\TreeMultiselectField;
use SilverStripe\ORM\DataObject;
use SilverStripe\Taxonomy\TaxonomyTerm;

class MyModel extends DataObject
{
    // ...
    private static $many_many = [
        'Terms' => TaxonomyTerm::class,
    ];

    public function getCMSFields()
    {
        $fields = parent::getCMSFields();
        // ...
        $fields->addFieldToTab('Root.Main', TreeMultiselectField::create(
            'Terms',
            $this->fieldLabel('Terms'),
            TaxonomyTerm::class
        ));
        return $fields;
    }
}
```

Run a `dev/build?flush=all` and you'll be able to add taxonomy terms to your record - but now you need to create some terms! See [the userhelp documentation](https://userhelp.silverstripe.org/en/optional_features/taxonomies/) for information about functionality from a content author perspective.

## Filtering by type

If you have implemented taxonomy types, you can filter them to ensure that only taxonomy terms of a given type or types can be selected.
This can be useful, for example, if you want to separate your terms between files/images/documents and CMS pages.

> [!WARNING]
> This relies on `TaxonomyType` records which have specific names being set up in the CMS.
> You will need to coordinate with content authors to ensure these are always available, or else
> ensure they are created by default by [populating defaults](https://docs.silverstripe.org/en/developer_guides/model/how_tos/dynamic_default_fields/)
> and ensure they cannot be deleted by [implementing an Extension](https://docs.silverstripe.org/en/developer_guides/extending/extensions/)
> with the `canDelete()` method.

To implement this filtering, call [`TreeMultiselectField::setFilterFunction()`](api:SilverStripe\Forms\TreeMultiselectField::setFilterFunction()) with the filtering logic:

```php
use SilverStripe\Forms\TreeMultiselectField;
use SilverStripe\Taxonomy\TaxonomyTerm;
use SilverStripe\Taxonomy\TaxonomyType;

/** @var TreeMultiselectField $treeField */
$typeID = TaxonomyType::get()->find('Name', 'CMS Page')?->ID;
$treeField->setFilterFunction(fn (TaxonomyTerm $term) => $term->TypeID === $typeID);
```

## Showing taxonomy terms

So you've got a set of terms associated with a page, and you want to show them on your site. You can loop through them
like any other relation:

```ss
<% loop $Terms %>
    <span class="tag">$Name</span>
<% end_loop %>
```

## Taxonomy Navigation

You might want a page that acts as a parent to all pages with a certain tag. This way it can act as a dynamic directory
that will always display content about a certain topic, automatically adding pages as they get created.

### Using a custom page type

To do so, you'll need to create a new page type (called `TaxonomyDirectory` here) and specify the tag or tags that
you'd like the page to display. In this example we'll just re-use the existing many-many relationship on `Page` but if
it's necessary for you to have two then you can create an extra many-many relationship.

The only two functions that this page type needs to define are `stageChildren` and `liveChildren`. Instead of getting
children from the usual parent-child relationship, they look up children based on their taxonomy:

```php
class TaxonomyDirectory extends Page
{
    public function stageChildren($showAll = false)
    {
        return Page::get()
            ->exclude('ID', $this->ID)
            ->filter(['Terms.ID' => $this->Terms()->getIDList()]);
    }

    public function liveChildren($showAll = false, $onlyDeletedFromStage = false)
    {
        return $this->stageChildren($showAll);
    }
}
```

### Using the provided default controller implementation

If you want to use the provided directory implementation named `TaxonomyDirectoryController`, more or less based on the previous example, the only thing
you need to do is enable access to the controller functions using a custom `routes.yml` file

```yaml

---
Name: directoryroutes
After: '#coreroutes'
---
SilverStripe\Control\Director:
  rules:
    'tag/$ID!': 'SilverStripe\Taxonomy\Controllers\TaxonomyDirectoryController'
```

In this example, any request made to `/tag/<TAG>` would render a page which contains a list of pages that uses the
Taxonomy Term specified by `<TAG>`. You can override the provided template `TaxonomyDirectoryController.ss`
in your own theme.

Currently the following variables are available to the template;
1. `Title` - the Taxonomy Term you are searching for
1. `Term` - the Taxonomy Term you are searching for
1. `Pages` - a list of `Page` objects

An example for a template;
```html

<h1>Taxonomy directory</h1>

<h2>Results for '$Term'</h2>

<ul>
<% loop $Pages %>
    <li><a href="$Link">$Title</a></li>
<% end_loop %>
</ul>


```

#### Using a URL Segment

The provided directory implementation has an option to use a URL-friendly field instead of `Name`. You can enable this in the above example by adding this to your project config `.yml`
```yaml
SilverStripe\Taxonomy\Controllers\TaxonomyDirectoryController:
  lookup_relation_field: 'Terms.URLSegment'
SilverStripe\Taxonomy\TaxonomyTerm:
  extensions:
    - SilverStripe\Taxonomy\Extensions\TaxonomyTermUrlExtension
```

Note: the default directory implementation assumes that you've setup a relation called `Terms` in your `Page.php` as in the example above.

If you're using a different class to `Page`, such as the cwp/cwp module which defines the `Terms` relation on the `BasePage` class you can specify the class using `directory_class` as below
```yaml
SilverStripe\Taxonomy\Controllers\TaxonomyDirectoryController:
  directory_class: CWP\CWP\PageTypes\BasePage
SilverStripe\Taxonomy\TaxonomyTerm:
  extensions:
    - SilverStripe\Taxonomy\Extensions\TaxonomyTermUrlExtension
```
