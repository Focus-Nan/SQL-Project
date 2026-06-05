# 采购管理系统 — 开发日志

> 项目路径：`d:\18785\C++\`  
> 技术栈：Vue 3 + Vite + Element Plus + Spring Boot 2.7 + MySQL 8 + JWT  
> 开发时间：2026-06-05

---

## 一、项目初始化

### 后端 Spring Boot
- 创建 Maven 项目，Spring Boot 2.7.18，JDK 17
- 依赖：spring-boot-starter-web、spring-boot-starter-data-jpa、mysql-connector-j、jjwt 0.9.1、spring-security-crypto、lombok、jaxb-api
- 配置文件 `application.yml`：端口 8080，MySQL `pms_db`，JWT 24h 过期
- 创建 6 张数据库表 + 初始数据（`init.sql`）

### 前端 Vue 3
- Vite 5 + Vue 3.4 + Element Plus 2.5 + Vue Router 4 + Pinia + Axios
- Vite 代理 `/api` → `localhost:8080`

---

## 二、数据库设计（6 表）

| 表 | 主要字段 | 关联 |
|----|---------|------|
| `supplier` | id, supplier_no(UK), name, short_name, address, phone, email, contact_person, contact_phone, remark | — |
| `employee` | id, employee_no(UK), name, password(BCrypt), level(1=admin/2=user), phone, salary, remark | — |
| `product` | id, product_no(UK), name, price, supplier_id(FK), description, remark | → supplier.id |
| `member` | id, member_no(UK), name, phone, address, email, **discount**(0~1), remark | — |
| `purchase_main` | id, purchase_no(UK), employee_id(FK), total_quantity, total_price, purchase_time, remark | → employee.id |
| `purchase_detail` | id, detail_no, purchase_main_id(FK), product_id(FK), quantity, unit_price, total_price, remark | → purchase_main.id, product.id |

> 所有表含 `create_time`，主键自增。

---

## 三、后端架构

```
com.pms
├── PmsApplication.java
├── config/
│   ├── CorsConfig.java          # 跨域（允许所有来源）
│   └── WebMvcConfig.java        # 注册 AuthInterceptor，排除 /auth/login 和 /auth/register
├── common/
│   ├── Result.java              # 统一响应 { code, msg, data }
│   └── GlobalExceptionHandler.java
├── entity/       (6 个 JPA Entity)
├── repository/   (6 个 JPA Repository，含模糊搜索方法)
├── service/      (AuthService + 5 个业务 Service)
├── controller/   (AuthController + 5 个 REST Controller)
├── interceptor/
│   └── AuthInterceptor.java     # JWT 校验，解析 token 注入 request attributes
└── util/
    └── JwtUtil.java             # HS256 签名，24h 过期
```

### API 权限分级
- **公开**：`POST /api/auth/login`、`POST /api/auth/register`
- **登录即可**：GET 类查询接口、`GET /api/auth/me`
- **仅管理员（level=1）**：所有 POST/PUT/DELETE 写操作

### 批量导入优化
- `ProductService.batchImport`：自动识别 `|` `,` `Tab` 分隔符；供应商支持填名称（模糊匹配）或 ID；逐行容错，部分成功返回
- `SupplierService`、`EmployeeService`、`MemberService` 同样支持多分隔符

---

## 四、前端架构

```
src/
├── main.js                    # 入口：Pinia + Router + Element Plus（中文）
├── App.vue
├── api/index.js               # 全部 API 封装（auth + 5模块 CRUD）
├── utils/request.js           # Axios 实例：自动附 token，401 跳登录
├── store/user.js              # Pinia：token + userInfo + isAdmin()
├── router/index.js            # 路由守卫：未登录→/login，非admin→/user
└── views/
    ├── Login.vue              # 登录页（含注册弹窗）
    ├── admin/
    │   ├── AdminLayout.vue    # 侧边栏 + 顶栏（面包屑 + 实时时钟 + 头像）
    │   ├── Dashboard.vue      # 统计卡片 + 各模块数据概览表
    │   ├── SupplierManage.vue # CRUD + 批量导入 + 搜索
    │   ├── ProductManage.vue  # CRUD + 批量导入 + 搜索 + 供应商下拉
    │   ├── EmployeeManage.vue # CRUD + 批量导入（密码可留空修改）
    │   ├── MemberManage.vue   # CRUD + 批量导入 + 折扣率下拉
    │   └── PurchaseManage.vue # 主表/明细两级 CRUD
    └── user/
        ├── UserLayout.vue     # 用户侧边栏
        ├── MyInfo.vue         # Hero 头部 + 快捷状态卡 + 详情双栏
        ├── MemberList.vue     # 会员卡片网格 + 折扣等级条 + 搜索
        ├── ProductList.vue    # 侧边栏筛选器 + 卡片/表格双视图 + 搜索高亮
        └── PurchaseList.vue   # 手风琴卡片 + 展开明细
```

---

## 五、认证与鉴权流程

1. 登录 `POST /api/auth/login` → 后端 BCrypt 校验 → 返回 JWT（含 employeeId/name/level）
2. 前端存 token 到 localStorage，Axios 拦截器自动附加 `Authorization: Bearer <token>`
3. AuthInterceptor 校验所有 `/api/**` 请求的 token 有效性
4. Controller 层通过 `request.getAttribute("level")` 判断管理员权限
5. 前端路由守卫：无 token → /login；非 admin → /user；admin 访问 /user → /admin

---

## 六、注册功能

- 后端 `POST /api/auth/register`：接受 employeeNo/name/password/phone，自动设 level=2
- 前端 Login.vue 底部"立即注册"链接，弹出注册对话框
- 注册成功自动登录跳转用户中心

---

## 七、UI 设计迭代

### 第一版 → 第二版
- 登录页：从单调渐变背景 → 深色背景 + 动态飘浮光球 + 点阵纹理 + 左右分栏玻璃拟态
- Dashboard：从 4 个纯数字卡片 → 统计卡片 + 供应商/商品/员工/会员概览表 + 采购记录表
- AdminLayout/UserLayout：深色渐变侧边栏 + 圆角菜单 + 激活态高亮 + 实时时钟 + 面包屑

### 第二版 → 第三版
- 5 个管理页全面统一设计语言：标题图标 + 计数条 + 圆角卡片表格 + label-position="top" 对话框
- 用户 MyInfo：Hero 渐变头部 + 快捷状态卡 + 双栏详情卡
- 用户 ProductList：侧边栏筛选器（供应商 checkbox + 价格区间）+ 搜索关键词黄底高亮 + 卡片/表格切换
- 用户 PurchaseList：手风琴卡片 + 明细展开动画
- 用户 MemberList：卡片网格 + 折扣等级彩色条 + 折扣进度条

---

## 八、测试账户

| 角色 | 账号 | 密码 | 折扣 |
|------|------|------|------|
| 管理员 | admin | 123456 | — |
| 普通用户 | user001 | 123456 | — |
| 员工 | EMP001 | 123456 | — |
| 会员 VIP | M001 / 张三 | — | 95% |
| 会员金牌 | M002 / 李四 | — | 88% |
| 会员银牌 | M003 / 王五 | — | 90% |

---

## 九、环境配置

| 组件 | 路径/版本 |
|------|----------|
| JDK | `C:\Program Files\Java\jdk-17` (17.0.12) |
| Maven | `C:\Users\18785\apache-maven-3.9.9` |
| MySQL | `C:\Program Files\MySQL\MySQL Server 9.7` (root/123456) |
| Node.js | v24.16.0 |

### 启动命令

```bash
# 后端
cd backend
set JAVA_HOME=C:\Program Files\Java\jdk-17
mvn spring-boot:run -DskipTests

# 前端
cd frontend
npm install
npm run dev
```

- 前端：http://localhost:5173
- 后端：http://localhost:8080

---

## 十、已知问题与待优化

1. 前端采购明细增删后主表 totalQuantity/totalPrice 未自动重算（需手动刷新或后端触发器）
2. 批量导入错误信息在前端仅显示为 error message，未做友好展示
3. 无分页——数据量大时表格加载慢
4. 无操作日志/审计功能
5. JWT 无刷新机制，24h 后需重新登录
