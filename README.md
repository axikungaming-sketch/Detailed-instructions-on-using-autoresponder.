# ✨️ YUKI – All-in-One Bot — Tài liệu `economy.py` & `autoresponder.py`

> Tài liệu kỹ thuật chi tiết cho 2 cog: **Hệ thống Menu Trợ Giúp** (`economy.py`) và **Autoresponder / Custom Command Builder** (`autoresponder.py`). Giải thích từng khối code làm gì, cách dùng, và cách ghép 2 file (cũng như ghép các block bên trong autoresponder) lại với nhau.
>
> 🆕 **Cập nhật mới nhất:** +100 khối (**Nhóm K** — 50 hàm thuần chuỗi/toán học mới + 25 khối kinh tế mở rộng + 25 khối Discord/kiểm duyệt mở rộng) vừa được thêm vào `autoresponder.py`, kèm theo trang trợ giúp mới ở cả 2 file (`!ar help` → Trang 7/7, `!help` → mục Autoresponder Trang 11/11).
>
> _Bản trước đó: +110 khối Nhóm J (xem lịch sử cập nhật ở cuối mỗi bảng)._

## 📑 Mục lục

1. [Tổng quan](#-tổng-quan)
2. [Phần 1 — `economy.py`: Hệ thống Menu Trợ Giúp](#-phần-1--economypy-hệ-thống-menu-trợ-giúp)
3. [Phần 2 — `autoresponder.py`: Autoresponder Engine](#-phần-2--autoresponderpy-autoresponder-engine)
   - [Cấu trúc xử lý 1 reply — chi tiết từng bước (pipeline)](#cấu-trúc-xử-lý-1-reply--chi-tiết-từng-bước-pipeline)
   - [Minh hoạ cách hoạt động của một số khối tiêu biểu](#minh-hoạ-cách-hoạt-động-của-một-số-khối-tiêu-biểu)
4. [Phần 3 — Cách ghép mọi thứ lại với nhau](#-phần-3--cách-ghép-mọi-thứ-lại-với-nhau)
5. [Giới hạn hệ thống](#️-giới-hạn-hệ-thống)

---

## 🧭 Tổng quan

Dù tên file là `economy.py`, cog bên trong (`EconomyCog`, hiển thị tên **"Kinh Tế"**) **không** chứa logic kinh tế thật — nó là **hệ thống Menu Trợ Giúp** (`!help` / `/help`) tổng hợp toàn bộ lệnh của bot (farm, casino, music, autoresponder, admin...) thành các embed điều hướng bằng dropdown/nút bấm. Logic GC/kinh tế thật nằm ở cog khác (ví dụ `levels.py` theo cấu trúc bot hiện tại).

`autoresponder.py` là một cog **độc lập hoàn toàn**, cho phép admin server tự tạo lệnh dạng `trigger → reply` với hàng trăm "khối" (`{block}`) — placeholder, điều kiện, hiệu ứng phụ, logic if/else — mà không cần biết code, giống tinh thần bot Mimu.

```
┌─────────────────────┐        ┌──────────────────────────┐
│    economy.py         │        │   autoresponder.py        │
│  (Cog "Kinh Tế")      │        │   (Cog "Autoresponder")        │
│                       │        │                            │
│  !help /help          │        │  !ar add/edit/del/...            │
│  !setprefix !sync     │        │  !ar welcome / leave         │
│  bật/tắt music (chat) │        │  !ar bridge / rotate      │
│                       │        │  AutoresponderEngine      │
│  → embed + dropdown   │◄──text─┤  (parse & render {block}) │
│    "🤖 Autoresponder"  │  only  │                            │
└─────────────────────┘        └──────────────────────────┘
        ▲                                    ▲
        │            bot nạp cả 2 làm 2 cog riêng biệt qua
        └──────────── main.py (EXTENSIONS list) ─────────────┘
```

Hai file **không import lẫn nhau** — chỉ liên kết ở mức nội dung: `economy.py` có sẵn vài trang trợ giúp tĩnh mô tả lại cú pháp `!ar` để người dùng không phải rời khỏi menu `!help`.

---

## 📂 Phần 1 — `economy.py`: Hệ thống Menu Trợ Giúp

### Vai trò thực sự

| Nhầm tưởng | Thực tế |
|---|---|
| "economy.py" = hệ thống tiền tệ | Đây là **hệ thống `!help`** — tổng hợp & hiển thị lệnh của *toàn bộ* bot |
| Cog tên "Kinh Tế" | Tên hiển thị lịch sử, không phản ánh chức năng thật |
| Chứa logic cộng/trừ GC | Không — chỉ hiển thị *text* lệnh (`!me`, `!daily`...), logic thật nằm ở cog khác |

Ngoài menu trợ giúp, file còn có 2 tính năng phụ không liên quan tới help: **bật/tắt trạng thái Music** (qua chat) và **`!setprefix`/`!sync`**.

### Yêu cầu & phụ thuộc

- `core.lang` — cần các hàm `t(key, uid)`, `get_lang(uid)`, `SUPPORTED` để hỗ trợ đa ngôn ngữ (vi/en/ko/zh/ja).
- `config` — cần `config.get_prefix(guild_id)` và `config.set_prefix(guild_id, prefix)`.
- Thư mục `data/` — chứa `data/music_status.json` (tự tạo nếu chưa có).
- Custom emoji server thật (các ID `discord.PartialEmoji`) — **phải đổi lại ID emoji cho đúng server của bạn**, nếu không emoji sẽ không hiện.

### Các khối chính (block-by-block)

| # | Khối | Dòng | Tác dụng |
|---|---|---|---|
| 1 | Trạng thái Music (`_load/_save/get/set_music_status`) | 12–72 | Lưu online/offline vào `data/music_status.json`, giữ nguyên qua restart. Chủ bot gõ `on`/`off` trong 1 kênh cố định để đổi. |
| 2 | `is_owner_or_administrator()` | 25–34 | Decorator check quyền: chủ bot **luôn** pass, người khác cần quyền Administrator. Dùng cho `!setprefix`. |
| 3 | Emoji tập trung (`EMOJI_*`, `PAGE_EMOJIS`) | 74–120 | Định nghĩa 1 chỗ duy nhất toàn bộ emoji custom dùng trong menu, tránh lặp code. |
| 4 | `PART_MAP` | 122–130 | Ánh xạ **"Phần"** (1, 2, 3, 4, 7...) → danh sách key danh mục thuộc phần đó. Đây là khung xương của menu 2 tầng (chọn Phần → chọn danh mục). |
| 5 | Dữ liệu lệnh tĩnh (`INTERACTIONS_COMMANDS`, `RATINGS_COMMANDS`...) | 132–228 | Danh sách `(tên_lệnh, mô tả)` cho các danh mục có quá nhiều lệnh để nhét vừa field, dùng chung với `PaginatedCommandsView`. |
| 6 | **`_pages(p, uid)`** | 229–1080+ | **Khối lớn nhất** — "kho dữ liệu" toàn bộ nội dung help: mỗi key (`"farm"`, `"economy"`, `"casino"`, `"autoresponder"`…`"autoresponder11"`...) là 1 dict `{label, color, title, desc, fields}`, hỗ trợ đa ngôn ngữ. Đây là nơi **duy nhất** cần sửa khi muốn đổi nội dung 1 trang help. |
| 7 | `_build(page_key, prefix, uid)` | — | Lấy 1 entry từ `_pages()` và dựng thành `discord.Embed` thật. |
| 8 | `RoastWarningView`, `CasinoSelect`/`CasinoHelpView` | — | View cảnh báo nội dung 18+ (roast) và view phụ điều hướng riêng cho danh mục casino (nhiều game con). |
| 9 | `PaginatedCommandsView` | — | View phân trang tái sử dụng cho danh mục có danh sách lệnh dài (vd. Thám Hiểm Rừng — 2 trang). |
| 10 | `AutoresponderHelpView` | — | View phân trang **🆕 11 trang tĩnh** (`autoresponder`…`autoresponder11` trong `_pages()`) mô tả cú pháp `!ar` ngay trong menu `!help`, không cần rời sang lệnh `!ar`. Trang 11 mới nhất giới thiệu +100 khối Nhóm K (chuỗi/toán học nâng cao, kinh tế mở rộng, Discord/kiểm duyệt mở rộng). |
| 11 | `Part4View` / `Part4Select` | — | UI riêng cho Phần 4 (Interactions/Ratings/Anime/Manga) — 4 select-menu con trên cùng 1 view. |
| 12 | `MainPartSelect` → `PartCategorySelect` → `PartCategoryView` → `HelpView` | — | Chuỗi điều hướng chính: **Trang chủ → chọn Phần → chọn danh mục trong Phần → xem embed**, nút "🔙" quay lại đúng cấp trước. |
| 13 | `EconomyCog` | — | Cog thật — nơi đăng ký các lệnh Discord (`!help`, `!sync`, `/help`, `!setprefix`) và listener `on_message` cho việc bật/tắt music. |

### Cách dùng (người dùng cuối)

```
!help          → mở menu trợ giúp (tự xoá sau 60 giây, kể cả tin nhắn gọi lệnh)
!yuhelp        → alias của !help
/help          → bản slash command, không tự xoá, ephemeral=False
!setprefix !   → đổi prefix server (cần Administrator, hoặc là chủ bot)
!sync          → đồng bộ slash command lên Discord (chỉ chủ bot dùng được)
```

Điều hướng: chọn dropdown **"Chọn phần"** ở trang chủ → chọn 1 danh mục trong dropdown thứ 2 → xem embed lệnh; nút **"🔙 Chọn phần khác"** / **"🏠 Trang chủ"** để quay lại.

Bật/tắt Music: chỉ chủ bot, gõ đúng chữ `on` hoặc `off` (không kèm prefix) trong kênh có ID khai báo ở `MUSIC_STATUS_CHANNEL_ID`.

### Cách tuỳ biến / mở rộng

Muốn thêm 1 trang help mới (vd cho cog mới bạn viết):

1. Thêm 1 entry vào dict trả về của `_pages()` — copy cấu trúc `{label, color, title, desc, fields}` của 1 trang có sẵn.
2. Thêm emoji cho trang đó vào `PAGE_EMOJIS`.
3. Thêm key đó vào danh sách của đúng "Phần" trong `PART_MAP` (hoặc tạo phần mới, ví dụ `8: ["ten_key_moi"]`).
4. Nếu trang cần logic điều hướng đặc biệt (như casino/autoresponder/tham_hiem) thì thêm 1 nhánh `elif key == "..."` trong `PartCategorySelect.callback`.

> 💡 **Ví dụ gần nhất áp dụng đúng quy trình trên:** khi thêm +100 khối Nhóm K vào `autoresponder.py`, trang `"autoresponder11"` được thêm vào `_pages()`, key `11: "autoresponder11"` được thêm vào dict ánh xạ trong `AutoresponderHelpView.build_embed()`, và điều kiện `if self.page < 10` được nâng lên `if self.page < 11` để nút "Trang sau ▶" đi tới được trang mới.

---

## 📂 Phần 2 — `autoresponder.py`: Autoresponder Engine

### Vai trò

Cho phép admin server tạo lệnh tùy chỉnh `trigger → reply` ngay trong Discord, không cần sửa code — reply có thể chứa **placeholder** (`{user}`, `[$1]`...) và **hàm/khối** (`{modifybal:}`, `{addrole:}`, `{ifbal:}`...) để tạo mini-game, hệ thống ticket, self-role, kiểm duyệt tự động, welcome dashboard...

### Kiến trúc tổng thể

```
SQLite (data/autoresponder.db, WAL mode)
   │  bảng: autoresponders, ar_cooldowns, ar_balances, ar_inventory, ar_bank,
   │  ar_embed_templates, ar_uservars, ar_guildvars, ar_counters, ar_events,
   │  ar_bridges, ar_rotations
   ▼
Economy Adapter (eco_get_balance, eco_modify_balance, eco_transfer...)
   │  ← 5-6 hàm CHỌN 1 CHỖ để nối vào GC thật nếu muốn (xem mục Phần 3)
   ▼
AutoresponderEngine  (class chính — parse + render 1 reply)
   ├─ check_gates()          → điều kiện chặn (permission, role, channel, time...)
   ├─ resolve_loop_blocks()   → xử lý {loop:}{start}...{stop}, hỗ trợ bước nhảy + 7 token mở rộng
   ├─ resolve_str_functions() → 🆕 203 hàm chuỗi/toán học/ngẫu nhiên/mã hoá thuần (Nhóm A+B, I, J, K)
   ├─ resolve_conditionals()  → xử lý {ifvar:}/{ifbal:}/{ifrole:}/{ifarg:}
   ├─ render_text()           → thay placeholder còn lại
   ├─ run_side_effects()      → thực thi hiệu ứng phụ (addrole, modifybal, kick, banktransfer, softban...)
   └─ run()                   → gọi tuần tự các bước trên, trả về ARResult
   ▼
Autoresponder (commands.Cog)
   ├─ on_message → find_matching_trigger() → engine.run()
   ├─ nhóm lệnh !ar (add/edit/del/list/show/embed/welcome/leave/bridge/rotate)
   └─ UI block: Select Menu / Role Select / Modal Button (no-code)
```

### Cấu trúc xử lý 1 reply — chi tiết từng bước (pipeline)

Khi 1 tin nhắn khớp trigger, `engine.run()` không "chạy 1 lần từ trái sang phải" — nó xử lý reply qua **6 giai đoạn tuần tự**, mỗi giai đoạn chỉ quan tâm 1 loại khối, giai đoạn sau chỉ nhận input đã được giai đoạn trước xử lý xong:

```
<reply gốc, còn nguyên mọi {block}>
        │
        ▼
① check_gates()            // quét toàn bộ {require*}/{deny*}/{cooldown}/{only*}
        │  Có gate nào FALSE?
        │   ├─ CÓ  → dừng ngay, không chạy tiếp bất kỳ giai đoạn nào,
        │   │        hiện lỗi (trừ khi có {silent})
        │   └─ KHÔNG → xoá hết khối gate khỏi text, đi tiếp
        ▼
② resolve_loop_blocks()    // tìm {loop:...}{start}...{stop}, lặp & nhân bản nội dung
        │  Output: text đã "giãn nở" — 1 vòng loop biến thành N đoạn text nối nhau
        ▼
③ resolve_str_functions()  // tính mọi {uppercase:} {math:} {ncr:} ... (203 hàm thuần)
        │  Hàm thuần = không tác động Discord, không cần await DB,
        │  chỉ nhận chuỗi/số từ tham số & trả ra 1 chuỗi kết quả
        ▼
④ resolve_conditionals()   // {ifvar:}/{ifbal:}/{ifrole:}/{ifarg:} → chọn nhánh đúng/sai
        │  Chỉ giữ lại nhánh thắng, xoá hẳn nhánh thua khỏi text
        ▼
⑤ render_text()            // thay nốt placeholder còn sót: {user} [$1] {server_name}...
        │  Output: text HIỂN THỊ CUỐI CÙNG sẽ gửi lên Discord
        ▼
⑥ run_side_effects()       // duyệt lại {block} theo ĐÚNG thứ tự xuất hiện trong reply gốc,
        │                     thực thi từng cái: addrole, modifybal, kick, banktransfer...
        ▼
   gửi tin / embed lên Discord (hoặc DM/kênh khác nếu có {dm}/{sendto:})
```

**Vì sao chia làm 6 giai đoạn thay vì xử lý 1 lượt?** Vì thứ tự phụ thuộc dữ liệu: `{ifbal:}` (giai đoạn ④) cần biết `{modifybal:}` **chưa** chạy (side effect luôn ở giai đoạn ⑥, sau cùng) để so sánh đúng số dư **trước khi** trigger này thay đổi nó; `{loop:}` (giai đoạn ②) phải chạy trước `resolve_str_functions` để các hàm chuỗi/toán học bên trong `{start}...{stop}` được tính **riêng cho từng vòng lặp** thay vì tính 1 lần rồi lặp lại y hệt.

### Minh hoạ cách hoạt động của một số khối tiêu biểu

#### 1. Khối Gate — `{requirerole:@VIP}`

```
Input:  {requirerole:@VIP}Chào {user}, khu vực VIP!
        │
        ▼
Bước 1: Parse "@VIP" → tra ra role_id thật trong guild
Bước 2: Lấy danh sách role hiện có của member vừa gõ trigger
Bước 3: role_id có nằm trong danh sách đó không?
        │
        ├─ CÓ   → xoá {requirerole:@VIP} khỏi text, tiếp tục pipeline
        │          → render ra: "Chào Tí, khu vực VIP!"
        │
        └─ KHÔNG → huỷ toàn bộ reply ngay tại đây (các giai đoạn ②-⑥ KHÔNG chạy)
                   → nếu không có {silent}: bot trả lời
                     "❌ Bạn cần role @VIP để dùng lệnh này."
                   → nếu có {silent}: im lặng hoàn toàn, không gửi gì
```

#### 2. Khối Side Effect — `{modifybal:+1000}`

```
Input:  {modifybal:+1000}Bạn vừa nhận 1000 GC!
        │
        ▼
Giai đoạn ⑤ (render_text): khối này KHÔNG bị đổi thành gì ở bước render —
        nó chỉ được "đánh dấu vị trí" rồi giữ nguyên tới giai đoạn ⑥.
        Text hiển thị cho user: "Bạn vừa nhận 1000 GC!"
        │
        ▼
Giai đoạn ⑥ (run_side_effects), CHẠY SAU KHI tin nhắn trên đã được gửi:
Bước 1: Parse toán tử "+" và số "1000"
Bước 2: Kiểm tra |1000| ≤ MAX_BAL_CHANGE (100,000) → hợp lệ
Bước 3: await eco_modify_balance(guild_id, user_id, '+', 1000)
Bước 4: Ghi số dư mới vào bảng ar_balances (hoặc bảng GC thật nếu đã nối Adapter)
        │
        ▼
Nếu có NHIỀU {modifybal:}/{addrole:}/{kick:}... trong 1 reply,
chúng chạy TUẦN TỰ đúng thứ tự xuất hiện trong text gốc — khối sau
luôn thấy được kết quả (số dư, role...) mà khối trước vừa đổi.
```

#### 3. Khối Loop — `{loop:1-3}{start}#[loop]{loopsep:, }{stop}`

```
Input:  {loop:1-3}{start}#[loop]{loopsep:, }{stop}
        │
        ▼
Bước 1: Parse spec "1-3" → range = [1, 2, 3], total = 3 vòng
Bước 2: Với mỗi vòng i (i = 1, 2, 3), lấy riêng nội dung giữa {start}...{stop}
        rồi thay các token: [loop]=i, [loopindex]=i-1,
        [loopfirst]="true" nếu i==1, [looplast]="true" nếu i==vòng cuối
Bước 3: Nối kết quả từng vòng lại, chèn {loopsep:, } giữa 2 vòng liên tiếp
        — riêng vòng CUỐI thì KHÔNG chèn separator phía sau

  Vòng 1 (i=1): "#1"        →  giữ nguyên (vòng đầu, chưa cần separator)
  Vòng 2 (i=2): "#2"        →  nối thêm ", " phía trước: ", #2"
  Vòng 3 (i=3): "#3"        →  nối thêm ", " phía trước: ", #3"
                                  (KHÔNG thêm ", " sau vì là vòng cuối)
        │
        ▼
Output sau giai đoạn ②:  "#1, #2, #3"
→ Text này mới được đưa tiếp qua giai đoạn ③④⑤ như 1 đoạn text bình thường.
```

#### 4. Khối Điều kiện (if/else) — `{ifbal:>=|1000|Bạn giàu rồi đó!|Cố kiếm thêm GC nhé.}`

```
Input:  {ifbal:>=|1000|Bạn giàu rồi đó!|Cố kiếm thêm GC nhé.}
        │
        ▼
Bước 1: Parse toán tử ">=" , mốc so sánh "1000"
Bước 2: await eco_get_balance(guild_id, user_id)  → giả sử trả về 1500
Bước 3: So sánh: 1500 >= 1000 ?
        │
        ├─ ĐÚNG  → giữ lại nhánh "Bạn giàu rồi đó!", XOÁ hẳn nhánh sai
        └─ SAI   → giữ lại nhánh "Cố kiếm thêm GC nhé.", XOÁ hẳn nhánh đúng
        │
        ▼
Output:  "Bạn giàu rồi đó!"   (nếu số dư ≥ 1000)
→ Nhánh bị loại biến mất hoàn toàn khỏi text, không hiện ra dù ẩn hay lộ.
```

#### 5. Khối Random liên kết — `{choose:A|B|C}` + `{lockedchoose:X|Y|Z}`

```
Input:  Bạn bốc được lá {choose:A|B|C}! Phần thưởng tương ứng: {lockedchoose:100 GC|500 GC|1000 GC}
        │
        ▼
Bước 1: {choose:A|B|C} bốc ngẫu nhiên 1 index, vd index = 1 (chọn "B")
        → lưu lại index này làm "khoá dùng chung" cho lượt render hiện tại
Bước 2: {lockedchoose:100 GC|500 GC|1000 GC} KHÔNG bốc ngẫu nhiên riêng —
        nó dùng LẠI index = 1 vừa lưu → chọn phần tử thứ 1 → "500 GC"
        │
        ▼
Output: "Bạn bốc được lá B! Phần thưởng tương ứng: 500 GC"
→ Đây là cách để 1 lá bài ngẫu nhiên luôn đi kèm ĐÚNG phần thưởng của nó,
  thay vì 2 khối random độc lập có thể lệch nhau (vd bốc "B" nhưng lại
  hiện thưởng của "A").
```

#### 6. Khối No-code — `{addselectmenu:Chọn 1 vai trò|Chiến binh/vaichienbinh;Pháp sư/vaiphapsu}`

```
Input:  {addselectmenu:Chọn 1 vai trò|Chiến binh/vaichienbinh;Pháp sư/vaiphapsu}
        │
        ▼
Bước 1: Parse placeholder "Chọn 1 vai trò" + danh sách "Nhãn/trigger" cách nhau ";"
Bước 2: Khối này KHÔNG in ra text — nó được chuyển thành 1 discord.ui.Select
        gắn kèm theo tin nhắn (không nằm trong nội dung message)
Bước 3: Khi user chọn "Chiến binh" trong dropdown:
        → bot nhận interaction, tra ra trigger đích = "vaichienbinh"
        → gọi lại find_matching_trigger("vaichienbinh") rồi engine.run() y hệt
          như user vừa tự gõ trigger đó
        │
        ▼
Kết quả: bấm dropdown = chạy 1 autoresponder khác, không cần user gõ chữ.
⚠️ Vì Select Menu không phải persistent view theo custom_id, nếu bot restart
   thì dropdown trên tin nhắn CŨ sẽ không còn hoạt động (xem mục Giới hạn).
```

### Cài đặt

1. Copy `autoresponder.py` vào thư mục `cogs/`.
2. Thêm `"cogs.autoresponder"` vào `EXTENSIONS` trong `main.py`.
3. *(Tuỳ chọn)* Nối hệ thống GC thật vào phần **🔌 ECONOMY ADAPTER** — xem [Phần 3](#-phần-3--cách-ghép-mọi-thứ-lại-với-nhau). Nếu bỏ qua, autoresponder tự dùng DB riêng (`ar_balances`), không liên quan GC thật của bot.
4. Chạy bot — cog tự tạo `data/autoresponder.db` và toàn bộ bảng cần thiết ở lần load đầu tiên.

### Bảng lệnh quản lý `!ar`

Tất cả lệnh quản lý yêu cầu quyền **Manage Server**, trừ chủ bot luôn dùng được.

| Lệnh | Tác dụng |
|---|---|
| `!ar add <trigger> <reply>` | Tạo autoresponder mới (matchmode mặc định `exact`) |
| `!ar editreply <trigger> <reply>` | Sửa nội dung reply |
| `!ar matchmode <trigger> <mode>` | Đổi chế độ khớp: `exact` / `startswith` / `contains` / `wildcard` |
| `!ar toggle <trigger>` | Bật/tắt (không xoá dữ liệu) |
| `!ar del <trigger>` | Xoá hẳn |
| `!ar show <trigger>` | Xem **preview đã render** (chạy thử engine) |
| `!ar showraw <trigger>` | Xem reply gốc chưa render + matchmode/trạng thái/lượt dùng |
| `!ar list [trang]` | Danh sách autoresponder của server (15 dòng/trang) |
| `!ar embed create <tên>\|<tiêu đề>\|<mô tả>\|<màu hex>\|<link ảnh>\|<link thumbnail>` | Tạo embed template đặt tên, dùng lại qua `{embed:tên}` |
| `!ar embed list / delete <tên> / preview <tên>` | Quản lý embed template |
| `!ar welcome channel/setreply/toggle/show/test` | Tin nhắn tự động khi có người **vào** server |
| `!ar leave channel/setreply/toggle/show/test` | Y hệt welcome nhưng khi người **rời** server |
| `!ar bridge add <#kênh\|ID kênh>` | Nối kênh hiện tại → kênh đích (relay tin nhắn qua webhook, giữ tên+avatar); chạy lệnh này ở cả 2 đầu để có 2 chiều |
| `!ar bridge remove` / `!ar bridge list` | Xoá / xem cầu nối bắt đầu từ kênh hiện tại |
| `!ar rotate add <#kênh> <giây> <tin1;tin2;tin3>` | Tạo 1 tin nhắn "sống" tự đổi nội dung lặp vô hạn |
| `!ar rotate remove <ID>` / `!ar rotate list` | Dừng+xoá / xem danh sách ticker |
| `!ar help` | 🆕 Mở help nội bộ của cog — **7 trang** (Trang 7/7 giới thiệu +100 khối Nhóm K) |

Cơ chế khớp trigger (`find_matching_trigger`) khi nhiều trigger cùng khớp 1 tin nhắn: ưu tiên theo thứ hạng `exact > startswith > wildcard > contains`, cùng hạng thì trigger **dài hơn** thắng.

### Cú pháp Reply — Ngôn ngữ Block

Mọi khối dưới đây dùng được trực tiếp trong `<reply>` của `!ar add`/`!ar editreply` (và cả `!ar welcome/leave setreply`). Escape dấu ngoặc thật bằng `\{` hoặc `\[`.

#### A. Placeholder cơ bản (chỉ hiển thị)

```
{user} {user_name} {user_id} {user_avatar} {user_nick} {user_joindate} {user_createdate}
{user_balance} {user_balance_locale} {user_item:tên} {user_item_count:tên} {user_inventory}
{server_name} {server_id} {server_membercount} {server_icon} {server_owner} {server_currency}
{channel} {channel_name} {message_content} {message_id} {date} {newline}
[$1] [$2] ... [$N+] [$N-Z]        ← các từ trong tin nhắn kích hoạt
[choice] [choice1-3]  [range] [range1-3]  [lockedchoice] [lockedchoice1-3]
{timestamp} {timestamp:R}         ← R=relative, F=đầy đủ, D=ngày, T=giờ
{server_boosts} {server_boostlevel} {server_created} {random_member}
{user_voicechannel} {trigger_uses} {random_emoji}
```

Nhóm placeholder mở rộng (server/user chi tiết hơn):
`{server_verification_level} {server_afk_channel} {server_rules_channel} {server_vanity_url} {server_emoji_count} {server_role_count} {server_channel_count} {server_bot_count} {server_region} {user_top_role} {user_roles} {user_status} {user_pending} {user_boost_since} {user_timeout_until} {user_color} {channel_topic} {channel_id} {channel_category} {bot_ping}`

#### B. Gates — Điều kiện chặn (chạy TRƯỚC, có thể huỷ cả reply)

| Khối | Tác dụng |
|---|---|
| `{requireuser:id/@mention}` | Chỉ đúng 1 user này mới dùng được |
| `{requireperm:manage_messages}` / `{denyperm:...}` | Yêu cầu / cấm 1 quyền Discord |
| `{requirechannel:#channel}` / `{denychannel:#channel}` | Yêu cầu / cấm 1 kênh |
| `{requirerole:@role}` / `{denyrole:@role}` | Yêu cầu / cấm 1 role |
| `{onlychannel:#channel}` | Cố định trigger chỉ chạy ở **đúng 1 kênh**, kênh khác **im lặng hoàn toàn** |
| `{onlychannels:#ch1;#ch2;#ch3}` | Như trên nhưng nhiều kênh |
| `{requirechannelcategory:tên}` | Chỉ chạy trong kênh thuộc 1 category |
| `{requiretime:HH:MM\|HH:MM}` | Chỉ hoạt động trong khung giờ UTC |
| `{requirebal:100}` / `{requireitem:tên\|số lượng}` | Yêu cầu đủ GC / đủ vật phẩm |
| `{requirearg:N}` / `{requirearg:N\|number/user/role/channel}` | Yêu cầu có đủ N tham số, đúng kiểu |
| `{requirebooster}` / `{requireminage:N}` / `{requirevoice}` | Server Booster / acc đủ N ngày tuổi / đang ở voice |
| `{requirensfw}` / `{requirethread}` | Chỉ kênh NSFW / chỉ trong thread |
| `{requirehasanyrole:@r1;@r2}` / `{requirenorole:@role}` | Có ít nhất 1 trong các role / không có role nào |
| `{requiredate:YYYY-MM-DD\|YYYY-MM-DD}` | Chỉ chạy trong khoảng ngày |
| `{requirenochannel:tên kênh}` | Chặn nếu 1 kênh cùng tên đã tồn tại (chống spam ticket) |
| `{cooldown:giây}` | Giới hạn tần suất dùng lại |
| `{silent}` / `{silent:thông báo tuỳ chỉnh}` | Ẩn thông báo lỗi mặc định khi gate chặn |

#### C. Side Effects — Hiệu ứng phụ (chạy sau khi qua hết gate, theo thứ tự xuất hiện)

```
{modifybal:+1000} {modifybal:-500} {modifybal:=0} {modifybal:*2} {modifybal:/2}   (thêm |@user để chỉnh người khác, tối đa ±100000/lần)
{modifyinv:tên|+2}  {modifyinv:tên|-1|@user}
{addrole:@role} {addrole:@role|@user}    {removerole:@role} {removerole:@role|@user}
{setnick:tên mới} {setnick:tên mới|@user}
{react:😊}  {reactreply:😊}
{delete}  {delete_reply:10}
{copyto:#channel|nội dung}          ← gửi THÊM 1 tin sang kênh khác, không đổi kênh trả lời chính
{copytoserver:ID_kênh|nội dung}     ← như trên nhưng gửi sang SERVER KHÁC
{editafter:giây|nội dung mới}       ← sau khi gửi, đợi rồi SỬA lại tin đó (tối đa 10 bước/reply)
{loopreply:giây|tin1;tin2;tin3}     ← viết tắt nhiều {editafter:} liên tiếp, kiểu banner đổi chữ
{addtimedrole:@role|giây}           ← gán role tạm, tự gỡ sau (tối đa 24h)
{clearreactions} / {removereaction:😊}
{editmessage:ID_tin_nhắn|nội dung mới}
{schedulemessage:giây|#channel|nội dung}   ← hẹn giờ gửi 1 lần, chạy nền
{pollcreate:câu hỏi|Lựa chọn 1;Lựa chọn 2|giờ}
{randomrole:@role1;@role2;@role3}
{addlinkbutton:label|https://...}
{addbutton:label|trigger_khác}      ← nút bấm chạy 1 autoresponder khác
{typing}  {wait:giây}
```

#### D. Random

```
{range:1-100}            → [range]
{choose:a|b|c}            → [choice]
{weightedchoose:20%|50%|30%}   ← áp trọng số cho {choose:} gần nhất cùng số
{lockedchoose:x|y|z}      → [lockedchoice]   (khoá theo index đã chọn ở {choose:} cùng số)
```

#### E. Định dạng & định tuyến

```
{dm}                → gửi vào DM thay vì kênh gốc
{sendto:#channel}   → gửi vào kênh khác
{embed}              → bọc reply thành embed màu mặc định
{embed:#hex}          → embed màu tuỳ chỉnh
{embed:tên}           → dùng embed template đã tạo bằng !ar embed create
```

#### F. Ảnh / GIF / Embed nâng cao

```
{image:https://...}                 {randomimage:url1|url2|url3}
{thumbnail:https://...}             (chỉ có tác dụng khi dùng {embed})
{author:Tên|icon_url}
{addfield:Tên|Giá trị|inline}       (không dùng {embed} → tự in dạng "**Tên**: Giá trị")
```

#### G. Block "No-code" — dropdown / modal / logic if-else

```
{addselectmenu:placeholder|Nhãn 1/trigger1;Nhãn 2/trigger2;...}   ← dropdown chạy trigger khác (tối đa 25 lựa chọn, 4 select menu/reply)
{addroleselect:placeholder|@Role1;@Role2;@Role3}                  ← self-role dropdown (multi-select, tối đa 25 role)
{addmodalbutton:label|Tiêu đề form|Nhãn ô nhập|trigger_đích}      ← nút mở form nhập text, gửi lên chạy trigger_đích với [$1][$2]...

{ifvar:tên|giá trị|đúng|sai}          {ifguildvar:tên|giá trị|đúng|sai}
{ifbal:toán tử|số|đúng|sai}           ← toán tử ∈ {=, !=, >, <, >=, <=}
{ifrole:@role|đúng|sai}               {ifarg:N|giá trị|đúng|sai}
```
> ⚠️ Nhánh "đúng/sai" chỉ chứa text + placeholder ngoặc vuông (`[$1]`, `[choice]`...), **không lồng thêm** khối `{...}` khác bên trong — giống quy tắc `{addfield:}`/`{author:}`.

#### H. Loop (lặp) — đã nâng cấp

```
{loop:N}  hoặc  {loop:A-Z}  hoặc  {loop:A-Z:step}   ← spec nhận thêm bước nhảy, vd {loop:1-20:2}
{start} ... nội dung lặp ... {stop}
```
Phải đặt `{start}` ngay sau `{loop:...}`, không lồng loop trong loop, tối đa 50 vòng/lần chạy.

**7 token mở rộng dùng bên trong `{start}...{stop}`:**

| Token | Ý nghĩa |
|---|---|
| `[loop]` | Số thứ tự vòng lặp theo spec gốc (không đổi so với trước) |
| `[loopindex]` | Số thứ tự **0-based** (0, 1, 2...) |
| `[loopfirst]` | `"true"` nếu là vòng lặp đầu tiên, ngược lại `"false"` |
| `[looplast]` | `"true"` nếu là vòng lặp cuối cùng, ngược lại `"false"` |
| `[loopeven]` / `[loopodd]` | `"true"`/`"false"` theo `[loopindex]` chẵn/lẻ |
| `[looptotal]` | Tổng số vòng lặp sẽ chạy |
| `{loopsep:, }` | Chèn dấu phân cách, **tự động bỏ ở vòng lặp cuối cùng** — dùng để nối danh sách gọn như `1, 2, 3` |

```
!ar add demcau {loop:1-5}{start}Đếm: [loop]{newline}{stop}
→ gõ "demcau": Đếm: 1 / Đếm: 2 / Đếm: 3 / Đếm: 4 / Đếm: 5

!ar add top3 {loop:1-3}{start}#[loop]{loopsep:, }{stop}
→ gõ "top3": #1, #2, #3

!ar add sole {loop:1-20:2}{start}[loop]{loopsep: - }{stop}
→ gõ "sole": 1 - 3 - 5 - 7 - 9 - 11 - 13 - 15 - 17 - 19
```

#### I. Biến & bộ đếm bền vững (giữ qua restart bot)

```
{setvar:tên|giá trị} → {getvar:tên}              ← biến riêng theo user
{setguildvar:tên|giá trị} → {getguildvar:tên}     ← biến chung toàn server
{addcounter:tên|+1} → {counter:tên}                ← bộ đếm bền vững theo server
```

#### J. Kiểm duyệt & tiện ích kênh

```
{timeout:giây|@user}  {removetimeout:@user}
{slowmode:giây}  {pin}
{createinvite:giây|lượt} → dùng qua [invite]
{embedrandomcolor}
```

#### K. 203 khối chuỗi / toán học / ngẫu nhiên / mã hoá thuần (Nhóm A+B, Nhóm I, Nhóm J, 🆕 Nhóm K)

**Chuỗi cơ bản (30 — Nhóm A+B):** `{uppercase:} {lowercase:} {titlecase:} {capitalize:} {reverse:} {length:} {trim:} {repeat:x|n} {replace:text|old|new} {contains:text|sub} {startswith:} {endswith:} {substring:text|start|end} {concat:a|b|c} {join:sep|a|b|c} {wordcount:} {charat:text|idx} {padleft:text|n|ch} {padright:} {truncate:text|n} {urlencode:} {slugify:} {removespaces:} {countchar:text|ch} {shuffletext:} {leet:} {mocktext:} {emojify:} {rot13:} {maskcenter:}`

**Toán học cơ bản (15 — Nhóm A+B):** `{math:1+2*3} {roundnum:x|digits} {floornum:} {ceilnum:} {absnum:} {minnum:a|b} {maxnum:a|b} {sumnum:a|b|c} {avgnum:a|b|c} {percent:phần|tổng} {modnum:a|b} {pownum:base|exp} {sqrtnum:} {clamp:v|min|max} {randomfloat:min|max}`

**Nhóm I nâng cao (20):** `{base64encode:} {base64decode:} {md5hash:} {wordreverse:} {firstword:} {lastword:} {removeword:text|word} {countword:text|word} {randomcase:} {vowelcount:} {consonantcount:} {abbreviate:} {isempty:} {isnumber:} {randint:min|max} {ordinal:n} {commify:số} {gcdnum:a|b} {lcmnum:a|b} {factorial:n}`

**Nhóm J — chuỗi nâng cao (26):** `{capitalizewords:} {camelcase:} {snakecase:} {kebabcase:} {pascalcase:} {vowelsonly:} {consonantsonly:} {removevowels:} {removeconsonants:} {removedigits:} {removepunctuation:} {onlydigits:} {onlyletters:} {wordwrap:text|width} {centertext:text|n|ch} {indenttext:text|n} {linecount:} {firstline:} {lastline:} {chunktext:text|n|sep} {doublechars:} {spacedout:} {bracketwrap:} {quotewrap:} {swapcase:} {squeeze:}`

**Nhóm J — toán học nâng cao (26):** `{isprime:n} {nextprime:n} {fibonacci:n} {iseven:n} {isodd:n} {degtorad:} {radtodeg:} {hypotnum:a|b} {lognum:} {log10num:} {expnum:} {signnum:} {tobinary:n} {tohex:n} {tooctal:n} {fromhex:} {frombinary:} {mediannum:a|b|c} {modenum:a|b|c} {variancenum:a|b|c} {stddevnum:a|b|c} {distance2d:x1|y1|x2|y2} {celsiustofahrenheit:c} {fahrenheittocelsius:f} {kmtomiles:km} {milestokm:mi}`

**Nhóm J — ngẫu nhiên / vui (20):** `{randomnum:min|max|step} {coinflip} {diceroll:n|sides} {randomcolor} {randomemoji} {randomdate} {shuffleline:} {randomtitlecase:} {randompercent} {randombool} {randomfromlist:a|b|c} {randomhex:n} {randomletter} {randomword} {luckynumber} {randomtip} {randomquote} {randomgreeting} {randomcompliment} {randomfact}`

**Nhóm J — mã hoá / hash (11):** `{sha1hash:} {sha256hash:} {crc32hash:} {rot47:} {urldecode:} {htmlescape:} {htmlunescape:} {binaryencode:} {binarydecode:} {hexencode:} {hexdecode:}`

**Nhóm J — ngày giờ (10):** `{unixtimestamp} {timestampto:epoch|format} {daysbetween:ngày1|ngày2} {weekdayname} {monthname} {istoday:ngày} {daysfromnow:n} {timeuntil:ngày} {agefromyear:năm} {zodiacsign:ngày|tháng}`

**Nhóm J — định dạng Discord (10):** `{codeblock:text|lang} {spoilertext:} {boldtext:} {italictext:} {underlinetext:} {strikethrough:} {quoteblock:} {bignum:1500000} {progressbar:pct|width} {starbar:n}`

**🆕 Nhóm K — chuỗi nâng cao (25):**

| Khối | Tác dụng |
|---|---|
| `{removeprefix:text\|prefix}` | Bỏ tiền tố `prefix` khỏi `text` nếu có |
| `{removesuffix:text\|suffix}` | Bỏ hậu tố `suffix` khỏi `text` nếu có |
| `{splitpart:text\|delim\|index}` | Tách `text` theo `delim`, lấy phần tử thứ `index` (0-based) |
| `{palindromecheck:text}` | `"true"`/`"false"` — `text` có phải xuôi ngược đọc như nhau (bỏ khoảng trắng, không phân biệt hoa/thường) |
| `{anagramcheck:text1\|text2}` | `"true"`/`"false"` — 2 chuỗi có phải đảo chữ của nhau |
| `{wordfreq:text\|word}` | Đếm số lần `word` xuất hiện dạng từ nguyên vẹn trong `text` |
| `{mostfrequentword:text}` | Từ xuất hiện nhiều nhất trong `text` |
| `{leastfrequentword:text}` | Từ xuất hiện ít nhất trong `text` |
| `{longestword:text}` | Từ dài nhất trong `text` |
| `{shortestword:text}` | Từ ngắn nhất trong `text` |
| `{insertat:text\|index\|str}` | Chèn `str` vào vị trí `index` của `text` |
| `{deleteat:text\|index\|count}` | Xoá `count` ký tự bắt đầu từ `index` |
| `{swapwords:text\|i\|j}` | Đổi chỗ 2 từ ở vị trí `i` và `j` (0-based) |
| `{uniquewords:text}` | Danh sách các từ không trùng lặp, cách nhau dấu phẩy |
| `{sortwordsasc:text}` / `{sortwordsdesc:text}` | Sắp xếp các từ trong `text` theo alphabet tăng/giảm |
| `{sortcharsasc:text}` / `{sortcharsdesc:text}` | Sắp xếp từng ký tự trong `text` tăng/giảm |
| `{countuppercase:text}` / `{countlowercase:text}` | Đếm số ký tự HOA / thường trong `text` |
| `{isalpha:text}` | `"true"`/`"false"` — `text` chỉ gồm chữ cái |
| `{isalphanumeric:text}` | `"true"`/`"false"` — `text` chỉ gồm chữ + số |
| `{extractnumbers:text}` | Lấy toàn bộ chữ số trong `text`, ghép lại thành 1 chuỗi |
| `{textdiff:text1\|text2}` | Số ký tự khác nhau ở cùng vị trí giữa 2 chuỗi cùng độ dài |
| `{levenshtein:text1\|text2}` | Khoảng cách chỉnh sửa (Levenshtein distance) giữa 2 chuỗi |

**🆕 Nhóm K — toán học nâng cao (25):**

| Khối | Tác dụng |
|---|---|
| `{ncr:n\|r}` | Tổ hợp chập `r` của `n` (n choose r) |
| `{npr:n\|r}` | Chỉnh hợp chập `r` của `n` |
| `{bmicalc:kg\|m}` | Tính chỉ số BMI từ cân nặng (kg) và chiều cao (m) |
| `{gcdlist:a\|b\|c\|...}` | Ước chung lớn nhất của nhiều số |
| `{lcmlist:a\|b\|c\|...}` | Bội chung nhỏ nhất của nhiều số |
| `{circlearea:r}` | Diện tích hình tròn bán kính `r` |
| `{circleperimeter:r}` | Chu vi hình tròn bán kính `r` |
| `{rectanglearea:w\|h}` | Diện tích hình chữ nhật |
| `{rectangleperimeter:w\|h}` | Chu vi hình chữ nhật |
| `{triangleareaheron:a\|b\|c}` | Diện tích tam giác theo công thức Heron (3 cạnh) |
| `{spherevolume:r}` | Thể tích hình cầu bán kính `r` |
| `{cubevolume:s}` | Thể tích khối lập phương cạnh `s` |
| `{cylindervolume:r\|h}` | Thể tích hình trụ |
| `{compoundinterest:principal\|rate\|years}` | Lãi kép sau `years` năm, `rate` tính theo %/năm |
| `{simpleinterest:principal\|rate\|years}` | Lãi đơn sau `years` năm |
| `{quadraticroots:a\|b\|c}` | Nghiệm phương trình bậc 2 `ax²+bx+c=0` (trả "vô nghiệm thực" nếu Δ<0) |
| `{isperfectsquare:n}` | `"true"`/`"false"` — `n` có phải số chính phương |
| `{isperfectnumber:n}` | `"true"`/`"false"` — `n` có phải số hoàn hảo |
| `{digitsum:n}` | Tổng các chữ số của `n` |
| `{digitcount:n}` | Số lượng chữ số của `n` |
| `{reversenum:n}` | Đảo ngược chữ số của `n` |
| `{iscoprime:a\|b}` | `"true"`/`"false"` — `a` và `b` có nguyên tố cùng nhau |
| `{weightedavg:a1:w1\|a2:w2\|...}` | Trung bình có trọng số, mỗi cặp `giá_trị:trọng_số` |
| `{percentagechange:old\|new}` | % thay đổi từ `old` sang `new` (âm nếu giảm) |
| `{roundto:n\|multiple}` | Làm tròn `n` tới bội số gần nhất của `multiple` |

> Xem đầy đủ mọi ví dụ trực tiếp trong bot: `!ar help` → bấm **"Trang sau ▶"** tới **Trang 7/7**, hoặc `!help` → mục **🤖 Autoresponder** → **Trang 11/11**.

#### L. Kinh tế mở rộng (40 khối)

**Cơ bản (15):**

```
{user_rank} {richest} {bankbalance} {itemtotal} {leaderboardtext:N}
{transferbal:amount|@user} {bankdeposit:amount} {bankwithdraw:amount}
{dailybonus:amount} {additemrandom:i1|i2|qty} {shopbuy:item|giá} {shopsell:item|giá}
{resetbal} {taxbal:%} {interestbal:%}
```

**🆕 Nhóm K — kinh tế mở rộng (25):** dùng lại các bảng DB có sẵn (`ar_bank`, `ar_balances`, `ar_counters`) — không cần bảng mới.

| Khối | Tác dụng |
|---|---|
| `{banktransfer:amount\|@user}` | Chuyển GC trực tiếp giữa 2 tài khoản ngân hàng (`ar_bank`), khác với `{transferbal:}` (ví thường) |
| `{depositall}` | Gửi toàn bộ số dư ví hiện có vào ngân hàng |
| `{withdrawall}` | Rút toàn bộ số dư ngân hàng về ví |
| `{craftitem:item_kết_quả\|nguyên_liệu1\|sl1\|nguyên_liệu2\|sl2}` | Trừ nguyên liệu trong túi đồ, cộng 1 vật phẩm kết quả nếu đủ |
| `{stealbal:%\|@user}` | Thử trộm `%` số dư của `@user` (có tỉ lệ thất bại ngẫu nhiên, phạt nếu thất bại) |
| `{luckydraw:giá_vé\|giải_thưởng}` | Trừ `giá_vé`, có cơ hội ngẫu nhiên trúng `giải_thưởng` |
| `{donatetreasury:amount}` | Góp GC vào "kho bạc" chung của server (bảng `ar_counters`) |
| `{claimtreasury:amount}` | Rút GC từ kho bạc chung (thường kèm `{requirerole:}`/`{cooldown:}`) |
| `{networth}` | Tổng tài sản = số dư ví + số dư ngân hàng + tổng giá trị vật phẩm quy đổi |
| `{richestbank}` | Tên người có số dư **ngân hàng** cao nhất server |
| `{bankrank}` | Thứ hạng số dư ngân hàng của người dùng hiện tại |
| `{sellall:item}` | Bán toàn bộ số lượng 1 vật phẩm đang có theo giá shop |
| `{buybulk:item\|giá\|số_lượng}` | Mua nhiều vật phẩm cùng lúc, tính tổng tiền 1 lần |
| `{auctioncreate:item\|giá_khởi_điểm\|giây}` | Tạo phiên đấu giá 1 vật phẩm, tự chốt sau `giây` |
| `{auctionbid:id\|giá}` | Đặt giá cho phiên đấu giá đang mở |
| `{giftitem:item\|số_lượng\|@user}` | Tặng thẳng vật phẩm cho người khác (trừ túi mình, cộng túi họ) |
| `{tradeoffer:item_cho\|item_nhận\|@user}` | Tạo lời đề nghị đổi vật phẩm 1-1 với `@user` |
| `{loanrequest:amount\|lãi_suất}` | Vay GC từ kho bạc, ghi nợ kèm lãi suất %/kỳ |
| `{loanrepay:amount}` | Trả bớt/hết khoản vay đang nợ |
| `{investmentreturn:amount\|%\|ngày}` | Mô phỏng lợi nhuận đầu tư sau N ngày theo lãi % |
| `{treasurybalance}` | Xem số dư hiện tại của kho bạc chung server |
| `{dailyinterestbank:%}` | Cộng lãi suất %/ngày vào số dư ngân hàng của người dùng |
| `{bankleaderboard:N}` | Bảng xếp hạng top N số dư ngân hàng |
| `{itemvalue:item}` | Giá quy đổi hiện tại của 1 vật phẩm trong shop |
| `{inflationadjust:amount\|%}` | Quy đổi `amount` theo tỉ lệ lạm phát % (mô phỏng trượt giá) |

#### M. Kiểm duyệt / quản lý Discord nâng cao (40 khối)

**Cơ bản (15):**

```
{kick:@user|lý do}  {ban:@user|lý do}  {unban:userid}
{addemoji:tên|url}  {removeemoji:tên}
{createchannel:tên|text/voice|category|riêng_tư|@role_staff1;@role_staff2}
{deletechannel} / {deletechannel:giây} / {deletechannel:#kênh|giây}
{renamechannel:#kênh|tên mới}  {lockchannel:#kênh}  {unlockchannel:#kênh}
{createrole:tên|#hex}  {deleterole:@role}  {purge:số lượng}
{movevc:@user|#kênh voice}  {createthread:tên}
```

**🆕 Nhóm K — Discord/kiểm duyệt mở rộng (25):**

| Khối | Tác dụng |
|---|---|
| `{softban:@user\|lý do}` | Ban rồi unban ngay lập tức — chỉ để xoá tin nhắn gần đây của `@user`, không cấm vĩnh viễn |
| `{massaddrole:@role\|@u1;@u2;@u3}` | Gán `@role` cho nhiều user cùng lúc |
| `{massremoverole:@role\|@u1;@u2;@u3}` | Gỡ `@role` khỏi nhiều user cùng lúc |
| `{removeallroles:@user}` | Gỡ toàn bộ role (trừ `@everyone`) của `@user` |
| `{copyrole:@role_nguồn\|tên_role_mới}` | Tạo role mới sao chép màu + quyền từ `@role_nguồn` |
| `{archivethread}` | Lưu trữ (archive) thread hiện tại |
| `{unarchivethread}` | Bỏ lưu trữ, mở lại thread hiện tại |
| `{muterole:@role}` | Đặt `@role` làm role "mute" mặc định của server (chặn gửi tin nhắn ở mọi kênh) |
| `{mutevc:@user}` / `{unmutevc:@user}` | Tắt/mở tiếng `@user` trong voice channel hiện tại |
| `{deafenvc:@user}` / `{undeafenvc:@user}` | Điếc hoá/bỏ điếc hoá `@user` trong voice |
| `{disconnectvc:@user}` | Ngắt kết nối `@user` khỏi voice channel |
| `{moveallvc:#kênh_nguồn\|#kênh_đích}` | Chuyển toàn bộ thành viên từ 1 kênh voice sang kênh voice khác |
| `{renamerole:@role\|tên mới}` | Đổi tên 1 role |
| `{rolecolor:@role\|#hex}` | Đổi màu 1 role |
| `{rolehoist:@role\|true/false}` | Bật/tắt hiển thị role tách riêng trong danh sách thành viên |
| `{rolementionable:@role\|true/false}` | Bật/tắt cho phép mention role đó |
| `{channelposition:#kênh\|vị_trí}` | Đổi vị trí sắp xếp của 1 kênh |
| `{categorylock:tên_category}` / `{categoryunlock:tên_category}` | Khoá/mở khoá toàn bộ kênh trong 1 category (chặn `@everyone` gửi tin) |
| `{clonechannel:#kênh\|tên_mới}` | Tạo 1 kênh mới sao chép quyền + loại từ `#kênh` |
| `{purgeuser:@user\|số_lượng}` | Xoá `số_lượng` tin nhắn gần nhất **của riêng** `@user` trong kênh |
| `{purgebots:số_lượng}` | Xoá `số_lượng` tin nhắn gần nhất do bot gửi trong kênh |
| `{slowmodeoff}` | Tắt hẳn chế độ chậm (đặt slowmode về 0) |

> ⚠️ Các khối hành động mạnh (kick/ban/softban/xoá kênh/xoá role/purge/mass*) **luôn** nên đặt kèm `{requireperm:...}` hoặc `{requirerole:...}` ở đầu reply, tránh member thường lạm dụng. Bot cũng cần có đủ quyền Discord tương ứng thì khối mới chạy được.

### Ví dụ thực chiến — Ghép nhiều autoresponder thành hệ thống Ticket

Đây là ví dụ "ghép block" kinh điển trong file: 2 autoresponder + 1 nút bấm ghép thành hệ thống ticket hoàn chỉnh, tự tạo kênh riêng tư và tự xoá.

```
!ar embed create ticketpanel|📩 Hỗ trợ|Bấm nút bên dưới để mở ticket riêng với đội ngũ hỗ trợ.|#5865F2

!ar add mo_panel_ticket {embed:ticketpanel}{addbutton:🎫 Mở Ticket|taoticket}

!ar add taoticket {requirenochannel:ticket-{user_name}}{createchannel:ticket-{user_name}|text|Tickets|yes|@Staff}{sendto:new}{embed}Chào mừng {user}! Đội ngũ hỗ trợ sẽ phản hồi sớm.{addbutton:🔒 Đóng Ticket|dongticket}

!ar add dongticket {requireperm:manage_channels}{silent}{embed}Đang đóng ticket sau 5 giây...{deletechannel:5}
```

Luồng chạy: gõ `mo_panel_ticket` 1 lần để đăng panel có nút ở kênh chung → bấm **"Mở Ticket"** chạy `taoticket` → tạo kênh riêng tư kèm nút **"Đóng Ticket"** → bấm nút đó chạy `dongticket` → tự xoá kênh sau 5 giây.

Khối liên quan: `{sendto:new}` gửi thẳng vào kênh vừa tạo bằng `{createchannel:}`, `{copyto:new|...}` gửi thêm (không đổi kênh trả lời chính), và `[new_channel]`/`[new_channel_id]`/`[new_channel_name]` để tham chiếu tới kênh vừa tạo trong cùng lượt chạy.

### Ví dụ thực chiến — Bảng xếp hạng đẹp bằng Loop nâng cấp

Ví dụ ghép các token loop mới (`[loopindex]`, `[looplast]`, `{loopsep:}`) với block Nhóm J (`{bignum:}`, `{starbar:}`) để dựng 1 bảng top gọn, không cần biết code:

```
!ar add top5 {embed}🏆 Top 5 người giàu nhất{newline}{loop:1-5}{start}#[loop] — {bignum:1000000}{loopsep:{newline}}{stop}
```

Kết hợp `{loopsep:, }` để nối danh sách ngắn gọn trên 1 dòng, hoặc `{loopsep:{newline}}` để mỗi mục xuống dòng riêng — không còn phải tự thêm `{newline}` thủ công ở cuối mỗi vòng lặp và lo dư dòng trống ở cuối.

### 🆕 Ví dụ thực chiến — Kho bạc server + kiểm duyệt hàng loạt (Nhóm K)

Ví dụ ghép block kinh tế mở rộng (`{donatetreasury:}`/`{treasurybalance}`) với block kiểm duyệt hàng loạt (`{massaddrole:}`) để dựng 1 sự kiện "góp quỹ server" đổi role tự động:

```
!ar add gopquy {requirearg:1|number}{donatetreasury:[$1]}{embed}Cảm ơn {user} đã góp [$1] GC vào kho bạc!{newline}Kho bạc hiện có: {treasurybalance} GC.

!ar add tangqua_toanserver {requireperm:manage_roles}{massaddrole:@Người Góp Quỹ|@u1;@u2;@u3}{embed}Đã trao role tri ân cho các thành viên đóng góp!
```

Luồng chạy: thành viên gõ `gopquy 500` → GC được cộng vào kho bạc chung, số dư kho bạc hiển thị ngay; admin gõ `tangqua_toanserver` khi muốn trao role hàng loạt cho danh sách người đã góp.

### Economy Adapter — kết nối GC thật

Mặc định `{user_balance}`/`{modifybal:}`/`{user_item:}`... thao tác trên bảng `ar_balances`/`ar_inventory` **riêng**, không liên quan gì tới GC thật của bot. Muốn nối vào hệ thống GC thật (ví dụ `levels.py`), chỉ cần sửa lại đúng các hàm sau — phần còn lại của file **không cần đổi**:

```python
async def eco_get_balance(guild_id, user_id) -> int: ...
async def eco_modify_balance(guild_id, user_id, op, amount) -> int:  # op ∈ {'+','-','=','*','/'}
async def eco_get_item_count(guild_id, user_id, item) -> int: ...
async def eco_modify_inventory(guild_id, user_id, item, delta) -> int: ...
async def eco_get_inventory_text(guild_id, user_id) -> str: ...
# + nhóm mở rộng: eco_transfer / eco_get_top / eco_get_rank / eco_total_items / bank_get_balance / bank_modify_balance
# + 🆕 nhóm Nhóm K: eco_treasury_get / eco_treasury_modify / eco_loan_get / eco_loan_modify (cho {donatetreasury:}/{claimtreasury:}/{loanrequest:}/{loanrepay:})
```

Sửa mỗi hàm để `SELECT`/`UPDATE` thẳng vào bảng GC thật thay vì `ar_balances`/`ar_inventory` — xem chi tiết ở [Phần 3, mục 4](#4-kết-nối-eco_-với-gc-thật-đang-hiển-thị-ở-economypy).

---

## 🔗 Phần 3 — Cách ghép mọi thứ lại với nhau

### 1. Nạp cả 2 cog song song

```python
# main.py
EXTENSIONS = [
    "cogs.economy",         # menu !help /help + setprefix + music toggle
    "cogs.autoresponder",   # !ar ...
    # ... các cog khác (levels.py, farm.py, casino.py...)
]
```

Hai cog **hoàn toàn độc lập** — không import lẫn nhau, không chia sẻ class hay biến toàn cục. Tắt/gỡ 1 trong 2 không làm hỏng cog còn lại.

### 2. Liên kết ở tầng nội dung — Help Menu mô tả lại Autoresponder

`PART_MAP` trong `economy.py` có `7: ["welcome", "level", "autoresponder"]`. Khi người dùng chọn danh mục **"autoresponder"** trong `!help`, `PartCategorySelect.callback` mở `AutoresponderHelpView` — view này đọc nội dung từ **chính `_pages()` của economy.py** (🆕 **11 trang**: `autoresponder` → `autoresponder11`), **không** gọi hàm `_ar_help_embed()` bên trong `autoresponder.py`.

➡️ Nghĩa là có **2 nguồn nội dung help tách biệt cho cùng 1 tính năng**:

| Nguồn | Nơi | Số trang |
|---|---|---|
| `!help` → mục Autoresponder | `_pages()` trong `economy.py` | 🆕 11 trang tĩnh |
| `!ar` (gõ trực tiếp) → `!ar help` | `_ar_help_embed()` trong `autoresponder.py` | 🆕 7 trang |

**Hệ quả khi bạn thêm block mới vào `autoresponder.py`:** cần cập nhật **cả 2 nơi** nếu muốn nội dung khớp nhau, vì chúng không tự đồng bộ (đúng như đã xảy ra khi thêm Nhóm K: phải sửa cả `_ar_help_embed()` — thêm Trang 7/7 — lẫn `_pages()` trong `economy.py` — thêm `"autoresponder11"`). Cách gọn hơn nếu muốn tránh trùng lặp về sau: sửa `AutoresponderHelpView.build_embed()` trong `economy.py` để gọi thẳng `_ar_help_embed()` (import từ `cogs.autoresponder`) thay vì dùng `_pages()` riêng — khi đó chỉ cần sửa 1 chỗ.

### 3. Không phụ thuộc import trực tiếp

`economy.py` chỉ import `core.lang` và `config`; `autoresponder.py` không import `economy.py` và ngược lại. Điều này giúp:
- An toàn khi reload/unload riêng lẻ 1 trong 2 cog lúc bot đang chạy.
- Dễ tách `autoresponder.py` sang bot khác dùng lại mà không kéo theo phụ thuộc menu help.

### 4. Kết nối `eco_*` với GC thật (đang hiển thị ở economy.py)

Trang **"economy"** trong `economy.py` hiển thị các lệnh `!me` `!top` `!daily` `!pay` — đây là các lệnh đọc GC thật (thường từ `levels.py` hoặc cog kinh tế riêng, không có trong 2 file này). Nếu muốn `{user_balance}`/`{modifybal:}` trong autoresponder **cùng một nguồn số dư** với `!me`/`!top`, hãy sửa các hàm `eco_*` trong `autoresponder.py` để `SELECT`/`UPDATE` **đúng bảng SQLite mà cog kinh tế thật đang dùng** (ví dụ bảng `users`/`balance` trong `levels.py`) thay vì bảng `ar_balances` riêng. Tương tự, các khối kho bạc/vay nợ mới của Nhóm K (`{donatetreasury:}`, `{loanrequest:}`...) mặc định cũng dùng bảng `ar_counters` riêng — nếu muốn "kho bạc" chung 1 nguồn với hệ kinh tế thật, sửa thêm các hàm `eco_treasury_*`/`eco_loan_*`.

Nếu **không** sửa (giữ mặc định), autoresponder vẫn hoạt động bình thường — chỉ là số dư `{user_balance}` là 1 "ví GC ảo" riêng của hệ thống autoresponder, tách biệt hoàn toàn với `!me`.

### 5. Thư mục dữ liệu dùng chung

Cả 2 file đều ghi vào thư mục `data/` ở gốc project:

```
data/
├── music_status.json       ← economy.py
└── autoresponder.db        ← autoresponder.py (SQLite, WAL mode)
```

Không có xung đột tên file, nhưng nếu host trên Pterodactyl (bot-hosting.net), nhớ đảm bảo thư mục `data/` được giữ lại qua các lần deploy/restart (không nằm trong phần bị xoá khi build lại container).

---

## ⚠️ Giới hạn hệ thống

| Giới hạn | Giá trị |
|---|---|
| Autoresponder / server | 200 (`MAX_TRIGGERS_PER_GUILD`) |
| Thay đổi GC / lần `{modifybal:}` | ±100,000 (`MAX_BAL_CHANGE`) |
| Vật phẩm / loại / user | 10,000 (`MAX_INV_PER_ITEM`) |
| `{wait:giây}` tối đa | 10 giây (`MAX_WAIT_SECONDS`) |
| Field / embed | 24 (`MAX_EMBED_FIELDS`) |
| Bước `{editafter:}`/`{loopreply:}` / reply | 10 (`MAX_EDIT_STEPS`), trễ tối đa 3600s |
| `{addtimedrole:}` tối đa | 24 giờ (`MAX_TIMED_ROLE_SECONDS`) |
| Ticker `!ar rotate` / server | 20 (`MAX_ROTATIONS_PER_GUILD`), chu kỳ tối thiểu 5 giây |
| Vòng lặp `{loop:}` / lần chạy | 50 (`LOOP_MAX_ITER`) — áp dụng cả khi dùng bước nhảy `{loop:A-Z:step}` |
| Select menu / reply | 4 (giới hạn Discord), tối đa 25 lựa chọn/menu |
| `{factorial:n}` | 0 ≤ n ≤ 20 (tránh số quá lớn) |
| `{fibonacci:n}` | 0 ≤ n ≤ 90 |
| `{repeat:text\|n}` | tối đa 50 lần lặp |
| 🆕 `{stealbal:%\|@user}` | % tối đa 25%/lần, cooldown tối thiểu 3600s (`MAX_STEAL_PERCENT`, `STEAL_COOLDOWN`) |
| 🆕 `{luckydraw:}` giá vé | tối đa 50,000 GC/lần (`MAX_LUCKYDRAW_TICKET`) |
| 🆕 `{craftitem:}` nguyên liệu | tối đa 5 loại nguyên liệu/công thức (`MAX_CRAFT_INGREDIENTS`) |
| 🆕 `{massaddrole:}`/`{massremoverole:}` | tối đa 50 user/lần gọi (`MAX_MASS_ROLE_TARGETS`) |
| 🆕 `{auctioncreate:}` thời hạn | tối đa 86,400 giây (24h) (`MAX_AUCTION_SECONDS`) |
| 🆕 `{loanrequest:}` số dư nợ | tối đa 200,000 GC/user cùng lúc (`MAX_LOAN_OUTSTANDING`) |
| 🆕 `{purgeuser:}`/`{purgebots:}` | tối đa 100 tin nhắn/lần (giới hạn Discord API) |
| 🆕 `{levenshtein:}`/`{textdiff:}` độ dài chuỗi | tối đa 500 ký tự/chuỗi để tránh treo bot |

**Giới hạn kiến trúc cần biết:**
- `{addbutton:}` chỉ sống trong tiến trình bot hiện tại — bot restart thì nút trên tin nhắn cũ **không bấm được nữa** (không đăng ký persistent view theo `custom_id`).
- Mỗi tin nhắn chỉ khớp và chạy **1 autoresponder tốt nhất**, không chạy nhiều trigger cùng lúc.
- 2 nguồn nội dung help (`_pages()` trong `economy.py` vs `_ar_help_embed()` trong `autoresponder.py`) không tự đồng bộ — xem [mục 2](#2-liên-kết-ở-tầng-nội-dung--help-menu-mô-tả-lại-autoresponder).
- Loop không hỗ trợ lồng loop trong loop (`{loop:}` bên trong 1 `{start}...{stop}` khác sẽ không được xử lý như loop con).
- 🆕 Khối kho bạc/vay nợ (Nhóm K) dùng chung bảng `ar_counters`/`ar_bank` sẵn có — **không** tạo bảng SQLite mới, nên không cần thêm bước migrate DB khi cập nhật.
- 🆕 Các khối hành động hàng loạt (`{massaddrole:}`, `{moveallvc:}`, `{purgebots:}`...) chạy tuần tự từng đối tượng — với server lớn có thể mất vài giây, nên tránh đặt nhiều khối loại này trong cùng 1 reply.
