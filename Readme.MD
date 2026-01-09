# GQL

A Go query library for REST APIs that enables rich, type-safe filtering capabilities.

## Installation

```bash
go get github.com/robmanganelly/gql
```

## Overview

GQL sits on top of standard REST APIs to allow clients to pass complex filters in a standardized way. It parses query parameters into typed filter objects and can generate SQL WHERE clauses.

## Quick Start

```go
package main

import (
    "fmt"
    "net/http"

    "github.com/robmanganelly/gql"
)

// Define your filter schema using struct tags
type UserFilters struct {
    Name   string  `gql:"name"`
    Age    float64 `gql:"age"`
    Status string  `gql:"status"`
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    // Parse query parameters
    var query gql.Query[UserFilters]
    if err := gql.ParseQuery(r.URL.Query(), &query); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Generate SQL WHERE clause
    result, err := gql.ToSQLWhere(query.Filters)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Use result.Clause and result.Args in your query
    fmt.Printf("WHERE %s\n", result.Clause)  // e.g., "name = $1 AND age > $2"
    fmt.Printf("Args: %v\n", result.Args)    // e.g., ["john", 25]
}
```

## Query Format

All filter parameters are prefixed with `f.` and use explicit suffixes to indicate value types:

| Suffix | Purpose | Used With |
|--------|---------|-----------|
| `f.<field>.o` | Operator (required) | All filters |
| `f.<field>.v` | Single value | Equality, Numeric |
| `f.<field>.r.from`, `f.<field>.r.to` | Range bounds | Range |
| `f.<field>.vs` | Multiple values | Set |
| `f.<field>.p` | Pattern | Text |
| `f.<field>.g` | Group key (optional) | All filters |

### Examples

```
# Equality
/users?f.name.o=eq&f.name.v=john

# Numeric comparison
/users?f.age.o=gt&f.age.v=25

# Range (between)
/users?f.date.o=bt&f.date.r.from=100&f.date.r.to=200

# Set (in list)
/users?f.status.o=in&f.status.vs=active&f.status.vs=pending

# Text pattern (includes)
/users?f.title.o=inc&f.title.p=hello

# Multiple filters with grouping
/users?f.name.o=eq&f.name.v=john&f.name.g=1&f.age.o=gt&f.age.v=25&f.age.g=1
```

## Supported Operators

### Equality
| Operator | Code | Description |
|----------|------|-------------|
| `eq` | `gql.OpEqual` | Equals |
| `ne` | `gql.OpNotEqual` | Not equals |

### Numeric
| Operator | Code | Description |
|----------|------|-------------|
| `gt` | `gql.OpGreaterThan` | Greater than |
| `gte` | `gql.OpGreaterThanOrEqual` | Greater than or equal |
| `lt` | `gql.OpLessThan` | Less than |
| `lte` | `gql.OpLessThanOrEqual` | Less than or equal |

### Range
| Operator | Code | Description |
|----------|------|-------------|
| `bt` | `gql.OpBetween` | Between (inclusive) |
| `nbt` | `gql.OpNotBetween` | Not between |

### Set
| Operator | Code | Description |
|----------|------|-------------|
| `in` | `gql.OpIn` | Value in set |
| `nin` | `gql.OpNotIn` | Value not in set |
| `sset` | `gql.OpSuperSetOf` | Is superset of |
| `nsset` | `gql.OpNotSuperSet` | Is not superset of |

### Text
| Operator | Code | Description |
|----------|------|-------------|
| `inc` | `gql.OpIncludes` | Contains substring (case-sensitive) |
| `ninc` | `gql.OpNotIncludes` | Does not contain (case-sensitive) |
| `iinc` | `gql.OpInsensitiveIncludes` | Contains substring (case-insensitive) |
| `ininc` | `gql.OpInsensitiveNotIncludes` | Does not contain (case-insensitive) |

## Grouping

Filters can be grouped using the `g` suffix. Filters within the same group are combined with `AND`, while different groups are combined with `OR`.

```
# (name = 'john' AND age > 25) OR (status = 'admin')
/users?f.name.o=eq&f.name.v=john&f.name.g=1&f.age.o=gt&f.age.v=25&f.age.g=1&f.status.o=eq&f.status.v=admin&f.status.g=2
```

## Filter Count (fc)

Because some proxies can truncate long URLs, you can provide an `fc` parameter with the expected number of filters. The parser will return an error if the count doesn't match.

```
/users?fc=2&f.name.o=eq&f.name.v=john&f.age.o=gt&f.age.v=25
```

## API Reference

### ParseQuery

```go
func ParseQuery[T any](params url.Values, target *Query[T]) error
```

Parses URL query parameters into a typed Query. The type parameter `T` defines allowed filter fields via `gql` struct tags.

### ToSQLWhere

```go
func ToSQLWhere(filters []Filter) (*SQLWhereResult, error)
```

Generates a SQL WHERE clause using field names as column names.

### ToSQLWhereAlias

```go
func ToSQLWhereAlias(filters []Filter, keymap map[string]string) (*SQLWhereResult, error)
```

Generates a SQL WHERE clause with field-to-column name mapping.

```go
keymap := map[string]string{
    "name": "users.full_name",
    "age":  "users.age",
}
result, err := gql.ToSQLWhereAlias(query.Filters, keymap)
// result.Clause: "users.full_name = $1 AND users.age > $2"
```

### SQLWhereResult

```go
type SQLWhereResult struct {
    Clause string // The WHERE clause (without "WHERE" prefix)
    Args   []any  // Positional arguments ($1, $2, ...)
}
```

## Filter Types

The parser returns typed filter objects that can be accessed via type switch:

```go
for _, f := range query.Filters {
    switch filter := f.(type) {
    case gql.EqualityFilter:
        fmt.Printf("Field: %s, Value: %v\n", filter.Field, filter.Value)
    case gql.NumericFilter:
        fmt.Printf("Field: %s, Value: %f\n", filter.Field, filter.Value)
    case gql.RangeFilter:
        fmt.Printf("Field: %s, From: %f, To: %f\n", filter.Field, filter.Value.From, filter.Value.To)
    case gql.SetFilter:
        fmt.Printf("Field: %s, Values: %v\n", filter.Field, filter.Values)
    case gql.TextFilter:
        fmt.Printf("Field: %s, Pattern: %s\n", filter.Field, filter.Pattern)
    }
}
```

## TypeScript Client

A TypeScript client library for building query strings is planned. See [GQL Client Library](#) (TBD)

## License

MIT
