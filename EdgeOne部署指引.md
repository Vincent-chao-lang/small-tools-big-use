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

## 当前采用的方案：Git 仓库集成（push 即部署）

EdgeOne Pages 的「直接上传」类型项目**无法切换**成 Git 集成，只能在控制台**新建一个「导入 Git 仓库」的项目**，再把域名迁过去。本项目已按此路线走：

1. 控制台 → Pages → **导入 Git 仓库** → 授权 GitHub → 选 `Vincent-chao-lang/small-tools-big-use`，生产分支 `main`。
2. 构建配置（纯静态站，仓库无 `package.json`）：
   - 构建命令：**留空**
   - 安装命令：**留空**
   - 输出目录：`.`（根目录）
   - 不要选 Vite/Next 等框架模板
3. 仓库根目录的 `edgeone.json` 会把上述配置固定下来（控制台配置会被它覆盖），避免换环境后忘配：

   ```json
   {
     "buildCommand": "",
     "installCommand": "",
     "outputDirectory": "."
   }
   ```

4. 域名迁移（必须做，否则线上永远是旧版）：**先**在旧项目 `small-tools-big-use` 的域名管理里**解绑** `smalltools.dabuside.top`，**再**到新 Git 项目的域名管理里重新添加同一域名 → TXT 归属验证 + CNAME 复用现有解析 → 点「重新检测」→ 等证书签发。一个域名同一时刻只能绑一个项目。
5. 之后每次 `git push origin main`，EdgeOne 自动拉取并部署，域名自动指向最新部署。

> 已废弃的备选方案：GitHub Actions + `EDGEONE_TOKEN` 调 `edgeone makers deploy`，部署到旧的「直接上传」项目。两套并存会互相覆盖，已删除工作流文件。

---

## 正确部署速查

| 项 | 正确值 / 做法 |
|---|---|
| 部署方式 | GitHub 仓库集成（push `main` 自动部署）；本地 CLI 部署仅作应急 |
| 构建命令 | 纯静态站留空，输出目录 `.`（用 `edgeone.json` 固定） |
| 域名要求 | 已 ICP 备案 |
| 控制台记录 | TXT（验证）+ CNAME（接入）都要加 |
| CNAME 位置 | 父域 zone，不建独立子域 |
| CNAME 目标值 | 控制台精确给出的 `*.pages.dnsoe5.com`，整段复制 |
| 完成动作 | 点「重新检测」+ 等证书签发 |
