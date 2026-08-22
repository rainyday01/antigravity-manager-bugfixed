# thought_signature 补丁

这两份 diff 都相对官方 tag **v4.5.6**（`a2e3c45`）。不要按顺序叠到同一棵树上，会冲突。

| 文件 | 轮次 | 当前源码要不要再 apply |
|---|---|---|
| `thought-signature-gemini-3.7.patch` | 第一轮：Flash 家族泛化 + 注入 `skip_thought_signature_validator` | **不要**。只作历史对照。官方 v4.5.7 已吸收这一轮 |
| `thought-signature-invalid-p0.patch` | 第二轮：禁止 latest 复用、禁止哨兵、sanitizer 保留客户端 thinking | **已经在工作树里**。这就是本仓库相对 v4.5.6 的生效 diff |

从一份干净的官方 v4.5.6 源码复现当前树：

```bash
git clone --branch v4.5.6 --depth 1 https://github.com/lbjlaq/Antigravity-Manager.git
cd Antigravity-Manager
git apply patches/thought-signature-invalid-p0.patch
```

背景、因果链、以及为什么不升级官方 v4.5.7 / v4.5.8，见仓库根目录 [WHY-WE-PATCH.md](../WHY-WE-PATCH.md)。
