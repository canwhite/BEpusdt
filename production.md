# BEpusdt 项目文档

> 最后更新：2026-01-04

## 项目定位

BEpusdt (Better Easy Payment USDT) 是一个基于 Go 语言开发的 USDT/USDC 区块链收款监控系统，支持多网络、实时汇率同步、无外部依赖（Redis/MySQL），适合个人和小型商户使用。

## 核心架构

### 技术栈
- **语言**: Go 1.23
- **Web框架**: Gin v1.9
- **数据库**: SQLite（无 MySQL/Redis 依赖）
- **部署**: Docker / systemd / 1Panel / 宝塔
- **通信**: Telegram Bot（可选）、WebHook、易支付API

### 支持的区块链网络
- 主流：TRON、Ethereum、BSC、Polygon
- 其他：X-Layer、Solana、Aptos、Arbitrum-One、Base 等

### 核心功能
- 区块链交易扫描和监控
- USDT/USDC 实时汇率同步
- 订单管理和回调
- 钱包余额变动通知
- 能量代理与回收（波场）
- 易支付兼容接口

## 目录结构

```
BEpusdt/
├── app/              # 应用核心代码
├── main/             # 主程序入口
├── data/             # 数据存储（SQLite数据库）
├── static/           # 静态资源
├── docs/             # 文档
├── changed/          # 变更记录
├── conf.toml         # 配置文件
├── docker-compose.yml
└── Dockerfile
```

## 部署流程

### Docker 部署
1. 配置 `conf.toml`
2. 运行 `docker-compose up -d`
3. 数据持久化在 `data/` 目录

### 系统服务部署
1. 配置 systemd 服务
2. 启动服务：`systemctl start bepusdt`
3. 配置时钟同步（订单强依赖时间）

## 关键配置

- **数据库**: SQLite (`data/sqlite.db`)
- **配置文件**: `conf.toml`
- **端口**: 8085（根据 git log 最近修改）
- **时间同步**: 强制要求（订单功能依赖）

## 注意事项

⚠️ **重要**：
- 数据库文件 `data/sqlite.db` 应该被 gitignore，不应提交到版本控制
- 配置文件 `conf.toml` 包含敏感信息，不应提交
- 确保服务器时间准确性
- 确保网络纯洁性（部分功能依赖外部网络）

## Git 状态

当前分支：`main`
最近变更：
- 修改端口为 8085
- 移除 bot 依赖
- 本地镜像构建优化
