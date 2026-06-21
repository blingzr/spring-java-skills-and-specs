# MySQL JSON Query (XML Mapper + JPQL)

Query JSON fields using MySQL native operators. Combine with the separate `type` column for efficient filtering.

## MySQL JSON Operators Reference

| Operator | Returns | Use Case |
|----------|---------|----------|
| `->` | JSON value (quoted) | Extract JSON fragment, keep JSON type |
| `->>` | Unquoted string | Extract scalar value for comparison |
| `JSON_EXTRACT(col, path)` | JSON value | Multiple path extraction, array search |
| `JSON_CONTAINS(col, val, path)` | Boolean (0/1) | Check if JSON contains a value |
| `JSON_KEYS(col)` | JSON array | List all keys in JSON object |

## MyBatis-Plus

### XML Mapper

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.AssetMapper">

    <!-- ResultMap with typeHandler -->
    <resultMap id="BaseResultMap" type="com.example.entity.Asset">
        <id column="id" property="id"/>
        <result column="type" property="type"/>
        <result column="config" property="config"
                typeHandler="com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler"/>
    </resultMap>

    <!-- Query by JSON field: CAR with specific brand -->
    <select id="findByBrand" resultMap="BaseResultMap">
        SELECT * FROM asset
        WHERE type = 'CAR'
          AND config ->> '$.brand' = #{brand}
    </select>

    <!-- Query by numeric JSON field: HOUSE with area > threshold -->
    <select id="findLargeHouses" resultMap="BaseResultMap">
        SELECT * FROM asset
        WHERE type = 'HOUSE'
          AND CAST(config ->> '$.area' AS DECIMAL(10,2)) > #{minArea}
    </select>

    <!-- Query by nested JSON field -->
    <select id="findByCity" resultMap="BaseResultMap">
        SELECT * FROM asset
        WHERE type = 'HOUSE'
          AND config ->> '$.address.city' = #{city}
    </select>

    <!-- Query by JSON array element -->
    <select id="findByTag" resultMap="BaseResultMap">
        SELECT * FROM product
        WHERE detail ->> '$.tags[0]' = #{tag}
           OR JSON_CONTAINS(detail ->> '$.tags', '"' || #{tag} || '"')
    </select>

    <!-- Check JSON path existence -->
    <select id="findWithMileage" resultMap="BaseResultMap">
        SELECT * FROM asset
        WHERE type = 'CAR'
          AND config ->> '$.mileage' IS NOT NULL
    </select>

    <!-- Complex multi-condition query -->
    <select id="searchCars" resultMap="BaseResultMap">
        SELECT * FROM asset
        WHERE type = 'CAR'
        <if test="brand != null">
            AND config ->> '$.brand' = #{brand}
        </if>
        <if test="minMileage != null">
            AND CAST(config ->> '$.mileage' AS UNSIGNED) >= #{minMileage}
        </if>
        <if test="maxMileage != null">
            AND CAST(config ->> '$.mileage' AS UNSIGNED) <= #{maxMileage}
        </if>
        ORDER BY id DESC
    </select>

    <!-- JSON_CONTAINS for array membership -->
    <select id="findByFeature" resultMap="BaseResultMap">
        SELECT * FROM product
        WHERE JSON_CONTAINS(detail -> '$.tags', '"' || #{feature} || '"', '$')
    </select>

    <!-- Partial JSON update (MySQL 8.0.3+) -->
    <update id="updateMileage">
        UPDATE asset
        SET config = JSON_SET(config, '$.mileage', #{mileage})
        WHERE id = #{id}
          AND type = 'CAR'
    </update>

    <!-- Remove JSON key -->
    <update id="removeDeprecatedField">
        UPDATE asset
        SET config = JSON_REMOVE(config, '$.deprecatedField')
        WHERE type = 'CAR'
    </update>

</mapper>
```

### @Select Annotation

```java
@Mapper
public interface AssetMapper extends BaseMapper<Asset> {

    @Select("SELECT * FROM asset WHERE type = 'CAR' AND config ->> '$.brand' = #{brand}")
    @ResultMap("BaseResultMap")
    List<Asset> findByBrand(@Param("brand") String brand);

    @Select("""
        SELECT * FROM asset
        WHERE type = 'HOUSE'
          AND CAST(config ->> '$.area' AS DECIMAL(10,2)) > #{minArea}
        """)
    @ResultMap("BaseResultMap")
    List<Asset> findLargeHouses(@Param("minArea") BigDecimal minArea);

    @Select("""
        SELECT * FROM asset
        WHERE type = #{type}
          AND config ->> ${jsonPath} = #{value}
        """)
    @ResultMap("BaseResultMap")
    List<Asset> findByJsonPath(
        @Param("type") String type,
        @Param("jsonPath") String jsonPath,   // e.g., "$.brand" — use ${} not #{} for path
        @Param("value") String value
    );
}
```

**Warning:** `${jsonPath}` uses direct string substitution (not prepared statement). Ensure `jsonPath` is server-generated, never user-input, to prevent SQL injection.

## JPA

### JPQL with `function()`

JPA does not natively support JSON operators. Use `function('JSON_EXTRACT', ...)` or `function('', ...)` for vendor-specific functions.

```java
@Repository
public interface AssetRepository extends JpaRepository<Asset, Long> {

    // Query by JSON field — returns JSON fragment (quoted)
    @Query("SELECT a FROM Asset a WHERE a.type = 'CAR' AND " +
           "function('JSON_EXTRACT', a.config, '$.brand') = function('JSON_QUOTE', :brand)")
    List<Asset> findByBrand(@Param("brand") String brand);

    // Query by JSON field — returns unquoted scalar
    @Query("SELECT a FROM Asset a WHERE a.type = 'HOUSE' AND " +
           "function('', a.config, '$.address') LIKE %:city%")
    List<Asset> findByCity(@Param("city") String city);

    // Numeric comparison — must cast
    @Query("SELECT a FROM Asset a WHERE a.type = 'HOUSE' AND " +
           "CAST(function('', a.config, '$.area') AS DECIMAL(10,2)) > :minArea")
    List<Asset> findLargeHouses(@Param("minArea") BigDecimal minArea);

    // Check JSON path existence
    @Query("SELECT a FROM Asset a WHERE a.type = 'CAR' AND " +
           "function('', a.config, '$.mileage') IS NOT NULL")
    List<Asset> findCarsWithMileage();

    // JSON_CONTAINS for array
    @Query("SELECT a FROM Asset a WHERE " +
           "function('JSON_CONTAINS', a.config, function('JSON_QUOTE', :feature), '$.tags') = 1")
    List<Asset> findByFeature(@Param("feature") String feature);
}
```

### Native Query

For complex JSON queries, native SQL is often cleaner than JPQL with `function()`:

```java
@Query(value = """
    SELECT * FROM asset
    WHERE type = :type
      AND config ->> :jsonPath = :value
    """, nativeQuery = true)
List<Asset> findByJsonPathNative(
    @Param("type") String type,
    @Param("jsonPath") String jsonPath,
    @Param("value") String value
);

@Query(value = """
    SELECT * FROM asset
    WHERE type = 'CAR'
      AND config ->> '$.brand' = :brand
      AND CAST(config ->> '$.mileage' AS UNSIGNED) BETWEEN :minMileage AND :maxMileage
    ORDER BY CAST(config ->> '$.mileage' AS UNSIGNED) DESC
    """, nativeQuery = true)
List<Asset> searchCars(
    @Param("brand") String brand,
    @Param("minMileage") Integer minMileage,
    @Param("maxMileage") Integer maxMileage
);
```

### EntityManager Dynamic Query

```java
@Service
public class AssetQueryService {

    @PersistenceContext
    private EntityManager em;

    public List<Asset> dynamicQuery(String type, Map<String, Object> jsonFilters) {
        StringBuilder sql = new StringBuilder("SELECT * FROM asset WHERE type = ?");
        List<Object> params = new ArrayList<>();
        params.add(type);

        for (Map.Entry<String, Object> entry : jsonFilters.entrySet()) {
            String path = entry.getKey();     // e.g., "$.brand"
            Object value = entry.getValue();

            if (value instanceof Number) {
                sql.append(" AND CAST(config ->> ? AS DECIMAL(10,2)) = ?");
                params.add(path);
                params.add(value);
            } else {
                sql.append(" AND config ->> ? = ?");
                params.add(path);
                params.add(value.toString());
            }
        }

        Query query = em.createNativeQuery(sql.toString(), Asset.class);
        for (int i = 0; i < params.size(); i++) {
            query.setParameter(i + 1, params.get(i));
        }
        return query.getResultList();
    }
}
```

## Query Patterns by Scenario

### Exact Match on JSON Field

```sql
-- MyBatis-Plus XML
WHERE type = 'CAR' AND config ->> '$.brand' = #{brand}

-- JPA JPQL
WHERE a.type = 'CAR' AND function('', a.config, '$.brand') = :brand
```

### Numeric Range

```sql
-- Must cast for numeric comparison
WHERE type = 'HOUSE'
  AND CAST(config ->> '$.area' AS DECIMAL(10,2)) BETWEEN #{min} AND #{max}

-- Integer
WHERE type = 'CAR'
  AND CAST(config ->> '$.mileage' AS UNSIGNED) > #{minMileage}
```

### Array Contains

```sql
-- Check if JSON array contains a value
WHERE JSON_CONTAINS(config -> '$.tags', '"sale"', '$')

-- Check if JSON array contains any of multiple values
WHERE JSON_OVERLAPS(config -> '$.tags', '["sale", "new"]')
```

### Nested Object Path

```sql
-- Access nested fields with dot notation
WHERE config ->> '$.owner.name' = #{ownerName}
WHERE config ->> '$.address.city' LIKE CONCAT('%', #{city}, '%')
```

### Existence Check

```sql
-- Check if a key exists in JSON (value may be null)
WHERE config ->> '$.optionalField' IS NOT NULL

-- Check if JSON is not null and not empty object
WHERE config IS NOT NULL AND config != '{}'
```

### Sort by JSON Field

```sql
-- Sort by numeric JSON field (must cast)
ORDER BY CAST(config ->> '$.price' AS DECIMAL(10,2)) DESC

-- Sort by string JSON field
ORDER BY config ->> '$.brand' ASC
```

## Performance Optimization

### Index Strategy

JSON path queries cannot use B-tree indexes directly on the JSON column. Use these strategies:

1. **Type column index** (most important):
```sql
CREATE INDEX idx_asset_type ON asset(type);
-- Always filter by type first, then JSON path
```

2. **Generated column index** (MySQL 5.7.6+ / 8.0):
```sql
-- Add generated column for frequently queried JSON path
ALTER TABLE asset ADD COLUMN config_brand VARCHAR(50)
    GENERATED ALWAYS AS (config ->> '$.brand') STORED;

CREATE INDEX idx_asset_brand ON asset(config_brand);

-- Query becomes sargable
SELECT * FROM asset WHERE config_brand = 'Tesla';
```

3. **Multi-column index**:
```sql
CREATE INDEX idx_asset_type_brand ON asset(type, config_brand);
```

4. **Virtual column** (not stored, computed on read):
```sql
ALTER TABLE asset ADD COLUMN config_area DECIMAL(10,2)
    GENERATED ALWAYS AS (CAST(config ->> '$.area' AS DECIMAL(10,2))) VIRTUAL;
```

### Query Execution Order

Always place the most selective condition first:

```sql
-- Good: type is indexed, reduces rows before JSON scan
SELECT * FROM asset WHERE type = 'CAR' AND config ->> '$.brand' = 'Tesla';

-- Bad: JSON path cannot use index, full table scan
SELECT * FROM asset WHERE config ->> '$.brand' = 'Tesla';
```

### Execution Plan Check

```sql
EXPLAIN SELECT * FROM asset WHERE type = 'CAR' AND config ->> '$.brand' = 'Tesla';
-- Look for: type=ref, key=idx_asset_type
```
