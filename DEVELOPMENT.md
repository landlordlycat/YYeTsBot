# 项目手册

# 部署运行

## docker-compose

* 参见 [这里](https://github.com/tgbot-collection/BotsRunner)
* 本目录下的 `docker-compose.yml` 也可以作为参考
* nginx reverse proxy可以[参考这里](https://github.com/BennyThink/WebsiteRunner)
* [参考这里获取数据库](web/README.md)

```shell
# 启动数据库
docker-compose up -d mongo
# 导入数据库
docker cp db.tgz 1234da:/tmp
# 进入容器
docker-compose exec mongo bash
tar xf db.tgz
mongorestore
exit
# 开启服务
docker-compose up -d
```

## 常规方式

### 1. 环境

推荐使用Python 3.6+，环境要求

* redis
* 可选MongoDB

```bash
pip install -r requirements.txt
```

### 2. 配置TOKEN

修改`config.py`，根据需求修改如下配置项

* TOKEN：bot token
* USERNAME：人人影视的有效的用户名
* PASSWORD ：人人影视的有效的密码
* MAINTAINER：维护者的Telegram UserID
* REDIS：redis的地址，一般为localhost
* MONGODB: mongodb的地址

### 3. 导入数据（可选）

如果使用yyets，那么需要导入数据到MongoDB。可以在将数据导入到MySQL之后使用如下脚本导入数据到MongoDB

```shell
python3 web/prepare/convert_db.py
```

**不再兼容旧版本数据**

### 4. 运行

```bash
python /path/to/YYeTsBot/yyetsbot/bot.py
```

### 5. systemd 单元文件

参考 `yyets.service`

### 6. 网站部署运行方式

参考 `worker`和`web`目录下的 `README`。需要注意，cf worker已经停止开发。


## 添加新的资源网站

欢迎各位开发提交新的资源网站！方法非常简单，重写 `BaseFansub`，实现`search_preview`和`search_result`，按照约定的格式返回数据。

然后把类名字添加到 `FANSUB_ORDER` 就可以了！是不是很简单！

## bot无响应

有时不知为何遇到了bot卡死，无任何反馈。😂~~这个时候需要client api了~~😂

原因找到了，是因为有时爬虫会花费比较长的时间，然后pytelegrambotapi默认只有两个线程，那么后续的操作就会被阻塞住。

临时的解决办法是增加线程数量，长期的解决办法是使用celery分发任务。

# 网站开发手册

## 接口列表
* `/api/resource?id=3` GET 获取id=3的资源
* `/api/resource?kw=逃避` GET 搜索关键词
* `/api/top` GET 获取大家都在看
* `/api/name` GET 所有剧集名字
* `/api/name?human=1` GET 人类可读的方式获取所有剧集名字
* `/api/metrics` GET 获取网站访问量
* `/api/user` POST登录，PATCH添加/取消收藏
* `/api/grafana` grafana相关接口
* `/api/blacklist` 黑名单信息

## 防爬

### 1. referer

网站使用referer验证请求

### 2. 加密headers

使用headers `ne1` 进行加密验证，详细信息可以[参考这里](https://t.me/mikuri520/726)

### 3. rate limit

404的访问会被计数，超过10次会被拉入黑名单，持续3600秒，再次访问会持续叠加。


# 持续部署

使用[Docker Hub Webhook](https://docs.docker.com/docker-hub/webhooks/)
(顺便吐槽一句，这是个什么垃圾文档……自己实现validation吧)

参考listener [Webhook listener](https://github.com/tgbot-collection/Webhook)

# 归档资源下载
## Telegram 频道分享
* 包含了2021年1月11日为止的人人影视最新资源，MySQL为主。有兴趣的盆友可以用这个数据进行二次开发[戳我查看详情](https://t.me/mikuri520/668)
* 字幕侠离线数据库 [从这里下载](https://t.me/mikuri520/715)，这个数据比较粗糙，并且字幕侠网站还在，因此不建议使用这个

## 本地下载
如果无法访问Telegram，可以使用如下网址下载数据
* [网站实时数据，MongoDB](https://yyets.dmesg.app/data/yyets_mongo.gz)
* [MySQL](https://yyets.dmesg.app/data/yyets_mysql.zip)
* [SQLite](https://yyets.dmesg.app/data/yyets_sqlite.zip)


# 评论API

## 1. 获取评论
GET `/api/comments`

分页，支持URL参数：
* size: 每页评论数量，默认5（或者其他数值）
* page: 当前页

返回
```json
{
    "data": [
        {
            "date": "2018-09-18 11:12:15",
            "username": "uuua2",
            "content": "tdaadd",
            "id": 2
        },
        {
            "date": "2018-09-01 11:12:15",
            "username": "abcd",
            "content": "tdaadd",
            "id": 1
        }
    ],
    "count": 2
}

```

## 2. 获取验证码
GET `/api/captha?id=1234abc`，id是随机生成的字符串
API 返回字符串，形如 `data:image/jpeg;base64,iVBORw0KGgoAAA....`

## 3. 提交评论
POST `/api/comments`
只有登录用户才可以发表评论，检查cookie `username` 是否为空来判断是否为登录用户；未登录用户提示“请登录后发表评论”

body `resource_id` 从URL中获取，id是上一步验证码的那个id， `captcha` 是用户输入的验证码
```json
{
    "resource_id": 39301,
    "content": "评论内容",
    "id": "1234abc",
    "captcha": "38op"
}
```

返回 HTTP 201添加评论成功，403/401遵循HTTP语义

```json
{
    "message": "评论成功/评论失败/etc"
}
```