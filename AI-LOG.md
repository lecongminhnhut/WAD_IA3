# Nhật ký sử dụng AI – IA#3 – Three Answers to One Brief

_Mỗi lần dùng AI được ghi thành một mục riêng theo trình tự thực hiện._

| #   | Ngày       | Nội dung                                                                                                                                                  | Nhóm            |
| --- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| 1   | 2026-11-04 | Rà soát độc lập ba implementation JavaScript gốc A, B và C theo `brief.md` và `README.md`; đối chiếu từng claim với test evidence sau khi bảo toàn review | CSC13008 – IA#3 |

---

## Mục 1: Rà soát độc lập các implementation gốc A, B và C

| Trường          | Nội dung                                                                                                                                  |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Ngày và giờ     | 2026-11-04 11:50:36 +07:00                                                                                                                |
| Người thực hiện | Lê Công Minh Nhựt                                                                                                                         |
| Công cụ AI      | Codex                                                                                                                                     |
| Model           | GPT-5 (Codex)                                                                                                                             |
| Mục đích        | Rà soát độc lập source gốc trước khi xem `VERDICTS.md`, test reports hoặc screenshots; sau đó kiểm chứng các claim bằng evidence thực tế. |

### Prompt đã dùng

```text
Please independently review the three original JavaScript implementations A, B, and C against the requirements in brief.md and README.md.

For each implementation, identify every potential violation of the requirements.

Pay particular attention to:
- Product loading and rendering.
- Product ordering and formatting.
- Loading and error handling.
- Stock retrieval and total calculation.
- Asynchronous execution and Promise handling.
- Request concurrency and parallel execution.
- Timeout behavior.
- Empty-state handling.
- Retry behavior.
- TypeScript compatibility using the original jsconfig.json.
- Any other requirement in the brief.

For every issue, provide:
1. The affected version.
2. The exact source code location.
3. The violated requirement.
4. Why the implementation is incorrect.
5. The expected correct behavior.
6. A suggested correction.

Also identify implementations or requirements that appear correct.

Do not modify any code. Do not assume that an implementation passes merely because it looks reasonable. Base your review on the actual source code and supplied requirements.
```

### Tóm tắt phản hồi của AI

- Version A: phát hiện stock requests tại `public/versions/a.js:52-56` được khởi tạo bằng `forEach(async ...)` nhưng không được await; stock value có thể được ghi trước khi requests hoàn tất. Review cũng ghi nhận khả năng lỗi stock request thoát khỏi `try...catch`.
- Version B: phát hiện `{ timeout: 5000 }` tại `public/versions/b.js:16` không tạo timeout cho browser Fetch API; review cũng dự đoán lỗi TypeScript tại row 8 vì `timeout` không thuộc `RequestInit`.
- Version C: không phát hiện vi phạm source-level rõ ràng.
- Review cũng ghi nhận các phần có vẻ đúng: rendering, thứ tự sản phẩm, formatting, `textContent`, concurrency, retry, empty state và timeout ở các version dùng `AbortSignal.timeout`.

### Phần đã sử dụng / chỉnh sửa / bỏ

| Phần                             | Quyết định                                                      | Lý do                                                                                     |
| -------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Review độc lập nguyên văn        | Dùng nguyên văn trong `ASSISTANT-REVIEW.md`                     | Tôi phải bảo toàn toàn bộ phản hồi trước khi đọc predictions và evidence.                 |
| Prompt review                    | Ghi nguyên văn trong `ASSISTANT-REVIEW.md` và mục này           | Đáp ứng yêu cầu lưu lại prompt chính xác.                                                 |
| Claim về A stock rejection       | Giữ nhưng phân loại là chưa được xác nhận bằng runtime evidence | Test chuẩn không làm stock endpoint trả lỗi.                                              |
| Claim về B timeout và type-check | Giữ và xác nhận bằng evidence                                   | `b-row6.png` cho thấy request còn pending sau 7 giây; `b-row8.png` cho thấy lỗi `TS2353`. |
| Claim về C                       | Giữ và đối chiếu với các screenshots row 1–9                    | Evidence không cho thấy lỗi trong các điều kiện đã kiểm thử.                              |

### Cách kiểm chứng

- Đọc requirements và source gốc trước khi đọc `VERDICTS.md` hoặc evidence.
- Lưu toàn bộ independent review trong `ASSISTANT-REVIEW.md` trước khi bắt đầu đối chiếu.
- Đọc `VERDICTS.md` sau đó và kiểm tra 27 screenshots trong `evidence/`, gồm row 1–9 cho A, B và C.
- Xác nhận Version A row 2 là lỗi stock value `0 đ` thay vì `12.360.000 đ`.
- Xác nhận Version B row 6 vẫn loading sau hơn 5 giây trong mode `hang`.
- Xác nhận Version B row 8 có lỗi `TS2353` cho thuộc tính `timeout`.
- Kiểm tra các screenshots còn lại để đối chiếu rendering, formatting, literal HTML text, slow mode, retry, Console, timeout, type-check và empty state.
- Không sửa implementation, không sửa `VERDICTS.md`.
