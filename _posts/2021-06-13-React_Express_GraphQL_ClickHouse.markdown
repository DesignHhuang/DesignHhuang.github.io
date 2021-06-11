## React-apollo + Express + GraphQL + Mysql 框架构建
首先确保系统中已经安装了node.js，以下都是基于nodejs环境。

### 1. 搭建react项目
使用命令创建一个react的简单项目，我建的项目的名称是react-graphql。
在这个项目中，会用到下面的几个库，在项目中执行命令安装这些库：

```
npm install apollo-boost react-apollo graphql-tag graphql
```
- apollo-boost：是设置apollo客户端需要用到的包，里面包含了各种设置，用于连接到Apollo后端。
- react-apollo：Apollo数据栈在React环境的集成包之一，用于执行GraphQL查询并将结果传给React的展示组件。
- graphql-tag：用于编写GraphQL查询。
- graphql： 解析GraphQL查询。

组件库使用Ant design
```
npm install antd
```
使用脚手架创建项目的时候，会自动启用React StrictMode，这边我手动将React.StrictMode去掉了，避免之后有些可能旧的弃用方法报错。

### 2. 写一个简单的Express GraphQL服务
我们需要创建一个GraphQL Endpoint，首先创建一个gql-server服务项目。

在这个服务中，也需要用到一些库：

```
npm install graphql express express-graphql --save
```
- express：Express 是一个保持最小规模的灵活的Node.js Web应用程序开发框架。
- express-graphql：GraphQL HTTP服务的中间件。

> 先创建一个比较基本的graphql服务。
在项目中新建一个server.js文件，将以下代码写到文件中。

```
var express = require('express');
var { graphqlHTTP } = require('express-graphql');
var { buildSchema } = require('graphql');

// GraphQL schema
var schema = buildSchema(`
    type Query {
        message: String
    }
`);

// Root resolver
var root = {
    message: () => '这是一个基于Express的GraphQL服务。'
};

// 创建Express服务和GraphQL endpoint
var app = express();
app.use('/graphql', graphqlHTTP({
    schema: schema,
    rootValue: root,
    graphiql: true
}));
app.listen(4000, () => console.log('Express GraphQL Server Now Running On localhost:4000/graphql'));
```
上述代码中我们先是建立了GraphQL schema，使用了buildSchema方法，这个schema是用来描述完整的API类型系统的。它包括完整的数据集，并定义客户端如何访问该数据。每次客户端进行API调用时，都会根据schema验证调用。只有验证成功，操作才会执行。否则返回错误。

接下来创建一个根解析器。解析程序包含操作到函数的映射。这个例子中写的比较简单，只是返回了一段字符串，在实际的项目中，往往不会这么简单。它可能包含多个操作，并且分配不同的解析器函数，这些函数中还包含了一些处理数据的逻辑。

最后，我们使用GraphQL endpoint：/graphql创建Express服务器。要创建GraphQL endpoint，首先要在app中存储一个新的express实例。接下来调用app.use方法并提供两个参数：

- 字符串类型的url
- graphqlHTTP函数结果，其中包含了三个参数

Express GraphQL中间件的三个配置参数：

- schema：就是我们在上面定义的schema，需要注册到这个endpoint中去。
- rootValue：也就是我们上面定义的根解析器对象。
- graphiql：是否开去GraphiQL工具，在浏览器中可以通过地址连接到这个endpoint，我们可以直接在页面中进行查询等操作。

最后我们可以使用以下代码运行服务：
```
node server.js
```
运行成功之后我们可以在浏览器中输入localhost:4000/graphql来进行测试查询：
```
{
    message
}
```
使用这样的查询就可以返回数据结果。之后我们会在实际应用中用到稍微复杂一点的查询。

> 问题：在数据的项目中，服务端与客户端之间可能会存在跨域的问题，在这里我使用的是cors库来解决，在项目中安装cors，然后在写服务的时候加上以下代码即可。
```
app.use(cors());
```

### 3. 搭建Mysql数据库
在系统中安装mysql数据库，新建数据库r_graphql，在数据库中新建表candidate，其中包含与schema中一样的字段。

依旧使用Expree来实现与mysql的连接，引入mysql模块，在项目中新建mysql.js测试连接数据库。在文件中加入以下代码：
```
npm install mysql
```
```
var express = require('express');   //引入express模块
var mysql = require('mysql');     //引入mysql模块
var app = express();        //创建express的实例

var connection = mysql.createConnection({      //创建mysql实例
    host: '127.0.0.1',
    port: '3306',
    user: 'root',
    password: 'huang',
    database: 'r_graphql'
});
connection.connect();
var sql = 'SELECT * FROM candidate';
var str = " ";
connection.query(sql, function (err, result) {
    if (err) {
        console.log('[SELECT ERROR]:', err.message);
    }
    str = JSON.stringify(result);
    //数据库查询的数据保存在result中，但浏览器并不能直接读取result中的结果，因此需要用JSON进行解析
    console.log(str);
});
app.get('/', function (req, res) {
    res.send(str);  //服务器响应请求
});
connection.end();
app.listen(3100, function () {
    // 监听3100端口
    console.log('Server running at 3100 port');
});
```
访问localhost:3100可以看到返回的结果。

### 4. 在GraphiQl中测试接口
我在GraphiQl测试了项目中需要用到的一些接口，如果需要了解语法的话，可以去官网看一下，Graphql还是有很多好玩的地方。
项目运行后可在localhost:4000中做测试。

### 5. 框架整合，实现候选人列表实例
#### 第一步：建立schema，在这里我们需要实现增删改查。
```
var schema = buildSchema(`
    type Query {
        getCandidate(id: Int!): Candidate
        getCandidates(gender: String): [Candidate]
    },
    type Mutation {
        createCandidate(input: CandidateInput!): Candidate
        updateCandidate(input: CandidateInput!, id: Int!): Candidate
    },
    input CandidateInput {
        id: Int
        name: String
        gender: String
        school: String
        mobile: String
        education: String
        workYears: Int
        description: String
        remarks: String
        age: Int
        isDeleted: Int
    }
    type Candidate {
        id: Int
        name: String
        gender: String
        school: String
        mobile: String
        education: String
        workYears: Int
        description: String
        remarks: String
        age: Int
        isDeleted: Int
    }
`);
```
这是我建的一个schema，里面包含了query和mutation，query负责查询数据，mutation用于新增和删除，这里的删除是假删除，所以我在定义candidate类型的时候加了isDeleted用于标记是否删除。

#### 第二步：连接数据库，并添加数据查询逻辑。
需要注意这边的数据库字段需要与我们建立的类型系统中的字段一致，以防查询解析的时候报错。

- 获取所有候选人的信息，这边都是异步操作，为了方便我直接返回了一个promise。注意这边我加了一个性别的判断，可以传入参数来实现按性别分组。

```
var getCandidates = function (args) {
    return new Promise((resolve, reject) => {
        pool.query('SELECT * from candidate where isDeleted = 0', (error, results, fields) => {
            if (error) throw error;
            if (args.gender) {
                var gender = args.gender;
                resolve(results.filter(candidate => candidate.gender === gender));
            } else {
                resolve(results);
            }
        });
    })
}
```
- 获取单个候选人数据，注意这边resolve的时候需要返回results的第一个值，因为query中返回的是一个数组，如果不加[0]的话，前端获取数据的时候会有问题。当然我们处理的时候也可以在前端处理，但是用graphiql测试的时候会报错，所以还是在服务端直接处理好了会比较方便。
```
var getById = function (id) {
    return new Promise((resolve, reject) => {
        pool.query('SELECT * from candidate where id = ?', id, (error, results, fields) => {
            if (error) throw error;
            resolve(results[0]);
        })
    })
}
```
- 新增候选人数据，这边可以注意到我用了async await，这边的场景是需要先获取新增后的主键id来获取新增的数据。
```
var createCandidate = function (data) {
    return new Promise((resolve, reject) => {
        pool.query('insert into candidate set ?', { ...data.input, isDeleted: 0 }, async (error, data) => {
            if (error) throw error;
            var res = await getById(data.insertId);
            resolve(res);
        });
    })
}
```
- 修改和删除候选人数据，修改和删除用的其实是同一个api，因为之前说过这边的删除是假删除，其实只是把isDeleted设置成1，删除只是特殊的修改。
```
var updateCandidate = function (arg) {
    return new Promise((resolve, reject) => {
        pool.query('update candidate set ? where id = ?', [arg.input, arg.id], async (error) => {
            if (error) throw error;
            var res = await getById(arg.id);
            resolve(res);
        });
    })
}
```
#### 第三步：编写前端页面并调用Api操作数据。
前端页面中我使用ant design的组件，页面部分使用了大量的react-apollo，一些我遇到的问题我会在文档中介绍一下，关于react-apollo的组件语法不熟悉的话也可以去学习一下。

首先我创建了一个Candidates组件用于显示候选人列表，为了方便，我直接把查询语句直接写在了这个文件中，我这边是比较少的，如果多的话还是建议拆出来：
```
var candidates = gql`{
    getCandidates {
        id
        age
        name
        gender
        mobile
        isDeleted
  }
}`

var createCandidateOpt = gql`
mutation createCandidate($input:CandidateInput!){
    createCandidate(input: $input){
        id
        name
        gender
        age
        mobile
        school
        education
        workYears
        description
        remarks
        isDeleted
    }
  }
`

var updateCandidateOpt = gql`
mutation updateCandidate($input:CandidateInput!,$id:Int!){
    updateCandidate(input: $input,id: $id){
        id
        name
        gender
        school
        mobile
        education
        workYears
        description
        remarks
        age
        isDeleted
    }
}`

var candidate = gql`
query getCandidate($id:Int!){
    getCandidate(id:$id){
        name
        gender
        age
        mobile
        school
        education
        workYears
        description
        remarks
        isDeleted
      }
  }
`
```
之后创建一个Ant design的表格：
```
<Query query={candidates}>
        {({ loading, error, data }) => {
            if (loading) return <p>加载中...</p>;
            if (error) return <p>出错了 :(</p>;
            return <div>
                <Mutation mutation={createCandidateOpt} update={(cache, { data }) => {
                    let items = cache.readQuery({ query: candidates });
                    let lists = [data.createCandidate, ...items.getCandidates];
                    cache.writeQuery({ query: candidates, data: { getCandidates: lists } });
                    return null;
                }}>
                    {(createCandidate, { loading, error, data }) => {
                        if (loading) return <p>加载中...</p>;
                        if (error) return <p>出错了 :(</p>;
                        return <OptCandidate type="add" updateValue={(value) => createCandidate({
                            variables: {
                                input: value,
                            }
                        })} />;
                    }}
                </Mutation>
                <Table columns={columns} dataSource={data.getCandidates} rowKey="id" />
            </div>;
        }}
    </Query>
```
这里面用到了```<Query query={query}></Query>```这样的语法，这是react-apollo的部分，感兴趣的同学可以自己去了解一下。

在表格的列中我设计了一个操作列，里面有详情，修改和删除操作，下面是配置属性和自定义render的部分内容：
```
{
        title: '操作',
        key: 'action',
        render: (text, record) => (
            <Space size="middle">
                <CandidateDetail candidateId={record.id} />
                <Mutation mutation={updateCandidateOpt} update={(cache, { data }) => {
                    let item = cache.readQuery({ query: candidate, variables: { id: record.id } });
                    let result = { ...item, ...data.updateCandidate }
                    cache.writeQuery({ query: candidate, variables: { id: record.id }, data: { getCandidate: result } });
                    return null;
                }}>
                    {(updateCandidate, { loading, error, data }) => {
                        if (loading) return <p>加载中...</p>;
                        if (error) return <p>出错了 :(</p>;
                        return <OptCandidate type="update" candidateId={record.id} updateValue={(value) => updateCandidate({
                            variables: {
                                input: value,
                                id: record.id
                            }
                        })} />;
                    }}
                </Mutation>
                <Mutation mutation={updateCandidateOpt} update={(cache, { data }) => {
                    let items = cache.readQuery({ query: candidates });
                    let lists = items.getCandidates.filter(res => res.id !== data.updateCandidate.id);
                    cache.writeQuery({ query: candidates, data: { getCandidates: lists } });
                    return null;
                }}>
                    {(deleteCandidate, { loading, error, data }) => {
                        if (loading) return <p>加载中...</p>;
                        if (error) return <p>出错了 :(</p>;
                        return <Button type="text" onClick={() => deleteCandidate({
                            variables: {
                                input: {
                                    isDeleted: 1
                                },
                                id: record.id
                            }
                        })}>删除</Button>;
                    }}
                </Mutation>
            </Space>
        ),
    },
```
里面有一些mutation的操作，update表示在mutation之后获取数据进行更新操作，这一步也是我花时间比较多的地方，我一直苦恼于怎么去自动更新列表数据，所以看了很多apollo的文档，最终才理解其中的逻辑，也可以在下一次再分享下。

最后我的代码和演示也在文档压缩包中，大家可以看一下，有错误的地方希望可以指正。
