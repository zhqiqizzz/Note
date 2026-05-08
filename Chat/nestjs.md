## 框架

### 与 spring boot 对比

| Spring Boot          | NestJS                              |
| -------------------- | ----------------------------------- |
| @Controller          | @Controller()                       |
| @Service             | @Injectable()                       |
| @Module / 配置类        | @Module()                           |
| @Autowired           | constructor 注入                      |
| Bean                 | Provider                            |
| DTO                  | DTO                                 |
| Filter / Interceptor | Guard / Interceptor / Pipe / Filter |
| ApplicationContext   | Nest IoC Container                  |
### 结构

NestJS 更强调工程结构。

它希望你把后端拆成几层：

```
Controller   接收请求
Service      写业务逻辑
Module       组织功能模块
Provider     提供可注入的服务
Entity/Model 数据结构
DTO          接收和校验参数
Guard        做权限判断
Pipe         做参数校验和转换
Interceptor  做请求/响应拦截
Filter       统一异常处理
```

所以 NestJS 适合做中大型后端项目，比如：

```
聊天软件
后台管理系统
电商系统
IM 系统
支付系统
SaaS 平台微服务
```

#### 第一：Controller

Controller 负责 **接收 HTTP 请求**。

比如：

```ts
@Controller('auth')
export class AuthController {
	@Post('register')
	register() {
		return '注册'
	}
}
```

这个就对应：

```
POST /auth/register
```

你可以把 Controller 理解成 Spring Boot 里的：

```java
@RestController
@RequestMapping("/auth")
public class AuthController {
}
```

Controller 不应该写复杂业务逻辑，它主要做：

```
接收请求
取参数
调用 Service
返回结果
```

#### 第二：Service

Service 负责 **业务逻辑**。

比如注册用户：

```ts
@Injectable()
export class AuthService {
	async register(username: string, password: string) {
		// 1. 查用户是否存在
		// 2. 加密密码
		// 3. 创建用户
		// 4. 返回结果
	}
}
```

这个就类似 Spring Boot 里的：

```
@Service
public class AuthService {
}
```

Service 里可以调用：

```
数据库Redis
其他 Service
第三方 SDK
文件服务
消息队列
```

你的聊天系统里，大量核心逻辑都会写在 Service 里。

#### 第三：Module

Module 负责 **组织代码和依赖关系**。

比如：

```ts
@Module({
	controllers: [AuthController],
	providers: [AuthService],
})
export class AuthModule {}
```

这个意思是：

```
AuthModule 这个模块里有：
- AuthController
- AuthService
```

你可以理解为：

```
一个业务模块 = controller + service + 相关依赖
```

比如你的聊天软件后面会有：

```
AuthModule          登录注册
UsersModule         用户资料
FriendsModule       好友关系
ConversationsModule 会话
MessagesModule      消息
GatewayModule       WebSocket
LivekitModule       音视频 token
FilesModule         文件上传
```

Spring Boot 用 Java 注解：

```
@RestController
@Service
@Autowired
```

NestJS 用 TypeScript 装饰器：

```
@Controller()
@Injectable()
@Module()
```

形式不同，本质很像。

## 依赖注入

比如你现在写的：

```ts
@Injectable()
export class AuthService {
	constructor(private readonly prisma: PrismaService) {
	}
}
```

你没有手动写：

```ts
const prisma = new PrismaService()
const authService = new AuthService(prisma)
```

而是 NestJS 自动帮你创建和注入。

这就是 **依赖注入**。

你可以理解成：

```
AuthService 说：我需要 PrismaService
NestJS 容器说：好，我帮你找一个 PrismaService 塞进去
```

它背后有一个 IoC 容器。

### IoC 容器

IoC 是 Inversion of Control，控制反转。

普通写法是你自己控制对象创建：

```ts
const prisma = new PrismaService()
const authService = new AuthService(prisma)
```

IoC 写法是交给框架控制：

```ts
constructor(private readonly prisma: PrismaService) {}
```

你只声明“我需要什么”，框架负责“怎么创建、什么时候创建、怎么传给你”。

这个思想和 Spring 的 ApplicationContext 很像。

在 NestJS 里，大致是这样：

```
启动应用
  ↓
扫描 AppModule
  ↓
找到 imports / controllers / providers
  ↓
创建 Provider 实例
  ↓
分析 constructor 依赖
  ↓
自动注入依赖
  ↓
开始监听 HTTP 请求
```

## 数据库操作

|特性|Prisma Client|MyBatis Plus|
|---|---|---|
|**定位**|增强型 ORM 客户端，简化数据库操作|增强型 MyBatis 框架，简化 CRUD|
|**核心价值**|不用写基础 SQL，通过 API 完成 90% 的查询|不用写基础 SQL，通过 API 完成 90% 的查询|
|**代码生成**|根据 Schema 自动生成类型安全的 Client|根据数据库表自动生成 Entity、Mapper|
|**条件构造器**|`where` 对象动态构建查询条件|`QueryWrapper`/`LambdaQueryWrapper` 动态构建条件|
|**关联查询**|`include` 关键字轻松实现关联|`@TableField` + 自定义方法实现关联|
|**分页排序**|`skip`/`take` + `orderBy` 原生支持|`Page` 对象 + `OrderBy` 注解|
|**批量操作**|`createMany`/`updateMany` 原生支持|`saveBatch`/`updateBatchById` 原生支持|

### 直观对比示例

**Prisma Client 写法**：

```typescript
// 查询某个房间的消息，包含发送者信息，按时间排序，分页
const messages = await prisma.message.findMany({
  where: { 
    roomId: 1,
    content: { contains: 'hello' } // 模糊查询
  },
  include: { user: true }, // 关联查询用户信息
  orderBy: { createdAt: 'asc' },
  skip: 0,
  take: 20
});
```

**MyBatis Plus 写法**：

```java
// 查询某个房间的消息，包含发送者信息，按时间排序，分页
Page<Message> page = new Page<>(1, 20);
QueryWrapper<Message> wrapper = new QueryWrapper<>();
wrapper.eq("room_id", 1)
       .like("content", "hello")
       .orderByAsc("created_at");

Page<Message> messages = messageMapper.selectPage(page, wrapper);
// 关联查询用户信息需要额外处理
```
