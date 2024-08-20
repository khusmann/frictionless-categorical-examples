# Questions about missing values

Field representations that include labelled missing values are hard to get
right. In this document, we explore some theoretical questions about how to
represent labelled missingness in frictionless.

## Theoretical background & motivation

Historically, missing value codes are intermixed with regular values, but this
makes them difficult to reason about. Working with this representation is
error-prone, because you constantly have to be masking out missing values in
your logic and casting types (see this
[retracted study](https://retractionwatch.com/2015/07/21/to-our-horror-widely-reported-study-suggesting-divorce-is-more-likely-when-wives-fall-ill-gets-axed/),
for example).

A key observation about labelled missing values is that they are fundamentally a
_different type_ from regular values. For example, a numeric column might have
missing values represented as `-99`, but the missing values are not numbers --
they're actually a coded categorical type.

So even though categorical variables often have additional levels that represent
missing values, we need to be careful to model our labelled missingness in such
a way that it can be applied to other types of variables as well. This is why in
frictionless datapackage v2, we separate our definition of missing values from
our definition of categorical levels:

```json
{
  "name": "categorical_field_with_missing_values",
  "type": "number",
  "categories": [1, 2, 3, 4, 5],
  "missingValues": [
    { "value": "-99", "label": "REFUSED" },
    { "value": "-98", "label": "SKIPPED" }
  ]
}
```

This way, we can use the same categorical definition on fields of
non-categorical types:

```json
{
  "name": "date_field_with_missing_values",
  "type": "date",
  "missingValues": [
    { "value": "-99", "label": "REFUSED" },
    { "value": "-98", "label": "SKIPPED" }
  ]
}
```

In other words, the field type of a field with labelled missingness is a
_compound type_: a type composed of a value type and missingness type. The R
package [interlacer](https://kylehusmann.com/interlacer/) capitalizes on this
idea by allowing users to explicitly interact with their data in terms of these
compound types.

That said, even implementations that do not support compound types can still
benefit from this representation. For example, implementations might choose to
import all the missing values as `NULL` values, import the missing value labels
in a separate column, or return a flattened textual representation of the
missing values intermixed with the regular values.

## Outstanding question #1: Should missing values always be specified as `string`?

For historical reasons, missing values are always represented via `string`
types:

```json
{
  "name": "field_with_missing_values",
  "type": "number",
  "missingValues": ["-99", "-98"]
}
```

The reasoning in the
[Table Schema spec](https://datapackage.org/standard/table-schema/#missingValues)
is as follows:

> Why strings: `missingValues` are strings rather than being the data type of
> the particular field. This allows for comparison prior to casting and for
> fields to have missing values which are not of their type, for example a
> `number` field to have missing values indicated by `-`.

More generally, missing values are categorical types. So missing values can be
represented in four ways:

1. As `string` categories
2. As `number` categories
3. As coded `string` categories (`strings` with labels)
4. As coded `numbe` categories (`numbers` with labels)

All of these cases are covered by the current form of the `categories` property,
and could also potentially be adopted by the `missingValues` property. For
example:

```json
{
  "name": "field_with_missing_values",
  "type": "date",
  "missingValues": [-99, -98]
}
```

could be interpreted as a field with date value types and numeric missing
reasons, and

```json
{
  "name": "field_with_missing_values",
  "type": "date",
  "missingValues": [
    { "value": -99, "label": "REFUSED" },
    { "value": -98, "label": "SKIPPED" }
  ]
}
```

could be interprested as a field with date value types and coded numeric missing
reasons.

The issue with this approach is that the underlying value type of the missing
values becomes somewhat implicit, because it must be inferred from the JSON
value type of `missingValues`. This could be remedied with a more explicit field
definition for missing values:

```json
{
  "name": "field_with_missing_values",
  "type": "date",
  "missingValues": {
    "type": "numeric",
    "categories": [
      { "value": -99, "label": "REFUSED" },
      { "value": -98, "label": "SKIPPED" }
    ]
  }
}
```

...but this seems rather verbose and overkill. The implicit approach seems more
"frictionless". Although it's not perfect, it's not ambiguous, and seems to be a
good compromise between verbosity and clarity.

## Outstanding issue #2: What about table-based missing values?

If we follow the table-based categorical approach for missing values, we get
something like this:

```json
{
  "name": "field_with_missing_values",
  "type": "date",
  "missingValues": {
    "resource": "missing-value-codes",
    "valueField": "code",
    "labelField": "label"
  }
}
```

Where the resource `missing-value-codes` might look like this:

```csv
code,label,description
-99,REFUSED,The participant refused to answer the question
-98,SKIPPED,The participant skipped the question
```

The first thing that's really nice about this approach is that it allows us to
explicitly type the missing values, because they are defined in a separate
resource. It's also easy to add a bunch of extra metadata about them.

Another nice thing about this approach is that we can have separate tables (with
different metadata) when specifying levels of a categorical and the levels of
missing values. For example:

```json
{
  "name": "field_with_missing_values",
  "type": "string",
  "categories": {
    "resource": "medical-codes",
    "valueField": "code",
    "labelField": "label"
  },
  "missingValues": {
    "resource": "missing-value-codes",
    "valueField": "code",
    "labelField": "label"
  }
}
```

Where the same `missing-value-codes` resource above is used, and the resource
`medical-codes` might look like this:

```csv
code,label,description,medicine_category
M001,Hypertension,High blood pressure,Cardiovascular
M002,Hyperlipidemia,High cholesterol,Cardiovascular
M003,Diabetes Mellitus Type 2,Chronic condition affecting blood sugar levels,Endocrine
M004,Asthma,Respiratory condition causing difficulty in breathing,Respiratory
M005,Osteoarthritis,Joint inflammation and pain,Musculoskeletal
```

The only potential downside of this approach is that it has a hard time
representing tables of missing values intermixed with regular values. For
example, say we had the following table of codes:

```csv
code,label,description
1,BELOW_AVERAGE,The participant scored below average
2,AVERAGE,The participant scored average
3,ABOVE_AVERAGE,The participant scored above average
-99,REFUSED,The participant refused to answer the question
-98,SKIPPED,The participant skipped the question
```

We cannot directly use this table with the above approach. Instead we need to
separate the categorical value codes from the categorical missing codes.

I would argue, however, this is a good thing, because the only reason we're able
to create this shared table is because we're using common metadata for values
and missing reasons. Look at what happens to our medicine table, for example,
when we try to add missing values to it:

```csv
code,label,description,medicine_category
M001,Hypertension,High blood pressure,Cardiovascular
M002,Hyperlipidemia,High cholesterol,Cardiovascular
M003,Diabetes Mellitus Type 2,Chronic condition affecting blood sugar levels,Endocrine
M004,Asthma,Respiratory condition causing difficulty in breathing,Respiratory
M005,Osteoarthritis,Joint inflammation and pain,Musculoskeletal
-99,REFUSED,The participant refused to answer the question,???
-98,SKIPPED,The participant skipped the question,???
```

As you can see, not all of the columns make sense for the missing values
(namely, the "category" column). In general, you might expect that metadata for
missing value levels and regular value levels to be different, so it makes sense
for them to reference separate tables.

ALSO! Having separate tables lets you re-use the missing value table definitions
with many different categorical variables, rather than needing to re-specify the
missing levels with all of the values.

That said, I wonder if this will hinder adoption in cases where missing value
codes are all in the same table?

What if we had `includeValues` and `excludeValues` properties?

e.g.:

```json
{
  "name": "field_name",
  "type": "number",
  "categories": {
    "resource": "likert-scale",
    "valueField": "code",
    "labelField": "label",
    "excludeValues": [-99, -98]
  },
  "missingValues": {
    "resource": "likert-scale",
    "valueField": "code",
    "labelField": "label",
    "includeValues": [-99, -98]
  }
}
```

Where `likert-scale` is:

```csv
code,label,description
1,BELOW_AVERAGE,The participant scored below average
2,AVERAGE,The participant scored average
3,ABOVE_AVERAGE,The participant scored above average
-99,REFUSED,The participant refused to answer the question
-98,SKIPPED,The participant skipped the question
```
