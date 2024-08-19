# Table-based categoricals

Table-based categoricals are a way to define categorical variables in a separate
table (rather than inline the field definition). This approach is useful for:

- Re-using categorical definitions across multiple fields
- Categorical defintiions that require extra metadata (e.g. hierarchical
  categorical structures)
- Categoricals with many levels

## Example

An example of table-based categoricals can be found in the
[datapackage.json](./datapackage.json) in the current directory.

The datapackage has the following model / relationship between tables:

```mermaid
erDiagram
    diet_log ||--|{ items : item_code is defined item_id in items
    diet_log {
        date date
        integer participant_id
        integer item_code
        number quantity
    }
    items ||--|{ meal_types : meal_type is defined by meal_type in meal_types
    items {
        integer item_id
        string name
        number weight_g
        number calories
        string meal_type
    }
    meal_types {
        string meal_type
        string description
        number popularity
    }
```
