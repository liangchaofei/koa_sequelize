使用koa+react+sequelize搭建博客系统，这篇文章讲述koa+sequelize的开发过程。
支持增删改查等功能。

代码放到了[github](https://github.com/liangchaofei/cms)上

先看下系统界面 

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a570d33ea529?w=2534&h=942&f=png&s=155715)


![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a578cd02ccd5?w=2684&h=1382&f=png&s=129630)


![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a57b692751ac?w=2414&h=1056&f=png&s=132719)


![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a58973b5a8b8?w=1784&h=758&f=png&s=77568)


![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a58bc0f078b0?w=1366&h=852&f=png&s=80901)


koa使用狼叔提供的koa-generate脚手架工具。
### 安装koa-generate
```
    npm install -g koa-generator
```
### 建立项目并初始化
```
    koa cms_blog 
    cd cms_blog
    npm install
```
### 安装成功后项目目录如下：

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a0d183e8252f?w=566&h=756&f=png&s=55383)


### 运行并在浏览器打开127.0.0.1:3000即可
```
    npm run dev
```

### 开始项目搭建：采用MVC模式
+ 在根目录下创建controllers,modules,schema,config文件夹
  - controllers:写控制逻辑部分
  - modules：写sql部分
  - schema：写数据表部分
  - config：写数据库配置部分

### 数据库用nodejs的ORM数据库：Sequelize

+ 在config目录下创建db.js，配置数据库
```js
    const Sequelize = require('sequelize')
    const sequelize = new Sequelize('koa','root','123456',{
        host:'localhost',
        dialect:'mysql',
        operatorsAliases:false,
        dialectOptions:{
            //字符集
            charset:'utf8mb4',
            collate:'utf8mb4_unicode_ci',
            supportBigNumbers: true,
            bigNumberStrings: true
        },
        pool:{
            max: 5,
            min: 0,
            acquire: 30000,
            idle: 10000
        },
        timezone: '+08:00'  //东八时区
    })
    
    module.exports = {
        sequelize
    }
```

+ 创建一个文章表article

| 名称 | 类型 | 长度 | 主键 |
|------|------------|------------|------------|
| id | int      |   11    | true | 
| title       | varchar      |   255    | false |
| authore        | varchar      |   255    | false|
| content        | varchar      |   255    | false|
| createdAt        | datetime      |   0    | false|
| updatedAt        | datetime      |   0    | false|
 
+ 在schema文件夹下创建article.js
```js
    const blog = (sequelize, DataTypes) => {
    return sequelize.define('blog',{
        id:{
            type:DataTypes.INTEGER,
            primaryKey:true,
            allowNull:true,
            autoIncrement:true
        },
        title: {
            type: DataTypes.STRING,
            allowNull: false,
            field: 'title'
        },
        author: {
            type: DataTypes.STRING,
            allowNull: false,
            field: 'author'
        },
        content: {
            type: DataTypes.STRING,
            allowNull: false,
            field: 'content'
        },
        createdAt:{
            type:DataTypes.DATE
        },
        updatedAt:{
            type:DataTypes.DATE
        }
    },{
        /**
         * 如果为true，则表示名称和model相同，即user
         * 如果为fasle，mysql创建的表名称会是复数，即users
         * 如果指定的表名称本身就是复数，则形式不变
         */
        freezeTableName: false
    })
}
module.exports =  blog
```

### 数据库部分配置好后，开始接口开发，采用restful api模式

#### get请求：数据查询
+ 在routes目录下创建article.js
```js
    const router = require('koa-router')() // 使用koa-router 来指定接口路由
    const BlogControll = require('../controllers/blog') // 引入Control部分
    
    // 使用router.get 提供get请求
    router.get('/blog', BlogControll.getAllBlog)
```

+ 在controllers目录下创建article.js
```js
    const BlogModel = require('../modules/blog') // 引入model
    
    static async getAllBlog(ctx) {
        const { query } = ctx.request; // 获取前端传来的参数
        try {
            let data = await BlogModel.getAllBlog(query) // 获取查询的数据
            ctx.response.status = 200;
            ctx.body = {
                code: 200,
                msg: 'success',
                data,
                count: data.length
            }
        } catch (err) {
            ctx.response.status = 412;
            ctx.body = {
                code: 412,
                msg: 'error',
                err,
            }
        }
    }
```

+ 在modules目录下创建article.js
```js
    const db = require('../config/db') // 引入数据库配置
    const Sequelize = db.sequelize; // 使用sequelize
    const Blog = Sequelize.import('../schema/blog.js')
    Blog.sync({force: false})
    
    
    static async getAllBlog(query){
        // 通过使用sequelize 的findAll 来查询数据
        // 根据query参数实现查询关键词功能
        return await Blog.findAll({
            where: {
                ...query
            },
            order:[
                ["id","DESC"]
            ],
        })
      
    }
```

+ 至此一个get请求的接口就写好了，运行npm run dev，打开浏览器运行http://127.0.0.1:3000/api/v1/blog 就可以看到数据了。

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a48bb81fce10?w=630&h=430&f=png&s=53309)

+ 可以在后台系统中查看

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a4e44d257234?w=2050&h=888&f=png&s=137708)

#### post请求：数据添加
+ 在routes/article.js
```js
    router.post('/blog', BlogControll.addBlog)
```

+ 在controllers/article.js
```js
    static async addBlog(ctx) {
        let req = ctx.request.body;
        if (req.title && req.author && req.content) {
            try {
                let data = await BlogModel.addBlog(req)
                ctx.response.status = 200
                ctx.body = {
                    code: 200,
                    msg: 'success',
                    data
                }
            } catch (err) {
                ctx.response.status = 412
                ctx.body = {
                    code: 412,
                    msg: 'error',
                    err
                }
            }
        } else {
            ctx.response.status = 416
            ctx.body = {
                code: 416,
                msg: '参数不全',
            }
        }
    }
```
+ 在module/article.js
```js
    static async addBlog(data){
        return await Blog.create({
            title: data.title,
            author: data.author,
            content: data.content,
        })
    }
```
+ 至此添加文章接口就写好了，可以在后台系统添加

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a4f1360b86e0?w=2248&h=1330&f=png&s=93250)

#### delete请求：删除文章
+ 在routes/article.js
```js
    router.delete('/blog/:id',BlogControll.deleteBlog)
```

+ 在controllers/article.js
```js
    static async deleteBlog(ctx) {
        let id = ctx.params.id; // 根据id删除
        if (id) {
            try {
                let data = await BlogModel.deleteBlogs(id)
                ctx.response.status = 200;
                ctx.body = {
                    code: 200,
                    msg: 'success',
                }
            } catch (err) {
                ctx.response.status = 412;
                ctx.body = {
                    code: 412,
                    msg: 'err',
                    err
                }
            }
        } else {
            ctx.response.status = 416;
            ctx.body = {
                code: 416,
                msg: '缺少id',
            }
        }
    }
```
+ 在module/article.js
```js
    static async deleteBlogs(id){
        return await Blog.destroy({
            where:{
                id
            }
        })
    }
```
+ 至此删除文章接口就写好了，可以在后台系统删除

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a50d181cc81d?w=2166&h=446&f=png&s=75706)

### put请求：编辑文章
+ 在routes/article.js
```js
    router.put('/blog', BlogControll.updateBlog)
```

+ 在controllers/article.js
```js
    static async updateBlog(ctx) {
        let req = ctx.request.body;
        try {
            let data = await BlogModel.updateBlog(req)
            ctx.response.status = 200;
            ctx.body = {
                code: 200,
                msg: 'success',
            }
        } catch (err) {
            ctx.response.status = 412;
            ctx.body = {
                code: 412,
                msg: 'error',
                err
            }
        }
    }
```
+ 在module/article.js
```js
    static async updateBlog(data){
        const {id,title,author,content} = data;
        console.log('id',id)
        return await Blog.update(
            {
                title,
                author,
                content,
                id
            },
            {
                where:{
                    id
                }
            }
        )
    }
```
+ 至此更新文章接口就写好了，可以在后台系统更新

![](https://user-gold-cdn.xitu.io/2019/11/14/16e6a52fc705d774?w=2302&h=1292&f=png&s=89563)


总结：以上通过koa+sequelize实现了增删改查的接口。
代码放到了[github](https://github.com/liangchaofei/cms)上，可以直接下载运行。如果这篇文章对您有帮助，感谢star
