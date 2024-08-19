# Self-referential table-based categoricals

As with
[self-referential foreign key definitions](https://datapackage.org/standard/table-schema/#foreignKeys),
the hierarchical structure of table-based categoricals can be self-referential.

The [datapackage.json](./datapackage.json) file in the current directory
demonstrates a self-referential table-based categorical.

In the resource `medication-log` the variable `medication_id` references the
`medication_id` field in the `medication-types` resource. In the
`medication-types` resource, the `parent_code` field also references the
`medication_id` field in the same resource.

The self-reference looks like this in the `parent_code` field:

```json
{
  "name": "parent_id",
  "type": "string",
  "categories": {
    "valueField": "medication_id",
    "labelField": "name"
  }
}
```

As you can see, the `resource` property is omitted in the `categories` object.
This is exactly how self-references are handled in `foreignKeys` definitions, as
mentioned above.
