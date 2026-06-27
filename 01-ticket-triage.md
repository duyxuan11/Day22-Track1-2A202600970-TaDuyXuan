# Case 1 - Support Ticket Triage

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

1. Happy path: Ticket `"Billing issue — I was charged twice this month"` từ khách enterprise. Kỳ vọng: `category=billing`, `urgency=high`, `requires_human=true`, `route_to=billing_ops`. Case này bắt lỗi nếu AI phân loại nhầm sang `technical` hoặc không escalate dù nội dung đủ rõ.

2. Ambiguous input: Ticket chỉ có tiêu đề `"Question"`, nội dung `"Hi can you help"`. Kỳ vọng: AI gán `category=clarification_needed`, `confidence` thấp (< 0.5), không tự chốn `urgency=high`. Case bắt lỗi hallucination — AI không được tự thêm urgency khi input không có đủ tín hiệu.

3. Missing information: Ticket có subject rõ `"Cannot export PDF report"` nhưng không có `customer_tier` trong input. Kỳ vọng: AI vẫn gán được `category=technical` và `urgency=medium`, nhưng `requires_human` không thể trigger rule enterprise. Case bắt lỗi AI giả định tier khi field bị thiếu.

4. High-risk / escalation: Ticket `"CRITICAL: our entire API is down and production orders are failing"` từ khách enterprise. Kỳ vọng: `urgency=critical`, `requires_human=true`, `route_to=human_escalation`. Case bắt lỗi nếu AI không nhận ra từ khoá production outage là P0.

5. Regression case: Ticket từ T-002 mock outcome — "URGENT: payment failed and account disabled" từ enterprise, nhưng AI gán `category=product_question`, `urgency=medium`, `requires_human=false` như trong mock outcome ban đầu. Case này phải luôn fail trong eval suite để đảm bảo lỗi không tái phát sau khi đã sửa prompt.

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

> **Lát cắt được chọn:** Một ticket mới đi vào hệ thống → AI đọc tiêu đề, nội dung, loại khách → trả về bộ nhãn gồm: `category`, `urgency`, `route_to`, `requires_human`, `reason_codes`, `confidence`.
>
> Đây là đơn vị đủ nhỏ vì mỗi ticket là một quyết định độc lập, output rõ ràng, có thể so với golden label, và eval từng ticket xong là ra kết quả ngay — không phụ thuộc vào session trước hay trạng thái hệ thống ngoài. Nếu đơn vị lớn hơn (ví dụ toàn bộ hội thoại hỗ trợ), sẽ rất khó quy trách nhiệm lỗi về đúng bước. Nếu nhỏ hơn (ví dụ chỉ chấm `category` đơn lẻ), sẽ bỏ sót rủi ro liên kết giữa `urgency` và `requires_human`.

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

> **Quality Question:** AI có gán đúng `route_to` và bật `requires_human = true` cho mọi ticket có dấu hiệu chặn công việc hoặc khách doanh nghiệp đang gặp sự cố nghiêm trọng không?
>
> Nếu AI fail ở đây, ticket sẽ rơi vào hàng xử lý sai team (ví dụ ticket billing vào product_team), hoặc tệ hơn là bị đánh `urgency = low` khi khách đang bị locked out. Điều này trực tiếp gây trễ SLA, ảnh hưởng doanh nghiệp khách hàng, và làm mất tin tưởng vào toàn bộ hệ thống triage. Bị sai ở route và escalation là lỗi P1 — không phải lỗi giao diện hay style.

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

> - `ticket_id` — khoá để trace kết quả eval về đúng ticket, bắt buộc trong mọi log.
> - `category` (enum: `technical`, `billing`, `feature_request`, `clarification_needed`) — quyết định team nào nhận ticket; sai đây là P1.
> - `urgency` (enum: `low`, `medium`, `high`, `critical`) — quyết định hàng đợi; với enterprise + dấu hiệu blocking, không được để `low`.
> - `requires_human` (boolean) — trigger escalation flag trên UI và đẩy vào hàng ưu tiên cao; cần để eval kiểm tra business rule `enterprise + high/critical → true`.
> - `route_to` (enum: `technical_support`, `billing_ops`, `product_team`, `human_escalation`) — gửi ticket đến đúng team; đây là output quyết định routing thật sự.
> - `reason_codes` (array of strings) — giải thích ngắn tại sao AI chọn như vậy, dùng để LLM judge kiểm tra hallucination và human review kiểm tra logic.
> - `confidence` (float 0–1) — dùng để route case xuống human review khi AI không chắc (confidence thấp = cần người xem lại).
>
> Các field không cần ở v0: timestamp (có ở metadata hệ thống), customer_name (không ảnh hưởng routing), internal notes (nguy cơ lộ data không cần thiết)..

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema / enum hợp lệ | ✓ | | | | Deterministic — có thể kiểm tra bằng validator; sai là vỡ hệ thống ngay |
| Business rule: enterprise + high/critical → requires_human | ✓ | | | | Rule rõ ràng, không cần đọc ngữ nghĩa, code bắt chắc hơn LLM |
| Routing đúng category (billing ≠ product_team) | ✓ | | | | Rule cứng từ policy; có thể viết assertion theo bảng category→route |
| confidence nằm trong [0, 1] | ✓ | | | | Numeric bound, code kiểm tra trong 1 dòng |
| category phù hợp nội dung ticket | | ✓ | | | Cần đọc hiểu ngữ nghĩa để phân loại đúng — code không bắt tốt khi input mơ hồ |
| reason_codes phản ánh đúng nội dung (không hallucinate) | | ✓ | | | Cần LLM so sánh claim trong reason với nội dung ticket gốc |
| urgency hợp lý với mức độ nghiêm trọng | | ✓ | | | "blocking work" hay "account disabled" cần đọc hiểu mới biết đủ để `high/critical` |
| Sample lỗi P1 từ enterprise customer | | | ✓ | | Nhân viên vận hành cần xem lại case có impact cao để xác nhận human gate hoạt động đúng |
| Taxonomy category và routing policy | | | | ✗ | Không cần domain expert — policy do team ops nội bộ định nghĩa, không cần chuyên môn bên ngoài |

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

- Kiểm tra: Output có đúng JSON schema không — tất cả field bắt buộc (`ticket_id`, `category`, `urgency`, `requires_human`, `route_to`, `reason_codes`, `confidence`) đều phải có mặt và đúng kiểu dữ liệu.
  Vì sao nên giao cho code: Đây là invariant kỹ thuật — nếu output sai schema, toàn bộ hệ thống downstream vỡ. Code validate schema trong 1 hàm, chạy mọi case, không tốn thêm chi phí.

- Kiểm tra: `category` phải thuộc enum `{technical, billing, feature_request, clarification_needed}`.
  Vì sao nên giao cho code: Tập cho phép cố định, không cần đọc ngữ nghĩa để biết "product_question" là sai enum.

- Kiểm tra: `urgency` phải thuộc enum `{low, medium, high, critical}`.
  Vì sao nên giao cho code: Như trên — giá trị ngoài enum báo ngay lỗi format.

- Kiểm tra: `confidence` phải là số trong khoảng `[0.0, 1.0]`.
  Vì sao nên giao cho code: Bound numeric, assert 1 dòng.

- Kiểm tra: Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, thì `requires_human` phải bằng `true`.
  Vì sao nên giao cho code: Business rule tường minh, có thể viết thành `if enterprise AND urgency in {high, critical}: assert requires_human == true`.

- Kiểm tra: Ticket có `category = billing` không được có `route_to = product_team`.
  Vì sao nên giao cho code: Policy cứng — bảng mapping category→allowed_routes có thể hardcode và assert.

- Kiểm tra: Nếu nội dung ticket chứa các từ khoá `blocking`, `locked out`, `account disabled`, `cannot login`, thì `urgency` không được là `low`.
  Vì sao nên giao cho code: Có thể dùng keyword matching — đây là safety net bắt lỗi thô nhất trước khi LLM judge chạy semantic.

- Kiểm tra: `reason_codes` không được là mảng rỗng khi `requires_human = true`.
  Vì sao nên giao cho code: Invariant — nếu AI đánh cờ escalation mà không có reason, hệ thống không có gì để nhân viên đọc; rule đơn giản, assert được.

- Kiểm tra: Mock outcome T-002 — `category = "Product question"` và `urgency = "Medium"` và `requires_human = false` → phải bị flag là fail theo business rule.
  Vì sao nên giao cho code: Regression test cụ thể; lưu case này vào reference dataset để bắt nếu sau này AI "học lại" sai.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

---

- Tiêu chí: `category` có phù hợp với nội dung thật của ticket không — ví dụ ticket nói "payment failed" nhưng AI gán `feature_request`.
  Vì sao code không bắt tốt: Code chỉ kiểm tra được enum hợp lệ, không đọc được ngữ nghĩa. Việc phân loại đúng cần hiểu ý định của khách, không chỉ kiểm tra format.

- Tiêu chí: `urgency` có khớp với mức độ nghiêm trọng thực sự trong ticket không — ví dụ khách nói "blocking my work" nhưng AI vẫn gán `medium`.
  Vì sao code không bắt tốt: Keyword matching bắt được từ khoá rõ như "locked out", nhưng không bắt được cách diễn đạt gián tiếp như "chúng tôi không làm việc được từ sáng".

- Tiêu chí: `reason_codes` có phản ánh đúng nội dung ticket không — AI không được bịa thêm sự thật không có trong input.
  Vì sao code không bắt tốt: Code không thể so sánh ngữ nghĩa giữa claim trong `reason_codes` và nội dung ticket gốc — cần LLM đọc cả hai và đánh giá xem có hallucination không.

- Tiêu chí: Với ticket mơ hồ (Seed B: "Help / Please help asap"), AI có đúng đắn không tự tin gán category mà để `clarification_needed` hoặc `confidence` thấp không?
  Vì sao code không bắt tốt: Code biết confidence thấp nhưng không biết "confidence thấp ở đây có phải vì thiếu thông tin hay vì model đang đoán mò" — LLM judge đọc cả ticket và output mới phán xét được.

- Tiêu chí: Toàn bộ output có nhất quán không — ví dụ `requires_human = false` nhưng `urgency = critical` là mâu thuẫn nội tại.
  Vì sao code không bắt tốt: Rule cứng bắt được một số combination, nhưng mâu thuẫn ngữ nghĩa tinh vi hơn (ví dụ reason mô tả mức high nhưng urgency trả về low) cần LLM đọc tổng thể.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

> **Ai review:** Nhân viên vận hành support (team lead hoặc senior agent) — không phải domain expert chuyên môn bên ngoài.
>
> **Review những case nào:**
> - Các ticket từ `enterprise` customer có `urgency = high/critical` để xác nhận AI escalate đúng.
> - Các case LLM judge cho điểm thấp hoặc conflict với code check (ví dụ code pass nhưng LLM judge flag).
> - Mẫu random 5–10% ticket trong tuần đầu pilot để ước lượng chất lượng tổng thể.
> - Bất kỳ case nào `confidence < 0.6` — AI không chắc thì người cần xem lại.
>
> **Vì sao không cần domain expert:** Case này là SaaS support triage thông thường — category và routing policy do team nội bộ định nghĩa, không liên quan đến pháp lý, y tế, hay tài chính. Nhân viên vận hành support đủ năng lực xác nhận "ticket này có đi đúng hàng không" mà không cần chuyên gia bên ngoài.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã route hoặc gắn nhãn gì,
- dấu hiệu hoặc evidence nào khiến case bị đẩy sang expert,
- expert có thể duyệt / sửa / escalation ở đâu.

**Trả lời của bạn:**

`Không áp dụng` — Policy triage là do team vận hành nội bộ xác định, không cần chuyên gia y tế, pháp lý hay tài chính. Senior support agent đủ thẩm quyền duyệt edge case và xác nhận routing taxonomy.

#### 7B. Tiêu chí review của Domain Expert

`Không áp dụng` — xem lý do ở 7A.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Điều kiện chặn (block nếu vi phạm bất kỳ điều nào):**
- Schema pass rate < 99.5% — output vỡ schema là lỗi hệ thống, không thể ship.
- Business rule violations > 0 — ví dụ: enterprise + critical mà `requires_human = false`, hoặc billing route vào product_team.
- Escalation recall (bắt đúng ticket cần người) < 95% — miss escalation là P1, tỷ lệ này phải cao.
- Category accuracy trên reference dataset < 88% — routing sai quá nhiều thì vận hành không tin tưởng hệ thống.
- Bất kỳ P0 nào: hallucination trong reason_codes (bịa thêm thông tin không có trong ticket), lộ internal ID.

**Ngưỡng cảnh báo (warn, không chặn):**
- Confidence trung bình < 0.70 → tăng tỷ lệ human review tạm thời.
- LLM judge và code check disagree > 10% cases → cần audit rubric.
- Latency P95 > 3 giây → cần xem lại prompt hoặc model.

**Trường hợp bắt buộc human review trước ship:**
- Sample 20 case enterprise + high/critical từ eval run, đủ điều kiện mới release.
- Nếu có bất kỳ failure mode mới nào chưa có trong taxonomy → chặn, thêm case vào dataset, chạy lại.

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

- **Model sử dụng:** Claude Haiku 4.5 cho triage chính (rẻ, nhanh), Claude Sonnet 4.6 làm LLM judge.
- **Giá API tham khảo (tháng 6/2026):**
  - Haiku 4.5: ~$0.80/1M input tokens, ~$4/1M output tokens
  - Sonnet 4.6 (judge): ~$3/1M input tokens, ~$15/1M output tokens
- **Cases pilot:** 75 cases (mix: ~30 happy path, ~20 ambiguous, ~15 escalation, ~10 regression).
- **Số lần chạy:** 40 lần (bao gồm 2–3 vòng prompt iteration, mỗi vòng chạy full 75 cases).

**Ước tính token:**
- Mỗi triage call: ~500 input + ~200 output tokens → $0.0005/call
- 75 cases × 40 runs = 3,000 calls → ~$1.50 (Haiku triage)
- LLM judge: mỗi call ~1,000 input + ~300 output → $0.008/call; chạy 50% cases = 1,500 calls → ~$12 (Sonnet judge)
- **Tổng chi phí API: ~$14**

**Giờ công:**
- PM / thiết kế eval: 6 giờ (viết rubric, lên dataset, xác nhận release gate)
- Kỹ thuật / chạy eval: 8 giờ (viết code assertions, eval runner, phân tích kết quả)
- Human review (nhân viên ops): 4 giờ (duyệt 20 case enterprise escalation + sample random)
- **Tổng nhân công: ~18 giờ**

**Tổng thời gian dự kiến:** 1 tuần (song song)

Giá API lấy từ trang pricing chính thức của Anthropic (anthropic.com/pricing). Với quy mô 75 cases và 40 lần chạy, tổng chi phí rơi vào khoảng **$14–20 tuỳ mức prompt dài ngắn**. Plan này đủ để chứng minh: schema pass rate, escalation recall, và category accuracy có thể đạt ngưỡng release gate — tức là đủ để đề xuất pilot thật với nhân viên ops.

---
