# T1K 模块编译工具

编译 OpenResty/Nginx 的 T1K 动态模块（雷池 WAF 检测模块）。

## 使用方法

用户提供以下信息，AI 自动完成编译：

**必需信息：**
1. 服务器连接信息：`IP地址/用户名/密码` 或 SSH 免密地址
2. 编译资源（二选一）：
   - **源码方式**：t1k 源码目录路径
   - **工具方式**：t1k_cli 编译工具路径 + 密码文件路径

**可选信息：**
- nginx 源码路径（默认自动查找）
- nginx 安装路径（默认 `/usr/local/openresty`）

## 编译流程

### 步骤 1：连接服务器并收集信息

```bash
# 连接服务器（如有密码则使用 sshpass）
sshpass -p '密码' ssh root@192.168.1.100 "命令"

# 或免密登录
ssh root@192.168.1.100 "命令"

# 获取系统信息
uname -r && cat /etc/os-release

# 获取 nginx 版本和编译参数
/usr/local/openresty/nginx/sbin/nginx -V 2>&1

# 查找 nginx 源码目录
find /usr/local/src -name "nginx-*.tar.gz" -o -name "openresty-*" -type d
```

### 步骤 2：选择编译方式

#### 方式 A：使用源码目录编译（推荐）

**适用场景：** 已有 t1k 源码目录

```bash
T1K_SOURCE=/usr/local/src/t1k_source
NGINX_SRC=/usr/local/src/openresty-1.31.1.1/build/nginx-1.31.1

cd $NGINX_SRC

# 设置 LuaJIT 环境变量
export LUAJIT_LIB=/usr/local/openresty/luajit/lib
export LUAJIT_INC=/usr/local/openresty/luajit/include/luajit-2.1

# 配置（使用完整参数）
./configure \
  --prefix=/usr/local/openresty/nginx \
  --with-cc-opt=-O2 \
  --add-module=../../bundle/ngx_devel_kit-0.3.4 \
  --add-module=../../bundle/echo-nginx-module-0.64 \
  --add-module=../../bundle/xss-nginx-module-0.07 \
  --add-module=../../bundle/ngx_coolkit-0.2 \
  --add-module=../../bundle/set-misc-nginx-module-0.33 \
  --add-module=../../bundle/form-input-nginx-module-0.12 \
  --add-module=../../bundle/encrypted-session-nginx-module-0.09 \
  --add-module=../../bundle/srcache-nginx-module-0.33 \
  --add-module=../../bundle/ngx_lua-0.10.31rc5 \
  --add-module=../../bundle/ngx_lua_upstream-0.08 \
  --add-module=../../bundle/headers-more-nginx-module-0.39 \
  --add-module=../../bundle/array-var-nginx-module-0.06 \
  --add-module=../../bundle/memc-nginx-module-0.20 \
  --add-module=../../bundle/redis2-nginx-module-0.15 \
  --add-module=../../bundle/redis-nginx-module-0.41 \
  --add-module=../../bundle/rds-json-nginx-module-0.17 \
  --add-module=../../bundle/rds-csv-nginx-module-0.10 \
  --add-module=../../bundle/ngx_stream_lua-0.0.19rc4 \
  --with-ld-opt="-Wl,-rpath,/usr/local/openresty/luajit/lib" \
  --with-stream --with-stream_ssl_module \
  --with-stream_ssl_preread_module --with-http_ssl_module \
  --add-dynamic-module=$T1K_SOURCE

# 编译模块
make modules

# 处理模块
strip objs/ngx_http_t1k_core_module.so
```

#### 方式 B：使用 t1k_cli 工具编译

**适用场景：** 只有 t1k_cli 编译工具和密码

```bash
T1K_CLI=/usr/local/src/t1k_cli_amd64
T1K_PASS=$(cat /usr/local/src/t1k_cli_amd64.passwd)
NGINX_SRC=/usr/local/src/openresty-1.31.1.1/build/nginx-1.31.1

cd $NGINX_SRC

# 复制工具到源码目录
cp $T1K_CLI .

# 编译并保存源码
./t1k_cli_amd64 -p "$T1K_PASS" -- "./configure \
  --prefix=/usr/local/openresty/nginx \
  --with-cc-opt=-O2 \
  --with-ld-opt=\"-Wl,-rpath,/usr/local/openresty/luajit/lib\" \
  --with-stream --with-stream_ssl_module \
  --with-stream_ssl_preread_module --with-http_ssl_module \
  --add-dynamic-module=\${t1k_dir} \
  && cp -r \${t1k_dir} /usr/local/src/t1k_source \
  && make modules"

# 处理模块
strip objs/ngx_http_t1k_core_module.so
```

### 步骤 3：部署模块

```bash
# 创建模块目录
mkdir -p /usr/local/openresty/nginx/modules

# 复制模块
cp objs/ngx_http_t1k_core_module.so /usr/local/openresty/nginx/modules/

# 验证
ls -la /usr/local/openresty/nginx/modules/
```

### 步骤 4：配置 safeline.conf

**完整配置模板：**

```bash
cat > /usr/local/openresty/nginx/conf/safeline.conf << 'EOF'
# ============================================
# T1K 模块配置文件 - 雷池 WAF 检测节点配置
# ============================================

# 检测节点 upstream（多节点配置示例）
upstream detector_server {
    keepalive 256;
    
    # 检测节点 IP，按实际环境修改
    # 格式：server IP:PORT weight=权重 max_fails=失败次数 fail_timeout=超时时间
    server 172.16.155.130:8000 weight=1 max_fails=3 fail_timeout=60s;
    server 172.16.155.131:8000 weight=1 max_fails=3 fail_timeout=60s;
    
    # 如有 nginx_upstream_check_module 健康检查模块可开启
    # check interval=5000 rise=1 fall=3 port=8001 type=http;
}

# ============================================
# 请求检测配置
# ============================================

# 启用请求检测，拦截恶意请求
t1k_intercept @safeline;

# 请求体最大转发大小（超过此大小不检测）
t1k_body_size 1m;

# 访问日志记录（单位：字节，0表示关闭）
t1k_ulog 10000;

# 统计信息记录
t1k_stat 10000;

# 额外信息配置
t1k_extra_header on;    # 记录额外请求头信息
t1k_extra_body on;      # 记录额外请求体信息

# ============================================
# 响应检测配置（可选，默认关闭）
# ============================================

# 启用响应检测（检测响应体是否包含恶意内容）
# tx_intercept @safelinex;

# 响应体最大转发大小
# tx_body_size 4k;

# 响应体检测触发大小（响应超过此大小才开始检测）
# tx_max_detect_size 1m;

# 忽略检测的响应类型（这些类型不进行响应检测）
tx_ignore_types
    image/gif
    image/png
    image/jpeg
    image/webp
    image/svg+xml
    audio/wave
    audio/wav
    audio/x-wav
    audio/x-pn-wav
    audio/webm
    audio/ogg
    video/webm
    video/ogg
    application/ogg
    application/octet-stream
    application/pdf;

# ============================================
# 内部 location 配置（必须）
# ============================================

foreach_server {
    # 请求检测内部 location
    location @safeline {
        internal;
        t1k_pass detector_server;
        t1k_connect_timeout 1s;    # 连接超时
        t1k_read_timeout 1s;       # 读取超时
        t1k_send_timeout 1s;       # 发送超时
    }
    
    # 响应检测内部 location（如启用响应检测需配置）
    location @safelinex {
        internal;
        tx_pass detector_server;
        tx_connect_timeout 1s;
        tx_read_timeout 1s;
        tx_send_timeout 1s;
    }
}
EOF
```

---

## safeline.conf 配置模板

根据不同场景选择合适的配置模板。

### 模板一：基础配置（单节点）

适用于测试环境或单检测节点场景。

```nginx
# safeline.conf - 基础配置模板
upstream detector_server {
    keepalive 256;
    server 192.168.1.100:8000 weight=1 max_fails=3 fail_timeout=60s;
}

t1k_intercept @safeline;
t1k_body_size 1m;

foreach_server {
    location @safeline {
        internal;
        t1k_pass detector_server;
        t1k_connect_timeout 1s;
        t1k_read_timeout 1s;
        t1k_send_timeout 1s;
    }
}
```

### 模板二：高可用配置（多节点）

适用于生产环境，多检测节点负载均衡。

```nginx
# safeline.conf - 高可用配置模板
upstream detector_server {
    keepalive 256;
    
    # 多检测节点负载均衡
    server 192.168.1.100:8000 weight=1 max_fails=3 fail_timeout=60s;
    server 192.168.1.101:8000 weight=1 max_fails=3 fail_timeout=60s;
    server 192.168.1.102:8000 weight=1 max_fails=3 fail_timeout=60s;
    
    # 失败后快速切换
    # 如有 nginx_upstream_check_module 可开启健康检查
    # check interval=5000 rise=1 fall=3 port=8001 type=http;
}

t1k_intercept @safeline;
t1k_body_size 1m;
t1k_ulog 10000;
t1k_stat 10000;
t1k_extra_header on;
t1k_extra_body on;

foreach_server {
    location @safeline {
        internal;
        t1k_pass detector_server;
        t1k_connect_timeout 1s;
        t1k_read_timeout 1s;
        t1k_send_timeout 1s;
    }
}
```

### 模板三：完整生产配置

适用于生产环境，包含请求检测和响应检测。

```nginx
# safeline.conf - 生产环境完整配置
upstream detector_server {
    keepalive 512;
    
    # 主检测节点
    server 192.168.1.100:8000 weight=2 max_fails=2 fail_timeout=30s;
    # 备检测节点
    server 192.168.1.101:8000 weight=1 max_fails=2 fail_timeout=30s backup;
}

# ===== 请求检测配置 =====
t1k_intercept @safeline;
t1k_body_size 1m;           # 允许检测 5MB 请求体
t1k_ulog 10000;             # 记录访问日志
t1k_stat 10000;             # 记录统计信息
t1k_extra_header on;        # 记录额外请求头
t1k_extra_body on;          # 记录额外请求体

# ===== 响应检测配置 =====
# 启用响应检测，检测响应体中的敏感信息泄露
tx_intercept @safelinex;
tx_body_size 4k;            # 响应体检测上限
tx_max_detect_size 500k;    # 响应超过 500KB 才开始检测

# 忽略检测的响应类型（媒体文件不检测）
tx_ignore_types
    image/gif
    image/png
    image/jpeg
    image/webp
    image/svg+xml
    audio/wave
    audio/wav
    audio/webm
    audio/ogg
    video/webm
    video/ogg
    video/mp4
    application/ogg
    application/octet-stream
    application/pdf
    application/zip;

foreach_server {
    # 请求检测 location
    location @safeline {
        internal;
        t1k_pass detector_server;
        t1k_connect_timeout 1s;
        t1k_read_timeout 3s;
        t1k_send_timeout 1s;
    }
    
    # 响应检测 location
    location @safelinex {
        internal;
        tx_pass detector_server;
        tx_connect_timeout 1s;
        tx_read_timeout 1s;
        tx_send_timeout 1s;
    }
}
```

### 模板四：响应检测补充配置

通常响应检测与请求检测一起启用，不建议单独开启响应检测。
响应检测主要用于检测敏感信息泄露，推荐 tx_body_size 设置为 4k。

```nginx
# safeline.conf - 请求+响应检测配置
upstream detector_server {
    keepalive 256;
    server 192.168.1.100:8000;
}

# ===== 请求检测（必开）=====
t1k_intercept @safeline;
t1k_body_size 1m;           # 请求体检测上限：1MB

# ===== 响应检测（可选）=====
tx_intercept @safelinex;
tx_body_size 4k;            # 响应体检测上限：4KB（检测敏感信息泄露）

tx_ignore_types
    image/*
    audio/*
    video/*
    application/pdf
    application/zip;

foreach_server {
    location @safeline {
        internal;
        t1k_pass detector_server;
        t1k_connect_timeout 1s;
        t1k_read_timeout 1s;
        t1k_send_timeout 1s;
    }
    
    location @safelinex {
        internal;
        tx_pass detector_server;
        tx_connect_timeout 1s;
        tx_read_timeout 1s;
        tx_send_timeout 1s;
    }
}
```

---

## 响应检测配置详解

响应检测用于检测服务器返回的响应内容，防止敏感信息泄露。

### 启用响应检测

```nginx
# 启用响应检测
tx_intercept @safelinex;
```

### 响应检测参数说明

| 参数 | 说明 | 默认值 | 建议值 |
|------|------|--------|--------|
| `tx_intercept` | 启用响应检测 | 关闭 | 开启后需配置 @safelinex location |
| `tx_body_size` | 响应体转发上限 | 1m | 4k，检测敏感信息足够 |
| `tx_max_detect_size` | 触发检测阈值 | 0 | 500k-1m，小文件不检测节省资源 |
| `tx_ignore_types` | 忽略的 MIME 类型 | 空 | 建议忽略媒体文件 |
| `tx_pass` | 检测节点地址 | - | 同 detector_server |

### 响应检测指令详解

**tx_intercept @location_name**

启用响应检测，将响应转发到指定 location 进行检测。

```nginx
tx_intercept @safelinex;
```

**tx_body_size size**

设置响应体最大转发大小。超过此大小的响应体不会被转发检测。

```nginx
# 最大转发 2MB 响应体
tx_body_size 4k;
```

**tx_max_detect_size size**

设置触发检测的阈值。响应体超过此大小才开始检测，小响应直接返回不检测。

```nginx
# 响应超过 500KB 才检测
tx_max_detect_size 500k;
```

**tx_ignore_types mime-types**

设置忽略检测的 MIME 类型列表。这些类型的响应不进行检测，减少资源消耗。

```nginx
tx_ignore_types
    image/gif
    image/png
    image/jpeg
    video/mp4
    application/pdf;
```

### 响应检测 location 配置

响应检测需要单独的 internal location：

```nginx
foreach_server {
    location @safelinex {
        internal;                   # 仅内部调用
        tx_pass detector_server;    # 转发到检测节点
        tx_connect_timeout 1s;      # 连接超时
        tx_read_timeout 1s;         # 读取超时
        tx_send_timeout 1s;         # 发送超时
    }
}
```

### 响应检测 vs 请求检测

| 特性 | 请求检测 (t1k_*) | 响应检测 (tx_*) |
|------|------------------|-----------------|
| 检测对象 | HTTP 请求 | HTTP 响应 |
| 默认状态 | 需手动开启 | 默认关闭 |
| 主要用途 | 防止攻击 | 防止信息泄露 |
| 性能影响 | 低 | 较高（需检测响应体） |
| 建议场景 | 所有站点 | 敏感接口/API |

### 响应检测使用场景

1. **API 接口保护** - 检测 API 返回的敏感数据
2. **错误信息检测** - 防止错误响应泄露系统信息
3. **数据泄露防护** - 检测响应中的敏感字段

```nginx
# API 响应检测示例
upstream detector_server {
    server 192.168.1.100:8000;
}

# 请求检测
t1k_intercept @safeline;
t1k_body_size 1m;

# 响应检测（检测 API 响应）
tx_intercept @safelinex;
tx_body_size 4k;
tx_max_detect_size 100k;

tx_ignore_types
    image/*
    video/*
    application/pdf;

foreach_server {
    location @safeline {
        internal;
        t1k_pass detector_server;
    }
    
    location @safelinex {
        internal;
        tx_pass detector_server;
        tx_read_timeout 5s;  # 响应可能较大，超时可延长
    }
}
```

### 性能优化建议

1. **合理设置 tx_body_size** - 不要过大，避免占用检测节点资源
2. **使用 tx_max_detect_size** - 小响应不检测，节省资源
3. **配置 tx_ignore_types** - 忽略媒体文件，减少无效检测
4. **调整超时时间** - 响应检测可能较慢，适当增加超时

```nginx
# 性能优化配置
tx_body_size 4k;           # 不要太大
tx_max_detect_size 500k;   # 小响应跳过检测
tx_ignore_types
    image/*
    audio/*
    video/*
    application/*;         # 忽略所有二进制文件

location @safelinex {
    tx_read_timeout 1s;    # 控制检测时间
}
```

---

## 配置说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `t1k_intercept` | 启用请求检测 | 必须配置 |
| `t1k_body_size` | 请求体检测上限 | 1m |
| `t1k_ulog` | 访问日志缓冲区大小 | 0（关闭） |
| `t1k_stat` | 统计信息缓冲区大小 | 0（关闭） |
| `t1k_extra_header` | 记录额外请求头 | off |
| `t1k_extra_body` | 记录额外请求体 | off |
| `tx_intercept` | 启用响应检测 | 关闭 |
| `tx_body_size` | 响应体检测上限 | 1m |
| `tx_max_detect_size` | 响应检测触发阈值 | 0 |
| `tx_ignore_types` | 忽略检测的 MIME 类型 | - |

### 步骤 5：修改 nginx.conf 加载模块

**nginx.conf 配置：**

```nginx
# 文件头部添加（必须在 events 块之前）
load_module modules/ngx_http_t1k_core_module.so;

worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    
    sendfile        on;
    keepalive_timeout  65;
    
    # 服务器配置
    server {
        listen       80;
        server_name  localhost;
        
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
    
    # 加载 t1k 配置（必须在所有 server 配置之后）
    include safeline.conf;
}
```

**重要提示：**
- `load_module` 必须在配置文件最顶部
- `include safeline.conf` 必须在 http 块内、所有 server 之后

### 步骤 6：验证模块加载

**测试配置并重载：**

```bash
# 测试配置
/opt/sbin/nginx -t

# 重载 nginx
/opt/sbin/nginx -s reload
```

**验证 t1k 模块是否被加载到内存：**

```bash
# 查看 nginx master 进程的内存映射
cat /proc/$(ps -ef | grep 'nginx: master' | grep -v 'grep' | awk '{print $2}')/maps | grep t1k
```

**输出示例（模块加载成功）：**

```
7f99a6d54000-7f99a6d58000 r--p 00000000 fd:00 2490401  /opt/modules/ngx_http_t1k_core_module.so
7f99a6d58000-7f99a6d67000 r-xp 00004000 fd:00 2490401  /opt/modules/ngx_http_t1k_core_module.so
7f99a6d67000-7f99a6d6c000 r--p 00013000 fd:00 2490401  /opt/modules/ngx_http_t1k_core_module.so
7f99a6d6c000-7f99a6d6d000 r--p 00017000 fd:00 2490401  /opt/modules/ngx_http_t1k_core_module.so
7f99a6d6d000-7f99a6d6f000 rw-p 00018000 fd:00 2490401  /opt/modules/ngx_http_t1k_core_module.so
```

**内存段说明：**

| 权限 | 段类型 | 说明 |
|------|--------|------|
| `r--p` | ELF头/只读数据 | 只读，不可执行 |
| `r-xp` | 代码段 | 可读可执行 |
| `r--p` | 只读数据段 | 常量数据 |
| `rw-p` | 数据段 | 可读写（全局变量） |

如果输出为空，说明模块未加载，检查：
1. nginx.conf 中 `load_module` 是否正确
2. 模块文件是否存在
3. nginx 是否已重启/重载

**测试检测节点连接：**

```bash
# 测试检测节点端口连通性
nc -zv 192.168.220.5 8000
```

**测试 t1k 模块检测功能：**

```bash
# 正常请求
curl -s http://127.0.0.1:8080/

# 发送 SQL 注入测试流量（应被拦截或记录）
curl -s "http://127.0.0.1:8080/?id=1%20or%201=1"

# 发送 XSS 测试流量
curl -s "http://127.0.0.1:8080/?name=<script>alert(1)</script>"

# 使用 sqlmap User-Agent 测试
curl -s -H "User-Agent: sqlmap/1.0" http://127.0.0.1:8080/

# 查看访问日志（检查是否有 t1k 记录）
tail -20 /opt/logs/access.log

# 查看错误日志
tail -20 /opt/logs/error.log
```

**预期结果：**
- 恶意请求可能被拦截（返回 403 或自定义拦截页面）
- 或请求正常通过但被记录（在日志中可见检测记录）
- 具体行为取决于检测节点的配置和策略

## 简化参数编译

如果无法获取完整的 nginx 源码依赖，可使用简化参数：

```bash
./configure \
  --prefix=/usr/local/openresty/nginx \
  --with-cc-opt=-O2 \
  --with-ld-opt="-Wl,-rpath,/usr/local/openresty/luajit/lib" \
  --with-stream --with-stream_ssl_module \
  --with-stream_ssl_preread_module --with-http_ssl_module \
  --add-dynamic-module=/path/to/t1k_source

make modules
```

## 执行清单

编译时按以下步骤执行：

1. [ ] 连接服务器
2. [ ] 获取系统信息（`uname -r`, `cat /etc/os-release`）
3. [ ] 获取 nginx 编译参数（`nginx -V`）
4. [ ] 查找 nginx 源码目录
5. [ ] 检查编译工具或源码
6. [ ] 设置 LuaJIT 环境变量（OpenResty 需要）
7. [ ] 执行 configure
8. [ ] 执行 make modules
9. [ ] strip 模块
10. [ ] 部署到 modules 目录
11. [ ] 创建 safeline.conf（配置检测节点 IP）
12. [ ] 修改 nginx.conf（添加 load_module 和 include）
13. [ ] 测试配置（`nginx -t`）
14. [ ] 重载 nginx（`nginx -s reload`）
15. [ ] 验证模块加载（`cat /proc/$(...)/maps | grep t1k`）
16. [ ] 测试检测节点连接（`nc -zv IP PORT`）

## 常见问题

### LuaJIT 未找到
```bash
export LUAJIT_LIB=/usr/local/openresty/luajit/lib
export LUAJIT_INC=/usr/local/openresty/luajit/include/luajit-2.1
```

### 编译依赖缺失
```bash
# Ubuntu/Debian
apt install -y build-essential libpcre3-dev libssl-dev zlib1g-dev

# CentOS/RHEL
yum install -y gcc make pcre-devel openssl-devel zlib-devel perl
```

### 模块版本不匹配
确保使用与 `nginx -V` 完全一致的编译参数。
