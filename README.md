# Hệ thống Muse — Sơ đồ tư duy + kiến trúc tổng quan

> Bản đồ tương tác của một hệ thống trợ lý AI cá nhân đang chạy thật:
> **Muse** (Boss) điều khiển **9Router** (cổng model), một đội agent,
> nhiều máy, các routine tự động và một hệ bộ nhớ có tổ chức.
>
> Mở file **`index.html`** (hoặc link GitHub Pages của repo) để xem sơ đồ
> tương tác: bấm nút +/− để thu/mở nhánh, bấm vào nút để xem thẻ chi tiết,
> dùng ô tìm kiếm để lọc theo từ khóa.
>
> **Lưu ý:** đây là bản đã khử thông tin riêng (IP, tài khoản, đường dẫn
> nội bộ, số liệu quota) từ hệ thống thật. Số liệu trong sơ đồ là snapshot
> ngày 28/10/2026, không tự cập nhật.

## 1. Vai trò Boss — Muse

Muse là **Boss / Controller / Orchestrator**, giữ quyết định cuối cùng
của mọi dự án. Boss làm 4 việc:

- Phân rã yêu cầu thành task có acceptance criteria kiểm chứng được.
- Giao việc cho đúng agent theo dependency và quyền.
- Chạy tool, kiểm chứng bằng lệnh thật (không nhận PASS miệng).
- Duyệt qua từng gate, rồi lưu trữ kèm bằng chứng.

Thứ tự ưu tiên khi xung đột: lệnh trực tiếp của người dùng >
`SYSTEM_RULES` > cổng Agent Rule > phương pháp luận Superpowers.

Một luật xuyên suốt: **Muse không tự viết code thay Coder**
(trừ việc lặt vặt tầng DIRECT: typo, config, docs). Việc phức tạp bắt
buộc gọi Codex phân tích ngay từ đầu.

## 2. Pipeline Triumvirate — 8 bước tranh biện

Mọi việc liên quan đến code đi qua pipeline 8 bước:

```
Planner → Assign → Coder → Critic → Rebuttal → Judge → Testing → Archive
```

| Vai trò | Model / route | Nhiệm vụ |
|---|---|---|
| Coder | GPT qua route `cx/*` | Viết/sửa code, phản biện lại Critic bằng bằng chứng kỹ thuật |
| Critic | Antigravity/Gemini qua route `ag/*` | Chỉ review DIFF, trả JSON đúng 2 key `verdict` + `issues` |
| Boss | Muse | Chọn phương án, quyết định cuối |

- Critic dừng ngay khi `APPROVE`, tối đa 2 vòng tranh biện.
- Testing chạy lệnh thật trên đúng revision; FAIL quay lại Coder,
  BLOCKED không bao giờ được tính là PASS.
- Đổi revision code = mất hiệu lực evidence review/test cũ.
- Retry tối đa 3 lần rồi ESCALATE.

Ba tầng pipeline, chọn theo mức quan trọng:

- **FULL** — đủ 8 bước, cho task quan trọng (production, bảo mật, migration).
- **LIGHT** — Coder → Critic 1 vòng → Boss quyết, mặc định cho task nhỏ.
- **DIRECT** — việc lặt vặt, bỏ qua pipeline.

## 3. 9Router — cổng model

Gateway chạy local, chỉ nghe `127.0.0.1`, gom nhiều subscription
(OAuth) thành một API chuẩn OpenAI (`/v1/models`,
`/v1/chat/completions`, `/v1/responses`):

- Route `ag/*` — Antigravity / Gemini (critic, tay sinh text).
- Route `cx/*` — GPT Plus / Codex (coder, não phân tích).
- Route `gh/*` — GitHub Copilot (vision, fallback).
- Route `oc/*`, `mmf/*` — các model phụ khác.

Console quản trị (artifact web) là **nguồn sự thật** cho cấu hình
agent: mọi thay đổi local phải upsert lên console rồi verify khớp
bằng `getagents`. Job đồng bộ chạy mỗi 4 giờ: refresh model,
provider và quota snapshot, đẩy lên console.

**Quota theo pool riêng:** mỗi nhóm model (Argon, Gemini weekly,
Claude & GPT weekly, Codex, GitHub premium) có counter và giờ reset
riêng. Bài học thực tế: 3 tier high/medium/low của cùng một họ model
có thể **chung 1 pool** — đổi tier không tạo thêm quota.

## 4. Đội agent (8 agent)

| Agent | Vai trò | Model chính |
|---|---|---|
| planner | Phân rã yêu cầu thành task | Codex |
| coder | Viết/sửa code | Codex |
| critic | Review độc lập | Claude Sonnet |
| testing | Chạy test thật, PASS/FAIL/BLOCKED | GPT Codex |
| researcher | Tìm kiếm, tổng hợp | Gemini Argon |
| writer | Viết nội dung | GPT Codex |
| archivist | Lưu trữ bằng chứng, manifest, SHA | Gemini Argon |
| backup | Dự phòng đa vai trò khi agent chính lỗi hết chain | Gemini Argon |

Mỗi agent có chain fallback riêng. Khi một route chết (503/timeout),
critic bắt buộc chuyển sang Gemini Argon, không được bỏ qua vòng critic.

**Tiết kiệm token — "não Codex, tay Muse/Gemini":**

- Việc cần suy nghĩ/phân tích → Codex (đốt quota Codex, 0 token Muse).
- "Tay" chỉ sinh text → Gemini Argon.
- Việc máy móc định kỳ → script thuần, không LLM.
- Worker có tool luôn là Muse.

## 5. Máy móc

- **VM của Muse** — nơi Boss chạy, mọi thứ quan trọng nằm trong home
  để sống sót qua VM rebuild.
- **Máy A** — môi trường dev/staging: tạo dựng, chỉnh sửa, kiểm chứng
  trước khi promotion lên production.
- **Máy B** — production: hệ vận hành thật, phải sống qua reboot,
  chỉ nhận release đã verify qua gate.
- **Máy C** — máy phụ, dự phòng.

Nguyên tắc phân tách: **B là sự thật, A là nơi cải thiện B.**
Luồng A→B là promotion release đã verify; luồng B→A là feedback
(incidents, nhãn, shadow, dữ liệu). SSH chỉ qua tunnel riêng, shell
Windows mặc định là cmd, process sống lâu đăng ký Scheduled Task.

## 6. Routine tự động

| Routine | Nhịp | Ghi chú |
|---|---|---|
| Bản tin tin tức hằng ngày | Mỗi sáng ~07:23 | Đọc kênh tin tức, bản tin tiếng Việt |
| Backup cài đặt + bộ nhớ | Thứ Hai ~03:23 | Im lặng khi thành công, báo khi lỗi |
| Đồng bộ console 9Router | Mỗi 4 giờ | Model, provider, quota snapshot |
| Cập nhật graph bộ nhớ | Hằng tuần | Rebuild/extract incremental |
| Watchdog gateway | Mỗi 15 phút | Giữ gateway + bộ nhớ agent sống, wake chỉ sau 3 fail liên tiếp |

## 7. Bộ nhớ

Thứ tự tra cứu, dừng ngay khi đủ trả lời (rẻ trước, đắt sau):

1. `MEMORY.md` đã nạp sẵn (giữ dưới ~3.500 từ: chỉ quyết định/số liệu chốt).
2. `memory_search` — quyết định, cuộc trò chuyện cũ.
3. Grep từ khóa chính xác trong daily notes (rẻ nhất khi biết từ khóa).
4. Knowledge graph — chỉ khi 0 hit, >50 hit, hoặc câu hỏi quan hệ.
5. Đọc cửa sổ ~40 dòng quanh hit, không đọc cả file daily.

Fact bền tìm được thì promote 1 dòng lên `MEMORY.md`; fact hết hạn
thì demote/xóa. Chi tiết ở daily notes, MEMORY chỉ giữ cái chốt.

**AgentMemory (best-effort, riêng cho agent 9Router):**
server bộ nhớ dài hạn cho agent stateless — `remember` sau review/test
fail, `smart-search` lấy past lessons cho IMPLEMENT/REBUTTAL.
Nguyên tắc: best-effort, không bao giờ làm hỏng pipeline. Verdict:
không dùng cho Muse để tránh phân mảnh bộ nhớ.

## 8. Quy tắc bắt buộc khi làm việc

- **Karpathy Guidelines** — nghĩ trước khi code, đơn giản trước
  (KISS/YAGNI), sửa kiểu phẫu thuật, mục tiêu kiểm chứng được.
- **Graphify** — đọc codebase bằng knowledge graph
  (`query`/`path`/`explain`) thay vì đọc từng file; sửa code xong
  cập nhật graph.
- **UI/UX Pro Max** — bắt buộc khi thiết kế/build/review giao diện.
- **Superpowers** — phương pháp luận cho Muse Code: brainstorming →
  viết plan → worktrees → thực thi bằng subagent → TDD/debugging →
  verification → review. Superpowers là methodology, Agent Rule mới
  là authority (gates và bằng chứng).
- **Clean-room (reverse engineering)** — model thấy code gốc thì không
  được viết code; model viết code thì không được thấy code gốc; chỉ
  đi qua SPEC đã duyệt.

## 9. Cấu trúc repo này

```
index.html   Sơ đồ tư duy tương tác (file tự chứa, mở là chạy)
README.md    Tài liệu kiến trúc tổng quan (file này)
LICENSE      MIT
```

## License

MIT — xem file `LICENSE`.
