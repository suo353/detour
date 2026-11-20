# 绕路 Detour · 城市目的地探索智能体

> 2026 飞猪 AI 旅行创新大赛参赛作品（赛题方向 8：目的地探索）
> 在线地图页：**https://suo353.github.io/detour/map.html**

用户只说一句「我在杭州武林广场，今晚有 3 小时，想走夜游」，它做的第一件事不是推荐，是**把榜单上的热门点全删掉**，然后给一条**同一条街区内、全程步行**的可执行路线。

---

## 为什么做这个

平台 POI 库里大量小馆、老铺、书场、街区**在库里、有坐标、有简介、有跳转链接**，但没进榜单，几乎不可能被看到。
杭州 10 个文化向类目实测：**87 个可去地点里 52 个没有榜单名次（59.8%）**。

常规行程助手「拉景点列表 → 排时间」，候选池就是榜单，所以永远只有那几个地方。绕路反过来：
**`listRank` 非空的 POI 默认不进路线**——上榜本身就是「今晚会挤」的同义词。

## 四个主题，同一座城市

| 主题 | 3 小时（武林广场出发）首条路线 |
|---|---|
| 夜游 | 武林夜市 → 浙江省文化馆小剧场 → ET PARK 酒吧（西湖店） |
| 建筑 | 富义仓艺术中心 → 香积寺广场 → 妙等觉 → 大兜路历史文化街区 |
| 非遗 | 中山公园-中山纪念亭 → 御座房 → 西泠印社 → 岳飞纪念馆 |
| 动漫 | 南山路一带手办 / 谷子 / 潮玩小店 |

## 三条硬承诺

1. **不拼线**：簇里点不够就不跨簇凑数，宁可按两站给；
2. **不编价**：门票/酒店拿不到就写「无数据，出发前确认」；
3. **来源可追溯**：每条答案带一行数据来源，例如「周边吃/看来源：随包/本地缓存 ×26；高德实时 ×4」。

---

## 仓库结构

```
detour-agent/            # 源码（本仓库根目录的 detour-agent-src.zip 解压后即为它）
├── explore.py           # 引擎：检索 / 打分 / 800m 聚类 / 剔榜 / 真实路网（单文件，零第三方依赖）
├── skill-package-engine/# 可直接打包成百炼技能包的目录（scripts/ + data/ + map.html + SKILL.md）
│   ├── scripts/explore.py
│   ├── scripts/amap_key.txt      # 可选：本地放自己的高德 Key（Web 服务），勿提交
│   ├── scripts/flyai_key.txt     # 可选：本地放自己的飞猪 Key，勿提交
│   └── data/                     # themes 308 份 / poi 232 份 / amenities 101 份 / snap 路网网格
├── detour-engine-skill.zip       # 上面目录打好的技能包（直接上传百炼）
├── prompts/system-prompt.md      # 智能体系统提示词（==== 之间是正文）
├── scripts/rebuild_theme_packs.py# 用飞猪重建某城「动漫/非遗」点库
└── docs/                         # 测试报告、优化清单、演示脚本、Issue 正文
```

`map.html`（本仓库根目录）是那条路线的离线展示页：URL 里带坐标，纯静态，无后端。

---

## 本地复现

```bash
# 0) 解压
unzip detour-agent-src-lite.zip && cd detour

# 1) 装飞猪 CLI（国内走 npmmirror 更快）
npm i -g @fly-ai/flyai-cli --registry=https://registry.npmmirror.com

# 2) 先注入自己的 Key（公开交付物不含任何 Key）
export AMAP_KEY=你的高德Web服务Key
export FLYAI_API_KEY=你的飞猪Key
# 2) 跑一条
python3 explore.py --city 杭州 --theme 夜游 --hours 3 \
  --anchor 武林广场 --auto-order --brief

# 输出里有三条方案，每条的「🗺 路线图」URL 点开就是那条街
```

可选参数：

| 参数 | 作用 |
|---|---|
| `--theme 夜游/建筑/非遗/动漫` | 主题 |
| `--anchor 武林广场` / `--coords 经度,纬度` | 出发点（门牌号用后者） |
| `--via "甲,乙"` | 点名必去的 2–3 个点，强插候选池 |
| `--hotel --hotel-date 2026-11-20` | 附按方案各 2 家酒店 |
| `DETOUR_ROUTE_PROVIDER=amap` | 路径从「OSRM 优先」切回「高德优先」（默认 OSRM，省配额） |
| `DETOUR_ENRICH_TTL_DAYS=30` | 周边缓存 TTL |

## 接进百炼（6 步）

1. 新建 **Managed Agent**，模型选 `DeepSeek-V4.1-Flash-1017`，启用 `bash` / `read` / `write`；
2. 把 `prompts/system-prompt.md` 里 `====` 之间的正文粘进系统提示词；
3. 添加 MCP：**Amap Maps 高德地图**（15/15 接口）；
4. 上传并添加技能：`detour-engine-skill.zip`（技能名 `detour-engine`）；
5. 建会话（在会话环境变量里填 **AMAP_KEY** 与 **FLYAI_API_KEY**，或用密钥库）；
6. 问「我在杭州武林广场，今晚 3 小时，想走夜游」。

> 注意：改完提示词或换技能，**老会话仍绑旧快照**，必须新建会话才生效。

---

## 数据来源与边界（写在明面上）

- **主题点库**：`.cache/themes` → 随包 `data/themes` → 高德 `place/text` → 飞猪现建（缓存优先）
- **每站吃/看**：MCP（可选）→ 单站缓存 → 城市池 → 高德 `place/around` → 飞猪
- **步行路径**：OSRM（免费，优先）→ 高德 `direction/walking` 兜底；30 天路段缓存
- **只有检索，没有下单**：飞猪返回 `jumpUrl`，点进去在飞猪 App 完成预订
- **拿不到的**：门票价格（`ticketInfo` 恒为 null）、实时人流、酒店房态——一律写「无数据」而不是编
- **聚类半径 800 m 是产品判断**，对应「走 10 分钟」，不是最优解

更细的取舍写在 `docs/` 里的测试报告与优化清单。

## License

仅用于本次大赛评审与复现，未授权商用。
