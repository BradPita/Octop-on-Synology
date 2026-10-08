# Octop on Synology

> **在群晖（Synology）NAS 上用 Docker 部署腾讯 [Octop](https://github.com/TencentCloud/Octop) 的分步安装指南** —— 容器管理器 Compose / SSH CLI 两条路线，照着做即可上线（首次拉取镜像的耗时视网络而定）。

[Octop](https://octop.cloud) 是腾讯云开源的自托管 AI 助手平台（MIT License）：多用户、多 Agent「专家团队」、长期记忆、知识库 RAG、Browser / Terminal AI+、远程桌面、IM 通道（飞书 / 钉钉 / QQ / 微信 / Telegram …）、自然语言定时任务——一个进程同时提供 Web 控制台、CLI 与 IM 机器人入口。

本仓库只回答一个问题：**怎么把它装到你的群晖上，并且真正用起来。**

**版本基线**：本指南基于官方镜像 `ghcr.io/tencentcloud/octop:1.0.2b6`（2026-10 发布，amd64 / arm64 多架构）撰写；命令中的标签可整体替换为 `latest` 或其他已有标签。

## 内容覆盖

- 前置检查清单：DSM 7.2+、容器管理器、x86_64 / ARM64 架构确认
- 官方镜像 `ghcr.io/tencentcloud/octop` 拉取说明（解决「群晖 Registry 里搜不到」的问题）
- 两套落地路线：**容器管理器 → 项目（Compose）** 纯图形界面；**SSH + docker run** 命令行
- 数据持久化（容器内 `/data/.octop` ← 宿主机数据目录）、端口映射与冲突处理
- 首次启动密码机制：`OCTOP_DEFAULT_PASSWORD` 自定 / `credential.txt` 随机
- Web 控制台初始化：配 LLM 供应商 → 建专家 → 接 IM 通道 → 开多用户
- 运维：日常启停查日志、版本升级、备份还原、彻底卸载
- 10 条常见问题 FAQ：拉不到镜像、exec format error、端口被占、改环境变量密码不生效……

## 适用条件

| 项 | 要求 |
| --- | --- |
| 系统 | Synology DSM 7.2 及以上（自带 Container Manager） |
| CPU | x86_64 或 ARM64 均可（官方镜像为多架构） |
| 内存 | 建议空闲 ≥ 4 GB（同机跑本地大模型另按其需要预留） |
| 磁盘 | 为 Docker 数据目录预留 ≥ 4 GB |
| 网络 | 可解析并访问 `ghcr.io`（拉取镜像） |

> 无需预装任何软件。接云端模型需联网；接本机 Ollama 可完全离线，数据不出硬盘。

## 怎么读

- **就想最快装好**：按 §0 → §3 → §4 → §5 → §7 → §8 的顺序走（定参数 → 建数据目录 → 粘贴 Compose → 创建项目并启动 → 拿初始密码登录 → 配模型），其余章节遇到问题再翻。
- **头一回在群晖上用 Docker**：先把 §1 的前置条件逐条过完，主操作看 §5 的图形界面路线，卡在哪一步就去 §10 对照。
- **出问题了**：直接跳 §10 FAQ；日志里报什么错就搜什么关键字。

## 目录

- 0. 你需要先确定的 5 个值（照抄或修改）
- 1. 前置条件核对清单
- 2. 步骤一：规划路径与端口
- 3. 步骤二：创建数据目录
- 4. 步骤三：准备 Compose 文件（核心）
- 5. 步骤四：创建并启动容器（图形界面 / SSH CLI）
- 6. 步骤五：确认容器状态与查看日志
- 7. 步骤六：首次登录与拿到初始密码
- 8. 步骤七：Web 控制台初次配置（让它真正能用）
- 9. 运维速查：日常 / 升级 / 备份 / 卸载
- 10. 常见问题（FAQ）
- 11. 参考链接（官方）

---

## 0. 你需要先确定的 5 个值（照抄或修改）

在下述命令中，把这 5 个值替换成你自己的：

| 变量 | 含义 | 示例 | 备注 |
| --- | --- | --- | --- |
| `NAS_IP` | 群晖的局域网 IP | `192.168.1.100` | 在「控制面板 → 网络」查看 |
| `DATA_DIR` | 容器数据挂载到宿主机的目录 | `/volume1/docker/octop/data` | 务必是**绝对路径**；建议放在 `/volume1` |
| `HOST_PORT` | 宿主机对外端口 | `8088` | 若被占用改成 `8348` 等；容器内固定 8088 |
| `IMAGE_TAG` | 拉取的镜像标签 | `1.0.2b6` | 推荐锁定版本；也可写 `latest` |
| `ADMIN_PASSWORD` | 首次启动管理员密码 | `Octop@2026x` | ≥8 位、含字母和数字；留空则随机生成 |

> 后面出现的 `<NAS_IP>`、`<DATA_DIR>` 等都按这张表替换。

---

## 1. 前置条件核对清单

逐项打勾，缺的先补：

- [ ] **DSM 版本 ≥ 7.2**（推荐）。控制面板 → 系统更新可查看；低于 7.2 请先升级 DSM。
- [ ] **已安装「容器管理器」（Container Manager）**。在「套件中心」搜 `Container Manager` 安装；它是 7.2 起的原生 Docker 管理界面。
- [ ] **确认 CPU 架构**（决定拉哪种架构的镜像）：
    ```bash
    # 通过 SSH 登录群晖后执行
    uname -m
    # 输出 x86_64  → 用 amd64 镜像
    # 输出 aarch64/arm64 → 用 arm64 镜像
    ```
    > `1.0.2b6` 已同时发布 `amd64` / `arm64`；若用多架构 manifest 会自动选对。拿不准就显式带架构标签，见第 3 步。
- [ ] **磁盘空间充足**：至少预留 **4 GB 以上**给 `/volume1/docker`（镜像 + 数据卷 + 日志）。存储管理器中查看剩余容量。
- [ ] **内存充足**：建议空闲内存 ≥ **4 GB**（Octop 服务本身数 GB，若要跑本地大模型另计显存/内存）。
- [ ] **网络可访问外网**：能解析并访问 `ghcr.io`（拉取镜像）。可在「控制面板 → 网络 → 常规」确认 DNS。
- [ ] （可选）**开启 SSH**：控制面板 → 终端机和 SNMP → SSH → 「启用 SSH 协议」。便于命令行操作与排障。

---

## 2. 步骤一：规划路径与端口

按第 0 节的表定好 `DATA_DIR`、`HOST_PORT`、`IMAGE_TAG`、`ADMIN_PASSWORD`。要点：

- **`DATA_DIR` 必须是绝对路径**，如 `/volume1/docker/octop/data`。它将被映射进容器的 `/data/.octop`——数据库、密钥、工作区、日志全在这里，**升级/重建容器不会丢数据的前提就是挂好它**。
- **`HOST_PORT` 别和别人撞**。8088 是 Octop 默认；若群晖上已有别的服务占 8088，把**左边**换成别的（如 `8348`），右边保持 `8088` 不变：`- "8348:8088"`。
- **`ADMIN_PASSWORD` 只在“首次启动”生效**（即还没有数据库时）。之后想改密码，去 Web 控制台「设置」里改，改这里的环境变量不会再影响已建好的管理员。

---

## 3. 步骤二：创建数据目录

让宿主机上先有那个数据目录（否则容器首次写入时由 Docker 自动创建，权限未必理想）。

**方式 A：图形界面**
1. 打开「文件Station / 资源管理器」。
2. 进入 `/volume1/docker/`（没有 `docker` 就先新建一个文件夹叫 `docker`）。
3. 在 `docker` 里新建文件夹 `octop`，再在 `octop` 里新建文件夹 `data`。最终得到 `/volume1/docker/octop/data`。

**方式 B：SSH 命令行（更快）**
```bash
ssh youruser@<NAS_IP>     # 先启用 SSH；youruser 是你的管理员账号
sudo mkdir -p /volume1/docker/octop/data
sudo ls -ld /volume1/docker/octop/data   # 确认存在
```

> 如果将来想在别的卷上存放数据（比如 `/volume2/...`），就把 `DATA_DIR` 换成对应卷路径，其余不变。

---

## 4. 步骤三：准备 Compose 文件（核心）

群晖的「容器管理器 → 项目（Project）」支持直接粘贴 Compose。下面这份是为 NAS 场景精简过的（用官方 ghcr 镜像，而非本地 build），可直接用。

> **两个版本**：推荐 **「方案 A：锁定版本 1.0.2b6」**（最稳、可复现）。若想省事跟最新，看「方案 B」。

### 方案 A（推荐）：Compose 文件全文

```yaml
# 文件可保存为 octop-compose.yml（粘贴到群晖「项目」里时无需保存文件）
services:
  octop:
    # 官方镜像（GitHub Container Registry）。锁定当前最新稳定标签。
    image: ghcr.io/tencentcloud/octop:1.0.2b6
    container_name: octop
    restart: unless-stopped
    ports:
      # 左 = 宿主机(群晖)端口，右 = 容器内固定 8088。8088 被占就把左改成 8348 之类。
      - "8088:8088"
    volumes:
      # 关键持久化：容器内 /data/.octop ←→ 宿主机 DATA_DIR。绝对路径！
      - "/volume1/docker/octop/data:/data/.octop"
    environment:
      - HOME=/data
      - OCTOP_BIND_HOST=0.0.0.0
      - OCTOP_PORT=8088
      - OCTOP_LOG_LEVEL=info
      # ---- 首次启动引导（仅在尚无数据库时读取）----
      - OCTOP_ADMIN_USERNAME=admin
      # 首次启动管理员密码。≥8 位且含字母+数字。
      # 强烈建议先设好；留空则随机生成并写入 credential.txt。
      - OCTOP_DEFAULT_PASSWORD=Octop@2026x
```

**逐字段说明：**

| 字段 | 值 | 说明 |
| --- | --- | --- |
| `image` | `ghcr.io/tencentcloud/octop:1.0.2b6` | 官方镜像；锁定版本 |
| `container_name` | `octop` | 固定名字，方便 `docker logs octop` |
| `restart` | `unless-stopped` | 崩溃/重启自动拉起 |
| `ports` | `"8088:8088"` | 对外端口；冲突改左边 |
| `volumes` | `DATA_DIR:/data/.octop` | **唯一最关键的一行**，数据落地 |
| `HOME` | `/data` | 让应用把家目录指向数据区 |
| `OCTOP_BIND_HOST` | `0.0.0.0` | 监听全部网卡，外部可访问 |
| `OCTOP_PORT` | `8088` | 容器内监听端口 |
| `OCTOP_LOG_LEVEL` | `info` | 日志级别 |
| `OCTOP_ADMIN_USERNAME` | `admin` | 首启管理员用户名 |
| `OCTOP_DEFAULT_PASSWORD` | 你自定的强密码 | 首启管理员密码；可选 |

### 方案 B：想用 `latest` 或指定其他版本

把 `image` 一行改成你想要的：

```yaml
    image: ghcr.io/tencentcloud/octop:latest          # 总是追最新
    # 或显式带架构（当你的 daemon 不认多架构 manifest 时）：
    # x86_64:  image: ghcr.io/tencentcloud/octop:1.0.2b6-amd64
    # ARM64:  image: ghcr.io/tencentcloud/octop:1.0.2b6-arm64
```

> 可用标签（截至 v1.0.2b6）：`latest`、`1.0.2b6`、`1.0.2b6-amd64`、`1.0.2b6-arm64`，以及更早的 `1.0.1`、`1.0.0`、`0.9.x`。需要固定某个旧版时从列表里挑。

---

## 5. 步骤四：创建并启动容器

### 方式 A（最推荐）：容器管理器 → 项目

1. 打开「**容器管理器**」→ 左侧「**项目**」（Project）。
2. 点「**创建**」→ 选「**Compose 文件**」。
3. 选择名称（如 `octop`）、描述随意。
4. 在 Compose 编辑框里**清空默认内容**，粘贴上面**方案 A 的完整 YAML**（记得替换 `OCTOP_DEFAULT_PASSWORD` 和你的数据路径）。
5. 点「创建」。然后回到「项目」列表，点该项目里的「**启动 / Start**」。

启动后：
- 会先**拉取镜像** `ghcr.io/tencentcloud/octop:1.0.2b6`（首次较慢，视网速几分钟到十几分钟）。
- 拉完后自动创建并运行容器。

> 小提示：如果「项目」功能在你的 DSM 版本不可用，直接用下面的方式 B（SSH + docker CLI）。

### 方式 B：SSH + docker CLI（等效，适合熟练者）

```bash
ssh youruser@<NAS_IP>
sudo -i    # 切 root 操作（或用 sudo 前缀）

# 1) 建数据目录
mkdir -p /volume1/docker/octop/data

# 2) 拉官方镜像（锁定 1.0.2b6；也可 latest / 指定 amd64|arm64）
docker pull ghcr.io/tencentcloud/octop:1.0.2b6

# 3) 运行容器（一次性写好，复制即可）
docker run -d \
  --restart unless-stopped \
  --name octop \
  -p 8088:8088 \
  -v /volume1/docker/octop/data:/data/.octop \
  -e HOME=/data \
  -e OCTOP_BIND_HOST=0.0.0.0 \
  -e OCTOP_PORT=8088 \
  -e OCTOP_LOG_LEVEL=info \
  -e OCTOP_ADMIN_USERNAME=admin \
  -e OCTOP_DEFAULT_PASSWORD=Octop@2026x \
  ghcr.io/tencentcloud/octop:1.0.2b6
```

> 如果 8088 被占，把 `-p 8088:8088` 改成 `-p 8348:8088`（只动左边）。`-e OCTOP_DEFAULT_PASSWORD` 这一行可整行删掉＝走随机密码流程（见第 7 步）。

---

## 6. 步骤五：确认容器状态与查看日志

```bash
# 状态应为 Up（不是 Exited/Restarting 反复跳）
docker ps

# 实时看日志（Ctrl+C 退出观察）
docker logs -f --tail 100 octop
```

在群晖图形界面下：容器管理器 → 「项目」里点该条目能看到容器与日志页签。

**正常启动的迹象**：日志出现服务监听 `0.0.0.0:8088` / FastAPI 就绪字样；若报 `address already in use` 就是端口被占，回去改映射端口。若反复 Restart，多半是镜像架构不匹配或挂载路径不存在，查日志定位。

---

## 7. 步骤六：首次登录与拿到初始密码

- 浏览器打开：**`http://<NAS_IP>:8088`**（本机用 `http://localhost:8088`）。
- 若你在 Compose 里填了 `OCTOP_DEFAULT_PASSWORD`：用户名 `admin`，密码就是你填的那个。
- 若你**没填**（留空 / 没写这行环境量）：系统随机生成一个密码，并写到数据区的 `credential.txt`。取出来：

```bash
# 二选一
docker exec octop cat /data/.octop/credential.txt
# 或直接在群晖上看宿主文件：
cat /volume1/docker/octop/data/credential.txt
```

- **密码策略**：≥8 位、必须同时含字母和数字；太弱（常见密码）会被拒绝并退回随机。

> 记牢第一套账号密码。后续可在 Web 控制台改密。管理员初始用户名为 `admin`（可用 `OCTOP_ADMIN_USERNAME` 覆盖）。

---

## 8. 步骤七：Web 控制台初次配置（让它真正能用）

登录后按顺序做这三件，Octop 才算“活起来”：

1. **配置 LLM 供应商（必做）**
   - 进入「专家 / 设置 → 模型与供应商（Models / Providers）」。
   - 二选一：
     - 云 API：OpenAI 兼容接口 / 通义 DashScope 等，填入你的 API Key（也可提前用环境变量 `OPENAI_API_KEY`、`DASHSCOPE_API_KEY` 注入）。
     - 本地模型：接 Ollama（在群晖上另跑 Ollama 容器），填 `http://<NAS_IP>:11434`。
   - 选中一个可用模型并保存。
2. **创建第一个专家（Agent）**
   - 「专家」→「新建专家」，起个名字、选人格模板/系统提示词，绑定刚配的模型供应商，保存。
   - 之后可继续创建更多专家、调整人设、开启技能与工具。
3. **（可选）接入 IM 通道 & 开多用户**
   - 「频道 / 通道（Channels）」接入飞书、钉钉、QQ、微信、企业微信、Telegram 等，扫码/填凭据即可在手机端对话。
   - 需要家人/同事共用：「设置 → 用户」创建多个普通用户，各自拥有独立的专家与工作区。
   - 其他可选能力：知识库 RAG（导入私有文档）、定时任务（自然语言配 Cron）、插件、远程桌面、ACP 对接编码 Agent。按需开，不必一次全开。

---

## 9. 运维速查：日常 / 升级 / 备份 / 卸载

### 日常
```bash
docker ps                          # 看是否 Up
docker logs -f --tail 50 octop   # 看实时日志
docker stop octop                  # 停
docker start octop                 # 启
docker restart octop               # 重启
```
- 群晖「容器管理器 → 项目」里也有对应的 停止/启动/重启 按钮，不想敲命令就用界面。

### 升级（换新版，数据不丢）
1. （建议）先备份数据：`tar czf /volume1/octop-backup-$(date +%F).tgz -C /volume1/docker octop`
2. 拉新镜像：`docker pull ghcr.io/tencentcloud/octop:<新版本>`
3. 用**新镜像**重建容器：
   - 方式 A（项目）：把 Compose 里 `image:` 的标签改成新版本 → 重新「启动」项目。
   - 方式 B（CLI）：
     ```bash
     docker stop octop && docker rm octop
     docker run -d --restart unless-stopped --name octop \
       -p 8088:8088 \
       -v /volume1/docker/octop/data:/data/.octop \
       -e HOME=/data -e OCTOP_BIND_HOST=0.0.0.0 -e OCTOP_PORT=8088 \
       ghcr.io/tencentcloud/octop:<新版本>
     ```
4. 重启后用第 7 步的方式验证。数据库结构会在下次启动时自动迁移。跨大版本升级前务必先备份。

### 备份 / 还原
```bash
# 停容器后备份整个数据区（SQLite WAL 一致性更好）
docker stop octop
tar czf /volume1/octop-backup-$(date +%F).tgz -C /volume1/docker octop
docker start octop
# 还原：把 tar 解回 /volume1/docker/，再启容器
```
> 也可以用应用自带的 `octop backup` 命令导出（在数据区内执行）。

### 卸载
```bash
docker stop octop && docker rm octop
docker rmi ghcr.io/tencentcloud/octop:1.0.2b6   # 删镜像省空间
rm -rf /volume1/docker/octop                # 彻底删数据（先备份！）
```
- 图形界面：容器管理器 →「项目」删除对应项目；再回「镜像」删 octop 镜像；最后资源管理器删数据目录。

---

## 10. 常见问题（FAQ）

1. **为什么在群晖的“注册表/Registry”搜索里找不到 Octop？**
   因为官方镜像托管在 **GitHub Container Registry（`ghcr.io`）**，而不在 Docker Hub 或群晖内置源里。所以要么用「项目（Compose）」方式，要么用 docker CLI 拉 `ghcr.io/tencentcloud/octop`。

2. **拉不到镜像 / 报错 connection refused / unknown registry？**
   检查网络能否到 `ghcr.io`（DNS + 出口）。国内偶尔慢，重试几次或配置镜像加速。也确认拼写正确。

3. **报 “exec format error” 或跑不起来，怀疑架构不对？**
   用 `uname -m` 确认群晖是 `x86_64` 还是 `arm64`。选对标签：`...amd64` 给 Intel，`...arm64` 给 ARM。拿不准就用多架构的 `latest`/版本号让其自选。

4. **8088 被占 / 打不开？**
   把映射改成 `8348:8088` 这类空闲端口，再用新端口访问。容器内始终是 8088。

5. **我忘了当初设的初始密码在哪？**
   若首启时没填 `OCTOP_DEFAULT_PASSWORD`，系统生成随机密码并写入 `credential.txt`。取法见第 7 步（`docker exec ... cat ...` 或直接读群晖上的数据文件）。

6. **以后改了 `OCTOP_DEFAULT_PASSWORD` 环境变量，怎么密码没变？**
   这个变量只在**首次启动**（还没有数据库时）读取一次。之后管理员密码由 Web 控制台管理，去「设置」里改，别再指望环境变量。

7. **浏览器里登进去一片白 / 转圈？**
   多半是前端资源没下全或版本不一致。确认容器日志无错、镜像标签一致；必要时升级到最新标签。仍不行可看 FAQ 4 排查端口/防火墙。

8. **升级或迁移前最稳的做法？**
   停容器 → 整个备份数据目录（tar 打包，见第 9 节“备份/还原”）→ 再动镜像或数据区。恢复时逆序解开。跨大版本一定先备份。

---

## 11. 参考链接（官方）

| 项 | 地址 |
| --- | --- |
| 官网 | <https://octop.cloud> |
| GitHub 仓库 | <https://github.com/TencentCloud/Octop> |
| 中文 README | <https://github.com/TencentCloud/Octop/blob/main/README_CN.md> |
| Docker Compose 模板 | <https://github.com/TencentCloud/Octop/blob/main/docker/docker-compose.yml> |
| 环境变量样例 `.env.example` | <https://github.com/TencentCloud/Octop/blob/main/.env.example> |
| Releases（下载镜像/安装包） | <https://github.com/TencentCloud/Octop/releases> |
| 官方容器镜像仓库 | `ghcr.io/tencentcloud/octop` |

---

## 附：一页速查（贴墙上）

```
拉镜像 :  docker pull ghcr.io/tencentcloud/octop:1.0.2b6
建目录 :  sudo mkdir -p /volume1/docker/octop/data
起容器 :  docker run -d --restart unless-stopped --name octop \
              -p 8088:8088 -v /volume1/docker/octop/data:/data/.octop \
              -e HOME=/data -e OCTOP_BIND_HOST=0.0.0.0 -e OCTOP_PORT=8088 \
              -e OCTOP_ADMIN_USERNAME=admin -e OCTOP_DEFAULT_PASSWORD='Octop@2026x' \
              ghcr.io/tencentcloud/octop:1.0.2b6
看状态 :  docker ps
看日志 :  docker logs -f --tail 100 octop
取密码 :  docker exec octop cat /data/.octop/credential.txt
访问    :  http://<NAS_IP>:8088   (admin / 上面密码)
升级    :  pull 新标签 → 重建容器（数据在 volume 里）
备份    :  停容器 → tar 整个数据目录
```

---

_本指引基于官方仓库 v1.0.2b6 时期的 Docker Compose、`.env.example` 与 README 整理；若后续官方变量名/端口有变，以仓库内 `docker/` 与 `.env.example` 为准。_
