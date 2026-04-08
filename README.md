# cYamlObject

A DataFlex class for parsing and serializing YAML, built on top of the built-in `cJsonObject` tree model. It exposes the full `cJsonObject` API for navigating the parsed structure, and supports automatic conversion to and from DataFlex structs via `DataTypeToJson` / `JsonToDataType`.

---

## How it works

`cYamlObject` extends `cJsonObject`. Parsing a YAML document populates the internal JSON tree that `cJsonObject` maintains, so every method you already know from `cJsonObject` — member access, type checks, array iteration — works identically on YAML-sourced data.

```
cYamlObject
  └── cJsonObject          (tree model, struct marshalling)
```

The parser is block-style and indentation-based. It reads YAML lines, infers scalar types, and builds the same node hierarchy that the JSON parser produces. Serialization walks that tree in reverse and emits valid YAML.

---

## API

### Parsing

```dataflex
Function ParseYaml(String sYaml) Returns Boolean
```
Parses a YAML string. Splits on line feeds (handles both LF and CRLF). Populates `Self` with the resulting tree. Returns `True` on success.

```dataflex
Function ParseYamlLines(String[] asLines) Returns Boolean
```
Parses YAML from a pre-split array of lines. Useful when the source is already line-buffered (e.g. read from a file). Returns `True` on success.

### Serialization

```dataflex
Function StringifyYaml() Returns String
```
Serializes the current tree back to a YAML string. Indentation, quoting, and type representation are handled automatically.

### Tree navigation (inherited from `cJsonObject`)

| Method | Description |
|---|---|
| `JsonType()` | Root node type (`JSON_TYPE_OBJECT`, `JSON_TYPE_ARRAY`, `JSON_TYPE_STRING`, …) |
| `MemberCount()` | Number of keys (object) or items (array) |
| `MemberByIndex(iIndex)` | Child node by position |
| `MemberNameByIndex(iIndex)` | Key name at position |
| `Member(sMember)` | Child node by key name |
| `MemberValue(sMember)` | Scalar value of a key |
| `MemberJsonType(sMember)` | Type of a member |
| `HasMember(sMember)` | Check whether a key exists |
| `SetMember(sMember, hoObj)` | Replace / add a child node |
| `SetMemberValue(sMember, eType, sVal)` | Set a scalar value |
| `AddMember(hoObj)` | Append a child node (arrays) |
| `AddMemberValue(eType, sVal)` | Append a scalar value (arrays) |
| `RemoveMember(sMember)` | Remove a key |
| `JsonValue()` | Scalar value of the root node |

---

## Struct conversion

Because `cYamlObject` is a `cJsonObject`, `DataTypeToJson` and `JsonToDataType` work directly on YAML-sourced data. This lets you parse a YAML document straight into a typed DataFlex struct with a single call.

### Example

```dataflex
// Define your structs to match the YAML shape
Struct tTutorialItem
    String id
    String name
    String type
    Integer born
End_Struct

Struct tCompanyInfo
    String company
    String[] domain
    tTutorialItem[] tutorial
    String author
    Boolean published
End_Struct
```

```yaml
# example.yaml
company: spacelift
domain:
  - devops
  - devsecops
tutorial:
  - id: yaml
    name: "YAML Ain't Markup Language"
    type: awesome
    born: 2001
author: omkarbirade
published: true
```

```dataflex
tCompanyInfo oData
Handle hoYaml
Boolean bOk

Get Create (RefClass(cYamlObject)) to hoYaml
Get ParseYaml of hoYaml sYamlString to bOk
Get JsonToDataType of hoYaml to oData
// oData.company = "spacelift"
// oData.domain[0] = "devops"
// oData.tutorial[0].born = 2001
Send Destroy to hoYaml
```

To go the other direction, build your struct in code and convert it to YAML:

```dataflex
Get Create (RefClass(cYamlObject)) to hoYaml
Send DataTypeToJson to hoYaml oData
Get StringifyYaml of hoYaml to sOutput
Send Destroy to hoYaml
```

Struct member names are matched to YAML keys by name. Nested structs map to YAML objects; struct arrays map to YAML sequences of mappings.

---

## Supported YAML features

| Feature | Supported |
|---|---|
| Block mappings (`key: value`) | Yes |
| Block sequences (`- item`) | Yes |
| Sequences of mappings | Yes |
| Nested objects / arrays (indentation-based) | Yes |
| Scalar type inference (string, integer, float, boolean, null) | Yes |
| Quoted strings (`"…"` / `'…'`) — forces string type | Yes |
| Full-line comments (`# …`) | Yes |
| Inline comments (`value # comment`) | Yes |
| Null (`~`, `null`, `Null`, `NULL`) | Yes |
| Booleans (`true`/`false`, `True`/`False`, `TRUE`/`FALSE`) | Yes |
| Locale-safe number parsing (decimal separator normalisation) | Yes |
| Anchors and aliases (`&anchor` / `*ref`) | No |
| Multi-line scalars (literal `\|` / folded `>`) | No |
| Flow collections (`{…}` / `[…]`) | No |
| Multiple documents (`---` / `...`) | No |
| Tags (`!!type`) | No |

The parser targets the common subset of YAML used in configuration files. Advanced YAML features are not required for that use case and are not implemented.

---

## Serialization behaviour

`StringifyYaml` produces clean, human-readable YAML:

- Strings are only quoted when necessary (e.g. the value would otherwise be interpreted as a number, boolean, or null, or contains a colon).
- Booleans are normalized to lowercase (`true` / `false`).
- Null values are written as `~`.
- Indentation uses two spaces per level.
- Numbers round-trip exactly (decimal separator is normalized to `.` during serialization regardless of the system locale).
