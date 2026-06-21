# Dynamic JSON: Polymorphic Deserialization (MyBatis-Plus + JPA)

JSON structure varies by a `type` discriminator. The Java field is an abstract base class; Jackson deserializes to the correct concrete subclass at runtime.

## Class Design

```java
// Abstract base — all subclasses extend this
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type",
    visible = true  // keep 'type' in deserialized object
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CarConfig.class, name = "CAR"),
    @JsonSubTypes.Type(value = HouseConfig.class, name = "HOUSE")
})
public abstract class AssetConfig {
    private String type;  // matches the discriminator property
}

// Concrete subclass: CAR
public class CarConfig extends AssetConfig {
    private String brand;
    private String licensePlate;
    private Integer mileage;
    private Integer seatCount;
}

// Concrete subclass: HOUSE
public class HouseConfig extends AssetConfig {
    private String address;
    private BigDecimal area;
    private Integer floorCount;
    private Integer roomCount;
}
```

**Critical:** `visible = true` is required. Without it, Jackson removes the `type` property during deserialization, causing the field to be missing when the object is later serialized back to JSON.

## MyBatis-Plus

MyBatis-Plus with `JacksonTypeHandler` fully supports Jackson polymorphism out of the box. The `type` field inside the JSON string drives the deserialization.

```java
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler;

@Data
@TableName(value = "asset", autoResultMap = true)
public class Asset {

    @TableId(type = IdType.AUTO)
    private Long id;

    // Database type column — for indexing and business logic
    private String type;

    // Polymorphic JSON — JacksonTypeHandler respects @JsonTypeInfo
    @TableField(typeHandler = JacksonTypeHandler.class)
    private AssetConfig config;
}
```

### Stored JSON Example

```json
-- CAR record
{
    "type": "CAR",
    "brand": "Tesla",
    "licensePlate": "京A12345",
    "mileage": 15000,
    "seatCount": 5
}

-- HOUSE record
{
    "type": "HOUSE",
    "address": "北京市朝阳区xxx",
    "area": 128.5,
    "floorCount": 32,
    "roomCount": 3
}
```

### Service Layer

```java
@Service
public class AssetService {

    @Autowired
    private AssetMapper assetMapper;

    public Asset create(String type, AssetConfig config) {
        // Validate type matches config class
        if (!type.equals(config.getType())) {
            throw new IllegalArgumentException("Type mismatch: " + type + " vs " + config.getType());
        }
        Asset asset = new Asset();
        asset.setType(type);
        asset.setConfig(config);
        assetMapper.insert(asset);
        return asset;
    }

    public Asset getById(Long id) {
        Asset asset = assetMapper.selectById(id);
        // Config is already the correct subclass (CarConfig or HouseConfig)
        return asset;
    }

    // Type-safe accessor with pattern matching (Java 17+)
    public String getDescription(Long id) {
        Asset asset = getById(id);
        return switch (asset.getConfig()) {
            case CarConfig car -> car.getBrand() + " " + car.getLicensePlate();
            case HouseConfig house -> house.getAddress() + ", " + house.getArea() + "m2";
            default -> "Unknown";
        };
    }
}
```

## JPA (Hibernate 6+)

JPA has two approaches for polymorphic JSON:

### Approach A: JSON self-describing (type inside JSON)

Same as MyBatis-Plus — the `type` property inside the JSON drives polymorphism. Uses `@JdbcTypeCode(SqlTypes.JSON)` directly.

```java
@Entity
@Table(name = "asset")
public class Asset {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String type;

    @JdbcTypeCode(SqlTypes.JSON)
    private AssetConfig config;  // @JsonTypeInfo on AssetConfig handles polymorphism
}
```

**Requirement:** The JSON stored in MySQL **must** contain the `"type"` property. Hibernate passes the JSON bytes to Jackson, which uses `@JsonTypeInfo` to select the correct subclass.

### Approach B: Database type column as discriminator (JSON may omit type)

Use a custom `AttributeConverter` that uses the entity's `type` column (not the JSON property) to decide the target class. This allows the JSON to omit the `type` field, saving storage.

```java
@Entity
@Table(name = "asset")
public class Asset {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String type;  // "CAR" or "HOUSE" — used as discriminator

    @Convert(converter = AssetConfigConverter.class)
    @Column(columnDefinition = "json")
    private AssetConfig config;
}
```

**Custom Converter (uses entity type column):**

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

@Converter
public class AssetConfigConverter implements AttributeConverter<AssetConfig, String> {

    private static final ObjectMapper mapper = new ObjectMapper();

    // Registry: type value -> target class
    private static final Map<String, Class<? extends AssetConfig>> TYPE_REGISTRY = Map.of(
        "CAR", CarConfig.class,
        "HOUSE", HouseConfig.class
    );

    @Override
    public String convertToDatabaseColumn(AssetConfig config) {
        if (config == null) return null;
        try {
            // Serialize — include type if not present
            return mapper.writeValueAsString(config);
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize AssetConfig", e);
        }
    }

    @Override
    public AssetConfig convertToEntityAttribute(String json) {
        if (json == null || json.isBlank()) return null;
        // Cannot determine type from json alone — need entity context
        // This approach requires a Hibernate UserType or Component instead
        throw new UnsupportedOperationException(
            "Use AssetConfigUserType with access to entity state"
        );
    }
}
```

**Better Approach: Hibernate Component with `@Parent` (recommended for JPA)**

```java
@Entity
@Table(name = "asset")
public class Asset {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String type;

    // Use Component pattern to access parent entity state
    @JdbcTypeCode(SqlTypes.JSON)
    @JsonDeserialize(using = AssetConfigDeserializer.class)
    private AssetConfig config;

    // Package-private accessor for deserializer
    String getTypeValue() {
        return type;
    }
}
```

**Custom Deserializer (accesses parent entity state):**

```java
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;

public class AssetConfigDeserializer extends JsonDeserializer<AssetConfig> {

    private static final Map<String, Class<? extends AssetConfig>> REGISTRY = Map.of(
        "CAR", CarConfig.class,
        "HOUSE", HouseConfig.class
    );

    @Override
    public AssetConfig deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        ObjectNode node = p.readValueAsTree();

        // If JSON contains type, use it (self-describing)
        if (node.has("type")) {
            String type = node.get("type").asText();
            Class<? extends AssetConfig> clazz = REGISTRY.get(type);
            if (clazz != null) {
                return p.getCodec().treeToValue(node, clazz);
            }
            throw new IllegalArgumentException("Unknown type: " + type);
        }

        // Otherwise, try to get type from parent entity context
        // This requires a ThreadLocal or similar mechanism to pass context
        String parentType = AssetTypeHolder.get();  // ThreadLocal set by service layer
        if (parentType != null) {
            Class<? extends AssetConfig> clazz = REGISTRY.get(parentType);
            if (clazz != null) {
                // Inject type into JSON node
                node.put("type", parentType);
                return p.getCodec().treeToValue(node, clazz);
            }
        }

        throw new IllegalArgumentException("Cannot determine AssetConfig type");
    }
}
```

**ThreadLocal context holder:**

```java
public class AssetTypeHolder {
    private static final ThreadLocal<String> TYPE = new ThreadLocal<>();

    public static void set(String type) {
        TYPE.set(type);
    }

    public static String get() {
        return TYPE.get();
    }

    public static void clear() {
        TYPE.remove();
    }
}
```

**Service layer with context:**

```java
@Transactional
public Asset save(Asset asset) {
    try {
        AssetTypeHolder.set(asset.getType());
        return assetRepository.save(asset);
    } finally {
        AssetTypeHolder.clear();
    }
}

@Transactional(readOnly = true)
public Optional<Asset> findById(Long id) {
    Optional<Asset> result = assetRepository.findById(id);
    result.ifPresent(asset -> {
        // After load, ensure type is set in config
        if (asset.getConfig() != null && asset.getConfig().getType() == null) {
            asset.getConfig().setType(asset.getType());
        }
    });
    return result;
}
```

## Comparison: MyBatis-Plus vs JPA for Polymorphic JSON

| Aspect | MyBatis-Plus | JPA (self-describing) | JPA (type column) |
|--------|-------------|----------------------|-------------------|
| **Type source** | JSON `type` property | JSON `type` property | Database `type` column |
| **JSON may omit type** | No | No | Yes |
| **Configuration** | Minimal — just annotations | Minimal — just annotations | Custom deserializer + ThreadLocal |
| **Storage efficiency** | JSON contains redundant type | Same | JSON smaller (no type field) |
| **Complexity** | Low | Low | Medium |
| **Recommendation** | Default choice | Default for JPA | Only when JSON size matters |

## Adding a New Subtype

When adding a new config type (e.g., `BOAT`):

1. Create the subclass:
```java
public class BoatConfig extends AssetConfig {
    private String hullNumber;
    private Integer enginePower;
    private BigDecimal length;
}
```

2. Register in `@JsonSubTypes`:
```java
@JsonSubTypes({
    @JsonSubTypes.Type(value = CarConfig.class, name = "CAR"),
    @JsonSubTypes.Type(value = HouseConfig.class, name = "HOUSE"),
    @JsonSubTypes.Type(value = BoatConfig.class, name = "BOAT")  // new
})
```

3. Add to JPA converter registry (if using custom converter):
```java
private static final Map<String, Class<? extends AssetConfig>> TYPE_REGISTRY = Map.of(
    "CAR", CarConfig.class,
    "HOUSE", HouseConfig.class,
    "BOAT", BoatConfig.class  // new
);
```

4. Add database check constraint if needed:
```sql
ALTER TABLE asset DROP CONSTRAINT chk_type;
ALTER TABLE asset ADD CONSTRAINT chk_type CHECK (type IN ('CAR', 'HOUSE', 'BOAT'));
```
