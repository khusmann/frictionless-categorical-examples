# Inline Categoricals

Inline categoricals are implemented in v2 of the Frictionless Datapackage
specification via two properties:
[categories](https://datapackage.org/standard/table-schema/#categories), and
[categoriesOrdered](https://datapackage.org/standard/table-schema/#categoriesOrdered).
These properties extend `integer` and `string` field types with categorical
information (available levels, and ordering, respectively).

Here's a basic example of a frictionless field with an inline categorical
definition:

```json
{
  "name": "numeric_cateogircal",
  "type": "number",
  "description": "A categorical variable with three numeric options",
  "categories": [1, 2, 3],
  "categoriesOrdered": true
}
```

## Key Features

The [datapackage.json](./datapackage.json) file in the current directory
demonstrates all of the features supported by frictionless inline categoricals,
including:

Basic categoricals:

- ordered / unordered levels
- numeric levels
- string levels

Categorical-level metadata (see [section below](#categorical-level-metadata)):

- coded numeric levels (numbers with labels)
- coded string levels (strings with labels)

Categorical missing values (see [section below](#coded-missing-values)):

- coded missing values (missing values with labels)

### Categorical-level metadata

In addition to defining the levels of a categorical, the `categories` property
can define metadata attached to particular _values_ of the categorical.
Categorical-level metadata is of particular use in _coded_ categorical data,
where the values stored in the data are (often arbitrary) codes with particular
meanings attached.

For example, a study might code a gender binary in a column as 1: MALE and 2:
FEMALE. Such a categorical can be represented in a frictionless categorical
definition like this:

```json
{
  "name": "gender",
  "type": "number",
  "categories": [
    { "value": 1, "label": "MALE" },
    { "value": 2, "label": "FEMALE" }
  ]
}
```

It's possible for levels to have additional properties, but so far only the
`label` property is officially supported in frictionless. Extensive categorical
level metadata gets unwieldly quickly, and so is probably not a good use case
for inline categoricals. This is where "table-based" categoricals come in;
discussed in the [next steps](#next-steps), below.

### Coded missing values

Missing values are often represented with _codes_, similar to coded categorical
values (categorical values with labels). For example, a study might use the code
`-99` to represent a `REFUSED` response, and `-98` to represent a `SKIPPED`
response.

To support data with coded missing values, frictionless datapackage v2 provides
an extension of the `missingValues` property with a similar structure as the
`categories` property. Here's an example of a field with coded missing values:

```json
{
  "name": "field_with_missing_values",
  "type": "number",
  "missingValues": [
    { "value": "-99", "label": "REFUSED" },
    { "value": "-98", "label": "SKIPPED" }
  ]
}
```

Note that this extension of `missingValues` can be used with ALL field types,
not just categorical field types.

## Example implementation ([interlacer](https://kylehusmann.com/interlacer) + [frictionless-r](https://docs.ropensci.org/frictionless/))

I have an
[outstanding PR in frictionless-r](https://github.com/frictionlessdata/frictionless-r/pull/213)
that implements the frictionless v2 spec for inline categoricals (and coded
missing values).

You can try it out by installing the "interlacer" branch of frictionless-r from
my github fork:

```r
devtools::install_github("khusmann/frictionless-r", "interlacer")
```

To load the datapackage in this directory:

```r
library(frictionless)
library(interlacer)
pkg <- read_package("datapackage.json")
pkg
#> A Data Package with 2 resources:
#> • example-data
#> • example-data-with-na
#> Use `unclass()` to print the Data Package as a list.
```

To load the first resource as a dataframe:

```r
df <- read_interlaced_resource(pkg, "example-data")
df
#> # A tibble: 6 × 4
#>   num_cat str_cat int_cat_l str_cat_l
#>   <ord>   <fct>   <cfct>    <cfct>
#> 1 1       apple   apple     apple
#> 2 2       orange  banana    apple
#> 3 1       orange  apple     banana
#> 4 3       banana  apple     orange
#> 5 1       orange  orange    banana
#> 6 2       apple   orange    orange
```

As you can see:

- numeric / string categoricals are read as `factor` / `ordinal` types
  (depending if they are unordered / ordered)
- coded categoricals are read as `cfactor` types

`cfactor` types are provided by the interlacer package, and extend base R
`factor` types to preserve their codes. In other words, if we wanted to convert
the `str_cat_l` column back to its coded representation, we can do so:

```r
df %>%
  mutate(
    int_cat_l = as.codes(int_cat_l),
    str_cat_l = as.codes(str_cat_l)
  )
#> # A tibble: 6 × 4
#>   num_cat str_cat int_cat_l str_cat_l
#>   <ord>   <fct>       <int> <chr>
#> 1 1       apple           1 a
#> 2 2       orange          3 a
#> 3 1       orange          1 b
#> 4 3       banana          1 o
#> 5 1       orange          2 b
#> 6 2       apple           2 o
```

Finally, (coded and uncoded) missing values are handled via interlacer's
interlaced column types:

```r
df2 <- read_interlaced_resource(pkg, "example-data-with-na")
df2
#> # A tibble: 6 × 4
#>   int_cat   str_cat   int_cat_l   str_cat_l
#>   <ord,int> <fct,fct> <cfct,cfct> <cfct,fct>
#> 1 1         apple     apple       apple
#> 2 2         orange    banana      apple
#> 3 <-99>     <REFUSED> apple       <SKIPPED>
#> 4 3         banana    <SKIPPED>   orange
#> 5 <-98>     <SKIPPED> orange      banana
#> 6 2         apple     <REFUSED>   <REFUSED>
```

Interlaced columns provide two key advantages over traditional ways of working
with tagged missingness:

1. they make it possible to have separate types for regular values and missing
   values. For example, a column can have numeric values, but categorical
   (factor) missing reasons.
2. when you do a operations on interlaced columns, they automatically apply to
   the values in the column, and _automatically mask out_ the missing values.

For more info on interlaced columns, check out the
["Introduction" vignette](https://kylehusmann.com/interlacer/articles/interlacer.html)
in the interlacer package.

## Next steps

The "inline categorical" definition approach is great for representing simple
variables, as well as interoperability with native categorical types across many
statistical file formats (e.g. SPSS, Stata, SAS). However, it has some key
weaknesses:

- Categorical levels are difficult to reuse across multiple variable definitions
- Categoricals with many levels get unwieldly quickly
- Extra categorical level metadata (level descriptions, hierarchical categorical
  structures, etc.) can get unwieldly quickly

To address these weaknesses, we are considering _table-based_ categorical
definitions.
