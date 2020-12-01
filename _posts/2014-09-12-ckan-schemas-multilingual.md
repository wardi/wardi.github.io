---
layout: post
title: 
categories: [Open Data, Python, CKAN]
redirect_from: /article/2014/09/ckan-schemas-and-multilingual/
excerpt_separator: <!--more-->
---


I've been working on two new [CKAN](http://ckan.org/) extensions that I'm really excited about. Since I helped build the custom metadata schema for [http://data.gc.ca/](http://data.gc.ca/) I've never been completely satisfied with the approach we took. These new extensions are the start of a proper solution to the compromises we made.

The new extensions are:

*   [ckanext-scheming](https://github.com/open-data/ckanext-scheming/) data-driven custom schemas
*   [ckanext-fluent](https://github.com/open-data/ckanext-fluent/) multilingual text fields

<!--more-->

## No programming required

[ckanext-scheming](https://github.com/open-data/ckanext-scheming/) provides a way to configure and share CKAN schemas using a JSON schema description that you turn on in your configuration file. This schema lets you customize all the fields and validation of your datasets without even needing to write a new extension in Python.

This is a generalization of the [data-driven schema we used for data.gc.ca](https://github.com/open-data/ckanext-canada/blob/master/ckanext/canada/metadata_schema/schema.json), but now it covers everything including validation rules ways of customizing form fields and the way fields are rendered.

## All the languages

The [always-bilingual](https://github.com/ckan/ckan/wiki/Multilingual-Datasets,-the-Government-of-Canada-approach) nature of [http://data.gc.ca/](http://data.gc.ca/) was baked in to our extension in a way that makes it hard to separate. Also, it doesn't extend to other languages and it's hard to support harvesting from other CKAN instances with this approach.

I spent lots of time speaking to [David Raznick](https://github.com/kindly) about the best way to support multiple language versions of text fields and we settled on using JSON objects instead of strings for multilingual field values. These objects have [BCP 47](http://tools.ietf.org/html/bcp47) language codes as keys and the text in those languages as values.

[ckanext-fluent](https://github.com/open-data/ckanext-fluent/) adds the necessary validators and template snippets to seamlessly integrate these multilingual fields into CKAN datasets.

## Ongoing development

These extensions and many others are rapidly evolving and show the strength of what can be done with CKAN's plugin architecture. Come join the [CKAN technical team](http://ckan.org/about/technical-team/) on [IRC or mailing lists](http://ckan.org/contact/) if you're interested in helping.
