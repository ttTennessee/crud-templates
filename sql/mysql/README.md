# MySQL DDL 模板（`sql/`）

本目录提供 **仅建表** 的 Handlebars 模板，`schema.sql.hbs`，与 Java/Vue/TS 业务代码生成解耦。

## 目录约定

| 路径 | 含义 |
|------|------|
| `sql/` | 落盘到 `db/migration/`（或你配置的 `[paths.aux]`） |
| `type_map.toml` | 可选；DSL → MySQL 类型映射 |

## 文件

| 文件 | 用途 |
|------|------|
| `schema.sql.hbs` | `CREATE TABLE` 语句 |
| `type_map.toml` | 无本地 map 时的默认映射 |

## 前置 YAML（`schema.sql.hbs` 顶部）

```yaml
---
basePath: "sql"
filename: "{{table}}.sql"
overwrite: never
---
```

- `basePath` / `filename`：单段路径，`filename` 不能含 `/`。
- `generateWhen` / `skipWhen`：与 `_variables.toml` 相同，互斥。

## 内置变量（`gen_context`）

| 变量 | 说明 |
|------|------|
| `{{model}}` / `{{model_pascal}}` / `{{model_snake}}` / `{{model_camel}}` / `{{model_kebab}}` | 命名 |
| `{{table}}` | 表名（`--table`） |
| `{{package}}` / `{{package_path}}` | 包路径 |
| `{{fields}}` | `{{#each fields}}` 列定义 |
| `{{git_user_name}}` / `{{user_email}}` / `{{user_name}}` / `{{user_email}}` | 作者 |
| `{{date}}` / `{{datetime}}` / `{{year}}` | 时间 |
| `{{ty_map type}}` | 需同 bundle 下存在 `type_map.toml` |

## `{{#each fields}}` 列

| 模板变量 | 来源 |
|----------|------|
| `name` / `name_*` | 列名 |
| `type` | 逻辑类型 |
| `is_pk` | 主键 |
| `nullable` | 可空 |
| `length` | 长度 |
| `unique` | 唯一 |
| `default` | 默认值（JSON 字面量） |
| `comment` | 列注释 |

**类型映射**（`type_map.toml`）：

| DSL | MySQL |
|-----|--------|
| `String` | `VARCHAR`；有 `length` 时 → `VARCHAR(n)` |
| `Long` / `Integer` / `Int` | `BIGINT` / `INT` |
| `Boolean` | `TINYINT(1)` |
| `LocalDateTime` | `DATETIME` |
| `LocalDate` | `DATE` |
| `BigDecimal` | `DECIMAL(19,2)` |
| `Double` / `Float` | `DOUBLE` / `FLOAT` |
| `Text` | `TEXT` |
| `Byte` | `TINYINT` |

## 调用示例

**JSON 实体**（`--file user.json`）：

```json
{
  "name": "User",
  "table": "sys_user",
  "package": "com.example.demo",
  "fields": [
    { "name": "id", "type": "Long", "is_pk": true, "comment": "主键" },
    { "name": "username", "type": "String", "length": 64, "nullable": false, "unique": true, "comment": "登录名" }
  ]
}
```

**生成**：

```bash
crud-cli gen --file user.json --type sql --stdout
# 预览，不写文件
```

```bash
crud-cli gen --file user.json --type sql \
  --var table_comment="系统用户"
```

**落盘**（Flyway 风格路径由 `setup.toml` 配置）：

```bash
crud-cli gen --file user.json --type sql \
  --var table_comment="系统用户"
```

## 与 `_variables.toml`

可在 `.crud/templates/_variables.toml` 声明：

```toml
[table_comment]
description = "表注释"
type = "string"
default = ""
```

通过 `--var table_comment=...` 传入；渲染进 `COMMENT='...'`。

## 注意

1. **`comment` 含单引号**：SQL 中需转义或换写法，见项目文档。
2. **主键**：`NOT NULL` + `AUTO_INCREMENT` + `PRIMARY KEY`；多列主键需改模板。
3. **`default`**：JSON 中字符串默认值用 `"..."`，数字用裸字面量。
