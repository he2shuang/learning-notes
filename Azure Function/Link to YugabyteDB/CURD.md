

# 本地的执行计划

==query==
URL：`http://<IP>:<port>/{table}?{key}={<Operators.value>}`
Method: GET

==insert==
URL：`http://<IP>:<port>/{table}`
Method: POST
Body: `{"col1":"value1", "col2":"value2", ...}`

==update==
URL：`http://<IP>:<port>/{table}?{key}={<Operators.value>}`
Method: PATCH
Body: `{"col1":"value1", "col2":"value2", ...}`

==delete==
URL：`http://<IP>:<port>/{table}?{key}={<Operators.value>}`
Method: DELETE


```
-- GET
http://localhost:7071/api/notification_history

-- GET 
http://localhost:7071/api/notification_history?status=Unfinished

-- POST
http://localhost:7071/api/notification_history
 -- Head
 "Content-Type: application/json"
 -- Body
 {"request_result": {"url": "api.test.com"}, "apikey": "some-api-key","status": "Unfinished"}
 

-- PATCH
http://localhost:7071/api/notification_history?notification_id=<YOUR_UUID_HERE>
 -- Head
 "Content-Type: application/json"
 -- Body
 {"status": "Finished", "confirmation_result": {"code": 200}}
 
 
-- DELETE
http://localhost:7071/api/notification_history?notification_id=<YOUR_UUID_HERE>

```



```
-- GET
http://localhost:7071/api/event_data

-- GET 
http://localhost:7071/api/event_data?status=Unfinished


-- POST
http://localhost:7071/api/event_data
 -- Head
 "Content-Type: application/json"
 -- Body
{"dealer_code": "DLR-996","apikey": "some-api-key","information": {"action": "logout"},"employee_code": "EMP-ABC"}


-- PATCH
http://localhost:7071/api/event_data?event_id=<YOUR_UUID_HERE>
 -- Head
 "Content-Type: application/json"
 -- Body
 {"management_information": {"user": "admin", "source": "test"},"information": {"action": "test"}}
 
 
-- DELETE
http://localhost:7071/api/event_data?event_id=<YOUR_UUID_HERE>

```

```json

{
	"dealer_code": "DLR-003",
	"employee_code": "EMP-67890",
	"apikey": "pqr-stu-789-012",
	"information": {"customer_id": "CUST-654", "action": "purchase", "item_id": "ITEM-A"},
	
}
```
