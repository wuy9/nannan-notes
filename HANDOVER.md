# 南南的沉淀库 · 交接文档

> 给接手的人或 AI：读完这一份就能改东西。最后更新 2026-11-16。

---

## 1. 一句话

一个**单文件**的个人笔记站，放在 GitHub Pages 上；数据不存服务器，而是每次保存自动提交到另一个**私有** GitHub 仓库里。

**三个地址**

| 用途 | 地址 |
|---|---|
| 站点（用户看的） | https://wuy9.github.io/nannan-notes/ |
| 源码仓库（公开） | https://github.com/wuy9/nannan-notes |
| 数据仓库（**私有**） | https://github.com/wuy9/nannan-notes-data |

---

## 2. 现状快照

| 项 | 值 |
|---|---|
| 技术栈 | 纯 HTML + CSS + 原生 JS（ES5 风格 + 局部 async/await），**零依赖、零构建、零 CDN** |
| 文件 | `index.html`（全部界面与逻辑，约 1440 行）、`README.md` |
| 部署 | GitHub Pages，`main` 分支 `/ (root)`，推完 1–3 分钟自动重建 |
| 数据量 | 5 条笔记，JSON 约 2 KB |
| 兼容性 | 需要支持 `color-mix()`、`dialog`、`AbortController`、`TextEncoder` 的浏览器（Chrome/Edge/Safari 16+） |

本地文件位置（写这份文档时）：
`C:\Users\Augenhsterns\WorkBuddy\2026-11-16-21-43-07\nannan-notes\`
其中 `index.v1.bak.html` 是第一版备份，可删。

---

## 3. 代码地图（`index.html`，行号仅供定位）

| 行 | 区块 | 负责什么 |
|---|---|---|
| 8–350 | `:root` / CSS | 亮暗两套 CSS 变量、卡片、富文本、调色板、弹窗、移动端媒体查询 |
| 355 | `<body>` | 头部（品牌/搜索/按钮）、两行筛选、列表、页脚、三个 `dialog` |
| 375 | 常量 | `KEY`/`PREF`/`TKEY` 三个 localStorage 键、8 色调色板 `PAL` 与亮暗两套色值 |
| 387 | 小工具 | `uid` `now` `fmt` `esc` `textToHtml` `textOf` |
| 410 | **富文本清洗** | `cssColor` `PALMAP` `styleParts` `wrap` `ser` `clean` —— 所有正文 HTML 都过这里 |
| 492 | 存储 | `loadEntries` `save` `migrate` `seed` |
| 565 | 状态 | `q` `activeTags` `activeGroup` `editingId` `editingColor` |
| 591 | 汇总 | `allTags` `allGroups` `colorOfTag` `filtered` |
| 631 | 筛选栏 | `renderGroups` `renderTags` |
| 700 | 列表 | `groupRows` `cardFor` `afterAppend` `render` |
| 843 | 搜索 | 输入防抖 90ms |
| 855 | 浮动调色板 | `openPal` `closePal`（改标签颜色用） |
| 902 | 编辑器 | `openEditor` `commit`、富文本工具条事件 |
| 1057 | 标签颜色面板 | `buildTagCfg` `openTagCfg` |
| **1120** | **云端同步** | `ghFetch` `cloudPull` `cloudPush` `checkRepo` `applyRemote` `setCloud` `scheduleSync` `runSync` `connectCloud` `disconnectCloud` `bootCloud` |
| 1382 | 导入/导出 | 与云端同格式的 JSON 打包 |

改之前先搜注释里那行 `/* ---------------- 区块名 ---------------- */`，一跳就到。

---

## 4. 数据模型

localStorage 键：

| 键 | 内容 |
|---|---|
| `nn.entries.v1` | 笔记数组（**既是缓存也是离线数据**） |
| `nn.tagcolors.v1` | `{ 标签名: 色号 }` |
| `nn.pref.v1` | `{ theme: "light"\|"dark", k: 0\|1\|2 }` 字号三档 |
| `nn.cloud.token` | 访问令牌（**绝不写进代码或仓库**） |

云端文件 `notes.json`（等于导出备份的格式）：

```json
{
  "v": 2,
  "updatedAt": "2026-11-16T14:52:10.988Z",
  "tagColors": { "搜索": "amber" },
  "entries": [
    {
      "id": "m3x9k2ab",
      "title": "标题",
      "body": "第一行<br>第二行 <b>加粗</b> 和 <span data-c=\"jade\">松绿的字</span>",
      "rich": true,
      "tags": ["读书"],
      "group": "读书笔记",
      "color": "clay",
      "created": "2026-11-16T14:00:00.000Z",
      "updated": "2026-11-16T14:52:00.000Z"
    }
  ]
}
```

**字段说明**

- `body` 是 HTML 字符串，不是纯文本；纯文本用 `textOf()` 取（搜索就是搜这个）。
- `rich: true` 表示已迁移成富文本；老数据没有这个字段时，`migrate()` 会把纯文本转义 + 换行转 `<br>` 再补上标记。
- `color` / `tagColors[标签]` 存的是**色号名**（`clay` `amber` `olive` `jade` `indigo` `plum` `rose` `stone`），不是 `#hex`。色值在 CSS 的 `[data-c="…"]` 里，亮暗各一套。
- `group` 空字符串表示未分组。

---

## 5. 同步机制（最容易改错的地方）

```
启动：有令牌 → GET contents/notes.json
        ├─ 有文件 → 直接用云端覆盖本地（然后 render）
        └─ 无文件 → 把本地 PUT 上去
      无令牌 → 纯本地模式，页脚显示「本地保存」

保存：save() 或 saveTagColors()
        → 写 localStorage
        → scheduleSync()：700ms 防抖
        → PUT 整份文档（带上次的 sha）
        → 409/412/422：重新 GET 拿新 sha，再 PUT 一次
        → 401/403/404：页脚变红「保存失败」，数据仍在本地
```

**三条不可动摇的约定**

1. **云端为准，不做 id 合并。** 如果启动时把本地和云端按 id 合并，在 A 设备删掉的笔记会被 B 设备的本地缓存“复活”，再推回云端。所以有令牌时一律用云端覆盖本地。
2. **连接时问方向。** `connectCloud()` 里如果云端和本地都有内容，弹 `confirm` 让用户选「云端覆盖本地」还是「本地覆盖云端」。
3. **令牌只在验证通过后才落盘。** 顺序是 `checkRepo()` → 成功 → 写 `nn.cloud.token`。这样粘错东西不会留下坏凭证。

请求全部走 `ghFetch()`（20 秒超时 + 中文错误提示），**不要**直接 `fetch`，否则网络卡住会一直转圈。

API：`https://api.github.com/repos/wuy9/nannan-notes-data/contents/notes.json`，鉴权 `Authorization: Bearer <token>`，内容 base64（`b64enc`/`b64dec` 按 UTF-8 处理，别用 `btoa(str)` 直接转，中文会炸）。

调试开关：`localStorage["nn.api.base"]` 可以覆盖 API 根地址，用来指向本地假服务器（见第 8 节）。

---

## 6. 令牌

- 存在**用户浏览器的 localStorage** 里（`nn.cloud.token`），不进代码、不进仓库、其他访客看不到。
- 当前是一把 **classic token**，名字 `nannan-notes-fixed`，scope 含 `repo`，无过期时间。
- 换令牌：站点页脚左下角「已存云端」→ 粘贴新令牌 → 「立即同步」。
- 撤销：https://github.com/settings/tokens
- 安全性：`repo` scope 理论上覆盖该账号**所有**私有仓库（当时账号下只有 `nannan-notes-data` 一个私有仓库，所以实际范围就这一个）。如果以后账号里有了别的私有仓库，建议换成 fine-grained token，只勾 `nannan-notes-data` + `Contents: Read and write`。

---

## 7. 改东西时必须知道的 7 个坑

1. **正文颜色存 `data-c` 不存 `#hex`。** `clean()` 里的 `PALMAP` 会把调色板颜色反查成色号。这样深色模式切换时文字颜色会自动变亮。存 hex 就会在深色下看不清。
2. **富文本用 `document.execCommand`。** 工具条按钮必须 `mousedown` 时 `preventDefault()`，否则点击会抢走选区，`bold`/`foreColor` 直接失效。
3. **`clean()` 是白名单过滤。** 只放行 `b/i/u/s/br/span` 和 4 个 style 属性；粘贴进来的外来样式会被剥掉。要支持新标签（比如 `<mark>`）得往 `ser()` 里加分支，否则会被拍平成纯文本。
4. **浮动调色板必须 append 进 `dialog`。** `dialog` 在 top layer，`document.body` 下的绝对定位元素会被它盖住。`openPal()` 里已经处理：`anchor.closest("dialog") || document.body`。
5. **折叠判断靠实测高度。** `afterAppend()` 在 DOM 插入后比较 `scrollHeight > clientHeight`，不要再改回按字符数估算。
6. **移动端工具栏用 `nowrap + overflow-x`。** 390px 下 B/I/U + 8 个色点 + 清除格式刚好吃满，`wrap` 会折成两行很难看。
7. **改数据结构必须同步改 `migrate()`。** 它兼着老数据升级的职责，加字段时给默认值，别让老条目读出来是 `undefined`。

---

## 8. 本地开发与验收

```bash
cd nannan-notes
python -m http.server 8848        # 必须后台跑，别用 & 挂起
# 打开 http://127.0.0.1:8848/
```

**每次改完至少过一遍这个清单：**

- [ ] 新增一条 → 卡片出现在最前，正文换行正常
- [ ] 选中文字 → 点 B 变粗、点色点变色；「清除格式」能还原
- [ ] 卡片颜色 → 左色条 + 淡底；切深色后文字仍看得清
- [ ] 标签栏 ⚙ → 改某个标签颜色，筛选 chip 上出现对应色点
- [ ] 分组 → 列表出现组标题 + 计数；顶部「分组」能筛
- [ ] 搜索中文命中正文（不是命中 HTML 标签）
- [ ] 删除后刷新，**不会复活**（前提：连着云端）
- [ ] 清掉 `nn.entries.v1` 再刷新 → 从云端完整恢复
- [ ] 390px 宽：头部不挤、工具栏单行、弹窗能滚
- [ ] Console 无报错

**测云端而不动真数据**：把 `localStorage["nn.api.base"]` 指到一个本地假 GitHub API（GET 404/200、PUT 校验 sha 并返回新 sha、鉴权头校验），然后填任何以 `ghp_` 开头的假令牌走完整流程。测完记得 `localStorage.removeItem("nn.api.base")`。

`nn.cloud.token` 一旦被清空，站点会退回纯本地模式——这是正常的，不是坏了。

---

## 9. 部署

**方式一（网页，无需本地环境）**

1. 打开 https://github.com/wuy9/nannan-notes/upload
2. 拖入 `index.html`（和/或 `README.md`）
3. 填一句提交说明 → Commit changes
4. **偶发 HTTP 400**：这是 GitHub 上传页的 manifest 任务失败，不是文件问题，原样重试即可（一般第 2–4 次就过）
5. 等 1–3 分钟，Pages 自动重建

**方式二（git，更稳）**

```bash
cd nannan-notes
git init && git add index.html README.md
git commit -m "改了什么"
git remote add origin https://<用户名>:<令牌>@github.com/wuy9/nannan-notes.git
git push -u origin main
```

Pages 设置：Settings → Pages → Source: `Deploy from a branch` → `main` / `/ (root)`。**站点仓库必须公开**，免费账号才能用 Pages。

---

## 10. 故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 页脚红点「保存失败」 | 令牌过期/被撤销/网络断 | 点开面板看红字；换令牌或等网络恢复（会自动重试一次） |
| 「令牌无效、已过期或被撤销」 | 令牌值不对 | 重新生成，注意是 `ghp_` 开头那串，不是网页地址 |
| 「看不到私有仓库」 | 用的是 fine-grained token 但没授权该仓库 | 把 `nannan-notes-data` 加进 Only select repositories，给 Contents 读写 |
| 打开是 404 | Pages 没开或还在构建 | 看 Settings → Pages；或等 1–3 分钟再刷新 |
| 换设备后是空的 | 没填令牌 | 页脚「本地保存」→ 粘令牌 → 提示「云端已有 N 条」时选**确定** |
| 保存后另一台设备没变 | 那台设备没联网/令牌失效 | 刷新页面，启动会重新拉 |
| 正文颜色在深色下看不清 | 直接把 `#hex` 写进了 style | 改成 `data-c="色号"` |
| 上传一直 400 | GitHub 上传页抖动 | 重试；或改用 git push |

---

## 11. 交接红线

- **不要把令牌写进 `index.html` 或任何仓库文件。** 它只该待在用户浏览器的 localStorage 里。
- **`nannan-notes-data` 必须保持私有。** 一旦改成公开，所有笔记立刻对全网可见。
- **`nannan-notes` 必须保持公开**（否则免费账号的 Pages 会失效）。
- **动数据结构前先让用户导出一次备份**（页脚「导出备份」），出问题能导回去。
- 用户不懂技术细节，沟通时先给结论、再说怎么办，别甩术语。

---

## 12. 想换掉 GitHub 当后端？

只需要改两个函数：`cloudPull()` 和 `cloudPush()`，其余逻辑（防抖、状态显示、启动拉取）都不用动。文档格式 `{v, updatedAt, tagColors, entries}` 保持不变，换成 Supabase / 自建 API 都是同一个套路。
