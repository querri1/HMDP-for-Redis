# hm-dianping

轻量级的本地生活服务示例项目（美团/饿了么风格），用于学习和演示基于 Spring Boot + MyBatis-Plus + Redis 的常见工程实践。

主要特性
- 短信验证码登录（验证码存在 Redis）
- 登录会话以 Token+Redis Hash 存储
- 店铺/店铺类型查询，支持缓存策略（List）
- 代金券、秒杀（库存扣减、异步下单示例）
- 签到功能（Bitmap 位图统计和连续签到统计）
- 文件上传（图片），适配 Nginx 静态目录
- 缓存穿透/互斥/逻辑过期防护示例
- 分布式锁（Lua 脚本 / Redisson）示例
- 异步下单示例（阻塞队列 / Redis Stream）

技术栈
- Java 11
- Spring Boot 2.3.x
- MyBatis-Plus
- MySQL
- Redis（缓存、Bitmap、消息等）
- Redisson（分布式锁，可选）
- Maven 构建

目录结构（部分）
- src/main/java/com/hmdp：核心代码
- src/main/resources/application.yaml：配置
- src/main/resources/db/hmdp.sql：建表与测试数据
- src/main/resources/*.lua：Lua 脚本（分布式锁、库存原子操作）

快速开始（开发环境）

1. 克隆仓库

```bash
git clone <你的仓库地址>
cd hm-dianping
```

2. 环境准备
- MySQL：导入 `src/main/resources/db/hmdp.sql`，并在 `application.yaml` 中设置数据源（url/username/password）。
- Redis：确保 Redis 可用（默认 127.0.0.1:6379），或在 `application.yaml` 中调整 host/port。
- 如果使用文件上传并希望通过 Nginx 提供静态资源，确保配置的图片目录存在并有写权限，例如 `/var/www/html/hmdp/imgs`。

3. 配置（编辑 `src/main/resources/application.yaml`）
- 修改数据库连接（spring.datasource）
- 修改 Redis 连接（spring.redis）
- 修改文件上传目录（如果 UploadController 写入到硬盘）
- 如果使用多个 profile（如 cluster1/cluster2），确保选择合适的 profile

4. 本地构建与运行

```bash
# 构建
mvn clean package -DskipTests

# 直接运行（热运行）
mvn spring-boot:run

# 或使用打包后的 jar
java -jar target/hm-dianping-0.0.1-SNAPSHOT.jar
```

常用命令
- 查看占用端口并结束进程（例：8081）

```bash
sudo lsof -i :8081
sudo kill -9 <PID>
```

- 创建上传目录并授权（示例）

```bash
sudo mkdir -p /var/www/html/hmdp/imgs
sudo chown -R $USER:$USER /var/www/html/hmdp/imgs
```

接口（准确映射）

下面的接口路径与 HTTP 方法来自项目 `src/main/java/com/hmdp/controller` 中的 Controller。
需要登录（拦截器会检查 ThreadLocal） 的接口需要在请求头中携带 `authorization` 字段，其值为登录返回的 token（注意：此项目的拦截器直接读取 `authorization` 的原始 token 字符串，不需要 `Bearer ` 前缀）。

- 用户相关（`UserController`）
  - POST /user/code
    - 参数（form/json）：phone
    - 说明：发送手机验证码并保存到 Redis（示例调用中以 POST 请求发送 phone）。

  - POST /user/login
    - Body (application/json): {"phone":"18888888888","code":"123456"}
    - 说明：使用手机号+验证码登录。成功返回一个 token 字符串（请将该 token 放在后续请求的 `authorization` 头中）。

  - POST /user/logout
    - 说明：登出（当前为 TODO，示例返回功能未完成）。

  - GET /user/me
    - 说明：获取当前登录用户信息（需登录，header: authorization: <token>）。

  - GET /user/info/{id}
    - 说明：获取指定用户详情（UserInfo）。

  - GET /user/{id}
    - 说明：根据 id 查询用户（返回 UserDTO）。

  - POST /user/sign
    - 说明：用户签到（需登录）。

  - GET /user/sign/count
    - 说明：查询连续签到天数（需登录）。

- 上传（`UploadController`）
  - POST /upload/blog
    - form field: file
    - 返回：文件相对路径（保存成功的文件名/路径），文件会保存到 `SystemConstants.IMAGE_UPLOAD_DIR` 指定的磁盘目录。
    - 注意：请确保 `IMAGE_UPLOAD_DIR` 指定的目录存在并且应用进程有写权限（例如 `/var/www/html/hmdp/imgs`）。

  - GET /upload/blog/delete?name={filename}
    - 说明：删除指定上传文件（name 为上传接口返回的相对路径）。

- 店铺类型（`ShopTypeController`）
  - GET /shop-type/list
    - 说明：查询店铺类型列表（返回 List）。

- 店铺（`ShopController`）
  - GET /shop/{id}
    - 说明：根据 id 查询商铺详情。

  - POST /shop
    - Body: Shop 对象 JSON
    - 说明：新增商铺，返回 shop id。

  - PUT /shop
    - Body: Shop 对象 JSON
    - 说明：更新商铺信息。

  - GET /shop/of/type?typeId={typeId}&current={page}&x={lng}&y={lat}
    - 说明：按类型分页查询附近商铺（可传坐标 x/y 进行距离排序）。

  - GET /shop/of/name?name={keyword}&current={page}
    - 说明：按名称关键字分页查询商铺。

- 代金券（`VoucherController`）
  - POST /voucher
    - Body: Voucher JSON
    - 说明：新增普通券，返回优惠券 id。

  - POST /voucher/seckill
    - Body: Voucher JSON（包含秒杀信息）
    - 说明：新增秒杀券，返回优惠券 id。

  - GET /voucher/list/{shopId}
    - 说明：查询某店铺的优惠券列表。

- 秒杀下单（`VoucherOrderController`）
  - POST /voucher-order/seckill/{id}
    - 说明：发起秒杀下单（id 为 voucherId）。实现可能为同步或异步（项目中包含阻塞队列 / Redis Stream 的示例）。

- 博客（`BlogController`）
  - POST /blog
    - Body: Blog JSON
    - 说明：发布博客。

  - PUT /blog/like/{id}
    - 说明：给指定博客点赞（需登录）。

  - GET /blog/of/me?current={page}
    - 说明：查询当前登录用户的博客列表（需登录）。

  - GET /blog/hot?current={page}
    - 说明：查询热门博客列表。

  - GET /blog/{id}
    - 说明：查询博客详情。

  - GET /blog/likes/{id}
    - 说明：查询博客点赞用户列表或点赞状态。

  - GET /blog/of/user?id={userId}&current={page}
    - 说明：查询指定用户的博客列表。

  - GET /blog/of/follow?lastId={lastId}&offset={offset}
    - 说明：基于滚动分页（时间线）查询关注者的博客流。

- 关注（`FollowController`）
  - PUT /follow/{id}/{isFollow}
    - 说明：关注或取消关注（isFollow 为 true/false）。

  - GET /follow/or/not/{id}
    - 说明：检查当前用户是否关注指定用户（需登录）。

  - GET /follow/common/{id}
    - 说明：查询当前用户与指定用户的共同关注（需登录）。

故障排查（基于常见问题与你在开发中遇到的错误）

1. 日志看不到 DEBUG 级别输出
- 修改 `application.yaml` 增加或调整日志级别：

```yaml
logging:
  level:
    com.hmdp: DEBUG
```

2. 验证码校验提示“验证码错误”，但日志显示正确码
- 检查 Redis 中实际存储的 key：`LOGIN_CODE_KEY + phone`（默认键前缀在 `com.hmdp.utils.RedisConstants`）。
- 确认前端/客户端提交的 code 字符串没有空格或编码问题。
- 确认应用读取的 Redis 与写入的 Redis 是同一实例（无错用不同 profile）。
- 如果使用容器或多个实例，注意时钟/时区或跨实例缓存不一致。

3. 单元测试失败：`Method should be public`
- JUnit4 要求测试方法为 `public`（如 `public void testXxx()`）。将测试方法声明为 public 或升级为 JUnit5（不同注解/Runner）。

4. 启动报错：`NoClassDefFoundError: org/aspectj/lang/annotation/Pointcut` 或缺少 AOP 相关依赖
- 在 `pom.xml` 中添加或确保存在：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
  <groupId>org.aspectj</groupId>
  <artifactId>aspectjweaver</artifactId>
  <version>1.9.6</version>
</dependency>
```

5. 启动失败：`BindException: 地址已在使用`
- 端口（如 8081）已被占用。修改 `application.yaml` 中 `server.port` 或释放端口。

6. 文件上传失败：`FileNotFoundException: /var/www/html/hmdp/imgs/... (没有那个文件或目录)`
- UploadController 将文件写到磁盘，需要确保目标父目录存在并且应用用户有写权限。创建目录并 chown 给运行用户，或修改 `application.yaml` 中的上传目录指向一个存在的路径。

7. 登录拦截/Session 问题：ClassCastException（User cannot be cast to UserDTO）
- 登录时在 Redis 存储的是 `UserDTO`（以 Map/Hash 存储）。拦截器或 UserHolder 获取对象时请确保一致的类型转换逻辑。建议：
  - 登录写入 Hash（字符串类型的字段）到 Redis；拦截器应读取 Hash 并组装为 UserDTO，或者在读取时使用 Map -> Bean 转换而不是直接强转对象。

8. Git 推送时报 `GnuTLS recv error (-110)`
- 通常为网络/TLS 通讯中断或代理问题。尝试：
  - 重试推送
  - 检查网络、公司代理或 VPN
  - 使用 SSH 方式推送（配置 SSH key）

9. 代码找不到 Bean（No qualifying bean of type 'IVoucherOrderService'）
- 确保该接口有一个实现类并标注 `@Service`，且包路径在 Spring Boot 的扫描范围之内（主类所在包为扫描根）。

10. Redis Lua 脚本、分布式锁与误删
- 使用 Lua 脚本（EVAL / EVALSHA）确保脚本运行的原子性。
- 卸载锁务必先比对持有者（value）再删除，推荐用官方/成熟库（Redisson）或已验证的 Lua 脚本以避免误删。

建议的开发与调试流程
- 本地运行前先确保 MySQL/Redis 启动并可连通。
- 使用 IDE 的 Run/Debug 并在 `application.yaml` 使用开发 profile（例如 `spring.profiles.active=dev`）。
- 逐步测试接口（先发送验证码 -> 登录 -> 使用 token 调用需登录接口）。

贡献
- 欢迎 Fork -> Feature branch -> Pull Request
- 提交前请运行 `mvn clean package` 并确保简单功能可用

License
- 本仓库示例代码默认未指定许可证。如果要开源，请在仓库根目录添加 LICENSE（例如 MIT）。
