# 学生校服支付后端（Go + 微信支付）详细设计（可落地版）

> 适用场景：学校统一收取校服费，约 2000 学生，学生通过微信小程序完成支付，管理员在后台建单/退款/对账。

---


## 0. 快速定位：每日对账看这里

你关心的“每日对账”在以下章节（本版已补全实现细节）：
- **3.4 每日自动对账**：业务流程总览。
- **4.2 + 4.3**：对账表结构 `reconcile_tasks/reconcile_results/reconcile_issues`。
- **8.4**：Go 定时任务代码示例（`RunDailyReconcile`）。
- **9**：对账规则与差异处理策略。
- **17（新增）**：每日对账实施方案（从调度、下载、解析、比对、落库到告警）。

---

## 1. 目标与非目标

### 1.1 目标
- 支持管理员创建单个/批量校服订单。
- 学生在小程序中查看并支付订单（微信 JSAPI）。
- 支持支付回调、退款回调、手动退款。
- 支持每日自动对账与差异工单。
- 支持审计追踪（谁创建订单、谁退款、回调原文）。

### 1.2 非目标（首期可不做）
- 不做复杂优惠券与满减。
- 不做多支付渠道（先只做微信支付）。
- 不做复杂分账（后续如需可扩展）。

---

## 2. 总体架构

```text
[微信小程序] --HTTPS/JWT--> [Go API (Gin)]
                                 |
                                 +-- OrderService
                                 +-- PaymentService (WeChat JSAPI)
                                 +-- RefundService
                                 +-- ReconcileService (对账)
                                 |
                                 +-- MySQL (核心事务数据)
                                 +-- Redis (幂等锁、缓存、限流)
                                 +-- Asynq/Cron (定时任务: 对账/催缴)

[WeChat Pay]
   |--- 下单API/退款API
   |--- 支付回调 --> /api/v1/payments/wechat/notify
   |--- 退款回调 --> /api/v1/refunds/wechat/notify
```

### 2.1 部署建议（2000 学生）
- Go API：2 实例（2C4G），Nginx/SLB 负载均衡。
- MySQL 8.0：单主 + 自动备份（全量 + binlog）。
- Redis：单实例即可（幂等键、锁、缓存）。
- 定时任务：和 API 同服务进程即可，或单独 worker。

---

## 3. 业务流程（端到端）

## 3.1 管理员建单
1. 管理员选择学生或班级，输入标题/金额/截止日。
2. 服务端校验：金额>0、截止时间合理、学生状态正常。
3. 写入 `orders`，状态 `PENDING`。
4. 记录操作日志 `audit_logs`（可选增强表）。

## 3.2 学生支付（JSAPI）
1. 小程序调用 `POST /api/me/orders/{orderNo}/pay`。
2. 服务端校验：订单归属、状态 `PENDING`、未超时。
3. 服务端调用微信 JSAPI 统一下单，得到 `prepay_id`。
4. 返回前端调起支付参数：`timeStamp/nonceStr/package/paySign/signType`。
5. 前端调起微信支付。
6. 微信异步通知服务端支付结果。
7. 服务端验签 + 解密 + 幂等处理 + 状态机更新。

## 3.3 退款
1. 管理员发起退款：`POST /admin/refunds`。
2. 校验可退金额（不能超过已支付未退款金额）。
3. 调微信退款 API。
4. 记录 `refunds` 为 `PROCESSING`。
5. 收到回调或主动查询后，置 `SUCCESS/FAIL`，并更新 `orders` 的退款相关状态。

## 3.4 每日自动对账
1. 凌晨定时下载微信交易账单（前一日）。
2. 与本地 `payments` 按 `out_trade_no` 对比。
3. 生成 `reconcile_results`：一致/本地缺失/微信缺失/金额不一致/状态不一致。
4. 差异进入 `reconcile_issues`，派给财务/运营处理。

---

## 4. 数据库设计（MySQL）

## 4.1 业务核心表
- `students`：学生主数据 + openid。
- `orders`：应付订单。
- `payments`：支付尝试流水。
- `refunds`：退款流水。
- `wechat_notify_logs`：微信回调审计。

## 4.2 对账相关表（新增）
- `reconcile_tasks`：每次对账任务。
- `reconcile_results`：逐笔对账结果。
- `reconcile_issues`：差异工单。

## 4.3 SQL DDL（含对账表）

```sql
CREATE TABLE students (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  student_no VARCHAR(64) NOT NULL UNIQUE COMMENT '学号',
  name VARCHAR(64) NOT NULL,
  class_name VARCHAR(64) NOT NULL,
  grade_name VARCHAR(64) NULL,
  openid VARCHAR(64) NULL UNIQUE COMMENT '小程序openid',
  phone VARCHAR(20) NULL,
  status TINYINT NOT NULL DEFAULT 1 COMMENT '1正常 0停用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_class_name(class_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no VARCHAR(64) NOT NULL UNIQUE COMMENT '业务订单号',
  student_id BIGINT NOT NULL,
  title VARCHAR(128) NOT NULL,
  biz_type VARCHAR(32) NOT NULL DEFAULT 'UNIFORM',
  amount_cent INT NOT NULL COMMENT '应付总额(分)',
  paid_cent INT NOT NULL DEFAULT 0 COMMENT '已支付金额(分)',
  refunded_cent INT NOT NULL DEFAULT 0 COMMENT '已退款金额(分)',
  status VARCHAR(24) NOT NULL COMMENT 'PENDING/PAID/CLOSED/PARTIAL_REFUNDED/REFUNDED',
  deadline_at DATETIME NULL,
  paid_at DATETIME NULL,
  ext_json JSON NULL,
  created_by BIGINT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_orders_student FOREIGN KEY (student_id) REFERENCES students(id),
  INDEX idx_student_status(student_id, status),
  INDEX idx_deadline(deadline_at),
  INDEX idx_created_at(created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE payments (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  payment_no VARCHAR(64) NOT NULL UNIQUE,
  order_id BIGINT NOT NULL,
  order_no VARCHAR(64) NOT NULL,
  channel VARCHAR(20) NOT NULL DEFAULT 'WECHAT',
  appid VARCHAR(32) NOT NULL,
  mchid VARCHAR(32) NOT NULL,
  out_trade_no VARCHAR(64) NOT NULL UNIQUE COMMENT '商户支付单号(幂等键)',
  transaction_id VARCHAR(64) NULL COMMENT '微信支付单号',
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
  INDEX idx_transaction_id(transaction_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE refunds (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  refund_no VARCHAR(64) NOT NULL UNIQUE,
  order_id BIGINT NOT NULL,
  payment_id BIGINT NOT NULL,
  out_refund_no VARCHAR(64) NOT NULL UNIQUE COMMENT '商户退款单号(幂等键)',
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
  INDEX idx_payment_id(payment_id)
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

CREATE TABLE reconcile_tasks (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  task_no VARCHAR(64) NOT NULL UNIQUE,
  bill_date DATE NOT NULL COMMENT '对账日期(账单日)',
  channel VARCHAR(20) NOT NULL DEFAULT 'WECHAT',
  status VARCHAR(20) NOT NULL COMMENT 'INIT/RUNNING/SUCCESS/PARTIAL_FAIL/FAIL',
  total_count INT NOT NULL DEFAULT 0,
  match_count INT NOT NULL DEFAULT 0,
  diff_count INT NOT NULL DEFAULT 0,
  remark VARCHAR(255) NULL,
  started_at DATETIME NULL,
  finished_at DATETIME NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_bill_date_channel (bill_date, channel)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE reconcile_results (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  task_id BIGINT NOT NULL,
  bill_date DATE NOT NULL,
  out_trade_no VARCHAR(64) NOT NULL,
  transaction_id VARCHAR(64) NULL,
  wechat_amount_cent INT NOT NULL DEFAULT 0,
  local_amount_cent INT NOT NULL DEFAULT 0,
  wechat_status VARCHAR(20) NULL,
  local_status VARCHAR(20) NULL,
  result_type VARCHAR(32) NOT NULL COMMENT 'MATCH/LOCAL_MISSING/WECHAT_MISSING/AMOUNT_MISMATCH/STATUS_MISMATCH',
  detail_json JSON NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_reconcile_results_task FOREIGN KEY (task_id) REFERENCES reconcile_tasks(id),
  INDEX idx_task_result(task_id, result_type),
  INDEX idx_out_trade_no(out_trade_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE reconcile_issues (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  task_id BIGINT NOT NULL,
  result_id BIGINT NOT NULL,
  issue_no VARCHAR(64) NOT NULL UNIQUE,
  issue_type VARCHAR(32) NOT NULL,
  issue_status VARCHAR(20) NOT NULL DEFAULT 'OPEN' COMMENT 'OPEN/PROCESSING/RESOLVED/CLOSED',
  handler VARCHAR(64) NULL,
  resolution VARCHAR(255) NULL,
  resolved_at DATETIME NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_reconcile_issues_task FOREIGN KEY (task_id) REFERENCES reconcile_tasks(id),
  CONSTRAINT fk_reconcile_issues_result FOREIGN KEY (result_id) REFERENCES reconcile_results(id),
  INDEX idx_issue_status(issue_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 5. 状态机设计（非常关键）

### 5.1 订单状态 `orders.status`
- `PENDING`：待支付。
- `PAID`：已支付。
- `CLOSED`：已关闭（超时/人工关闭）。
- `PARTIAL_REFUNDED`：部分退款。
- `REFUNDED`：全额退款。

**合法流转**
- `PENDING -> PAID`
- `PENDING -> CLOSED`
- `PAID -> PARTIAL_REFUNDED`
- `PAID/PARTIAL_REFUNDED -> REFUNDED`

### 5.2 支付状态 `payments.status`
- `CREATED -> SUCCESS/FAIL/CLOSED`

### 5.3 退款状态 `refunds.status`
- `PROCESSING -> SUCCESS/FAIL/CLOSED`

> 建议在 service 层封装状态流转函数，所有写状态必须经过统一校验，不允许 DAO 直接任意更新。

---

## 6. Go 项目结构（含职责）

```text
.
├── cmd/server/main.go
├── configs/config.yaml
├── internal
│   ├── api
│   │   ├── admin_order_handler.go
│   │   ├── student_order_handler.go
│   │   ├── wechat_notify_handler.go
│   │   └── admin_refund_handler.go
│   ├── service
│   │   ├── order_service.go
│   │   ├── payment_service.go
│   │   ├── refund_service.go
│   │   └── reconcile_service.go
│   ├── repo
│   │   ├── order_repo.go
│   │   ├── payment_repo.go
│   │   ├── refund_repo.go
│   │   └── reconcile_repo.go
│   ├── wechatpay
│   │   ├── client.go
│   │   ├── verify.go
│   │   └── types.go
│   ├── job
│   │   └── reconcile_job.go
│   └── middleware
│       ├── auth.go
│       ├── trace.go
│       └── rate_limit.go
└── migrations
```

---

## 7. 核心接口定义（含请求/响应示例）

## 7.1 学生侧：创建支付
`POST /api/me/orders/{orderNo}/pay`

响应：
```json
{
  "timeStamp": "1710000000",
  "nonceStr": "abc123",
  "package": "prepay_id=wx201410272009395522657a690389285100",
  "signType": "RSA",
  "paySign": "xxxx"
}
```

## 7.2 微信支付回调
`POST /api/v1/payments/wechat/notify`
- Header 带签名字段；Body 为加密资源。
- 返回必须是微信要求格式：
```json
{"code":"SUCCESS","message":"成功"}
```

## 7.3 管理侧：发起退款
`POST /admin/refunds`

请求：
```json
{
  "orderNo": "ORD202601010001",
  "refundCent": 5000,
  "reason": "尺码不合适"
}
```

---

## 8. 关键代码示例（含注解）

## 8.1 `payment_service.go`：创建 JSAPI 支付

```go
package service

import (
	"context"
	"errors"
	"time"
)

// PaymentService 支付业务服务
type PaymentService struct {
	orderRepo   OrderRepo
	studentRepo StudentRepo
	paymentRepo PaymentRepo
	wechat      WechatClient
	idGen       IDGenerator
	cfg         Config
}

// CreateJSAPIPay 为学生创建微信 JSAPI 支付参数
func (s *PaymentService) CreateJSAPIPay(ctx context.Context, studentID int64, orderNo string) (*PayParams, error) {
	// 1) 查询订单并做归属校验
	order, err := s.orderRepo.GetByOrderNo(ctx, orderNo)
	if err != nil {
		return nil, err
	}
	if order.StudentID != studentID {
		return nil, errors.New("forbidden")
	}

	// 2) 状态 + 截止时间校验
	if order.Status != "PENDING" {
		return nil, errors.New("order status invalid")
	}
	if order.DeadlineAt != nil && order.DeadlineAt.Before(time.Now()) {
		return nil, errors.New("order expired")
	}

	// 3) 获取学生 openid（微信 JSAPI 必填）
	stu, err := s.studentRepo.GetByID(ctx, studentID)
	if err != nil {
		return nil, err
	}
	if stu.OpenID == "" {
		return nil, errors.New("openid required")
	}

	// 4) 生成商户单号（强幂等建议：一个订单同一时刻只允许一个有效支付单）
	outTradeNo := s.idGen.New("PAY")

	// 5) 调微信下单
	prepay, err := s.wechat.CreateJSAPIOrder(ctx, WechatPrepayReq{
		OutTradeNo:  outTradeNo,
		Description: order.Title,
		AmountCent:  order.AmountCent,
		PayerOpenID: stu.OpenID,
		NotifyURL:   s.cfg.WechatPay.NotifyURL,
	})
	if err != nil {
		return nil, err
	}

	// 6) 落支付流水
	err = s.paymentRepo.Create(ctx, &Payment{
		PaymentNo:  s.idGen.New("PMT"),
		OrderID:    order.ID,
		OrderNo:    order.OrderNo,
		OutTradeNo: outTradeNo,
		PrepayID:   prepay.PrepayID,
		AmountCent: order.AmountCent,
		PayerOpenID: stu.OpenID,
		Status:     "CREATED",
	})
	if err != nil {
		return nil, err
	}

	// 7) 返回小程序调起参数
	return s.wechat.BuildPayParams(prepay.PrepayID), nil
}
```

## 8.2 `wechat_notify_handler.go`：支付回调处理

```go
package api

import (
	"io"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

// PayNotify 微信支付回调入口
func (h *WechatNotifyHandler) PayNotify(c *gin.Context) {
	body, _ := io.ReadAll(c.Request.Body)

	// 1) 记录原始回调日志（建议先记日志，再处理）
	logID, _ := h.notifyLogRepo.CreateRaw(c, "PAY", c.Request.Header, body)

	// 2) 验签（微信支付v3）
	if err := h.wechat.VerifySignature(c.Request.Header, body); err != nil {
		h.notifyLogRepo.Mark(c, logID, "FAIL", "invalid signature")
		c.JSON(http.StatusOK, gin.H{"code": "FAIL", "message": "invalid signature"})
		return
	}

	// 3) 解密 resource
	notify, err := h.wechat.DecryptPayNotify(body)
	if err != nil {
		h.notifyLogRepo.Mark(c, logID, "FAIL", "decrypt failed")
		c.JSON(http.StatusOK, gin.H{"code": "FAIL", "message": "decrypt failed"})
		return
	}

	// 4) 防重：分布式锁（同一个 out_trade_no 只处理一次）
	lockKey := "wxpay_notify:" + notify.OutTradeNo
	if ok := h.locker.TryLock(c, lockKey, 30*time.Second); !ok {
		h.notifyLogRepo.Mark(c, logID, "REPEAT", "duplicate notify")
		c.JSON(http.StatusOK, gin.H{"code": "SUCCESS", "message": "duplicate"})
		return
	}
	defer h.locker.Unlock(c, lockKey)

	// 5) 事务更新 payment + order 状态
	if err := h.paymentSvc.MarkPaymentSuccess(c, notify); err != nil {
		h.notifyLogRepo.Mark(c, logID, "FAIL", err.Error())
		c.JSON(http.StatusOK, gin.H{"code": "FAIL", "message": "process failed"})
		return
	}

	h.notifyLogRepo.Mark(c, logID, "SUCCESS", "ok")
	c.JSON(http.StatusOK, gin.H{"code": "SUCCESS", "message": "成功"})
}
```

## 8.3 `payment_service.go`：回调落库事务（重点）

```go
func (s *PaymentService) MarkPaymentSuccess(ctx context.Context, n PayNotify) error {
	return s.txManager.WithTx(ctx, func(txCtx context.Context) error {
		p, err := s.paymentRepo.GetByOutTradeNoForUpdate(txCtx, n.OutTradeNo)
		if err != nil {
			return err
		}
		if p == nil {
			return errors.New("payment not found")
		}

		// 幂等：已经成功则直接返回
		if p.Status == "SUCCESS" {
			return nil
		}

		// 更新支付状态
		if err := s.paymentRepo.MarkSuccess(txCtx, p.ID, n.TransactionID, n.SuccessTime); err != nil {
			return err
		}

		// 锁订单并更新
		o, err := s.orderRepo.GetByIDForUpdate(txCtx, p.OrderID)
		if err != nil {
			return err
		}
		if o.Status != "PENDING" {
			// 若订单已非待支付，不回滚支付状态，记录异常待人工
			return nil
		}
		return s.orderRepo.MarkPaid(txCtx, o.ID, o.AmountCent, n.SuccessTime)
	})
}
```

## 8.4 `reconcile_job.go`：每日对账任务（核心示例）

```go
package job

import (
	"context"
	"time"
)

// RunDailyReconcile 每日凌晨运行：对前一日账单进行对账
func (j *ReconcileJob) RunDailyReconcile(ctx context.Context) error {
	billDate := time.Now().AddDate(0, 0, -1).Format("2006-01-02")

	// 1) 创建任务记录
	task, err := j.reconcileSvc.StartTask(ctx, billDate)
	if err != nil {
		return err
	}

	// 2) 拉取微信账单（可先下载文件，再解析成结构化记录）
	wxRows, err := j.wechat.DownloadAndParseTradeBill(ctx, billDate)
	if err != nil {
		_ = j.reconcileSvc.FailTask(ctx, task.ID, err.Error())
		return err
	}

	// 3) 逐笔比对
	for _, row := range wxRows {
		_ = j.reconcileSvc.CompareOne(ctx, task.ID, row)
	}

	// 4) 补查：本地成功但微信账单未出现（可能漏单/账单延迟）
	_ = j.reconcileSvc.FindLocalMissingInWechat(ctx, task.ID, billDate)

	// 5) 汇总任务状态
	return j.reconcileSvc.FinishTask(ctx, task.ID)
}
```

---

## 9. 对账规则（建议直接照搬）

按 `out_trade_no` 主键对比：

1. 本地有，微信无：`WECHAT_MISSING`
   - 原因：账单延迟、单号错误、微信侧未成功。
2. 微信有，本地无：`LOCAL_MISSING`
   - 原因：回调丢失、入库失败。
3. 金额不一致：`AMOUNT_MISMATCH`
   - 必须人工确认，通常是数据异常。
4. 状态不一致：`STATUS_MISMATCH`
   - 常见：微信成功，本地仍 CREATED。
5. 全部一致：`MATCH`

处理建议：
- `LOCAL_MISSING`：自动补单（调用订单查询 API 二次确认后修复）。
- `WECHAT_MISSING`：延迟一天重试再升级人工。
- `AMOUNT_MISMATCH`：直接人工优先处理。

---

## 10. 幂等与一致性策略

### 10.1 幂等点
- 下单幂等：`out_trade_no` 唯一。
- 支付回调幂等：`out_trade_no + 分布式锁 + DB 行锁`。
- 退款幂等：`out_refund_no` 唯一。

### 10.2 一致性
- 关键更新都在事务内（支付状态 + 订单状态）。
- 回调先记日志后处理，任何失败可重放。
- 对账任务兜底修复漏单。

---

## 11. 安全与合规

- 前端金额不可信，以后端订单金额为准。
- 微信回调必须验签、验时间戳、验商户号。
- 密钥（APIv3 Key、私钥）必须环境变量/密管系统，不入库不入 git。
- 管理端和学生端 JWT 分开 audience / role。
- 重要接口加限流与操作审计。

---

## 12. 配置样例（`config.yaml`）

```yaml
server:
  addr: ":8080"
  read_timeout: 5s
  write_timeout: 10s

database:
  dsn: "user:pass@tcp(127.0.0.1:3306)/school_pay?charset=utf8mb4&parseTime=True&loc=Local"
  max_open_conns: 50
  max_idle_conns: 10

redis:
  addr: "127.0.0.1:6379"
  db: 0

wechat_pay:
  appid: "wx123"
  mchid: "1900000109"
  notify_url: "https://pay.yourschool.com/api/v1/payments/wechat/notify"
  refund_notify_url: "https://pay.yourschool.com/api/v1/refunds/wechat/notify"
  serial_no: "XXXX"
  private_key_path: "/etc/secrets/apiclient_key.pem"
  platform_cert_path: "/etc/secrets/wechat_platform_cert.pem"
```

---

## 13. 性能容量与优化建议（2000 学生）

- 统一通知后的峰值，估算 QPS < 30，系统压力不大。
- 优化重点不是“吞吐”，而是“稳定与一致性”：
  - 减少支付失败重试。
  - 回调处理快速应答。
  - 对账任务可追溯。
- 索引优化：
  - `payments.out_trade_no` 唯一索引（必备）。
  - `orders(student_id, status)` 复合索引。
  - 对账结果按 `task_id + result_type` 索引。

---

## 14. 里程碑（4周建议）

1. 第1周：学生、订单、登录、基础后台。
2. 第2周：微信下单 + 支付回调 + 幂等。
3. 第3周：退款 + 退款回调 + 操作审计。
4. 第4周：每日对账任务 + 差异工单 + 监控告警。

---

## 15. 测试清单（上线前）

### 功能测试
- 正常下单支付。
- 重复点击支付按钮（幂等）。
- 回调重复推送（幂等）。
- 订单过期后支付。
- 部分退款、全额退款。

### 异常测试
- 微信回调签名错误。
- 回调解密失败。
- DB 事务中断后重试。
- 对账任务下载失败重试。

### 压测建议
- 模拟 500 并发调用 `pay` 接口，观察错误率与耗时。
- 模拟 1000 条重复回调，确保只处理一次。

---

## 16. 下一步可直接交付内容

如果你需要，我可以继续给你以下“可运行代码版”：
1. 完整 `main.go`（路由、配置加载、MySQL/Redis 初始化）。
2. `payment_service.go` + `wechat_notify_handler.go` 的可编译版本。
3. `reconcile_job.go` + 对账 CSV 解析代码。
4. 一份 Postman 集合 + 初始化 SQL + 演示数据。


---

## 17. 每日对账实施方案（落地步骤）

下面给你一个可以直接按开发任务拆分的“每日对账”实现方案。

### 17.1 调度策略
- 每天 `01:10` 执行前一日对账（避开微信账单生成延迟窗口）。
- 同一天对账任务只允许一个实例执行（`uk_bill_date_channel` + Redis 分布式锁）。
- 失败自动重试 3 次（指数退避：1m/5m/15m）。

Cron 示例：
```go
// 每日 01:10
spec := "0 10 1 * * *"
_, _ = c.AddFunc(spec, func() {
    _ = reconcileJob.RunDailyReconcile(context.Background())
})
```

### 17.2 账单下载与解析
1. 调微信“下载交易账单”接口拿到下载链接。
2. 拉取账单文件（通常 CSV/GZIP）。
3. 解析出结构化字段：
   - `out_trade_no`
   - `transaction_id`
   - `trade_state`
   - `total_fee`
   - `success_time`
4. 落临时内存 map（key=`out_trade_no`）用于比对。

建议结构体：
```go
type WxBillRow struct {
    OutTradeNo    string
    TransactionID string
    TradeState    string // SUCCESS/REFUND/CLOSED...
    AmountCent    int
    SuccessTime   time.Time
}
```

### 17.3 比对逻辑（逐笔）
- 维度：`out_trade_no`。
- 规则：
  - 本地无、微信有 => `LOCAL_MISSING`
  - 本地有、微信无 => `WECHAT_MISSING`
  - 金额不同 => `AMOUNT_MISMATCH`
  - 状态不同 => `STATUS_MISMATCH`
  - 否则 => `MATCH`

比对伪代码：
```go
func compare(local Payment, wx WxBillRow) string {
    if local.ID == 0 { return "LOCAL_MISSING" }
    if wx.OutTradeNo == "" { return "WECHAT_MISSING" }
    if local.AmountCent != wx.AmountCent { return "AMOUNT_MISMATCH" }
    if mapLocalStatus(local.Status) != wx.TradeState { return "STATUS_MISMATCH" }
    return "MATCH"
}
```

### 17.4 落库与差异工单
- 每笔比对都写 `reconcile_results`。
- 非 `MATCH` 自动创建 `reconcile_issues`。
- 任务结束更新 `reconcile_tasks` 汇总统计：
  - `total_count`
  - `match_count`
  - `diff_count`
  - `status`

状态建议：
- 无差异：`SUCCESS`
- 有差异但任务完成：`PARTIAL_FAIL`
- 下载或解析失败：`FAIL`

### 17.5 自动修复（建议先做两类）
1. `LOCAL_MISSING` 自动补单
   - 调微信“查单接口”二次确认。
   - 若确认成功：补写 `payments` + 更新 `orders`。
2. `STATUS_MISMATCH` 自动纠偏
   - 仅当微信成功、本地非成功时才自动改（保守修复）。

> `AMOUNT_MISMATCH` 不建议自动修复，直接人工处理。

### 17.6 告警与报表
- 告警触发条件：
  - `diff_count > 0`
  - 任务 `FAIL`
  - 连续 2 天 `WECHAT_MISSING` 未收敛
- 告警渠道：企业微信机器人 / 邮件。
- 每日输出报表字段：
  - 账单日
  - 总笔数
  - 一致笔数
  - 差异笔数
  - 差异率

### 17.7 管理端接口（建议新增）
- `GET /admin/reconcile/tasks?billDate=2026-01-01`
- `GET /admin/reconcile/tasks/{taskNo}/results`
- `GET /admin/reconcile/issues?status=OPEN`
- `POST /admin/reconcile/issues/{issueNo}/resolve`

示例响应：
```json
{
  "taskNo": "REC20260102",
  "billDate": "2026-01-01",
  "status": "PARTIAL_FAIL",
  "totalCount": 398,
  "matchCount": 392,
  "diffCount": 6
}
```

### 17.8 SQL 查询模板（运维常用）

1) 查询当天对账任务汇总
```sql
SELECT task_no, bill_date, status, total_count, match_count, diff_count, started_at, finished_at
FROM reconcile_tasks
WHERE bill_date = '2026-01-01';
```

2) 查询未处理差异工单
```sql
SELECT issue_no, issue_type, issue_status, created_at
FROM reconcile_issues
WHERE issue_status IN ('OPEN', 'PROCESSING')
ORDER BY created_at ASC;
```

3) 查询金额不一致明细
```sql
SELECT rr.out_trade_no, rr.local_amount_cent, rr.wechat_amount_cent, rr.result_type
FROM reconcile_results rr
JOIN reconcile_tasks rt ON rt.id = rr.task_id
WHERE rt.bill_date = '2026-01-01'
  AND rr.result_type = 'AMOUNT_MISMATCH';
```
