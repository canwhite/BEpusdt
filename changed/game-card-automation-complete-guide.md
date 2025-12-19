# 二次元游戏卡密发卡自动化完整指南

## 🎯 概述

本文档详细讲解如何使用BEpusdt实现一个完全自动化的二次元游戏卡密发卡网站，从用户买卡到收到卡密的全过程。

## 📋 完整业务流程

```
用户浏览商品 → 点击购买 → 创建BEpusdt订单 → 显示支付页面 → 用户扫码支付
                                    ↓
                         BEpusdt监听区块链 → 确认支付 → 调用你的回调接口
                                    ↓
                         你的网站自动发货 → 用户收到卡密 → 完成交易
```

## 🔗 BEpusdt与发卡系统的联系点

### 1. 核心联系：支付回调接口

**这是最重要的联系点！**

```php
// payment/notify.php - 第27行：这是BEpusdt调用你网站的地方
$input = file_get_contents('php://input');
$data = json_decode($input, true);  // ← BEpusdt发送的数据
```

**BEpusdt什么时候会调用你的网站？**

```
用户支付 → BEpusdt检测到 → 调用你的notify.php
```

### 2. BEpusdt发送给你的数据格式

```json
{
    "trade_id": "trade_20241217_001",        // BEpusdt的订单ID
    "order_id": "GAME_20241217_001",        // 你的订单号
    "amount": 648.00,                        // 用户支付的金额
    "actual_amount": "92.571429",            // 实际USDT金额
    "token": "TXxxxxxxxxxxxxxxxxxxxxxxx",   // 收款地址
    "block_transaction_id": "0x123...abc",   // 区块交易哈希
    "status": 2,                             // 支付状态(2=成功)
    "signature": "generated_signature_here"  // 签名验证
}
```

### 3. 你的发卡逻辑触发点

```php
// payment/notify.php - 第85行：这是发卡的触发逻辑
if ($status === 2) { // 支付成功 ← BEpusdt告诉你的
    // 自动发货
    $deliveryResult = autoDeliverCard($order, $data);
}
```

## 🎮 完整流程详解

让我用一个具体例子来说明：用户小王想买一张《原神》的充值卡

### 📋 第1步：用户浏览商品

```
用户小王打开你的二次元发卡网站
├── 看到《原神》6480创世结晶卡
├── 价格：648元人民币
└── 点击"立即购买"按钮
```

### 🛒 第2步：创建支付订单

**你的网站后端操作**：
```php
// user clicks "购买" button
function createOrder($productId, $userId) {
    // 1. 生成你的订单号
    $orderId = "GAME_" . date("YmdHis") . "_" . rand(1000, 9999);

    // 2. 调用BEpusdt创建支付订单
    $response = callBepusdtAPI([
        'order_id' => $orderId,           // 你的订单号
        'amount' => 648,                  // 订单金额648元
        'trade_type' => 'usdt.trc20',     // 用USDT支付
        'notify_url' => 'https://your-site.com/payment/notify', // ←关键！BEpusdt会通知这个地址
        'redirect_url' => 'https://your-site.com/payment/success',
        'name' => '原神6480创世结晶'
    ]);

    // 3. 把BEpusdt返回的支付信息存到数据库
    saveOrderToDatabase([
        'order_id' => $orderId,
        'bepusdt_trade_id' => $response['trade_id'],
        'product_id' => $productId,
        'amount' => 648,
        'status' => 'waiting_payment',
        'user_id' => $userId
    ]);

    return $response;
}
```

**BEpusdt返回给你的数据**：
```json
{
    "code": 200,
    "data": {
        "trade_id": "trade_20241217_001",        // BEpusdt的订单ID
        "actual_amount": "92.571429",            // 需要支付92.57 USDT
        "token": "TXxxxxxxxxxxxxxxxxxxxxxxxxxx", // 收款地址
        "payment_url": "https://your-site.com/pay/checkout-counter/trade_20241217_001"
    }
}
```

### 💳 第3步：显示支付页面

**你的网站前端显示**：
```html
<div class="payment-page">
    <h3>🎮 原神6480创世结晶 - 648元</h3>

    <!-- 显示二维码 -->
    <div id="qrcode">
        <img src="generate-qrcode.php?data=TRON:TXxxxxxxxxxx:92.571429:usdt.trc20">
        <p>用TRON钱包扫描二维码支付</p>
    </div>

    <!-- 显示支付信息 -->
    <div class="payment-info">
        <p>💰 支付金额: <strong>92.571429 USDT</strong></p>
        <p>🏠 收款地址: <code>TXxxxxxxxxxxxxxxxxxxxxxxxxxx</code></p>
        <p>⏰ 有效期: <strong>30分钟</strong></p>
        <button onclick="copyAddress()">复制地址</button>
    </div>

    <!-- 实时状态检查 -->
    <div id="status">
        <p id="status-text">⏳ 等待支付中...</p>
    </div>
</div>

<script>
// 每5秒检查一次支付状态
setInterval(() => {
    checkPaymentStatus('trade_20241217_001');
}, 5000);

function checkPaymentStatus(tradeId) {
    fetch(`/pay/check-status/${tradeId}`)
        .then(response => response.json())
        .then(data => {
            if (data.status === 2) { // 支付成功
                document.getElementById('status-text').innerHTML = '✅ 支付成功，正在发货...';
                setTimeout(() => {
                    window.location.href = data.return_url;
                }, 2000);
            }
        });
}
</script>
```

### 📱 第4步：用户支付

**用户小王的操作**：
1. 打开TRON钱包APP（比如TokenPocket）
2. 扫描你网站的二维码
3. 确认支付92.571429 USDT到指定地址
4. 点击确认转账

**BEpusdt在做什么**：
```
BEpusdt实时监听TRON区块链
├── 扫描新区块 → 检测到小王的转账交易
├── 解析交易 → 地址匹配，金额匹配 ✅
├── 匹配订单 → 找到对应的原神卡订单
└── 确认交易 → 等待区块链确认（约30秒）
```

### 🔔 第5步：BEpusdt通知你的网站

**支付成功后，BEpusdt会立即调用你的回调接口**：

```http
POST https://your-site.com/payment/notify
Content-Type: application/json

{
    "trade_id": "trade_20241217_001",
    "order_id": "GAME_20241217150000_1234",
    "amount": 648.00,
    "actual_amount": "92.571429",
    "token": "TXxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "block_transaction_id": "0x123...abc",
    "status": 2,
    "signature": "generated_signature_here"
}
```

### 🎁 第6步：你的网站处理发货

**你的回调处理程序** (`payment/notify.php`)：
```php
<?php
// 接收BEpusdt的回调通知
$data = json_decode(file_get_contents('php://input'), true);

// 1. 验证签名（非常重要！防止伪造）
$signature = $data['signature'];
unset($data['signature']);
ksort($data);
$stringToSign = implode('&', array_map(function($k, $v) {
    return $k . '=' . $v;
}, array_keys($data), $data));

$calculatedSignature = hash_hmac('sha256', $stringToSign, 'your_secret_token');
if ($signature !== $calculatedSignature) {
    die('invalid signature'); // 签名不对，拒绝处理
}

// 2. 检查订单状态（防止重复处理）
$order = getOrderFromDatabase($data['order_id']);
if ($order['status'] === 'paid') {
    die('ok'); // 已经处理过了
}

// 3. 验证金额
if ($order['amount'] != $data['amount']) {
    die('amount mismatch'); // 金额不对，可能有问题
}

// 4. 处理订单 - 这是最重要的部分！
if ($data['status'] == 2) { // 支付成功
    // 4.1 更新订单状态
    updateOrderStatus($data['order_id'], 'paid');

    // 4.2 自动发货 - 核心业务逻辑！
    deliverGameCard($order);

    // 4.3 记录日志
    logInfo("订单 {$data['order_id']} 支付成功，已发货");

    echo 'ok'; // 告诉BEpusdt处理成功
} else {
    // 处理其他状态（超时、失败等）
    echo 'ok';
}

function deliverGameCard($order) {
    // 自动发货逻辑
    $productId = $order['product_id'];
    $userId = $order['user_id'];

    // 从卡密库中获取一张未使用的卡密
    $cardKey = getUnusedCardKey($productId);

    if ($cardKey) {
        // 标记卡密为已使用
        markCardKeyAsUsed($cardKey, $userId);

        // 保存发货记录
        saveDeliveryRecord([
            'order_id' => $order['order_id'],
            'card_key' => $cardKey,
            'user_id' => $userId,
            'delivery_time' => date('Y-m-d H:i:s')
        ]);

        // 发送邮件/短信通知用户
        sendNotification($userId, [
            'product_name' => $order['product_name'],
            'card_key' => $cardKey,
            'delivery_time' => date('Y-m-d H:i:s')
        ]);
    } else {
        // 卡密不足，需要人工处理
        markOrderAsPendingManual($order['order_id']);
        notifyAdmin("卡密不足，订单 {$order['order_id']} 需要人工处理");
    }
}
?>
```

### 📧 第7步：用户收到卡密

**你的网站自动执行**：
1. **从卡密库取出一张未使用的原神充值卡**
2. **标记卡密为已售出**
3. **记录发货信息到数据库**
4. **发送邮件/短信给用户小王**

**用户小王收到**：
```
📧 邮件标题：【二次元商城】您的订单已发货

🎮 商品：原神6480创世结晶
💳 订单号：GAME_20241217150000_1234
🔑 卡密：GWHT-K8M2-PX4Y-N9Q1
⏰ 发货时间：2024-12-17 15:35:20

🌟 温馨提示：
1. 请在游戏内兑换，有效期至2025-12-31
2. 如有问题请联系客服QQ：123456789
3. 更多精彩游戏请访问我们的网站
```

### 📊 用户查看订单状态

**用户在你的网站**：
1. **自动跳转到"支付成功"页面**
2. **显示"已发货，请查收邮件"**
3. **在"我的订单"中可以看到**：
   ```
   订单号：GAME_20241217150000_1234
   商品：原神6480创世结晶
   状态：✅ 已发货
   卡密：GWHT-K8M2-PX4Y-N9Q1 (点击复制)
   ```

## 💾 数据库设计

### 完整的数据库表结构

```sql
-- 创建数据库
CREATE DATABASE card_shop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE card_shop;

-- 商品表
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL COMMENT '商品名称',
    description TEXT COMMENT '商品描述',
    price DECIMAL(10,2) NOT NULL COMMENT '价格(人民币)',
    category VARCHAR(50) NOT NULL COMMENT '分类(原神、王者、和平精英等)',
    image_url VARCHAR(500) COMMENT '商品图片URL',
    stock_count INT DEFAULT 0 COMMENT '库存数量',
    sales_count INT DEFAULT 0 COMMENT '销量',
    is_active BOOLEAN DEFAULT TRUE COMMENT '是否上架',
    sort_order INT DEFAULT 0 COMMENT '排序权重',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_category (category),
    INDEX idx_active (is_active)
) COMMENT='商品信息表';

-- 卡密表
CREATE TABLE card_keys (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL COMMENT '关联商品ID',
    card_key VARCHAR(200) NOT NULL COMMENT '卡密内容',
    batch_no VARCHAR(50) COMMENT '批次号',
    is_used BOOLEAN DEFAULT FALSE COMMENT '是否已使用',
    used_by INT COMMENT '使用者用户ID',
    used_at TIMESTAMP NULL COMMENT '使用时间',
    order_id VARCHAR(50) COMMENT '关联的订单号',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_product_used (product_id, is_used),
    INDEX idx_order_id (order_id)
) COMMENT='卡密库存表';

-- 用户表
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(100) UNIQUE NOT NULL COMMENT '用户邮箱',
    username VARCHAR(50) COMMENT '用户名',
    phone VARCHAR(20) COMMENT '手机号',
    qq VARCHAR(20) COMMENT 'QQ号',
    wechat VARCHAR(50) COMMENT '微信号',
    password VARCHAR(255) COMMENT '密码哈希',
    avatar_url VARCHAR(500) COMMENT '头像URL',
    is_active BOOLEAN DEFAULT TRUE COMMENT '账户状态',
    last_login_at TIMESTAMP NULL COMMENT '最后登录时间',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_active (is_active)
) COMMENT='用户信息表';

-- 订单表
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id VARCHAR(50) UNIQUE NOT NULL COMMENT '订单号',
    bepusdt_trade_id VARCHAR(100) COMMENT 'BEpusdt交易ID',
    user_id INT NOT NULL COMMENT '用户ID',
    product_id INT NOT NULL COMMENT '商品ID',
    product_name VARCHAR(200) NOT NULL COMMENT '商品名称快照',
    product_price DECIMAL(10,2) NOT NULL COMMENT '商品价格快照',
    amount DECIMAL(10,2) NOT NULL COMMENT '订单金额',
    usdt_amount DECIMAL(18,8) COMMENT 'USDT金额',
    status ENUM('pending', 'paid', 'expired', 'cancelled', 'failed', 'delivered') DEFAULT 'pending',
    card_key VARCHAR(200) COMMENT '发货的卡密',
    delivery_time TIMESTAMP NULL COMMENT '发货时间',
    delivery_method ENUM('email', 'sms', 'system') DEFAULT 'email' COMMENT '发货方式',
    recipient_email VARCHAR(100) COMMENT '收货邮箱',
    recipient_phone VARCHAR(20) COMMENT '收货手机',
    notify_sent BOOLEAN DEFAULT FALSE COMMENT '是否已发送通知',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
) COMMENT='订单表';

-- 发货记录表
CREATE TABLE delivery_records (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id VARCHAR(50) NOT NULL COMMENT '订单号',
    card_key VARCHAR(200) NOT NULL COMMENT '卡密',
    user_id INT NOT NULL COMMENT '用户ID',
    product_id INT NOT NULL COMMENT '商品ID',
    delivery_method ENUM('email', 'sms', 'system') NOT NULL COMMENT '发货方式',
    recipient VARCHAR(200) COMMENT '接收者(邮箱/手机号)',
    delivery_status ENUM('sent', 'failed', 'pending') DEFAULT 'sent' COMMENT '发送状态',
    sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '发送时间',
    error_message TEXT COMMENT '错误信息',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_order_id (order_id),
    INDEX idx_delivery_status (delivery_status)
) COMMENT='发货记录表';

-- 系统配置表
CREATE TABLE system_config (
    id INT PRIMARY KEY AUTO_INCREMENT,
    config_key VARCHAR(100) UNIQUE NOT NULL COMMENT '配置键',
    config_value TEXT COMMENT '配置值',
    description VARCHAR(500) COMMENT '配置说明',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) COMMENT='系统配置表';

-- 插入示例数据
INSERT INTO products (name, description, price, category, stock_count) VALUES
('原神6480创世结晶', '原神游戏内6480创世结晶充值', 648.00, '原神', 100),
('王者荣耀1000点券', '王者荣耀1000点券充值', 100.00, '王者荣耀', 200),
('和平精英2800点券', '和平精英2800点券充值', 280.00, '和平精英', 150),
('崩坏：星穹铁道6480星琼', '崩坏：星穹铁道6480星琼充值', 648.00, '崩坏星穹铁道', 80);

-- 插入示例卡密
INSERT INTO card_keys (product_id, card_key, batch_no) VALUES
(1, 'GWHT-K8M2-PX4Y-N9Q1', 'BATCH001'),
(1, 'RTYU-L4M3-QX7W-K2P9', 'BATCH001'),
(1, 'ZXCV-B5N6-W3E8-R1T4', 'BATCH001'),
(2, 'KJHG-F3D2-S6A9-M7P1', 'BATCH002'),
(2, 'YUIO-P2Q1-W5R8-T6Y3', 'BATCH002'),
(3, 'MNBV-X1Z9-A4S7-D2F5', 'BATCH003');

-- 插入系统配置
INSERT INTO system_config (config_key, config_value, description) VALUES
('site_name', '二次元游戏发卡平台', '网站名称'),
('site_url', 'https://your-domain.com', '网站URL'),
('admin_email', 'admin@your-domain.com', '管理员邮箱'),
('smtp_host', 'smtp.gmail.com', 'SMTP服务器'),
('smtp_port', '587', 'SMTP端口'),
('smtp_username', 'your-email@gmail.com', 'SMTP用户名'),
('smtp_password', 'your-app-password', 'SMTP密码'),
('bepusdt_auth_token', 'your_bepusdt_token', 'BEpusdt认证令牌');
```

## 🔧 后端API实现

### 完整的PHP后端代码

```php
<?php
// config.php - 配置文件
class Config {
    private static $config = [
        'database' => [
            'host' => 'localhost',
            'dbname' => 'card_shop',
            'username' => 'your_db_user',
            'password' => 'your_db_password',
            'charset' => 'utf8mb4'
        ],
        'bepusdt' => [
            'api_url' => 'https://your-bepusdt-server.com',
            'auth_token' => 'your_bepusdt_auth_token'
        ],
        'email' => [
            'smtp_host' => 'smtp.gmail.com',
            'smtp_port' => 587,
            'username' => 'your-email@gmail.com',
            'password' => 'your-app-password',
            'from_name' => '二次元游戏发卡平台'
        ]
    ];

    public static function get($key) {
        $keys = explode('.', $key);
        $value = self::$config;

        foreach ($keys as $k) {
            if (isset($value[$k])) {
                $value = $value[$k];
            } else {
                return null;
            }
        }

        return $value;
    }
}

// database.php - 数据库连接类
class Database {
    private static $pdo = null;

    public static function getConnection() {
        if (self::$pdo === null) {
            $config = Config::get('database');
            $dsn = "mysql:host={$config['host']};dbname={$config['dbname']};charset={$config['charset']}";

            try {
                self::$pdo = new PDO($dsn, $config['username'], $config['password'], [
                    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                    PDO::ATTR_EMULATE_PREPARES => false
                ]);
            } catch (PDOException $e) {
                die('数据库连接失败: ' . $e->getMessage());
            }
        }

        return self::$pdo;
    }
}

// api.php - 主API文件
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type');

// 处理OPTIONS请求
if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    http_response_code(200);
    exit();
}

$action = $_GET['action'] ?? '';
$method = $_SERVER['REQUEST_METHOD'];

try {
    switch ($action) {
        case 'products':
            if ($method === 'GET') {
                getProducts();
            }
            break;

        case 'create-order':
            if ($method === 'POST') {
                createOrder();
            }
            break;

        case 'check-payment':
            if ($method === 'GET') {
                checkPayment();
            }
            break;

        default:
            sendErrorResponse('未知的API请求');
    }
} catch (Exception $e) {
    sendErrorResponse('服务器错误: ' . $e->getMessage());
}

// 获取商品列表
function getProducts() {
    $pdo = Database::getConnection();

    $stmt = $pdo->prepare("
        SELECT id, name, description, price, category, image_url, stock_count, sales_count
        FROM products
        WHERE is_active = TRUE
        ORDER BY sort_order DESC, id DESC
    ");
    $stmt->execute();
    $products = $stmt->fetchAll();

    sendSuccessResponse($products);
}

// 创建订单
function createOrder() {
    $input = json_decode(file_get_contents('php://input'), true);

    // 验证输入
    if (!isset($input['product_id']) || !isset($input['user_id'])) {
        sendErrorResponse('参数不完整');
    }

    $productId = (int)$input['product_id'];
    $userId = (int)$input['user_id'];

    $pdo = Database::getConnection();

    // 获取商品信息
    $stmt = $pdo->prepare("SELECT * FROM products WHERE id = ? AND is_active = TRUE");
    $stmt->execute([$productId]);
    $product = $stmt->fetch();

    if (!$product) {
        sendErrorResponse('商品不存在或已下架');
    }

    if ($product['stock_count'] <= 0) {
        sendErrorResponse('商品库存不足');
    }

    // 生成订单号
    $orderId = "GAME_" . date("YmdHis") . "_" . rand(1000, 9999);

    // 调用BEpusdt API
    $bepusdtData = [
        'order_id' => $orderId,
        'amount' => $product['price'],
        'trade_type' => 'usdt.trc20',
        'notify_url' => Config::get('site_url') . '/payment/notify.php',
        'redirect_url' => Config::get('site_url') . '/payment/success',
        'name' => $product['name']
    ];

    $signature = generateBepusdtSignature($bepusdtData);
    $bepusdtData['signature'] = $signature;

    $ch = curl_init(Config::get('bepusdt.api_url') . '/api/v1/order/create-transaction');
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($bepusdtData));
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'User-Agent: GameCardPlatform/1.0'
    ]);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 30);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $error = curl_error($ch);
    curl_close($ch);

    if ($error) {
        sendErrorResponse('创建支付订单失败: ' . $error);
    }

    if ($httpCode !== 200) {
        sendErrorResponse('BEpusdt API请求失败，HTTP状态码: ' . $httpCode);
    }

    $result = json_decode($response, true);

    if (!isset($result['code']) || $result['code'] != 200) {
        sendErrorResponse('BEpusdt API返回错误: ' . ($result['msg'] ?? '未知错误'));
    }

    // 保存订单到数据库
    $stmt = $pdo->prepare("
        INSERT INTO orders (order_id, user_id, product_id, product_name, product_price, amount, usdt_amount, status)
        VALUES (?, ?, ?, ?, ?, ?, ?, 'pending')
    ");
    $stmt->execute([
        $orderId,
        $userId,
        $product['id'],
        $product['name'],
        $product['price'],
        $product['price'],
        $result['data']['actual_amount']
    ]);

    // 减少库存
    $stmt = $pdo->prepare("UPDATE products SET stock_count = stock_count - 1, sales_count = sales_count + 1 WHERE id = ?");
    $stmt->execute([$product['id']]);

    sendSuccessResponse([
        'trade_id' => $result['data']['trade_id'],
        'usdt_amount' => $result['data']['actual_amount'],
        'wallet_address' => $result['data']['token']
    ]);
}

// 检查支付状态
function checkPayment() {
    $tradeId = $_GET['trade_id'] ?? '';

    if (empty($tradeId)) {
        sendErrorResponse('缺少交易ID');
    }

    $pdo = Database::getConnection();

    $stmt = $pdo->prepare("SELECT status FROM orders WHERE bepusdt_trade_id = ?");
    $stmt->execute([$tradeId]);
    $order = $stmt->fetch();

    if (!$order) {
        sendErrorResponse('订单不存在');
    }

    sendSuccessResponse(['status' => $order['status']]);
}

// 生成BEpusdt签名
function generateBepusdtSignature($data) {
    ksort($data);
    $stringToSign = implode('&', array_map(function($k, $v) {
        return $k . '=' . $v;
    }, array_keys($data), $data));

    return hash_hmac('sha256', $stringToSign, Config::get('bepusdt.auth_token'));
}

// 发送成功响应
function sendSuccessResponse($data = null) {
    echo json_encode([
        'success' => true,
        'data' => $data
    ]);
    exit;
}

// 发送错误响应
function sendErrorResponse($message) {
    http_response_code(400);
    echo json_encode([
        'success' => false,
        'message' => $message
    ]);
    exit;
}
?>
```

## 📧 支付回调处理

### BEpusdt回调处理程序

```php
<?php
// payment/notify.php - BEpusdt支付回调处理
header('Content-Type: text/plain');

// 引入配置文件
require_once '../config.php';
require_once '../database.php';

// 读取回调数据
$input = file_get_contents('php://input');
$data = json_decode($input, true);

// 验证签名
function verifySignature($data, $signature) {
    if (!isset($data['signature'])) {
        return false;
    }

    $receivedSignature = $data['signature'];
    unset($data['signature']);

    ksort($data);
    $stringToSign = implode('&', array_map(function($k, $v) {
        return $k . '=' . $v;
    }, array_keys($data), $data));

    $calculatedSignature = hash_hmac('sha256', $stringToSign, Config::get('bepusdt.auth_token'));

    return hash_equals($calculatedSignature, $receivedSignature);
}

// 记录日志
function logMessage($message) {
    $logFile = __DIR__ . '/payment.log';
    $timestamp = date('Y-m-d H:i:s');
    file_put_contents($logFile, "[$timestamp] $message\n", FILE_APPEND | LOCK_EX);
}

try {
    // 验证请求方法
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        logMessage('ERROR: 非POST请求');
        echo 'invalid method';
        exit;
    }

    // 验证签名
    if (!verifySignature($data, $data['signature'] ?? '')) {
        logMessage('ERROR: 签名验证失败 - ' . $input);
        echo 'invalid signature';
        exit;
    }

    // 验证必要参数
    $requiredFields = ['trade_id', 'order_id', 'status', 'amount', 'actual_amount'];
    foreach ($requiredFields as $field) {
        if (!isset($data[$field])) {
            logMessage("ERROR: 缺少必要参数 $field");
            echo 'missing parameters';
            exit;
        }
    }

    $orderId = $data['order_id'];
    $status = (int)$data['status'];

    $pdo = Database::getConnection();

    // 获取订单信息
    $stmt = $pdo->prepare("SELECT * FROM orders WHERE order_id = ?");
    $stmt->execute([$orderId]);
    $order = $stmt->fetch();

    if (!$order) {
        logMessage("ERROR: 订单不存在 - $orderId");
        echo 'order not found';
        exit;
    }

    // 防止重复处理
    if ($order['status'] === 'paid') {
        logMessage("INFO: 订单已处理 - $orderId");
        echo 'ok';
        exit;
    }

    // 更新订单状态
    if ($status === 2) { // 支付成功
        // 更新订单状态
        $stmt = $pdo->prepare("UPDATE orders SET status = 'paid' WHERE order_id = ?");
        $stmt->execute([$orderId]);

        logMessage("INFO: 订单支付成功 - $orderId, 开始自动发货");

        // 自动发货
        $deliveryResult = autoDeliverCard($order, $data);

        if ($deliveryResult['success']) {
            logMessage("INFO: 自动发货成功 - $orderId, 卡密: " . $deliveryResult['card_key']);
            echo 'ok';
        } else {
            logMessage("ERROR: 自动发货失败 - $orderId, 错误: " . $deliveryResult['error']);
            echo 'delivery failed';
        }

    } else if ($status === 3) { // 订单超时
        $stmt = $pdo->prepare("UPDATE orders SET status = 'expired' WHERE order_id = ?");
        $stmt->execute([$orderId]);

        // 恢复库存
        $stmt = $pdo->prepare("UPDATE products SET stock_count = stock_count + 1 WHERE id = ?");
        $stmt->execute([$order['product_id']]);

        logMessage("INFO: 订单超时 - $orderId, 已恢复库存");
        echo 'ok';

    } else if ($status === 6) { // 支付失败
        $stmt = $pdo->prepare("UPDATE orders SET status = 'failed' WHERE order_id = ?");
        $stmt->execute([$orderId]);

        // 恢复库存
        $stmt = $pdo->prepare("UPDATE products SET stock_count = stock_count + 1 WHERE id = ?");
        $stmt->execute([$order['product_id']]);

        logMessage("INFO: 订单支付失败 - $orderId, 已恢复库存");
        echo 'ok';

    } else {
        logMessage("WARNING: 未知订单状态 - $orderId, status: $status");
        echo 'ok';
    }

} catch (Exception $e) {
    logMessage("ERROR: 处理回调异常 - " . $e->getMessage());
    echo 'error';
}

// 自动发货函数
function autoDeliverCard($order, $paymentData) {
    $pdo = Database::getConnection();

    try {
        // 开始事务
        $pdo->beginTransaction();

        // 获取未使用的卡密
        $stmt = $pdo->prepare("SELECT * FROM card_keys WHERE product_id = ? AND is_used = FALSE ORDER BY id ASC LIMIT 1");
        $stmt->execute([$order['product_id']]);
        $cardKey = $stmt->fetch();

        if (!$cardKey) {
            // 卡密不足，标记订单需要人工处理
            $stmt = $pdo->prepare("UPDATE orders SET status = 'delivered' WHERE order_id = ?");
            $stmt->execute([$order['order_id']]);

            // 通知管理员
            notifyAdmin("卡密不足", "订单 {$order['order_id']} 需要人工处理");

            $pdo->commit();
            return ['success' => false, 'error' => '卡密不足'];
        }

        // 标记卡密为已使用
        $stmt = $pdo->prepare("
            UPDATE card_keys
            SET is_used = TRUE, used_by = ?, used_at = NOW(), order_id = ?
            WHERE id = ?
        ");
        $stmt->execute([$order['user_id'], $order['order_id'], $cardKey['id']]);

        // 更新订单信息
        $stmt = $pdo->prepare("
            UPDATE orders
            SET status = 'delivered', card_key = ?, delivery_time = NOW()
            WHERE order_id = ?
        ");
        $stmt->execute([$cardKey['card_key'], $order['order_id']]);

        // 获取用户信息
        $stmt = $pdo->prepare("SELECT email, username FROM users WHERE id = ?");
        $stmt->execute([$order['user_id']]);
        $user = $stmt->fetch();

        // 发送邮件通知
        $emailSent = sendDeliveryEmail($user, $order, $cardKey['card_key']);

        // 记录发货记录
        $stmt = $pdo->prepare("
            INSERT INTO delivery_records (order_id, card_key, user_id, product_id, delivery_method, recipient, delivery_status)
            VALUES (?, ?, ?, ?, 'email', ?, ?)
        ");
        $stmt->execute([
            $order['order_id'],
            $cardKey['card_key'],
            $order['user_id'],
            $order['product_id'],
            $user['email'],
            $emailSent ? 'sent' : 'failed'
        ]);

        $pdo->commit();

        return ['success' => true, 'card_key' => $cardKey['card_key']];

    } catch (Exception $e) {
        $pdo->rollback();
        logMessage("ERROR: 自动发货异常 - " . $e->getMessage());
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// 发送邮件通知
function sendDeliveryEmail($user, $order, $cardKey) {
    try {
        $to = $user['email'];
        $subject = '【二次元游戏发卡平台】您的订单已发货';

        $message = "
        <html>
        <head>
            <style>
                body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
                .container { max-width: 600px; margin: 0 auto; padding: 20px; }
                .header { text-align: center; padding: 20px 0; }
                .content { background: #f9f9f9; padding: 20px; border-radius: 8px; margin: 20px 0; }
                .card-key { background: #e8f5e8; padding: 15px; border-radius: 5px; font-size: 18px; font-weight: bold; text-align: center; margin: 15px 0; }
                .footer { text-align: center; font-size: 12px; color: #666; margin-top: 30px; }
            </style>
        </head>
        <body>
            <div class='container'>
                <div class='header'>
                    <h1>🎮 二次元游戏发卡平台</h1>
                </div>

                <div class='content'>
                    <h3>🎉 您的订单已发货！</h3>

                    <p><strong>订单号：</strong>{$order['order_id']}</p>
                    <p><strong>商品名称：</strong>{$order['product_name']}</p>
                    <p><strong>支付金额：</strong>¥{$order['product_price']}</p>

                    <div class='card-key'>
                        🔑 卡密：<br>
                        {$cardKey}
                    </div>

                    <p><strong>发货时间：</strong>" . date('Y-m-d H:i:s') . "</p>

                    <h4>🌟 温馨提示：</h4>
                    <ul>
                        <li>请在游戏内正确使用卡密</li>
                        <li>卡密有效期为一年，请及时使用</li>
                        <li>如有问题请联系客服</li>
                    </ul>
                </div>

                <div class='footer'>
                    <p>感谢您的信任，祝您游戏愉快！</p>
                    <p>如有问题，请联系客服QQ：123456789</p>
                </div>
            </div>
        </body>
        </html>
        ";

        // 设置邮件头
        $headers = [
            'MIME-Version: 1.0',
            'Content-type: text/html; charset=utf-8',
            'From: ' . Config::get('email.from_name') . ' <' . Config::get('email.username') . '>',
            'Reply-To: ' . Config::get('email.username')
        ];

        // 发送邮件
        return mail($to, $subject, $message, implode("\r\n", $headers));

    } catch (Exception $e) {
        logMessage("ERROR: 发送邮件失败 - " . $e->getMessage());
        return false;
    }
}

// 通知管理员
function notifyAdmin($subject, $message) {
    $to = Config::get('admin_email');
    $adminMessage = "【系统通知】$subject\n\n$message";

    $headers = 'From: ' . Config::get('email.from_name') . ' <' . Config::get('email.username') . '>';

    mail($to, $subject, $adminMessage, $headers);
}
?>
```

## 🌐 前端页面代码

### 完整的前端页面

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>二次元游戏发卡平台</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Microsoft YaHei', Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            color: #333;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
        }

        .header {
            text-align: center;
            margin-bottom: 40px;
            color: white;
        }

        .header h1 {
            font-size: 3em;
            margin-bottom: 10px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
        }

        .header p {
            font-size: 1.2em;
            opacity: 0.9;
        }

        .product-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 20px;
            margin-bottom: 40px;
        }

        .product-card {
            background: white;
            border-radius: 15px;
            padding: 25px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.1);
            transition: all 0.3s ease;
            position: relative;
            overflow: hidden;
        }

        .product-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 15px 40px rgba(0,0,0,0.15);
        }

        .product-card h3 {
            color: #333;
            margin-bottom: 15px;
            font-size: 1.3em;
        }

        .product-card .price {
            color: #ff6b6b;
            font-size: 2em;
            font-weight: bold;
            margin-bottom: 10px;
        }

        .product-card .category {
            background: linear-gradient(45deg, #667eea, #764ba2);
            color: white;
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 0.9em;
            display: inline-block;
            margin-bottom: 15px;
        }

        .product-card .stock {
            color: #666;
            margin-bottom: 20px;
        }

        .buy-btn {
            background: linear-gradient(45deg, #667eea, #764ba2);
            color: white;
            border: none;
            padding: 15px 30px;
            border-radius: 25px;
            cursor: pointer;
            font-size: 1.1em;
            font-weight: bold;
            transition: all 0.3s ease;
            width: 100%;
        }

        .buy-btn:hover {
            transform: scale(1.05);
            box-shadow: 0 5px 20px rgba(0,0,0,0.2);
        }

        .buy-btn:disabled {
            background: #ccc;
            cursor: not-allowed;
            transform: none;
        }

        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.8);
            z-index: 1000;
            animation: fadeIn 0.3s ease;
        }

        @keyframes fadeIn {
            from { opacity: 0; }
            to { opacity: 1; }
        }

        .modal-content {
            background: white;
            margin: 5% auto;
            padding: 40px;
            width: 90%;
            max-width: 600px;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            animation: slideIn 0.3s ease;
        }

        @keyframes slideIn {
            from { transform: translateY(-50px); opacity: 0; }
            to { transform: translateY(0); opacity: 1; }
        }

        .modal-content h3 {
            text-align: center;
            color: #333;
            margin-bottom: 20px;
            font-size: 1.8em;
        }

        .payment-info {
            text-align: center;
            margin: 30px 0;
        }

        .qr-code-container {
            margin: 20px 0;
            text-align: center;
        }

        .qr-code-container img {
            border: 2px solid #ddd;
            border-radius: 10px;
            box-shadow: 0 5px 15px rgba(0,0,0,0.1);
        }

        .payment-details {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 10px;
            margin: 20px 0;
            border-left: 4px solid #667eea;
        }

        .payment-details p {
            margin: 10px 0;
            font-size: 1em;
        }

        .payment-details strong {
            color: #333;
        }

        .copy-btn {
            background: #28a745;
            color: white;
            border: none;
            padding: 8px 15px;
            border-radius: 5px;
            cursor: pointer;
            margin: 5px;
            font-size: 0.9em;
            transition: background 0.3s ease;
        }

        .copy-btn:hover {
            background: #218838;
        }

        .status-text {
            font-size: 1.3em;
            font-weight: bold;
            margin: 30px 0;
            padding: 15px;
            background: #f0f8ff;
            border-radius: 10px;
            border-left: 4px solid #667eea;
        }

        .close-btn {
            background: #6c757d;
            color: white;
            border: none;
            padding: 15px 30px;
            border-radius: 25px;
            cursor: pointer;
            width: 100%;
            margin-top: 20px;
            font-size: 1.1em;
            transition: background 0.3s ease;
        }

        .close-btn:hover {
            background: #5a6268;
        }

        .stats {
            background: white;
            padding: 30px;
            border-radius: 15px;
            margin-bottom: 40px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.1);
            text-align: center;
        }

        .stats h2 {
            color: #333;
            margin-bottom: 20px;
        }

        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin-top: 20px;
        }

        .stat-item {
            padding: 20px;
            background: #f8f9fa;
            border-radius: 10px;
            border-left: 4px solid #667eea;
        }

        .stat-number {
            font-size: 2em;
            font-weight: bold;
            color: #667eea;
        }

        .stat-label {
            color: #666;
            margin-top: 5px;
        }

        @media (max-width: 768px) {
            .header h1 {
                font-size: 2em;
            }

            .product-grid {
                grid-template-columns: 1fr;
            }

            .modal-content {
                margin: 10% auto;
                padding: 20px;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🎮 二次元游戏发卡平台</h1>
            <p>安全、快捷、自动发货 | 24小时营业</p>
        </div>

        <!-- 统计信息 -->
        <div class="stats">
            <h2>📊 平台数据</h2>
            <div class="stats-grid">
                <div class="stat-item">
                    <div class="stat-number">15+</div>
                    <div class="stat-label">游戏支持</div>
                </div>
                <div class="stat-item">
                    <div class="stat-number">1000+</div>
                    <div class="stat-label">累计售出</div>
                </div>
                <div class="stat-item">
                    <div class="stat-number">99.9%</div>
                    <div class="stat-label">发货成功率</div>
                </div>
                <div class="stat-item">
                    <div class="stat-number">24/7</div>
                    <div class="stat-label">自动服务</div>
                </div>
            </div>
        </div>

        <!-- 商品展示区 -->
        <div class="product-grid" id="productGrid">
            <!-- JavaScript动态加载商品 -->
        </div>
    </div>

    <!-- 支付弹窗 -->
    <div id="paymentModal" class="modal">
        <div class="modal-content">
            <h3 id="productTitle">商品名称</h3>
            <p style="text-align: center; font-size: 20px; color: #ff6b6b; margin-bottom: 20px;">
                价格：¥<span id="productPrice">0</span>
            </p>

            <div id="paymentInfo" style="display:none;">
                <h4>💳 支付信息</h4>

                <div class="qr-code-container">
                    <div id="qrcode"></div>
                    <p><small>请使用TRON钱包扫描二维码支付</small></p>
                    <p><small>支持TokenPocket、TronLink等钱包</small></p>
                </div>

                <div class="payment-details">
                    <p><strong>💰 支付金额：</strong><span id="usdtAmount" style="color: #ff6b6b; font-weight: bold;">0</span> USDT</p>
                    <p><strong>🏠 收款地址：</strong><br><code id="walletAddress" style="word-break: break-all; background: #f0f0f0; padding: 5px; border-radius: 3px;"></code></p>
                    <p><strong>🌐 网络类型：</strong>TRON (TRC20)</p>
                    <p><strong>⏰ 有效期：</strong><span id="expireTime">30</span> 分钟</p>
                    <div style="margin-top: 15px;">
                        <button class="copy-btn" onclick="copyAddress()">📋 复制地址</button>
                        <button class="copy-btn" onclick="copyAmount()">📋 复制金额</button>
                        <button class="copy-btn" onclick="copyAll()">📋 复制全部</button>
                    </div>
                </div>

                <div class="status-text" id="statusText">
                    ⏳ 等待支付中...
                </div>

                <div style="margin-top: 20px; padding: 15px; background: #fff3cd; border-radius: 8px; border-left: 4px solid #ffc107;">
                    <p style="font-size: 0.9em; color: #856404;">
                        ⚠️ <strong>重要提醒：</strong><br>
                        1. 请确保转账金额完全一致<br>
                        2. 请使用TRON (TRC20) 网络<br>
                        3. 支付后请等待1-3分钟自动发货<br>
                        4. 如有问题请联系客服QQ：123456789
                    </p>
                </div>
            </div>

            <button class="close-btn" onclick="closeModal()">关闭窗口</button>
        </div>
    </div>

    <script>
        let currentTradeId = null;
        let statusCheckInterval = null;

        // 加载商品列表
        async function loadProducts() {
            try {
                showLoading();
                const response = await fetch('/api/products');
                const products = await response.json();
                hideLoading();

                const grid = document.getElementById('productGrid');
                if (products.length === 0) {
                    grid.innerHTML = `
                        <div style="grid-column: 1 / -1; text-align: center; padding: 40px; background: white; border-radius: 15px;">
                            <h3 style="color: #666;">📦 暂无商品</h3>
                            <p style="color: #999;">商品正在上架中，请稍后再来</p>
                        </div>
                    `;
                    return;
                }

                grid.innerHTML = products.map(product => `
                    <div class="product-card">
                        <div style="text-align: center; margin-bottom: 15px;">
                            <span class="category">${getCategoryEmoji(product.category)} ${product.category}</span>
                        </div>
                        <h3>🎮 ${product.name}</h3>
                        <div style="color: #666; font-size: 0.9em; margin-bottom: 10px;">${product.description || ''}</div>
                        <div class="price">¥${product.price}</div>
                        <div class="stock">库存：${product.stock_count} 件</div>
                        <div style="margin-top: 15px;">
                            <div style="display: flex; gap: 10px;">
                                <div style="flex: 1; background: #e8f5e8; padding: 8px; border-radius: 5px; text-align: center; font-size: 0.9em;">
                                    🎯 ${product.sales_count || 0} 人已购买
                                </div>
                            </div>
                        </div>
                        <button class="buy-btn" onclick="buyProduct(${product.id}, '${product.name}', ${product.price})"
                                ${product.stock_count <= 0 ? 'disabled' : ''}>
                            ${product.stock_count <= 0 ? '暂时缺货' : '立即购买'}
                        </button>
                    </div>
                `).join('');
            } catch (error) {
                hideLoading();
                console.error('加载商品失败:', error);
                document.getElementById('productGrid').innerHTML =
                    '<div style="grid-column: 1 / -1; text-align: center; padding: 40px; background: white; border-radius: 15px; color: red;">' +
                    '<h3>❌ 加载商品失败</h3>' +
                    '<p>请检查网络连接或联系客服</p>' +
                    '</div>';
            }
        }

        // 获取分类emoji
        function getCategoryEmoji(category) {
            const categoryMap = {
                '原神': '🌟',
                '王者荣耀': '⚔️',
                '和平精英': '🎯',
                '崩坏星穹铁道': '🚀',
                '英雄联盟': '🗡️',
                'DNF': '⚔️',
                'QQ飞车': '🏎️'
            };
            return categoryMap[category] || '🎮';
        }

        // 购买商品
        async function buyProduct(productId, productName, price) {
            // 检查用户登录
            const userId = getCurrentUserId();
            if (!userId) {
                alert('请先登录后再购买');
                return;
            }

            document.getElementById('productTitle').textContent = productName;
            document.getElementById('productPrice').textContent = price;
            document.getElementById('paymentModal').style.display = 'block';

            try {
                // 显示加载状态
                document.getElementById('statusText').innerHTML = '🔄 正在创建支付订单，请稍候...';

                // 调用后端创建BEpusdt订单
                const response = await fetch('/api/create-order', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        product_id: productId,
                        user_id: userId
                    })
                });

                const result = await response.json();

                if (result.success) {
                    // 显示支付信息
                    document.getElementById('paymentInfo').style.display = 'block';
                    document.getElementById('usdtAmount').textContent = result.usdt_amount;
                    document.getElementById('walletAddress').textContent = result.wallet_address;
                    document.getElementById('expireTime').textContent = '30';
                    currentTradeId = result.trade_id;

                    // 生成二维码
                    generateQRCode(result.wallet_address, result.usdt_amount);

                    // 开始检查支付状态
                    document.getElementById('statusText').innerHTML = '⏳ 等待用户支付，请使用TRON钱包扫描上方二维码';
                    startPaymentStatusCheck(result.trade_id);
                } else {
                    alert('创建订单失败：' + result.message);
                    closeModal();
                }
            } catch (error) {
                console.error('创建订单失败:', error);
                alert('网络错误，请重试');
                closeModal();
            }
        }

        // 生成二维码
        function generateQRCode(address, amount) {
            const qrData = `TRON:${address}:${amount}:usdt.trc20`;
            const qrUrl = `https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${encodeURIComponent(qrData)}`;
            document.getElementById('qrcode').innerHTML = `<img src="${qrUrl}" alt="支付二维码" style="border: 2px solid #ddd; border-radius: 8px;">`;
        }

        // 开始检查支付状态
        function startPaymentStatusCheck(tradeId) {
            // 清除之前的定时器
            if (statusCheckInterval) {
                clearInterval(statusCheckInterval);
            }

            let checkCount = 0;
            const maxChecks = 60; // 最多检查10分钟 (60 * 10秒)

            statusCheckInterval = setInterval(async () => {
                checkCount++;

                try {
                    const response = await fetch(`/api/check-payment/${tradeId}`);
                    const result = await response.json();

                    if (result.status === 'paid') {
                        // 支付成功
                        clearInterval(statusCheckInterval);
                        document.getElementById('statusText').innerHTML =
                            '✅ 支付成功！卡密已发送到您的邮箱<br>页面将在3秒后关闭...';

                        setTimeout(() => {
                            alert('🎉 支付成功！请查收邮件获取卡密\n如果未收到邮件，请检查垃圾邮件箱');
                            closeModal();
                            // 可以跳转到订单页面
                            window.location.href = '/my-orders';
                        }, 3000);

                    } else if (result.status === 'expired') {
                        // 订单超时
                        clearInterval(statusCheckInterval);
                        document.getElementById('statusText').innerHTML =
                            '⏰ 订单已超时，请重新购买';

                    } else if (result.status === 'not_found') {
                        // 订单不存在
                        if (checkCount > 5) {
                            clearInterval(statusCheckInterval);
                            document.getElementById('statusText').innerHTML =
                                '❌ 订单状态异常，请刷新页面重试';
                        }
                    }

                    // 更新检查次数显示
                    if (checkCount < maxChecks) {
                        const remainingTime = Math.max(0, maxChecks - checkCount);
                        if (remainingTime % 6 === 0) { // 每分钟更新一次
                            console.log(`继续检查支付状态... 剩余检查次数: ${remainingTime}`);
                        }
                    }

                } catch (error) {
                    console.error('检查支付状态失败:', error);
                    if (checkCount > 3) {
                        clearInterval(statusCheckInterval);
                        document.getElementById('statusText').innerHTML =
                            '❌ 网络错误，请刷新页面重试';
                    }
                }
            }, 10000); // 每10秒检查一次

            // 设置超时检查
            setTimeout(() => {
                if (statusCheckInterval) {
                    clearInterval(statusCheckInterval);
                    statusCheckInterval = null;
                    document.getElementById('statusText').innerHTML =
                        '⏰ 检查超时，请手动确认支付状态';
                }
            }, 600000); // 10分钟超时
        }

        // 复制地址到剪贴板
        function copyAddress() {
            const address = document.getElementById('walletAddress').textContent;
            copyToClipboard(address, '地址');
        }

        // 复制金额到剪贴板
        function copyAmount() {
            const amount = document.getElementById('usdtAmount').textContent;
            copyToClipboard(amount, '金额');
        }

        // 复制全部信息
        function copyAll() {
            const address = document.getElementById('walletAddress').textContent;
            const amount = document.getElementById('usdtAmount').textContent;
            const allInfo = `收款地址：${address}\n支付金额：${amount} USDT\n网络：TRON (TRC20)`;
            copyToClipboard(allInfo, '完整信息');
        }

        // 通用复制函数
        function copyToClipboard(text, type) {
            navigator.clipboard.writeText(text).then(() => {
                alert(`✅ ${type}已复制到剪贴板`);
            }).catch(() => {
                // 备用方案
                const textArea = document.createElement('textarea');
                textArea.value = text;
                document.body.appendChild(textArea);
                textArea.select();
                document.execCommand('copy');
                document.body.removeChild(textArea);
                alert(`✅ ${type}已复制到剪贴板`);
            });
        }

        function closeModal() {
            document.getElementById('paymentModal').style.display = 'none';
            if (statusCheckInterval) {
                clearInterval(statusCheckInterval);
                statusCheckInterval = null;
            }
        }

        function getCurrentUserId() {
            // 获取当前登录用户的ID（从cookie、session等）
            // 这里需要根据你的用户系统实现
            return getCookie('user_id') || sessionStorage.getItem('user_id') || localStorage.getItem('user_id');
        }

        function getCookie(name) {
            const value = `; ${document.cookie}`;
            const parts = value.split(`; ${name}=`);
            if (parts.length === 2) return parts.pop().split(';').shift();
        }

        // 显示加载状态
        function showLoading() {
            document.getElementById('productGrid').innerHTML = `
                <div style="grid-column: 1 / -1; text-align: center; padding: 40px; background: white; border-radius: 15px;">
                    <div style="display: inline-block; padding: 20px;">
                        <div style="border: 4px solid #667eea; border-radius: 50%; width: 40px; height: 40px; border-top: none; border-right: none; animation: spin 1s linear infinite; margin: 0 auto 10px;"></div>
                        <div>正在加载商品...</div>
                    </div>
                </div>
            `;
        }

        function hideLoading() {
            // 加载完成，内容会被替换
        }

        // 点击模态框外部关闭
        window.onclick = function(event) {
            const modal = document.getElementById('paymentModal');
            if (event.target === modal) {
                closeModal();
            }
        }

        // ESC键关闭模态框
        document.addEventListener('keydown', function(event) {
            if (event.key === 'Escape') {
                closeModal();
            }
        });

        // 页面加载时执行
        document.addEventListener('DOMContentLoaded', function() {
            loadProducts();

            // 添加页面可见性检测
            document.addEventListener('visibilitychange', function() {
                if (!document.hidden) {
                    // 页面重新可见时，刷新商品列表
                    loadProducts();
                }
            });

            // 添加网络状态检测
            window.addEventListener('online', function() {
                console.log('网络连接正常');
            });

            window.addEventListener('offline', function() {
                alert('网络连接已断开，请检查网络设置');
            });
        });

        // 错误处理
        window.addEventListener('error', function(event) {
            console.error('页面加载错误:', event.error);
        });

        // 添加页面性能监控
        if (window.performance) {
            window.addEventListener('load', function() {
                const perfData = performance.timing;
                const loadTime = perfData.loadEventEnd - perfData.navigationStart;
                console.log(`页面加载完成，用时：${loadTime}ms`);
            });
        }
    </script>
</body>
</html>
```

## ⚙️ BEpusdt配置

### BEpusdt配置文件

```toml
# conf.toml
app_uri = "https://your-game-card-platform.com"
auth_token = "your_secure_auth_token_here"
listen = ":8080"
output_log = "/var/log/bepusdt/app.log"
sqlite_path = "./data/bepusdt.db"

# 支付配置
[pay]
trx_atom = 0.000001
trx_rate = ""
usdt_atom = 0.000001
usdc_atom = 0.000001
usdt_rate = ""
usdc_rate = ""
expire_time = 1800
wallet_address = [
    "TXxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # 你的TRON地址
]

# EVM RPC节点配置
[evm_rpc]
bsc = "https://bsc-dataseed1.binance.org/"
ethereum = "https://eth-mainnet.alchemyapi.io/v2/your-api-key"
polygon = "https://polygon-rpc.com/"
arbitrum = "https://arb1.arbitrum.io/rpc"
plasma = "https://rpc.plasma.network"
base = "https://mainnet.base.org"
xlayer = "https://rpc.xlayer.tech/"
solana = "https://api.mainnet-beta.solana.com"
aptos = "https://fullnode.mainnet.aptoslabs.com/v1"

# TRON网络配置
tron_grpc_node = "grpc.trongrid.io:50091"
aptos_rpc_node = "https://fullnode.mainnet.aptoslabs.com/v1"

# Webhook配置（可选）
webhook_url = "https://your-game-card-platform.com/webhook"

# 不配置Bot，使用网站通知
```

## ✅ 总结

### 完整流程总结

整个自动化发卡流程如下：

```
用户浏览商品 → 点击购买 → 创建BEpusdt订单 → 显示支付页面 → 用户扫码支付
                                    ↓
                         BEpusdt监听区块链 → 确认支付 → 调用你的回调接口
                                    ↓
                         你的网站自动发货 → 用户收到卡密 → 完成交易
```

### BEpusdt和发卡系统的联系点

1. **API调用** - 你的网站调用BEpusdt创建支付订单
2. **回调通知** - BEpusdt支付成功后调用你的notify.php
3. **发货触发** - notify.php中的支付成功判断触发自动发货
4. **业务完成** - 自动发送卡密给用户

### 关键文件

- `index.html` - 前端商品展示和支付页面
- `api.php` - 后端API接口，调用BEpusdt创建订单
- `payment/notify.php` - BEpusdt回调处理，自动发货逻辑
- `database.sql` - 数据库表结构

### 关键流程

1. **用户下单** → 调用BEpusdt API → 生成支付二维码
2. **用户支付** → BEpusdt监听区块链 → 检测交易
3. **支付确认** → BEpusdt调用notify.php → 验证签名
4. **自动发货** → 从数据库取卡密 → 发送邮件给用户
5. **完成交易** → 用户收到卡密 → 自动完成

### 核心优势

- **完全自动化** - 无需人工干预，24小时自动营业
- **安全可靠** - 使用区块链支付，无法伪造
- **用户体验好** - 扫码即付，自动发货
- **成本低廉** - 只需服务器成本，无手续费
- **扩展性强** - 支持多种游戏和商品类型
- **监控完善** - 完整的日志和监控机制

现在你就有了一个完全自动化的二次元游戏卡密发卡平台！用户随时购买，随时自动发货，真正实现躺着赚钱！