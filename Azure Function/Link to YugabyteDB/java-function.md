[Create and deploy function code to Azure using Visual Studio Code | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/how-to-create-function-vs-code?pivot=programming-language-java&pivots=programming-language-java)

graph TD
    A[Flowable 流程启动] --> B{BPMN 模型};
    B --> C[服务任务: "查询员工信息"];
    C -- 1. 发起 HTTP GET 请求 --> D[您的 Azure Function (在云上运行)];
    D -- 2. 执行 Java 代码 --> E{连接 YugabyteDB};
    E -- 3. 执行 SQL 查询 --> F[YugabyteDB 数据库];
    F -- 4. 返回查询结果 --> E;
    E -- 5. 封装结果 --> D;
    D -- 6. 将结果作为 JSON 返回 --> C;
    C -- 7. 将返回的 JSON 存为流程变量 --> G[流程继续...];

```
graph TD
    A[Flowable 流程启动] --> B{BPMN 模型};
    B --> C[服务任务: "查询员工信息"];
    C -- 1. 发起 HTTP GET 请求 --> D[您的 Azure Function (在云上运行)];
    D -- 2. 执行 Java 代码 --> E{连接 YugabyteDB};
    E -- 3. 执行 SQL 查询 --> F[YugabyteDB 数据库];
    F -- 4. 返回查询结果 --> E;
    E -- 5. 封装结果 --> D;
    D -- 6. 将结果作为 JSON 返回 --> C;
    C -- 7. 将返回的 JSON 存为流程变量 --> G[流程继续...];
```


```
mvn clean package -DskipTests

func host start
```