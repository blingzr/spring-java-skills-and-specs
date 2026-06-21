# Static JSON: Fixed Type (MyBatis-Plus + JPA)

JSON structure never changes. Single Java class maps directly to the JSON column.

## MyBatis-Plus

```java
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler;

@Data
@TableName(value = "product", autoResultMap = true)  // autoResultMap is required
public class Product {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String name;

    // Static JSON: always ProductDetail shape
    @TableField(typeHandler = JacksonTypeHandler.class)
    private ProductDetail detail;
}

@Data
public class ProductDetail {
    private String brand;
    private BigDecimal weight;
    private List<String> tags;
}
```

**Critical:** `@TableName(autoResultMap = true)` is mandatory. Without it, MyBatis-Plus won't apply the type handler on `SELECT` result mapping, only on `INSERT`/`UPDATE`.

## JPA (Hibernate 6+)

```java
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Hibernate 6 native JSON support
    @JdbcTypeCode(SqlTypes.JSON)
    private ProductDetail detail;
}

// ProductDetail is a plain POJO, no JPA annotations needed
public class ProductDetail {
    private String brand;
    private BigDecimal weight;
    private List<String> tags;
    // getters/setters
}
```

### Alternative: hibernate-types (for Hibernate 5)

```java
import com.vladmihalcea.hibernate.type.json.JsonType;
import org.hibernate.annotations.Type;
import org.hibernate.annotations.TypeDef;

@Entity
@Table(name = "product")
@TypeDef(name = "json", typeClass = JsonType.class)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Type(type = "json")
    @Column(columnDefinition = "json")
    private ProductDetail detail;
}
```

## List of JSON Objects

### MyBatis-Plus

```java
@Data
@TableName(value = "order", autoResultMap = true)
public class Order {

    @TableId(type = IdType.AUTO)
    private Long id;

    // List of JSON objects — each item is a static type
    @TableField(typeHandler = JacksonTypeHandler.class)
    private List<OrderItem> items;
}

@Data
public class OrderItem {
    private Long productId;
    private Integer quantity;
    private BigDecimal price;
}
```

### JPA

```java
@Entity
@Table(name = "order_table")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @JdbcTypeCode(SqlTypes.JSON)
    private List<OrderItem> items;
}

public class OrderItem {
    private Long productId;
    private Integer quantity;
    private BigDecimal price;
}
```

## API Response (both ORMs)

Controller returns the entity directly. Jackson serializes `ProductDetail` as JSON Object:

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public Product get(@PathVariable Long id) {
        return productService.getById(id);
    }
}
```

Response:

```json
{
    "id": 1,
    "name": "Tesla Model 3",
    "detail": {
        "brand": "Tesla",
        "weight": 1611.00,
        "tags": ["electric", "sedan", "autopilot"]
    }
}
```

**Not:** `{"detail": "{\"brand\":\"Tesla\",...}"}` — never a String.

## Common Pitfalls

| Pitfall | Cause | Fix |
|---------|-------|-----|
| Detail field is `null` on SELECT | Missing `autoResultMap = true` | Add to `@TableName` |
| Detail stored as escaped string `"{\"...\"}"` | Double serialization in setter | Never call `JSON.toJSONString()` manually |
| Detail stored as string instead of JSON | `columnDefinition` mismatch | Ensure MySQL column is `JSON` type, not `TEXT` |
