---
layout: post
title: Multiple Form Pages for CKAN Dataset Metadata Entry
categories: [CKAN]
redirect_from:
excerpt_separator: <!--more-->
---

<img src="/images/formpages.jpg" alt="ckanext-scheming 3.0 form pages">

With [ckanext-scheming](https://github.com/ckan/ckanext-scheming) 3.0
[CKAN](https://ckan.org) dataset metadata forms may now be split across multiple pages using
[`start_form_page`](https://github.com/ckan/ckanext-scheming/blob/master/README.md#start_form_page).

[Example ckanext-scheming schema](https://github.com/ckan/ckanext-scheming/blob/master/ckanext/scheming/ckan_formpages.yaml):

```yaml
- start_form_page:
    title: Detailed Metadata
    description:
      These fields improve search and give users important links

  field_name: address
  label: Address
```

Each `start_form_page` block marks a field at the beginning of a new page of fields.
All following fields without a new `start_form_page` block will appear on the same
page. This lets us maintain compatibility with existing schemas and makes it easy
to reassign pages over time without affecting how metadata appears in the API.

<!--more-->

This ckanext-scheming release also comes with:

- `datastore_additional_choices` option for adding static choices to a dynamic choice list
- new `markdown` and `radio` field form snippets and presets
- automated tests against CKAN 2.8 (py2), 2.9 (py2), 2.9 and 2.10

and many other fixes and improvements. See the
[changelog](https://github.com/ckan/ckanext-scheming/blob/master/CHANGELOG.md) for all the details.

## Acknowledgements

Some features in this release were sponsored by
[Texas Natural Resources Information System](https://tnris.org/),
the [Texas Water Development Board](https://www.twdb.texas.gov/),
[AppGeo](https://www.appgeo.com/) and [datHere](https://dathere.com/)
in connection with the upcoming
[Texas Water Data Hub project](https://internetofwater.org/blog/building-the-texas-water-data-hub-from-the-ground-up/).
