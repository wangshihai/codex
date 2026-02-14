# 学生校服支付后端（Go + 微信支付）设计方案

## 1. 目标与规模

- **业务目标**：学校给学生创建“校服订单”，学生在小程序中完成支付，后端完成对账、状态流转、退款、通知。
- **用户规模**：约 2000 名学生（中小规模）。
- **技术目标**：稳定、可审计、易扩展，满足微信支付合规要求。

---

## 2. 总体架构

```text
[微信小程序]
   |
   | HTTPS + JWT
   v
[API 网关 / Gin HTTP 服务]
   |
   +--> [订单服务 Order]
   +--> [支付服务 Payment(微信 JSAPI)]
   +--> [退款服务 Refund]
   +--> [学生/班级服务 Student]
   +--> [管理后台 Admin API]
   |
   +--> [MySQL 8.0]
   +--> [Redis]
   +--> [消息队列(可选: NATS/RabbitMQ)]

微信侧回调：
[WeChat Pay Notify] --> [/api/v1/payments/wechat/notify]
```

**建议部署（2000 学生）**
- 2 台应用实例（主备/负载均衡），每台 2C4G 即可。
- MySQL 单主 + 自动备份。
- Redis 单实例（缓存、幂等键、限流）。

---

## 3. 核心业务流程

## 3.1 管理员创建订单
1. 管理员选择学生（或按班级批量）创建校服订单。
2. 系统生成 `order_no`、金额、截止时间，状态为 `PENDING`。
3. 订单写库并推送通知（可选：订阅消息/短信）。

## 3.2 学生支付（微信 JSAPI）
1. 小程序调用后端：`POST /orders/{id}/pay`。
2. 后端校验订单归属、状态、金额。
3. 后端调用微信 `transactions/jsapi` 下单，生成 `prepay_id`。
4. 后端返回前端调起支付所需参数（timeStamp、nonceStr、package、paySign）。
5. 小程序发起支付。
6. 微信异步通知后端支付结果。
7. 后端验签成功后，将订单置为 `PAID`，写支付流水。

## 3.3 退款流程
1. 管理员发起退款申请。
2. 后端调用微信退款接口。
3. 退款结果通过回调或主动查询确认后更新 `refund_status`。

---

## 4. 数据库设计（MySQL）

## 4.1 核心表

### 4.1.1 `students`
- 存学生基础信息，含小程序 `openid`。

### 4.1.2 `orders`
- 一笔校服应付记录。

### 4.1.3 `payments`
- 一笔订单可以有多次支付尝试（通常成功 1 次）。

### 4.1.4 `refunds`
- 退款记录。

### 4.1.5 `wechat_notify_logs`
- 保存微信回调原文，便于审计与排错。

## 4.2 SQL DDL（可直接落地）

```sql
CREATE TABLE students (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  student_no VARCHAR(64) NOT NULL UNIQUE COMMENT '学号',
  name VARCHAR(64) NOT NULL,
  class_name VARCHAR(64) NOT NULL,
  grade_name VARCHAR(64) NULL,
  openid VARCHAR(64) NULL UNIQUE,
  phone VARCHAR(20) NULL,
  status TINYINT NOT NULL DEFAULT 1 COMMENT '1=正常,0=停用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_class_name(class_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no VARCHAR(64) NOT NULL UNIQUE COMMENT '业务订单号',
  student_id BIGINT NOT NULL,
  title VARCHAR(128) NOT NULL COMMENT '例如: 2026春季校服',
  amount_cent INT NOT NULL COMMENT '金额(分)',
  paid_cent INT NOT NULL DEFAULT 0,
  status VARCHAR(20) NOT NULL COMMENT 'PENDING/PAID/CLOSED/REFUNDED/PARTIAL_REFUNDED',
  deadline_at DATETIME NULL,
  paid_at DATETIME NULL,
  biz_type VARCHAR(32) NOT NULL DEFAULT 'UNIFORM',
  ext_json JSON NULL,
  created_by BIGINT NULL COMMENT '管理员ID',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_orders_student FOREIGN KEY (student_id) REFERENCES students(id),
  INDEX idx_student_status(student_id, status),
  INDEX idx_created_at(created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE payments (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  payment_no VARCHAR(64) NOT NULL UNIQUE,
  order_id BIGINT NOT NULL,
  order_no VARCHAR(64) NOT NULL,
  channel VARCHAR(20) NOT NULL DEFAULT 'WECHAT',
  mchid VARCHAR(32) NOT NULL,
  appid VARCHAR(32) NOT NULL,
  transaction_id VARCHAR(64) NULL COMMENT '微信支付单号',
  out_trade_no VARCHAR(64) NOT NULL COMMENT '商户支付单号',
  prepay_id VARCHAR(128) NULL,
  amount_cent INT NOT NULL,
  payer_openid VARCHAR(64) NOT NULL,
  status VARCHAR(20) NOT NULL COMMENT 'CREATED/SUCCESS/FAIL/CLOSED',
  success_at DATETIME NULL,
  fail_reason VARCHAR(255) NULL,
  raw_response JSON NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_payments_order FOREIGN KEY (order_id) REFERENCES orders(id),
  INDEX idx_order_id(order_id),
  INDEX idx_out_trade_no(out_trade_no),
  INDEX idx_transaction_id(transaction_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE refunds (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  refund_no VARCHAR(64) NOT NULL UNIQUE,
  order_id BIGINT NOT NULL,
  payment_id BIGINT NOT NULL,
  out_refund_no VARCHAR(64) NOT NULL,
  refund_id VARCHAR(64) NULL COMMENT '微信退款单号',
  reason VARCHAR(255) NULL,
  refund_cent INT NOT NULL,
  status VARCHAR(20) NOT NULL COMMENT 'PROCESSING/SUCCESS/FAIL/CLOSED',
  success_at DATETIME NULL,
  raw_response JSON NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_refunds_order FOREIGN KEY (order_id) REFERENCES orders(id),
  CONSTRAINT fk_refunds_payment FOREIGN KEY (payment_id) REFERENCES payments(id),
  INDEX idx_order_id(order_id),
  INDEX idx_payment_id(payment_id),
  INDEX idx_out_refund_no(out_refund_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE wechat_notify_logs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  notify_type VARCHAR(32) NOT NULL COMMENT 'PAY/REFUND',
  request_id VARCHAR(64) NULL,
  serial_no VARCHAR(64) NULL,
  signature VARCHAR(512) NULL,
  nonce VARCHAR(64) NULL,
  timestamp_str VARCHAR(32) NULL,
  body_text TEXT NOT NULL,
  decrypt_text TEXT NULL,
  process_status VARCHAR(20) NOT NULL DEFAULT 'INIT' COMMENT 'INIT/SUCCESS/FAIL/REPEAT',
  process_msg VARCHAR(255) NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_notify_type_created(notify_type, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 5. Go 项目结构建议（单体分层，易演进）

```text
.
├── cmd/
│   └── server/main.go
├── internal/
│   ├── api/              # gin handler
│   ├── service/          # 核心业务
│   ├── repo/             # DAO
│   ├── model/            # 实体/DTO
│   ├── wechatpay/        # 微信支付客户端封装
│   ├── middleware/       # JWT、日志、幂等等
│   └── pkg/
│       ├── xerr/
│       ├── xid/
│       └── xlog/
├── configs/
│   └── config.yaml
├── migrations/
└── go.mod
```

---

## 6. 关键接口设计（REST）

## 6.1 管理端
- `POST /admin/orders`：创建单个订单
- `POST /admin/orders/batch`：按班级批量建单
- `GET /admin/orders`：订单列表
- `POST /admin/orders/{orderNo}/close`：关闭未支付订单
- `POST /admin/refunds`：发起退款

## 6.2 学生端（小程序）
- `GET /api/me/orders`：我的订单列表
- `GET /api/me/orders/{orderNo}`：订单详情
- `POST /api/me/orders/{orderNo}/pay`：创建微信支付参数

## 6.3 微信回调
- `POST /api/v1/payments/wechat/notify`：支付结果通知
- `POST /api/v1/refunds/wechat/notify`：退款结果通知

---

## 7. 关键代码骨架（示例）

## 7.1 下单服务（核心逻辑）

```go
func (s *PaymentService) CreateJSAPIPay(ctx context.Context, studentID int64, orderNo string) (*PayParams, error) {
    order, err := s.orderRepo.GetByOrderNo(ctx, orderNo)
    if err != nil {
        return nil, err
    }
    if order.StudentID != studentID {
        return nil, ErrForbidden
    }
    if order.Status != "PENDING" {
        return nil, ErrOrderStatus
    }

    student, err := s.studentRepo.GetByID(ctx, studentID)
    if err != nil {
        return nil, err
    }
    if student.OpenID == "" {
        return nil, ErrOpenIDRequired
    }

    paymentNo := s.idGen.New("PAY")
    outTradeNo := paymentNo

    // 幂等：避免重复创建支付单
    if exist, _ := s.paymentRepo.GetByOutTradeNo(ctx, outTradeNo); exist != nil {
        return s.wechat.BuildPayParams(exist.PrepayID), nil
    }

    prepayResp, err := s.wechat.CreateJSAPIOrder(ctx, WechatPrepayReq{
        OutTradeNo:  outTradeNo,
        Description: order.Title,
        AmountCent:  order.AmountCent,
        PayerOpenID: student.OpenID,
        NotifyURL:   s.cfg.WechatPay.NotifyURL,
    })
    if err != nil {
        return nil, err
    }

    _ = s.paymentRepo.Create(ctx, &Payment{
        PaymentNo:  paymentNo,
        OrderID:    order.ID,
        OrderNo:    order.OrderNo,
        OutTradeNo: outTradeNo,
        PrepayID:   prepayResp.PrepayID,
        AmountCent: order.AmountCent,
        Status:     "CREATED",
    })

    return s.wechat.BuildPayParams(prepayResp.PrepayID), nil
}
```

## 7.2 微信支付回调处理（幂等 + 验签）

```go
func (h *WechatHandler) PayNotify(c *gin.Context) {
    body, _ := io.ReadAll(c.Request.Body)

    if err := h.wechat.VerifySignature(c.Request.Header, body); err != nil {
        c.JSON(http.StatusOK, gin.H{"code": "FAIL", "message": "invalid signature"})
        return
    }

    notify, err := h.wechat.DecryptNotify(body)
    if err != nil {
        c.JSON(http.StatusOK, gin.H{"code": "FAIL", "message": "decrypt failed"})
        return
    }

    // 幂等锁: Redis SETNX(out_trade_no, 1, 30s)
    locked := h.locker.TryLock(c, "wxpay_notify:"+notify.OutTradeNo, 30*time.Second)
    if !locked {
        c.JSON(http.StatusOK, gin.H{"code": "SUCCESS", "message": "duplicate"})
        return
    }
    defer h.locker.Unlock(c, "wxpay_notify:"+notify.OutTradeNo)

    err = h.paymentSvc.MarkPaymentSuccess(c, notify)
    if err != nil {
        c.JSON(http.StatusOK, gin.H{"code": "FAIL", "message": "process failed"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"code": "SUCCESS", "message": "成功"})
}
```

---

## 8. 安全与风控要点

- **金额可信源在后端**：前端传金额一律忽略，后端按订单金额支付。
- **回调验签必须做**：严格按微信支付 v3 证书验签。
- **幂等处理**：
  - 创建支付：按 `out_trade_no` 去重。
  - 回调处理：分布式锁 + 状态机校验。
- **状态机约束**：只允许 `PENDING -> PAID`，防止状态回滚。
- **审计日志**：保存回调原文、关键操作人、退款原因。
- **权限隔离**：管理端与学生端 JWT 使用不同 `aud`/`role`。

---

## 9. 性能与容量评估（2000学生）

- 峰值场景（统一收费通知后 10 分钟内集中支付）估算：
  - 假设 20% 学生在短时间支付 ≈ 400 笔。
  - 峰值 QPS 通常 < 30（足够低）。
- MySQL + Redis 单实例可支撑。
- 建议预留：应用层连接池、慢 SQL 监控、接口限流。

---

## 10. 部署与运维建议

- **配置管理**：`config.yaml` + 环境变量覆盖（密钥放环境变量）。
- **日志**：JSON 结构化日志，字段包含 `trace_id/order_no/payment_no`。
- **监控**：
  - 接口成功率、P95 延迟
  - 微信下单失败率
  - 回调处理失败告警
- **备份**：MySQL 每日全备 + Binlog，至少保留 30 天。

---

## 11. 开发里程碑（建议）

1. **MVP（1~2周）**：学生、订单、微信 JSAPI 下单、支付回调。
2. **增强（第3周）**：批量建单、后台筛选导出、退款。
3. **完善（第4周）**：监控告警、对账脚本、自动化测试。

---

## 12. 对账与财务闭环（强烈建议）

- 每日凌晨下载微信账单（交易单、资金账单）。
- 按 `out_trade_no` 与本地 `payments` 对账。
- 输出差异单（漏单、金额不一致、状态不一致）。
- 差异单自动进入人工处理队列。

---

## 13. 你可以直接照着做的技术选型

- Web: `gin`
- ORM: `gorm`（或 `sqlc`）
- DB: `MySQL 8.0`
- Cache/Lock: `Redis`
- Auth: `JWT`
- 微信支付 SDK: `wechatpay-apiv3` 官方 Go SDK
- Job: `asynq`（可选，用于通知、对账任务）

如果你愿意，我下一步可以给你：
1) 一份可直接运行的 **Go 项目脚手架**（含 `main.go`、路由、配置、数据库连接）；
2) 完整的 **支付下单/回调代码**（可复制粘贴）；
3) 一套 **Postman 接口集合** 和初始化 SQL。
