# BEpusdt 端口配置 - FAKA 系统集成指南

## 📍 当前端口配置

### 对外端口：8080
```yaml
ports:
  - "8080:8080"  # 主机端口:容器端口
```

### 服务内部端口：8080
```toml
listen = ":8080"
```

## 🔗 与异次元等FAKA网站集成

### 端口配置方面：
- ✅ **不需要特殊端口** - 8080端口完全适用
- ✅ **标准HTTP端口** - 大多数FAKA系统都支持自定义端口
- ✅ **反向代理支持** - 可以通过Nginx等代理到80/443

## 📋 集成需要的关键信息

### 1. API端点地址：
```
http://your-server-ip:8080
```

### 2. 认证Token：
```toml
auth_token = "715705zym"  # 当前配置，建议修改
```

### 3. 重要配置项：
```toml
app_uri = "https://coin.pay.com"  # 需要修改为你的实际域名
```

## 🚀 推荐的部署方式

### 方案1：直接使用8080端口
```
API地址: http://your-server-ip:8080
```

### 方案2：使用Nginx反向代理（推荐）
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
然后API地址变为：`http://your-domain.com`

## 📝 FAKA系统集成步骤

### 1. 修改配置文件：
- 更新 `app_uri` 为你的实际域名
- 修改 `auth_token` 为安全的token

### 2. 在FAKA系统中配置：
- 支付网关地址：`http://your-server-ip:8080/api`
- 认证密钥：使用 `auth_token` 的值

### 3. 网络要求：
- 确保服务器防火墙允许8080端口
- 如果使用Nginx代理，确保80/443端口开放

## 💡 总结

**不需要特殊端口配置，8080端口完全满足FAKA系统集成需求！** 🚀