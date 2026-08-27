# 版本审计程序

jj 升级后（`jj --version` 与 SKILL.md 版本戳不一致时）重跑本程序，用实测 diff 危险面，决定 SKILL.md 改不改。这个仓库的全部价值是"实测而非记忆"，本文件是它的再生产手段。

## 派发方式

派一个后台 agent，约束如下：

- **主源只有两个**：本机 `jj` 二进制的 `jj help`（含 `jj help -k` 主题页），和一次性 `mktemp -d` 空仓里的实测。
- **禁用**：预训练记忆里的 jj 知识、网上教程、旧版文档。所有断言必须附实测命令与退出码。
- 实验统一走独立配置，避免污染用户配置：

```bash
cat > /tmp/jj-audit.toml <<'EOF'
[user]
name = "T"
email = "t@example.com"
[ui]
paginate = "never"
EOF
export JJ_CONFIG=/tmp/jj-audit.toml
```

## 必测覆盖面

1. **阻塞面**（挂死检测模板）：

   ```bash
   printf '#!/bin/sh\nsleep 60\n' > /tmp/slowed.sh && chmod +x /tmp/slowed.sh
   timeout 8 env EDITOR=/tmp/slowed.sh jj <cmd...>; echo "exit=$?"   # 124 = 挂死
   ```

   逐一测：`describe`/`commit` 无 `-m`；`squash` 在源/目标均有描述时无 `-m`；`split <paths>` 无 `-m`；`resolve`；`diffedit`；各命令 `-i`。

2. **沉默失败面**（exit 0 说谎检测）：同样的命令换 `EDITOR=true` 跑，看 exit 0 之后状态到底变没变（`jj log` 对照）。重点：`describe`、`commit`、`split` 的描述是否真的写进去了。

3. **推送语义**：空仓 + 本地 bare remote，验证 `jj git push` 默认 revset 拒推新 bookmark 时的退出码与文案；`-b` 形式对新 bookmark 是否自动 track；范围内有空描述提交时两种形式各自的退出码。

4. **破坏面**：`jj restore`（无参）对新建文件的行为；`jj abandon`（无参）的默认目标。

5. **revset 语义**：`description("x")` 的模式类型（glob/substring/exact）；`jj log` 默认 revset 对 abandoned 提交与深层祖先的可见性。

6. **命令存废**：上一版危险面里点名的每个命令/标志，用 `jj help <cmd>` 确认还在；报错文案变了要记录原文。

## 产出与收尾

1. 报告落到 `.scratch/facts/jj-<version>-cli.md`（gitignored），每条结论附实测证据。
2. 与上一版 facts 文件 diff **危险面**部分。
3. 危险面有变 → 改 SKILL.md 对应行；无变 → 只更新版本戳 "Verified against jj X.Y.Z"。
4. 本次审计的方法论若有改进（新发现的坑类别），回写进本文件的"必测覆盖面"。
