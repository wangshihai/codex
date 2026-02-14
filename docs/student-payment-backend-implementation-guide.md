# 学生支付后端实施手册（Go + 微信支付）

> 这份文档是给“马上开工”的开发手册，和设计文档配套使用。你可以把它当作 0 到 1 的执行清单。

## 1. 你要先准备什么

- 微信支付商户号（mchid）
- 小程序 appid
- APIv3 Key
- 商户私钥（apiclient_key.pem）
- 微信平台证书
- 一台 MySQL 8.0
- 一台 Redis

---

## 2. 第一天落地步骤

1. 建库并执行 DDL（students/orders/payments/refunds/wechat_notify_logs/reconcile_*）。
2. 初始化 Go 项目结构（api/service/repo/wechatpay/job）。
3. 打通基础接口：
   - `GET /healthz`
   - `POST /api/me/orders/{orderNo}/pay`
   - `POST /api/v1/payments/wechat/notify`
4. 配置微信支付参数，先跑沙箱或测试环境。
5. 用 1 个测试学生 + 1 笔 1 分钱订单联调支付回调。

---

## 3. 最小可用 API 清单（MVP）

### 学生端
- `GET /api/me/orders`
- `GET /api/me/orders/{orderNo}`
- `POST /api/me/orders/{orderNo}/pay`

### 管理端
- `POST /admin/orders`
- `POST /admin/orders/batch`
- `POST /admin/refunds`

### 支付回调
- `POST /api/v1/payments/wechat/notify`
- `POST /api/v1/refunds/wechat/notify`

---

## 4. 核心代码骨架（可直接抄）

## 4.1 main.go 启动骨架

```go
func main() {
    cfg := mustLoadConfig()

    db := mustInitMySQL(cfg.Database.DSN)
    rdb := mustInitRedis(cfg.Redis.Addr)

    orderRepo := repo.NewOrderRepo(db)
    paymentRepo := repo.NewPaymentRepo(db)

    wxClient := wechatpay.NewClient(cfg.WechatPay)

    paymentSvc := service.NewPaymentService(orderRepo, paymentRepo, wxClient, rdb)
    notifyHandler := api.NewWechatNotifyHandler(paymentSvc, wxClient)

    r := gin.Default()
    r.GET("/healthz", func(c *gin.Context) { c.JSON(200, gin.H{"ok": true}) })
    r.POST("/api/v1/payments/wechat/notify", notifyHandler.PayNotify)

    _ = r.Run(cfg.Server.Addr)
}
```

## 4.2 支付回调最小事务

```go
func (s *PaymentService) MarkPaymentSuccess(ctx context.Context, outTradeNo, transactionID string, payTime time.Time) error {
    return s.tx.WithTx(ctx, func(txCtx context.Context) error {
        p, err := s.paymentRepo.GetByOutTradeNoForUpdate(txCtx, outTradeNo)
        if err != nil { return err }
        if p == nil { return errors.New("payment not found") }
        if p.Status == "SUCCESS" { return nil } // 幂等

        if err := s.paymentRepo.MarkSuccess(txCtx, p.ID, transactionID, payTime); err != nil {
            return err
        }
        return s.orderRepo.MarkPaidByOrderID(txCtx, p.OrderID, payTime)
    })
}
```

---

## 5. 每日对账怎么做（简版）

1. 每天凌晨 01:10 拉取前一天微信账单。
2. 按 `out_trade_no` 对比本地 `payments`。
3. 写入 `reconcile_results`。
4. 差异写 `reconcile_issues`。
5. 发告警（企业微信机器人/邮件）。

差异类型建议：
- `LOCAL_MISSING`
- `WECHAT_MISSING`
- `AMOUNT_MISMATCH`
- `STATUS_MISMATCH`

---

## 6. 上线前检查清单

- [ ] 回调验签通过（错误签名应拒绝）
- [ ] 回调重复投递只处理一次
- [ ] 订单状态流转正确（PENDING->PAID）
- [ ] 退款后状态正确
- [ ] 对账任务可运行并产出结果
- [ ] 核心日志包含 trace_id/order_no/out_trade_no

---

## 7. 推荐里程碑

- 第 1 周：下单 + 支付回调打通。
- 第 2 周：退款 + 退款回调。
- 第 3 周：每日对账 + 差异工单。
- 第 4 周：监控告警 + 压测 + 上线。

---

## 8. 常见坑（你可以提前规避）

1. 用前端传的金额创建支付单（错误）→ 必须以后端订单金额为准。
2. 回调不验签（高危）→ 必须验签 + 解密。
3. 没有幂等（高频踩坑）→ `out_trade_no` 唯一 + 行锁。
4. 不做对账（后期财务对不上）→ 每日定时对账必须上线。

