# iOS 26 使用个人 Cloudflare Worker + Shadowrocket 配置 Apple WLOC 网络定位

使用没问题，修改了apple定位来进行applewatch高血压，睡觉呼吸暂停开通

> 本文记录如何在 iOS 26 上部署个人 Cloudflare Worker，并通过 Shadowrocket 修改 Apple WLOC 网络定位返回结果。
>
> 本方案不需要越狱，但需要启用 HTTPS 解密并在 iPhone 中信任 Shadowrocket 生成的 CA 证书。请先完整阅读“安全说明”和“恢复真实定位”章节。

## 目录

- [实现原理](#实现原理)
- [适用范围与限制](#适用范围与限制)
- [准备条件](#准备条件)
- [第一步：部署个人 Cloudflare Worker](#第一步部署个人-cloudflare-worker)
- [第二步：安装 Shadowrocket WLOC 模块](#第二步安装-shadowrocket-wloc-模块)
- [第三步：开启 HTTPS 解密并安装 CA 证书](#第三步开启-https-解密并安装-ca-证书)
- [第四步：直接在个人 Worker 页面设置位置](#第四步直接在个人-worker-页面设置位置)
- [第五步：让新位置在 iOS 26 生效](#第五步让新位置在-ios-26-生效)
- [验证是否生效](#验证是否生效)
- [切换位置](#切换位置)
- [恢复真实定位](#恢复真实定位)
- [可选：修改快捷指令使用个人 Worker](#可选修改快捷指令使用个人-worker)
- [常见问题](#常见问题)
- [安全说明](#安全说明)
- [致谢](#致谢)

## 实现原理

整套链路由三部分组成：

1. **个人 Cloudflare Worker**
   - 提供地图选点页面；
   - 解析 Apple Maps、高德等地图链接；
   - 不负责直接修改 iPhone 定位；
   - 不负责生成 CA 证书。

2. **Shadowrocket WLOC 模块**
   - 拦截 `gs-loc.apple.com` 与 `gs-loc-cn.apple.com` 的指定请求；
   - 把选中的坐标保存到 Shadowrocket 本地持久化存储；
   - 在 Apple WLOC 网络定位响应中替换经纬度。

3. **Shadowrocket HTTPS 解密**
   - 让 Shadowrocket 能读取并修改上述 Apple WLOC HTTPS 响应；
   - CA 证书由 Shadowrocket 在本机生成；
   - Worker 页面不会提供 CA 证书。

数据流如下：

```text
个人 Worker 页面选择目标位置
        ↓
请求 gs-loc.apple.com/wloc-settings/save
        ↓
Shadowrocket 模块在本机拦截请求
        ↓
坐标写入 Shadowrocket 本地存储
        ↓
Apple 发起 WLOC 网络定位请求
        ↓
Shadowrocket 修改 WLOC 响应中的坐标
```

## 适用范围与限制

- 不需要越狱。
- 主要修改 Apple 的 Wi-Fi/基站网络定位结果。
- 不会修改 iPhone 的 GPS 硬件。
- 室内、地下空间或 GPS 信号较弱的环境通常更容易观察到效果。
- 室外 GPS 信号较强时，iOS 或某些 App 可能继续采用真实 GPS 位置。
- “查找”、运动记录、导航、考勤、游戏及第三方 App 不保证有效。
- iOS 26 的定位缓存较强，切换坐标后通常需要按本文顺序重启设备。
- 不要把本方案用于欺诈、考勤作弊、规避地区限制或违反服务条款的用途。

## 准备条件

- 一台运行 iOS 26 的 iPhone；
- 官方 Shadowrocket；
- 一个 Cloudflare 账户；
- 能让 Shadowrocket 正常建立 VPN 隧道的配置；
- Safari；
- 如采用命令行部署，还需要 Git、Node.js 和 npm。

上游项目：

- [Yu9191/wloc](https://github.com/Yu9191/wloc)

## 第一步：部署个人 Cloudflare Worker

### 方法一：Cloudflare 一键部署

打开：

[Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/?url=https://github.com/Yu9191/wloc/tree/main/worker)

然后：

1. 登录 Cloudflare；
2. 授权 Cloudflare 读取部署所需的公开仓库内容；
3. 根据页面提示创建 Worker；
4. 等待部署完成；
5. 记录 Cloudflare 分配的 Worker 域名。

下文统一使用：

```text
https://<你的 Worker 域名>/
```

例如，Cloudflare 实际生成的地址通常类似：

```text
https://wloc-spoofer.example.workers.dev/
```

### 方法二：命令行部署

```bash
git clone https://github.com/Yu9191/wloc.git
cd wloc/worker
npm install
npx wrangler login
npm run deploy
```

部署成功后，终端会显示 Worker 地址。

### 验证 Worker

先在浏览器打开：

```text
https://<你的 Worker 域名>/
```

如果能看到 WLOC 地图选点页面，说明主页部署成功。

还可以访问：

```text
https://<你的 Worker 域名>/api/parse?format=json&u=22.544577%2C113.941140
```

正常情况下会返回包含 `lat` 和 `lon` 的 JSON。

> 部署 Worker 只完成了选点页面和地图链接解析。此时 iPhone 定位还不会发生变化。

## 第二步：安装 Shadowrocket WLOC 模块

在 iPhone 上打开 Shadowrocket：

1. 进入底部“配置”；
2. 进入“模块”；
3. 点击右上角 `+`；
4. 粘贴下面的模块地址；
5. 下载并启用模块。

```text
https://raw.githubusercontent.com/Yu9191/wloc/refs/heads/main/modules/wloc.module
```

模块名称通常显示为：

```text
Apple WLOC 定位修改
```

当前模块会添加两类脚本规则：

- 拦截 `/clls/wloc` 响应并替换坐标；
- 拦截 `/wloc-settings/save` 请求并保存坐标。

它还会把下面两个域名加入 MITM 范围：

```text
gs-loc.apple.com
gs-loc-cn.apple.com
```

模块原文可在这里核对：

- [wloc.module](https://raw.githubusercontent.com/Yu9191/wloc/refs/heads/main/modules/wloc.module)

## 第三步：开启 HTTPS 解密并安装 CA 证书

这一步必须完成。只导入模块但不启用 HTTPS 解密，WLOC 响应无法被修改。

### 3.1 在 Shadowrocket 生成证书

1. 打开 Shadowrocket；
2. 进入“配置”；
3. 找到当前使用的配置文件，例如 `default.conf`；
4. 点击配置文件并选择“使用配置”或“编译配置”；
5. 再次点击配置文件；
6. 选择“编辑配置”；
7. 进入“HTTPS 解密”；
8. 开启 HTTPS 解密；
9. 进入“证书”；
10. 选择“生成新的 CA 证书”；
11. 点击“安装证书”。

不同 Shadowrocket 版本的文字可能略有区别，也可以从配置文件右侧的 `ⓘ` 进入“HTTPS 解密”。

> 必须使用自己手机里的 Shadowrocket 生成证书。不要从网页、网盘或其他人那里下载 CA 证书。

### 3.2 在 iOS 安装描述文件

打开 iPhone“设置”：

1. 进入“通用”；
2. 进入“VPN 与设备管理”；
3. 找到刚下载的 Shadowrocket 描述文件；
4. 点击安装；
5. 输入锁屏密码；
6. 再次确认安装。

部分 iOS 版本会直接在设置首页显示“已下载描述文件”，也可以从这里进入。

### 3.3 开启 CA 完全信任

证书安装后还没有自动获得 SSL/TLS 完全信任，需要继续操作：

1. 打开“设置”；
2. 进入“通用”；
3. 进入“关于本机”；
4. 滑动到页面底部；
5. 进入“证书信任设置”；
6. 找到 Shadowrocket CA；
7. 开启“完全信任”；
8. 确认系统警告。

Apple 对手动安装根证书的说明：

- [Trust manually installed certificate profiles](https://support.apple.com/102390)

### 3.4 返回 Shadowrocket 检查

确认以下条件全部满足：

- 当前配置的 HTTPS 解密已开启；
- WLOC 模块已启用；
- MITM 域名包含：

```text
gs-loc.apple.com
gs-loc-cn.apple.com
```

- 没有使用 `hostname=*` 之类的全域名解密配置；
- Shadowrocket 可以正常连接；
- iPhone 控制中心能够看到 VPN 状态。

## 第四步：直接在个人 Worker 页面设置位置

这条路线不需要安装快捷指令。

### 4.1 打开 Shadowrocket

1. 确认 WLOC 模块已开启；
2. 确认当前配置已启用 HTTPS 解密；
3. 把“全局路由”设为“配置”；
4. 启动 Shadowrocket；
5. 确认 VPN 已连接。

### 4.2 在 Safari 打开个人 Worker

```text
https://<你的 Worker 域名>/
```

建议把页面“添加到主屏幕”，后续可像普通 App 一样打开。

### 4.3 选择并保存位置

在页面中：

1. 点击地图选择目标位置，或者搜索地点；
2. 确认页面显示的经纬度；
3. 点击“储存到设备”；
4. 等待页面提示“已储存”；
5. 查看“当前生效坐标”是否更新。

“储存到设备”的实际含义是：

- 页面向 `gs-loc.apple.com/wloc-settings/save` 发起请求；
- Shadowrocket 在 iPhone 本机截获该请求；
- `wloc-settings.js` 把坐标写入 Shadowrocket 本地持久化存储；
- 坐标并不是储存在 Cloudflare 数据库中。

如果页面出现“模块未生效”或“储存失败”，请先查看“常见问题”。

## 第五步：让新位置在 iOS 26 生效

根据上游项目的实测，iOS 26 会长时间缓存 `locationd` 已获取的定位结果。仅切换飞行模式或关闭定位服务，可能不足以清理缓存。

### 推荐流程：有蜂窝网络

1. 完全关闭 Apple 地图及其他正在使用定位的 App；
2. 在个人 Worker 页面选择位置并点击“储存到设备”；
3. 开启飞行模式；
4. 进入“设置 → 隐私与安全性 → 定位服务”；
5. 关闭定位服务；
6. 重启 iPhone；
7. 开机后暂时不要打开地图，也不要先开启定位服务；
8. 关闭飞行模式，暂时保持 Wi-Fi 关闭；
9. 使用蜂窝网络打开 Shadowrocket；
10. 启动 Shadowrocket 并确认 VPN 已连接；
11. 再开启定位服务；
12. 最后打开 Apple 地图验证。

关键点是：

```text
先让 Shadowrocket 和 WLOC 模块就绪，再让系统重新获取定位。
```

### 没有蜂窝网络时

1. 保存目标位置；
2. 关闭定位服务；
3. 重启 iPhone；
4. 开机后保持定位服务关闭；
5. 连接 Wi-Fi；
6. 立即启动 Shadowrocket 并确认 VPN 已连接；
7. 再开启定位服务；
8. 打开 Apple 地图验证。

这种方式可能比蜂窝网络流程更容易受到 Wi-Fi 定位缓存影响，但重点仍然是不要在 Shadowrocket 就绪前开启地图或定位服务。

### 备用流程

如果不想立即重启，可以尝试：

1. 关闭定位服务；
2. 打开个人 Worker 页面并储存位置；
3. 重新开启定位服务；
4. 如果系统询问定位权限，选择“下次询问或在我共享时”；
5. 打开 Apple 地图验证。

如果无效，仍建议执行完整重启流程。

## 验证是否生效

建议先使用 Apple 地图测试：

1. 在 Worker 中选择距离真实位置较远、容易识别的地点；
2. 按 iOS 26 推荐顺序重启；
3. 打开 Apple 地图；
4. 点击定位按钮；
5. 观察蓝色定位点是否移动到目标位置。

还可以查看 Shadowrocket 日志。模块成功处理响应时，通常能看到与 WLOC 修改相关的日志。

注意：

- 页面显示“已储存”只表示坐标写入 Shadowrocket 本地；
- 不代表 iOS 已立即抛弃旧的定位缓存；
- Apple 地图成功不代表所有第三方 App 都会使用相同位置；
- 室外 GPS 信号较强时，结果可能被真实 GPS 覆盖。

## 切换位置

后续切换位置不需要重新安装模块和证书：

1. 启动 Shadowrocket；
2. 确认 VPN、HTTPS 解密和 WLOC 模块正常；
3. 打开个人 Worker 页面；
4. 选择新位置；
5. 点击“储存到设备”；
6. 按 iOS 26 流程清理定位缓存；
7. 打开 Apple 地图验证。

## 恢复真实定位

### 方法一：清除坐标并关闭模块

1. 启动 Shadowrocket；
2. 打开个人 Worker 页面；
3. 在“当前生效坐标”区域点击“清除数据”；
4. 回到 Shadowrocket；
5. 关闭或删除 WLOC 模块；
6. 关闭定位服务；
7. 重启 iPhone；
8. 开机后恢复正常网络；
9. 重新开启定位服务。

关闭模块是最可靠的恢复方式。

如果你曾手动修改模块参数中的默认经纬度，仅清除持久化数据可能仍会继续使用模块参数，因此应直接关闭或删除模块。

### 方法二：完全移除 HTTPS 解密证书

如果以后不再使用任何需要 Shadowrocket HTTPS 解密的模块：

1. 在 Shadowrocket 当前配置中关闭 HTTPS 解密；
2. 打开“设置 → 通用 → VPN 与设备管理”；
3. 找到 Shadowrocket CA 描述文件；
4. 点击“移除描述文件”；
5. 返回“设置 → 通用 → 关于本机 → 证书信任设置”；
6. 确认对应 CA 已不存在；
7. 重启 iPhone。

> 如果其他 Shadowrocket 模块也依赖 HTTPS 解密，删除证书会使这些模块同时失效。

## 可选：修改快捷指令使用个人 Worker

如果主要通过个人 Worker 网页选点，可以跳过本节。

上游提供的“WLOC 设置位置”快捷指令默认调用公共 Worker。安装快捷指令后，可以手动改成自己的域名：

1. 打开“快捷指令”；
2. 找到 WLOC 设置位置快捷指令；
3. 点击 `…` 编辑；
4. 找到负责解析地图链接的“获取 URL 内容”动作；
5. 找到类似下面的地址：

```text
https://wloc-spoofer.wloc.workers.dev/api/parse?format=json&u=...
```

6. 把域名替换为：

```text
https://<你的 Worker 域名>/api/parse?format=json&u=...
```

7. 保留后面的快捷指令变量；
8. 保存并测试。

“恢复定位”快捷指令只调用本机拦截接口，一般不需要修改 Worker 地址。

## 常见问题

### Worker 页面可以打开，但“储存到设备”失败

依次检查：

1. Shadowrocket 是否已经连接；
2. 控制中心是否显示 VPN；
3. WLOC 模块是否启用；
4. 当前配置是否启用 HTTPS 解密；
5. Shadowrocket CA 是否已经安装；
6. CA 是否已经开启完全信任；
7. Safari 是否通过 Shadowrocket；
8. 当前配置是否已经“使用配置”或重新编译。

### 已经安装模块，但地图位置没有变化

可能原因：

- iOS 26 仍在使用旧定位缓存；
- 重启后先打开了地图，后启动 Shadowrocket；
- GPS 信号较强，真实 GPS 覆盖了网络定位结果；
- 模块没有成功处理新的 WLOC 请求；
- 保存的坐标没有写入本地存储；
- 当前使用的不是启用了 HTTPS 解密的配置文件。

建议重新执行：

```text
储存坐标
→ 关闭定位服务
→ 重启
→ 先启动 Shadowrocket
→ 再开启定位服务
→ 最后打开地图
```

### 找不到“HTTPS 解密”

1. 进入 Shadowrocket 的“配置”页面；
2. 找到 `default.conf` 或当前配置文件；
3. 点击配置文件；
4. 选择“编辑配置”；
5. 向下查找“HTTPS 解密”。

如果当前是 Clash YAML 或某些远程配置，UI 编辑项目可能不完整。可以先使用 Shadowrocket 内置的 `default.conf`。

### 已安装证书，但“证书信任设置”里没有

先确认描述文件是否真正安装完成：

```text
设置
→ 通用
→ VPN 与设备管理
```

安装描述文件后，再进入：

```text
设置
→ 通用
→ 关于本机
→ 证书信任设置
```

### 部署了个人 Worker，但快捷指令仍访问公共 Worker

部署 Worker 不会自动修改已经安装的快捷指令。需要手动编辑快捷指令里的 `/api/parse` 地址。

### 某些 App 仍显示真实位置

这是可能出现的正常结果。本项目修改的是 Apple WLOC 网络定位，不是 GPS 硬件，也不是所有 App 的统一定位接口。

### 开启 HTTPS 解密后部分网站或 App 无法访问

检查是否存在过宽的 MITM 域名配置。WLOC 模块只需要：

```text
gs-loc.apple.com
gs-loc-cn.apple.com
```

不要为了本项目配置：

```text
hostname=*
```

银行、支付和部分高安全性 App 还可能使用证书绑定，不接受中间人证书。

## 安全说明

### 1. CA 证书具有较高权限

完全信任 Shadowrocket CA，意味着 Shadowrocket 可以对配置范围内的 HTTPS 流量进行解密。请确保：

- 证书由自己的 Shadowrocket 本地生成；
- 不安装别人提供的 CA；
- MITM 域名严格限制为需要的 Apple WLOC 域名；
- 不使用通配符解密全部 HTTPS 流量；
- 不使用时关闭模块和 HTTPS 解密。

### 2. 自建 Worker 不等于全部链路都已自托管

个人 Worker 解决的是：

- 选点页面；
- 地图链接解析；
- 避免把地图分享链接交给作者的公共 Worker。

但默认模块脚本仍然从 GitHub `main` 分支加载，选点页面也会使用第三方地图、地图瓦片、搜索和前端资源。

如果需要更高的供应链控制，应：

1. Fork 上游仓库；
2. 审查 `dist/wloc.js` 和 `dist/wloc-settings.js`；
3. 把模块中的 `script-path` 改为自己的 Fork；
4. 最好固定到明确的提交版本，而不是长期跟随 `main`；
5. 在更新前重新检查代码差异。

### 3. 不建议长期无意识开启

启用模块后，Shadowrocket 会持续处理指定 Apple WLOC 请求。建议只在测试期间使用，并记住如何恢复：

```text
清除坐标
→ 关闭模块
→ 关闭 HTTPS 解密
→ 按需要删除 CA
→ 重启设备
```

### 4. 不要用于安全关键场景

不要在以下场景依赖被修改后的定位：

- 紧急求助；
- 驾车或步行导航；
- 家庭成员安全定位；
- 运动和健康记录；
- 需要证明人员实际位置的业务。

## 致谢

本流程基于以下开源项目整理并完成 iOS 26 + Shadowrocket + 个人 Cloudflare Worker 实机配置：

- [Yu9191/wloc](https://github.com/Yu9191/wloc)
- [proxypin-wloc-spoofer](https://github.com/FFF686868/proxypin-wloc-spoofer)
- [NSNanoCat/Util](https://github.com/NSNanoCat/Util)

请保留上游项目署名。本文仅为部署与使用记录，不代表 Apple、Cloudflare 或 Shadowrocket 官方支持该方案。
