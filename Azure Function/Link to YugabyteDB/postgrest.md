
_PostgREST_是一个开源的工具，用于**将_PostgreSQL_数据库自动转换为_REST_ _API_**。 它是一个独立的_Web_服务器，可以根据数据库结构自动生成符合_RESTful_ _API_规范的_API_，而无需手动编写代码或进行配置。 _PostgREST_基于_PostgreSQL_数据库的表和关系自动生成_RESTful_ _API_，并遵循_RESTful_ _API_的规范，包括使用_HTTP_方法（_GET_、_POST_、_PUT_、_DELETE_等）、基于资源的_URL_和标准的_HTTP_响应代码等。


[表和视图 — PostgREST 12.2 文档 - PostgREST 中文](https://postgrest.postgresql.ac.cn/en/v12/references/api/tables_views.html#read)

## HTTP PUT 和 PATCH 方法的区别
[1](https://www.jianshu.com/p/bee85cf4e33a)[2](https://blog.csdn.net/varyall/article/details/80895945)[3](https://segmentfault.com/q/1010000005685904)


在HTTP协议中，PUT和PATCH都是用于更新资源的方法，但它们有一些关键的区别。

PUT 方法

**PUT** 方法用于更新资源，并且要求前端提供一个完整的资源对象。如果使用PUT方法更新资源，但没有提供完整的资源对象，那么缺少的字段应该被清空。例如，如果我们有一个UserInfo对象，其中包含userId、userName、userGender等10个字段，使用PUT方法更新时，需要提供包含所有字段的完整UserInfo对象。

PUT方法是幂等的，这意味着多次执行相同的请求对系统的影响与执行一次是相同的。例如，使用PUT方法更新一个用户信息，无论执行多少次，结果都是相同的。

```
# 使用PUT方法更新用户信息
import requests
  
url = 'http://example.com/api/user/123'
data = {
'userId': 123,
'userName': 'John Doe',
'userGender': 'Male'
}
  
response = requests.put(url, json=data)
print(response.status_code)
```



PATCH 方法

**PATCH** 方法用于对已知资源进行局部更新。这意味着只需要提供需要更新的字段，而不需要提供完整的资源对象。例如，如果我们只需要更新UserInfo对象中的userName字段，可以使用PATCH方法只传递userName字段。

PATCH方法不是幂等的，这意味着多次执行相同的请求可能会对系统产生不同的影响。例如，使用PATCH方法更新用户信息中的userName字段，每次请求都会更新该字段。


```
# 使用PATCH方法局部更新用户信息
import requests
  
url = 'http://example.com/api/user/123'
data = {
'userName': 'Jane Doe'
}
  
response = requests.patch(url, json=data)
print(response.status_code)
```


重要考虑事项

- **幂等性**：PUT方法是幂等的，而PATCH方法不是幂等的。
    
- **带宽使用**：使用PATCH方法可以减少带宽使用，因为只需要传递需要更新的字段。
    
- **安全性**：使用PATCH方法可以减少暴露不需要修改的字段，从而提高安全性。
    
- **幂等性**：PUT方法是幂等的，而PATCH方法不是幂等的[2](https://blog.csdn.net/varyall/article/details/80895945)[3](https://segmentfault.com/q/1010000005685904)。
    
- **带宽使用**：使用PATCH方法可以减少带宽使用，因为只需要传递需要更新的字段[1](https://www.jianshu.com/p/bee85cf4e33a)[2](https://blog.csdn.net/varyall/article/details/80895945)。
    
- **安全性**：使用PATCH方法可以减少暴露不需要修改的字段，从而提高安全性[3](https://segmentfault.com/q/1010000005685904)。
    

总之，PUT方法适用于需要更新完整资源的情况，而PATCH方法适用于只需要局部更新资源的情况。根据具体需求选择合适的方法，可以提高效率和安全性。
