# BEpusdt 支付完成通知接口详解

## 📋 流程概述

从发卡到支付再到通知的完整流程中，BEpusdt提供了以下通知接口：

```
商户网站 → 创建订单 → 用户支付 → 区块链监听 → 支付确认 → 通知接口触发
    ↓                                                            ↓
  notify_url                                                多种通知方式
    ↓                                                            ↓
  HTTP回调 ←─────────────────────────────────────────────────┘
    ↓
  商户系统更新订单状态
```

## 🎯 通知接口分类

### 1. HTTP回调通知 (主要接口)

#### 接口调用者
- **调用方**: BEpusdt服务器
- **接收方**: 商户网站
- **调用时机**: 支付状态变更时

#### 调用时机和事件

| 时机 | 状态 | 触发条件 | 调用位置 |
|------|------|----------|----------|
| **支付成功** | `status = 2` | 交易确认完成 | `app/task/transfer.go:58` |
| **订单超时** | `status = 3` | 订单过期 | `app/task/transfer.go:223` |
| **支付失败** | `status = 6` | 交易确认失败 | `app/task/transfer.go:253` |

#### 调用方式
```go
// 代码位置: app/task/transfer.go:58
func markFinalConfirmed(o model.TradeOrders) {
    model.PushWebhookEvent(model.WebhookEventOrderPaid, o)  // 触发Webhook事件
    o.SetSuccess()                                          // 更新订单状态
    go notify.Handle(o)         // 调用HTTP回调通知          ← 主要通知接口
    go bot2.SendTradeSuccMsg(o) // 发送Telegram通知(可选)
}
```

#### 请求方式

**EPUSDT标准回调**:
- **方法**: `POST`
- **Content-Type**: `application/json`
- **超时**: 10秒

**易支付兼容回调**:
- **方法**: `GET`
- **参数**: URL查询参数
- **超时**: 5秒

#### 接口要求

商户需要实现一个接收回调的接口，例如：
```
POST https://your-website.com/payment/notify
```

#### 回调数据格式

**EPUSDT标准格式**:
```json
{
    "trade_id": "trade_20241217_001",           // BEpusdt订单ID
    "order_id": "merchant_001",                // 商户订单ID
    "amount": 100.00,                          // 订单金额(CNY)
    "actual_amount": "14.285714",              // 实际支付金额(USDT)
    "token": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxx",  // 收款地址
    "block_transaction_id": "0x123...abc",     // 区块交易哈希
    "status": 2,                               // 订单状态
    "signature": "generated_signature_here"     // 签名验证
}
```

#### 响应要求

- **成功响应**: `ok` (EPUSDT) / `success` (易支付)
- **失败处理**: BEpusdt会自动重试，最多重试`NotifyMaxRetry`次

### 2. Webhook事件通知 (辅助接口)

#### 接口调用者
- **调用方**: BEpusdt服务器
- **接收方**: 商户配置的Webhook URL
- **调用时机**: 订单生命周期各个阶段

#### 支持的事件类型

```go
// 代码位置: app/model/webhook.go:20-24
const (
    WebhookEventOrderCreate  = "order.create"  // 订单创建
    WebhookEventOrderPaid    = "order.paid"    // 订单支付
    WebhookEventOrderTimeout = "order.timeout" // 订单超时
    WebhookEventOrderCancel  = "order.cancel"  // 订单取消
    WebhookEventOrderFailed  = "order.failed"  // 订单失败
)
```

#### 触发位置和时机

| 事件 | 触发时机 | 代码位置 |
|------|----------|----------|
| `order.create` | 订单创建成功 | `app/web/order.go:107` |
| `order.paid` | 支付确认完成 | `app/task/transfer.go:54` |
| `order.timeout` | 订单过期 | `app/task/transfer.go:223` |
| `order.cancel` | 订单取消 | `app/web/epusdt.go:165` |
| `order.failed` | 支付失败 | `app/task/transfer.go:253` |

#### 调用方式
```go
// 代码位置: app/model/webhook.go:76-92
func PushWebhookEvent(event string, data any) {
    go func() {
        // 1. 获取Webhook URL
        var url = conf.GetWebhookUrl()
        if url == "" {
            return
        }

        // 2. 序列化事件数据
        bytes, _ := json.Marshal(data)

        // 3. 创建Webhook记录并加入队列
        var w = Webhook{
            Status: WebhookStatusWait,
            Url:    url,
            Event:  event,
            Data:   bytes,
        }

        if err := DB.Create(&w).Error; err == nil {
            WebhookHandleQueue.In <- w  // 加入处理队列
        }
    }()
}
```

#### 配置要求
```toml
# conf.toml
webhook_url = "https://your-website.com/webhook"
```

#### 事件数据格式
```json
{
    "event": "order.paid",
    "data": {
        "id": 123,
        "trade_id": "trade_20241217_001",
        "order_id": "merchant_001",
        "status": 2,
        "amount": "14.285714",
        "payment_time": "2024-12-17T15:30:00Z"
    }
}
```

### 3. 主动查询接口 (补充方式)

#### 接口调用者
- **调用方**: 商户网站/前端应用
- **接收方**: BEpusdt服务器
- **调用时机**: 主动查询订单状态

#### 接口地址
```
GET /pay/check-status/{trade_id}
```

#### 响应数据
```json
{
    "trade_id": "trade_20241217_001",
    "trade_hash": "0x123...abc",
    "status": 2,
    "return_url": "https://merchant.com/success?params=..."
}
```

## 🚀 完整的通知流程

### 步骤1: 创建订单 (商户 → BEpusdt)

```bash
POST /api/v1/order/create-transaction
{
    "order_id": "merchant_001",
    "amount": 100.00,
    "trade_type": "usdt.trc20",
    "notify_url": "https://your-website.com/payment/notify",  // ← 关键配置
    "redirect_url": "https://your-website.com/payment/success",
    "name": "商品购买"
}
```

### 步骤2: 用户支付 (用户 → 区块链)

用户向指定地址转账指定金额的USDT/USDC。

### 步骤3: 区块链监听 (BEpusdt内部)

1. **区块扫描**: 实时扫描区块链新区块
2. **交易解析**: 提取转账交易信息
3. **订单匹配**: 匹配地址+金额+交易类型
4. **状态确认**: 验证交易最终确认状态

### 步骤4: 通知触发 (BEpusdt → 商户)

#### 4.1 支付成功通知流程

```go
// 代码位置: app/task/transfer.go:53-60
func markFinalConfirmed(o model.TradeOrders) {
    // 1. 触发Webhook事件
    model.PushWebhookEvent(model.WebhookEventOrderPaid, o)

    // 2. 更新订单状态为成功
    o.SetSuccess()

    // 3. 并发执行通知
    go notify.Handle(o)         // ← HTTP回调通知
    go bot2.SendTradeSuccMsg(o) // ← Telegram通知(可选)
}
```

#### 4.2 HTTP回调执行流程

```go
// 代码位置: app/web/notify/notify.go:33-49
func Handle(order model.TradeOrders) {
    if order.Status != model.OrderStatusSuccess {
        return
    }

    ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
    defer cancel()

    if order.ApiType == model.OrderApiTypeEpay {
        epay(ctx, order)    // 易支付兼容回调
        return
    }

    epusdt(ctx, order)      // EPUSDT标准回调
}
```

#### 4.3 EPUSDT回调执行

```go
// 代码位置: app/web/notify/notify.go:99-172
func epusdt(ctx context.Context, order model.TradeOrders) {
    // 1. 构建回调数据
    data := make(map[string]interface{})
    body := EpNotify{
        TradeId:            order.TradeId,
        OrderId:            order.OrderId,
        Amount:             order.Money,
        ActualAmount:       order.Amount,
        Token:              order.Address,
        BlockTransactionId: order.TradeHash,
        Status:             order.Status,
    }

    // 2. 生成签名
    jsonBody, _ := json.Marshal(body)
    json.Unmarshal(jsonBody, &data)
    body.Signature = help.EpusdtSign(data, conf.GetAuthToken())

    // 3. 发送HTTP请求
    finalJsonBody, _ := json.Marshal(body)
    client := http.Client{Timeout: time.Second * 5}
    req, _ := http.NewRequestWithContext(ctx, "POST", order.NotifyUrl, strings.NewReader(string(finalJsonBody)))

    // 4. 设置请求头
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Powered-By", "https://github.com/v03413/bepusdt")
    req.Header.Set("User-Agent", "BEpusdt/"+app.Version)

    // 5. 发送请求并处理响应
    resp, err := client.Do(req)
    if err != nil {
        markNotifyFail(order, err.Error())
        return
    }
    defer resp.Body.Close()

    // 6. 验证响应
    if resp.StatusCode != 200 {
        markNotifyFail(order, "resp.StatusCode != 200")
        return
    }

    all, _ := io.ReadAll(resp.Body)
    if string(all) != "ok" {
        markNotifyFail(order, fmt.Sprintf("body != ok (%s)", string(all)))
        return
    }

    // 7. 标记回调成功
    err = order.SetNotifyState(model.OrderNotifyStateSucc)
}
```

### 步骤5: 商户处理 (商户系统)

```php
<?php
// payment/notify.php - 商户回调处理示例
$data = json_decode(file_get_contents('php://input'), true);

// 1. 验证签名
$signature = $data['signature'];
unset($data['signature']);
ksort($data);
$stringToSign = implode('&', array_map(function($k, $v) {
    return $k . '=' . $v;
}, array_keys($data), $data));

$calculatedSignature = hash_hmac('sha256', $stringToSign, 'your_auth_token');
if ($signature !== $calculatedSignature) {
    die('invalid signature');
}

// 2. 根据状态处理订单
switch ($data['status']) {
    case 2: // 支付成功
        updateOrderStatus($data['order_id'], 'paid');
        deliverProduct($data['order_id']);
        echo 'ok';
        break;

    case 3: // 订单超时
        updateOrderStatus($data['order_id'], 'expired');
        echo 'ok';
        break;

    case 6: // 支付失败
        updateOrderStatus($data['order_id'], 'failed');
        echo 'ok';
        break;
}
?>
```

## 🔄 重试机制

### 失败重试逻辑

BEpusdt实现了完善的重试机制：

1. **立即重试**: 首次失败后立即重试
2. **指数退避**: 后续重试间隔逐渐增加 (1分钟, 2分钟, 4分钟, 8分钟...)
3. **最大重试次数**: 由`NotifyMaxRetry`配置控制，默认10次
4. **最终失败**: 超过重试次数后标记为失败状态

### 重试配置
```toml
# conf.toml
notify_max_retry = 10  # 最大重试次数
```

## 📊 通知状态跟踪

### 数据库记录

所有通知都会在数据库中记录：

```sql
-- 订单通知状态
SELECT order_id, notify_num, notify_state FROM trade_orders;

-- Webhook事件记录
SELECT event, status, url, created_at FROM bep_webhook;

-- 通知记录
SELECT txid, created_at FROM notify_records;
```

### 日志监控

```bash
# 查看通知相关日志
grep "订单通知" /var/log/bepusdt/app.log
grep "Webhook" /var/log/bepusdt/app.log
grep "Notify" /var/log/bepusdt/app.log
```

## ⚠️ 注意事项

### 1. 安全要求
- **HTTPS强制**: 回调接口必须使用HTTPS
- **签名验证**: 必须验证请求签名，防止伪造请求
- **IP白名单**: 建议限制只允许BEpusdt服务器IP访问
- **重复处理**: 需要处理重复通知，确保幂等性

### 2. 性能要求
- **快速响应**: 接口响应时间必须在5秒内
- **异步处理**: 业务逻辑应该异步执行，不阻塞响应
- **错误处理**: 妥善处理各种异常情况

### 3. 可靠性要求
- **幂等设计**: 同一订单的多次通知应该产生相同结果
- **状态检查**: 处理前检查订单当前状态，避免重复处理
- **日志记录**: 详细记录所有通知操作，便于排查问题

## ✅ 总结

BEpusdt提供了完整的三层通知机制：

1. **HTTP回调** - 主要的通知方式，支付成功时自动调用商户接口
2. **Webhook事件** - 辅助通知方式，覆盖订单全生命周期
3. **主动查询** - 补充方式，商户可随时查询订单状态

商户只需要实现一个接收回调的接口，就能获得完整的支付通知功能，实现从发卡到支付到发货的完整业务流程。