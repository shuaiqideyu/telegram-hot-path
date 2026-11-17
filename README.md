# 自建 Telegram Bot API

把自己的 bot 从 `https://api.telegram.org` 改到一台自己运行的官方 Bot API 服务上。服务是 [tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api)。bot 只连这台机器，由它用 MTProto 访问数据中心，不再经过共享的云端入口。

适合三种情况：

- 文件大小顶到云端上传或下载上限。
- 业务处理已经很薄，计时之后这一跳仍是最慢的一段。
- 某一个 token 在云端入口上整体不回包，或传字节卡死；同一台机器上的其它 token 正常。

纯文本指令只为省几十毫秒时，把处理做薄即可，不必单独起这台服务。三种情况都不成立，也不要迁。

下面按准备、启动、迁移、退回写完。命令里的 token、`api_id`、`api_hash` 都是占位符。对照日期是 2026-11-17。来源是 [Bot API：Using a Local Bot API Server](https://core.telegram.org/bots/api#using-a-local-bot-api-server)、[logOut](https://core.telegram.org/bots/api#logout)、[Bot API FAQ](https://core.telegram.org/bots/faq)，以及 [telegram-bot-api](https://github.com/tdlib/telegram-bot-api) master 上的 `Client::finish_closing`、`Client::send` 和 `MAX_CONCURRENTLY_SENT_CHAT_MESSAGES`。真正迁移前再读一遍官方说明。

## 本机模式多出来的能力

启动时加上 `--local` 之后，官方服务允许：

- 下载文件不再受云端那个大小上限约束。
- 上传最大 2000 MB。
- 用本机路径和 `file://` 上传。
- webhook 可以使用 HTTP、任意本机 IP、任意端口。
- `max_webhook_connections` 最大可以到 100000。
- `getFile` 的 `file_path` 直接是本机绝对路径，不必再下载一次。

云端发送文件的上限，同一天的文档是：

| 方式 | 照片 | 其它文件 |
|---|---|---|
| 把 HTTP URL 交给 Telegram 去拉取 | 5 MB | 20 MB |
| `multipart/form-data` 上传 | 10 MB | 50 MB |
| 已有的 `file_id` | 不重复上传 | 不重复上传 |

`file_id` 属于这一个 bot，不能拿给另一个 bot 用。同一个 bot 在云端拿到的 `file_id`，改连本机服务后继续有效。

## 频率

发送限额跟云端是同一套，由数据中心执行。对照 2026-11-17 的 [Bot API FAQ](https://core.telegram.org/bots/faq)：

| 范围 | 免费上限 |
|---|---|
| 同一个聊天 | 大约每秒 1 条。可以短时间冲一下，接着会收到 429 |
| 同一个群 | 每分钟 20 条 |
| 群发 | 大约每秒 30 条 |

在 @BotFather 打开 Paid Broadcasts 之后，群发可以到大约每秒 1000 条。超过每秒 30 条的部分，每条 0.1 Stars，从 bot 余额里扣。打开这个档位要余额里至少有 100000 Stars，并且月活至少 100000。只对发送成功的消息收费。余额、月活或单价变了，以当天 FAQ 为准。

`--local` 不替调用方守这三档。云端入口上的两段排队在源码里都写着非 local 模式才生效：启动满 60 秒后，活跃请求过多、同时上传的文件过多或过大，会直接拒绝；带文件的请求还会按文件大小把下一次发送错开。本机模式跳过这两段。本机只在同一个聊天里、还没发完的消息堆到 310 条时拒绝，源码把这个数标成一个故意放得很宽的闸，不是上面的官方限额。

所以 bot 自己按这三档排队。收到 429，停 `retry_after` 再发；连续 429 就把速度降下来。群发排进 bot 已经在用的那条发送队列。

## 要接受的代价

- 这是单点。进程停了，挂在上面的每个 bot 都停。用会自动拉起的监督进程，工作目录放在磁盘上，不要放在临时盘。
- `api_id` 和 `api_hash` 属于一个 Telegram 账号。这个应用失效时，这台服务上的全部 bot 一起掉线。用长期保留的账号申请，不要用测试号或自动化号。
- Bot API 升级要重新编译再重启服务。8 核机器上一次编译大约 30 分钟。
- 服务只接受 HTTP。bot 和它在同一台机器上时，webhook 走环回地址。要从别的机器连进来，前面加 TLS 终结，不要把只说 HTTP 的端口直接暴露到公网。

## 准备应用凭据

到 [获得 api_id](https://core.telegram.org/api/obtaining_api_id) 申请 `api_id` 和 `api_hash`。这是 [my.telegram.org](https://my.telegram.org) 的应用凭据，不是 BotFather 的 bot token。

写进环境文件，不要写进命令行，也不要提交进仓库：

```
TELEGRAM_API_ID=123456
TELEGRAM_API_HASH=your_api_hash
```

文件权限设为仅所有者可读。

## 编译

依赖是 OpenSSL、zlib、支持 C++17 的编译器、gperf、CMake 3.10 及以上。按目标系统生成的完整命令在 [官方构建说明](https://tdlib.github.io/telegram-bot-api/build.html)。通用步骤：

```sh
git clone --recursive https://github.com/tdlib/telegram-bot-api.git
cd telegram-bot-api
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --target install
```

装好后的二进制一般在 `/usr/local/bin/telegram-bot-api`，以本机 CMake 的安装前缀为准。

## 启动

只监听环回，工作目录持久保存。默认端口是 8081。

```sh
install -d -m 700 /var/lib/telegram-bot-api
set -a
. /etc/telegram-bot-api.env
set +a
exec telegram-bot-api \
  --local \
  --http-ip-address=127.0.0.1 \
  --http-port=8081 \
  --dir=/var/lib/telegram-bot-api
```

用 systemd 时，单元只负责拉起这一个进程：

```ini
[Unit]
Description=Telegram Bot API
After=network-online.target

[Service]
EnvironmentFile=/etc/telegram-bot-api.env
ExecStart=/usr/local/bin/telegram-bot-api --local --http-ip-address=127.0.0.1 --http-port=8081 --dir=/var/lib/telegram-bot-api
Restart=on-failure
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

`--http-ip-address` 不写时，服务接受任意本机 IPv4 地址上的连接。和 bot 同机部署时固定写 `127.0.0.1`。

## 先证明服务可用

用一个格式正确、但并不存在的 token 打 `getMe`。`api_id` 和 `api_hash` 可用时，几秒内返回 `401 Unauthorized`。返回 `Unauthorized: invalid api-id/api-hash` 说明应用凭据不能用，先停在这里，不要动线上 bot。

```sh
curl -sS "http://127.0.0.1:8081/bot123456:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA/getMe"
```

## 确认是云端入口卡住

第三种情况要在服务器上确认，不要靠重启猜。对出问题的 token 直接打一次 `getMe`，再发一张很小的图片；旁边用一个正常的 token 做同样两步。这个 token 的每个方法都超时或连续几分钟返回 502，或者传字节一直挂到 504，而同 token 的文字和 `file_id` 发送在一秒内返回，才是云端入口的问题。

重启 bot 解决不了这种情况。卡住期间重启，bot 还会登记不上 webhook。

## 把一个 bot 迁过来

一个 token 同一时刻只能在一个入口登录。云端和本机同时登录时，更新会拆到两边，不保证收全。

1. 本机服务先按上一节跑起来，并用假 token 确认返回的是 `401 Unauthorized`。
2. 对这个 token 在云端调用一次 [logOut](https://core.telegram.org/bots/api#logout)。成功之后可以马上在本机登录，但 10 分钟内不能再登回云端。

   ```sh
   curl -sS "https://api.telegram.org/bot<token>/logOut"
   ```

3. 把这个 bot 的请求根地址改到 `http://127.0.0.1:8081`。记下「这个 token 已经迁走」，进程重启时不要再去请求 `api.telegram.org`。
4. 把 webhook 登记成本机地址，`secret_token` 与原来相同。`--local` 允许发往环回的明文 HTTP，更新不再经过 CDN 和反向代理。

   ```sh
   curl -sS "http://127.0.0.1:8081/bot<token>/setWebhook" \
     -d "url=http://127.0.0.1:<bot 端口>/<路径>" \
     -d "secret_token=<原来的 secret>"
   ```

5. 同一步改完这个 token 的全部调用方：业务进程、管理接口、健康检查、手敲的 `curl`。之后再有一次打到 `api.telegram.org` 的请求，就会把 bot 重新登录回云端，两个入口拆分更新。
6. 负责告警的那个 bot 留在云端。本机服务或业务进程停了，告警仍要发得出去。告警发送放在业务进程外面。
7. 先迁一个没有流量的 bot。其余的在安静时段之后迁，一次迁一个部署单元。

`--local` 之下，`getFile` 返回的 `file_path` 是本机绝对路径。业务代码要能直接读这个路径。

## 迁完之后的事实

- 本机服务上的 `update_id` 从另一套数字开始。用 `update_id` 做去重的键，迁移后进入新的编号空间。
- `getMe` 和 `getWebhookInfo` 由本机服务自己回答，不能证明到数据中心的链路是通的。探测用 `getMyCommands`，它每次都会向 Telegram 询问。`getWebhookInfo` 只说明更新有没有送到 bot 进程：登记的 URL、待投递数量、最近一次投递错误。
- 调用 `logOut` 的那个服务进程还会把 token 留在关闭窗口里。这个 bot 在该服务上已经授权，且授权时间早于 30 分钟，窗口是 1 秒；否则是 30 分钟。窗口内再请求得到 `Logged out`。这是源码 `Client::finish_closing` 的行为。不要把同一个 token 在云端和本机连着各登出一次当彩排。本机这一侧重启服务，可以清掉进程里的这把锁。云端文档写的 10 分钟不受本机重启影响。
- 本机服务重启后，会按工作目录里已经保存的登录恢复 webhook。bot 进程不用为此重启。
- bot 从系统里删除，或换了新 token 时，在本机服务上对旧 token 调用 `logOut`。旧 token 再拿去云端用，会和本机登录拆分更新。

## 退回云端

先停掉这个 token 的全部调用方。在本机服务上对每个 token 调用 `deleteWebhook` 和 `logOut`，清掉「已经迁走」的记录，去掉本机根地址，再启动调用方，让它们重新登记公网 webhook。

退回同样要等关闭窗口。刚在云端登出过的 token，10 分钟内登不回云端。

## 从一台本机换到另一台

两台本机同时登录同一个 bot 时，更新不保证收全。先在旧服务器上 `logOut`，再改到新服务器。

想避免两步之间丢掉更新：在旧服务器上 `deleteWebhook`，再调用 [close](https://core.telegram.org/bots/api#close) 关闭这个 bot 实例。实例关闭后，按 bot 的用户 ID 在旧服务工作目录里找到它的子目录，整份搬到新服务的工作目录，然后把请求发到新服务。

## 对照

- 服务与迁移步骤：[tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api)
- 本机模式的能力列表：[Using a Local Bot API Server](https://core.telegram.org/bots/api#using-a-local-bot-api-server)
- 从云端登出：[logOut](https://core.telegram.org/bots/api#logout)
- 应用凭据：[obtaining_api_id](https://core.telegram.org/api/obtaining_api_id)
- 构建命令生成器：[build.html](https://tdlib.github.io/telegram-bot-api/build.html)
- 频率上限：[Bot API FAQ](https://core.telegram.org/bots/faq)
