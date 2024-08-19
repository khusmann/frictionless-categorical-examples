# Inline Categoricals

Inline categoricals are implemented in v2 of the Frictionless Datapackage
specification via two properties:
[categories](https://datapackage.org/standard/table-schema/#categories), and
[categoriesOrdered](https://datapackage.org/standard/table-schema/#categoriesOrdered).
These properties extend `integfer` and `string` field types with categorical
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

The [datapackage.json] file in the current directory demonstrates all of the
features supported by frictionless inline categoricals, including:

- ordered / unordered levels
- numeric levels
- string levels
- coded numeric levels (numbers with labels)
- coded string levels (strings with labels)
- coded missing values (missing values with labels)

### Categorical-level metadata

In addition to defining the levels of a categorical, the `categories` property
can define metadata attached to particular _values_ of the categorical.
Categorical-level metadata is of particular use in _coded_ categorical data,
where the values stored in the data are (often arbitrary) codes with particular
meanings attached.

For example, a study might code a gender binary in a column as 1: MALE and 2:
FEMALE. This can be represented in a frictionless categorical definition like
this:

```json
{
  "name": "gender",
  "type": "number",
  "categories": [
    { "value": 1, "lable": "MALE" },
    { "value": 2, "label": "FEMALE" }
  ]
}
```

### Missing values

Missing values are

## Example implementation ([interlacer](https://kylehusmann.com/interlacer) + [frictionless-r](https://docs.ropensci.org/frictionless/))

I have an
[outstanding PR in frictionless-r](https://github.com/frictionlessdata/frictionless-r/pull/213)
that implements the frictionless v2 spec for inline categoricals.

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
