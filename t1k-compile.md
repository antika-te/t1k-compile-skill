---
name: t1k-compile
description: 编译 OpenResty/Nginx 的 T1K 动态模块（雷池WAF引流/流量复制模块）
---

# T1K 模块编译工具

编译 OpenResty/Nginx 的 T1K 动态模块（雷池WAF引流/流量复制模块）。

## 交互规则

### 1. 编译出错处理

编译过程如遇错误，必须将完整错误信息展示给用户，不允许静默跳过。

```
编译出错，错误信息如下：
[显示完整的 gcc 错误输出]
```

### 2. 错误诊断与修复

如果错误可以修复（如兼容性问题），按以下流程：

1. **分析错误根因** - 定位具体报错文件、行号、错误类型
2. **给出修复方案** - 说明需要修改哪些文件、如何修改
3. **询问用户确认** - 等待用户同意后再执行修复

示例：
```
错误分析：
  ngx_http_t1k_anticookie_tampered.c:198: error: 'headers_in.cookies' 在 nginx 1.31.1 中已改为 'cookie' (指针类型)

修复方案：
  修改 ngx_http_t1k_anticookie_tampered.c 第 198/199/222 行
  修改 ngx_http_t1k_bot_req_deconfuse.c 第 45/46/69 行
  将 cookies 数组遍历改为 cookie 指针访问

是否执行修复？
```

### 3. 不可修复错误

如果错误无法自动修复（如缺少编译依赖、不支持的版本），直接告知用户原因和建议。

### 4. 打印日志

关键步骤必须打印输出，让用户了解进度：
- configure 结果
- 编译进度
- 模块文件路径和大小
- 部署验证结果

### 5. 环境签名验证

编译完成后，打印 nginx 签名用于验证环境一致性。

```bash
# 提取 nginx 环境签名（33位环境指纹）
egrep -ao ".,.,.,[01]{33}" /usr/local/openresty/nginx/sbin/nginx
```

**签名格式：**

```
8,4,8,000011111101011100111011111100011
主.次.补丁  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^--- 33位环境指纹
```

**验证方法：**

```bash
# 在编译环境执行
nginx_sig=$(egrep -ao ".,.,.,[01]{33}" /usr/local/openresty/nginx/sbin/nginx)

# 在目标环境执行相同命令，对比结果
```

- 签名一致：编译环境与目标环境匹配，模块可直接使用
- 签名不一致：需要调整编译环境（操作系统/内核/gcc 版本）

**影响环境指纹的因素：**
- 操作系统版本（ubuntu/centos 等）
- 内核版本
- gcc 编译器版本
- nginx 编译参数

---

## 使用方法

用户提供以下信息，AI 自动完成编译。

**不需要登录客户的生产服务器**——用户直接提供生产环境参数即可。
编译服务器需用户提供 SSH 连接信息，AI 远程执行编译、部署和验证。

**必需信息（参数提供）：**
1. 生产环境参数（从客户的 nginx 服务器获取，直接提供值即可）：
   - `uname -r` — 内核版本
   - `/etc/os-release` — 系统版本
   - `nginx -V` — nginx 完整编译参数
2. nginx 安装路径（如 `/usr/local/openresty`）
3. 编译服务器 SSH 连接信息：`IP/用户名/密码` 或 SSH 免密地址
4. 编译资源（二选一）：
   - **源码方式**：编译服务器上 t1k 源码目录路径
   - **工具方式**：编译服务器上 t1k_cli 编译工具路径 + 密码文件路径
5. 检测节点地址

**可选信息：**
- 编译服务器上 nginx 源码路径（默认自动查找）

## 编译流程

### 步骤 1：确认生产环境参数

> **关键要求：** 以下参数必须来自客户正在运行 nginx 的**生产服务器**，由用户直接提供。
> 编译服务器是根据这些参数**复制**出来的相同环境（内核版本、系统版本、nginx 版本一致），
> 否则编译出的 `.so` 模块会因 ABI 不兼容导致 nginx 无法加载：
> - `uname -r`（内核版本）
> - `/etc/os-release`（系统版本）
> - `nginx -V`（编译参数）

用户提供以下生产环境参数，无需登录生产服务器：

```bash
# 用户提供 — 从生产服务器获取的这些值
UNAME_R="3.10.0-1160.el7.x86_64"
OS_RELEASE="CentOS Linux 7 (Core)"
NGINX_V_OUTPUT='nginx version: openresty/1.21.4.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.1.1k  25 Mar 2021
TLS SNI support enabled
configure arguments: --prefix=/usr/local/openresty/nginx --with-cc-opt=-O2 ...'
```

### 步骤 2：连接编译服务器，验证环境一致性并执行编译

> **重要：** 编译服务器必须与步骤 1 的生产环境参数（内核、系统版本、nginx 版本）一致。
> 连接编译服务器后，先验证 `uname -r` 和 `/etc/os-release` 与步骤 1 提供的参数匹配，
> 再使用步骤 1 的 `nginx -V` 中的 `configure arguments` 进行编译。

**连接编译服务器：**

```bash
# 连接编译服务器（如有密码则使用 sshpass）
sshpass -p '密码' ssh root@编译服务器IP "命令"

# 或免密登录
ssh root@编译服务器IP "命令"

# 验证编译服务器内核版本与生产环境一致
uname -r
# 验证系统版本与生产环境一致
cat /etc/os-release
# 确认 nginx 源码目录存在（路径由用户提供或自动查找）
find /usr/local/src -name "nginx-*.tar.gz" -o -name "openresty-*" -type d 2>/dev/null
```

#### 方式 A：使用源码目录编译（推荐）

**适用场景：** 已有 t1k 源码目录

```bash
T1K_SOURCE=/usr/local/src/t1k_source
NGINX_SRC=/usr/local/src/openresty-1.31.1.1/build/nginx-1.31.1

cd $NGINX_SRC

# 设置 LuaJIT 环境变量
export LUAJIT_LIB=/usr/local/openresty/luajit/lib
export LUAJIT_INC=/usr/local/openresty/luajit/include/luajit-2.1

# 配置 — 以下参数为示例，必须从「步骤 1」的 nginx -V 输出中提取真实参数替换
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

# 编译 — configure 参数必须从「步骤 1」的 nginx -V 输出中提取真实参数替换
# 系统内核和版本信息必须在步骤 1 确认与编译环境一致
./t1k_cli_amd64 -p "$T1K_PASS" -- "./configure \
  --prefix=/usr/local/openresty/nginx \
  --with-cc-opt=-O2 \
  --with-ld-opt=\"-Wl,-rpath,/usr/local/openresty/luajit/lib\" \
  --with-stream --with-stream_ssl_module \
  --with-stream_ssl_preread_module --with-http_ssl_module \
  --add-dynamic-module=\${t1k_dir} \
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
# tx_body_size 1m;

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

**配置说明：**

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

> **注意：** `--prefix`、`--with-cc-opt`、`--with-ld-opt` 等参数仍需从目标服务器 `nginx -V` 输出中提取，
> 不能随意填写。

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

1. [ ] 确认用户已提供生产环境参数（`uname -r`、`/etc/os-release`、`nginx -V`）
2. [ ] 连接编译服务器
3. [ ] 验证编译服务器内核版本（`uname -r`）与步骤 1 的生产参数一致
4. [ ] 验证编译服务器系统版本（`cat /etc/os-release`）与步骤 1 的生产参数一致
5. [ ] 查找编译服务器上的 nginx 源码目录
6. [ ] 检查编译工具或源码
7. [ ] 设置 LuaJIT 环境变量（OpenResty 需要）
8. [ ] 执行 configure（参数从步骤 1 的 `nginx -V` 输出中提取）
9. [ ] 执行 make modules
10. [ ] strip 模块
11. [ ] 部署到 modules 目录
12. [ ] 创建 safeline.conf（配置检测节点 IP）
13. [ ] 修改 nginx.conf（添加 load_module 和 include）
14. [ ] 测试配置（`nginx -t`）
15. [ ] 重载 nginx（`nginx -s reload`）
16. [ ] 验证模块加载（`cat /proc/$(...)/maps | grep t1k`）
17. [ ] 测试检测节点连接（`nc -zv IP PORT`）

## 常见问题

### 编译环境与运行环境不匹配（模块无法加载）
编译 `.so` 时必须确保：
1. `nginx -V` 输出的 configure 参数与编译时使用的参数一致
2. 编译服务器的内核版本（`uname -r`）与目标服务器一致
3. 系统版本（`/etc/os-release`）一致，尤其是 glibc 版本
不一致会导致 nginx 加载模块时报 `undefined symbol` 或 `version GLIBC_X.XX not found` 等错误。

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
