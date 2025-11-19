# 员工打卡与任务进度上报工具设计方案

## 1. 产品概述
- **产品名称**：员工打卡与任务进度上报工具
- **产品目标**：通过统一的微信小程序与后端服务，完成员工打卡、任务管理、进度上报、报销与审批流程的一体化管理，提高组织运营透明度与效率。
- **目标用户**：
  - 普通员工：打卡、查看任务、提交进度与报销。
  - 管理员（部门经理、HR、财务）：审批打卡异常、任务与报销，导出报表。
  - 高层管理：获取全局进度、考勤与报销统计，辅助决策。

## 2. 核心模块
| 模块 | 关键功能 | 主要用户 |
| --- | --- | --- |
| 打卡与考勤 | GPS打卡、异常提醒、月度统计、考勤导出 | 员工、HR |
| 任务管理 | 任务分配、进度更新、甘特图/进度条展示、报表 | 员工、经理 |
| 报销管理 | 申请、凭证上传、审批、批量操作、统计 | 员工、财务 |
| 后台与权限 | 多角色权限、审计日志、数据导出 | 管理员 |
| 可视化与提醒 | 仪表盘、图表、智能提醒、消息通知 | 全体 |

## 3. 业务流程
1. **打卡流程**：
   1. 员工在小程序发起打卡，采集 GPS + 时间。
   2. 后端进行地理围栏/合法性校验，写入考勤表。
   3. 触发实时反馈（打卡成功/失败原因），并在个人统计中累积。
   4. HR 管理后台可浏览、筛选、导出并处理异常记录。
2. **任务管理流程**：
   1. 经理在后台或 Web 端创建任务，分配给员工或团队。
   2. 员工在小程序查看任务详情并提交进度百分比、备注和附件。
   3. 系统通过进度条、甘特图展示任务状态，同时生成部门/公司级统计。
3. **报销流程**：
   1. 员工填写报销申请并上传凭证（图片/PDF）。
   2. 审批链按角色顺序流转（部门经理→财务→高层可选）。
   3. 各阶段审批结果实时同步，员工可跟踪状态；财务可批量处理并导出报表。
4. **后台与审计**：
   - 所有管理员操作写入 `audit_logs`，支持按时间、操作者、模块查询。

## 4. 技术架构
```
微信小程序 (WXML/WXSS/JS)
        │ RESTful/GraphQL API (HTTPS, JWT)
        ▼
Node.js (NestJS) 或 Python (FastAPI/Django) 服务
        │
MySQL (主写)+Redis(缓存)+对象存储(OSS/S3)   消息队列(RabbitMQ/Redis Streams)
        │                                   │
数据分析与报表服务 -------------------------┘
```

- **部署**：容器化（Docker + Kubernetes），使用 Nginx 反向代理与 HTTPS 终端。
- **扩展**：微服务拆分为考勤、任务、报销、权限模块，可平行伸缩。

## 5. 数据模型（MySQL 示例）

### 5.1 关键数据表
| 表名 | 说明 | 关键字段 |
| --- | --- | --- |
| `users` | 员工与管理员账户 | id, name, role, department_id, hashed_password, status |
| `attendance_records` | 打卡记录 | id, user_id, type(in/out), timestamp, location, status |
| `tasks` | 任务基础信息 | id, title, description, deadline, priority, creator_id |
| `task_assignments` | 任务分派 | id, task_id, assignee_id, progress, last_report_at |
| `task_reports` | 进度上报历史 | id, assignment_id, percent, note, attachment_url |
| `expenses` | 报销申请 | id, applicant_id, category, amount, status, currency |
| `expense_attachments` | 报销凭证 | id, expense_id, file_url, checksum |
| `approval_flows` | 审批链配置 | id, module, step_order, role_required |
| `approvals` | 审批记录 | id, module, module_id, approver_id, decision, comment |
| `audit_logs` | 操作日志 | id, operator_id, action, metadata, created_at |

### 5.2 数据加密与隐私
- 打卡地理坐标、报销凭证等敏感字段使用 AES-256 加密后存储。
- 文件存储采用私有桶 + 临时签名 URL；下载需鉴权。

## 6. API 设计示例
- `POST /api/v1/auth/login`：用户登录，返回 JWT + Refresh Token。
- `POST /api/v1/attendance/clock`：提交打卡（body: type, gps, deviceInfo）。
- `GET /api/v1/attendance/records?month=2024-05`：获取当月记录与统计。
- `GET /api/v1/tasks` / `POST /api/v1/tasks`：任务列表/创建。
- `PATCH /api/v1/tasks/{id}/progress`：更新任务进度。
- `POST /api/v1/expenses`：提交报销申请。
- `POST /api/v1/expenses/{id}/approve`：审批节点操作。
- `GET /api/v1/dashboard/summary`：管理端仪表盘数据。

所有接口：
- 使用 HTTPS + JWT Bearer 认证。
- 根据角色（RBAC）与资源拥有者校验访问权限。
- 支持分页、筛选、导出（CSV/Excel）。

## 7. 微信小程序前端设计
- **页面结构**：
  1. 登录/切换公司。
  2. 首页仪表盘（今日打卡状态、任务列表、报销提醒）。
  3. 打卡页面（实时定位、打卡按钮、历史记录）。
  4. 任务页（列表、详情、进度上报、甘特图/进度条）。
  5. 报销页（申请、附件上传、审批进度）。
  6. 消息中心（提醒、审批通知）。
  7. 管理端入口（角色识别后显示管理菜单）。
- **组件**：打卡按钮组件、进度条组件、统计图组件（ECharts for WeChat）、附件上传组件、审批状态组件。
- **状态管理**：使用微信小程序原生 store 或 mobx-miniprogram，支持离线缓存与重新同步。
- **交互特性**：实时提示、失败重试、长按查看详情、下拉刷新。

## 8. 权限与安全
- **RBAC**：角色（employee、manager、hr_admin、finance_admin、super_admin），以 `roles` 表或 IAM 服务实现。
- **细粒度权限**：任务/报销按部门或个人范围限定，审批时校验 `approvals` 流程。
- **审计**：所有敏感操作写入 `audit_logs`，含 request_id、IP、设备信息。
- **登录安全**：支持企业微信/小程序登录、MFA（短信/OTP 可选）。

## 9. 性能与高可用
- 使用 Redis 缓存热门统计数据与权限信息。
- 打卡写入采用消息队列削峰，后台异步计算统计报表。
- 数据库读写分离、定期备份、慢查询监控。
- 使用 CDN/对象存储承载附件，减轻主服务压力。

## 10. 运维与监控
- Prometheus + Grafana 监控 API 延迟、错误率、队列堆积。
- Loki/ELK 收集日志。
- 配置灰度发布与自动回滚。

## 11. 未来扩展
- Web 与移动 App 客户端，复用 GraphQL/REST API。
- 智能提醒（基于任务截止、打卡异常、报销滞留）。
- AI 分析：任务完成预测、员工绩效画像、异常行为识别。
- 与第三方 HR、财务系统通过 Webhook/ETL 同步数据。

## 12. 开发计划（示例）
| Sprint | 目标 | 关键输出 |
| --- | --- | --- |
| Sprint 1 | 搭建基础服务、用户与权限、打卡 MVP | Auth、RBAC、打卡 API、小程序登录页面 |
| Sprint 2 | 任务管理与进度上报 | 任务 API、甘特图组件、部门仪表盘 |
| Sprint 3 | 报销模块与审批流程 | 报销 API、附件上传、安全加密 |
| Sprint 4 | 报表、导出、监控与上线 | 数据仓库、统计报表、监控告警 |

## 13. 测试策略
- **单元测试**：后端至少 80% 覆盖，Mock DB/Cache。
- **集成测试**：API 套件覆盖主流程，模拟多角色。
- **性能测试**：JMeter/Locust 模拟高并发打卡、任务查询、报销导出。
- **安全测试**：SQL 注入、权限绕过、文件上传扫描、数据加密验证。

## 14. 交付物
- 微信小程序前端代码与设计稿（Figma）。
- 后端服务（Docker 镜像、Helm chart）。
- 数据库 schema 与迁移脚本。
- 运维手册、管理员培训文档。
- PRD、测试报告、上线 checklist。
