# Gemini CLI 一次性（Headless）使用說明（供其他 AI Coding CLI 呼叫）

本文件提供 Gemini CLI 在「命令列一次性（非互動）」模式下的完整整合指南，方便其他 AI Coding 工具（如 Claude Code CLI、Codex CLI）以程式化方式呼叫、取得結構化輸出並進行可靠錯誤處理。

- 專案首頁：`README.md`
- Headless 詳細文件：`docs/cli/headless.md`
- 旗標與設定：`packages/cli/src/config/config.ts`、`docs/get-started/configuration.md`

---

## 概觀

- Gemini CLI 是終端機 AI 代理工具，支援一次性非互動（headless）執行：以一條指令送出提示語，輸出文字或 JSON，並以退出碼回報狀態。
- 特色：
  - 工具（檔案、Shell、Web、MCP）擴充能力
  - 多模型選擇（如 gemini-2.5-pro / gemini-2.5-flash / auto 路由）
  - 安全控制（核准模式、白名單工具、沙箱）
  - 可機器解析輸出（JSON、JSONL 事件串流）

---

## 安裝

```bash
# 全域安裝（建議）
npm install -g @google/gemini-cli

# 或免安裝快速體驗
npx https://github.com/google-gemini/gemini-cli

# Homebrew（macOS/Linux）
brew install gemini-cli
```

需求：Node.js 20+；支援 macOS/Linux/Windows。

---

## 認證（非互動建議）

最建議在一次性模式下使用 API Key 或 Vertex AI：

- Gemini API Key：

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

- Vertex AI：

```bash
export GOOGLE_API_KEY="YOUR_API_KEY"
export GOOGLE_GENAI_USE_VERTEXAI=true
#（若需）
export GOOGLE_CLOUD_PROJECT="YOUR_PROJECT_ID"
```

- Login with Google（OAuth）通常用於互動模式，不建議做 headless 自動化。

若未正確設定，非互動流程會以非 0 退出碼結束並輸出明確錯誤（可機器解析）。

---

## 一次性（Headless）用法

- 基本：

```bash
gemini -p "你的提示語"
# 或使用位置參數（與 -p 擇一）
gemini "你的提示語"
```

- 從 STDIN：

```bash
echo "內容" | gemini -p "請摘要"
```

- 引入檔案（@ 路徑）— 會讀入檔案/資料夾內容並附加至訊息：

```bash
gemini -p "請摘要 @README.md 與 @src/**.ts"
# 路徑含空白可跳脫：@docs/My\ File.md
```

- 多資料夾工作區：

```bash
gemini -p "分析專案" --include-directories src,docs
```

限制（避免衝突）：
- 不可同時用位置參數與 `-p`
- 不可同時用 `-p` 與 `-i`
- 透過管線輸入 STDIN 時不可用 `-i`（互動模式）

---

## 輸出格式

- 文字（預設）：

```bash
gemini -p "問題"
```

- JSON（建議給程式解析）：

```bash
gemini -p "問題" --output-format json
```

結構（重點）：

```json
{
  "response": "string",            // 主要回答
  "stats": { /* 模型/工具/檔案統計 */ },
  "error": { "type": "string", "message": "string", "code": 1 }
}
```

- 串流 JSON（JSONL 事件，一行一事件）：

```bash
gemini -p "問題" --output-format stream-json
```

事件類型：`init`、`message`（含 assistant delta）、`tool_use`、`tool_result`、`error`、`result`

重建最終回答：串接 `type=message` 且 `role=assistant` 的 `content`（`delta=true` 代表增量）。

---

## 可靠退出碼（可據此實作重試/告警）

- 成功：`0`
- 一般失敗：`1`（或錯誤物件帶出的 `code/status`）
- 已定義致命錯誤：
  - 認證：`41`
  - 輸入：`42`
  - 沙箱：`44`
  - 設定：`52`
  - 超過回合數（max turns）：`53`
  - 工具致命錯誤：`54`
  - 使用者中斷（Ctrl+C）：`130`

---

## 工具執行與安全（非互動重點）

- 核准模式（`--approval-mode`）：
  - `default`：非互動時會排除需確認/有副作用的工具（預設安全）
  - `auto_edit`：允許檔案編輯類（`edit`/`write_file`）自動核准；`run_shell_command` 仍排除
  - `yolo`：全部工具自動核准（高風險）
  - 舊旗標：`--yolo` 等同 `--approval-mode yolo`（不可與 `--approval-mode` 並用）

- 白名單工具（免確認）：

```bash
--allowed-tools "run_shell_command(git),read_many_files,glob"
```

  - 支援前綴比對與鏈結拆解，搭配 `auto_edit` 可在保守前提下允許特定工具。

- 非互動預設防護：若未設定合適 `--approval-mode/--allowed-tools`，CLI 會自動排除 `run_shell_command`、`edit`、`write_file` 以避免等待核准卡住。

- 限制 MCP 伺服器（避免外部工具干擾）：

```bash
--allowed-mcp-server-names serverA,serverB
```

- 沙箱：

```bash
# 啟用沙箱
gemini -s -p "你的提示"
# 透過環境變數
export GEMINI_SANDBOX=true   # 或 docker/podman/sandbox-exec
# 自訂容器旗標（Podman 例）
export SANDBOX_FLAGS="--security-opt label=disable"
```

---

## 建議旗標組合（整合常見場景）

- 純文字回覆（最小、可解析）：

```bash
gemini -p "你的指令" --output-format json -e none
```

- 可讀檔但不可執行 Shell（安全）：

```bash
gemini -p "說明 @README.md" \
  --output-format json \
  --approval-mode default \
  --allowed-tools "read_many_files,glob" \
  -e none
```

- 允許檔案編輯（不跑 Shell，建議配沙箱）：

```bash
gemini -p "請修正 @src/app.ts" \
  --output-format json \
  --approval-mode auto_edit \
  --allowed-tools "read_many_files,glob" \
  -s
```

- 允許特定 Shell（受限）：

```bash
gemini -p "執行測試並摘要" \
  --output-format stream-json \
  --approval-mode auto_edit \
  --allowed-tools "run_shell_command(npm),read_many_files,glob" \
  -s
```

- 完全自動（高風險，務必沙箱與白名單）：

```bash
gemini -p "部署" \
  --output-format stream-json \
  --approval-mode yolo \
  --allowed-tools "run_shell_command(git),run_shell_command(npm),read_many_files,glob" \
  -s
```

---

## 整合範例

- 擷取 JSON 並解析：

```bash
resp=$(gemini -p "摘要 @README.md" --output-format json)
echo "$resp" | jq -r '.response'
```

- 事件串流監控（逐行 JSONL）：

```bash
gemini -p "執行測試" --output-format stream-json | jq -rc 'select(.type=="result")'
```

- 管線輸入：

```bash
git diff | gemini -p "寫 120 字內的 commit 訊息" --output-format json | jq -r '.response'
```

- 多資料夾：

```bash
gemini -p "分析此專案" --include-directories ../lib,../docs --output-format json
```

---

## 錯誤處理與重試

- 依退出碼分類與重試（如 5xx/暫時性錯誤指數退避）。
- JSON 模式讀取 `error.type/message/code`；串流模式監看 `type=error/result`。
- 若下游提前關閉管線（EPIPE），CLI 會優雅地以 0 結束（不視為錯誤）。

---

## 模型與效能

- 指定模型：

```bash
gemini -m gemini-2.5-flash   # 快速
gemini -m gemini-2.5-pro     # 較強
```

- 預設可由 `GEMINI_MODEL` 或設定檔決定；未設時 `auto` 由路由器挑選。
- 除錯：`--debug`；停用色彩：`NO_COLOR=1`。

---

## 常見限制與注意事項

- 不能同時：位置參數與 `-p`；`-p` 與 `-i`；管線輸入與 `-i`。
- 非互動模式若未適當設定核准/白名單，需核准的工具會被排除或阻塞，請套用上節建議旗標。
- 未信任資料夾時會強制降回 `approval-mode=default`（企業/安全環境常見）。

---

## 設定與環境變數（摘要）

- 設定檔（JSON）：
  - 使用者：`~/.gemini/settings.json`
  - 專案：`./.gemini/settings.json`

- 常用環境變數：
  - 認證：`GEMINI_API_KEY`、`GOOGLE_API_KEY`、`GOOGLE_CLOUD_PROJECT`
  - 模型：`GEMINI_MODEL`
  - 沙箱：`GEMINI_SANDBOX`、`SANDBOX_FLAGS`
  - Proxy：`HTTPS_PROXY/HTTP_PROXY`
  - 輸出與除錯：`NO_COLOR`、`DEBUG`/`DEBUG_MODE`

---

## 旗標速覽（重點）

- `-p, --prompt`：一次性非互動提示語
- `-m, --model`：指定模型
- `-o, --output-format`：`text|json|stream-json`
- `-s, --sandbox`：啟用沙箱
- `--approval-mode`：`default|auto_edit|yolo`
- `--allowed-tools`：白名單工具（可逗號或多次指定）
- `--include-directories`：加入額外目錄
- `-e, --extensions`：指定擴充（`-e none` 停用全部）
- `--allowed-mcp-server-names`：限制可用 MCP 伺服器
- `-d, --debug`、`-h, --help`、`-v, --version`

---

## 內嵌最小範本（給其他 CLI）

- 安全 JSON 一次性（僅讀檔）：

```bash
gemini -p "$PROMPT" --output-format json --approval-mode default --allowed-tools "read_many_files,glob" -e none
```

- 允許編輯（建議配沙箱）：

```bash
gemini -p "$INSTRUCTION with @path" --output-format json --approval-mode auto_edit --allowed-tools "read_many_files,glob" -s
```

---

## 參考

- `docs/cli/headless.md`：headless 模式細節
- `docs/cli/sandbox.md`：沙箱
- `docs/tools/*.md`：工具與限制（含 `run_shell_command`）
- `docs/get-started/configuration.md`：完整設定
- `packages/cli/src/config/config.ts`：旗標解析與非互動安全策略

