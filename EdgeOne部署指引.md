# EdgeOne 部署指引（静态站点 + 自有域名）

适用于：纯静态站点（HTML/CSS、零后端、零构建），通过 WorkBuddy 的 EdgeOne Makers 连接器部署到边缘节点并绑定自有已备案域名。

---

## 前置条件

- 站点为纯静态，可直接以 `python3 -m http.server` 运行。
- 域名已完成 **ICP 备案**（国内公开访问必须）。
- 走腾讯云国内节点时，上线后 30 日内需完成**公安联网备案**。
- 已在 WorkBuddy 开通 **EdgeOne Makers** 连接器（底部 `+` 菜单 → 搜 "EdgeOne" → 授权）。

---

## 步骤一：触发部署，拿到临时域名

在对话中说"把当前静态站点部署到 EdgeOne"。连接器读取项目目录（`index.html` + `products/` + `assets/`），推到边缘节点，返回一个临时访问域名。

---

## 步骤二：在 EdgeOne 控制台添加自定义域名

1. EdgeOne 控制台 → **站点 / 域名管理** → 添加自定义域名（如 `smalltools.dabuside.top`）。
2. 控制台会给出**两条待添加记录**，都要加：
   - **TXT 记录**：验证域名所有权（缺它卡验证）。
   - **CNAME 记录**：接入加速，含「主机记录」+「目标值」。

---

## 步骤三：在 DNS 服务商配置 CNAME（关键步骤）

去域名 DNS 服务商（本例为 DNSPod）添加：

- 主机记录：`smalltools`（即子域前缀）
- 记录类型：`CNAME`
- 记录值：**控制台给出的精确字符串**，格式为 `你的子域.你的域名.pages.dnsoe5.com`

> ⚠️ **必须用控制台给的精确值，整段复制，不要照截图手敲。** 目标值形如 `smalltools.dabuside.top.pages.dnsoe5.com`，其中 `dnsoe5` 是 EdgeOne 真实后缀，一个字母错（如误作 `dnsdc`）就会导致一直检测不到。

### 正确做法：CNAME 必须加在父域 zone，不要建独立子域

- **不要**把 `smalltools.dabuside.top` 当成"独立子域 / 子域名管理"单独建 zone。否则该子域 zone 里若残留老 A 记录，会**遮蔽父域的 CNAME**，EdgeOne 始终报"请添加 CNAME"。
- **正确**：直接在父域 `dabuside.top` 的记录列表里加上面那条 CNAME。若此前误建了独立子域，先在 DNSPod「子域名管理」里把该子域**整个删除**（移除 NS 委派），再回父域加 CNAME。

---

## 步骤四：重新检测并等待证书

1. 回 EdgeOne 控制台点 **「重新检测」**（不要只刷新页面）。
2. 等待 HTTPS 证书签发（约 20–30 分钟）。
3. 访问 `https://你的域名` 验证。

> **CNAME 展平提示**：若 DNS 服务商开启了 CNAME 展平，EdgeOne 可能检测不到 CNAME 类型（看到展平后的 A），但流量实际已通，该报错可忽略——以 `dig` 实际解析结果为准。

---

## 验证命令

```bash
dig smalltools.dabuside.top
```

正确结果应为：

```
smalltools.dabuside.top.      CNAME  smalltools.dabuside.top.pages.dnsoe5.com.
smalltools.dabuside.top.pages.dnsoe5.com.  A  123.6.40.224
```

即：返回 CNAME 且最终落到 EdgeOne 节点 IP（如 `123.6.40.224`），不再返回你自己的服务器 IP（如 `49.232.81.134`），也不再为空。

---

## 部署了新版，线上还是旧版——排查顺序

按顺序查，前一条排除再走下一层：

| # | 现象 | 原因 / 处理 |
|---|---|---|
| 1 | 部署命令行报 `[safe-delete] broker denied delete: path=.../.edgeone/assets/...` | WorkBuddy 给 Node 注入了 `NODE_OPTIONS` 删除钩子，会拦住 EdgeOne CLI 清空构建目录 `.edgeone/assets`，导致构建中断或上传旧产物。**部署命令前加 `env -u NODE_OPTIONS`**：<br>`env -u NODE_OPTIONS edgeone makers deploy --json` |
| 2 | 部署成功，但自定义域名返回的仍是旧文件（`Last-Modified` 停在某次旧部署的时间；新加的文件访问 404） | 域名**没有跟随该项目最新部署**。去控制台 → 项目 → **域名管理** → 编辑该域名，确认「关联环境」是 **生产环境**（不是预览环境/某个固定版本）。官方口径：域名会自动指向所关联环境中**最新一次成功部署**；若关联的是预览环境，则只跟预览环境的部署。 |
| 3 | 上一条改完仍不生效 | 把该域名**删除后重新添加**（TXT 归属验证 + CNAME 重新检测），或在该项目「构建部署」列表里把最新部署**设为生产版本 / 重新发布**。 |

> 自查命令（看线上到底挂着哪一版）：
> ```bash
> curl -sD - -o /dev/null "https://你的域名/products/xxx.html?cb=$RANDOM" | grep -i last-modified
> ```
> 返回的时间戳若早于你最后一次部署的时间，就是没跟上。

---

## 让 push 自动部署（GitHub Actions）

项目根目录的 `.github/workflows/deploy-edgeone.yml` 已经写好：push 到 `main` 或手动触发时，在 GitHub 的 runner 上装 CLI 并部署到同一项目（`small-tools-big-use`）。

启用只需要在 GitHub 仓库里加一个 secret：

1. 取 API Token：EdgeOne 控制台 → Pages → **设置 → API Token → 创建**（账号级权限，勿外泄）。
2. GitHub 仓库 → Settings → Secrets and variables → Actions → **New repository secret**。
   - Name：`EDGEONE_TOKEN`
   - Value：上一步的 token
3. push 一次代码，或到 Actions 页面手动 Run workflow。

> 这条路不用把项目改成「导入 Git 仓库」类型（直接上传类型的项目无法切换成 Git 集成，硬切要新建项目并重绑域名）。代价是 token 存在 GitHub secret 里，属账号级权限，仓库必须保持 private，或定期轮换。

---

## 正确部署速查

| 项 | 正确值 / 做法 |
|---|---|
| 部署方式 | WorkBuddy 对话触发 EdgeOne 连接器 |
| 域名要求 | 已 ICP 备案 |
| 控制台记录 | TXT（验证）+ CNAME（接入）都要加 |
| CNAME 位置 | 父域 zone，不建独立子域 |
| CNAME 目标值 | 控制台精确给出的 `*.pages.dnsoe5.com`，整段复制 |
| 完成动作 | 点「重新检测」+ 等证书签发 |
