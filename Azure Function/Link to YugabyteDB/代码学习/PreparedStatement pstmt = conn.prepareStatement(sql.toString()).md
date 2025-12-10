```java
// 3. 执行查询
        try (PreparedStatement pstmt = conn.prepareStatement(sql.toString())) {
            for (int i = 0; i < params.size(); i++) {
                pstmt.setObject(i + 1, params.get(i));
            }

            ResultSet rs = pstmt.executeQuery();
            List<Map<String, Object>> resultList = new ArrayList<>();
            ResultSetMetaData md = rs.getMetaData();
            int columns = md.getColumnCount();
            while (rs.next()) {
                Map<String, Object> row = new HashMap<>();
                for (int i = 1; i <= columns; ++i) {
                    // 对于 jsonb 类型，getObject 可能会返回 PGobject，其 getValue() 方法返回 json 字符串
                    Object obj = rs.getObject(i);
                    if (obj instanceof org.postgresql.util.PGobject && "jsonb".equalsIgnoreCase(((org.postgresql.util.PGobject) obj).getType())) {
                        // 使用 Gson 将 json 字符串解析回 Java 对象，使输出更美观
                        row.put(md.getColumnName(i), gson.fromJson(((org.postgresql.util.PGobject) obj).getValue(), Object.class));
                    } else {
                        row.put(md.getColumnName(i), obj);
                    }
                }
                resultList.add(row);
            }
            return request.createResponseBuilder(HttpStatus.OK).header("Content-Type", "application/json").body(gson.toJson(resultList)).build();
        }

```


这一段查询的代码是什么逻辑，sql.toString()的值为SELECT * FROM notification_history WHERE notification_id = ?，，为什么sql语句中没有具体的参数值却能够正确的查出数据

==============================

您的问题触及了现代数据库编程中一个极其重要且安全的核心概念：**预编译语句 (PreparedStatement)**。

您观察得非常准确：SQL 语句字符串 `SELECT * FROM notification_history WHERE notification_id = ?` 中确实没有具体的参数值。它能正确查询出数据，是因为它采用了“**先发送模板，后填充数据**”的两步执行方式。

这是一个绝妙的设计，我们来详细分解一下。

### 核心逻辑：`PreparedStatement` 的工作原理

可以把 `PreparedStatement` 想象成一个“填空题”模板。

1.  **第一步：发送“模板”到数据库并进行预编译**
    ```java
    // sql.toString() 的值是 "SELECT * FROM ... WHERE notification_id = ?"
    PreparedStatement pstmt = conn.prepareStatement(sql.toString());
    ```
    当这行代码执行时，Java 程序会将这个带有问号 `?` 占位符的 SQL 语句**原封不动地**发送给数据库。

    数据库收到后，会做几件非常重要的事：
    *   **解析语法 (Parse)**：检查 SQL 语句的语法是否正确。
    *   **生成执行计划 (Execution Plan)**：分析这张表，决定最高效的查询方式（比如是否使用索引）。
    *   **预编译 (Pre-compile)**：将这个“模板”编译成一个可执行的中间状态，并**知道**在 `?` 的位置将来会有一个值被填入。

    此时，数据库已经做好了所有的准备工作，就等着您告诉它“空格”里该填什么了。

2.  **第二步：填充“空格”并执行**
    ```java
    // 假设 params 列表中有一个 UUID 对象
    for (int i = 0; i < params.size(); i++) {
        pstmt.setObject(i + 1, params.get(i)); // 填充第 1 个问号
    }

    ResultSet rs = pstmt.executeQuery(); // 执行！
    ```
    *   `pstmt.setObject(i + 1, params.get(i));` 这行代码的作用就是“填空”。它通过一个独立的、安全信道将**参数值**发送给数据库，并告诉它：“请把这个值填到我刚才给你的模板的第 1 个 `?` 里”。(注意：JDBC 的参数索引是从 1 开始的)。
    *   `pstmt.executeQuery();` 这行代码是最后的“执行”命令。数据库收到这个命令后，会拿出第一步预编译好的执行计划，把第二步收到的参数值安全地放进去，然后执行查询。

因为整个过程是分两步走的，所以 SQL 语句本身不需要包含具体的参数值。

### 为什么要这么做？`PreparedStatement` 的两大优势

这种看似复杂的方式，带来了两个巨大的好处，这也是为什么现代数据库编程**必须**使用它的原因。

#### 1. 终极安全：防止 SQL 注入 (SQL Injection)

这是最重要的原因。如果使用简单的字符串拼接来构造 SQL，会非常危险。

**危险的例子（字符串拼接）：**
```java
String id = "some-uuid' OR '1'='1"; // 恶意输入
String sql = "SELECT * FROM notification_history WHERE notification_id = '" + id + "'";
// 最终的 SQL 会变成:
// SELECT * FROM notification_history WHERE notification_id = 'some-uuid' OR '1'='1'
```
这个被篡改的 SQL 会绕过 `WHERE` 条件，返回表中的**所有数据**！这就是 SQL 注入攻击。

**`PreparedStatement` 如何防止注入：**
`PreparedStatement` 从根本上解决了这个问题，因为它严格区分了 **SQL 结构（代码）** 和 **参数值（数据）**。

*   SQL 结构在第一步就已经被数据库编译并固定了。
*   之后通过 `setObject()` 发送的任何数据，都**永远**只会被当作**纯粹的数据**来处理，绝不会被数据库误解为 SQL 命令的一部分。

即使恶意用户输入 `' OR '1'='1'`，数据库也只会把它当作一个普通的字符串，去尝试查找一个 `notification_id` 恰好等于这个奇怪字符串的记录（当然，肯定找不到），而不会改变 SQL 的执行逻辑。

#### 2. 性能提升

如果需要多次执行结构相同但参数不同的 SQL（例如，在一个循环中根据不同的 ID 查询用户信息），数据库只需要在第一次对 SQL 模板进行编译。后续的执行可以直接重用这个编译好的执行计划，只需替换参数即可，从而省去了反复解析和生成执行计划的开销，大大提高了性能。

### 总结您的代码段逻辑

1.  **`conn.prepareStatement(...)`**: 向数据库发送 `SELECT ... WHERE notification_id = ?` 这个“模板”，让数据库准备好执行计划。
2.  **`pstmt.setObject(...)`**: 单独、安全地将 `notification_id` 的具体值（一个 UUID 对象）发送给数据库，用于“填空”。
3.  **`pstmt.executeQuery()`**: 命令数据库执行已经准备好的计划和填充好的数据。
4.  **`while (rs.next()) { ... }`**: 遍历数据库返回的结果集，将每一行数据转换成一个 `Map`。
5.  **`if (obj instanceof org.postgresql.util.PGobject ...)`**: 对特殊数据类型（如 `jsonb`）进行特别处理，使其在最终的 JSON 响应中格式更友好。
6.  **`gson.toJson(resultList)`**: 将整个结果列表转换成一个 JSON 数组字符串，作为 HTTP 响应的主体返回给客户端。




好的，我们来逐行、逐块地详细分解您提供的这段代码，解释每一步的作用和目的。

### 代码概览

```java
// 3. 执行查询
try (PreparedStatement pstmt = conn.prepareStatement(sql.toString())) { // 块 1
    // 块 2
    for (int i = 0; i < params.size(); i++) {
        pstmt.setObject(i + 1, params.get(i));
    }

    // 块 3
    ResultSet rs = pstmt.executeQuery();
    List<Map<String, Object>> resultList = new ArrayList<>();
    ResultSetMetaData md = rs.getMetaData();
    int columns = md.getColumnCount();
    
    // 块 4
    while (rs.next()) {
        Map<String, Object> row = new HashMap<>();
        for (int i = 1; i <= columns; ++i) {
            Object obj = rs.getObject(i);
            // 块 5
            if (obj instanceof org.postgresql.util.PGobject && "jsonb".equalsIgnoreCase(((org.postgresql.util.PGobject) obj).getType())) {
                row.put(md.getColumnName(i), gson.fromJson(((org.postgresql.util.PGobject) obj).getValue(), Object.class));
            } else {
                row.put(md.getColumnName(i), obj);
            }
        }
        resultList.add(row);
    }
    
    // 块 6
    return request.createResponseBuilder(HttpStatus.OK)
            .header("Content-Type", "application/json")
            .body(gson.toJson(resultList))
            .build();
}
```

---

### 分步详解

#### 块 1: 准备 SQL 执行环境

```java
try (PreparedStatement pstmt = conn.prepareStatement(sql.toString())) {
```

*   **`try (...)`**: 这是 `try-with-resources` 语句。它是一个强大的 Java 特性，用于自动管理资源。在这里，它确保 `PreparedStatement` 对象 `pstmt` 在 `try` 代码块结束时（无论正常结束还是因异常中断）都会被**自动关闭**。这可以有效防止数据库连接资源泄漏。
*   **`conn.prepareStatement(sql.toString())`**: 这是核心准备步骤。
    *   **`conn`**: 这是一个已经建立好的数据库连接对象。
    *   **`sql.toString()`**: 获取之前构建好的 SQL 语句字符串，例如 `SELECT * FROM notification_history WHERE notification_id = ?`。
    *   **作用**: 将这个带有 `?` 占位符的 SQL 语句发送到数据库进行**预编译**。数据库会解析这个 SQL 模板，生成一个高效的执行计划，并返回一个 `PreparedStatement` 对象 (`pstmt`)。这个对象现在就代表了那个准备好的、等待填充参数的 SQL 命令。

#### 块 2: 填充 SQL 参数

```java
for (int i = 0; i < params.size(); i++) {
    pstmt.setObject(i + 1, params.get(i));
}
```

*   **`for (int i = 0; i < params.size(); i++)`**: 这是一个循环，用于遍历之前收集到的所有查询参数。`params` 是一个 `List<Object>`，里面存放着需要填入 SQL 语句 `?` 占位符的实际值。
*   **`pstmt.setObject(i + 1, params.get(i))`**: 这是将参数值绑定到 SQL 模板上的关键操作。
    *   **`i + 1`**: 第一个参数是参数的索引。**非常重要**：JDBC 中 `PreparedStatement` 的参数索引是从 **1** 开始的，而不是从 0 开始。所以这里用 `i + 1`。
    *   **`params.get(i)`**: 获取 `params` 列表中的第 `i` 个参数值（例如一个 `UUID` 对象或一个字符串）。
    *   **`setObject(...)`**: 这是一个通用的设置参数的方法，它可以接受任何类型的 Java 对象。JDBC 驱动会尝试将这个 Java 对象映射到合适的数据库类型。这比使用 `setString()`, `setInt()` 等更灵活。

#### 块 3: 执行查询并准备处理结果

```java
ResultSet rs = pstmt.executeQuery();
List<Map<String, Object>> resultList = new ArrayList<>();
ResultSetMetaData md = rs.getMetaData();
int columns = md.getColumnCount();
```

*   **`ResultSet rs = pstmt.executeQuery()`**: 执行已经准备好并填充了参数的 SQL 查询。数据库会返回查询结果，`executeQuery` 方法将其封装成一个 `ResultSet` 对象 (`rs`)。你可以把 `ResultSet` 想象成一个指向查询结果集第一行之前的**游标 (cursor)**。
*   **`List<Map<String, Object>> resultList = new ArrayList<>()`**: 创建一个空的 `List`。这个列表最终将用来存放所有查询结果行，每一行都是一个 `Map`。
*   **`ResultSetMetaData md = rs.getMetaData()`**: 获取关于 `ResultSet` 的**元数据 (metadata)**。元数据就是“关于数据的数据”。`ResultSetMetaData` 对象 (`md`) 包含了结果集的各种信息，比如有多少列、每一列的名称是什么、每一列的数据类型是什么等。
*   **`int columns = md.getColumnCount()`**: 从元数据中获取结果集的总列数，并存入 `columns` 变量。这使得后续的循环可以动态地处理任意数量的列，而无需硬编码。

#### 块 4: 遍历结果集并处理每一行

```java
while (rs.next()) {
    // ... 内部代码 ...
}
```

*   **`while (rs.next())`**: 这是遍历 `ResultSet` 的标准方式。
    *   `rs.next()` 方法做了两件事：1. 将游标从当前位置向下移动一行。2. 如果移动后游标指向一个有效的数据行，则返回 `true`；如果已经没有更多行（到达结果集末尾），则返回 `false`。
    *   因此，这个 `while` 循环会一直执行，直到处理完所有的数据行。

#### 块 5: 将一行数据转换为 Map

```java
Map<String, Object> row = new HashMap<>();
for (int i = 1; i <= columns; ++i) {
    Object obj = rs.getObject(i);
    if (obj instanceof org.postgresql.util.PGobject && "jsonb".equalsIgnoreCase(((org.postgresql.util.PGobject) obj).getType())) {
        row.put(md.getColumnName(i), gson.fromJson(((org.postgresql.util.PGobject) obj).getValue(), Object.class));
    } else {
        row.put(md.getColumnName(i), obj);
    }
}
resultList.add(row);
```

*   **`Map<String, Object> row = new HashMap<>()`**: 在每次循环开始时，创建一个新的 `HashMap`。这个 `Map` 用于表示当前行的数据，键是列名，值是列的值。
*   **`for (int i = 1; i <= columns; ++i)`**: 遍历当前行的每一列。列的索引也是从 1 开始。
*   **`Object obj = rs.getObject(i)`**: 从 `ResultSet` 的当前行中，获取第 `i` 列的值。`getObject()` 是一个非常通用的方法，它会以最合适的 Java 对象类型返回数据（如 `Integer`, `String`, `Timestamp`, `UUID` 等）。
*   **`if (obj instanceof org.postgresql.util.PGobject ...)`**: 这是一个针对 **PostgreSQL 特殊类型**的处理。
    *   像 `jsonb`, `json` 等类型，PostgreSQL 的 JDBC 驱动默认会将它们作为 `PGobject` 返回。
    *   这个 `if` 语句检查返回的对象是不是 `PGobject`，并且其内部类型是不是 `jsonb`。
    *   如果是，`((org.postgresql.util.PGobject) obj).getValue()` 会获取其原始的 JSON **字符串**。
    *   `gson.fromJson(..., Object.class)` 使用 `Gson` 库将这个 JSON 字符串**反序列化**成一个 Java 对象（通常是 `Map` 或 `List`）。这样做的目的是让最终输出的 JSON 更加美观和结构化，而不是一个嵌套的字符串。
*   **`else { row.put(md.getColumnName(i), obj); }`**: 如果不是 `jsonb` 类型，就直接将从数据库获取的对象 `obj` 和对应的列名（通过 `md.getColumnName(i)` 获取）存入 `row` 这个 Map 中。
*   **`resultList.add(row)`**: 当一行中的所有列都被处理完毕后，将代表这一整行数据的 `row` Map 添加到 `resultList` 中。

#### 块 6: 构建并返回 HTTP 响应

```java
return request.createResponseBuilder(HttpStatus.OK)
        .header("Content-Type", "application/json")
        .body(gson.toJson(resultList))
        .build();
```

*   **`request.createResponseBuilder(HttpStatus.OK)`**: 开始构建一个 HTTP 响应，并设置状态码为 `200 OK`，表示请求成功。
*   **`.header("Content-Type", "application/json")`**: 设置一个 HTTP 响应头，告诉客户端响应体的内容是 JSON 格式。
*   **`.body(gson.toJson(resultList))`**: 将包含所有查询结果的 `resultList`（一个 `List<Map>`）使用 `Gson` 库序列化成一个 JSON 字符串（例如 `[{"id":1, "name":"A"}, {"id":2, "name":"B"}]`），并将其设置为响应的主体。
*   **`.build()`**: 完成构建，生成最终的 `HttpResponseMessage` 对象并返回。这个对象将被 Azure Functions 运行时发送给客户端。