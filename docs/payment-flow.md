# BEpusdt 接收转账流程详解

## 概述

BEpusdt是一个基于区块链扫描的支付监听系统，它通过实时监控区块链上的交易来确认支付。本文档详细解释了用户如何通过BEpusdt接收USDT/USDC转账的完整流程。

## 系统架构概览

BEpusdt采用以下核心机制：
- **直接区块扫描**：连接区块链RPC节点，实时监控交易
- **智能订单匹配**：通过地址+金额+交易类型精确匹配
- **多链支持**：支持TRON、Ethereum、BSC、Polygon等多条网络
- **实时确认**：秒级响应交易状态变化

## 接收转账的完整流程

### 1. 订单创建 - 获取收款地址

当商户需要收款时，首先调用API创建订单：

#### API请求示例

```bash
POST /api/v1/order/create
Content-Type: application/json

{
    "money": 100.00,           // 订单金额 (人民币)
    "order_id": "merchant_001", // 商户订单ID
    "trade_type": "usdt.trc20", // 选择的转账类型
    "notify_url": "https://merchant.com/notify", // 异步回调地址
    "return_url": "https://merchant.com/success", // 同步跳转地址
    "name": "商品名称",         // 商品描述
    "timeout": 1800             // 订单超时时间(秒)，可选，默认使用配置
}
```

#### 系统处理流程

系统处理订单创建的核心逻辑位于 `app/web/order.go:111-139`：

```go
func buildTrade(p orderParams) (trade, error) {
    // 1. 获取可用的钱包地址
    wallet := model.GetAvailableAddress(p.PayAddress, p.TradeType)
    if len(wallet) == 0 {
        return trade{}, fmt.Errorf("类型(%s)未检测到可用钱包地址", p.TradeType)
    }

    // 2. 根据汇率计算需要的USDT金额
    address, amount := model.CalcTradeAmount(wallet, rate, p.Money, p.TradeType)

    return trade{
        Address: address,   // 分配给这个订单的收款地址
        Amount: amount,     // 需要支付的USDT金额
        Rate:   rate,       // 当时的汇率
    }, nil
}
```

#### API响应结果

```json
{
    "code": 200,
    "msg": "success",
    "data": {
        "trade_id": "trade_20241217_001",
        "order_id": "merchant_001",
        "amount": "14.285714",        // 需要支付的USDT金额
        "address": "TXxxxxxxxxxxx",   // 收款地址
        "trade_type": "usdt.trc20",
        "money": "100.00",
        "trade_rate": "7.00",
        "status": 1,
        "expired_at": "2024-12-17T15:30:00Z",
        "qrcode_url": "https://yourdomain.com/qrcode?data=TRON:TXxxxxxxxxxxx:14.285714:usdt.trc20",
        "detail_url": "https://tronscan.org/#/transaction/trade_20241217_001"
    }
}
```

### 2. 支付信息的获取和展示方式

#### 方式1：二维码支付（推荐）

系统会自动生成包含支付信息的二维码：

```go
// 二维码数据格式 (app/web/order.go)
qrcode_data := fmt.Sprintf("%s:%s:%s:%s",
    network,    // "TRON"
    address,    // "TXxxxxxxxxxxx"
    amount,     // "14.285714"
    token)      // "usdt.trc20"

// 例如: TRON:TXxxxxxxxxxxx:14.285714:usdt.trc20
```

**用户操作流程**：
1. 使用TRON钱包APP（如TronLink、TokenPocket等）扫描二维码
2. 钱包自动解析收款信息：
   - 收款地址：`TXxxxxxxxxxxx`
   - 支付金额：`14.285714 USDT`
   - 网络类型：`TRON (TRC20)`
3. 确认转账

#### 方式2：手动输入

商户可以展示以下信息给用户：

```
📱 支付信息
━━━━━━━━━━━━━━━━━
💰 支付金额：14.285714 USDT
🏠 收款地址：TXxxxxxxxxxxx
🌐 网络：TRON (TRC20)
⏰ 有效期：30分钟
```

**用户手动操作流程**：
1. 打开自己的TRON钱包APP
2. 选择USDT-TRC20代币
3. 点击转账/发送
4. 输入收款地址：`TXxxxxxxxxxxx`
5. 输入金额：`14.285714`
6. 确认转账

### 3. 地址分配和金额计算机制

#### 智能地址分配

系统通过 `app/model/orders.go:218-247` 中的算法智能分配收款地址：

```go
func CalcTradeAmount(wa []WalletAddress, rate, money float64, tradeType string) (WalletAddress, string) {
    // 1. 获取当前所有等待支付的订单，锁定已使用的地址+金额组合
    var lock = make(map[string]bool)
    DB.Where("status = ? and trade_type = ?", OrderStatusWaiting, tradeType).Find(&orders)
    for _, order := range orders {
        lock[order.Address+order.Amount] = true // 锁定这个组合
    }

    // 2. 根据汇率计算精确的支付金额
    payAmount := decimal.NewFromFloat(money/rate)

    // 3. 寻找可用的地址+金额组合
    for {
        for _, address := range wa {
            key := address.Address + payAmount.String()
            if !lock[key] { // 如果这个地址+金额组合可用
                return address, payAmount.String()
            }
        }
        // 如果都被占用，按最小精度递增继续寻找
        payAmount = payAmount.Add(atom)
    }
}
```

#### 金额计算示例

假设当前USDT汇率为 1 USDT = 7.00 CNY：

- 订单金额：100.00 CNY
- 计算USDT：100.00 ÷ 7.00 = 14.285714 USDT
- 系统会在可用地址中寻找一个未被 14.285714 USDT 金额占用的地址
- 如果被占用，会自动递增最小精度（如0.000001），直到找到可用组合

### 4. 区块链监听和订单匹配

#### 交易扫描流程

系统通过多个网络模块实时扫描区块链交易：

**TRON网络扫描** (`app/task/tron.go:66-117`)：
```go
func (t *tron) blockRoll(context.Context) {
    // 1. 获取当前最新区块高度
    block, err1 := client.GetNowBlock2(ctx, nil)
    now := block.BlockHeader.RawData.Number

    // 2. 计算确认偏移量（如果启用交易确认）
    if conf.GetTradeIsConfirmed() {
        now = now - t.blockConfirmedOffset // 默认30个区块确认
    }

    // 3. 将待扫描区块号加入队列
    for n := t.lastBlockNum + 1; n <= now; n++ {
        t.blockScanQueue.In <- n
    }
}
```

**EVM网络扫描** (`app/task/evm.go:97-164`)：
```go
func (e *evm) blockRoll(ctx context.Context) {
    // 1. 获取当前区块高度
    post := []byte(`{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}`)

    // 2. 应用确认偏移量
    if conf.GetTradeIsConfirmed() {
        now = now - e.Block.ConfirmedOffset // 通常12-40个区块确认
    }

    // 3. 批量处理区块
    for from := lastBlockNumber + 1; from <= now; from += blockParseMaxNum {
        e.blockScanQueue.In <- evmBlock{From: from, To: to}
    }
}
```

#### 订单匹配逻辑

核心匹配算法位于 `app/task/transfer.go:62-100`：

```go
func orderTransferHandle(context.Context) {
    for transfers := range transferQueue.Out {
        // 1. 获取所有等待支付的订单
        var orders = getAllWaitingOrders()

        for _, t := range transfers {
            // 2. 金额范围检查
            if !inAmountRange(t.Amount) {
                continue
            }

            // 3. 精确匹配订单：地址+金额+交易类型
            orderKey := fmt.Sprintf("%s%v%s", t.RecvAddress, t.Amount.String(), t.TradeType)
            o, ok := orders[orderKey]
            if !ok {
                // 不是订单交易，转入非订单队列进行余额变动通知
                other = append(other, t)
                continue
            }

            // 4. 时间有效性检查
            if !o.CreatedAt.Before(t.Timestamp) || !o.ExpiredAt.After(t.Timestamp) {
                continue
            }

            // 5. 标记为确认中状态
            o.MarkConfirming(t.BlockNum, t.FromAddress, t.TxHash, t.Timestamp)
        }
    }
}
```

#### 智能合约事件解析

**TRC20代币转账解析** (`app/task/tron.go:279-308`)：
```go
// 解析transfer方法 (0xa9059cbb)
if bytes.Equal(data[:4], []byte{0xa9, 0x05, 0x9c, 0xbb}) {
    receiver, amount := t.parseTrc20ContractTransfer(data)
    if amount != nil {
        transfers = append(transfers, transfer{
            Network:     conf.Tron,
            TxHash:      id,
            Amount:      decimal.NewFromBigInt(amount, exp),
            FromAddress: t.base58CheckEncode(foo.OwnerAddress),
            RecvAddress: receiver,
            Timestamp:   timestamp,
            TradeType:   tradeType,
            BlockNum:    cast.ToInt64(num),
        })
    }
}
```

### 5. 实际集成示例

#### 电商网站集成

```javascript
// 前端支付页面
class BEpusdtPayment {
    constructor() {
        this.paymentForm = document.getElementById('payment-form');
        this.qrContainer = document.getElementById('qr-code');
        this.detailsContainer = document.getElementById('payment-details');
    }

    async createOrder(amount, orderId) {
        try {
            // 1. 调用后端创建BEpusdt订单
            const response = await fetch('/api/payment/create', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    money: amount,
                    order_id: orderId,
                    trade_type: 'usdt.trc20',
                    notify_url: window.location.origin + '/api/payment/notify',
                    return_url: window.location.origin + '/payment/success',
                    name: '商品购买'
                })
            });

            const result = await response.json();

            if (result.code === 200) {
                // 2. 显示支付信息
                this.showPaymentInfo(result.data);
            } else {
                this.showError(result.msg);
            }
        } catch (error) {
            this.showError('订单创建失败：' + error.message);
        }
    }

    showPaymentInfo(data) {
        // 3. 显示二维码
        this.qrContainer.innerHTML = `
            <img src="${data.qrcode_url}" alt="支付二维码" style="width: 200px; height: 200px;">
            <p>请使用TRON钱包扫描二维码支付</p>
        `;

        // 4. 显示支付详情
        this.detailsContainer.innerHTML = `
            <div class="payment-details">
                <h3>支付详情</h3>
                <p><strong>支付金额：</strong>${data.amount} USDT</p>
                <p><strong>收款地址：</strong><code>${data.address}</code></p>
                <p><strong>网络：</strong>TRON (TRC20)</p>
                <p><strong>有效期：</strong>${new Date(data.expired_at).toLocaleString()}</p>
                <button onclick="copyAddress('${data.address}')">复制地址</button>
                <button onclick="openTronWallet('${data.address}', '${data.amount}')">打开钱包</button>
            </div>
        `;

        // 5. 开始轮询订单状态
        this.startOrderStatusCheck(data.trade_id);
    }

    async checkOrderStatus(tradeId) {
        try {
            const response = await fetch(`/api/payment/status/${tradeId}`);
            const result = await response.json();

            if (result.data.status === 2) { // 支付成功
                window.location.href = result.data.return_url || '/payment/success';
            } else if (result.data.status === 3) { // 订单过期
                this.showError('订单已过期，请重新创建订单');
            }
        } catch (error) {
            console.error('检查订单状态失败：', error);
        }
    }

    startOrderStatusCheck(tradeId) {
        this.statusInterval = setInterval(() => {
            this.checkOrderStatus(tradeId);
        }, 5000); // 每5秒检查一次
    }
}

// 辅助函数
function copyAddress(address) {
    navigator.clipboard.writeText(address).then(() => {
        alert('地址已复制到剪贴板');
    });
}

function openTronWallet(address, amount) {
    // 尝试打开TRON钱包的deeplink
    const deepLink = `tronlinkoutside://pull/activity?param=${encodeURIComponent(JSON.stringify({
        "protocol": "TronLink",
        "version": "1.0",
        "action": "transfer",
        "to": address,
        "amount": amount
    }))}`;

    window.location.href = deepLink;
}

// 使用示例
document.addEventListener('DOMContentLoaded', () => {
    const payment = new BEpusdtPayment();

    // 用户点击支付按钮时创建订单
    document.getElementById('pay-button').addEventListener('click', async () => {
        const amount = 100.00; // 订单金额
        const orderId = 'order_' + Date.now();

        await payment.createOrder(amount, orderId);
    });
});
```

#### 后端集成示例 (Node.js)

```javascript
// 后端支付处理
const express = require('express');
const crypto = require('crypto');
const axios = require('axios');

class BEpusdtService {
    constructor(apiUrl, authToken) {
        this.apiUrl = apiUrl;
        this.authToken = authToken;
    }

    async createOrder(params) {
        try {
            const response = await axios.post(`${this.apiUrl}/api/v1/order/create`, params, {
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${this.authToken}`
                }
            });

            return response.data;
        } catch (error) {
            throw new Error(`创建订单失败: ${error.response?.data?.msg || error.message}`);
        }
    }

    async checkOrderStatus(tradeId) {
        try {
            const response = await axios.get(`${this.apiUrl}/api/v1/order/query/${tradeId}`, {
                headers: {
                    'Authorization': `Bearer ${this.authToken}`
                }
            });

            return response.data;
        } catch (error) {
            throw new Error(`查询订单状态失败: ${error.response?.data?.msg || error.message}`);
        }
    }

    verifySignature(data, signature) {
        const sortedKeys = Object.keys(data).sort();
        const queryString = sortedKeys
            .filter(key => key !== 'signature')
            .map(key => `${key}=${data[key]}`)
            .join('&');

        const calculatedSignature = crypto
            .createHmac('sha256', this.authToken)
            .update(queryString)
            .digest('hex');

        return calculatedSignature === signature;
    }
}

// Express路由示例
const app = express();
app.use(express.json());

const bepusdtService = new BEpusdtService('https://your-bepusdt-domain.com', 'your-auth-token');

// 创建支付订单
app.post('/api/payment/create', async (req, res) => {
    try {
        const { amount, orderId, name } = req.body;

        const result = await bepusdtService.createOrder({
            money: amount,
            order_id: orderId,
            trade_type: 'usdt.trc20',
            notify_url: `${req.protocol}://${req.get('host')}/api/payment/notify`,
            return_url: `${req.protocol}://${req.get('host')}/payment/success`,
            name: name || '商品购买'
        });

        res.json({
            code: 200,
            msg: 'success',
            data: result.data
        });
    } catch (error) {
        res.status(500).json({
            code: 500,
            msg: error.message
        });
    }
});

// 查询订单状态
app.get('/api/payment/status/:tradeId', async (req, res) => {
    try {
        const result = await bepusdtService.checkOrderStatus(req.params.tradeId);

        res.json({
            code: 200,
            msg: 'success',
            data: result.data
        });
    } catch (error) {
        res.status(500).json({
            code: 500,
            msg: error.message
        });
    }
});

// 接收支付回调
app.post('/api/payment/notify', (req, res) => {
    try {
        const { signature, ...data } = req.body;

        // 验证签名
        if (!bepusdtService.verifySignature(data, signature)) {
            return res.status(400).send('invalid signature');
        }

        // 处理订单支付成功
        if (data.status === 2) { // 支付成功
            console.log(`订单 ${data.order_id} 支付成功，金额：${data.amount}`);

            // 在这里执行业务逻辑，如发货、更新订单状态等
            // updateOrderStatus(data.order_id, 'paid');
        }

        res.send('ok');
    } catch (error) {
        console.error('处理支付回调失败：', error);
        res.status(500).send('error');
    }
});

// 支付成功页面
app.get('/payment/success', (req, res) => {
    res.render('payment-success', {
        title: '支付成功'
    });
});

app.listen(3000, () => {
    console.log('服务器运行在端口 3000');
});
```

### 6. 安全注意事项

#### 商户端安全

1. **签名验证**：必须验证所有回调请求的签名
2. **金额校验**：在回调处理时再次验证订单金额
3. **防重放**：确保订单ID的唯一性，避免重复处理
4. **超时检查**：检查订单是否已过期

#### 用户端安全

1. **地址验证**：用户转账前务必仔细核对收款地址
2. **网络选择**：确保选择正确的区块链网络（如TRON-TRC20）
3. **金额确认**：确认支付金额与订单要求一致
4. **手续费**：了解并预留足够的网络手续费

### 7. 故障排除

#### 常见问题

1. **订单未及时确认**
   - 检查区块链网络拥堵情况
   - 确认交易手续费是否足够
   - 验证收款地址和网络是否正确

2. **金额不匹配**
   - 确保支付金额与订单要求完全一致（包括小数位数）
   - 检查是否因为网络手续费导致实际到账金额不足

3. **地址识别问题**
   - 确保复制完整的收款地址
   - 检查是否有额外的空格或换行符
   - 验证地址格式是否正确

4. **网络选择错误**
   - TRC20代币必须在TRON网络上转账
   - ERC20代币必须在Ethereum或兼容链上转账

#### 日志检查

系统提供了详细的日志记录，可以通过以下方式排查问题：

```bash
# 查看系统日志
tail -f /var/log/bepusdt/app.log

# 查看特定订单的日志
grep "trade_20241217_001" /var/log/bepusdt/app.log

# 查看区块扫描状态
grep "区块扫描完成" /var/log/bepusdt/app.log
```

### 8. 总结

BEpusdt的接收转账流程可以总结为以下几个关键步骤：

1. **订单创建** → 系统分配收款地址和计算精确金额
2. **信息展示** → 生成二维码和支付详情
3. **用户转账** → 用户在钱包中向指定地址转账
4. **实时监听** → 系统扫描区块链检测交易
5. **精确匹配** → 通过地址+金额+交易类型匹配订单
6. **状态确认** → 验证交易并更新订单状态
7. **通知商户** → 发送支付成功的异步回调

这种方式的优点是：
- **安全性高**：商户无需托管用户私钥
- **实时性好**：秒级响应交易状态
- **准确性高**：精确的匹配算法避免误判
- **扩展性强**：支持多条区块链网络
- **集成简单**：标准的REST API和回调机制

通过理解这个流程，商户可以更好地集成BEpusdt支付系统，用户也可以更安全地完成转账操作。