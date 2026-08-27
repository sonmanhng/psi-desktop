# psi-desktop — Tài liệu đầy đủ

> Giao diện Electron cho psi-cli. **Không có logic agent riêng**: main process import thẳng `src/` của psi-cli (loop, tool, permission, session, quality loop…) và bọc bằng một `AgentBridge` phát sự kiện sang renderer React. Mọi thứ trong psi-cli (repo riêng, `docs/psi-cli.md`) (tool, slash command, hooks, plugin, vòng "xong việc", cascade) đều áp dụng y nguyên ở đây — tài liệu này chỉ nói phần **thêm** của GUI.

---

## Mục lục
1. [Chạy & build](#1-chạy--build)
2. [Bố cục màn hình](#2-bố-cục-màn-hình)
3. [Sidebar: workspace & session](#3-sidebar)
4. [Khung chat](#4-khung-chat)
5. [Ô nhập: gửi, đính kèm, slash, skill, model](#5-ô-nhập)
6. [Hộp thoại phê duyệt & câu hỏi](#6-phê-duyệt--câu-hỏi)
7. [Bốn panel bên phải: Changes, Tools, Trace, Code graph](#7-panel-bên-phải)
8. [Settings](#8-settings)
9. [Provider & cách kết nối từng loại](#9-provider)
10. [Theme](#10-theme)
11. [Nhiều session chạy song song](#11-nhiều-session-song-song)
12. [Slash command trong GUI: khác gì terminal](#12-slash-trong-gui)
13. [Kiến trúc: main / preload / renderer, IPC](#13-kiến-trúc)
14. [Bản đồ source](#14-bản-đồ-source)

---

## 1. Chạy & build

```bash
cd desktop
npm install
npm run dev        # tsup --watch (main) + vite :5173 (renderer) + electron
npm run build      # verify:electron → build:main (tsup) → build:renderer (vite) → ../dist
npm run typecheck
```
`verify:electron` (`scripts/ensure-electron-runtime.mjs`) kiểm tra runtime Electron trong `node_modules/electron/dist` — máy từng gặp cache zip hỏng nên script này tự sửa. Output nằm ở `<repo>/dist/main` và `<repo>/dist/renderer`.

Cấu hình đọc từ **cùng** `~/.psi-cli/config.json` với CLI; lựa chọn riêng của GUI (provider, theme, key từng provider) ở `~/.psi-cli/desktop-settings.json` — đổi provider trong desktop **không** ảnh hưởng terminal.

---

## 2. Bố cục màn hình

```
┌ Sidebar ─┬───────────── Chat ──────────────┬─ Panel phải (tuỳ chọn) ─┐
│ workspace │  message / thinking / tool rows  │ Changes · Tools · Trace │
│ sessions  │                                  │ · Code graph            │
│ settings  ├──────────────────────────────────┤                         │
│           │  Ô nhập  [model ▾] [📎] [Gửi]     │                         │
└───────────┴──────────────────────────────────┴─────────────────────────┘
```
Panel phải kéo đổi cỡ được (`usePanelWidth`); sidebar ẩn/hiện bằng nút ở header.

---

## 3. Sidebar

- **Workspace**: thư mục làm việc hiện tại; nút chọn thư mục khác (`workspace:pick`) → bridge đổi cwd, nạp lại system prompt, PSI.md, skills, agents, plugin.
- **Session mới** / danh sách session của mọi thư mục, ô **tìm lịch sử** (theo tên, nội dung, thư mục). Mỗi dòng: tên (đặt bằng `/rename` hoặc nút ✎), thư mục, số message; **chấm "Still running in background"** khi session đó đang có lượt chạy.
- Resume: mở lại session (history + todos), xoá session (file `.jsonl`), đổi tên.
- Nhãn provider · model đang dùng, nút Settings.

---

## 4. Khung chat

- Message người dùng (kèm ảnh/file đính kèm), assistant (markdown render, code highlight), **thinking** (card thu gọn, mở ra xem reasoning + thời gian + số token ước lượng), tool call (thẻ ngắn — chi tiết ở panel Tools), system row màu xám cho ghi chú hook / vòng chất lượng (`[verify] …`, `[critique] …`, `· repaired malformed arguments…`, `· planning gate …`, `· <model> failed … retrying on fallback …`).
- Trong lúc chạy: chunk stream, thinking live, dòng tool "Running…", có thể **Cancel (Esc)**.
- Đầu mỗi lượt, nếu là message đầu của session mới mà trông như "tiếp tục…", bridge tự đính **ghi chú định hướng** (các session gần đây của thư mục) để model không mò mẫm.
- Nếu psi-ide đang mở cùng thư mục và `ide.autoContext` bật, file đang mở + vùng chọn của IDE được đính vào lượt.

---

## 5. Ô nhập

| Thao tác | Hành vi |
|---|---|
| **Enter** | Gửi; **Shift+Enter** xuống dòng |
| **Esc** | Huỷ lượt đang chạy |
| `/` + gõ | Menu gợi ý slash command (↑↓ chọn, Tab/Enter điền, Esc đóng) |
| `/skill:` hoặc `/skill <tên>` | Chuyển sang **picker skill** |
| 📎 hoặc kéo-thả | Đính kèm: **ảnh** (png/jpg/gif/webp…) gửi dạng multimodal; **file text** được nhúng nội dung vào prompt; file nhị phân chỉ gửi tên |
| Chip model ▾ | Đổi model ngay: danh sách lấy **live** từ provider (Anthropic `/v1/models`, Empero `/v1/models`, AgentRouter 3 model opus, Token Harbor `th-orchestra` + model bạn nhập) |
| Chip thư mục | Chọn workspace khác |

---

## 6. Phê duyệt & câu hỏi

- **Approval**: khi tool cần quyền (Bash, ghi ngoài workspace, tạo tool mới, `git init` cho worktree…) hiện hộp thoại `Once / Always / Deny`; "Always" lưu rule vào `~/.psi-cli/config.json`. Lệnh nguy hiểm không có "Always". Hộp thoại xếp hàng — hai agent song song không bao giờ mở hai dialog cùng lúc.
- **AskUserQuestion**: `QuestionModal` 1–4 câu trắc nghiệm (chọn một/nhiều, ô "khác"); trả lời quay về agent.
- Permission mode (`acceptEdits`, `bypassPermissions`…) chỉnh ở Settings → Quality hoặc `/permissions mode`.

---

## 7. Panel bên phải

Nút ở rail phải mở từng panel; các panel dùng chung khung kéo cỡ.

| Panel | Nội dung |
|---|---|
| **Changes** | Mọi file Write/Edit trong hội thoại (mỗi path giữ lần sửa mới nhất), bấm xem diff có đánh số dòng và highlight |
| **Tools** | Từng tool call với **INPUT / RESULT** đầy đủ (đã chuyển khỏi luồng chat cho gọn) |
| **Trace** | Sơ đồ thao tác **thời gian thực**: request LLM (ψ), tool call, mỗi lần spawn sub-agent một **lane** riêng (foreground **và** background). Hai chế độ: *List* (timeline thụt lề theo depth) và *Flow* (đồ thị n8n-style: fork/join cho batch song song, cạnh nét đứt agent cha → con). Nút Clear |
| **Code graph** | Đồ thị import/dependency của workspace (`codegraph:get`), cột trái = importer, phải = imported; lọc theo thư mục và "hub-first" cho repo lớn |

---

## 8. Settings

Tab: **General · Models · Quality · API Keys · Skills · Theme**.

- **General**: phiên bản, đường dẫn session/settings, *Danger zone → Clear current chat*.
- **Models**: thẻ provider (chọn thẻ = provider hoạt động): OpenRouter, Anthropic, AgentRouter, Token Harbor, Empero, Ollama. Mỗi thẻ có ô model/base URL/key riêng và nút **Test & save** — test đúng giá trị trên màn hình rồi mới lưu và **chuyển provider live**.
- **Quality**: toàn bộ vòng "xong việc" (`verify`, chạy cả test, `critique`, planning gate, sửa tool-call, test-first, autofix), **permission mode**, **effort**, và **model cascade** (planner/judge/critic/fallback). Ghi vào `~/.psi-cli/config.json` → có hiệu lực cho cả terminal và IDE.
- **API Keys**: quản lý key mọi provider một chỗ: Test & save / Hiện / Copy / Clear, nút *Kiểm tra tất cả*.
- **Skills**: tạo skill mới (global hoặc project) ngay trong app, xem/xoá skill.
- **Theme**: 7 theme (xem §10).

---

## 9. Provider

| Provider | Cần gì | Lưu ý |
|---|---|---|
| **OpenRouter** | key `sk-or-…` (test bằng request thật) | Mặc định; model là chuỗi tự do |
| **Anthropic** | *API key* (`sk-ant-api…`) **hoặc** *Login tài khoản*: nút "Mở claude.ai" → dán OAuth token `sk-ant-oat…` | Danh sách model lấy live; header beta cho OAuth |
| **AgentRouter** | token `AR-…` từ agentrouter.org/dashboard + **Claude Code CLI đã cài** (`npm i -g @anthropic-ai/claude-code`) | WAF chỉ nhận client Claude Code thật nên psi spawn `claude`. Hai mode: **Chat** (claude không có tool, psi dùng tool của mình) / **Agent** (claude tự dùng tool, bỏ permission của psi). Tự fallback giữa 3 model opus khi 503 |
| **Token Harbor** | key `thk_…` | Nhập model tự do (mặc định `th-orchestra`) |
| **Empero** | không cần key | 2 model miễn phí; **prompt/completion bị log** — cảnh báo đỏ ngay trong thẻ |
| **Ollama** | base URL `http://localhost:11434/v1` | Nút test; tắt prompt caching |

Chọn model ở chip trong ô nhập **ghi vào đúng ô của provider đang dùng** (không đè model OpenRouter như bản cũ); chỉ khi provider là OpenRouter mới đồng bộ sang `config.json` của CLI.

---

## 10. Theme

`light · dark · abyss · nebula · collider · collapse · horizon` — biến CSS đặt lên `<html data-theme>`, đồng bộ màu cửa sổ native (`window:set-theme`). psi-ide dùng cùng bộ.

---

## 11. Nhiều session song song

`SessionManager` giữ **nhiều `AgentBridge`** cùng lúc: chuyển sang session khác trong lúc một session đang chạy thì lượt đó vẫn tiếp tục nền; sidebar đánh dấu; mọi sự kiện renderer mang `sid` để định tuyến đúng cửa sổ chat. Cancel chỉ huỷ session đang xem.

---

## 12. Slash trong GUI

Toàn bộ bảng lệnh của CLI chạy được trong ô chat (`/help` liệt kê). Một số lệnh có bản GUI thay cho thao tác terminal:
- `/exit` → hướng dẫn tạo session mới; `/model` → dùng chip; `/clear` → xoá chat; `/figures --live` → chỉ terminal.
- `/empero` → chỉ hiển thị trạng thái và trỏ sang Settings (bản terminal có prompt chọn model sẽ treo main process).
- `/plugins install|remove|enable|disable` nạp lại agent/skill/hook/prompt ngay.
- Nút "Nén với trọng tâm tuỳ chỉnh" ở header = `/compact <focus>`; compact tôn trọng hook `PreCompact`, backup session gốc `.bak.jsonl`.

---

## 13. Kiến trúc

```
desktop/src/
  main/
    index.ts          tạo cửa sổ, menu, theme
    agent-bridge.ts   AgentBridge: 1 session = 1 bridge (client LLM, registry tool, permission UI, hookHost, LSP registry,
                      change tracker/checkpoint, trace lanes, quality loop, compaction, provider switching)
    session-manager.ts  nhiều bridge, bridge "active"
    ipc.ts            mọi ipcMain.handle (agent:*, session:*, workspace:*, settings:*, codegraph:get, fs:read-text…)
    settings.ts       DesktopSettings (provider, theme, key từng provider) ↔ desktop-settings.json
    slash-runner.ts   chạy bảng slash của CLI với console.log bị bắt lại → khối output
  preload/index.ts    contextBridge → window.electronAPI (invoke + on)
  renderer/
    App.tsx           state hội thoại, sự kiện agent:*, session/workspace, panel
    components/       Sidebar, ChatPanel, MessageView, ThinkingView, InputPanel, QuestionModal,
                      ChangesPanel, ToolsPanel, TracePanel, TraceDiagram, CodeGraphPanel,
                      SettingsModal, QualitySettings, Markdown
    styles.css        theme tokens
```

**Luồng một lượt**: renderer `agent:send-message` → bridge `sendMessage` (slash? skill? continuation note? IDE context?) → `runAgentLoop` với callback stream (`agent:message-chunk`, `agent:thinking`, `agent:tool-call`, `agent:trace`, `agent:hook-note`) → hook Stop → **vòng chất lượng** (verify/critique, các vòng sửa stream vào cùng chat) → `agent:complete` → persist session.

Sự kiện chính renderer nhận: `agent:user-message`, `agent:message-chunk`, `agent:thinking`, `agent:tool-call`, `agent:trace`, `agent:hook-note`, `agent:approval-request`, `agent:question`, `agent:command-output`, `agent:complete`, `agent:cancelled`, `agent:error`, `background:notification`.

---

## 14. Bản đồ source

Xem cây ở §13. Test của lõi (`tests/`) bao phủ bridge gián tiếp qua các module dùng chung; `tests/desktop-settings.test.ts` kiểm tra đọc/ghi/migration `desktop-settings.json`.
