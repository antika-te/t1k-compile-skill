# T1K 模块编译 Skill

Claude Code Skill - 自动化编译 OpenResty/Nginx 的 T1K 动态模块（雷池WAF引流/流量复制模块）。

## 功能

- 自动连接远程服务器编译 T1K 模块
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

在 Claude Code 中输入：

```
帮我编译 t1k 模块
服务器：192.168.1.100/root/password
编译工具：/usr/local/src/t1k_cli_amd64
工具密码：/usr/local/src/t1k_cli_amd64.passwd
检测节点：192.168.1.5:8000
```

或提供源码路径：

```
帮我编译 t1k 模块
服务器：192.168.1.100/root/password
源码：/usr/local/src/t1k_source
检测节点：192.168.1.5:8000
```

## 编译流程

1. 连接服务器并获取系统信息
2. 获取 nginx 编译参数（`nginx -V`）
3. 查找 nginx 源码目录
4. 编译 t1k 动态模块
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

- 服务器需有 nginx/opresty 源码
- 或有 t1k_cli 编译工具 + 密码
- 编译依赖：gcc, make, pcre-devel, openssl-devel, zlib-devel

## 相关链接

- [雷池 WAF](https://github.com/chaitin/safeline) - 长亭科技开源 WAF
- [Claude Code](https://github.com/anthropics/claude-code) - Anthropic CLI 工具

## License

MIT

## 贡献

欢迎提交 Issue 和 Pull Request！