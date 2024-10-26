---
layout: post
title: Taming Dynamic Data with Table Designer
categories: [CKAN]
image: /images/tdb1.png
excerpt_separator: <!--more-->
---

CKAN 2.11 introduces Table Designer: form builder and enforced validation for your data

<img src="/images/tdb4.png" alt="Data Dictionary in Excel" width=600>

This article also appears in the CKAN Blog: [Taming Dynamic Data with Table Designer](https://ckan.org/blog/taming-dynamic-data-with-table-designer) and Table Designer was presented in video form at [CKAN Monthly Live 22](https://www.youtube.com/watch?v=9eD7HDS9PWk&t=362s)

## Publishing high-quality data is hard.

- Data schemas, custom rules, and enforced validation help

Publishing high-quality data over time is much harder.

- Data collected changes
- Validation rules change
- Publishers and tools producing the data change
- The way we use the data changes
- We want use old data and new data together

In CKAN, time series data and regularly updated data is often published as a new CSV or Excel file every release, either replacing an existing file or being added to a growing collection.

<!--more-->

When publishing separate files with CKAN we can use [ckanext-validation](https://github.com/ckan/ckanext-validation) to check our data against a [JSON schema](https://json-schema.org/) but this requires someone checking the reports and fixing errors when they are found. Also, JSON schema is technical and difficult for many users and has limited validation rules.

Even *with* validation, separate files are a challenge for users. For time series data every user first cleans and combines all files themselves, hoping that each release is compatible with the last so they have time left to do their actual work. For frequently updated data the file format or structure might have changed leaving users’ existing tools broken or needing continuous maintenance.

## Table Designer is Schema-First

Instead of flagging bad data after it’s uploaded, Table Designer rejects bad data as it’s uploaded (or even before it’s uploaded). So the first step in creating a Table Designer resource is to *design your table*:

<img src="/images/tdb1.png" alt="Data Dictionary | Add Field | Choice" width=600>

Create fields with standard types and constraints, or even your own [custom types](https://docs.ckan.org/en/2.11/extensions/custom-columns-constraints.html#custom-columns-constraints), using this web form. Set constraints like minimum, maximum, and required fields. Add a list of choices to choice fields. Choose primary key fields, values that are guaranteed unique across the table. Provide friendly labels and helpful descriptions for each field.

Table Designer creates an empty table for your data in the [DataStore](https://docs.ckan.org/en/2.11/maintaining/datastore.html) database with your types and constraints applied. Your labels, descriptions, and constraints are published in the Data Dictionary.

As an example, here are fields for collecting municipal bicycle counter time-series data. We want to publish daily counts from each location so we use a date and location as the primary key:

<img src="/images/tdb2.png" alt="Data Dictionary Preview">

## Live Validation in a Spreadsheet

With our table created we have a few options for loading data. We can use [ckanext-excelforms](https://docs.ckan.org/en/2.11/maintaining/table-designer.html#creating-and-updating-rows-with-ckanext-excelforms) to generate a template spreadsheet for loading up to a few thousand records at a time. This template may be used with Excel, Google Docs, LibreOffice etc. to paste in data from another source and it immediately reports on data errors *before uploading*.

Data types, custom constraints, required fields, and duplicate primary keys are all checked in the template spreadsheet and highlighted for the user. Choice fields are provided as drop-down choice lists. Errors are shown in red, missing required fields are shown in blue:

<img src="/images/tdb3.png" alt="Excel Template">

These template spreadsheets let data experts use a familiar tool to collect, combine, and clean up data before it’s published.

Once errors are corrected and data is uploaded, Table Designer presents the data in a [DataTables view](https://docs.ckan.org/en/2.11/maintaining/data-viewer.html#datatables-view) where it can be sorted, searched, and filtered. Rows can also be selected for editing and downloaded as pre-filled spreadsheet templates when updates are required:

<img src="/images/tdb5.png" alt="DataTables view with search">

## Auto-generated Forms

Simple things should be easy too! If we want to add, edit or delete only a few rows of data Table Designer provides auto-generated web forms:

<img src="/images/tdb6.png" alt="Data entry form">

Data can be added, updated or deleted without leaving CKAN and is immediately available for users.

## Use Existing DataStore APIs

Table Designer is built on the CKAN DataStore so we can use the existing [DataStore APIs](https://docs.ckan.org/en/2.11/maintaining/datastore.html#the-data-api) to create tables, update, retrieve and delete data.

Table Designer automatically extends the generated API documentation on the resource page to include examples from real data and examples with updating and deleting. This helps encourage users to integrate with existing applications and eliminate manual (and error-prone!) data publishing steps.

## Tracking Changes

With many users contributing to the same published data it can be important to record a history of changes. Use CKAN’s activity plugin with [ckanext-dsaudit](https://docs.ckan.org/en/2.11/maintaining/table-designer.html#tracking-changes-with-ckanext-dsaudit) to record insert, update and delete actions with data previews:

<img src="/images/tdb7.png" alt="Activity log of changes">

## Bringing it Together

Regardless of how data is loaded: entered in a web form, pasted into Excel, or synchronized with DataStore APIs the same types and custom Table Designer constraints apply to guarantee high quality.

Standard types and constraints can even be extended to cover almost any business logic required with custom database types, trigger functions, and error messages.

As data schemas change fields can be added and removed, and data can be migrated more easily.

Publishing time series and dynamic data with Table Designer means users have access to all the data without needing to clean it first. Users can rely on the automatically published Data Dictionary and high-quality data for building new workflows and applications.

We are really excited about this new feature and look forward to seeing how it will be leveraged by users to improve their data publication workflows. If you have any questions or want to share how you are using Table Designer, reach out on the [community channels](https://ckan.org/community).
