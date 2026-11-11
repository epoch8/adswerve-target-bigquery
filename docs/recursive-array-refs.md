# Recursive Singer array refs

`tap-postgres` emits self-referential `$ref`s for Postgres array columns. BigQuery has no recursive column type, so `simplify()` used to raise and the load failed. `_helper_simplify` now cuts that cycle and keeps one array of the scalar.

The failing stream was `public-stories`. The schema is saved in `sandbox/schema.json`. The loader error is in `sandbox/error.txt`:

```text
JSONSchemaError: `$ref` path "{'type': ['null', 'string', 'array'], 'items': {'$ref': '#/definitions/sdc_recursive_string_array'}}" is recursive
```

## Where the schema comes from

This is valid JSON Schema, and the tap is doing what the Meltano SDK tells it to do. The SDK's SQL-to-JSON-Schema converter writes a definition like this for a Postgres array (`text[]`, `varchar[]`, and the same pattern for other scalar item types):

```json
"sdc_recursive_string_array": {
    "items": {
        "$ref": "#/definitions/sdc_recursive_string_array"
    },
    "type": ["null", "string", "array"]
}
```

`items.$ref` points at the definition itself. Draft-04 JSON Schema can only say "a string, or an array of the same thing, nested to any depth" with a `$ref` cycle. Postgres arrays can be multidimensional, so the SDK uses this generic encoding for every array column. `highlight_ids` and `polls` are almost certainly one-dimensional string arrays. The recursive form does not mean the tap saw nested arrays in the data.

A field that uses the definition looks like this:

```json
"highlight_ids": {
    "items": {
        "$ref": "#/definitions/sdc_recursive_string_array"
    },
    "type": ["null", "array"]
}
```

`polls` has the same shape. The same file also defines recursive boolean, integer, number, object, and timestamp arrays. This stream only references the string one.

## What `_helper_simplify` does

`simplify()` calls `_helper_simplify(schema, schema)`. Each call handles one schema node and returns a `Cachable` dict. `Cachable` is a dict this function has already finished; seeing one means return it and do not walk it again. The simplified dict is also hashable, which the `anyOf` merger needs when it dedupes branches.

Every node takes one of four branches, in this order:

| Branch | When | What it does |
|---|---|---|
| `$ref` | The node has `$ref` | Look the definition up in the root schema and simplify that. A self-referential array is reduced to its scalar first. |
| `allOf` | The node has `allOf` and no `$ref` | The value must match every listed schema. Merge them into one schema. |
| `anyOf` | The node has `anyOf` and no `$ref` or `allOf` | The value may match any listed schema. Fold the list into as few schemas as possible. |
| implicit `anyOf` | Anything else | A `type` array such as `["null", "string"]` is the same idea without the keyword. Split it into an explicit `anyOf` and fold that. |

`$ref` wins over every other keyword in the same node. That follows JSON Schema: a `$ref` replaces the schema it sits on.

### `allOf`

`allOf` means the value must satisfy every listed schema at once. This stories schema has none. The merger exists for objects whose properties are split across several fragments: each fragment is simplified, then their `properties` are copied into one object. Later fragments overwrite earlier ones on the same key. For arrays, the same merge is applied to `items`.

### Implicit `anyOf`

`"type": ["null", "string", "array"]` means the value may be any of those types. `_simplify__implicit_anyof` turns that into one single-type schema per non-null type:

- `string` with `format: date-time` becomes one string schema that keeps the format. Same for `date`, `bq-geography`, `bq-decimal`, and `bq-bigdecimal`.
- `number` becomes a string schema with `format: number`, and `multipleOf` is kept so precision and scale can be derived later.
- `object` becomes one object schema. Each property is simplified on its own.
- `array` becomes one array schema. Its `items` are simplified. A missing `items` defaults to `{ "type": "string" }`.
- Any remaining literal (`string`, `integer`, `boolean`, …) becomes `{ "type": ["<that type>"] }`.

`null` is not its own branch. If the original type list contained `null`, `make_nullable` appends it to every branch, so `"string"` becomes `["string", "null"]`.

The list is then passed straight back into `_helper_simplify` as an explicit `anyOf`.

### Explicit `anyOf`

`_simplify__anyof` simplifies each branch, then folds them:

- Literal schemas of the same shape are deduped.
- A nested `anyOf` is flattened into the parent list.
- All objects become one object. Each shared property is itself an `anyOf` of the versions from each object.
- All arrays become one array. Their `items` become one `anyOf`.
- If any branch was nullable, every surviving branch is made nullable.
- A list of one schema is unwrapped, so the `anyOf` key disappears. Two or more different kinds (an object and an array, a string and an integer) stay as `anyOf`.

BigQuery still needs one column type when an `anyOf` survives. That choice is later, in `prioritize_one_data_type_from_multiple_ones_in_any_of`. `string` outranks `array` there, which is why the recursive definition must not keep both a string branch and an array branch.

## How `highlight_ids` is simplified

1. The field type is `["null", "array"]`, so the implicit step builds one branch: a nullable array. The `items` of that array are the `$ref`.
2. The `$ref` resolves to `sdc_recursive_string_array`. Its `items.$ref` is its own name, so the `$ref` branch copies the definition, drops `array` from `type`, and deletes `items`. What remains is `{ "type": ["null", "string"] }`. `format` is kept when the timestamp variant has one.
3. That scalar goes through the implicit step. The only non-null type is `string`, `null` is appended, and the single-element `anyOf` is unwrapped to `{ "type": ["string", "null"] }`.
4. The array branch is the only branch on `highlight_ids`, so its `anyOf` is unwrapped too.

The field that comes out is:

```json
{"type": ["array", "null"], "items": {"type": ["string", "null"]}}
```

`build_field` reads `items.type[0]` (`string`) and sees `items`, so the BigQuery column is `REPEATED STRING`. `polls` takes the same path.

The parent property supplies the single array layer. The definition supplies the scalar. A value nested deeper than one array does not fit that column.

A property that is the recursive definition itself, with no parent `array`, becomes the scalar only. `sdc_recursive_timestamp_array` used that way simplifies to a nullable `date-time` string, which loads as `TIMESTAMP`.

## What this does not cover

`media` and `moderation_response` do not use these refs. Their type is `["null", "object", "array"]` with no `properties` and no `items`. The implicit step makes two branches: an empty object, and an array whose missing `items` default to `string`. An object and an array do not merge, so the result stays:

```json
{
    "anyOf": [
        {"type": ["array", "null"], "items": {"type": ["string"]}},
        {"type": ["object", "null"], "properties": {}}
    ]
}
```

A `$ref` cycle that runs through `properties` instead of `items` is also unchanged. The cut is only the Singer pattern where `items.$ref` points at the definition currently being resolved.
