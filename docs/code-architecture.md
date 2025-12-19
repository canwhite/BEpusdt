# BEpusdt 代码架构详解

## 概览

BEpusdt是一个基于Go语言开发的USDT/USDC支付监听系统，采用模块化设计，通过直接扫描区块链来实时监控和确认支付交易。本文档详细梳理整个系统的代码逻辑和架构设计。

## 项目结构

```
BEpusdt/
├── main/                     # 程序入口
│   └── main.go              # 主程序文件
├── app/                     # 应用核心逻辑
│   ├── bot/                 # Telegram Bot模块
│   ├── conf/                # 配置管理
│   ├── help/                # 辅助工具函数
│   ├── log/                 # 日志管理
│   ├── model/               # 数据模型
│   ├── task/                # 区块链监听任务
│   └── web/                 # Web服务和API
├── static/                  # 静态资源
├── docs/                    # 文档
└── conf.example.toml        # 配置文件示例
```

## 核心模块架构

### 1. 程序启动流程 (`main/main.go`)

程序启动的核心流程如下：

```go
// 系统初始化器顺序
var initializers = []Initializer{
    conf.Init,    // 1. 配置初始化
    log.Init,     // 2. 日志初始化
    bot.Init,     // 3. Bot初始化
    model.Init,   // 4. 数据库初始化
    task.Init,    // 5. 任务监听初始化
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 启动任务监听
    task.Start(ctx)

    // 启动Web服务
    web.Start(ctx)

    // 监听系统信号
    gracefulShutdown()
}
```

**关键设计理念**：
- **依赖注入**：通过初始化器顺序控制模块启动顺序
- **Context管理**：使用Go Context进行优雅关闭
- **并发处理**：任务监听和Web服务并行运行

### 2. 配置管理模块 (`app/conf/`)

配置结构定义在 `app/conf/conf.go`：

```go
type Conf struct {
    AppUri       string `toml:"app_uri"`        // 应用URI
    AuthToken    string `toml:"auth_token"`     // API认证令牌
    Listen       string `toml:"listen"`         // Web服务监听地址
    SqlitePath   string `toml:"sqlite_path"`    // SQLite数据库路径

    // 支付配置
    Pay struct {
        TrxAtom          float64  `toml:"trx_atom"`           // TRX最小精度
        UsdtAtom         float64  `toml:"usdt_atom"`          // USDT最小精度
        UsdtRate         string   `toml:"usdt_rate"`          // USDT汇率配置
        ExpireTime       int      `toml:"expire_time"`        // 订单超时时间
        WalletAddress    []string `toml:"wallet_address"`     // 收款钱包地址列表
        TradeIsConfirmed bool     `toml:"trade_is_confirmed"` // 是否需要交易确认
    } `toml:"pay"`

    // EVM RPC节点配置
    EvmRpc struct {
        Bsc      string `toml:"bsc"`      // BSC RPC节点
        Ethereum string `toml:"ethereum"` // Ethereum RPC节点
        Polygon  string `toml:"polygon"`  // Polygon RPC节点
        // ... 其他网络
    } `toml:"evm_rpc"`

    // Telegram Bot配置
    Bot struct {
        Token   string `toml:"token"`   // Bot Token
        AdminID int64  `toml:"admin_id"` // 管理员ID
        GroupID string `toml:"group_id"` // 群组ID
    } `toml:"bot"`
}
```

### 3. 数据模型层 (`app/model/`)

#### 3.1 核心数据表结构

**订单表 (TradeOrders)** `app/model/orders.go:54-77`：
```go
type TradeOrders struct {
    Id          int64     `gorm:"primary_key;AUTO_INCREMENT"`
    OrderId     string    `gorm:"column:order_id;type:varchar(128);not null;index"`       // 商户订单ID
    TradeId     string    `gorm:"column:trade_id;type:varchar(128);not null;uniqueIndex"`  // 本地交易ID
    TradeType   string    `gorm:"column:trade_type;type:varchar(20);not null"`            // 交易类型
    Address     string    `gorm:"column:address;type:varchar(64);not null"`               // 收款地址
    Amount      string    `gorm:"type:decimal(10,2);not null;default:0"`                  // USDT金额
    Money       float64   `gorm:"type:decimal(10,2);not null;default:0"`                  // CNY金额
    Status      int       `gorm:"type:tinyint(1);not null;default:1"`                     // 订单状态
    CreatedAt   time.Time `gorm:"autoCreateTime;type:timestamp;not null"`                 // 创建时间
    ExpiredAt   time.Time `gorm:"column:expired_at;type:timestamp;not null"`               // 过期时间
    // ... 其他字段
}
```

**钱包地址表 (WalletAddress)** `app/model/address.go`：
```go
type WalletAddress struct {
    Id           int64  `gorm:"primary_key;AUTO_INCREMENT"`
    Address      string `gorm:"column:address;type:varchar(64);not null;unique"`      // 钱包地址
    TradeType    string `gorm:"column:trade_type;type:varchar(20);not null"`          // 支持的交易类型
    Status       int    `gorm:"type:tinyint(1);not null;default:1"`                   // 状态
    OtherNotify  int    `gorm:"type:tinyint(1);not null;default:0"`                   // 是否通知余额变动
    // ... 其他字段
}
```

**Webhook事件表** `app/model/webhook.go`：
```go
type Webhook struct {
    ID        int64           `gorm:"column:id;type:INTEGER PRIMARY KEY AUTOINCREMENT"`
    Status    int8            `gorm:"column:status;type:tinyint;not null;default:0"`   // 状态
    Url       string          `gorm:"column:url;type:varchar(255);not null;default:''"` // 回调URL
    Event     string          `gorm:"column:event;type:varchar(64);not null;default:''"` // 事件类型
    Data      json.RawMessage `gorm:"column:data;type:json;not null"`                   // 事件数据
    // ... 其他字段
}
```

#### 3.2 订单状态流转

```go
const (
    OrderStatusWaiting    = 1 // 等待支付
    OrderStatusSuccess    = 2 // 交易确认成功
    OrderStatusExpired    = 3 // 订单过期
    OrderStatusCanceled   = 4 // 订单取消
    OrderStatusConfirming = 5 // 等待交易确认
    OrderStatusFailed     = 6 // 交易确认失败
)
```

**状态流转逻辑**：
```
[创建] → Waiting → Confirming → Success
   ↓         ↓          ↓
Expired   Canceled   Failed
```

### 4. Web服务层 (`app/web/`)

#### 4.1 路由架构 (`app/web/web.go`)

```go
func Start(ctx context.Context) {
    engine := gin.New()

    // 中间件配置
    engine.Use(gin.LoggerWithWriter(log.GetWriter()), gin.Recovery())
    engine.Use(securityHeadersMiddleware)

    // 路由分组
    {
        // 首页
        engine.GET("/", indexHandler)

        // 支付页面
        payGrp := engine.Group("/pay")
        payGrp.GET("/checkout-counter/:trade_id", checkoutCounter)
        payGrp.GET("/check-status/:trade_id", checkStatus)

        // API接口
        orderGrp := engine.Group("/api/v1/order")
        orderGrp.Use(signVerify)  // 签名验证中间件
        orderGrp.POST("/create-transaction", createTransaction)
        orderGrp.POST("/cancel-transaction", cancelTransaction)

        // 易支付兼容
        engine.POST("/submit.php", epaySubmit)
        engine.GET("/submit.php", epaySubmit)
    }
}
```

#### 4.2 核心API实现

**创建订单API** `app/web/epusdt.go:57-135`：

```go
func createTransaction(ctx *gin.Context) {
    // 1. 签名验证（在中间件中完成）
    data := ctx.GetStringMap("data")

    // 2. 参数验证
    validateRequiredParams(data, []string{"order_id", "amount", "notify_url", "redirect_url"})

    // 3. 交易类型验证
    tradeType := validateTradeType(data["trade_type"])

    // 4. 地址验证（可选）
    address := validateAddress(data["address"])

    // 5. 构建订单
    params := orderParams{
        Money:       cast.ToFloat64(data["amount"]),
        OrderId:     cast.ToString(data["order_id"]),
        TradeType:   tradeType,
        PayAddress:  address,
        NotifyUrl:   cast.ToString(data["notify_url"]),
        RedirectUrl: cast.ToString(data["redirect_url"]),
        // ... 其他参数
    }

    // 6. 创建订单
    order, err := buildOrder(params)
    if err != nil {
        ctx.JSON(200, respFailJson(err.Error()))
        return
    }

    // 7. 返回响应
    ctx.JSON(200, respSuccJson(gin.H{
        "trade_id":        order.TradeId,
        "status":          order.Status,
        "actual_amount":   order.Amount,
        "token":           order.Address,
        "payment_url":     fmt.Sprintf("%s/pay/checkout-counter/%s", conf.GetAppUri(), order.TradeId),
    }))
}
```

**签名验证中间件** `app/web/epusdt.go:18-55`：

```go
func signVerify(ctx *gin.Context) {
    // 1. 获取原始数据
    rawData, err := ctx.GetRawData()

    // 2. 解析JSON
    m := make(map[string]any)
    json.Unmarshal(rawData, &m)

    // 3. 提取签名
    sign, ok := m["signature"]
    if !ok {
        ctx.JSON(400, gin.H{"error": "signature not found"})
        ctx.Abort()
        return
    }

    // 4. 验证签名
    if help.EpusdtSign(m, conf.GetAuthToken()) != sign {
        ctx.JSON(400, gin.H{"error": "签名错误"})
        ctx.Abort()
        return
    }

    ctx.Set("data", m)
}
```

### 5. 区块链监听层 (`app/task/`)

#### 5.1 任务管理架构 (`app/task/task.go`)

```go
// 任务定义
type task struct {
    duration time.Duration              // 执行间隔
    callback func(ctx context.Context)  // 回调函数
}

var tasks []task

// 注册任务
func register(t task) {
    tasks = append(tasks, t)
}

// 启动所有任务
func Start(ctx context.Context) {
    for _, t := range tasks {
        go func(t task) {
            if t.duration <= 0 {
                // 只执行一次的任务
                t.callback(ctx)
                return
            }

            // 定时任务
            ticker := time.NewTicker(t.duration)
            defer ticker.Stop()

            for {
                select {
                case <-ctx.Done():
                    return
                case <-ticker.C:
                    t.callback(ctx)
                }
            }
        }(t)
    }
}
```

#### 5.2 网络初始化 (`app/task/task.go:22-32`)

```go
func Init() error {
    // 初始化各个网络的监听任务
    bscInit()       // BSC网络
    ethInit()       // Ethereum网络
    polygonInit()   // Polygon网络
    arbitrumInit()  // Arbitrum网络
    plasmaInit()    // Plasma网络
    xlayerInit()    // X-Layer网络
    baseInit()      // Base网络

    return nil
}
```

#### 5.3 EVM网络通用监听器 (`app/task/evm.go`)

**区块轮询逻辑** `app/task/evm.go:97-164`：

```go
func (e *evm) blockRoll(ctx context.Context) {
    // 1. 检查是否需要暂停扫描
    if rollBreak(e.Network) {
        return
    }

    // 2. 获取当前区块高度
    now := e.getCurrentBlockNumber(ctx)
    if now <= 0 {
        return
    }

    // 3. 应用确认偏移量
    if conf.GetTradeIsConfirmed() {
        now = now - e.Block.ConfirmedOffset
    }

    // 4. 处理区块高度跳跃
    lastBlockNumber := e.getLastBlockNumber()
    if now-lastBlockNumber > conf.BlockHeightMaxDiff {
        lastBlockNumber = e.blockInitOffset(now, e.Block.InitStartOffset) - 1
    }

    // 5. 批量处理区块
    for from := lastBlockNumber + 1; from <= now; from += blockParseMaxNum {
        to := from + blockParseMaxNum - 1
        if to > now {
            to = now
        }
        e.blockScanQueue.In <- evmBlock{From: from, To: to}
    }

    // 6. 更新最后处理的区块号
    e.updateLastBlockNumber(now)
}
```

**区块解析逻辑** `app/task/evm.go:210-282`：

```go
func (e *evm) getBlockByNumber(a any) {
    blockNumbers := a.(evmBlock)

    // 1. 批量获取区块数据
    blockData := e.fetchBlocks(blockNumbers)

    // 2. 提取区块时间戳
    timestamp := e.extractTimestamps(blockData)

    // 3. 解析转账事件
    transfers, err := e.parseBlockTransfer(blockNumbers, timestamp)
    if err != nil {
        // 处理错误，重新入队
        e.blockScanQueue.In <- blockNumbers
        return
    }

    // 4. 将转账事件加入处理队列
    if len(transfers) > 0 {
        transferQueue.In <- transfers
    }
}
```

**事件日志解析** `app/task/evm.go:284-354`：

```go
func (e *evm) parseBlockTransfer(b evmBlock, timestamp map[string]time.Time) ([]transfer, error) {
    // 1. 监听Transfer事件
    // 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
    post := []byte(fmt.Sprintf(`{
        "jsonrpc":"2.0",
        "method":"eth_getLogs",
        "params":[{
            "fromBlock":"0x%x",
            "toBlock":"0x%x",
            "topics":["%s"]
        }],
        "id":1
    }`, b.From, b.To, evmTransferEvent))

    // 2. 调用RPC获取日志
    resp, err := client.Post(e.Endpoint, "application/json", bytes.NewBuffer(post))

    // 3. 解析日志
    for _, logItem := range data.Get("result").Array() {
        // 4. 验证合约地址
        to := logItem.Get("address").String()
        tradeType, ok := contractMap[to]
        if !ok {
            continue  // 不是目标合约
        }

        // 5. 解析事件参数
        topics := logItem.Get("topics").Array()
        from := fmt.Sprintf("0x%s", topics[1].String()[26:])
        recv := fmt.Sprintf("0x%s", topics[2].String()[26:])
        amount, _ := new(big.Int).SetString(logItem.Get("data").String()[2:], 16)

        // 6. 构建转账记录
        transfers = append(transfers, transfer{
            Network:     e.Network,
            FromAddress: from,
            RecvAddress: recv,
            Amount:      decimal.NewFromBigInt(amount, decimals[to]),
            TxHash:      logItem.Get("transactionHash").String(),
            TradeType:   tradeType,
        })
    }

    return transfers, nil
}
```

#### 5.4 TRON网络监听器 (`app/task/tron.go`)

**区块扫描逻辑** `app/task/tron.go:66-117`：

```go
func (t *tron) blockRoll(context.Context) {
    // 1. 检查是否需要暂停
    if t.rollBreak() {
        return
    }

    // 2. 连接TRON GRPC节点
    conn, err := grpc.NewClient(conf.GetTronGrpcNode(), grpc.WithTransportCredentials(insecure.NewCredentials()))
    defer conn.Close()

    client := api.NewWalletClient(conn)

    // 3. 获取最新区块
    block, err := client.GetNowBlock2(ctx, nil)
    now := block.BlockHeader.RawData.Number

    // 4. 应用确认偏移
    if conf.GetTradeIsConfirmed() {
        now = now - t.blockConfirmedOffset
    }

    // 5. 处理区块跳跃
    if now-t.lastBlockNum > conf.BlockHeightMaxDiff {
        t.blockInitOffset(now)
        t.lastBlockNum = now - 1
    }

    // 6. 将区块加入扫描队列
    for n := t.lastBlockNum + 1; n <= now; n++ {
        t.blockScanQueue.In <- n
    }

    t.lastBlockNum = now
}
```

**交易解析逻辑** `app/task/tron.go:138-322`：

```go
func (t *tron) blockParse(n any) {
    blockNum := n.(int64)

    // 1. 获取区块数据
    bok, err := client.GetBlockByNum2(ctx, &api.NumberMessage{Num: blockNum})

    transfers := make([]transfer, 0)
    resources := make([]resource, 0)
    timestamp := time.UnixMilli(bok.GetBlockHeader().GetRawData().GetTimestamp())

    // 2. 遍历区块中的所有交易
    for _, trans := range bok.GetTransactions() {
        if !trans.Result.Result {
            continue  // 跳过失败的交易
        }

        transaction := trans.GetTransaction()
        txId := hex.EncodeToString(trans.Txid)

        // 3. 解析合约调用
        for _, contract := range transaction.GetRawData().GetContract() {
            switch contract.GetType() {
            case core.Transaction_Contract_TransferContract:
                // TRX原生转账
                transfers = append(transfers, t.parseTrxTransfer(contract, txId, timestamp, blockNum))

            case core.Transaction_Contract_TriggerSmartContract:
                // 智能合约调用
                transfers = append(transfers, t.parseTrc20Transfer(contract, txId, timestamp, blockNum)...)

            case core.Transaction_Contract_DelegateResourceContract:
                // 资源代理
                resources = append(resources, t.parseResourceDelegate(contract, txId, timestamp))
            }
        }
    }

    // 4. 将结果加入处理队列
    if len(transfers) > 0 {
        transferQueue.In <- transfers
    }
    if len(resources) > 0 {
        resourceQueue.In <- resources
    }
}
```

**TRC20转账解析** `app/task/tron.go:279-308`：

```go
func (t *tron) parseTrc20Transfer(contract *core.Transaction_Contract, txId string, timestamp time.Time, blockNum int64) []transfer {
    var foo = &core.TriggerSmartContract{}
    contract.GetParameter().UnmarshalTo(foo)

    data := foo.GetData()

    // 1. 判断代币类型
    var tradeType = "None"
    if bytes.Equal(foo.GetContractAddress(), usdtTrc20ContractAddress) {
        tradeType = model.OrderTradeTypeUsdtTrc20
    } else if bytes.Equal(foo.GetContractAddress(), usdcTrc20ContractAddress) {
        tradeType = model.OrderTradeTypeUsdcTrc20
    } else {
        return nil  // 不是目标代币
    }

    var transfers []transfer

    // 2. 解析transfer方法 (0xa9059cbb)
    if bytes.Equal(data[:4], []byte{0xa9, 0x05, 0x9c, 0xbb}) {
        receiver, amount := t.parseTrc20ContractTransfer(data)
        if amount != nil {
            transfers = append(transfers, transfer{
                Network:     conf.Tron,
                TxHash:      txId,
                Amount:      decimal.NewFromBigInt(amount, trc20TokenDecimals[tradeType]),
                FromAddress: t.base58CheckEncode(foo.OwnerAddress),
                RecvAddress: receiver,
                Timestamp:   timestamp,
                TradeType:   tradeType,
                BlockNum:    blockNum,
            })
        }
    }

    // 3. 解析transferFrom方法 (0x23b872dd)
    if bytes.Equal(data[:4], []byte{0x23, 0xb8, 0x72, 0xdd}) {
        from, to, amount := t.parseTrc20ContractTransferFrom(data)
        if amount != nil {
            transfers = append(transfers, transfer{
                Network:     conf.Tron,
                TxHash:      txId,
                Amount:      decimal.NewFromBigInt(amount, trc20TokenDecimals[tradeType]),
                FromAddress: from,
                RecvAddress: to,
                Timestamp:   timestamp,
                TradeType:   tradeType,
                BlockNum:    blockNum,
            })
        }
    }

    return transfers
}
```

### 6. 转账处理模块 (`app/task/transfer.go`)

#### 6.1 转账事件处理队列

```go
// 全局队列定义
var transferQueue = chanx.NewUnboundedChan[[]transfer](context.Background(), 30)  // 订单转账队列
var notOrderQueue = chanx.NewUnboundedChan[[]transfer](context.Background(), 30)  // 非订单转账队列
var resourceQueue = chanx.NewUnboundedChan[[]resource](context.Background(), 30)   // 资源操作队列

// 注册处理器
func init() {
    register(task{callback: orderTransferHandle})      // 订单转账处理
    register(task{callback: notOrderTransferHandle})   // 非订单转账处理
    register(task{callback: tronResourceHandle})       // TRON资源操作处理
}
```

#### 6.2 订单匹配核心逻辑

**订单匹配处理器** `app/task/transfer.go:62-100`：

```go
func orderTransferHandle(context.Context) {
    for transfers := range transferQueue.Out {
        // 1. 获取所有等待支付的订单
        orders := getAllWaitingOrders()

        // 2. 预分离非订单交易
        var other = make([]transfer, 0)

        // 3. 逐笔匹配
        for _, t := range transfers {
            // 金额范围检查
            if !inAmountRange(t.Amount) {
                continue
            }

            // 精确匹配：地址+金额+交易类型
            orderKey := fmt.Sprintf("%s%v%s", t.RecvAddress, t.Amount.String(), t.TradeType)
            o, ok := orders[orderKey]
            if !ok {
                // 不是订单交易，转入非订单队列
                other = append(other, t)
                continue
            }

            // 时间有效性检查
            if !o.CreatedAt.Before(t.Timestamp) || !o.ExpiredAt.After(t.Timestamp) {
                continue
            }

            // 标记为确认中状态
            o.MarkConfirming(t.BlockNum, t.FromAddress, t.TxHash, t.Timestamp)
        }

        // 4. 将非订单交易转入非订单队列
        if len(other) > 0 {
            notOrderQueue.In <- other
        }
    }
}
```

**等待订单获取逻辑** `app/task/transfer.go:216-237`：

```go
func getAllWaitingOrders() map[string]model.TradeOrders {
    // 1. 获取所有等待支付的订单
    var tradeOrders = model.GetOrderByStatus(model.OrderStatusWaiting)
    var data = make(map[string]model.TradeOrders)

    // 2. 处理订单过期
    for _, order := range tradeOrders {
        if time.Now().Unix() >= order.ExpiredAt.Unix() {
            order.SetExpired()
            notify.Bepusdt(order)
            model.PushWebhookEvent(model.WebhookEventOrderTimeout, order)
            continue
        }

        // 3. Polygon地址小写化处理
        if order.TradeType == model.OrderTradeTypeUsdtPolygon {
            order.Address = strings.ToLower(order.Address)
        }

        // 4. 构建订单键值：地址+金额+交易类型
        data[order.Address+order.Amount+order.TradeType] = order
    }

    return data
}
```

#### 6.3 最终确认处理

**交易确认处理器** `app/task/transfer.go:53-60`：

```go
func markFinalConfirmed(o model.TradeOrders) {
    // 1. 触发Webhook事件
    model.PushWebhookEvent(model.WebhookEventOrderPaid, o)

    // 2. 更新订单状态为成功
    o.SetSuccess()

    // 3. 异步通知处理
    go notify.Handle(o)         // 发送支付回调
    go bot2.SendTradeSuccMsg(o) // 发送Telegram通知
}
```

### 7. 地址分配算法 (`app/model/orders.go`)

**智能地址分配** `app/model/orders.go:218-247`：

```go
func CalcTradeAmount(wa []WalletAddress, rate, money float64, tradeType string) (WalletAddress, string) {
    calcMutex.Lock()
    defer calcMutex.Unlock()

    // 1. 获取所有等待中的订单，构建锁定映射
    var orders []TradeOrders
    var lock = make(map[string]bool)
    DB.Where("status = ? and trade_type = ?", OrderStatusWaiting, tradeType).Find(&orders)
    for _, order := range orders {
        // 锁定已使用的地址+金额组合
        lock[order.Address+order.Amount] = true
    }

    // 2. 获取代币原子精度
    var atom, prec = getTokenAtomicityByTradeType(tradeType)

    // 3. 计算基础支付金额
    var payAmount, _ = decimal.NewFromString(strconv.FormatFloat(money/rate, 'f', prec, 64))

    // 4. 寻找可用的地址+金额组合
    for {
        for _, address := range wa {
            key := address.Address + payAmount.String()
            if _, ok := lock[key]; !ok {
                // 找到可用组合
                return address, payAmount.String()
            }
        }

        // 如果都被占用，递增原子精度继续寻找
        payAmount = payAmount.Add(atom)
    }
}
```

### 8. 通知系统 (`app/web/notify/` & `app/model/webhook.go`)

#### 8.1 支付回调处理

**统一回调处理器** `app/web/notify/notify.go:33-49`：

```go
func Handle(order model.TradeOrders) {
    // 只处理成功状态的订单
    if order.Status != model.OrderStatusSuccess {
        return
    }

    ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
    defer cancel()

    // 根据API类型选择回调方式
    switch order.ApiType {
    case model.OrderApiTypeEpay:
        epay(ctx, order)      // 易支付兼容
    default:
        epusdt(ctx, order)    // 标准EPUSDT回调
    }
}
```

**EPUSDT标准回调** `app/web/notify/notify.go:99-172`：

```go
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

    // 3. 发送回调
    finalJsonBody, _ := json.Marshal(body)
    client := http.Client{Timeout: time.Second * 5}
    req, _ := http.NewRequestWithContext(ctx, "POST", order.NotifyUrl, strings.NewReader(string(finalJsonBody)))

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Powered-By", "https://github.com/v03413/bepusdt")
    req.Header.Set("User-Agent", "BEpusdt/"+app.Version)

    resp, err := client.Do(req)
    if err != nil {
        markNotifyFail(order, err.Error())
        return
    }
    defer resp.Body.Close()

    // 4. 验证响应
    if resp.StatusCode != 200 {
        markNotifyFail(order, "resp.StatusCode != 200")
        return
    }

    all, _ := io.ReadAll(resp.Body)
    if string(all) != "ok" {
        markNotifyFail(order, fmt.Sprintf("body != ok (%s)", string(all)))
        return
    }

    // 5. 标记回调成功
    err = order.SetNotifyState(model.OrderNotifyStateSucc)
    if err != nil {
        log.Error("订单标记通知成功错误：", err, order.OrderId)
    } else {
        log.Info("订单通知成功：", order.OrderId)
    }
}
```

#### 8.2 Webhook事件系统

**事件推送机制** `app/model/webhook.go:76-92`：

```go
func PushWebhookEvent(event string, data any) {
    go func() {
        // 1. 获取Webhook URL
        var url = conf.GetWebhookUrl()
        if url == "" {
            return
        }

        // 2. 序列化事件数据
        bytes, _ := json.Marshal(data)

        // 3. 创建Webhook记录
        var w = Webhook{
            Status: WebhookStatusWait,
            Url:    url,
            Event:  event,
            Data:   bytes,
        }

        // 4. 保存到数据库
        if err := DB.Create(&w).Error; err == nil {
            // 5. 加入处理队列
            WebhookHandleQueue.In <- w
        }
    }()
}
```

**Webhook处理队列** `app/task/webhook.go`：

```go
func Init() {
    register(task{callback: webhookHandle, duration: time.Second * 5})
}

func webhookHandle(context.Context) {
    // 1. 处理队列中的事件
    for webhook := range model.WebhookHandleQueue.Out {
        // 2. 发送Webhook请求
        success := sendWebhookRequest(webhook)

        // 3. 更新状态
        if success {
            webhook.SetStatus(model.WebhookStatusSucc)
        } else {
            webhook.SetStatus(model.WebhookStatusFail)
        }
    }
}
```

### 9. Telegram Bot模块 (`app/bot/`)

Bot模块提供以下功能：
- 订单状态通知
- 余额变动提醒
- 资源操作监控
- 管理员命令处理

### 10. 辅助工具模块 (`app/help/` & `app/log/`)

#### 10.1 辅助函数 (`app/help/help.go`)

主要提供以下工具函数：
- 地址验证（TRON、EVM、Solana、Aptos）
- 签名生成和验证
- 字符串处理和掩码
- 时间处理
- 数值转换

#### 10.2 日志管理 (`app/log/log.go`)

- 统一日志格式
- 日志级别控制
- 文件和控制台输出
- 日志轮转

## 数据流图

```
[用户] → [Web API] → [订单创建] → [数据库] → [支付页面]
    ↓
[区块链网络] → [任务监听] → [交易解析] → [订单匹配] → [状态更新]
    ↓
[通知系统] → [回调通知] → [Webhook] → [Telegram Bot]
```

## 关键设计模式

### 1. 生产者-消费者模式
```go
// 区块扫描（生产者） → 交易队列 → 订单匹配（消费者）
blockScanQueue → transferQueue → orderTransferHandle
```

### 2. 策略模式
```go
// 不同网络的扫描策略
type NetworkScanner interface {
    blockRoll(context.Context)
    blockParse(any)
    parseTransfer(any) []transfer
}
```

### 3. 模板方法模式
```go
// EVM网络扫描的通用模板
func (e *evm) scanTemplate() {
    e.getCurrentBlockNumber()
    e.batchProcessBlocks()
    e.parseBlockTransfer()
    e.addToQueue()
}
```

### 4. 观察者模式
```go
// Webhook事件通知
model.PushWebhookEvent(event, data) → Queue → Processors
```

## 性能优化策略

### 1. 并发处理
- 多个网络并行扫描
- 批量区块处理
- Worker池模式解析交易

### 2. 缓存机制
- 区块高度缓存
- 订单状态缓存
- 回调防重复处理

### 3. 队列缓冲
- 非阻塞队列设计
- 失败重试机制
- 优雅关闭处理

### 4. 智能暂停
- 无订单时暂停扫描
- 根据网络情况动态调整
- 资源使用优化

## 安全考虑

### 1. API安全
- 签名验证
- 参数校验
- 访问频率限制

### 2. 数据安全
- 敏感信息掩码
- 日志脱敏
- 数据库访问控制

### 3. 交易安全
- 多重验证机制
- 时间窗口限制
- 金额范围控制

## 扩展性设计

### 1. 网络扩展
- 标准化网络接口
- 配置化网络参数
- 插件化网络支持

### 2. 代币扩展
- 动态代币配置
- 精度自适应
- 合约地址管理

### 3. 功能扩展
- Webhook事件扩展
- 回调方式扩展
- 通知渠道扩展

这个架构设计确保了系统的模块化、可扩展性和高可靠性，同时通过合理的设计模式保证了代码的可维护性。