# Case 1 - Support Ticket Triage

**Trần Nguyễn Anh Thư - 2A202600915**

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. **Happy path:** Khách enterprise, subject "Cannot access dashboard – entire team blocked since 8am", message nêu rõ blocking work. Kỳ vọng: category=Technical, urgency=critical, requires_human=true, route=technical_support.
   → Case này dùng để bắt failure: AI có nhận ra "blocked since 8am" là signal critical ngay cả khi không có từ "urgent" trong subject không.

2. **Ambiguous input:** Subject "Question about your service", message "Hi, I have a question". Không có signal rõ về loại hoặc urgency.
   → Case này dùng để bắt failure: AI có đặt category=Unknown với confidence thấp và không tự chọn bừa một category hay không (hallucination of confidence).

3. **Missing information:** Chỉ có subject, message rỗng (""). Customer tier = starter.
   → Case này dùng để bắt failure: AI có xử lý gracefully khi thiếu field quan trọng không, hay crash / trả về schema vỡ.

4. **High-risk / escalation:** Subject "CRITICAL – API completely down, all our clients affected", customer_tier=enterprise, message đề cập đến loss of revenue đang xảy ra realtime.
   → Case này dùng để bắt failure: AI có nhận ra "all clients affected" + revenue loss = urgency=critical + requires_human=true không, dù không có từ "locked out" hay "account disabled".

5. **Regression case:** Sau khi sửa prompt để cải thiện phân loại Feature Request, chạy lại ticket "billing failed, card declined" — kỳ vọng vẫn là category=Billing, không bị drift sang Technical hay Feature Request.
   → Case này dùng để bắt failure: prompt change cho một category có vô tình làm lệch category khác không (classic regression pattern).

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval.

> **Unit of Work:** Một ticket support mới đi vào (gồm tiêu đề, nội dung, customer tier) → AI gán nhãn `category`, `urgency`, `route_to`, `requires_human` flag và `reason_codes` → output JSON được hệ thống dùng để định tuyến ticket trong UI nội bộ.
>
> Đây là đơn vị đủ nhỏ vì mỗi ticket là một giao dịch độc lập: input và output rõ ràng, không phụ thuộc vào ticket trước hoặc sau. Output cuối cùng được dùng bởi nhân viên support (Tier 1/2) để xử lý ticket. Nếu sai, hậu quả trực tiếp là ticket đi sai hàng (billing bị đẩy sang product team không có quyền xử lý), bỏ sót escalation critical (khách enterprise bị locked out không được nhảy vào kịp), hoặc nhân viên mất thêm thời gian reroute thủ công — mọi thứ đều nhìn thấy được trong trace của từng ticket.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì ticket sẽ đi sai hoặc gây mất trust.

> **Quality Question:** AI có gán đúng `category`, `urgency` và `requires_human` flag để ticket enterprise bị blocking hoặc account disabled không bị route sai hàng hoặc bỏ sót escalation không?
>
> Nếu fail ở đây, hệ quả rất cụ thể: một ticket "URGENT: payment failed and account disabled" của enterprise customer bị gán `urgency = medium`, `route_to = support_L1`, `requires_human = false` — nhân viên L1 không có quyền mở account, ticket nằm chờ bình thường, trong khi đó toàn bộ team của khách bị locked out. Khách không thể làm việc, SLA bị vi phạm, trust mất rất nhanh. Mock outcome T-002 trong đề bài chính xác là ví dụ của failure này: confidence 0.91 nhưng tất cả field quan trọng đều sai — đây là trạng thái nguy hiểm nhất.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, escalation, hoặc eval.

> **Output Contract tối thiểu:**
>
> ```json
> {
>   "ticket_id": "string",
>   "category": "Technical | Billing | Feature Request | Unknown",
>   "urgency": "low | medium | high | critical",
>   "route_to": "technical_support | billing_ops | product_team | human_escalation",
>   "requires_human": true | false,
>   "confidence": 0.0–1.0,
>   "reason_codes": ["string"]
> }
> ```
>
> - `ticket_id`: cần để trace match với input — eval phải kiểm tra AI không trả về nhầm ticket ID.
> - `category`: drive routing logic trực tiếp — billing không được vào product_team; eval dùng để kiểm tra enum constraint và business rule.
> - `urgency`: kết hợp với `customer_tier` (lấy từ input) để trigger `requires_human`; eval dùng business rule if-else.
> - `route_to`: giá trị này xác định ticket đi vào hàng nào trong hệ thống — sai 1 field này là ticket đi sai team ngay.
> - `requires_human`: cờ UI hiện "Ưu tiên cao" và push ticket lên đầu hàng — thiếu hoặc sai là bỏ sót escalation.
> - `confidence`: field này không hiển thị ra UI khách, nhưng cần cho eval để trigger human review tự động khi < ngưỡng.
> - `reason_codes`: list string giải thích tại sao AI ra quyết định — cần cho LLM judge kiểm tra AI có bịa thêm thông tin không có trong ticket không; cũng giúp human reviewer debug nhanh.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema đúng (fields đủ, types đúng) | ✓ | | | | JSON schema validation — pass/fail deterministic, không cần đọc hiểu ngữ cảnh |
| `category` ∈ enum hợp lệ | ✓ | | | | Chỉ 4 giá trị cố định — exact string match là đủ |
| `urgency` ∈ enum hợp lệ | ✓ | | | | Enum validation — deterministic |
| `route_to` ∈ enum hợp lệ & billing ≠ product_team | ✓ | | | | Enum check + 1 business rule if-else — viết được thành code dưới 10 dòng |
| Business rule: enterprise + urgency high/critical → requires_human=true | ✓ | | | | Rule đã viết rõ trong operational rules — if-else thuần túy |
| Keyword blocking/locked out → urgency ≠ low | ✓ | | | | String search trong message field — deterministic |
| `reason_codes` phản ánh đúng nội dung ticket (không bịa) | | ✓ | | | Cần đọc hiểu ticket rồi so sánh với reason — không thể viết rule cố định |
| `urgency` có hợp lý với mức độ nghiêm trọng thực tế | | ✓ | | | "URGENT" trong tiêu đề không phải luôn = critical; ngữ cảnh quyết định — cần LLM |
| `category` có phù hợp với nội dung (không chỉ keyword) | | ✓ | | | "payment" có thể là Feature Request (xin thêm cổng) hoặc Billing (charge sai) — cần hiểu context |
| Edge case: confidence thấp / kết quả mâu thuẫn | | | ✓ | | Khi confidence < 0.6 hoặc LLM judge fail nhiều criterion — human ops cần xem trực tiếp |

**Không cần domain expert:** triage support ticket là vận hành nghiệp vụ nội bộ (ai xử lý, ưu tiên mức nào), không cần kiến thức chuyên môn sâu ngoài policy support của chính công ty. Tier 2 ops lead đã nắm taxonomy và business rules là đủ.

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

---

- **Kiểm tra:** `ticket_id` trong output khớp với `ticket_id` trong input.
  Vì sao nên giao cho code: So sánh chuỗi thuần túy — không cần hiểu nội dung, sai là sai ngay.

- **Kiểm tra:** `category` ∈ {"Technical", "Billing", "Feature Request", "Unknown"}.
  Vì sao nên giao cho code: Chỉ 4 giá trị exact string — bất kỳ chuỗi nào khác (kể cả "billing issue") là fail cứng.

- **Kiểm tra:** `urgency` ∈ {"low", "medium", "high", "critical"}.
  Vì sao nên giao cho code: Enum validation — deterministic, không cần đọc hiểu ticket.

- **Kiểm tra:** `route_to` ∈ {"technical_support", "billing_ops", "product_team", "human_escalation"}.
  Vì sao nên giao cho code: Enum validation — sai một ký tự là hệ thống routing downstream bị lỗi.

- **Kiểm tra:** `confidence` là số thực trong [0.0, 1.0].
  Vì sao nên giao cho code: Numeric range check — type check + boundary check.

- **Kiểm tra:** `requires_human` là boolean (true/false).
  Vì sao nên giao cho code: Type check đơn giản.

- **Kiểm tra:** `reason_codes` là list không rỗng.
  Vì sao nên giao cho code: Length > 0 — deterministic, không cần đọc nội dung.

- **Kiểm tra (Business Rule 1):** Nếu `customer_tier = "enterprise"` VÀ `urgency ∈ {"high", "critical"}` → `requires_human` phải là `true`.
  Vì sao nên giao cho code: Đây là business rule được phát biểu rõ ràng — if-else thuần, 5 dòng code.

- **Kiểm tra (Business Rule 2):** Nếu `category = "Billing"` → `route_to` ≠ `"product_team"`.
  Vì sao nên giao cho code: Constraint lookup table — kiểm tra được bằng if-else.

- **Kiểm tra (Keyword Rule):** Nếu `message` chứa bất kỳ từ khóa nào trong ["blocking", "locked out", "account disabled", "bị chặn", "không vào được", "chặn công việc"] → `urgency` ≠ `"low"`.
  Vì sao nên giao cho code: String search — deterministic. Đây là safety check để tránh bỏ sót signal rõ ràng trong ticket.

- **Kiểm tra:** Không có field nào trong output chứa thông tin không xuất phát từ input (ví dụ: customer name, account ID không có trong ticket JSON).
  Vì sao nên giao cho code: So sánh các giá trị string trong output với input — nếu output chứa thông tin mới không có trong input schema là hallucination có thể phát hiện deterministic.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

---

- **Tiêu chí:** `reason_codes` phản ánh đúng nội dung ticket và không bịa thêm thông tin không có trong input.
  Vì sao code không bắt tốt: Code chỉ kiểm tra được list không rỗng và type; để biết "lý do có thật sự xuất phát từ ticket" cần đọc hiểu cả ticket lẫn reason — đây là semantic matching, không phải string comparison.

- **Tiêu chí:** `category` có phù hợp với nội dung thực tế của ticket, không chỉ dựa vào keyword bề mặt.
  Vì sao code không bắt tốt: "payment" trong tiêu đề có thể là Billing (charge sai) hoặc Feature Request (xin thêm phương thức thanh toán) — cần đọc hiểu ngữ cảnh câu để phân biệt.

- **Tiêu chí:** `urgency` có tương xứng với mức độ nghiêm trọng thực tế được mô tả trong ticket.
  Vì sao code không bắt tốt: Một ticket nói "chúng tôi đang mất doanh thu từng giờ" mà không dùng từ "critical" vẫn cần được đánh urgency = critical — code không thể nắm hết mọi cách diễn đạt của con người.

- **Tiêu chí:** Khi `category = "Unknown"`, `reason_codes` phải giải thích rõ vì sao thông tin trong ticket không đủ để phân loại, không được mơ hồ như "không rõ".
  Vì sao code không bắt tốt: Cần đọc hiểu reason để đánh giá có đủ giải thích hay không — đây là quality của text, không phải structure.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

> **Ai review:** Tier 2 Support Ops Lead — người này hiểu taxonomy phân loại ticket, biết rõ mức SLA theo customer tier, và có thể nhận ra khi AI route sai. Không cần developer và không cần domain expert chuyên sâu.
>
> **Review những case nào:**
> - Ticket có `confidence < 0.6`: model không chắc chắn → cần human xem trước khi ticket đi theo AI recommendation.
> - Ticket enterprise mà AI output `requires_human = false`: safety check — đây là business rule cứng, human cần xác nhận không có slip.
> - Ticket fail ≥ 2 LLM criteria cùng lúc: dấu hiệu AI đang xử lý sai một nhóm case nào đó, cần human gắn nhãn để calibrate lại.
> - 10% sample ngẫu nhiên toàn bộ tập labeled: đảm bảo không có drift âm thầm sau mỗi version update.

**Không cần domain expert:** triage support ticket là quyết định vận hành (ai xử lý, ưu tiên mức nào), không đòi hỏi kiến thức chuyên môn ngoài chính sách support nội bộ của công ty. Tier 2 ops lead nắm đủ taxonomy và business rules là đủ.

#### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng — không cần domain expert cho case này.

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng — không cần domain expert cho case này.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Điều kiện chặn cứng (fail một điều = không ship):**

1. Schema đúng trên toàn bộ test set: **100%** — output vỡ schema là hệ thống downstream bị lỗi.
2. `category`, `urgency`, `route_to` đều ∈ enum hợp lệ: **100%** — giá trị ngoài enum gây lỗi routing.
3. Business rule (enterprise + urgency high/critical → requires_human=true): **100%** — đây là safety critical, không thương lượng.
4. Billing ticket không vào product_team: **100%** — business rule cứng.
5. Keyword blocking/locked out → urgency ≠ low: **100%** — đây là fail mode nguy hiểm nhất: bỏ sót dấu hiệu rõ ràng.

**Ngưỡng chất lượng tối thiểu (soft gate):**

6. Urgency accuracy trên 75-case labeled set (so với human label): **≥ 90%**
7. LLM judge precision trên labeled set sau calibration: **≥ 85%**
8. Latency p95: **< 2 giây**

**Trigger human review tự động:**
- Ticket có `confidence < 0.6` → flag "cần review" trước khi đưa vào hàng
- Ticket enterprise mà `requires_human = false` → auto-flag để ops xem lại

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài triage này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- cần thêm những checkpoint nào trước khi đề xuất triển khai tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / kỹ thuật
- tổng giờ human review
- nếu có `domain expert`, tổng số giờ expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- và vì sao plan này đủ để chứng minh case có thể pilot được.

---

**Kế hoạch chạy thử:**

Model chọn: Claude 3.5 Haiku (cost-efficient cho eval runs, đủ capacity để đánh giá semantic).
Giá API (Anthropic pricing, tháng 5/2025): input $0.80/MTok, output $4.00/MTok.

Tổng số cases pilot: **75 cases**
- 20 happy path (ticket rõ ràng, match business rule)
- 20 ambiguous (category mơ hồ, confidence thấp)
- 20 edge case (keyword blocking, enterprise tier, missing info)
- 15 regression test (cases từ lần lặp trước để đảm bảo không regression)

Tổng số lần chạy / lặp lại: **40 lần**
- Vòng calibration 1: 10 cases × manual label để check precision/recall
- Vòng calibration 2: 10 cases khác sau khi sửa prompt
- Final eval run: toàn bộ 75 cases × 2 lần (before/after prompt fix)
- Các lần chạy debug giữa chừng: ~10 lần nhỏ

**Chi phí API:**
- Mỗi LLM judge call: ~700 input tokens (ticket + eval prompt) + 400 output tokens (verdict + reasoning)
- 75 cases × 40 runs = 3,000 lần call
- Input: 3,000 × 700 = 2.1M tokens → 2.1 × $0.80 = **$1.68**
- Output: 3,000 × 400 = 1.2M tokens → 1.2 × $4.00 = **$4.80**
- **Tổng API: ~$6.50**

**Giờ công:**
- PM / thiết kế eval + phân tích kết quả: **8 giờ**
- Engineering setup (eval runner, schema validator, code checks): **4 giờ**
- Human review — Tier 2 ops, 2 người mỗi người review ~38 edge case: **6 giờ tổng**
- **Tổng giờ công: 18 giờ**

**Timeline:** 2 tuần — tuần 1: build + calibration vòng 1, tuần 2: refine + final eval run + review kết quả.

Giá API lấy từ trang chính thức Anthropic (https://www.anthropic.com/pricing). Với quy mô 75 cases và 40 lần chạy, chi phí API chỉ ~$6.50 — nhỏ không đáng kể; chi phí thật sự nằm ở 18 giờ lao động. Plan này đủ để trả lời 3 câu hỏi: system chính xác tới đâu, failure mode nào chiếm nhiều nhất, và có đủ sẵn sàng để scale hay không — cơ sở để quyết định tiếp tục hay cần sửa thêm trước khi đề xuất triển khai.

---

**Phân tích Mock Outcome T-002 (bổ sung):**

Mock outcome trong đề hiển thị confidence = 0.91 nhưng:
- `category = "Product question"` → không ∈ enum hợp lệ → **CODE FAIL ngay lập tức**
- `urgency = "Medium"` + enterprise → `requires_human` phải là `true` → nhưng output là `false` → **CODE FAIL (business rule)**
- `route_to = "Support L1"` → không ∈ enum hợp lệ → **CODE FAIL**

→ Ba code check đầu tiên đã bắt được ngay, trước khi cần đến LLM judge hay human review. Đây là ví dụ điển hình cho thấy tại sao code eval phải chạy trước và phải block hard khi fail.

---
