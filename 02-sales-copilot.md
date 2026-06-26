# Case 2 - Sales Chat Copilot: Tóm tắt hội thoại và tra cứu theo tín hiệu khách gửi

**Trần Nguyễn Anh Thư - 2A202600915**


## Mục tiêu

Case này đại diện cho một kiểu AI rất hay gặp ở thị trường Việt Nam:

- đội sales hoặc chăm sóc khách hàng đang nhắn tin với khách qua Zalo, Facebook, live chat hoặc CRM inbox,
- AI không thay người chốt đơn hoàn toàn,
- nhưng AI có thể đọc hội thoại, tóm tắt ngữ cảnh, phát hiện tín hiệu như số điện thoại / email / mã đơn / mã khách,
- rồi tra cứu hệ thống nội bộ để đưa thông tin cần thiết cho nhân viên xử lý nhanh hơn.

Điểm khó của case này là:

- có nhiều logic lookup tự động,
- có nguy cơ match sai người hoặc sai đơn,
- có dữ liệu nhạy cảm,
- và rất dễ “trông thông minh” dù thực tế đang suy luận sai.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

Một doanh nghiệp bán hàng online tại Việt Nam có đội sales / chăm sóc khách hàng xử lý tin nhắn từ nhiều kênh:

- Zalo OA,
- Facebook Messenger,
- web chat,
- CRM inbox nội bộ.

Khi khách nhắn tin, nhân viên thường phải làm nhiều việc thủ công:

- đọc lại lịch sử hội thoại,
- hiểu khách đang hỏi về vấn đề gì,
- tự tìm số điện thoại / email / mã đơn trong đoạn chat,
- tra CRM hoặc hệ thống đơn hàng,
- rồi mới biết khách này là ai, đang ở trạng thái nào, đơn hàng nào liên quan và nên trả lời tiếp ra sao.

Nhóm muốn thêm một **Sales Chat Copilot** để:

- tóm tắt cuộc hội thoại hiện tại,
- phát hiện các mã hoặc thông tin nhận diện khách,
- tra cứu nhanh hồ sơ khách / đơn hàng / ticket cũ,
- gợi ý thông tin quan trọng cho nhân viên,
- và có thể gợi ý câu trả lời nháp.

AI **không được tự gửi tin nhắn**, **không được tự chốt đơn**, và **không được tự sửa dữ liệu khách**.

---

## 2. Workflow logic tham khảo (ASCII)

```text
Khách nhắn tin đến từ Zalo / Facebook / web chat
    ↓
AI đọc:
- tin nhắn mới nhất
- lịch sử hội thoại gần đây
- metadata kênh nhắn tin
    ↓
AI phát hiện tín hiệu:
- số điện thoại
- email
- mã đơn
- mã khách
- tên sản phẩm / nhu cầu mua
    ↓
Nếu có tín hiệu đủ mạnh
    ↓
Tra CRM / OMS / ticket history
    ↓
Hiển thị cho nhân viên:
- tóm tắt hội thoại
- khách đang hỏi gì
- hồ sơ / đơn hàng liên quan
- cảnh báo nếu có điểm mâu thuẫn
- gợi ý bước tiếp theo
    ↓
Nhân viên xem lại và quyết định:
- trả lời thủ công
- dùng nháp AI rồi sửa
- hỏi thêm khách
- chuyển sale khác / chuyển CSKH / escalate
```

---

## 3. Khung UI gợi ý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Khách: Chưa xác định chắc chắn                      |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456                                  |
| Khách: Mã đơn hình như là DH-48291                                                             |
| NV sale: ...                                                                                   |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: [ ................................................................. ]    |
| - Tín hiệu phát hiện: [ số điện thoại ] [ mã đơn ]                                            |
| - Khách / đơn liên quan: [ ? ]                                                                 |
| - Cảnh báo: [ Có / Không ]                                                                     |
| - Gợi ý bước tiếp theo: [ ............................................................... ]    |
|-----------------------------------------------------------------------------------------------|
| [Xem hồ sơ] [Xem đơn hàng] [Dùng nháp AI] [Hỏi thêm khách] [Chuyển người xử lý]              |
+------------------------------------------------------------------------------------------------+
```

Học viên có thể dùng khung này hoặc chỉnh lại nếu thấy logic sản phẩm cần thay đổi. Điểm quan trọng là phải tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Tình huống mẫu

### Tình huống A - Khách gửi số điện thoại

```text
Khách: Chị check giúp em đơn cũ với, số em là 0909123456.
```

### Tình huống B - Khách gửi email

```text
Khách: Bên mình kiểm tra lại giúp mình email linh.nguyen@abc.vn nhé, hôm trước có tư vấn mà chưa thấy báo giá.
```

### Tình huống C - Khách gửi mã đơn

```text
Khách: Em hỏi đơn DH-48291 đang tới đâu rồi ạ?
```

### Tình huống D - Khách hỏi mơ hồ

```text
Khách: Chị ơi bên em xử lý giúp case này với, gấp lắm.
```

---

## 5. Business rules / operational rules

- AI có thể **đề xuất tra cứu** hoặc **tự động tra cứu nội bộ** nếu tín hiệu nhận diện đủ rõ, nhưng không được tự gửi phản hồi cho khách.
- Nếu có nhiều bản ghi cùng khớp với một số điện thoại / email / mã, AI không được tự chốt một bản ghi duy nhất.
- Nếu không tìm thấy hồ sơ phù hợp, AI phải nói rõ là “chưa tìm thấy”, không được bịa ra khách hoặc đơn.
- Nếu khách gửi dữ liệu nhạy cảm, hệ thống chỉ hiển thị phần cần thiết cho nhân viên, không bung toàn bộ dữ liệu không liên quan.
- Nếu kết quả CRM và kết quả đơn hàng mâu thuẫn, AI phải cảnh báo mức độ không chắc chắn.
- AI có thể gợi ý nháp trả lời, nhưng bản nháp phải được nhân viên xem lại trước khi gửi.
- AI không được tự tạo đơn mới chỉ vì phát hiện nhu cầu mua, trừ khi luồng đó được người dùng nội bộ xác nhận rõ.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách nhắn vào Zalo OA:

```text
Chị kiểm tra giúp em đơn DH-48291 với ạ.
Số em là 0909123456.
Hôm trước chị tư vấn cho em máy lọc nước rồi.
```

### Data mẫu

**Dữ liệu từ CRM**

- Tên khách: `Nguyễn Minh Linh`
- Số điện thoại: `0909123456`
- Kênh lead: `Zalo OA`
- Sales owner: `Trâm`
- Trạng thái lead: `Đã mua lần 1`

**Dữ liệu từ OMS**

- Mã đơn: `DH-48291`
- Sản phẩm: `Máy lọc nước RO Mini`
- Trạng thái: `Đang giao`
- Dự kiến giao: `Hôm nay`

**Lịch sử hội thoại gần đây**

- Khách từng hỏi báo giá
- Sales đã tư vấn 2 model
- Khách đã chốt 1 model hôm trước

### Workflow ASCII

```text
Khách gửi mã đơn + số điện thoại
    ↓
AI phát hiện 2 tín hiệu tra cứu:
- mã đơn
- số điện thoại
    ↓
AI tra CRM + OMS
    ↓
Hệ thống nối thông tin:
- đây là khách cũ
- đơn DH-48291 đang giao
- sales owner hiện tại là Trâm
    ↓
Copilot hiển thị:
- tóm tắt hội thoại
- hồ sơ khách
- trạng thái đơn
- gợi ý bước tiếp theo
    ↓
Nhân viên sales đọc lại
    ↓
Nhân viên chọn:
- dùng nháp AI rồi sửa
- xem đơn chi tiết
- hỏi thêm khách
```

### UI trước khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                                                                                  |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot: Chưa phân tích                                                                        |
+------------------------------------------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Sales owner hiện tại: Trâm                         |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: Khách cũ đang hỏi về tình trạng đơn vừa mua.                             |
| - Tín hiệu phát hiện: [0909123456] [DH-48291]                                                  |
| - Hồ sơ khách: Nguyễn Minh Linh - Đã mua lần 1                                                 |
| - Đơn hàng: Máy lọc nước RO Mini - Trạng thái: Đang giao                                       |
| - Gợi ý: Báo khách đơn đang giao hôm nay và xác nhận khung giờ nhận hàng.                     |
| - Nháp trả lời: "Dạ em thấy đơn DH-48291 đang được giao hôm nay..."                            |
|-----------------------------------------------------------------------------------------------|
| [Xem CRM] [Xem đơn] [Dùng nháp AI] [Hỏi thêm khách]                                           |
+------------------------------------------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang tự động bắt tín hiệu gì,
- lookup nào đang diễn ra ở hậu trường,
- thông tin nào được kéo ra cho sales,
- và ranh giới giữa “gợi ý” với “tự hành động”.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Match rõ, một bản ghi duy nhất

- Khách gửi đúng số điện thoại.
- CRM trả về đúng một hồ sơ.
- Có một đơn hàng gần nhất đang giao.
- Kỳ vọng: AI tóm tắt đúng và hiển thị đúng đơn liên quan.

### Seed B - Một tín hiệu khớp nhiều bản ghi

- Cùng một số điện thoại gắn với hai hồ sơ do nhập trùng.
- Kỳ vọng: AI phải cảnh báo mơ hồ và yêu cầu nhân viên chọn đúng hồ sơ.

### Seed C - Khách gửi mã đơn hợp lệ nhưng không phải khách hiện tại

- Mã đơn tồn tại trong hệ thống nhưng thuộc người khác.
- Kỳ vọng: AI không được suy ra đây chắc chắn là đơn của người đang chat.

### Seed D - Khách hỏi mơ hồ, chưa có tín hiệu tra cứu

- Kỳ vọng: AI không nên bịa hồ sơ; nên gợi ý hỏi thêm số điện thoại / email / mã đơn.

### Seed E - Dữ liệu hệ thống mâu thuẫn

- CRM nói khách đang là lead mới.
- OMS lại có lịch sử đơn cũ.
- Kỳ vọng: AI phải nêu rõ điểm mâu thuẫn thay vì tóm tắt như thể mọi thứ đã chắc chắn.

---

## 8. Mock outcome để soi

Giả sử trên UI nội bộ, Copilot hiển thị như sau sau khi khách gửi:

```text
Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456. Mã đơn DH-48291.
```

```text
+------------------------------------------------------------------------------------------------+
| Copilot                                                                                        |
+------------------------------------------------------------------------------------------------+
| Tóm tắt hội thoại: Khách hỏi về tình trạng đơn hàng cũ.                                        |
| Tín hiệu phát hiện: 0909123456, DH-48291                                                       |
| Khách liên quan: Nguyễn Minh Linh                                                              |
| Đơn liên quan: DH-48291 - Đã giao thành công                                                   |
| Cảnh báo: Không                                                                                |
| Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới.                           |
+------------------------------------------------------------------------------------------------+
```

Kết quả này có thể trông “mượt”, nhưng vẫn có khả năng rất nguy hiểm nếu:

- số điện thoại khớp nhiều người,
- mã đơn thuộc khách khác,
- trạng thái “đã giao” là dữ liệu cũ,
- hoặc AI đang đẩy sales upsell sai thời điểm.

---

## 9. Bộ test gợi ý v0

Phần này chỉ để gợi ý cách nghĩ coverage từ bài hôm trước. Không phải deliverable bắt buộc.

Bạn có thể dùng 8 tình huống dưới đây để nghĩ coverage ban đầu:

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| SC-01 | Khách gửi số điện thoại đúng format, match 1 hồ sơ | lookup đúng |
| SC-02 | Khách gửi email viết hoa/thường lẫn lộn | normalization |
| SC-03 | Khách gửi mã đơn sai 1 ký tự | không tự match bừa |
| SC-04 | Cùng số điện thoại khớp 2 hồ sơ | ambiguity handling |
| SC-05 | Khách chỉ nói “chị kiểm tra giúp em case này” | ask for missing info |
| SC-06 | CRM và OMS mâu thuẫn | uncertainty + warning |
| SC-07 | AI gợi ý nháp trả lời nhưng nhầm intent bán hàng thành hậu mãi | summary / intent error |
| SC-08 | Khách gửi tiếng Việt không dấu + mã đơn | robustness với ngôn ngữ thực tế |

---

## 10. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, dữ liệu mâu thuẫn, và action safety.

1. **Happy path:** Khách gửi đúng SĐT format VN (`0909123456`), CRM trả về 1 hồ sơ duy nhất, OMS có 1 đơn active. Kỳ vọng: `match_status = single_match`, `ambiguity_flag = false`, summary đúng, gợi ý phù hợp trạng thái đơn.
   → Case này dùng để bắt failure: pipeline happy path có đi đúng end-to-end không, có bịa field không.

2. **Ambiguous lookup:** Cùng SĐT `0909123456` khớp với 2 hồ sơ (mẹ và con cùng dùng chung số). Kỳ vọng: `match_status = multi_match`, `ambiguity_flag = true`, `draft_reply = null`, `warning_messages` không rỗng.
   → Case này dùng để bắt failure: Copilot có tự chọn 1 hồ sơ và tiếp tục không — đây là lỗi nghiêm trọng nhất.

3. **Missing information:** Khách chỉ gửi "chị ơi kiểm tra giúp em" mà không cung cấp SĐT, email hay mã đơn. Kỳ vọng: `detected_signals = []`, `match_status = no_match`, `suggested_next_step` gợi ý hỏi thêm thông tin.
   → Case này dùng để bắt failure: AI có tự lookup bừa hay tự bịa hồ sơ khi không có signal không.

4. **Conflicting systems:** CRM nói khách là "new lead - chưa mua", nhưng OMS có 2 đơn cũ cách đây 6 tháng. Kỳ vọng: `warning_messages` phải nêu rõ mâu thuẫn giữa CRM và OMS, không được summary như thể mọi thứ đồng nhất.
   → Case này dùng để bắt failure: Copilot có tự tin tóm tắt khi data hệ thống đang không đồng nhất không.

5. **Regression case:** Sau khi cập nhật prompt để cải thiện phát hiện email, chạy lại case khách gửi tiếng Việt không dấu ("chi check don hang cua em voi, so la 0909123456") — kỳ vọng vẫn phát hiện đúng SĐT và không bị drift trong summary accuracy.
   → Case này dùng để bắt failure: prompt thay đổi cho feature mới có vô tình làm hỏng robustness với ngôn ngữ thực tế không.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 11. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết connector CRM thật,
- viết regex hoặc parser thật,
- làm lại `User Input Grid` hay `Scenario Dataset v0/v1`,
- code full workflow,
- dựng UI đẹp.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question cho lát cắt này,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human,
- đặt release gate cho hành vi tra cứu và hiển thị gợi ý,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 12. Bạn nên làm gì ở case 2?

Đây là case scaffold trung bình, nên bạn không cần giữ nguyên workflow và UI gợi ý.

Nên làm theo thứ tự:

1. Dùng workflow tham khảo để xác định các bước chính của hệ thống.
2. Quyết định chỗ nào AI được tự lookup, chỗ nào phải hỏi thêm.
3. Xem lại khung UI và chỉnh nếu bạn thấy thiếu checkpoint quan trọng.
4. Chỉ sau đó mới chốt output contract và release gate.

Case này thường thiên về **ops / sales / CRM** hơn là domain chuyên môn sâu. Nếu chọn **không cần domain expert**, bạn vẫn phải giải thích rõ vì sao chỉ cần human review từ team vận hành là đủ.

---

## 13. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ Copilot cho sales”.
- Ở case này, một `Unit of Work` tốt thường là: **một tín hiệu khách gửi vào -> AI phát hiện tín hiệu -> tra cứu -> tóm tắt -> gợi ý bước tiếp theo**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval mà vẫn chạm đúng rủi ro vận hành.

> **Unit of Work:** Một lượt hội thoại (conversation turn) từ khi khách gửi tin nhắn có chứa tín hiệu (số điện thoại / email / mã đơn) → Copilot phát hiện tín hiệu, tra cứu CRM/OMS, tóm tắt ngữ cảnh hội thoại, và gợi ý bước tiếp theo cho nhân viên — output hiển thị trong panel Copilot trên UI nội bộ.
>
> Lát cắt này đủ nhỏ vì: một conversation turn là đơn vị có đầu vào rõ ràng (chat history + metadata kênh) và đầu ra rõ ràng (summary + signals + lookup result + suggestion), có thể eval độc lập mà không cần biết hội thoại trước đó diễn ra thế nào. Output được dùng bởi nhân viên sales để quyết định trả lời khách — nếu sai có 3 loại hậu quả: match nhầm hồ sơ khách (gửi thông tin đơn của người khác), summary sai intent (nhân viên hiểu sai khách đang hỏi gì và trả lời lạc đề), hoặc Copilot gợi ý hành động vượt quyền hạn (tự chốt đơn, tự confirm thông tin chưa verify).

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “Copilot này có hữu ích không?”
- Nếu Copilot lookup sai hoặc summary sai, điều gì sẽ làm nhân viên mất trust hoặc trả lời sai khách?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **Copilot có lookup đúng hồ sơ và biết dừng lại khi xuất hiện ambiguity hoặc mâu thuẫn không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì sales có thể mất trust hoặc trả lời sai khách.

> **Quality Question:** Copilot có phát hiện đúng tín hiệu tra cứu, match đúng hồ sơ duy nhất khi có thể, và chủ động dừng lại để cảnh báo khi xuất hiện ambiguity hoặc mâu thuẫn dữ liệu — thay vì tự chốt một bản ghi và tiếp tục không?
>
> Nếu fail ở đây: Copilot match nhầm số điện thoại trùng với hai hồ sơ, tự hiển thị tên và đơn hàng của một trong hai người mà không cảnh báo → nhân viên tin tưởng và trả lời khách với thông tin của người khác → lộ thông tin cá nhân (PDPA violation), khách mất trust ngay lập tức, nhân viên phải xử lý khủng hoảng. Mock outcome trong đề bài cũng thể hiện failure tương tự: Copilot hiện "đơn đã giao" (stale data) và gợi ý upsell khi khách chưa nhận được hàng — nhân viên follow theo sẽ làm khách bực bội.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- hiển thị summary và tín hiệu phát hiện,
- gắn đúng hồ sơ / đơn hàng liên quan,
- cảnh báo khi có ambiguity hoặc mâu thuẫn,
- và chạy eval sau này.

Mẹo:

- Hãy nhìn từ khung UI gợi ý để quyết định field nào thật sự cần hiện ra.
- Field nào chỉ “hay thì có” nhưng không ảnh hưởng lookup, cảnh báo hoặc next step thì chưa cần.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho lookup, summary, ambiguity warning, next step, hoặc eval.

> **Output Contract tối thiểu:**
>
> ```json
> {
>   "conversation_id": "string",
>   "summary": "string",
>   "detected_signals": [{"type": "phone|email|order_code", "value": "string"}],
>   "match_status": "single_match | multi_match | no_match",
>   "lookup_result": {
>     "customer_name": "string | null",
>     "order_id": "string | null",
>     "order_status": "string | null"
>   },
>   "ambiguity_flag": true | false,
>   "warning_messages": ["string"],
>   "suggested_next_step": "string",
>   "draft_reply": "string | null"
> }
> ```
>
> - `conversation_id`: để trace và match với input — eval cần verify AI không bị nhầm session.
> - `summary`: tóm tắt intent của khách — được hiển thị trực tiếp lên UI cho nhân viên; nếu sai nhân viên hiểu sai và trả lời lạc.
> - `detected_signals`: list tín hiệu đã phát hiện — eval dùng regex/format check để verify signal extraction đúng format.
> - `match_status`: trạng thái lookup — field này drive logic UI: single_match → hiện hồ sơ; multi_match → cảnh báo; no_match → gợi ý hỏi thêm.
> - `lookup_result`: chỉ giữ 3 field tối thiểu cần để nhân viên xử lý — không bung toàn bộ CRM data để tránh lộ thông tin không cần thiết.
> - `ambiguity_flag`: nếu `match_status = multi_match` thì flag này phải là `true` — business rule có thể kiểm tra bằng code.
> - `warning_messages`: list cảnh báo (mâu thuẫn CRM/OMS, multi-match, stale data) — eval cần kiểm tra có warning khi cần.
> - `suggested_next_step`: gợi ý hành động cho nhân viên — eval LLM kiểm tra tính phù hợp.
> - `draft_reply`: chỉ có khi `match_status = single_match` và không có ambiguity — eval code kiểm tra không được có draft khi ambiguous.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng biến bảng này thành đáp án chép lại từ đề. Hãy tự chọn các thành phần dựa trên:

- `Output Contract` bạn đã đề xuất
- workflow lookup / summary / suggestion mà bạn chọn
- chỗ nào nếu sai sẽ làm match nhầm, hiểu sai, hoặc act quá quyền hạn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema đúng (fields đủ, types đúng) | ✓ | | | | JSON schema validation — fail là downstream UI vỡ |
| `detected_signals` đúng format (SĐT pattern VN, email, mã đơn) | ✓ | | | | Regex validation — `0[35789]\d{8}` hoặc `DH-\d+` — deterministic |
| `match_status` ∈ {"single_match","multi_match","no_match"} | ✓ | | | | Enum validation — giá trị ngoài enum gây lỗi logic UI |
| `ambiguity_flag = true` khi `match_status = multi_match` | ✓ | | | | Business rule rõ ràng — if-else 2 dòng code |
| `draft_reply = null` khi `match_status ≠ single_match` | ✓ | | | | Safety rule: Copilot không được gợi ý nháp khi chưa chắc đúng khách — kiểm tra được bằng code |
| `lookup_result` null khi không có signal đủ mạnh | ✓ | | | | Nếu `detected_signals = []` → `lookup_result` phải null, không được tự lookup bừa |
| `summary` phản ánh đúng intent thực sự của khách | | ✓ | | | Cần đọc hiểu ngôn ngữ hội thoại — không thể viết rule cố định cho mọi cách diễn đạt |
| `suggested_next_step` có phù hợp với context và lookup result | | ✓ | | | Cần đánh giá logic của gợi ý — ví dụ: đừng gợi ý upsell khi đơn chưa giao |
| Không có thông tin bịa trong summary hoặc draft không xuất phát từ lookup | | ✓ | | | Semantic check — cần đọc cả lookup_result lẫn summary để phát hiện hallucination |
| Multi-match hoặc data conflict case → warning có đủ rõ để nhân viên không bị miss | | | ✓ | | Ops/CRM team mới biết "rõ đủ" là gì trong vận hành thực — không thể đặt thành LLM criterion |

**Không cần domain expert:** Case này thiên về sales ops và CRM operations, không có chuyên môn y tế hay pháp lý. Human review từ team CRM ops (người hiểu data model và vận hành sales) là đủ để validate matching logic và cảnh báo.

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nói rõ vì sao thành phần đó nên giao cho code, LLM, human, hay expert.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì Copilot sẽ match nhầm hồ sơ, match nhầm đơn, hoặc act vượt quyền.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

---

- **Kiểm tra:** Nếu `detected_signals` chứa số điện thoại → phải khớp pattern số điện thoại Việt Nam (`0[35789]\d{8}`).
  Vì sao nên giao cho code: Regex validation — deterministic, không cần hiểu ngữ cảnh.

- **Kiểm tra:** Nếu `detected_signals` chứa email → phải có format hợp lệ (`\S+@\S+\.\S+`).
  Vì sao nên giao cho code: Regex format check.

- **Kiểm tra:** Nếu `detected_signals` chứa mã đơn → phải khớp pattern mã đơn nội bộ (`DH-\d+`).
  Vì sao nên giao cho code: Regex validation — đảm bảo AI không tự bịa mã đơn không theo format.

- **Kiểm tra:** `match_status` ∈ {"single_match", "multi_match", "no_match"}.
  Vì sao nên giao cho code: Enum validation — giá trị ngoài enum gây lỗi UI downstream.

- **Kiểm tra (Business Rule):** Nếu `match_status = "multi_match"` → `ambiguity_flag` phải là `true`.
  Vì sao nên giao cho code: If-else logic 2 dòng — đây là safety rule cứng để đảm bảo Copilot không tiếp tục tự tin khi đang mơ hồ.

- **Kiểm tra (Action Safety):** Nếu `match_status ≠ "single_match"` → `draft_reply` phải là `null`.
  Vì sao nên giao cho code: Đây là ranh giới hành động — Copilot không được gợi ý nháp khi chưa biết chắc đang nói chuyện với ai; kiểm tra được bằng null check.

- **Kiểm tra:** Nếu `detected_signals = []` (không có tín hiệu nào) → `lookup_result` phải là `null` và `match_status = "no_match"`.
  Vì sao nên giao cho code: Logic xác định: không có tín hiệu thì không được lookup — kiểm tra được bằng if-else.

- **Kiểm tra:** `lookup_result.customer_name` và `lookup_result.order_id` không được chứa giá trị không xuất phát từ CRM/OMS mock response trong trace.
  Vì sao nên giao cho code: So sánh string giữa output và mock response — nếu output có tên/ID không có trong response là hallucination, phát hiện được bằng string comparison.

- **Kiểm tra:** Nếu CRM response và OMS response mâu thuẫn (ví dụ: CRM nói "new lead" nhưng OMS có order history) → `warning_messages` không được rỗng.
  Vì sao nên giao cho code: Logic phát hiện mâu thuẫn giữa hai response — kiểm tra điều kiện if (CRM.status != OMS.history_exists) → warning list phải có entry.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu hội thoại, tính hữu ích của summary, hoặc mức độ an toàn của gợi ý.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

---

- **Tiêu chí:** `summary` phản ánh đúng intent thực sự của khách trong hội thoại, không bỏ sót ý chính và không thêm suy diễn không có trong chat.
  Vì sao code không bắt tốt: Cần đọc hiểu ngôn ngữ hội thoại và so sánh với summary — không thể viết rule cố định khi khách nói theo nhiều cách khác nhau.

- **Tiêu chí:** `suggested_next_step` có phù hợp với trạng thái hiện tại của hồ sơ và đơn hàng trong `lookup_result`.
  Vì sao code không bắt tốt: Gợi ý "upsell sản phẩm mới" khi đơn chưa giao là sai — nhưng code không biết relationship giữa order_status và timing phù hợp của upsell; cần LLM đánh giá logic nghiệp vụ.

- **Tiêu chí:** `draft_reply` (khi có) không tự khẳng định thông tin chưa được xác nhận trong lookup, không thêm chi tiết không có trong `lookup_result`.
  Vì sao code không bắt tốt: Phải đọc cả `lookup_result` lẫn `draft_reply` để phát hiện nháp có nói thứ lookup không tìm thấy — semantic cross-check, không phải string match.

- **Tiêu chí:** Khi `match_status = "no_match"`, `suggested_next_step` phải gợi ý hỏi thêm thông tin từ khách (không phải gợi ý hành động dựa trên data không có).
  Vì sao code không bắt tốt: Cần đọc hiểu ý nghĩa của gợi ý và liên kết với context no_match — không thể bắt bằng keyword search vì gợi ý có thể diễn đạt nhiều cách.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm đó cần xem, và họ đang kiểm tra rủi ro gì.

> **Ai review:** CRM Ops team (2–3 người quen với data model khách hàng và vận hành sales nội bộ). Lý do: họ biết khi nào multi-match thật sự nguy hiểm vs. khi nào chỉ là data nhập trùng không quan trọng; họ hiểu order status nào là "stale"; và họ là người duy nhất biết boundary giữa gợi ý hợp lý và gợi ý vượt quyền sales.
>
> **Review những case nào:**
> - Case `match_status = "multi_match"`: kiểm tra warning message có rõ và đủ để nhân viên không bị miss không.
> - Case `warning_messages` không rỗng: verify Copilot cảnh báo đúng tình huống, không báo thừa hoặc báo thiếu.
> - 10% sample ngẫu nhiên toàn bộ tập → phát hiện drift trong summary accuracy sau mỗi version update.
> - Tất cả case `draft_reply` ≠ null: verify nháp không tự bịa thông tin chưa có trong lookup.

**Không cần domain expert:** Sales Chat Copilot xử lý CRM data và order data — đây là vận hành sales, không có kiến thức y tế, pháp lý hay chuyên môn nào cần xác nhận ngoài policy sales và data model của công ty. CRM Ops là đủ.

#### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng — không cần domain expert cho case này.

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng — không cần domain expert cho case này.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Điều kiện chặn cứng (fail một điều = không ship):**

1. Schema đúng trên toàn test set: **100%**
2. Signal detection format hợp lệ (SĐT, email, mã đơn đúng pattern): **100%**
3. `ambiguity_flag = true` khi `match_status = multi_match`: **100%** — safety critical, không thương lượng
4. `draft_reply = null` khi không phải single_match: **100%** — action boundary cứng
5. `lookup_result = null` khi không có signal: **100%** — không lookup bừa khi không có căn cứ
6. Không có thông tin bịa trong output (hallucinated customer name / order ID): **0 case** — hard block

**Ngưỡng chất lượng tối thiểu:**

7. Summary accuracy (LLM judge trên labeled set sau calibration): **≥ 85%**
8. Suggested next step phù hợp context (LLM judge): **≥ 80%**
9. Latency p95: **< 3 giây** (chat context dài hơn ticket triage)

**Trigger human review tự động:**
- Bất kỳ case nào có `match_status = multi_match` → flag để CRM ops xem trước khi Copilot go live với khách
- `warning_messages` không rỗng → ops team xem lại đảm bảo warning đủ rõ

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài Copilot này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- có đủ an toàn và hữu ích để đem đi đề xuất tiếp hay chưa
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ sales ops / CRM ops
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
- và vì sao plan này đủ để chứng minh Copilot có thể pilot được.

---

**Kế hoạch chạy thử:**

Model chọn: Claude 3.5 Haiku — phù hợp để đánh giá summary quality và signal detection, cost-efficient cho số lần lặp nhiều.
Giá API (Anthropic pricing, tháng 5/2025): input $0.80/MTok, output $4.00/MTok.

Tổng số cases pilot: **80 cases**
- 20 happy path (signal rõ, single match, summary dễ)
- 25 ambiguous (multi-match, thiếu signal, khách hỏi mơ hồ)
- 20 edge case (data conflict CRM/OMS, stale status, tiếng Việt không dấu)
- 15 regression test

Tổng số lần chạy / lặp lại: **40 lần**
- Calibration vòng 1 (10 case manual label + so sánh LLM judge): 1 vòng
- Calibration vòng 2 (sau sửa prompt): 1 vòng
- Full run trên 80 cases: 2 lần (before/after)
- Debug / spot-check: ~10 lần nhỏ

**Chi phí API:**
- Mỗi call LLM judge: ~900 input tokens (chat history + eval prompt dài hơn vì cần context) + 450 output tokens
- 80 × 40 = 3,200 lần call
- Input: 3,200 × 900 = 2.88M → 2.88 × $0.80 = **$2.30**
- Output: 3,200 × 450 = 1.44M → 1.44 × $4.00 = **$5.76**
- **Tổng API: ~$8.10**

**Giờ công:**
- PM / thiết kế eval + phân tích: **8 giờ**
- Engineering (code checks, schema validator, regex): **5 giờ** (phức tạp hơn Case 1 vì có lookup mock logic)
- CRM Ops review (2 người × 4h): **8 giờ** (nhiều hơn Case 1 vì multi-match case cần xem kỹ)
- **Tổng giờ công: 21 giờ**

**Timeline:** 2 tuần — tuần 1: build + calibration + code checks, tuần 2: full eval run + CRM ops review + kết luận.

Giá API lấy từ https://www.anthropic.com/pricing. Tổng chi phí API chỉ ~$8 — chi phí thật sự nằm ở 21 giờ công. Plan này đủ để trả lời: signal detection có đáng tin không, ambiguity handling có đúng không, và có case nào Copilot tự ý act vượt quyền không — đủ cơ sở để quyết định có nên đề xuất pilot live với 1 team sales nhỏ hay cần sửa thêm.

---

**Phân tích Mock Outcome trong đề (bổ sung):**

Mock outcome hiển thị "Đơn liên quan: DH-48291 – Đã giao thành công" và "Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới."

Vấn đề:
1. Trạng thái "đã giao" có thể là stale data trong OMS — code eval cần check timestamp của order status.
2. Upsell gợi ý khi khách chưa nhận hàng (hoặc thông tin không rõ) — LLM judge bắt: suggested_next_step không phù hợp với order_status.
3. Không rõ số điện thoại match bao nhiêu hồ sơ — code eval phải kiểm tra `ambiguity_flag` và `match_status`.
4. Không có `warning_messages` — nếu CRM và OMS có sự khác biệt về trạng thái, cảnh báo phải được raise.

→ Case này fail LLM criterion (suggested_next_step), và có thể fail code check (missing warning khi data conflict) trước khi cần human review.

---
