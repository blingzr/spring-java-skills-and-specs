---
name: mysql-json-handler
description: MySQL JSON field handling in Java Spring Boot with Jackson polymorphism. Covers static-type JSON (single fixed type), dynamic-type JSON (polymorphic deserialization based on a type discriminator), MySQL native JSON query operators (arrow operator and JSON_EXTRACT), and API response formatting (object not string). Supports both MyBatis-Plus and JPA implementations.
---

# MySQL JSON Handler

**TL;DR** — Jackson + MySQL JSON columns. Static pattern: single Java type ↔ single JSON column. Dynamic pattern: `@JsonTypeInfo` discriminator for polymorphic JSON. Native `->>` and `JSON_EXTRACT` queries via MyBatis-Plus or JPA. Always wrap nullable JSON fields in `Optional`.

```java
// Static
@TableField(typeHandler = JacksonTypeHandler.class)
private Address address;

// Dynamic
@TableField(typeHandler = JacksonTypeHandler.class)
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "@type")
private Payload payload;
```

Handle MySQL `JSON` type fields in Java with proper serialization, polymorphic deserialization, and native SQL query support.

## Core Requirements

| Concern | Rule |
|---------|------|
| **Storage** | MySQL native `JSON` column, not `TEXT` or `VARCHAR` |
| **Static JSON** | Fixed Java type → simple Jackson conversion |
| **Dynamic JSON** | Abstract base class + `type` discriminator → correct subclass |
| **API Response** | Return Java objects (Jackson serializes to JSON Object), never `String` |
| **SQL Query** | Use `->>` or `JSON_EXTRACT` for JSON path conditions |
| **Type Column** | Separate `type` column (VARCHAR) alongside JSON column for indexing and business logic |

## Two Patterns

### Pattern 1: Static JSON (Single Type)

The JSON structure is always the same shape. Map directly to a fixed Java class.

```java
// Entity
public class Product {
    private Long id;
    private String name;
    private ProductDetail detail;  // JSON column, always ProductDetail shape
}

// Detail class (fixed structure)
public class ProductDetail {
    private String brand;
    private BigDecimal weight;
    private List<String> tags;
}
```

See `references/static-json.md` for MyBatis-Plus and JPA implementations.

### Pattern 2: Dynamic JSON (Polymorphic)

The JSON structure varies by a discriminator. Store an abstract base, deserialize to the correct subclass based on a `type` field.

```java
// Entity with type discriminator
public class Asset {
    private Long id;
    private String type;       // "CAR" or "HOUSE" — database column
    private AssetConfig config; // JSON column — polymorphic
}

// Abstract base
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type", visible = true)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CarConfig.class, name = "CAR"),
    @JsonSubTypes.Type(value = HouseConfig.class, name = "HOUSE")
})
public abstract class AssetConfig {
    private String type;
}

// Concrete subclass for CAR
public class CarConfig extends AssetConfig {
    private String brand;
    private String licensePlate;
    private Integer mileage;
}

// Concrete subclass for HOUSE
public class HouseConfig extends AssetConfig {
    private String address;
    private BigDecimal area;
    private Integer floorCount;
}
```

See `references/dynamic-json.md` for MyBatis-Plus and JPA implementations.

## Key Differences: MyBatis-Plus vs JPA

| Aspect | MyBatis-Plus | JPA |
|--------|-------------|-----|
| **Static JSON** | `@TableField(typeHandler = JacksonTypeHandler.class)` | `@Type(JsonType.class)` (hibernate-types) or `@JdbcTypeCode(SqlTypes.JSON)` |
| **Dynamic JSON** | Jackson `@JsonTypeInfo` + `@JsonSubTypes` inside JSON | Custom `@Converter` or `@Type` using the entity's `type` column as discriminator |
| **Type source** | JSON must contain `type` property (self-describing) | Can use the database `type` column as discriminator (JSON may omit `type`) |
| **Query** | Raw XML SQL or `@Select` | JPQL with `function('JSON_EXTRACT', ...)` or native query |

## MySQL JSON Query Operators

```sql
-- JSON path extraction (returns unquoted value)
SELECT config ->> '$.brand' FROM asset WHERE type = 'CAR';

-- JSON path existence check
SELECT * FROM asset WHERE config ->> '$.licensePlate' IS NOT NULL;

-- JSON_EXTRACT (returns JSON, may be quoted)
SELECT JSON_EXTRACT(config, '$.area') FROM asset WHERE type = 'HOUSE';

-- Combined with type column (use type for indexing, JSON for field-level filter)
SELECT * FROM asset WHERE type = 'CAR' AND config ->> '$.brand' = 'Tesla';

-- JSON path with array index
SELECT config ->> '$.tags[0]' FROM product;

-- Complex condition
SELECT * FROM asset
WHERE type = 'HOUSE'
  AND CAST(config ->> '$.area' AS DECIMAL(10,2)) > 100.00;
```

## API Response Rules

| Wrong | Right |
|-------|-------|
| `{"config":"{\"brand\":\"Tesla\"}"}` (String) | `{"config":{"brand":"Tesla","mileage":5000}}` (Object) |
| `{"detail":"{\"weight\":1.5}"}` | `{"detail":{"weight":1.5,"tags":["new","sale"]}}` |

Always return Java objects. Never manually `JSON.toJSONString()` into entity fields.

## Implementation Notes

- See `references/static-json.md` for Pattern 1: fixed-type JSON with MyBatis-Plus and JPA.
- See `references/dynamic-json.md` for Pattern 2: polymorphic JSON with MyBatis-Plus and JPA, including custom deserializer and converter.
- See `references/sql-query.md` for MySQL JSON query examples in XML mapper and JPQL.
