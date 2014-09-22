#django-datatable-view change log

These logs are also available on GitHub: https://github.com/pivotal-energy-solutions/django-datatable-view/releases

## 0.8.0
This release modifies the way model fields are detected for ORM queries when submitting a search.  Previously, the approach was primarily hard-coded to support only the core Django fields that served as base classes for all others, but this approach failed to provide any value for third-party fields that were built from scratch.

As of this release, ``datatableview.utils`` contains a dictionary ``FIELD_TYPES`` that contains keys for the major field categories, and maps each one to a list of fields that can be treated identically in ORM query terms.  This dictionary can be extended by importing it and appending new fields to a category.

The default registration dictionary looks like this:

```python
FIELD_TYPES = {
    'text': [models.CharField, models.TextField, models.FileField, models.GenericIPAddressField],
    'date': [models.DateField],
    'boolean': [models.BooleanField],
    'integer': [models.IntegerField, models.AutoField],
    'float': [models.FloatField, models.DecimalField],

    # No direct search ORM queries make sense against these field types
    'ignored': [models.ForeignKey],
}
```

If a field type used in a table but isn't found in any of these categories, the old familiar exception will be raised stating that it is an unhandled field type.  The correct solution is to import this registry and append the field class:

```python
from datatableview.utils import FIELD_TYPES
from netfields.fields import InetAddressField
FIELD_TYPES['text'].append(InetAddressField)
```

In the future, looking towards the full 1.0 release, we would like to offer a more sophistocated field type registration that allows the new field to also declare what unique ORM extensions it offers, in the way that a ``CharField`` offers ``__icontains``, or ``InetAddressField`` offers ``__net_contains``, etc.  Until 1.0, we will be limited to offering this simple registration that limits the field categories to those already listed.

## 0.7.3
This release fixes an exception that is raised when using Django 1.7 release candidate, involving the removal of the deprecated ``StrAndUnicode`` utility base class.

## 0.7.2
This release fixes an issue involving ``verbose_name`` values wrapped in a proxy function used by
Django to defer instantiation of an underlying object.  These proxy values could not go through the ``re`` framework without raising errors about "expecting a string or buffer".

## 0.7.1
This release reverts a change to the Javascript that could cause errors in some cases.  A fix will be reintroduced at a later time to correct a potential issue with Chrome and Safari not updating the footer text when a filter is applied.

The ``default_structure.html`` template has received a minor update to include the use of a CSS class ``"display"`` on the table, which how the datatables.js 1.10.0 version has begun to show examples on their documentation website, which gives the table a more modern default appearance.  The bootstrap template is unchanged.

A fix from @michaeldjeffrey should allow the browser's locale setting to control how the number of results is rendered in the table footer.

A fix from @danmac-uk has been merged that adds Django 1.7 support, because of our current use of Django's ``StrAndUnicode`` base class for the template-renderable datatable object.

Various updates were made to the example project including a fix for "None" table headers in the embedded table demo.

New issues are being tracked at the github repository and we're building our milestone goals for release 1.0.  We would like to include a streamlined strategy for column filters, an updated configuration strategy, and new callbacks to simplify support for datatables new ``"DT_RowData"`` key and modifying the internals of how special field types are handled for ORM querying.

## 0.7.0
This release adds Python 3 support with the help of the [six](http://pythonhosted.org/six/) project.  We've taken some time to verify that unicode handling is handled correctly for both Python 2 and 3 simultaneously.

The options provided by a DatatableView are processed and merged with incoming GET data from ajax calls, and the resulting object replaces the view's ``datatable_options`` attribute at runtime.  Previous to this release, that merged object implemented an attribute API to support lookups like ``options.search`` to get the search string.

As of this release, that attribute API has been removed for the sake of simplicity.  Treat ``options`` like a normal dictionary and use the keys and values it contains to read any options.  The only reason the object is a subclass of the base ``dict`` itself is to provide a constructor that accepts a GET query string and normalizes and validates options.  Other than the normalization phase, it is a dictionary, first and foremost.

## 0.6.5
This release fixes a bug with sorting columns backed by concrete db fields not on the local model.  The sorting operation would fail to recognize the field as concrete and would fall back to text-only sorting, which produced unexpected results.

## 0.6.4
This release adds incremental support for Python 3 (not completed yet, but the occational pull request comes in to help get things ready), and fixes a Python 2.6 compatibility issue with string formatting syntax.

The history of changes on each release has also been added to the code repository, available at ``django-datatable-view/CHANGELOG.md``.

## 0.6.3
This release fixes an issue with non-local fields sometimes raising errors when requested for sorting operations over ajax.

## 0.6.2
This release fixes a mistake that was made because the author was overconfident about the jQuery-dataTables API interchangeability.

The return value of the Javascript late initialization function ``datatableview.initialize()`` should now work like this:

```javascript
$(function(){
    var oTable = datatableview.initialize($(".datatable"));
});
```

Release 0.6.1 mistakenly returned an awkward set of nested arrays, leading to a need to re-call the ``dataTable()`` constructor, defeating the purpose of the return value as a convenience.

## 0.6.1
This release adds a return statement to the ``datatableview.initialize()`` javascript function that was added in version 0.6.0.  The returned value is the jQuery collection of objects that are the initialized ``oTable`` objects from the initialization process.  This should support multiple or single initializations with a uniform interaction.

## 0.6.0
This release makes a number of changes for the benefit of flexibility and future development.

#### utils.get_datatable_structure() and DatatableStructure

Unfortunately, I made a decision to reverse the 2nd and 3rd arguments of this utility function.  The only reason the model class was being sent to the utility at all was because of certain default conditions that, if options weren't specified (like a column list, or the preferred ordering), the internals could look up those details and fill them in.

The reality was that supplying the model here was 9 times out of 10 not required.  The argument has been moved to the end of the parameter list, and now accepts ``None`` as a default value, making the signature:

```python
get_datatable_structure(ajax_url, options, model=None)
```

The constructor to the internal context variable object ``DatatableStructure`` matches this signature.

#### Helpers

* ``format_date``: This helper used to have trouble with columns that were ``null=True``, and would likely raise an error, worse even when the ``localize=True`` argument was set.  The helper now aborts sooner in its rendering process if the value is ``None``, returning the blank string.
* ``through_filter``: A new helper that takes a simple callable mapping function (e.g., takes one argument, returns a new value), wrapping that target function so that it is compatible with use as a column callback.  Django template filters are good examples of target functions.  These can not be used directly as callbacks because they don't accept extra ``**kwargs`` that get sent by the caller.  This helper swallows those extra arguments so that the target function can do its work.
* ``itemgetter`` and ``attrgetter``: While still convenient shorthand, these aren't strictly necessary anymore.  ``through_filter`` can wrap the built-in ``operator.itemgetter`` and ``operator.attrgetter`` and accomplish the same thing.  ``itemgetter`` still supports an ``ellipsis="..."`` argument, but more robust behavior can probably be found using ``through_filter(django.template.defaultfilters.truncatechars)``.

#### datatableview.js

Automatic initialization of tables via a jQuery ``document.ready`` event is still enabled by default, but it can be turned off via a global object property.  If the following is executed after you include the ``datatableview.js`` file, but before the jQuery ready handler, no tables will be initialized:

```javascript
datatableview.auto_initialize = false;
```

In this scenario, you can do late initialization and option declaration on your own, using the same global object:

```javascript
var datatableOptions = {};
datatableview.initialize($(".mytable"), datatableOptions);
```

Note that all of the configuration options specified on the view are represented on the DOM in ``data-*`` attributes.  dataTables.js doesn't actually support these, but this ``datatableview.initialize`` function does.  This method of late binding can also replace the need to use the global javascript hook ``confirm_datatable_options`` to specify options for tables that were previously being auto-initialized.

#### Bootstrap integration

To make your life slightly easier, we've included a new template fragment at ``datatableview/bootstrap_structure.html``.  This inclusion template simply adds the Bootstrap-esque ``table striped-table bordered-table`` classes to the main ``<table>`` tag.

To use this template, you can either specify on your view's options ``datatable_options["structure_template"] = "datatableview/bootstrap_structure.html"``, or you can copy the contents of this file into your own overridden ``"datatableview/default_structure.html"`` as a starting point for customization.

#### Tests

This release adds a set of tests for the ``helpers`` and ``utils`` modules, and for the new example project's views.

#### Example project

http://django-datatable-view.appspot.com/ — This is a public instance of the new example project, which includes live demos, descriptions, and short code examples.

The goal of this example project is to replace the incredibly long-winded README that was previously the only real information resource.  While a Django-powered example project might feel more annoying to set up, it's the only way to provide actual live demos, instead of faking them and defeating the purpose of the exercise.

The new example project is included as part of the ``datatableview.tests`` module.  The easiest way to get it running on your own machine is to do something like this:

```bash
$ git clone https://github.com/pivotal-energy-solutions/django-datatable-view.git
$ cd django-datatable-view
$ mkvirtualenv datatableview
(datatableview)$ pip install -r requirements.txt
(datatableview)$ datatableview/tests/example_project/manage.py syncdb
(datatableview)$ datatableview/tests/example_project/manage.py runserver
```

This downloads the latest code, uses virtualenvwrapper to set up a new environment, and installs the basic requirements for the project (Django and python-dateutil).  After that, it's just a matter of sync-ing the sqlite3 database and running the dev server.

## Smaller things

Less important for the average use cases, but maybe good to know:

* Detection of the default column value sent to a helper has been improved.  If values were ``False``-like, they were sometimes ignored as not being present, which led to the helper trying to act upon the row's instance as a fallback value.  This usually didn't produce an intuitive result.
* Due to the above point, the ``key`` argument to a few of the helpers is facing removal sometime in the future.
* Minor parameter-passing bug with the new ``make_xeditable`` helper.
* Added a utility that normalizes field definitions to an internal format.  Unless there was a lot of modification of the ``DatatableView``'s internal methods for deriving column data, this change shouldn't have an outward impact.
* Fixed bug concerning default "show all columns" mode and no data appearing in the rows
* Fixed bug that prevented the same field from being represented in multiple columns.  The field was reduced to a dictionary mapping to column definitions, which resulted in dropping one at random.  All column definitions are now properly respected.
* Fixed bug with Django < 1.5 versions using the ``XEditableDatatableView``.  The asynchronous ajax updates failed due to the use of a Django 1.5 ``update_fields`` parameter to the ``save()`` method.  Django 1.5 will still attempt to use this option, but older versions will not.
* Fixed bug with status code in ``make_xeditable`` responses.

## 0.5.5
This release fixes a mistake in the ``format_date`` helper function that prevented it from returning the expected output in a non-``key`` usage.

Thanks to @foobar76 for raising the issue.

## 0.5.4
Addresses the following issues:

* #19 — Uses py3 ``collections.UserDict`` if possible
* #20 — Defer object_list pagination until serialization time, not in ``apply_queryset_options()``.
* #16 — Built-in support for [x-editable](http://vitalets.github.io/x-editable/) datatables.  See documentation for a new view base class [``XEditableDatatableView``](https://github.com/pivotal-energy-solutions/django-datatable-view#x-editable-datatables) and the new [``make_xeditable()``](https://github.com/pivotal-energy-solutions/django-datatable-view#make_xeditable) helper for details on how to get this working with minimal configuration.

A backwards-incompatible change has been made to the return value of ``get_object_list()`` and ``apply_queryset_options()``, and the parameter type of ``serialize_to_json()``.  The former two methods previously returned a 3-tuple of ``(object_list, total, filtered_total)``, and the latter method received this 3-tuple as a single argument.  This was not ideal, so the former two are now updated to return a list-like object, and the latter method now receives just the python dictionary to serialize.

A new method has been broken out of the ``DatatableView`` request-response workflow:

[get_json_response_object()](https://github.com/pivotal-energy-solutions/django-datatable-view#get_json_response_objectobject_list-total-filtered_total) — This method receives several arguments that will be placed into a dictionary.  If custom fields need to be added to the AJAX response object, you can implement this method, call ``super()`` to get the normal object, and then add to it:

```python
class MyDatatableView(DatatableView):
    datatable_options = { ... }
    def get_json_response_object(self, object_list, *args, **kwargs):
        data = super(MyDatatableView, self).get_json_response_object(object_list, *args, **kwargs)

        # Keep customizations JSON-compatible! :) 
        data.update({
            'special_arg': self.kwargs['special_arg'],
        })
        return data
```

## 0.5.1
Addressed in this bugfix release:

* #17 'console' is undefined error in IE8
* #8 pypi integration? — packaging error excluded relevant package_data items from being included in an sdist release

## 0.5.0
This is the first PyPI-compatible release.
