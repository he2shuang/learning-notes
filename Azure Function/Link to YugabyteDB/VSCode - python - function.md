

#### 您需要准备的工具：

1. **Visual Studio Code (VS Code)**
2. **Azure Functions 扩展** (在 VS Code 中安装)
3. **Python 3.8+**
4. **YugabyteDB Docker 容器** (确保它正在运行)

**注意**：当您在 VS Code 中创建第一个 Azure Function 项目时，Azure Functions 扩展会自动检测并提示您安装一个叫做 **Azure Functions Core Tools** 的东西。这是一个命令行工具，也是实现本地运行的核心模拟器。请务必同意安装它。

**Azure Functions Core Tools** ：[Azure/azure-functions-core-tools: Command line tools for Azure Functions](https://github.com/Azure/azure-functions-core-tools?tab=readme-ov-file#windows)
![[Pasted image 20251121141618.png]]



![[Pasted image 20251121141717.png]]![[Pasted image 20251121141757.png]]

```
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "YUGABYTE_CONNECTION_STRING": "postgresql://yugabyte:yugabyte@localhost:5433/my_first_db"
  }
}
```

![[Pasted image 20251121142020.png]]

![[Pasted image 20251121141827.png]]

```
import azure.functions as func
import logging
import os
import json
import psycopg2

# 创建一个 FunctionApp 实例
app = func.FunctionApp()

# 使用装饰器来定义一个 HTTP 触发器函数
# route="GetEmployees" 定义了 URL 路径的一部分
# auth_level=func.AuthLevel.ANONYMOUS 允许匿名访问
@app.route(route="GetEmployees", auth_level=func.AuthLevel.ANONYMOUS)
def GetEmployees(req: func.HttpRequest) -> func.HttpResponse:
    """
    这个函数连接到 YugabyteDB，查询 employees 表，并返回 JSON 结果。
    """
    logging.info('Python HTTP trigger function processed a request.')

    try:
        # 1. 从环境变量中获取连接字符串 (从 local.settings.json 读取)
        conn_string = os.environ.get("YUGABYTE_CONNECTION_STRING")
        if not conn_string:
            return func.HttpResponse("Database connection string is not set.", status_code=500)

        # 2. 连接到 YugabyteDB
        conn = psycopg2.connect(conn_string)
        cursor = conn.cursor()
        logging.info("Successfully connected to the database.")

        # 3. 执行查询
        cursor.execute("SELECT id, name, department, salary FROM employees;")
        
        # 获取查询结果的列名
        colnames = [desc[0] for desc in cursor.description]
        
        # 将查询结果转换为字典列表
        rows = cursor.fetchall()
        results = []
        for row in rows:
            results.append(dict(zip(colnames, row)))

        # 4. 关闭连接
        cursor.close()
        conn.close()

        # 5. 返回 JSON 格式的结果
        return func.HttpResponse(
            json.dumps(results, default=str),  # default=str 用于处理 Decimal 等特殊类型
            mimetype="application/json",
            status_code=200
        )

    except psycopg2.Error as e:
        logging.error(f"Database error: {e}")
        return func.HttpResponse(f"Database connection error: {e}", status_code=500)
    except Exception as e:
        logging.error(f"An unexpected error occurred: {e}")
        return func.HttpResponse(f"An error occurred: {e}", status_code=500)

```

![[Pasted image 20251121141901.png]]
遇到如下问题，点击“Use Local Emulator”
![[Pasted image 20251121142240.png]]![[Pasted image 20251121142312.png]]

![[Pasted image 20251121142323.png]]

遇到这个问题，点击“Install Azurite”
![[Pasted image 20251121142330.png]]![[Pasted image 20251121142350.png]]

![[Pasted image 20251121142411.png]]

```
.venv\Scripts\python -m pip install -r requirements.txt 
.venv\Scripts\activate ; func host start 
```