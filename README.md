# T1K 模块编译 Skill

Claude Code Skill - 自动化编译 OpenResty/Nginx 的 T1K 动态模块（雷池WAF引流/流量复制模块）。

## 功能

- **参数驱动**：用户直接提供生产环境参数，无需登录生产服务器
- 自动连接编译服务器验证环境一致性
- 支持两种编译方式：源码编译 / t1k_cli 工具编译
- 自动配置 safeline.conf 检测节点
- 自动验证模块加载状态

## 安装

### 方法一：手动安装

```bash
# 创建 skills 目录
mkdir -p ~/.claude/skills

# 下载 skill 文件
curl -o ~/.claude/skills/t1k-compile.md https://raw.githubusercontent.com/antika-te/t1k-compile-skill/main/t1k-compile.md

# 或手动复制
cp t1k-compile.md ~/.claude/skills/
```

### 方法二：克隆仓库

```bash
git clone https://github.com/antika-te/t1k-compile-skill.git
cp t1k-compile-skill/t1k-compile.md ~/.claude/skills/
```

## 使用

在 Claude Code 中提供以下信息即可触发 skill：

**使用方式：提供生产环境参数，AI 自动连接编译服务器完成编译**

```
帮我编译 t1k 模块

生产环境信息：
  内核版本：3.10.0-1160.el7.x86_64
  系统版本：CentOS Linux 7 (Core)
  nginx -V：nginx version: openresty/1.21.4.1
            configure arguments: --prefix=/usr/local/openresty/nginx --with-cc-opt=-O2 ...

nginx 安装路径：/usr/local/openresty

编译服务器：192.168.1.100 / root / password
编译方式：t1k_cli 工具
  工具路径：/usr/local/src/t1k_cli_amd64
  密码文件：/usr/local/src/t1k_cli_amd64.passwd

检测节点：192.168.1.5:8000
```

或使用源码方式编译：

```
帮我编译 t1k 模块

生产环境信息：
  内核版本：3.10.0-1160.el7.x86_64
  系统版本：CentOS Linux 7 (Core)
  nginx -V：nginx version: openresty/1.21.4.1
            configure arguments: --prefix=/usr/local/openresty/nginx --with-cc-opt=-O2 ...

编译服务器：192.168.1.100 / root / password
源码路径：/usr/local/src/t1k_source

检测节点：192.168.1.5:8000
```

## 编译流程

1. 用户提供生产环境参数（`uname -r`、`/etc/os-release`、`nginx -V`）
2. AI 连接编译服务器，验证内核和系统版本与生产环境一致
3. 从 `nginx -V` 的 `configure arguments` 中提取编译参数
4. 编译 t1k 动态模块（源码方式或 t1k_cli 工具方式）
5. 部署模块到 nginx
6. 配置 safeline.conf
7. 验证模块加载

## 验证模块加载

```bash
# 查看模块是否加载到内存
cat /proc/$(ps -ef | grep 'nginx: master' | grep -v 'grep' | awk '{print $2}')/maps | grep t1k

# 测试攻击拦截
curl -s -w "\nHTTP_CODE: %{http_code}\n" "http://127.0.0.1/?id=1%20or%201=1"
```

## 支持的 Nginx 版本

- OpenResty 1.19.x - 1.31.x
- Nginx 1.18.x - 1.31.x

## 前置条件

- 编译服务器需有 nginx/openresty 源码（与生产环境版本一致）
- 或有 t1k_cli 编译工具 + 密码
- 编译依赖：gcc, make, pcre-devel, openssl-devel, zlib-devel
- 编译服务器的内核版本和系统版本需与生产环境一致（编译 `.so` 要求 ABI 兼容）

## 相关链接

- [雷池 WAF](https://github.com/chaitin/safeline) - 长亭科技开源 WAF
- [Claude Code](https://github.com/anthropics/claude-code) - Anthropic CLI 工具

## License

MIT

## 贡献

欢迎提交 Issue 和 Pull Request！