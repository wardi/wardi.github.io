---
layout: post
title: Repeating Subfields with ckanext-scheming
categories: [CKAN]
redirect_from:
excerpt_separator: <!--more-->
---

[ckanext-scheming](https://github.com/ckan/ckanext-scheming) 2.1
now support Datasets with repeating subfields and repeating text fields.
Repeating subfieds support custom snippets and validation without any changes.

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

Repeating subfields can't be indexed in SOLR with CKAN's default schema and indexing code since
they expect extra fields to contain simple string values. Create a new IPackageController plugin
and SOLR schema to handle indexing repeating subfields the best way for your own site, or use the
included `scheming_nerf_index` plugin that passes repeating fields to SOLR as JSON strings to
prevent indexing errors.

Future CKAN support for dynamic fields in SOLR will simplify this required configuration.

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

`required: true` validation is supported to require at least one entry. Per-field and other types of
validation are not yet implemented. Add a comment to the
[multiple text validation issue](https://github.com/ckan/ckanext-scheming/issues/276)
if you would like to work on this feature.
