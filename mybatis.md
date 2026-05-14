---
description: "MyBatis SQL injection prevention — when to use #{} vs ${} and how to fix scanner findings"
applyTo: "**/*Mapper.xml,**/mappers/**/*.xml,**/mapper/**/*.java,**/*Mapper.java,**/*Dao.java"
---

# MyBatis SQL Injection Prevention

## Core rule

MyBatis already uses JDBC `PreparedStatement` under the hood — you do not need to replace it with hand-written prepared statements. The vulnerability is almost always misuse of `${}` vs `#{}`.

- **`#{param}`** → compiles to `?` placeholder + `PreparedStatement.setX()`. **Safe.** This IS a prepared statement.
- **`${param}`** → raw string substitution into SQL before execution. **Vulnerable to SQL injection.**

When a scanner flags MyBatis code, the fix is almost always changing `${}` to `#{}`, not migrating the framework.

## Detection — flag these as vulnerabilities

### 1. `${}` used for a value (always vulnerable)

```xml
<!-- VULNERABLE -->
<select id="findUser" resultType="User">
  SELECT * FROM users WHERE username = '${username}'
</select>

<select id="search" resultType="Item">
  SELECT * FROM items WHERE name LIKE '%${keyword}%'
</select>
```

```java
// VULNERABLE
@Select("SELECT * FROM users WHERE id = ${id}")
User findById(@Param("id") Long id);
```

### 2. String concatenation inside `@SelectProvider` / `@UpdateProvider` / SQL builder classes

```java
// VULNERABLE
public String buildQuery(Map<String, Object> params) {
    return "SELECT * FROM users WHERE name = '" + params.get("name") + "'";
}
```

### 3. `<sql>` fragments that interpolate `${}` for values

```xml
<!-- VULNERABLE — the fragment leaks ${} into every caller -->
<sql id="userFilter">
  WHERE username = '${username}'
</sql>
```

### 4. Dynamic SQL tags using `${}` for value comparisons

```xml
<!-- VULNERABLE -->
<if test="status != null">
  AND status = '${status}'
</if>
```

## Fixes

### Standard fix — replace `${}` with `#{}`

```xml
<!-- SAFE -->
<select id="findUser" resultType="User">
  SELECT * FROM users WHERE username = #{username}
</select>

<select id="search" resultType="Item">
  SELECT * FROM items WHERE name LIKE CONCAT('%', #{keyword}, '%')
</select>
```

```java
// SAFE
@Select("SELECT * FROM users WHERE id = #{id}")
User findById(@Param("id") Long id);
```

Note for `LIKE`: do not write `'%#{keyword}%'` — quotes around `#{}` break parameter binding. Use `CONCAT('%', #{keyword}, '%')` or build the wildcard string in Java before passing it.

### Legitimate uses of `${}` — structural SQL only

`${}` is required (and acceptable) when you genuinely cannot use a JDBC placeholder:

- Dynamic table names
- Dynamic column names
- `ORDER BY` column / direction
- `IN` clause with a dynamic list (prefer `<foreach>` with `#{}` instead — see below)

When `${}` is unavoidable, **the value must be whitelisted in Java before reaching the mapper**. Never let raw user input flow into `${}`.

```java
// SAFE — whitelist before passing to mapper
private static final Set<String> ALLOWED_SORT_COLUMNS =
    Set.of("name", "created_at", "id", "email");
private static final Set<String> ALLOWED_DIRECTIONS = Set.of("ASC", "DESC");

public List<User> findUsers(String sortColumn, String direction) {
    if (!ALLOWED_SORT_COLUMNS.contains(sortColumn)) {
        throw new IllegalArgumentException("Invalid sort column: " + sortColumn);
    }
    if (!ALLOWED_DIRECTIONS.contains(direction.toUpperCase())) {
        throw new IllegalArgumentException("Invalid sort direction");
    }
    return mapper.findUsers(sortColumn, direction);
}
```

```xml
<!-- SAFE — only reached with whitelisted values -->
<select id="findUsers" resultType="User">
  SELECT * FROM users ORDER BY ${sortColumn} ${direction}
</select>
```

### `IN` clauses — use `<foreach>`, not `${}`

```xml
<!-- VULNERABLE -->
<select id="findByIds" resultType="User">
  SELECT * FROM users WHERE id IN (${ids})
</select>

<!-- SAFE -->
<select id="findByIds" resultType="User">
  SELECT * FROM users WHERE id IN
  <foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
  </foreach>
</select>
```

## Review checklist

When reviewing a MyBatis mapper or generating new mapper code:

1. Search for `${` — every occurrence needs justification.
2. If `${}` holds a value → change to `#{}`.
3. If `${}` holds a table/column/direction → confirm a whitelist exists in the calling Java code.
4. Check `@Select`, `@Update`, `@Insert`, `@Delete` annotations for string concatenation with `+`.
5. Check `@SelectProvider` / SQL builder methods for `String` concatenation of user input.
6. `IN` clauses should use `<foreach>` with `#{}`, never `${}` on a joined string.
7. `LIKE` patterns should use `CONCAT('%', #{x}, '%')`, not quotes around `#{}`.

## What NOT to recommend

- Do **not** suggest replacing MyBatis with raw JDBC prepared statements — MyBatis already uses prepared statements via `#{}`.
- Do **not** suggest input sanitization / escaping as a fix — parameterization via `#{}` is the correct fix.
- Do **not** suggest stored procedures as a fix — they have the same `#{}` vs `${}` discipline.

## Quick reference

| Need | Use |
|------|-----|
| Bind a value (WHERE, SET, VALUES, etc.) | `#{param}` |
| Dynamic table / column name | `${param}` + Java-side whitelist |
| ORDER BY direction | `${param}` + Java-side whitelist (`ASC`/`DESC` only) |
| `IN (...)` with a list | `<foreach>` with `#{}` |
| `LIKE` pattern | `CONCAT('%', #{param}, '%')` |
