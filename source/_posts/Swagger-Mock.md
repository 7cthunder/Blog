---
title: Swagger & Mock
date: 2019-06-23 23:49:33
tags: 
  - Mock
  - Swagger
categories:
  - Swagger
---
## 简介
之前在开发数据库课程项目的时候，作为 Leader，一直在想怎么解决前后端开发同时进行的问题，这学期知道了Swagger 这个东西，有了它我们的开发流程是这样的：
```
- 协商API
    - 前端开发
        - 编写前端页面
        - 通过 Mock Server 充当真服务端 
    - 服务端开发
        - ...
```
也就是说，前端在开发过程中，不需要等待服务端的小火鸡写完接口再进行测试，通过 Mock Server 即可返回想要的 Response 进行各种测试。


## Swagger
何为 Swagger，照我的理解，就是一个写API文档的框架，之前写项目基本都是手撸API文档，然后从零编写服务端和客户端代码，而有了它，我们只需要编写一个 yaml 文件，它就可以自动帮我们生成客户端 API 调用代码、服务端大致框架以及一个好用的API文档。

多说无益，上手玩玩才知道多棒！传送门：[Swagger Editer](http://editor.swagger.io/)

### 怎么用？
看看下面这个小例子吧（主要看有注释的那几行即可）
```yaml
paths:
  /pet:                                     # api path
    post:                                   # api method
      tags:
      - "pet"
      summary: "Add a new pet to the store"
      description: ""
      operationId: "addPet"                 # 这个名字会成为上面所说生成代码的调用接口
      consumes:                             # Request 数据格式为：
      - "application/json"                  # - json
      - "application/xml"                   # - xml
      produces:                             # Response 数据格式为：
      - "application/xml"                   # - xml
      - "application/json"                  # - json
      parameters:                           # Request 参数
      - in: "body"                          # 参数位于 body 中 (因为 Method 为 POST)
        name: "body"
        description: "Pet object that needs to be added to the store"
        required: true                      # Request 必须有此参数
        schema:
          $ref: "#/definitions/Pet"         # 参数的样式，这里为 Pet
      responses:                            # Response 返回样例
        405:                                # Response status
          description: "Invalid input"

```
上面编写的 yaml 文件就帮我们描述了 POST /pet 这个添加宠物的接口啦！

### 生成代码
我们可以分别生成 Server 和 Client 代码，这里分别选择 go-server 和 JavaScript

先看看前端 JavaScript 代码，实际上，如果让我们手撸一个这样的接口，我们并不会写得这么健全，如果我们需要发送一个POST请求，那么我们可能只会写 body 这部分的参数，而不会像下面所示代码那样，面面俱到，也许是有些许冗余了，但是机器生成的代码，还要什么自行车呢？
```javascript
/**
 * Add a new pet to the store
 * 
 * @param {module:model/Pet} body Pet object that needs to be added to the store
 * @param {module:api/PetApi~addPetCallback} callback The callback function, accepting three arguments: error, data, response
 */
this.addPet = function(body, callback) {
  var postBody = body;

  // verify the required parameter 'body' is set
  if (body === undefined || body === null) {
    throw new Error("Missing the required parameter 'body' when calling addPet");
  }

  var pathParams = {
  };
  var queryParams = {
  };
  var collectionQueryParams = {
  };
  var headerParams = {
  };
  var formParams = {
  };

  var authNames = ['petstore_auth'];
  var contentTypes = ['application/json', 'application/xml'];
  var accepts = ['application/xml', 'application/json'];
  var returnType = null;

  return this.apiClient.callApi(
    '/pet', 'POST',
    pathParams, queryParams, collectionQueryParams, headerParams, formParams, postBody,
    authNames, contentTypes, accepts, returnType, callback
  );
}
```

再看看 go-server 代码，其只是一个简单的模板，帮我们写好了 Response 的头，以及帮我们定义了 Pet 这个结构体，需要注意这个需要我们在 yaml 中定义。服务端需要做的，就是搞搞数据库，填填逻辑代码，有了这么一个框架，工作量真的少了！

```go
// pet_api.go
func AddPet(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json; charset=UTF-8")
    w.WriteHeader(http.StatusOK)
}

// pet.go
type Pet struct {

    Id int64 `json:"id,omitempty"`

    Category *Category `json:"category,omitempty"`

    Name string `json:"name"`

    PhotoUrls []string `json:"photoUrls"`

    Tags []Tag `json:"tags,omitempty"`

    // pet status in the store
    Status string `json:"status,omitempty"`
}
```

## Mock
mock 是在测试过程中，对于一些不容易构造/获取的对象，创建一个mock对象来模拟对象的行为，这里我们不深究，我们选 Mockjs 来玩玩看。

Mock.js 是一款模拟数据生成器，旨在帮助前端工程师独立于后端进行开发，帮助编写单元测试。提供了以下模拟功能：
* 根据数据模板生成模拟数据
* 模拟 Ajax 请求，生成并返回模拟数据
* 基于 HTML 模板生成模拟数据

直接 [传送](http://mockjs.com/examples.html)，打开控制台就可以照着示例玩耍了：
```javascript
// 例如我想创建一个博客列表，每个条目包含id、评分
Mock.mock({
  'posts|5': [{
    'id|+1': 1,
    'stars|1-10': '*'
  }]
})
// 下面为上述代码所生成
{
  "posts": [
    {
      "id": 1,
      "stars": "*"
    },
    {
      "id": 2,
      "stars": "*"
    },
    {
      "id": 3,
      "stars": "**"
    },
    {
      "id": 4,
      "stars": "*********"
    },
    {
      "id": 5,
      "stars": "*"
    }
  ]
}
```

## Swagger-Mock
当两者结合起来的威力有多大呢，试想一下，一个可以帮你搭建好服务端框架，一个可以帮你任意生成数据，那么合起来，好像就能弄出一个能 work 的 Server 了不是吗！

### swagger-node
作为一个前端开发人员，node 当然是最亲的啦！swagger支持多种语言，所以我们选择 [swagger-node](https://github.com/swagger-api/swagger-node) 作为伪服务端不是很棒吗。

#### 安装
```bash
npm install -g swagger
```

#### 创建一个新项目
```bash
swagger project create hello-world
```

看看项目的结构：
```
.
├── README.md
├── api
│   ├── assets
│   ├── controllers
│   ├── helpers
│   ├── mocks
│   └── swagger
├── app.js
├── config
│   ├── README.md
│   └── default.yaml
├── package-lock.json
├── package.json
└── test
    └── api

9 directories, 6 files
```

#### 编辑你的API文档
这里与前面说的编写 yaml 基本一致，所编辑的 yaml 文件就在 ./api/swagger 中
```bash
swagger project edit
```

不过你需要干一件事，就是给你的 swagger-node 标记一下 controller 所在位置，即：
```yaml
paths:
  /hello:
      x-swagger-router-controller: hello_world  # 处理/hello的controller在hello_world.js
```

然后再看看 controller：
```javascript
// ./api/controllers/hello_world.js

/* 记住要将接口暴露出去 */
module.exports = { hello }; 

/*
  Functions in a127 controllers used for operations should take two parameters:

  Param 1: a handle to the request object
  Param 2: a handle to the response object
 */
function hello(req, res) {
  // variables defined in the Swagger document can be referenced using req.swagger.params.{parameter_name}
  var name = req.swagger.params.name.value || 'stranger';
  var hello = util.format('Hello, %s!', name);

  // this sends back a JSON response which is a single string
  res.json(hello);
}
```

## 总结
如果你还在手撸 API 文档，不如玩玩 Swagger 啦！

如果你还在从零构建服务端代码，不如让 Swagger 帮帮你啦！

如果你还在等服务端的小火鸡完成API开发，不如自己搭建一个 Swagger-Mock-Server 啦！