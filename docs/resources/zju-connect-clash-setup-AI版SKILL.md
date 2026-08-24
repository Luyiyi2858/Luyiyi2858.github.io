---
name: zju-connect-clash-setup
description: 浙江大学校外访问校内资源（CC98、图书馆数据库、教务网等）与科学上网共存的智能分流配置，基于 zju-connect（aTrust 协议）+ Clash Verge Rev。触发词：zju-connect、浙大校网、校内资源、CC98、Clash 分流、RVPN、aTrust、校外访问、校园网 VPN。
agent_created: true
---

# 浙大 zju-connect + Clash Verge 智能分流配置

## 目标
在校外同时实现「访问校内资源」+「科学上网」，两者互不干扰。核心：zju-connect 只当本地 SOCKS5 节点，Clash Verge 当唯一代理入口，用分流规则把校网流量导给 zju-connect、外网流量走机场。

## 原理速记
- zju-connect 是浙大 RVPN 的开源客户端，负责从校外穿透回校内（10.0.0.0/8 内网）。
- Clash Verge 负责系统代理 + 分流。校网域名/IP → zju-connect 节点；其余 → 机场/直连。

## 一、zju-connect 配置

### 下载
- GUI 版：`ZJU Connect for Windows`（内含 `zju-connect.exe` 引擎，可直接命令行调用）。
- 纯命令行：github.com/mythologyli/zju-connect 的 Release。

### config.toml 关键项
```toml
username = "学号"
password = "上网账号密码"          # 含 ! # 等特殊字符时用单引号包裹
protocol = "atrust"               # 必须 atrust（新协议）
graph_code_file = ""              # 留空=弹浏览器点选验证码（推荐）；填路径=存图+手动 JSON（麻烦）
client_data_file = "clientData.json"  # 首次登录后缓存凭证，之后免验证码
server_address = "vpn.zju.edu.cn"
server_port = 443
socks_bind = ":21080"             # 记住这个端口，Clash 节点要用
```

### 启动（Windows PowerShell）
```powershell
cd <zju-connect目录>
.\zju-connect.exe -config <config.toml路径>
```

### 关键点
- 账号密码 = **上网账号**（在 `myvpn.zju.edu.cn` 激活），不是统一身份认证密码、不是校园卡密码。
- 浙大自 2025-05 起 RVPN 已迁移到 aTrust（`vpn.zju.edu.cn`），旧 `rvpn.zju.edu.cn` 大量端口被关，旧教程易失效。
- 首次登录需：图形验证码 + 短信验证码；成功后生成 `clientData.json`，**日常启动免验证码**。仅当凭证过期、更换设备/登录地点、或 `clientData.json` 被删时才需重新验证。
- 验证码若卡在 `Please enter the graph check code JSON`，把 `graph_code_file` 改成空串，重启后弹浏览器点选，比手动算坐标粘 JSON 简单得多。

## 二、Clash Verge Rev 分流配置

> 原版 Clash Verge 已停维护、删了 prepend 功能；务必用 **Clash Verge Rev**（Mihomo 内核）。

### 1. 全局扩展脚本（翻译 prepend-* 键，必须装）
位置：订阅页底部「全局扩展脚本」卡片 → Script。整段替换默认模板：
```javascript
function main(config, profileName) {
    for (const key in config) {
        const ind = key.indexOf('-');
        if (ind !== -1) {
            const action = key.substring(0, ind);
            const target = key.substring(ind + 1);
            const currentValue = config[key];
            if (action === 'prepend') {
                config[target] = [...currentValue, ...(config[target] || [])];
                delete config[key];
            }
        }
    }
    return config;
}
```

### 2. 全局扩展覆写配置
位置：订阅页底部「全局扩展覆写配置」卡片 → Merge。清空后粘贴：
```yaml
prepend-proxies:
  - name: "ZJU校内"
    type: socks5
    server: 127.0.0.1
    port: 21080

prepend-proxy-groups:
  - name: "校内服务"
    type: select
    proxies:
      - ZJU校内
      - DIRECT

prepend-rules:
  - DOMAIN-SUFFIX,vpn.zju.edu.cn,DIRECT   # 防套娃关键，必须放最前
  - DOMAIN-SUFFIX,zju.edu.cn,校内服务
  - DOMAIN-SUFFIX,cc98.org,校内服务
  - IP-CIDR,10.0.0.0/8,校内服务
  - IP-CIDR,210.32.0.0/16,校内服务
```

### 3. 激活与选择
1. 保存脚本 + 覆写配置后，在订阅页**重新点击激活机场 profile**（仅"更新订阅"不够）。
2. 代理页：「校内服务」选 `ZJU校内`，机场分组选一个节点。
3. 代理模式 = **规则**，开启系统代理。

## 三、常见坑
1. **看不到「校内服务」分组**：没装全局扩展脚本，prepend 键没被翻译成真正配置。
2. **验证码反复要输**：改用浏览器方案（`graph_code_file` 留空）。
3. **端口对不上**：config 的 `socks_bind` 端口必须和 Clash 节点 `port` 一致。
4. **套娃/连不上**：必须加 `vpn.zju.edu.cn → DIRECT` 且放最前，否则 TUN 下 zju-connect 连服务器的流量被 Clash 又导回自己。
5. **同账号只能 1 台设备在线**：别处（手机 aTrust/EasyConnect）还挂着会互踢。

## 四、Windows 守护进程（开机自启 + 崩溃重启）
macOS 用 LaunchAgent；Windows 用「任务计划程序」做等价方案：
1. `Win+R` → 输入 `taskschd.msc` → 右侧「创建任务」（不是创建基本任务）。
2. 常规：名称 `zju-connect`；选「只在用户登录时运行」+「使用最高权限运行」。
3. 触发器：新建 →「登录时」。
4. 操作：新建 → 启动程序；程序填 `zju-connect.exe` 绝对路径，参数填 `-config <config.toml 绝对路径>`（务必用绝对路径）；**「起始于（可选）」务必填 zju-connect 目录**——不填则工作目录默认 System32，config 里相对路径的 `clientData.json` 找不到，任务报 `0x8007042B` 启动即退。
5. 条件：取消「只有在使用交流电时才启动此任务」（笔记本电池时也要跑）。
6. 设置：勾选「如果任务失败按以下频率重新启动」（1 分钟 / 999 次，等价 macOS 的 KeepAlive）+「如果请求后任务仍在运行则强行停止」。
7. 验证：右键任务→运行，确认 cc98 可访问后，**关闭手动开的前台窗口**（否则两个进程抢端口，且同一上网账号并发=1 会互踢）。

## 验证
- 校网：`curl -x socks5://127.0.0.1:21080 -I http://cc98.org` 返回 301 即通。
- 浏览器访问 cc98.org（走校内）+ google.com（走机场）都正常即完成。
