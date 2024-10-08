# Table-based categoricals

Table-based categoricals are a way to define categorical variables in a separate
table (rather than inline the field definition). This approach is useful for:

- Re-using categorical definitions across multiple fields
- Categorical defintiions that require extra metadata (e.g. hierarchical
  categorical structures)
- Categoricals with many levels

Table-based categoricals are defined by way of an extension of the `categories`
property used by inline categoricals. Table-based categoricals are defined by
setting `categories` to be an object of the following form:

```
{
    "package"?: <package name>,
    "resource": <resource name>
}
```

The referenced resource should be of `"type": "category-table"`. Category table
resources have the following extra fields:

```
{
    "name": <field name>,
    "type": "category-table",
    "valueField"?: <field name> (default = "value"),
    "labelField"?: <field name> (default = "label"),
    ...Table Resource fields
}
```

## Example

An example of table-based categoricals can be found in the
[datapackage.json](./datapackage.json) in the current directory.

The datapackage has the following model / relationship between tables:

```mermaid
erDiagram
    diet-log ||--|{ items : "item_code is defined item_id in items"
    diet-log {
        date date
        integer participant_id
        integer item_code
        number quantity
    }
    items ||--|{ meal_types : "meal_type is defined by meal_type in meal-types"
    items {
        integer item_id
        string name
        number weight_g
        number calories
        string meal_type
    }
    meal-types {
        string meal_type
        string description
        number popularity
    }
```

In this example, the field `item_code` in the `diet-log` resource is a
hierarchical categorical type defined by the `items` and `meal-types` resources.

## Relationship to `foreignKeys`

The table-based categorical approach is very similar to the `foreignKeys`
properties that can already be defined on table schemas.
[datapackage.foreignkeys.json](./datapackage.foreignkeys.json) illustrates how
the previous example could be defined using `foreignKeys` instead of table-based
`categories`.

This example illustrates some key differences between `foreignKeys` and
`categories`:

- `foreignKeys` are defined at the _schema_ level, whereas the `categories`
  property is defined at the _field_ level.
- Because `foreignKeys` are defined at the schema level, it makes it possible to
  use _multiple_ fields in a key, whereas `categories` is defined at the field
  level and can only be used to connect a single field to a foreign table
- The `categories` property allows you to define a `labelField` which can be
  used to define a label for the categorical level, which is not possible with
  `foreignKeys` (because it would not make sense when multiple fields are
  involved)
- When using the `categories` property, you can additionally define
  `categoriesOrdered` to indicate the value is ordinal. This is not possible
  with `foreignKeys` (again because it would not make sense when multiple fields
  are involved)

## Some theoretical questions

1. When we define a table-based `categories` property, should the referenced
   value and label fields automatically get a `unique` constraint? (yes)

1. When we define a table-based `categories` property, should the referenced
   field automatically get a `not null` constraint? (i.e. `missingValues: []`)
   (yes)

1. When a `valueField` is not specified, should it default to the name of the
   current column? (No, it should default to "value"; and "labelField" should
   default to "label")

1. Must the type of the current field match the foreign `valueField`? (yes)

1. Right now, `categoriesOrdered` is separate from the `categories` property.
   This means that when re-using a table-based categorical definition, it must
   be redefined in each field. Should `categoriesOrdered` be part of the Table
   referenced by `categories` somehow? Or should the concept of ordering remain
   decoupled? (This is addressed via the "category-table" pattern)

1. Can the foreign `valueField` have the `categories` or `categoriesOrdered`
   property defined? Might this be useful for the above issue? (Value field MUST
   be bare integer or string, so no `categories` or `categoriesOrdered`)

1. What happens when definitions of `foreignKeys` and `categories` conflict?
   (Error: choose only one representation)

1. Are all single-field `foreignKeys` functionally equivalent to `categories`?
   I.e. should single-field `foreignKeys` be considered categorical values?
   (N/A)

1. Does the idea of value labels generalize to multi-field `foreignKeys`? (N/A)
