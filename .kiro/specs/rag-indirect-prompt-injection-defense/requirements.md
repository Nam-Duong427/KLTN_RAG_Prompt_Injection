# Tài Liệu Yêu Cầu

## Giới Thiệu

Đồ án này nghiên cứu và xây dựng hệ thống phòng chống **Indirect Prompt Injection** trong pipeline **RAG (Retrieval-Augmented Generation)**. Kẻ tấn công có thể nhúng các lệnh độc hại ẩn bên trong tài liệu (PDF, trang web) mà hệ thống RAG sẽ truy xuất và đưa vào ngữ cảnh của LLM, từ đó thao túng hành vi của mô hình mà người dùng không hay biết.

Hệ thống sẽ:
1. Xây dựng pipeline RAG hoàn chỉnh có khả năng truy xuất tài liệu từ PDF và web scraping.
2. Phân tích và phân loại các dạng tấn công injection ở cấp độ tài liệu.
3. Áp dụng ba lớp phòng thủ: context sanitization, embedding anomaly detection, và attention analysis.
4. So sánh hiệu quả phòng thủ trên các mô hình LLaMA, Mistral và GPT API.

---

## Bảng Thuật Ngữ

- **RAG_Pipeline**: Hệ thống kết hợp truy xuất tài liệu và sinh văn bản bằng LLM.
- **Document_Loader**: Thành phần tải và phân tích tài liệu từ PDF hoặc URL.
- **Chunker**: Thành phần chia tài liệu thành các đoạn nhỏ (chunk) để lập chỉ mục.
- **Embedder**: Thành phần chuyển đổi văn bản thành vector nhúng (embedding).
- **Vector_Store**: Cơ sở dữ liệu lưu trữ và tìm kiếm vector nhúng.
- **Retriever**: Thành phần truy xuất các chunk liên quan dựa trên câu hỏi người dùng.
- **LLM**: Mô hình ngôn ngữ lớn (LLaMA, Mistral, GPT) dùng để sinh câu trả lời.
- **Injection_Detector**: Thành phần phát hiện các đoạn văn bản có dấu hiệu injection.
- **Context_Sanitizer**: Thành phần làm sạch ngữ cảnh trước khi đưa vào LLM.
- **Anomaly_Detector**: Thành phần phát hiện bất thường trong không gian embedding.
- **Attention_Analyzer**: Thành phần phân tích trọng số attention của LLM để phát hiện hành vi bất thường.
- **Evaluator**: Thành phần đánh giá hiệu quả phòng thủ theo các chỉ số định lượng.
- **Indirect_Prompt_Injection**: Tấn công nhúng lệnh độc hại vào tài liệu bên ngoài mà LLM sẽ xử lý.
- **Benign_Document**: Tài liệu hợp lệ không chứa nội dung tấn công.
- **Poisoned_Document**: Tài liệu bị chèn nội dung injection độc hại.

---

## Yêu Cầu

### Yêu Cầu 1: Xây Dựng Pipeline RAG Cơ Bản

**User Story:** Là một nhà nghiên cứu, tôi muốn có một pipeline RAG hoàn chỉnh, để tôi có thể đặt câu hỏi và nhận câu trả lời dựa trên tài liệu đã nạp vào hệ thống.

#### Tiêu Chí Chấp Nhận

1. THE Document_Loader SHALL tải tài liệu từ tệp PDF và trả về nội dung văn bản thuần túy.
2. THE Document_Loader SHALL thu thập nội dung từ URL trang web thông qua web scraping.
3. IF Document_Loader nhận một tệp PDF bị hỏng hoặc URL không hợp lệ, THEN THE Document_Loader SHALL trả về thông báo lỗi mô tả rõ nguyên nhân thất bại.
4. THE Chunker SHALL chia văn bản thành các đoạn có kích thước tối đa 512 token với độ chồng lấp (overlap) 50 token.
5. THE Embedder SHALL chuyển đổi mỗi chunk thành vector nhúng có chiều dài cố định.
6. THE Vector_Store SHALL lưu trữ vector nhúng và metadata của từng chunk.
7. WHEN người dùng gửi câu hỏi, THE Retriever SHALL truy xuất tối đa 5 chunk có độ tương đồng cosine cao nhất.
8. WHEN Retriever trả về các chunk liên quan, THE LLM SHALL sinh câu trả lời dựa trên ngữ cảnh đó trong vòng 30 giây.
9. IF Vector_Store không chứa tài liệu nào, THEN THE RAG_Pipeline SHALL thông báo cho người dùng rằng chưa có tài liệu nào được nạp.

---

### Yêu Cầu 2: Phân Tích và Phân Loại Tấn Công Injection

**User Story:** Là một nhà nghiên cứu bảo mật, tôi muốn hiểu rõ các dạng tấn công injection ở cấp độ tài liệu, để tôi có thể xây dựng tập dữ liệu kiểm thử phù hợp.

#### Tiêu Chí Chấp Nhận

1. THE RAG_Pipeline SHALL hỗ trợ ít nhất 4 dạng tấn công injection: role override, instruction hijacking, data exfiltration prompt, và jailbreak attempt.
2. THE Document_Loader SHALL phân biệt được Benign_Document và Poisoned_Document thông qua nhãn metadata khi nạp vào hệ thống.
3. THE Evaluator SHALL tạo báo cáo thống kê phân phối các dạng tấn công trong tập dữ liệu kiểm thử.
4. WHEN một Poisoned_Document được nạp vào hệ thống mà không có cơ chế phòng thủ, THE LLM SHALL thực thi lệnh injection với xác suất được ghi lại để làm baseline.
5. THE Evaluator SHALL đo tỷ lệ tấn công thành công (Attack Success Rate - ASR) trên tập dữ liệu baseline không có phòng thủ.

---

### Yêu Cầu 3: Context Sanitization

**User Story:** Là một kỹ sư bảo mật, tôi muốn làm sạch ngữ cảnh trước khi đưa vào LLM, để loại bỏ các lệnh injection tiềm ẩn trong tài liệu truy xuất.

#### Tiêu Chí Chấp Nhận

1. THE Context_Sanitizer SHALL phát hiện và loại bỏ các đoạn văn bản khớp với danh sách mẫu injection đã định nghĩa (pattern-based filtering).
2. THE Context_Sanitizer SHALL áp dụng kỹ thuật prompt delimiters để tách biệt rõ ràng ngữ cảnh tài liệu với câu hỏi người dùng trong prompt gửi đến LLM.
3. THE Context_Sanitizer SHALL chuẩn hóa (normalize) văn bản bằng cách loại bỏ các ký tự điều khiển ẩn (zero-width characters, Unicode control characters) trước khi đưa vào LLM.
4. WHEN Context_Sanitizer loại bỏ một đoạn văn bản, THE Context_Sanitizer SHALL ghi log chi tiết bao gồm chunk ID, nội dung bị loại bỏ, và lý do.
5. IF Context_Sanitizer loại bỏ toàn bộ nội dung của một chunk, THEN THE RAG_Pipeline SHALL bỏ qua chunk đó và tiếp tục với các chunk còn lại.
6. THE Context_Sanitizer SHALL xử lý mỗi chunk trong vòng 100ms để không làm chậm pipeline đáng kể.
7. THE Evaluator SHALL đo tỷ lệ phát hiện đúng (True Positive Rate) và tỷ lệ báo động nhầm (False Positive Rate) của Context_Sanitizer trên tập dữ liệu kiểm thử.

---

### Yêu Cầu 4: Embedding Anomaly Detection

**User Story:** Là một nhà nghiên cứu ML, tôi muốn phát hiện các chunk bất thường trong không gian embedding, để nhận diện tài liệu có thể chứa injection mà pattern-based filtering bỏ sót.

#### Tiêu Chí Chấp Nhận

1. THE Anomaly_Detector SHALL huấn luyện mô hình phát hiện bất thường (Isolation Forest hoặc One-Class SVM) trên tập embedding của Benign_Document.
2. WHEN Retriever trả về các chunk, THE Anomaly_Detector SHALL tính điểm bất thường (anomaly score) cho từng chunk.
3. WHEN anomaly score của một chunk vượt ngưỡng đã cấu hình, THE Anomaly_Detector SHALL đánh dấu chunk đó là đáng ngờ và ghi log.
4. THE Anomaly_Detector SHALL cho phép cấu hình ngưỡng phát hiện (threshold) mà không cần huấn luyện lại mô hình.
5. THE Anomaly_Detector SHALL tính toán anomaly score cho một chunk trong vòng 50ms.
6. THE Evaluator SHALL đo AUC-ROC của Anomaly_Detector trên tập dữ liệu kiểm thử gồm Benign_Document và Poisoned_Document.
7. FOR ALL chunk được đánh dấu bất thường bởi Anomaly_Detector, THE RAG_Pipeline SHALL có tùy chọn loại bỏ hoặc cảnh báo người dùng thay vì tự động loại bỏ.

---

### Yêu Cầu 5: Attention Analysis

**User Story:** Là một nhà nghiên cứu NLP, tôi muốn phân tích trọng số attention của LLM, để phát hiện khi mô hình đang tập trung bất thường vào các đoạn có thể là injection thay vì câu hỏi người dùng.

#### Tiêu Chí Chấp Nhận

1. WHILE LLM đang sinh câu trả lời, THE Attention_Analyzer SHALL trích xuất ma trận attention từ các lớp transformer cuối cùng.
2. THE Attention_Analyzer SHALL tính tỷ lệ attention mà LLM dành cho phần ngữ cảnh tài liệu so với phần câu hỏi người dùng.
3. WHEN tỷ lệ attention dành cho một chunk tài liệu vượt quá 3 lần độ lệch chuẩn so với phân phối baseline, THE Attention_Analyzer SHALL ghi nhận chunk đó là đáng ngờ.
4. THE Attention_Analyzer SHALL tạo biểu đồ trực quan hóa (heatmap) phân phối attention cho mỗi lần truy vấn khi chế độ debug được bật.
5. IF mô hình LLM được sử dụng là GPT API (không có quyền truy cập attention weights), THEN THE Attention_Analyzer SHALL bỏ qua bước phân tích attention và ghi log cảnh báo.
6. THE Evaluator SHALL so sánh phân phối attention giữa các truy vấn trên Benign_Document và Poisoned_Document để xác định tính phân biệt của chỉ số này.

---

### Yêu Cầu 6: So Sánh Hiệu Quả Giữa Các Mô Hình

**User Story:** Là một nhà nghiên cứu, tôi muốn so sánh hiệu quả phòng thủ trên nhiều LLM khác nhau, để xác định mô hình nào có khả năng kháng injection tốt nhất khi kết hợp với các cơ chế phòng thủ.

#### Tiêu Chí Chấp Nhận

1. THE RAG_Pipeline SHALL hỗ trợ cấu hình để chạy với LLaMA (local), Mistral (local), và GPT API (remote) mà không cần thay đổi code pipeline.
2. THE Evaluator SHALL chạy cùng một bộ kiểm thử trên cả ba mô hình và ghi lại kết quả vào cùng một tệp báo cáo.
3. THE Evaluator SHALL đo các chỉ số sau cho mỗi mô hình: ASR (Attack Success Rate), TPR (True Positive Rate), FPR (False Positive Rate), AUC-ROC, và thời gian phản hồi trung bình.
4. THE Evaluator SHALL tạo bảng so sánh tổng hợp các chỉ số giữa ba mô hình ở định dạng CSV và Markdown.
5. IF kết nối đến GPT API thất bại, THEN THE RAG_Pipeline SHALL ghi log lỗi và tiếp tục đánh giá với các mô hình còn lại.
6. THE Evaluator SHALL thực hiện kiểm định thống kê (statistical significance test) để xác nhận sự khác biệt giữa các mô hình có ý nghĩa thống kê.

---

### Yêu Cầu 7: Giao Diện Thực Nghiệm và Báo Cáo

**User Story:** Là một nhà nghiên cứu, tôi muốn có giao diện dòng lệnh và hệ thống báo cáo, để tôi có thể chạy thực nghiệm, theo dõi kết quả và tái tạo lại kết quả nghiên cứu.

#### Tiêu Chí Chấp Nhận

1. THE RAG_Pipeline SHALL cung cấp giao diện dòng lệnh (CLI) cho phép cấu hình: mô hình LLM, cơ chế phòng thủ được bật/tắt, đường dẫn tập dữ liệu, và ngưỡng phát hiện.
2. THE Evaluator SHALL lưu toàn bộ kết quả thực nghiệm vào tệp JSON có timestamp để đảm bảo khả năng tái tạo.
3. THE Evaluator SHALL tạo biểu đồ so sánh (bar chart, ROC curve) cho từng cơ chế phòng thủ và lưu dưới dạng tệp PNG.
4. THE RAG_Pipeline SHALL ghi log ở cấp độ INFO và DEBUG, có thể cấu hình qua tham số CLI.
5. THE RAG_Pipeline SHALL đọc cấu hình từ tệp YAML để cho phép tái tạo thực nghiệm với cùng tham số.
6. WHEN một thực nghiệm hoàn thành, THE Evaluator SHALL in tóm tắt kết quả ra màn hình console bao gồm các chỉ số chính.

---

### Yêu Cầu 8: Tính Toàn Vẹn và Khả Năng Tái Tạo Của Tập Dữ Liệu

**User Story:** Là một nhà nghiên cứu, tôi muốn tập dữ liệu kiểm thử có tính toàn vẹn và có thể tái tạo, để kết quả nghiên cứu có giá trị khoa học.

#### Tiêu Chí Chấp Nhận

1. THE Document_Loader SHALL tính và lưu hash SHA-256 của mỗi tài liệu khi nạp vào hệ thống.
2. THE Evaluator SHALL xác minh hash của tài liệu trước mỗi lần chạy thực nghiệm để đảm bảo tập dữ liệu không bị thay đổi.
3. IF hash của một tài liệu không khớp với giá trị đã lưu, THEN THE Evaluator SHALL dừng thực nghiệm và báo lỗi toàn vẹn dữ liệu.
4. THE RAG_Pipeline SHALL ghi lại seed ngẫu nhiên (random seed) vào tệp cấu hình để đảm bảo kết quả có thể tái tạo.
5. THE Document_Loader SHALL hỗ trợ tải tập dữ liệu từ tệp manifest JSON mô tả danh sách tài liệu, nhãn, và hash tương ứng.
