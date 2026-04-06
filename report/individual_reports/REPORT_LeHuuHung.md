# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Lê Hữu Hưng 
- **Student ID**: 2A202600098
- **Date**: 06/04/2026

---

## I. Technical Contribution (15 Points)

- **Modules Implemented**: `src/agent/agent.py` (lines 31–43), `src/tools/tools.py` (lines 8–12)

### 1. Realtime Datetime Injection (`src/agent/agent.py:31-43`)

Phương thức `get_system_prompt()` gọi `datetime.now()` tại runtime và nhúng ngày hiện tại vào template `ReAct.v2.txt` thông qua placeholder `{current_date}`:

```python
def get_system_prompt(self) -> str:
    current_date = datetime.now().strftime("%A, %d/%m/%Y")
    return Path("src/prompts/ReAct.v2.txt").read_text().format(
        current_date=current_date,
        tool_descriptions=...,
    )
```

Hàm này được gọi ở **mỗi bước sinh LLM**, đảm bảo ngày luôn được cập nhật. Lợi ích: agent có thể hiểu các tham chiếu thời gian tương đối như "cuối tuần này" ngay trong `Thought` đầu tiên mà không cần gọi thêm tool `get_system_time[]`.

### 2. Web Search Tool (`src/tools/tools.py:8-12`)

Ban đầu được xây dựng trên Brave Search API, nhưng API này liên tục trả về kết quả rỗng cho các truy vấn tiếng Việt. Giải pháp là chuyển sang **DuckDuckGo (`ddgs`)** — không cần API key và hoạt động tốt với nội dung tiếng Việt:

```python
ddgs = DDGS()

def web_search(query: str) -> str:
    res = ddgs.text(query, max_results=5)
    return "\n".join(t['body'] for t in res)
```

Kết quả trả về dạng plain text (không JSON, không markdown) để LLM có thể đọc và trích dẫn trực tiếp trong `Thought` tiếp theo. Code Brave cũ (lines 13–41) vẫn được giữ lại dưới dạng unreachable dead code để ghi lại lịch sử migration.

---

## II. Debugging Case Study (10 Points)

- **Problem Description**: Agent tạo ra sự kiện `PARSE_ERROR` khi gọi `get_system_time` — nó xuất ra `Action: get_system_time` thay vì `Action: get_system_time[]`, khiến bước đó thất bại hoàn toàn.

- **Log Source** (`logs/2026-04-06.log`):
```json
{ "event": "PARSE_ERROR", "data": { "step": 2, "reason": "No Action or Final Answer found", "output": "Action: get_system_time" } }
```

- **Diagnosis**: Parser tại `agent.py:103` dùng regex `r"Action:\s*(\w+)\[([^\]]*)\]"`, bắt buộc phải có dấu ngoặc vuông `[...]`. Với các tool không có tham số, LLM đôi khi bỏ qua cặp `[]` rỗng vì cú pháp trông có vẻ thừa — đây là lỗi thiết kế prompt, không phải lỗi của model.

- **Solution**: Thêm nudge prompt tại `agent.py:152-155` để nhắc nhở LLM khi xảy ra parse error. LLM thường tự sửa ở bước tiếp theo. Giải pháp lâu dài: thêm ví dụ tường minh trong system prompt — `Action: get_system_time[]` — để loại bỏ hoàn toàn sự mơ hồ.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**: Block `Thought` buộc agent phân tách bài toán trước khi hành động. Với câu hỏi *"Thời tiết Đà Nẵng cuối tuần này?"*, Chatbot đoán từ dữ liệu huấn luyện cũ. Agent tách thành hai bước rõ ràng: *(1) "cuối tuần này" là ngày nào? → (2) tìm kiếm thời tiết cho những ngày đó.* Cách phân tách từng bước này không thể thực hiện trong một lần gọi Chatbot. Ngoài ra, khi có lỗi, block `Thought` làm nguyên nhân thất bại hiện ra rõ ràng thay vì im lặng hallucinate.

2. **Reliability**: Agent hoạt động *kém hơn* Chatbot trong hai trường hợp:
   - **Câu hỏi sự kiện đơn giản**: Overhead ~10× token do system prompt 366 token được nạp mỗi lần gọi, trong khi LLM đã biết câu trả lời từ training.
   - **Search failure cascades**: Khi `web_search` liên tục trả về rỗng, agent cuối cùng bịa ra câu trả lời sau khi cạn các lần retry — tệ hơn Chatbot vốn ít nhất sẽ nói "không chắc chắn".

3. **Observation**: Mỗi kết quả tool được nối vào running prompt (`agent.py:138`), tạo ra vòng phản hồi. Hai hành vi quan sát được: **(a)** khi tìm kiếm thất bại, `Thought` tiếp theo nhận ra lỗi và tự mở rộng query để thử lại; **(b)** khi lập kế hoạch ngân sách, agent nối chuỗi `web_search` (lấy giá) → `calculator[]` (tính tổng) — dạng multi-tool composition tuần tự không thể làm trong một lần gọi Chatbot.

---

## IV. Future Improvements (5 Points)

- **Scalability**: Các lần gọi tool hiện tại hoàn toàn tuần tự. Dùng `asyncio` để dispatch các lookup độc lập song song và gộp Observations trước bước LLM tiếp theo. Thêm TTL cache để tránh tìm kiếm trùng lặp trong cùng một phiên.

- **Safety**: Kết quả `web_search` được nối thẳng vào prompt — dễ bị tấn công prompt injection từ nội dung web độc hại. Giải pháp là thêm một Supervisor LLM để làm sạch Observations trước khi đưa vào context của agent.

- **Performance**: Với 50+ tool, việc nhúng toàn bộ mô tả vào prompt sẽ tốn nhiều token không cần thiết. Dùng vector database (ChromaDB/FAISS) để chỉ truy xuất top-k tool liên quan nhất với query, giữ prompt gọn nhẹ.

---

> [!NOTE]
> Submit this report by renaming it to `REPORT_[YOUR_NAME].md` and placing it in this folder.

