# BEpusdt 网站通知功能使用指南

## 📋 概述

BEpusdt已经内置了完善的网站通知功能，无需依赖Telegram Bot即可实现完整的支付结果通知。本指南详细说明现有功能和配置方法。

## 🎯 现有网站通知功能

### 1. HTTP回调通知 (主要通知方式)

#### 主要作用
- **支付结果通知**：支付成功、失败、超时时的实时通知
- **商户系统集成**：与商户网站/APP无缝集成
- **订单状态同步**：保持商户系统订单状态最新
- **业务流程触发**：自动触发发货、服务等业务流程

#### 配置方式
在创建订单时通过API参数设置：

```json
POST /api/v1/order/create-transaction
{
    "order_id": "merchant_001",
    "amount": 100.00,
    "trade_type": "usdt.trc20",
    "notify_url": "https://your-website.com/payment/notify",  // 回调接口
    "redirect_url": "https://your-website.com/payment/success", // 成功跳转页面
    "name": "商品购买"
}
```

#### 回调数据格式
```json
{
    "trade_id": "trade_20241217_001",           // BEpusdt订单ID
    "order_id": "merchant_001",                // 商户订单ID
    "amount": 100.00,                          // 订单金额(CNY)
    "actual_amount": "14.285714",              // 实际支付金额(USDT)
    "token": "TXxxxxxxxxxxxxxxxxxxxxxxxxxxx",  // 收款地址
    "block_transaction_id": "0x123...abc",     // 区块交易哈希
    "status": 2,                               // 订单状态(2=成功)
    "signature": "generated_signature_here"     // 签名验证
}
```

#### 状态码说明
- `1` - 等待支付
- `2` - 支付成功
- `3` - 订单超时
- `4` - 订单取消
- `6` - 支付失败

#### 服务端实现示例
```php
<?php
// payment/notify.php - PHP示例
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

// 2. 处理订单
if ($data['status'] == 2) {
    // 支付成功 - 执行业务逻辑
    updateOrderStatus($data['order_id'], 'paid');
    deliverProduct($data['order_id']);
    echo 'ok';
} else {
    // 其他状态处理
    echo 'ok';
}
?>
```

```javascript
// Node.js示例
const express = require('express');
const crypto = require('crypto');
const app = express();

app.post('/payment/notify', express.json(), (req, res) => {
    const { signature, ...data } = req.body;

    // 验证签名
    const stringToSign = Object.keys(data)
        .sort()
        .map(key => `${key}=${data[key]}`)
        .join('&');

    const calculatedSignature = crypto
        .createHmac('sha256', 'your_auth_token')
        .update(stringToSign)
        .digest('hex');

    if (signature !== calculatedSignature) {
        return res.status(400).send('invalid signature');
    }

    // 处理订单
    if (data.status === 2) {
        // 支付成功
        console.log(`订单 ${data.order_id} 支付成功`);
        // 执行发货逻辑
    }

    res.send('ok');
});
```

### 2. Webhook事件系统 (高级通知)

#### 主要作用
- **全生命周期事件**：订单创建、支付、超时、取消、失败等所有事件
- **系统监控**：监控系统运行状态和异常情况
- **数据分析**：收集业务数据进行分析
- **多系统集成**：可同时通知多个系统

#### 配置方式
在配置文件中设置Webhook URL：

```toml
# conf.toml
webhook_url = "https://your-website.com/webhook"
```

#### 支持的事件类型

| 事件类型 | 说明 | 触发时机 |
|---------|------|----------|
| `order.create` | 订单创建 | 用户创建支付订单时 |
| `order.paid` | 支付成功 | 交易确认完成时 |
| `order.timeout` | 订单超时 | 订单超过有效期时 |
| `order.cancel` | 订单取消 | 用户或系统取消订单时 |
| `order.failed` | 支付失败 | 交易确认失败时 |

#### 事件数据格式
```json
{
    "event": "order.paid",
    "data": {
        "id": 123,
        "trade_id": "trade_20241217_001",
        "order_id": "merchant_001",
        "trade_type": "usdt.trc20",
        "trade_hash": "0x123...abc",
        "amount": "14.285714",
        "money": 100.00,
        "address": "TXxxxxxxxxxxx",
        "status": 2,
        "created_at": "2024-12-17T15:00:00Z",
        "updated_at": "2024-12-17T15:30:00Z",
        "expired_at": "2024-12-17T15:30:00Z"
    }
}
```

#### Webhook处理示例
```javascript
app.post('/webhook', express.json(), (req, res) => {
    const { event, data } = req.body;

    switch (event) {
        case 'order.create':
            console.log(`新订单创建: ${data.order_id}`);
            // 发送创建通知
            break;

        case 'order.paid':
            console.log(`订单支付成功: ${data.order_id}`);
            // 发送支付成功通知
            // 触发发货流程
            break;

        case 'order.timeout':
            console.log(`订单超时: ${data.order_id}`);
            // 发送超时提醒
            break;

        case 'order.cancel':
            console.log(`订单取消: ${data.order_id}`);
            // 发送取消通知
            break;

        case 'order.failed':
            console.log(`支付失败: ${data.order_id}`);
            // 发送失败通知
            break;
    }

    res.status(200).send('ok');
});
```

### 3. API轮询查询 (主动查询)

#### 主要作用
- **状态查询**：主动查询订单支付状态
- **页面同步**：前端页面实时显示支付状态
- **订单验证**：验证支付结果的准确性
- **客服支持**：客服查询订单状态

#### API接口
```
GET /pay/check-status/{trade_id}
```

#### 响应数据
```json
{
    "trade_id": "trade_20241217_001",
    "trade_hash": "0x123...abc",
    "status": 2,
    "return_url": "https://merchant.com/success?order_id=merchant_001"
}
```

#### 使用示例
```javascript
// 前端轮询示例
function checkOrderStatus(tradeId) {
    setInterval(async () => {
        try {
            const response = await fetch(`/pay/check-status/${tradeId}`);
            const result = await response.json();

            if (result.status === 2) {
                // 支付成功
                window.location.href = result.return_url;
            } else if (result.status === 3) {
                // 订单超时
                alert('订单已超时，请重新创建订单');
            }
        } catch (error) {
            console.error('查询订单状态失败:', error);
        }
    }, 5000); // 每5秒查询一次
}

// 页面加载时开始轮询
checkOrderStatus('trade_20241217_001');
```

## 🚀 部署和配置指南

### 步骤1：配置系统

#### 1.1 基础配置
```toml
# conf.toml
app_uri = "https://your-domain.com"
auth_token = "your_secure_auth_token_here"
listen = ":8080"

# Webhook配置（可选，用于接收系统事件）
webhook_url = "https://your-website.com/webhook"

# 支付配置
[pay]
wallet_address = ["TXxxxxxxxxxxxxxxxxxxxxxxxxxxx"]
expire_time = 1800
```

#### 1.2 网络配置
```toml
[evm_rpc]
bsc = "https://bsc-dataseed1.binance.org/"
ethereum = "https://eth-mainnet.alchemyapi.io/v2/your-api-key"
polygon = "https://polygon-rpc.com/"

tron_grpc_node = "grpc.trongrid.io:50091"
```

### 步骤2：创建接收通知的服务

#### 2.1 HTTP回调接收服务
```python
# Flask示例 - payment_notify.py
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)

AUTH_TOKEN = "your_auth_token_here"

@app.route('/payment/notify', methods=['POST'])
def payment_notify():
    data = request.get_json()

    # 验证签名
    signature = data.pop('signature', '')
    string_to_sign = '&'.join(f"{k}={v}" for k, v in sorted(data.items()))
    calculated_signature = hmac.new(
        AUTH_TOKEN.encode(),
        string_to_sign.encode(),
        hashlib.sha256
    ).hexdigest()

    if signature != calculated_signature:
        return jsonify({'error': 'invalid signature'}), 400

    # 处理订单
    trade_id = data['trade_id']
    order_id = data['order_id']
    status = data['status']
    amount = data['actual_amount']

    if status == 2:  # 支付成功
        # 更新数据库订单状态
        update_order_status(order_id, 'paid', amount)
        # 触发发货流程
        trigger_delivery(order_id)
        print(f"订单 {order_id} 支付成功，金额: {amount}")

    return 'ok'

def update_order_status(order_id, status, amount):
    # 更新你的数据库
    pass

def trigger_delivery(order_id):
    # 执行发货逻辑
    pass

if __name__ == '__main__':
    app.run(port=5000)
```

#### 2.2 Webhook事件接收服务
```python
# webhook_handler.py
@app.route('/webhook', methods=['POST'])
def webhook_handler():
    data = request.get_json()
    event = data['event']
    order_data = data['data']

    print(f"收到事件: {event}")

    if event == 'order.create':
        # 处理订单创建事件
        send_notification(f"新订单: {order_data['order_id']}")

    elif event == 'order.paid':
        # 处理支付成功事件
        send_notification(f"支付成功: {order_data['order_id']}")

    elif event == 'order.timeout':
        # 处理订单超时事件
        send_notification(f"订单超时: {order_data['order_id']}")

    return 'ok'

def send_notification(message):
    # 发送通知到你的监控系统
    print(f"通知: {message}")
```

### 步骤3：集成到你的网站

#### 3.1 创建支付订单
```javascript
// 前端调用示例
async function createPayment() {
    const response = await fetch('/api/v1/order/create-transaction', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            order_id: 'order_' + Date.now(),
            amount: 100.00,
            trade_type: 'usdt.trc20',
            notify_url: 'https://your-website.com/payment/notify',
            redirect_url: 'https://your-website.com/payment/success',
            name: '商品购买'
        })
    });

    const result = await response.json();

    if (result.code === 200) {
        // 显示支付信息
        showPaymentInfo(result.data);
        // 开始轮询状态
        checkOrderStatus(result.data.trade_id);
    }
}
```

#### 3.2 支付页面
```html
<!DOCTYPE html>
<html>
<head>
    <title>支付页面</title>
    <script src="https://cdn.jsdelivr.net/npm/qrcodejs@1.0.0/qrcode.min.js"></script>
</head>
<body>
    <div id="payment-info">
        <h3>支付信息</h3>
        <div id="qrcode"></div>
        <p>支付金额: <span id="amount"></span> USDT</p>
        <p>收款地址: <span id="address"></span></p>
        <p>有效期: <span id="expire"></span> 分钟</p>
    </div>

    <div id="status-info" style="display:none;">
        <h3>支付状态</h3>
        <p id="status-text"></p>
    </div>

    <script>
        let paymentData = {};

        function showPaymentInfo(data) {
            paymentData = data;

            document.getElementById('amount').textContent = data.actual_amount;
            document.getElementById('address').textContent = data.token;
            document.getElementById('expire').textContent = Math.floor(data.expiration_time / 60);

            // 生成二维码
            new QRCode(document.getElementById("qrcode"), {
                text: `TRON:${data.token}:${data.actual_amount}:usdt.trc20`,
                width: 200,
                height: 200
            });
        }

        function checkOrderStatus(tradeId) {
            const interval = setInterval(async () => {
                try {
                    const response = await fetch(`/pay/check-status/${tradeId}`);
                    const result = await response.json();

                    if (result.status === 2) {
                        // 支付成功
                        clearInterval(interval);
                        document.getElementById('status-info').style.display = 'block';
                        document.getElementById('status-text').textContent = '支付成功，正在跳转...';

                        // 跳转到成功页面
                        setTimeout(() => {
                            window.location.href = result.return_url;
                        }, 2000);

                    } else if (result.status === 3) {
                        // 订单超时
                        clearInterval(interval);
                        document.getElementById('status-info').style.display = 'block';
                        document.getElementById('status-text').textContent = '订单已超时，请重新创建订单';
                    }
                } catch (error) {
                    console.error('查询订单状态失败:', error);
                }
            }, 5000);
        }
    </script>
</body>
</html>
```

## 🔍 调试和监控

### 1. 日志监控
```bash
# 查看支付相关日志
tail -f /var/log/bepusdt/app.log | grep -E "(订单|支付|通知)"

# 查看Webhook发送日志
grep "Webhook" /var/log/bepusdt/app.log

# 查看回调发送日志
grep "订单通知" /var/log/bepusdt/app.log
```

### 2. 回调测试
```bash
# 测试回调接口
curl -X POST https://your-website.com/payment/notify \
  -H "Content-Type: application/json" \
  -d '{
    "trade_id": "test_001",
    "order_id": "test_order",
    "amount": 100.00,
    "actual_amount": "14.285714",
    "status": 2,
    "signature": "test_signature"
  }'
```

### 3. 状态验证
```bash
# 查询订单状态
curl http://localhost:8080/pay/check-status/trade_20241217_001
```

## ⚠️ 注意事项

### 1. 安全性
- **签名验证**：必须验证回调请求的签名
- **IP白名单**：限制只有BEpusdt服务器能访问回调接口
- **HTTPS**：回调接口必须使用HTTPS
- **重复处理**：防止重复处理同一个订单

### 2. 可靠性
- **超时重试**：BEpusdt会自动重试失败的回调
- **幂等性**：回调接口要支持重复调用
- **错误处理**：妥善处理各种异常情况
- **日志记录**：详细记录所有回调操作

### 3. 性能
- **异步处理**：回调处理要异步执行，不阻塞响应
- **数据库优化**：及时更新订单状态，避免重复查询
- **缓存策略**：合理使用缓存减少数据库压力

## ✅ 总结

BEpusdt的网站通知功能已经非常完善，提供了三种互补的通知方式：

1. **HTTP回调** - 主要的支付结果通知，适合商户系统集成
2. **Webhook事件** - 全生命周期事件通知，适合系统监控和数据分析
3. **API轮询** - 主动查询方式，适合前端状态同步

通过合理配置和使用这些功能，完全可以替代Telegram Bot的通知作用，实现更稳定、更可靠的网站通知系统。