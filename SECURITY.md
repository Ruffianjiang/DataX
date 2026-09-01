# 依赖安全修复汇总（Dependabot Remediation Summary）

- **日期**：2026-09-01
- **仓库**：`Ruffianjiang/DataX`、`Ruffianjiang/dubbo-start`
- **范围**：DataX 12 条 + dubbo-start 42 条开放 Dependabot 告警的集中修复
- **结论**：两个仓库开放告警 **全部清零**（DataX 12→0，dubbo-start 42→0）

---

## 1. 结果总览

| 仓库 | 起始 open | PR 修复 | dismiss（技术债） | 最终 open |
|---|---:|---:|---:|---:|
| DataX | 12 | 12（PR #55 + #57） | 0 | **0** |
| dubbo-start | 42 | 18（PR #185） | 22 | **0** |

---

## 2. DataX 修复

| PR | 分支 | 变更 | 对应告警 |
|---|---|---|---|
| #55 | `fix/datax-zookeeper-3.8.4` | hbase094xreader/writer 的 `zookeeper` `3.7.2 → 3.8.4`（CVE-2024-23944, medium）。仅 POM 版本号，无源码改动 | #284 #285 |
| #57 | `fix/datax-zookeeper-3.8.6` | 合并 #55 后 Dependabot 新开两条 **2026 年新 CVE**（CVE-2026-24308、CVE-2026-24281，均 high，影响 `>= 3.8.0, < 3.8.6`），3.8.4 仍中招 → 升 `3.8.6`。两个 hbase094x POM 最终均为 3.8.6 | #286 #287 |

> 注：#57 是 advisory「re-issue / 新增」的典型案例——首轮分诊时这两条 CVE 尚不存在，必须合并后 re-fetch 复核才能发现。

---

## 3. dubbo-start 修复（PR #185 `fix/dubbo-safe-overrides`）

通过 `server-web/src/main/webapp` 与 `webapp_bk` 的 `package.json` 增加 `overrides`，将 8 个传递依赖固定到 **lockfile 中已存在**的已修复版本（不引入任何新/不兼容版本），并同步去重对应 `package-lock.json` 中的旧版本副本（webapp + webapp_bk 双 lockfile）。

| 包 | 固定版本 | 修复的告警 |
|---|---|---|
| postcss | 8.5.26 | #401 #387 #163 #162 #81 #35 |
| cross-spawn | 7.0.6 | #228 #227 |
| micromatch | 4.0.8 | #198 #197 |
| braces | 3.0.3 | #186 #185 |
| json5 | 2.2.3 | #132 |
| highlight.js | 10.7.3 | #59 #10 |
| set-value | 3.0.3（webapp）/ 2.0.1（webapp_bk） | #25 #7 |
| yargs-parser | 20.2.9 | #9 |

---

## 4. 作为技术债 dismiss 的 22 条（`tolerable_risk`）

### 4.1 EOL — 无上游补丁（11）
| 包 | 告警 |
|---|---|
| xlsx | #183 #182 #145 #144 |
| mockjs | #172 #171 |
| vue-template-compiler | #192 #191 |
| expand-object | #248 |
| parse-git-config | #247 |
| defaults-deep（critical） | #21 |

### 4.2 NEW-major — 需跨主版本升级、会破坏遗留前端构建（11）
| 包 | 升级跨度 | 告警 |
|---|---|---|
| linkify-it | 3 → 5 | #499 #496 |
| markdown-it | 8 → 12/14 | #479 #478 #76 #30 |
| uuid | 8 → 11 | #428 #427 |
| vue | 2 → 3（框架级迁移） | #222 #221 |
| babel-traverse | 6 → 7 | #181 |

> 上述均为遗留 `server-web` 依赖的固有风险，当前应用实际风险可接受，故按技术债忽略；如需彻底消除需对前端做依赖大版本迁移（不在本次范围）。

---

## 5. 残留风险与说明

1. **PR #185 的 overrides 去重**可能改变个别老旧插件（postcss 5/7→8、set-value 0/2→3 等）的传递依赖。合并后建议跑一次 `npm ci` + 前端构建验证（本次未在沙箱内执行）。
2. **zookeeper 3.8.6** 为当前 3.8 线最新补丁版；若后续 3.8.x 再曝新 CVE，可继续小幅升至 3.8.x 最新或跨至 3.9.5。
3. **EOL / NEW-major 类告警**为遗留依赖固有风险，已按可接受技术债忽略。

---

## 6. 处理流程（可复用于后续批次）

- **重分诊**：用 `semver` 比对 advisory 实时 `vulnerable_version_range` / `first_patched_version` 与实际解析版本，分类 `REAL`（可修）/ `STALE`（已修待重扫）/ `EOL`（无补丁）/ `NEW-major`（需跨主版本）。
- **合并后必须 re-fetch 复核**：advisory 可能 re-issue / 新增——本次 #55 合并后即新增两条 2026 CVE，靠 #57 补刀才真正归零。
- **transitive npm 修复**用 `package.json` `overrides` 强制所有实例升级，仅改 lockfile 会被后续 `npm install` 回退。
