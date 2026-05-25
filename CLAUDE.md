# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

苍穹外卖（sky-take-out）— 中文外卖订餐系统后端，基于 Spring Boot 2.7.3，传智教育学习项目。当前处于早期脚手架阶段，仅员工登录流程完整实现，其余模块包含 POJO 定义和工具类。

## 构建与运行

```bash
# 构建（在 backend/ 根目录执行）
mvn clean package

# 运行（需要 MySQL 数据库 sky_take_out，建表脚本 sky.sql）
mvn spring-boot:run -pl sky-server

# 服务端口: 8080
# API 文档: http://localhost:8080/doc.html (Knife4j)
```

无测试代码，`mvn test` 无实质内容。

---

## 完整目录结构

```
backend/
├── pom.xml                    # 父 POM，Spring Boot 2.7.3，统一管理依赖版本
├── sky.sql                    # MySQL 建表脚本（含种子数据，可直接导入）
│
├── sky-common/                # 公共模块：工具类、常量、异常、配置属性
│   ├── pom.xml
│   └── src/main/java/com/sky/
│       ├── constant/          # 全局常量
│       ├── context/           # ThreadLocal 上下文
│       ├── enumeration/       # 枚举类型
│       ├── exception/         # 自定义业务异常
│       ├── json/              # Jackson 序列化配置
│       ├── properties/        # @ConfigurationProperties 配置类
│       ├── result/            # 统一响应封装
│       └── utils/             # 工具类
│
├── sky-pojo/                  # 数据对象模块：Entity、DTO、VO
│   ├── pom.xml
│   └── src/main/java/com/sky/
│       ├── entity/            # 数据库实体（与表一一对应）
│       ├── dto/               # 请求数据传输对象
│       └── vo/                # 响应视图对象
│
└── sky-server/                # 应用模块：Spring Boot 主程序
    ├── pom.xml
    └── src/main/
        ├── java/com/sky/
        │   ├── SkyApplication.java          # 启动类，@EnableTransactionManagement
        │   ├── config/                      # Spring MVC 配置
        │   ├── controller/
        │   │   └── admin/                   # 管理端 Controller
        │   ├── handler/                     # 全局异常处理
        │   ├── interceptor/                 # JWT 认证拦截器
        │   ├── mapper/                      # MyBatis Mapper 接口
        │   └── service/
        │       ├── (接口)                   # Service 接口
        │       └── impl/                    # Service 实现类
        └── resources/
            ├── application.yml              # 主配置（端口8080，profile=dev）
            ├── application-dev.yml          # 开发环境配置（MySQL连接）
            └── mapper/                      # MyBatis XML 映射文件
```

---

## sky-common 模块详解

### `com.sky.constant` — 全局常量

| 类 | 用途 |
|---|---|
| `AutoFillConstant` | 自动填充方法名常量（`setCreateTime`, `setUpdateTime`, `setCreateUser`, `setUpdateUser`） |
| `JwtClaimsConstant` | JWT Claims 键名（`EMP_ID`） |
| `MessageConstant` | 业务错误消息（中文），如"账号被锁定"、"密码错误" |
| `PasswordConstant` | 默认密码 `123456` |
| `StatusConstant` | 启用/禁用状态：`ENABLE=1`, `DISABLE=0` |

### `com.sky.context` — 请求上下文

- `BaseContext`: ThreadLocal<Long> 存储当前登录用户 ID，拦截器解析 JWT 后写入，全链路通过 `BaseContext.getCurrentId()` 获取

### `com.sky.enumeration` — 枚举

- `OperationType`: `INSERT`/`UPDATE`，配合 AOP 自动填充使用

### `com.sky.exception` — 异常体系

所有业务异常继承 `BaseException`，被 `GlobalExceptionHandler` 统一捕获返回 `Result.error()`：

| 异常类 | 场景 |
|---|---|
| `AccountLockedException` | 账号被禁用 |
| `AccountNotFoundException` | 账号不存在 |
| `PasswordErrorException` | 密码错误 |
| `LoginFailedException` | 登录失败（通用） |
| `DeletionNotAllowedException` | 不允许删除（如起售中的菜品） |
| `OrderBusinessException` | 订单业务异常 |
| `ShoppingCartBusinessException` | 购物车业务异常 |
| `AddressBookBusinessException` | 地址簿业务异常 |
| `SetmealEnableFailedException` | 套餐启用失败（含未启售菜品） |
| `PasswordEditFailedException` | 密码修改失败 |
| `UserNotLoginException` | 用户未登录 |

### `com.sky.json` — 序列化配置

- `JacksonObjectMapper`: 自定义 Jackson ObjectMapper，配置 Java 8 日期时间格式（LocalDateTime→`yyyy-MM-dd HH:mm`，LocalDate→`yyyy-MM-dd`，LocalTime→`HH:mm:ss`）

### `com.sky.properties` — 配置属性类

| 类 | 前缀 | 绑定字段 |
|---|---|---|
| `JwtProperties` | `sky.jwt` | admin-secret-key, admin-ttl, admin-token-name, user-secret-key, user-ttl, user-token-name |
| `AliOssProperties` | `sky.alioss` | endpoint, access-key-id, access-key-secret, bucket-name |
| `WeChatProperties` | `sky.wechat` | appid, secret, mchid, api-key-v3, cert-path, notify-url |

### `com.sky.result` — 响应封装

- `Result<T>`: 统一 API 响应，`code`(1成功/0失败) + `msg` + `data`。静态方法 `success()`/`error()`
- `PageResult`: 分页结果，`total`(总记录数) + `records`(数据列表)

### `com.sky.utils` — 工具类

| 类 | 功能 |
|---|---|
| `JwtUtil` | JWT 生成/解析（HS256，JJWT 0.9.1） |
| `AliOssUtil` | 阿里云 OSS 文件上传 |
| `WeChatPayUtil` | 微信 JSAPI 支付/退款 |
| `HttpClientUtil` | 通用 HTTP 客户端（GET/POST/PUT/DELETE） |

---

## sky-pojo 模块详解

### `com.sky.entity` — 数据库实体

| 实体 | 对应表 | 关键字段 |
|---|---|---|
| `Employee` | employee | id, username, password, name, phone, sex, idNumber, status, createTime, updateTime, createUser, updateUser |
| `User` | user | id, openid, name, phone, sex, idNumber, avatar, createTime |
| `Category` | category | id, type(1菜品/2套餐), name, sort, status, createTime, updateTime, createUser, updateUser |
| `Dish` | dish | id, name, categoryId, price, image, description, status, createTime, updateTime, createUser, updateUser |
| `DishFlavor` | dish_flavor | id, dishId, name, value(JSON数组) |
| `Setmeal` | setmeal | id, categoryId, name, price, image, description, status, createTime, updateTime, createUser, updateUser |
| `SetmealDish` | setmeal_dish | id, setmealId, dishId, name, price, copies |
| `Orders` | orders | id, number, status, userId, addressBookId, orderTime, checkoutTime, payMethod, payStatus, amount, remark, phone, address, userName, consignee 等 |
| `OrderDetail` | order_detail | id, orderId, dishId, setmealId, name, image, dishFlavor, number, amount |
| `ShoppingCart` | shopping_cart | id, userId, dishId, setmealId, name, image, dishFlavor, number, amount, createTime |
| `AddressBook` | address_book | id, userId, consignee, phone, provinceCode, provinceName, cityCode, cityName, districtCode, districtName, detail, label, isDefault |

`Orders` 实体内定义订单状态常量：PENDING_PAYMENT(1), TO_BE_CONFIRMED(2), CONFIRMED(3), DELIVERY_IN_PROGRESS(4), COMPLETED(5), CANCELLED(6)

### `com.sky.dto` — 请求对象

按业务分组：

- **员工**: `EmployeeLoginDTO`(username/password), `EmployeeDTO`(新增/编辑员工), `EmployeePageQueryDTO`(分页查询)
- **分类**: `CategoryDTO`, `CategoryPageQueryDTO`
- **菜品**: `DishDTO`(含 flavors 列表), `DishPageQueryDTO`
- **套餐**: `SetmealDTO`(含 setmealDishes 列表), `SetmealPageQueryDTO`
- **订单**: `OrdersSubmitDTO`, `OrdersPageQueryDTO`, `OrdersConfirmDTO`, `OrdersCancelDTO`, `OrdersRejectionDTO`, `OrdersPaymentDTO`, `OrdersDTO`
- **购物车**: `ShoppingCartDTO`
- **用户**: `UserLoginDTO`(wxCode), `PasswordEditDTO`
- **报表**: `DataOverViewQueryDTO`, `GoodsSalesDTO`

### `com.sky.vo` — 响应对象

- **认证**: `EmployeeLoginVO`(id/username/name/token), `UserLoginVO`(id/token/openid)
- **菜品**: `DishVO`(含 flavors), `DishItemVO`(菜品列表项)
- **套餐**: `SetmealVO`
- **订单**: `OrderVO`(含 orderDetailList), `OrderSubmitVO`(orderId/orderNumber/amount), `OrderPaymentVO`(nonceStr/paySign/timeStamp/signType/package), `OrderStatisticsVO`(待确认/待派送/派送中 各数量)
- **运营数据**: `BusinessDataVO`(新增用户/订单/营业额/有效订单/完成率/平均客单价)
- **报表**: `OrderReportVO`, `TurnoverReportVO`, `SalesTop10ReportVO`, `UserReportVO`
- **概览**: `DishOverViewVO`, `SetmealOverViewVO`, `OrderOverViewVO`

---

## sky-server 模块详解

### `com.sky` — 启动类

- `SkyApplication`: `@SpringBootApplication` + `@EnableTransactionManagement`，`mapperScan` 扫描 `com.sky.mapper`

### `com.sky.config` — 配置类

- `WebMvcConfiguration`:
  - 注册 `JwtTokenAdminInterceptor`，拦截 `/admin/**`，排除 `/admin/employee/login`
  - 配置 Knife4j Docket（API 文档），扫描 `com.sky.controller` 包
  - 静态资源映射

### `com.sky.controller.admin` — 管理端控制器

- `EmployeeController` (`/admin/employee`):
  - `POST /login` — 员工登录，接收 `EmployeeLoginDTO`，返回 `EmployeeLoginVO`（含 JWT token）
  - `POST /logout` — 员工登出（无状态，直接返回成功）

### `com.sky.handler` — 全局异常处理

- `GlobalExceptionHandler`: `@RestControllerAdvice`，捕获所有 `BaseException` 子类，返回 `Result.error()`

### `com.sky.interceptor` — 拦截器

- `JwtTokenAdminInterceptor`: `HandlerInterceptor`，从请求头 `token` 解析 JWT，校验签名和过期时间，将 `empId` 存入 `BaseContext`

### `com.sky.mapper` — MyBatis Mapper

- `EmployeeMapper`: 使用 `@Select` 注解查询，XML 映射文件在 `resources/mapper/EmployeeMapper.xml`（当前为空）

### `com.sky.service` / `com.sky.service.impl` — 业务层

- `EmployeeService` 接口 + `EmployeeServiceImpl` 实现：
  - `login()`: 查用户名 → 校验密码(MD5) → 校验状态 → 生成 JWT → 返回 token
  - `logout()`: 空实现（无状态）

### resources 配置

- `application.yml`: 端口8080，profile=dev，Druid 连接池，MyBatis（mapper XML 路径 + 驼峰映射），日志级别（mapper=debug），JWT admin 配置
- `application-dev.yml`: MySQL `localhost:3306/sky_take_out`，root/123456
- `mapper/`: MyBatis XML 映射文件目录

---

## 数据库

MySQL，库名 `sky_take_out`，建表脚本 `sky.sql`（含种子数据，可直接导入）。

11 张核心表：employee, user, category, dish, dish_flavor, setmeal, setmeal_dish, orders, order_detail, shopping_cart, address_book。全部使用 bigint 自增主键 + InnoDB 引擎。

## 分层架构约定

严格四层分离，禁止跨层调用：

```
Controller → Service 接口 → Service 实现 → Mapper
```

- Controller 只注入 Service，不直接访问 Mapper
- 新增功能需同时创建 Service 接口和 ServiceImpl 实现类
- Mapper 使用 `@Select`/`@Insert` 注解或 XML 映射文件

## 数据对象分层

| 类型 | 包 | 用途 |
|---|---|---|
| Entity | `com.sky.entity` | 数据库表映射，字段与表列一一对应 |
| DTO | `com.sky.dto` | 请求参数封装，Controller 接收入参 |
| VO | `com.sky.vo` | 响应数据封装，Controller 返回出参 |

Entity 不直接暴露给前端，必须通过 DTO 接收、VO 返回。

## 统一响应格式

所有 API 返回 `Result<T>`：`code`(1成功/0失败) + `msg` + `data`。分页用 `PageResult`（total + records）。

## 认证机制

JWT 无状态认证，`JwtTokenAdminInterceptor` 拦截 `/admin/**`（排除登录接口）。请求头携带 `token`，解析后员工 ID 存入 `BaseContext`（ThreadLocal）。密码 MD5 哈希，默认密码 `123456`。

## 关键约定

- **自动填充**：`createTime`/`updateTime`/`createUser`/`updateUser` 由 AOP 自动填充（`AutoFillConstant` + `OperationType`）
- **状态常量**：`StatusConstant.ENABLE=1`, `DISABLE=0`；订单状态在 `Orders` 实体内定义
- **异常体系**：业务异常继承 `BaseException`，`GlobalExceptionHandler` 统一捕获
- **错误消息**：中文，定义在 `MessageConstant`
- **配置属性**：`@ConfigurationProperties` 绑定，前缀 `sky.jwt`/`sky.alioss`/`sky.wechat`
- **API 路径**：管理端 `/admin/**`，用户端（规划中）`/user/**`
- **Swagger**：Knife4j 扫描 `com.sky.controller`，文档地址 `/doc.html`
