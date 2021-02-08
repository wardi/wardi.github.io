---
layout: post
title: Repeating Subfields and Multiple Text with ckanext-scheming
categories: [CKAN]
redirect_from:
excerpt_separator: <!--more-->
---

[ckanext-scheming](https://github.com/ckan/ckanext-scheming) 2.1
now support Datasets with repeating subfields and repeating text fields.
Repeating subfieds support custom snippets and validation just like regular fields.

This work is inspired by the excellent [ckanext-composite](https://github.com/EnviDat/ckanext-composite)
extension and replaces [ckanext-repeating](https://github.com/open-data/ckanext-repeating) `repeating_text`
fields.

You must be using CKAN 2.8 or later and a custom IPackageController plugin to index datasets with repeating
subfields.

<!--more-->

## Repeating Subfields

Repeating subfields let you define a group of fields to repeat within dataset and resource forms.
All of the features available for normal ckanext-scheming fields can be used in subfields.

![ckanext-scheming repeating fields](/images/repeating_subfields.gif)

`repeating_label` may be used to provide a singular label for each group of subfields.

```yaml
- field_name: submission
  label: Submissions
  repeating_label: Submission
  repeating_subfields:

  - field_name: date
    label: Date
    preset: date
    required: true

  - field_name: text
    label: Text
    preset: markdown
    required: true

  - field_name: flags
    label: Flags
    preset: multiple_checkbox
    choices:
    - label: Draft
      value: D
    - label: Approved
      value: A
    - label: Response Required
      value: R
```

Data stored in subfields is represented as lists of JSON objects in the API.

```json
"submission": [
  {
    "date": "2021-02-01",
    "text": "an example submission",
    "flags": [
      "D"
    ]
  },
  {
    "date": "2021-02-05",
    "text": "another one",
    "flags": [
      "A",
      "R"
    ]
  }
],
```

Normal and custom validation rules apply and are displayed in the form by referencing the subfield
group and the field with the error, e.g. "Submission 2: Text: Missing value"

### Indexing Repeating Subfields

Repeating subfields can't be indexed in Solr with CKAN's default schema and indexing code since
they expect extra fields to contain simple string values. Create a new
[IPackageController `before_index` plugin](https://docs.ckan.org/en/2.9/extensions/plugin-interfaces.html?#ckan.plugins.interfaces.IPackageController.before_index)
and Solr schema to handle indexing repeating subfields the best way for your own site.

```python
class SubmissionsIndexPlugin(p.SingletonPlugin):
    """
    Map submission dataset fields to Solr fields
    """
    p.implements(p.IPackageController, inherit=True)

    def before_index(self, data_dict):
        flags = set()
        text = []
        for sub in data_dict.get('submission', []):
            text.append(sub['text'])
            flags |= set(sub['flags'])

        # replace list of dicts with plain text to prevent Solr errors
        data_dict['submission'] = '\n'.join(text)
        # index flags present in any submission
        data_dict['submission_flags] = sorted(flags)

        return data_dict
```

For `submission_flags` to accept multiple values we must add a multivalued field to our Solr schema
`<fields>` configuration:

```xml
<field name="submission_flags" type="string" indexed="true" stored="true" multiValued="true"/>
```

These new fields will now be available for use with CKAN
[advanced search](https://docs.ckan.org/en/2.9/user-guide.html#advanced-search) e.g.
`submission:example` or `submission_flags:D`.

If you don't need advanced search or faceting based on repeating subfields you may use the included
`scheming_nerf_index` plugin. This plugin passes repeating fields to Solr as JSON strings to prevent
indexing errors instead and doesn't require a customized Solr schema.

Future CKAN support for dynamic fields in Solr will simplify this required configuration.

### Future Work

Some features not yet supported:
 - validating the number of subfield groups (e.g. requiring at least one)
 - nesting repeating subfields

If you need these features consider discussing your work on a
[new issue](https://github.com/ckan/ckanext-scheming/issues/new) and working a pull request.


## Multiple Text

Multiple text fields improve on the `repeating_text` fields from ckanext-repeating with a dynamic form
with add and remove buttons.

![ckanext-scheming multiple_text](/images/multiple_text.gif)

```yaml
- field_name: contributors
  label: Contributors
  preset: multiple_text
```

Data stored in repeating text fields is represented as lists of strings in the API.

```json
"contributors": [
  "Person A",
  "Person B",
  "Person C"
],
```

`required: true` may be used to require at least one entry. Per-field and other types of
validation are not yet implemented. Add a comment to the
[multiple text validation issue](https://github.com/ckan/ckanext-scheming/issues/276)
if you would like to work on this feature.
