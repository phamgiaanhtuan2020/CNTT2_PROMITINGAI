---


---

<h2 id="so-sánh-2-giải-pháp-prompt-cho-bài-toán-xử-lý-đơn-hàng">So sánh 2 giải pháp prompt cho bài toán xử lý đơn hàng</h2>
<p><strong>Bối cảnh (giữ nguyên bài trước):</strong> Cửa hàng thời trang online nhận đơn qua Zalo/Facebook. Cần biến tin nhắn thô thành đơn hàng chuẩn, tự phát hiện lỗi dữ liệu và không đoán bừa.</p>
<p><strong>Dữ liệu cấu hình dùng chung cho cả 2 prompt:</strong></p>
<pre><code>NGÀY HIỆN TẠI: 17/11/2026
DANH MỤC:
- SP01 | Áo thun basic  | 150.000đ | tồn 10
- SP02 | Quần jean slim | 350.000đ | tồn 3
- SP03 | Nón lưỡi trai  | 90.000đ  | tồn 0
CHÍNH SÁCH:
- SALE10: giảm 10% tổng tiền hàng, đơn từ 500.000đ, hạn đến 30/11/2026
- Ship 30.000đ; miễn phí nếu tiền hàng sau giảm ≥ 600.000đ
</code></pre>
<hr>
<h3 id="bước-1-đề-xuất-2-prompt">Bước 1: Đề xuất 2 prompt</h3>
<h4 id="giải-pháp-1-đơn-giản-ngắn-gọn">Giải pháp 1: Đơn giản, ngắn gọn</h4>
<pre><code>Bạn là nhân viên bán hàng. Hãy đọc tin nhắn của khách, tính tổng tiền đơn hàng
theo danh mục và chính sách bên dưới, rồi viết tin nhắn xác nhận gửi khách.
Hãy suy nghĩ từng bước.

&lt;danh_muc_va_chinh_sach&gt;[dán cấu hình]&lt;/danh_muc_va_chinh_sach&gt;
&lt;tin_nhan&gt;[dán tin nhắn khách]&lt;/tin_nhan&gt;
</code></pre>
<p>Đặc điểm: khoảng 40 từ, chỉ có Chain-of-Thought ở dạng “suy nghĩ từng bước”, không có điều kiện xử lý lỗi và không ép định dạng đầu ra.</p>
<h4 id="giải-pháp-2-chi-tiết-nhiều-ràng-buộc">Giải pháp 2: Chi tiết, nhiều ràng buộc</h4>
<pre><code># VAI TRÒ
Bạn là nhân viên xử lý đơn hàng cẩn thận, chính xác về số học. Bạn thà báo lỗi
còn hơn đoán bừa.

# NHIỆM VỤ
Trích xuất đơn từ tin nhắn, kiểm tra dữ liệu, tính tiền, rồi soạn phản hồi cho khách.

# DỮ LIỆU
&lt;danh_muc_va_chinh_sach&gt;[dán cấu hình]&lt;/danh_muc_va_chinh_sach&gt;
&lt;tin_nhan&gt;[dán tin nhắn khách]&lt;/tin_nhan&gt;

# QUY TRÌNH SUY LUẬN (làm tuần tự, ghi vào "reasoning")
B1. Liệt kê thông tin tìm thấy: tên, SĐT, địa chỉ, sản phẩm, số lượng, mã giảm giá.
B2. Kiểm tra từng trường theo QUY TẮC LỖI.
B3. Với mỗi dòng hàng: qty_fulfillable = min(qty_yêu_cầu, tồn_kho); line_total = qty_fulfillable × đơn giá.
B4. subtotal → kiểm tra mã giảm giá (tồn tại? còn hạn? đủ mức tối thiểu?) → ship → total.
B5. Tự kiểm tra lại phép tính một lần trước khi xuất.

# QUY TẮC XỬ LÝ LỖI (bắt buộc)
- NẾU thiếu tên/SĐT/địa chỉ → null, thêm vào "issues", KHÔNG tự bịa.
- NẾU SĐT không đúng 10 chữ số hoặc không bắt đầu bằng 0 → phone_valid=false.
- NẾU địa chỉ thiếu tỉnh/thành → thêm vào "issues".
- NẾU sản phẩm không có trong danh mục → sku=null, thêm vào "issues".
- NẾU qty yêu cầu &gt; tồn kho → chỉ tính phần giao được, báo rõ, KHÔNG tự thay sản phẩm.
- NẾU mã giảm giá sai/hết hạn/chưa đủ điều kiện → discount=0 và nêu lý do.
- NẾU không đọc được sản phẩm nào → order_status="FAILED", chuyển nhân viên.
- NẾU có bất kỳ issue nào → order_status="NEEDS_CONFIRMATION", total là tạm tính,
  TUYỆT ĐỐI KHÔNG viết câu "đã chốt đơn".
- Chỉ "READY" khi không có lỗi nào.

# ĐỊNH DẠNG ĐẦU RA
Chỉ trả về JSON:
{ "reasoning": [...], "customer": {...}, "lines": [...], "subtotal", "discount",
  "shipping", "total", "total_is_provisional", "issues": [...], "order_status",
  "customer_message": "tối đa 100 từ, nêu từng vấn đề và hỏi lại đúng thứ cần xác nhận" }
</code></pre>
<hr>
<h3 id="bước-2-bảng-so-sánh-trade-off">Bước 2: Bảng so sánh Trade-off</h3>
<p>Tiêu chí</p>
<p>Giải pháp 1 (Đơn giản)</p>
<p>Giải pháp 2 (Chi tiết)</p>
<p>Độ dễ hiểu, dễ viết</p>
<p>Cao: viết trong 1 phút</p>
<p>Trung bình: cần hiểu nghiệp vụ để viết đủ quy tắc</p>
<p>Tốn token</p>
<p>Ít (~60 token prompt)</p>
<p>Nhiều hơn (~500 token prompt, nhân với số đơn/ngày)</p>
<p>Chặn lỗi biên (SĐT sai, hết hàng, thiếu địa chỉ)</p>
<p>Thấp: phụ thuộc “may rủi”</p>
<p>Cực tốt: mỗi bẫy có một câu IF-THEN</p>
<p>Độ chính xác phép tính</p>
<p>Trung bình: có CoT nhưng không có bước tự kiểm tra</p>
<p>Cao: có B3 (min với tồn kho) và B5 (tự kiểm tra lại)</p>
<p>Tính nhất quán giữa các lần chạy</p>
<p>Thấp: mỗi lần có thể ra định dạng khác nhau</p>
<p>Cao: JSON cố định, máy đọc được</p>
<p>Khả năng tích hợp tự động</p>
<p>Khó: đầu ra là văn bản tự do</p>
<p>Dễ: parse JSON, rẽ nhánh theo <code>order_status</code></p>
<p>Rủi ro nghiệp vụ</p>
<p><strong>Cao</strong>: có thể xác nhận đơn sai, báo giá sai</p>
<p>Thấp: sai thì bị gắn cờ cho nhân viên</p>
<p>Chi phí bảo trì</p>
<p>Thấp</p>
<p>Trung bình: đổi chính sách phải sửa quy tắc</p>
<p>Phù hợp khi</p>
<p>Thử nghiệm nhanh, dữ liệu sạch, người đọc kiểm lại</p>
<p>Chạy thật, đơn nhiều, dữ liệu bẩn</p>
<hr>
<h3 id="bước-3-chạy-thử-cả-hai-trên-cùng-một-ca-khó">Bước 3: Chạy thử cả hai trên cùng một ca khó</h3>
<p><strong>Tin nhắn test</strong> (chứa đủ 4 bẫy: SĐT thiếu số, địa chỉ thiếu thành phố, vượt tồn kho, hết hàng):</p>
<blockquote>
<p>chị lấy 2 áo thun, 4 quần jean với 1 nón nha shop. mã SALE10. giao về 25 Lê Lợi Q1. sdt 0901 234 56. Tên Lan</p>
</blockquote>
<h4 id="kết-quả-giải-pháp-1">Kết quả Giải pháp 1</h4>
<pre><code>Chào chị Lan! Shop đã nhận đơn của chị:
- 2 áo thun basic: 300.000đ
- 4 quần jean slim: 1.400.000đ
- 1 nón lưỡi trai: 90.000đ
Tổng tiền hàng: 1.790.000đ. Áp mã SALE10 giảm 179.000đ, miễn phí ship.
Tổng thanh toán: 1.611.000đ. Shop sẽ giao về 25 Lê Lợi, Q1 và liên hệ
chị qua số 0901 234 56. Cảm ơn chị đã ủng hộ shop!
</code></pre>
<p><strong>Đánh giá:</strong> Phép tính đúng theo dữ liệu AI hiểu, nhưng nghiệp vụ sai ở cả 4 bẫy:</p>
<ol>
<li>Không phát hiện quần jean chỉ còn 3 (đã báo bán 4).</li>
<li>Không phát hiện nón hết hàng (vẫn tính tiền).</li>
<li>Chấp nhận SĐT thiếu số và còn gửi lại nguyên số sai.</li>
<li>Không hỏi lại thành phố.</li>
</ol>
<p>Hậu quả thực tế là shop xác nhận đơn không giao được, báo giá sai 396.000đ (1.611.000đ so với 1.215.000đ), khách không liên lạc được và phải xử lý lại bằng tay.</p>
<h4 id="kết-quả-giải-pháp-2">Kết quả Giải pháp 2</h4>
<p>json</p>
<pre class=" language-json"><code class="prism  language-json"><span class="token punctuation">{</span>
  <span class="token string">"reasoning"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token string">"B1. Tên Lan; SĐT '0901 234 56'; địa chỉ '25 Lê Lợi Q1'; mã SALE10; 2 áo thun, 4 quần jean, 1 nón."</span><span class="token punctuation">,</span>
    <span class="token string">"B2. SĐT bỏ khoảng trắng = 090123456 (9 chữ số) → không hợp lệ. Địa chỉ thiếu tỉnh/thành."</span><span class="token punctuation">,</span>
    <span class="token string">"B3. SP01: min(2,10)=2 → 300.000. SP02: min(4,3)=3 → 1.050.000. SP03: min(1,0)=0 → 0."</span><span class="token punctuation">,</span>
    <span class="token string">"B4. subtotal=1.350.000. SALE10 tồn tại, còn hạn (30/11 ≥ 17/11), 1.350.000 ≥ 500.000 → giảm 135.000. Sau giảm 1.215.000 ≥ 600.000 → ship 0. total=1.215.000."</span><span class="token punctuation">,</span>
    <span class="token string">"B5. Kiểm tra: 300.000+1.050.000=1.350.000; ×0,9=1.215.000. Khớp."</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"customer"</span><span class="token punctuation">:</span> <span class="token punctuation">{</span><span class="token string">"name"</span><span class="token punctuation">:</span> <span class="token string">"Lan"</span><span class="token punctuation">,</span> <span class="token string">"phone_valid"</span><span class="token punctuation">:</span> <span class="token boolean">false</span><span class="token punctuation">,</span> <span class="token string">"address"</span><span class="token punctuation">:</span> <span class="token string">"25 Lê Lợi Q1"</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token string">"lines"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token punctuation">{</span><span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP01"</span><span class="token punctuation">,</span> <span class="token string">"qty_requested"</span><span class="token punctuation">:</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"qty_fulfillable"</span><span class="token punctuation">:</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"line_total"</span><span class="token punctuation">:</span> <span class="token number">300000</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
    <span class="token punctuation">{</span><span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP02"</span><span class="token punctuation">,</span> <span class="token string">"qty_requested"</span><span class="token punctuation">:</span> <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"qty_fulfillable"</span><span class="token punctuation">:</span> <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"line_total"</span><span class="token punctuation">:</span> <span class="token number">1050000</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
    <span class="token punctuation">{</span><span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP03"</span><span class="token punctuation">,</span> <span class="token string">"qty_requested"</span><span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"qty_fulfillable"</span><span class="token punctuation">:</span> <span class="token number">0</span><span class="token punctuation">,</span> <span class="token string">"line_total"</span><span class="token punctuation">:</span> <span class="token number">0</span><span class="token punctuation">}</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"subtotal"</span><span class="token punctuation">:</span> <span class="token number">1350000</span><span class="token punctuation">,</span>
  <span class="token string">"discount"</span><span class="token punctuation">:</span> <span class="token number">135000</span><span class="token punctuation">,</span>
  <span class="token string">"shipping"</span><span class="token punctuation">:</span> <span class="token number">0</span><span class="token punctuation">,</span>
  <span class="token string">"total"</span><span class="token punctuation">:</span> <span class="token number">1215000</span><span class="token punctuation">,</span>
  <span class="token string">"total_is_provisional"</span><span class="token punctuation">:</span> <span class="token boolean">true</span><span class="token punctuation">,</span>
  <span class="token string">"issues"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token string">"SĐT chỉ có 9 chữ số"</span><span class="token punctuation">,</span>
    <span class="token string">"Địa chỉ thiếu tỉnh/thành phố"</span><span class="token punctuation">,</span>
    <span class="token string">"Quần jean slim: yêu cầu 4, chỉ còn 3"</span><span class="token punctuation">,</span>
    <span class="token string">"Nón lưỡi trai: hết hàng"</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"order_status"</span><span class="token punctuation">:</span> <span class="token string">"NEEDS_CONFIRMATION"</span><span class="token punctuation">,</span>
  <span class="token string">"customer_message"</span><span class="token punctuation">:</span> <span class="token string">"Chào chị Lan, shop nhận được đơn rồi ạ. Chị giúp shop xác nhận: (1) SĐT đang thiếu 1 số, chị gửi lại giúp shop; (2) chị cho shop xin địa chỉ kèm thành phố; (3) quần jean chỉ còn 3 cái, chị lấy 3 được không ạ; (4) nón đã hết hàng, chị bỏ hay chọn mẫu khác ạ? Tạm tính 2 áo + 3 quần: 1.215.000đ (đã giảm SALE10, miễn ship). Shop chờ chị phản hồi để chốt đơn nhé!"</span>
<span class="token punctuation">}</span>
</code></pre>
<h4 id="bảng-kết-quả-thực-nghiệm">Bảng kết quả thực nghiệm</h4>
<p>Chỉ số</p>
<p>Giải pháp 1</p>
<p>Giải pháp 2</p>
<p>Bẫy bắt được</p>
<p>0/4</p>
<p>4/4</p>
<p>Tổng tiền</p>
<p>1.611.000đ (sai)</p>
<p>1.215.000đ (đúng, tạm tính)</p>
<p>Có xác nhận đơn sai không</p>
<p>Có</p>
<p>Không (<code>NEEDS_CONFIRMATION</code>)</p>
<p>Máy đọc được không</p>
<p>Không</p>
<p>Có (JSON)</p>
<hr>
<h3 id="kết-luận-chọn-giải-pháp-2">Kết luận: chọn Giải pháp 2</h3>
<p>Lý do chọn:</p>
<ol>
<li><strong>Chi phí lỗi lớn hơn chi phí token.</strong> Giải pháp 2 tốn thêm khoảng 450 token mỗi lần gọi, nhưng một đơn xác nhận sai (như ở Giải pháp 1) khiến shop mất tiền, mất uy tín và mất công xử lý lại. Với đơn hàng thật, an toàn quan trọng hơn tiết kiệm.</li>
<li><strong>Dữ liệu thực tế là dữ liệu bẩn.</strong> Khách nhắn tắt, thiếu, sai là chuyện thường ngày. Chỉ các câu IF-THEN mới buộc AI báo lỗi thay vì “cố gắng làm hài lòng”.</li>
<li><strong>Tự động hóa được.</strong> JSON cùng trường <code>order_status</code> cho phép hệ thống tự rẽ nhánh (<code>READY</code> → tạo đơn, <code>NEEDS_CONFIRMATION</code> → gửi tin hỏi lại, <code>FAILED</code> → chuyển nhân viên).</li>
</ol>
<p><strong>Điểm yếu của Giải pháp 2 và hướng khắc phục:</strong></p>
<ul>
<li>Prompt dài, khó bảo trì → tách danh mục và chính sách ra biến riêng để chỉ sửa dữ liệu, không sửa prompt.</li>
<li>Một prompt làm cả trích xuất, tính tiền và soạn tin → khi quy mô lớn, nên tách thành chuỗi 3 prompt (Prompt Chaining) như bài trước để dễ tìm lỗi từng bước.</li>
</ul>
<p><strong>Khi nào vẫn nên dùng Giải pháp 1:</strong> khi thử nghiệm ý tưởng, dữ liệu sạch và có người kiểm tra lại kết quả trước khi gửi khách.</p>
<p><strong>Lưu ý khi nộp bài:</strong> các kết quả trên do tôi tạo ra theo đúng từng prompt, minh họa đúng hành vi thường gặp. Kết quả thực tế của Giải pháp 1 có thể khác giữa các lần chạy, vì không có ràng buộc nên đôi khi AI tự hỏi lại. Bạn nên tự chạy cả 2 prompt trên công cụ AI của mình và dán kết quả thật vào bài. Bạn cũng có thể chạy thêm ca “mã giảm giá hết hạn” hoặc “tin nhắn không có sản phẩm” để so sánh thêm.</p>
<p>Nếu cần, tôi có thể xuất bài này thành file Word/PDF hoặc viết thêm 2-3 ca test khác để bạn đối chiếu.</p>

