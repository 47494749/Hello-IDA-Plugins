# Hello-IDA-Plugins by Luigi Origa
# ColQ - IDA Database Query Plugin

ColQ is an IDA Pro plugin that lets you search the IDA database using a simple query language. Instead of writing scripts, type queries like `name = '*CAN*' AND size > 100` and get results instantly in a sortable grid.

**Shortcut:** `Ctrl-Shift-Q`

## Syntax

```
field = value [AND field = value ...]
```

- **Operators:** `=` `!=` `<` `>` `<=` `>=`
- **Wildcards:** `*` (any chars), `?` (one char) — case-insensitive
- **Values:** decimal, `0xHex`, strings, `'quoted strings'`

## Query Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Function/symbol name (wildcards) | `name = '*CAN*'` |
| `address` | Address range | `address >= 0x80010000` |
| `size` | Function size in bytes | `size > 100` |
| `mem_access` | Reads or writes memory (address/label/wildcard) | `mem_access = 0xF0000400` |
| `mem_read` | Reads memory | `mem_read = MyVariable` |
| `mem_write` | Writes memory | `mem_write = '*counter*'` |
| `call` | Direct function call (wildcard) | `call = '*alloc*'` |
| `deep_call` | Transitive call (16 levels deep) | `deep_call = 'memcpy'` |
| `deep_mem_access` | Memory access through call chain | `deep_mem_access = 0xF000` |
| `deep_mem_read` | Memory read through call chain | `deep_mem_read = MyVar` |
| `deep_mem_write` | Memory write through call chain | `deep_mem_write = '*flag*'` |
| `xref_to` | References an address | `xref_to = 0xF0001234` |
| `has_string` | Contains a string reference | `has_string = '*error*'` |
| `value` | Immediate operand value (combinable) | `value >= 0x1000 AND value <= 0x2000` |

## Tables

The default table is **functions**. Switch to other tables with the table prefix:

```
names: name = '*error*'
strings: name = '*version*'
imports
exports
segments
```

## Examples

```
name = '*CAN*'
name != 'sub_*'
size > 500
mem_access = 0xF0000400
mem_read = MyVariable
name = '*Send*' AND mem_write = 0xF0200000
call = 'memcpy'
name = '*CAN*' AND deep_call = 'memcpy'
value = 0x1234
has_string = '*error*'
names: name = '*error*'
strings: name = '*version*'
```

## Features

- Sortable results grid (click column headers)
- Double-click a row to jump to the address in IDA
- Autocomplete suggestions while typing
- Built-in help (`? Help` button)
- Modeless window — keeps working while you use IDA

