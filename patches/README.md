# thought_signature 补丁

| 文件 | 轮次 | 怎么用 |
|---|---|---|
| `thought-signature-gemini-3.7.patch` | 第一轮：Flash 家族泛化 + 注入 `skip_thought_signature_validator` | **不要**再 apply。官方 v4.5.7 已吸收。只作历史对照 |
| `thought-signature-invalid-p0.patch` | 第二轮：禁止 latest 复用、禁止哨兵、sanitizer 保留客户端 thinking | 相对官方 **v4.5.6** |
| `thought-signature-openai-p1.patch` | 第三轮：OpenAI 兼容层按 tool_call 透传/回放签名，session/tool 缓存 7 天 | 相对 **已经 apply 了 P0 的树**，不要直接打到裸 v4.5.6 |

从一份干净的官方 v4.5.6 复现当前树：

```bash
git clone --branch v4.5.6 --depth 1 https://github.com/lbjlaq/Antigravity-Manager.git
cd Antigravity-Manager
git apply patches/thought-signature-invalid-p0.patch
git apply patches/thought-signature-openai-p1.patch
```

背景见仓库根目录 [WHY-WE-PATCH.md](../WHY-WE-PATCH.md)。
