# C# Web API 参数传递详解（总结与勘误版）

> 原文：《C#进阶系列------WebApi 接口参数不再困惑：传参详解》\
> 原文链接：https://www.cnblogs.com/landeanfen/p/5337072.html\
> 本文定位：保留原文中有价值的参数绑定思路，并修正过时、过度概括或容易误导的表述。\
> 重要背景：原文发表于 2016 年，示例主要针对**传统 ASP.NET Web API
> 2**。本文会明确区分 Web API 2 与 ASP.NET
> Core，不应将所有规则不加区分地套用到不同框架版本。

------------------------------------------------------------------------

## 一、核心结论先看

1.  **参数从哪里来，取决于 HTTP 请求的实际格式以及框架的参数绑定规则。**
    不能只看前端传入的是"对象"还是"字符串"。
2.  **GET 通常通过 URL 查询参数传递数据**，例如
    `?id=1&name=Jim`。复杂对象可以由多个查询参数绑定，但一般不应把大型对象或敏感数据放进
    URL。
3.  **POST/PUT 的参数可以来自请求体，也可以来自 URL
    或路由**。具体取决于框架、参数类型、绑定特性和请求的
    `Content-Type`。
4.  发送 JSON 请求体时，前端通常使用 `JSON.stringify(data)`，并设置
    `Content-Type: application/json`；后端应使用与框架相匹配的 DTO /
    模型接收。
5.  在传统 ASP.NET Web API 2
    中，通常只有一个参数可以由请求体读取。若需要传多个字段，推荐定义一个请求
    DTO，而不是默认使用 `dynamic`。
6.  明确写出 `[HttpGet]`、`[HttpPost]`、`[HttpPut]`、`[HttpDelete]`
    有利于路由与维护；不要依赖方法名以 `Get` 开头来隐式决定 HTTP 方法。
7.  `GET`、`POST`、`PUT`、`DELETE`
    的语义不同，但**参数绑定并不简单等于"GET 用 URL，其他方法全用
    Body"**。
8.  请求体大小、URL 长度限制并非统一固定为 1024
    字节；实际限制取决于浏览器、服务器、代理、网关和应用配置。

------------------------------------------------------------------------

## 二、先区分几个概念

  -----------------------------------------------------------------------------------------
  概念                    含义                             示例
  ----------------------- -------------------------------- --------------------------------
  URL 路由参数            路径中的变量                     `/api/products/12` 中的 `12`

  查询字符串              URL `?` 后的键值对               `/api/products?id=12&name=pen`

  请求体（Body）          HTTP 请求携带的数据内容          JSON、表单数据、纯文本等

  `Content-Type`          声明请求体采用什么媒体类型       `application/json`

  `Accept`                客户端希望接收的响应媒体类型     `application/json`

  JSON 文本               一种数据交换格式                 `{"name":"Jim","age":25}`

  JavaScript 对象         浏览器 JavaScript 运行时中的对象 `{ name: "Jim", age: 25 }`

  模型绑定 / 反序列化     框架把请求数据转换为方法参数或   JSON Body → `CreateUserRequest`
                          DTO 的过程                       
  -----------------------------------------------------------------------------------------

### `Content-Type` 与 `JSON.stringify()` 不是一回事

``` javascript
const data = { name: "Jim", age: 25 };

// 把 JS 对象序列化成 JSON 字符串
const body = JSON.stringify(data);

// body 是字符串；Content-Type 是请求头，告诉服务器如何解释请求体
```

-   `JSON.stringify(data)`：负责把 JavaScript 值转换成 JSON 文本。
-   `Content-Type: application/json`：声明请求体是 JSON。
-   后端模型绑定 / JSON 输入格式化器：负责把 JSON 转成 C# 参数对象。

只设置 `Content-Type: application/json`，并不会自动把一个 JavaScript
对象变成正确的 JSON 请求体。

------------------------------------------------------------------------

## 三、GET 请求如何传参数

### 1. 基础类型：查询字符串

前端：

``` javascript
$.ajax({
    type: "GET",
    url: "/api/charging",
    data: { id: 1, name: "Jim" }
});
```

通常会生成类似请求：

``` http
GET /api/charging?id=1&name=Jim
```

传统 ASP.NET Web API 2 示例：

``` csharp
[HttpGet]
public IHttpActionResult Get(int id, string name)
{
    return Ok(new { id, name });
}
```

在 ASP.NET Core 中，通常可以写成：

``` csharp
[HttpGet]
public IActionResult Get([FromQuery] int id, [FromQuery] string name)
{
    return Ok(new { id, name });
}
```

如果参数名与查询字符串键名匹配，框架就可以进行绑定。实际项目应处理缺失值、格式错误和验证失败的情况。

### 2. 复杂对象：使用查询参数绑定

传统 ASP.NET Web API 2 可以使用 `[FromUri]`：

``` csharp
public class ChargingQuery
{
    public string? Id { get; set; }
    public string? Name { get; set; }
}

[HttpGet]
public IHttpActionResult Find([FromUri] ChargingQuery query)
{
    return Ok(query);
}
```

请求：

``` http
GET /api/charging/find?id=1&name=Jim
```

ASP.NET Core 对应的常见写法是 `[FromQuery]`：

``` csharp
[HttpGet("find")]
public IActionResult Find([FromQuery] ChargingQuery query)
{
    return Ok(query);
}
```

**勘误：** 不能笼统地说 GET 不能传复杂对象。更准确的说法是：GET
请求通常通过 URL 查询参数传值，框架可以把多个查询参数绑定到复杂类型；但
GET 请求一般不使用请求体来传递查询对象。

### 3. 不推荐把整个 JSON 对象塞进 GET 查询字符串

例如：

``` javascript
const query = { id: 1, name: "Jim" };

$.ajax({
    type: "GET",
    url: "/api/charging/find",
    data: { query: JSON.stringify(query) }
});
```

这会把 JSON 文本作为查询参数传递，而不是作为 JSON 请求体。后端需要接收
`query` 字符串并自行反序列化。

这种方案并非绝对不能用，但通常不如使用清晰的查询参数或专门的查询 DTO
直观。查询字符串还可能出现在浏览器历史、服务器日志、代理日志等位置，不适合放敏感信息。

### 4. GET 长度限制不是固定 1024 字节

原文把 GET 参数限制概括为"最大 1024 字节"，这个说法不准确。

-   HTTP 标准没有规定一个适用于所有环境的统一 1024 字节上限。
-   浏览器、Web 服务器、反向代理、负载均衡器和网关可能各自设置 URL
    长度限制。
-   超限时可能返回错误，也可能被中间设备拒绝；不能依赖"超出后一定会丢失参数"。
-   大量筛选条件或大型数据集不适合放在 URL 中。可以考虑 POST
    查询接口或重新设计请求参数。

------------------------------------------------------------------------

## 四、POST 请求如何传参数

POST 的参数来源不只有一种。最常见的有：JSON
请求体、表单编码请求体、路由参数和查询参数。

### 1. JSON 请求体：推荐用于结构化数据

前端：

``` javascript
const request = {
    name: "Jim",
    description: "备注"
};

$.ajax({
    type: "POST",
    url: "/api/charging",
    contentType: "application/json; charset=utf-8",
    data: JSON.stringify(request),
    success: function (result) {
        console.log(result);
    }
});
```

传统 ASP.NET Web API 2：

``` csharp
public class SaveChargingRequest
{
    public string? Name { get; set; }
    public string? Description { get; set; }
}

[HttpPost]
public IHttpActionResult Save([FromBody] SaveChargingRequest request)
{
    return Ok(new { request.Name, request.Description });
}
```

ASP.NET Core：

``` csharp
[ApiController]
[Route("api/[controller]")]
public class ChargingController : ControllerBase
{
    [HttpPost]
    public IActionResult Save([FromBody] SaveChargingRequest request)
    {
        return Ok(new { request.Name, request.Description });
    }
}
```

在 ASP.NET Core 中，`[ApiController]`
会启用一些约定，例如推断复杂类型参数的绑定来源以及自动处理模型验证错误；具体行为要结合项目配置和框架版本理解。

### 2. 表单编码请求体：与 JSON 不同

传统 HTML 表单和某些 AJAX 请求使用：

``` http
Content-Type: application/x-www-form-urlencoded
```

请求体形如：

``` text
name=Jim&description=%E5%A4%87%E6%B3%A8
```

jQuery 示例：

``` javascript
$.ajax({
    type: "POST",
    url: "/api/charging/form",
    data: {
        name: "Jim",
        description: "备注"
    }
});
```

在传统 ASP.NET Web API 2 中，这类数据通常由表单数据绑定机制读取；在
ASP.NET Core 中，可以根据需要使用 `[FromForm]`。

``` csharp
// ASP.NET Core 示例
[HttpPost("form")]
public IActionResult SaveForm([FromForm] SaveChargingRequest request)
{
    return Ok(request);
}
```

**勘误：** 原文中"实体对象不能指定
`application/json`"的说法不正确。可以使用
`application/json`，但必须让请求体与该媒体类型匹配，例如将对象序列化为
JSON 字符串，并由服务器按 JSON 方式读取。

### 3. 为什么传统 Web API 2 的 `[FromBody] string` 看起来很奇怪？

在传统 ASP.NET Web API 2 中：

``` csharp
[HttpPost]
public IHttpActionResult Save([FromBody] string name)
{
    return Ok(name);
}
```

`[FromBody]`
会让框架尝试通过请求体格式化器读取内容。它不是在请求体中寻找名为 `name`
的键。若使用 JSON 格式发送字符串，合法 JSON 值应类似：

``` json
"Jim"
```

而不是：

``` json
{"name":"Jim"}
```

后者是 JSON 对象，不是 JSON
字符串值。具体支持的格式和绑定行为取决于格式化器及框架版本。

如果需要传多个字段，不要把多个 `[FromBody]` 参数作为方案。在传统 ASP.NET
Web API 2
中，请求体通常只能由一个参数读取，因为请求体流不能简单地被多个参数各自独立消费。

**推荐做法：用一个 DTO 接收多个字段。**

``` csharp
public class SaveChargingRequest
{
    public string? Name { get; set; }
    public string? Description { get; set; }
}

[HttpPost]
public IHttpActionResult Save([FromBody] SaveChargingRequest request)
{
    return Ok(request);
}
```

------------------------------------------------------------------------

## 五、为什么不建议把 `dynamic` 当作通用推荐方案

原文推荐用 `dynamic`
接收多个基础类型，主要是为了减少参数类数量。它在某些动态 JSON
场景下确实有用，但不适合被概括为"所有基础类型参数都推荐使用 `dynamic`"。

### DTO 的优势

-   **类型明确**：参数类型、字段和约束一目了然。
-   **编译期检查**：拼写错误和类型问题更容易被发现。
-   **验证更方便**：可以使用数据注解或自定义验证。
-   **重构更安全**：IDE 可以追踪属性和引用。
-   **接口文档更清楚**：Swagger / OpenAPI 更容易生成准确的请求结构。
-   **维护成本更低**：调用方和服务端有明确的数据契约。

### `dynamic` 的适用场景

-   JSON 结构确实动态且无法预先定义。
-   编写临时数据处理工具。
-   与某些动态 API 或兼容性场景交互。

如果 JSON 结构已知，优先使用强类型
DTO。不要仅仅为了少写一个类就牺牲接口的可维护性。

------------------------------------------------------------------------

## 六、数组与集合参数

如果需要传递 ID 数组或实体集合，JSON 请求体通常清晰易用。

### 1. 基础类型数组

前端：

``` javascript
const ids = [1, 2, 3, 4];

$.ajax({
    type: "POST",
    url: "/api/charging/batch",
    contentType: "application/json; charset=utf-8",
    data: JSON.stringify(ids)
});
```

传统 ASP.NET Web API 2 示例：

``` csharp
[HttpPost]
public IHttpActionResult Batch([FromBody] int[] ids)
{
    return Ok(new { Count = ids.Length });
}
```

### 2. 实体集合

前端：

``` javascript
const items = [
    { id: "1", name: "Jim" },
    { id: "2", name: "Lilei" }
];

$.ajax({
    type: "POST",
    url: "/api/charging/batch-save",
    contentType: "application/json; charset=utf-8",
    data: JSON.stringify(items)
});
```

传统 ASP.NET Web API 2 示例：

``` csharp
[HttpPost]
public IHttpActionResult BatchSave([FromBody] List<ChargingItem> items)
{
    return Ok(new { Count = items.Count });
}
```

如果批量操作还需要额外选项，可以把数组放进 DTO：

``` json
{
  "dryRun": false,
  "items": [
    { "id": "1", "name": "Jim" },
    { "id": "2", "name": "Lilei" }
  ]
}
```

``` csharp
public class BatchSaveRequest
{
    public bool DryRun { get; set; }
    public List<ChargingItem> Items { get; set; } = new();
}
```

这通常比同时传递多个互不相关的参数更容易维护。

------------------------------------------------------------------------

## 七、PUT 与 DELETE

### PUT

PUT 常用于替换资源或按接口约定更新资源，但具体语义取决于 API
设计。它的参数绑定原则与 POST
类似：参数可以来自路由、查询字符串或请求体。

``` javascript
$.ajax({
    type: "PUT",
    url: "/api/charging/1",
    contentType: "application/json; charset=utf-8",
    data: JSON.stringify({
        name: "Jim",
        description: "更新后的描述"
    })
});
```

ASP.NET Core 示例：

``` csharp
[HttpPut("{id:int}")]
public IActionResult Update(int id, [FromBody] UpdateChargingRequest request)
{
    // 根据 id 查找并更新资源
    return NoContent();
}
```

### DELETE

DELETE 通常用于删除资源。资源标识一般放在 URL 路由中：

``` http
DELETE /api/charging/1
```

ASP.NET Core 示例：

``` csharp
[HttpDelete("{id:int}")]
public IActionResult Delete(int id)
{
    // 删除对应资源
    return NoContent();
}
```

虽然某些框架和客户端可以发送带请求体的 DELETE 请求，但各个中间件、代理和
API 约定对其支持可能不一致。若只是删除单个资源，优先把资源 ID
放在路由中；批量删除时应设计明确、文档化的接口，并确认整条链路支持所选方案。

------------------------------------------------------------------------

## 八、HTTP 方法与 Action 选择

原文观察到，传统 Web API 可能根据方法名是否以 `Get` 开头推断 HTTP
方法。但不要依赖这种隐式行为。

推荐显式标注：

``` csharp
[HttpGet]
public IHttpActionResult Find()
{
    return Ok();
}

[HttpPost]
public IHttpActionResult Create()
{
    return Ok();
}

[HttpPut]
public IHttpActionResult Update()
{
    return Ok();
}

[HttpDelete]
public IHttpActionResult Delete()
{
    return Ok();
}
```

这是传统 ASP.NET Web API 2 风格。ASP.NET Core 常使用
`[HttpGet]`、`[HttpPost]` 等特性，并可结合 `[Route]` 或属性路由模板：

``` csharp
[HttpGet("{id:int}")]
public IActionResult GetById(int id)
{
    return Ok();
}
```

**勘误：** HTTP
动词与方法名的约定、路由匹配规则在不同框架版本中可能有差异。显式声明
HTTP 方法和路由，可以减少歧义，但也仍需确保路由模板、控制器路由和请求
URL 匹配。

------------------------------------------------------------------------

## 九、从 C# 后台发送 HTTP 请求

原文用 `WebRequest` 发送请求。它可以帮助理解 HTTP 请求的组成，但在现代
.NET 项目中，通常优先使用 `HttpClient`。

``` csharp
using System.Net.Http;
using System.Net.Http.Json;

var client = new HttpClient();

var request = new SaveChargingRequest
{
    Name = "Jim",
    Description = "备注"
};

using var response = await client.PostAsJsonAsync(
    "https://example.com/api/charging",
    request);

response.EnsureSuccessStatusCode();

var result = await response.Content.ReadAsStringAsync();
Console.WriteLine(result);
```

生产项目中，ASP.NET Core 通常通过 `IHttpClientFactory` 管理和配置
`HttpClient`，避免在业务代码中随意反复创建客户端。上面代码是用于说明参数发送方式的简化示例。

如果手动拼接 JSON 字符串，必须保证它是合法 JSON。比如原文中类似：

``` csharp
string postData = "{ ID: \"1\", NAME: \"Jim\" }";
```

这**不是合法 JSON**，因为 JSON
属性名必须使用双引号。应使用序列化器生成请求体，而不是手工拼字符串：

``` csharp
var json = System.Text.Json.JsonSerializer.Serialize(request);
```

------------------------------------------------------------------------

## 十、常见误区与修正

  ----------------------------------------------------------------------------------------------------------------------
  容易误解的说法                      更准确的理解
  ----------------------------------- ----------------------------------------------------------------------------------
  GET 最多只能传 1024 字节            没有适用于所有环境的统一 1024 字节限制；限制由浏览器、服务器、代理和配置共同决定

  GET 不能传复杂对象                  可以把多个查询参数绑定到复杂对象；一般不通过 GET 请求体传递查询对象

  POST 的参数都必须放在 Body          POST 参数也可以来自路由或查询字符串；要看接口设计与绑定来源

  `application/json` 不能接收实体对象 可以；请求体应是合法 JSON，后端需使用合适的 JSON 反序列化 / 模型绑定

  所有基础类型都推荐用 `dynamic`      已知结构优先使用强类型 DTO，类型安全、验证和文档都更好

  一个请求可以给多个参数分别加        传统 ASP.NET Web API 2 中通常只能有一个请求体参数；多个字段应封装成 DTO
  `[FromBody]`                        

  `Content-Type` 设置为 JSON 就够了   还要将请求数据序列化成 JSON 文本，且后端必须支持该媒体类型

  PUT 一定是更新，POST 一定是新增     这是常见设计约定，不是对所有 API 的绝对限制；应遵循 HTTP 语义和接口契约

  DELETE 的请求体一定可用             不应假设所有客户端、中间件和代理都一致支持；优先使用路由标识，批量操作需验证链路

  省略 HTTP 方法特性也总能正确路由    不同框架有不同的 Action 选择约定；建议显式声明 HTTP 方法与路由
  ----------------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 十一、实际项目的推荐选择

  -------------------------------------------------------------------------------------------
  业务场景                            推荐方式
  ----------------------------------- -------------------------------------------------------
  根据 ID 查询单条数据                `GET /api/products/123`，ID 放路由

  简单筛选和分页                      GET 查询参数，例如 `?page=1&pageSize=20&keyword=pen`

  复杂查询条件                        小型、适合 URL 的条件用查询 DTO；复杂或大型条件可设计
                                      POST 查询接口

  创建资源                            POST + JSON DTO

  更新资源                            PUT 或 PATCH + JSON DTO，按接口语义选择

  删除单个资源                        DELETE + 路由 ID

  批量新增或修改                      POST/PUT + JSON 集合或批量操作 DTO

  多个基础字段一起提交                定义请求 DTO，而不是多个 `[FromBody]` 或默认使用
                                      `dynamic`

  上传文件                            通常使用
                                      `multipart/form-data`，并按框架要求绑定文件和其他字段

  与第三方 API 通信                   使用 `HttpClient`，通过 JSON 序列化器构造请求体
  -------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 十二、排查"后端接收不到参数"的检查清单

遇到参数为 `null`、默认值或模型绑定失败时，按顺序检查：

1.  **请求方法**：实际发送的是 GET、POST、PUT 还是 DELETE？
2.  **URL**：路由、路径参数、查询参数是否与接口一致？
3.  **请求体**：浏览器开发者工具的 Network 中，实际 Body 是什么？
4.  **`Content-Type`**：它与请求体格式是否匹配？
5.  **是否序列化**：发送 JSON 时是否使用 `JSON.stringify()`
    或等价序列化方式？
6.  **绑定来源**：传统 Web API 2 的 `[FromUri]` / `[FromBody]`，或
    ASP.NET Core 的 `[FromRoute]` / `[FromQuery]` / `[FromBody]` /
    `[FromForm]` 是否使用正确？
7.  **DTO 属性**：字段名、类型、可写性和嵌套结构是否匹配？
8.  **日期和数字格式**：格式是否能被服务器正确解析？是否受区域性设置影响？
9.  **模型验证**：是否存在必填字段缺失或类型转换失败？
10. **框架版本**：当前项目是 ASP.NET Web API 2、ASP.NET Core MVC，还是
    Minimal API？不要直接复制另一种框架的示例。

------------------------------------------------------------------------

## 十三、知识要点速记

-   GET 查询参数通常位于 URL 中，适合读取、筛选和分页。
-   JSON 请求体应使用合法 JSON 文本，并设置正确的 `Content-Type`。
-   `Content-Type` 声明格式，不负责把对象自动序列化。
-   传统 ASP.NET Web API 2 与 ASP.NET Core
    的绑定特性和默认行为不完全相同。
-   多个相关字段通常应该封装为一个 DTO。
-   已知数据结构优先使用强类型模型；`dynamic`
    留给确实需要动态结构的场景。
-   显式声明 HTTP 方法和路由，避免依赖方法名约定。
-   URL 长度限制没有统一的 1024 字节标准。
-   对复杂 API，优先以明确的请求契约、模型验证、清晰的 HTTP
    语义和可测试性来设计接口。

## 参考资料

-   原文：<https://www.cnblogs.com/landeanfen/p/5337072.html>
-   ASP.NET Web API 2 参数绑定（Microsoft
    Learn）：<https://learn.microsoft.com/en-us/aspnet/web-api/overview/formats-and-model-binding/parameter-binding-in-aspnet-web-api>
-   ASP.NET Core 模型绑定（Microsoft
    Learn）：<https://learn.microsoft.com/en-us/aspnet/core/mvc/models/model-binding>
-   ASP.NET Core Web
    API：<https://learn.microsoft.com/en-us/aspnet/core/web-api/>
-   `HttpClient` 概述（Microsoft
    Learn）：<https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines>

> 本笔记是对原文知识点的整理与技术勘误，不是原文逐字转载。示例应根据项目实际使用的
> .NET / ASP.NET 框架版本进行验证。
