---


---

<h3 id="bước-1-thiết-kế-giải-pháp-inputoutput">Bước 1: Thiết kế giải pháp (Input/Output)</h3>
<h4 id="input">1.1. Input</h4>
<p>Nhóm</p>
<p>Nội dung</p>
<p>Nguồn</p>
<p>Tin nhắn khách</p>
<p>Văn bản tự do (tên, SĐT, địa chỉ, sản phẩm, số lượng, mã giảm giá)</p>
<p>Chat</p>
<p>Danh mục sản phẩm</p>
<p>SKU, tên, giá, tồn kho</p>
<p>Hệ thống kho</p>
<p>Chính sách</p>
<p>Mã giảm giá (điều kiện, hạn), phí ship, ngày hiện tại</p>
<p>Cấu hình cửa hàng</p>
<h4 id="sơ-đồ-chuỗi-prompt-prompt-chaining">1.2. Sơ đồ chuỗi prompt (Prompt Chaining)</h4>
<pre><code>Tin nhắn khách
   │
   ▼
[P1] Trích xuất + kiểm tra dữ liệu ──► JSON_1 (có trường status)
   │  nếu status = FAILED → DỪNG, chuyển nhân viên
   ▼
[P2] Tính toán đơn (CoT: tồn kho → khuyến mãi → ship → tổng) ──► JSON_2
   │
   ▼
[P3] Soạn tin nhắn phản hồi khách + ghi chú cho nhân viên
</code></pre>
<p>Mỗi bước chỉ làm một việc và đầu ra của bước trước là đầu vào của bước sau. Cách này dễ kiểm tra lỗi hơn một prompt khổng lồ.</p>
<h4 id="output-mong-muốn">1.3. Output mong muốn</h4>
<ul>
<li><strong>P1:</strong> JSON gồm <code>customer_name</code>, <code>phone</code>, <code>address</code>, <code>items[]</code>, <code>coupon</code>, <code>errors[]</code>, <code>status</code> (<code>OK</code> / <code>PARTIAL</code> / <code>FAILED</code>)</li>
<li><strong>P2:</strong> JSON gồm từng dòng hàng (<code>qty_requested</code>, <code>qty_fulfillable</code>, <code>line_total</code>), <code>subtotal</code>, <code>discount</code>, <code>shipping</code>, <code>total</code>, <code>order_status</code> (<code>READY</code> / <code>NEEDS_CONFIRMATION</code> / <code>REJECTED</code>)</li>
<li><strong>P3:</strong> Tin nhắn tiếng Việt thân thiện, nêu rõ vấn đề và hỏi lại đúng thứ cần xác nhận</li>
</ul>
<h4 id="bẫy-dữ-liệu-và-cách-xử-lý-đưa-vào-prompt-dưới-dạng-if-then">1.4. Bẫy dữ liệu và cách xử lý (đưa vào prompt dưới dạng IF-THEN)</h4>
<p>Bẫy</p>
<p>Xử lý</p>
<p>Thiếu tên/SĐT/địa chỉ</p>
<p>Ghi <code>null</code>, thêm vào <code>errors</code>, không tự bịa</p>
<p>SĐT sai định dạng (không đủ 10 số, không bắt đầu bằng 0)</p>
<p><code>valid: false</code>, yêu cầu khách gửi lại</p>
<p>Sản phẩm không có trong danh mục</p>
<p><code>sku: null</code>, hỏi lại khách</p>
<p>Số lượng vượt tồn kho / hết hàng</p>
<p>Không tự đổi đơn, báo <code>qty_fulfillable</code> và hỏi khách</p>
<p>Mã giảm giá sai, hết hạn, chưa đủ điều kiện</p>
<p>Không áp dụng, giải thích lý do</p>
<p>Đơn hoàn toàn không đọc được</p>
<p><code>FAILED</code>, dừng chuỗi</p>
<hr>
<h3 id="bước-2-prompt-hoàn-chỉnh-và-chạy-thử">Bước 2: Prompt hoàn chỉnh và chạy thử</h3>
<h4 id="dữ-liệu-cấu-hình-dùng-chung">Dữ liệu cấu hình dùng chung</h4>
<pre><code>NGÀY HIỆN TẠI: 17/11/2026

DANH MỤC:
- SP01 | Áo thun basic   | 150.000đ | tồn 10
- SP02 | Quần jean slim  | 350.000đ | tồn 3
- SP03 | Nón lưỡi trai   | 90.000đ  | tồn 0

CHÍNH SÁCH:
- SALE10: giảm 10% tổng tiền hàng, áp dụng cho đơn từ 500.000đ, hạn dùng đến 30/11/2026
- Phí ship 30.000đ; miễn phí nếu tổng tiền hàng sau giảm giá ≥ 600.000đ
</code></pre>
<h4 id="prompt-1-trích-xuất-và-kiểm-tra">Prompt 1: Trích xuất và kiểm tra</h4>
<pre><code># VAI TRÒ
Bạn là nhân viên nhập liệu đơn hàng cẩn thận của cửa hàng thời trang online.

# NHIỆM VỤ
Đọc tin nhắn của khách và trích xuất thành đơn hàng có cấu trúc, đối chiếu với DANH MỤC.

# DỮ LIỆU
&lt;danh_muc&gt;[dán danh mục]&lt;/danh_muc&gt;
&lt;tin_nhan&gt;
chị lấy 2 áo thun, 4 quần jean với 1 nón nha shop. mã SALE10. giao về 25 Lê Lợi Q1. sdt 0901 234 56. Tên Lan
&lt;/tin_nhan&gt;

# QUY TRÌNH (suy luận từng bước, ghi vào trường "reasoning")
1. Liệt kê từng thông tin tìm thấy trong tin nhắn.
2. Đối chiếu từng sản phẩm với danh mục để lấy SKU.
3. Kiểm tra từng trường theo QUY TẮC LỖI.
4. Chốt status.

# QUY TRÌNH XỬ LÝ LỖI (bắt buộc)
- NẾU thiếu tên / SĐT / địa chỉ → đặt giá trị null và thêm lỗi vào "errors". TUYỆT ĐỐI không tự suy đoán hay bịa.
- NẾU SĐT không đúng 10 chữ số hoặc không bắt đầu bằng 0 → valid=false, normalized=null.
- NẾU địa chỉ thiếu tỉnh/thành → valid=true nhưng thêm "warning".
- NẾU tên sản phẩm không khớp danh mục → sku=null, thêm lỗi.
- NẾU không có sản phẩm nào đọc được → status="FAILED".
- NẾU có lỗi nhưng vẫn còn sản phẩm hợp lệ → status="PARTIAL". Không có lỗi → "OK".
- KHÔNG xử lý tồn kho hay giá ở bước này.

# ĐỊNH DẠNG ĐẦU RA
Chỉ trả về JSON hợp lệ, không thêm văn bản ngoài JSON:
{ "reasoning": [...], "customer_name": ..., "phone": {"raw","normalized","valid","reason"},
  "address": {"raw","valid","warning"}, "items": [{"raw","sku","qty"}],
  "coupon": ..., "errors": [...], "status": ... }
</code></pre>
<p><strong>Kết quả AI trả về (P1):</strong></p>
<p>json</p>
<pre class=" language-json"><code class="prism  language-json"><span class="token punctuation">{</span>
  <span class="token string">"reasoning"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token string">"Tìm thấy: tên Lan; SĐT '0901 234 56'; địa chỉ '25 Lê Lợi Q1'; mã SALE10; 3 mặt hàng."</span><span class="token punctuation">,</span>
    <span class="token string">"'áo thun' → SP01; 'quần jean' → SP02; 'nón' → SP03."</span><span class="token punctuation">,</span>
    <span class="token string">"SĐT sau khi bỏ khoảng trắng là 090123456 = 9 chữ số, thiếu 1 số so với chuẩn 10 số."</span><span class="token punctuation">,</span>
    <span class="token string">"Địa chỉ có số nhà, đường, quận nhưng không ghi thành phố."</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"customer_name"</span><span class="token punctuation">:</span> <span class="token string">"Lan"</span><span class="token punctuation">,</span>
  <span class="token string">"phone"</span><span class="token punctuation">:</span> <span class="token punctuation">{</span>
    <span class="token string">"raw"</span><span class="token punctuation">:</span> <span class="token string">"0901 234 56"</span><span class="token punctuation">,</span>
    <span class="token string">"normalized"</span><span class="token punctuation">:</span> <span class="token keyword">null</span><span class="token punctuation">,</span>
    <span class="token string">"valid"</span><span class="token punctuation">:</span> <span class="token boolean">false</span><span class="token punctuation">,</span>
    <span class="token string">"reason"</span><span class="token punctuation">:</span> <span class="token string">"Chỉ có 9 chữ số, cần 10 chữ số"</span>
  <span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token string">"address"</span><span class="token punctuation">:</span> <span class="token punctuation">{</span>
    <span class="token string">"raw"</span><span class="token punctuation">:</span> <span class="token string">"25 Lê Lợi Q1"</span><span class="token punctuation">,</span>
    <span class="token string">"valid"</span><span class="token punctuation">:</span> <span class="token boolean">true</span><span class="token punctuation">,</span>
    <span class="token string">"warning"</span><span class="token punctuation">:</span> <span class="token string">"Thiếu tỉnh/thành phố, cần khách xác nhận"</span>
  <span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token string">"items"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token punctuation">{</span><span class="token string">"raw"</span><span class="token punctuation">:</span> <span class="token string">"áo thun"</span><span class="token punctuation">,</span> <span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP01"</span><span class="token punctuation">,</span> <span class="token string">"qty"</span><span class="token punctuation">:</span> <span class="token number">2</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
    <span class="token punctuation">{</span><span class="token string">"raw"</span><span class="token punctuation">:</span> <span class="token string">"quần jean"</span><span class="token punctuation">,</span> <span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP02"</span><span class="token punctuation">,</span> <span class="token string">"qty"</span><span class="token punctuation">:</span> <span class="token number">4</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
    <span class="token punctuation">{</span><span class="token string">"raw"</span><span class="token punctuation">:</span> <span class="token string">"nón"</span><span class="token punctuation">,</span> <span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP03"</span><span class="token punctuation">,</span> <span class="token string">"qty"</span><span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">}</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"coupon"</span><span class="token punctuation">:</span> <span class="token string">"SALE10"</span><span class="token punctuation">,</span>
  <span class="token string">"errors"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token string">"SĐT không hợp lệ (9 chữ số)"</span><span class="token punctuation">,</span>
    <span class="token string">"Địa chỉ thiếu tỉnh/thành phố"</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"status"</span><span class="token punctuation">:</span> <span class="token string">"PARTIAL"</span>
<span class="token punctuation">}</span>
</code></pre>
<p>→ <code>status = PARTIAL</code> nên chuỗi tiếp tục sang P2.</p>
<h4 id="prompt-2-tính-toán-đơn-hàng-chain-of-thought">Prompt 2: Tính toán đơn hàng (Chain-of-Thought)</h4>
<pre><code># VAI TRÒ
Bạn là kế toán kho của cửa hàng, chính xác tuyệt đối về số học.

# NHIỆM VỤ
Dựa trên JSON_1 (đơn đã trích xuất), DANH MỤC và CHÍNH SÁCH, tính toán đơn hàng.

# DỮ LIỆU
&lt;json_1&gt;[dán kết quả P1]&lt;/json_1&gt;
&lt;danh_muc_va_chinh_sach&gt;[dán cấu hình]&lt;/danh_muc_va_chinh_sach&gt;

# QUY TRÌNH SUY LUẬN (làm tuần tự, ghi từng bước vào "reasoning")
Bước A. Với mỗi dòng hàng: qty_fulfillable = min(qty_yêu_cầu, tồn_kho). Tính line_total = qty_fulfillable × đơn giá.
Bước B. subtotal = tổng các line_total.
Bước C. Kiểm tra mã giảm giá: (1) mã tồn tại? (2) còn hạn so với ngày hiện tại? (3) subtotal đạt mức tối thiểu? Chỉ áp dụng khi cả 3 đều đúng; nếu không, discount=0 và nêu lý do.
Bước D. Tính ship dựa trên (subtotal − discount).
Bước E. total = subtotal − discount + shipping.
Bước F. Tự kiểm tra lại phép tính một lần nữa trước khi xuất.

# QUY TRÌNH XỬ LÝ LỖI
- NẾU qty_fulfillable &lt; qty yêu cầu → thêm cảnh báo, KHÔNG tự ý thay thế sản phẩm hay chốt đơn.
- NẾU JSON_1.errors không rỗng hoặc có dòng thiếu hàng → order_status="NEEDS_CONFIRMATION" và ghi rõ "total" chỉ là tạm tính.
- NẾU không dòng nào có qty_fulfillable &gt; 0 → order_status="REJECTED".
- Chỉ "READY" khi không có lỗi nào.

# ĐỊNH DẠNG ĐẦU RA
Chỉ trả về JSON: { "reasoning": [...], "lines": [{"sku","qty_requested","qty_fulfillable","unit_price","line_total","note"}],
 "subtotal", "coupon_result": {"code","applied","discount","reason"}, "shipping", "total", "total_is_provisional", "order_status", "issues": [...] }
</code></pre>
<p><strong>Kết quả AI trả về (P2):</strong></p>
<p>json</p>
<pre class=" language-json"><code class="prism  language-json"><span class="token punctuation">{</span>
  <span class="token string">"reasoning"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token string">"A. SP01: yêu cầu 2, tồn 10 → giao 2 → 2 × 150.000 = 300.000."</span><span class="token punctuation">,</span>
    <span class="token string">"A. SP02: yêu cầu 4, tồn 3 → giao tối đa 3 → 3 × 350.000 = 1.050.000."</span><span class="token punctuation">,</span>
    <span class="token string">"A. SP03: yêu cầu 1, tồn 0 → giao 0 → 0."</span><span class="token punctuation">,</span>
    <span class="token string">"B. subtotal = 300.000 + 1.050.000 + 0 = 1.350.000."</span><span class="token punctuation">,</span>
    <span class="token string">"C. SALE10 tồn tại; hạn 30/11/2026 ≥ 17/11/2026 nên còn hạn; 1.350.000 ≥ 500.000 nên đạt điều kiện → áp dụng. discount = 10% × 1.350.000 = 135.000."</span><span class="token punctuation">,</span>
    <span class="token string">"D. Tiền hàng sau giảm = 1.215.000 ≥ 600.000 → miễn phí ship."</span><span class="token punctuation">,</span>
    <span class="token string">"E. total = 1.350.000 − 135.000 + 0 = 1.215.000."</span><span class="token punctuation">,</span>
    <span class="token string">"F. Kiểm tra lại: 300.000 + 1.050.000 = 1.350.000; 1.350.000 × 0,9 = 1.215.000. Khớp."</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"lines"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token punctuation">{</span><span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP01"</span><span class="token punctuation">,</span> <span class="token string">"qty_requested"</span><span class="token punctuation">:</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"qty_fulfillable"</span><span class="token punctuation">:</span> <span class="token number">2</span><span class="token punctuation">,</span> <span class="token string">"unit_price"</span><span class="token punctuation">:</span> <span class="token number">150000</span><span class="token punctuation">,</span> <span class="token string">"line_total"</span><span class="token punctuation">:</span> <span class="token number">300000</span><span class="token punctuation">,</span> <span class="token string">"note"</span><span class="token punctuation">:</span> <span class="token string">""</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
    <span class="token punctuation">{</span><span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP02"</span><span class="token punctuation">,</span> <span class="token string">"qty_requested"</span><span class="token punctuation">:</span> <span class="token number">4</span><span class="token punctuation">,</span> <span class="token string">"qty_fulfillable"</span><span class="token punctuation">:</span> <span class="token number">3</span><span class="token punctuation">,</span> <span class="token string">"unit_price"</span><span class="token punctuation">:</span> <span class="token number">350000</span><span class="token punctuation">,</span> <span class="token string">"line_total"</span><span class="token punctuation">:</span> <span class="token number">1050000</span><span class="token punctuation">,</span> <span class="token string">"note"</span><span class="token punctuation">:</span> <span class="token string">"Chỉ còn 3 sản phẩm"</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
    <span class="token punctuation">{</span><span class="token string">"sku"</span><span class="token punctuation">:</span> <span class="token string">"SP03"</span><span class="token punctuation">,</span> <span class="token string">"qty_requested"</span><span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">,</span> <span class="token string">"qty_fulfillable"</span><span class="token punctuation">:</span> <span class="token number">0</span><span class="token punctuation">,</span> <span class="token string">"unit_price"</span><span class="token punctuation">:</span> <span class="token number">90000</span><span class="token punctuation">,</span> <span class="token string">"line_total"</span><span class="token punctuation">:</span> <span class="token number">0</span><span class="token punctuation">,</span> <span class="token string">"note"</span><span class="token punctuation">:</span> <span class="token string">"Hết hàng"</span><span class="token punctuation">}</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  <span class="token string">"subtotal"</span><span class="token punctuation">:</span> <span class="token number">1350000</span><span class="token punctuation">,</span>
  <span class="token string">"coupon_result"</span><span class="token punctuation">:</span> <span class="token punctuation">{</span><span class="token string">"code"</span><span class="token punctuation">:</span> <span class="token string">"SALE10"</span><span class="token punctuation">,</span> <span class="token string">"applied"</span><span class="token punctuation">:</span> <span class="token boolean">true</span><span class="token punctuation">,</span> <span class="token string">"discount"</span><span class="token punctuation">:</span> <span class="token number">135000</span><span class="token punctuation">,</span> <span class="token string">"reason"</span><span class="token punctuation">:</span> <span class="token string">"Hợp lệ"</span><span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token string">"shipping"</span><span class="token punctuation">:</span> <span class="token number">0</span><span class="token punctuation">,</span>
  <span class="token string">"total"</span><span class="token punctuation">:</span> <span class="token number">1215000</span><span class="token punctuation">,</span>
  <span class="token string">"total_is_provisional"</span><span class="token punctuation">:</span> <span class="token boolean">true</span><span class="token punctuation">,</span>
  <span class="token string">"order_status"</span><span class="token punctuation">:</span> <span class="token string">"NEEDS_CONFIRMATION"</span><span class="token punctuation">,</span>
  <span class="token string">"issues"</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>
    <span class="token string">"SĐT không hợp lệ (9 chữ số)"</span><span class="token punctuation">,</span>
    <span class="token string">"Địa chỉ thiếu tỉnh/thành phố"</span><span class="token punctuation">,</span>
    <span class="token string">"Quần jean slim: yêu cầu 4, chỉ còn 3"</span><span class="token punctuation">,</span>
    <span class="token string">"Nón lưỡi trai: hết hàng"</span>
  <span class="token punctuation">]</span>
<span class="token punctuation">}</span>
</code></pre>
<h4 id="prompt-3-soạn-phản-hồi-cho-khách">Prompt 3: Soạn phản hồi cho khách</h4>
<pre><code># VAI TRÒ
Bạn là nhân viên chăm sóc khách hàng thân thiện, xưng "shop", gọi khách theo tên.

# NHIỆM VỤ
Dựa trên JSON_2, viết tin nhắn Zalo gửi khách và một ghi chú nội bộ cho nhân viên.

# DỮ LIỆU
&lt;json_2&gt;[dán kết quả P2]&lt;/json_2&gt;

# QUY TẮC
- NẾU order_status = "READY" → xác nhận đơn, liệt kê sản phẩm và tổng tiền.
- NẾU "NEEDS_CONFIRMATION" → KHÔNG xác nhận đơn. Nêu từng vấn đề trong "issues" và hỏi đúng thông tin cần bổ sung. Báo giá là tạm tính.
- NẾU "REJECTED" → xin lỗi, gợi ý khách chọn sản phẩm khác.
- Chỉ dùng số liệu có trong JSON_2, không tự tính lại hay thêm khuyến mãi.
- Tối đa 120 từ, không dùng thuật ngữ kỹ thuật.

# ĐỊNH DẠNG ĐẦU RA
[TIN NHẮN KHÁCH]
...
[GHI CHÚ NỘI BỘ]
(1-2 dòng: việc nhân viên cần làm)
</code></pre>
<p><strong>Kết quả AI trả về (P3):</strong></p>
<pre><code>[TIN NHẮN KHÁCH]
Chào chị Lan ạ, shop đã nhận đơn của chị. Chị giúp shop xác nhận vài điều nhé:

1. Số điện thoại chị gửi (0901 234 56) đang thiếu 1 số, chị gửi lại giúp shop ạ.
2. Chị cho shop xin địa chỉ đầy đủ kèm thành phố (hiện là 25 Lê Lợi, Q1) nhé.
3. Quần jean slim hiện chỉ còn 3 cái (chị đặt 4), chị lấy 3 cái được không ạ?
4. Nón lưỡi trai hiện đã hết hàng, chị muốn bỏ hay chọn mẫu khác ạ?

Tạm tính nếu giữ 2 áo thun + 3 quần jean: 1.350.000đ, đã áp mã SALE10 giảm 135.000đ, miễn phí ship, tổng 1.215.000đ. Chị phản hồi để shop chốt đơn nhé!

[GHI CHÚ NỘI BỘ]
Chưa chốt đơn, chờ khách xác nhận SĐT, địa chỉ đầy đủ, số lượng quần jean và phương án thay nón. Chốt xong cần tính lại tổng tiền nếu số lượng thay đổi.
</code></pre>
<hr>
<h3 id="nhận-xét-các-kỹ-thuật-đã-dùng">Nhận xét: các kỹ thuật đã dùng</h3>
<ul>
<li><strong>Chain-of-Thought:</strong> trường <code>reasoning</code> buộc AI tính từng bước (tồn kho → tiền hàng → mã giảm giá → ship → tổng) và tự kiểm tra lại. Phép tính nhờ đó chính xác hơn.</li>
<li><strong>Prompt Chaining:</strong> mỗi prompt chỉ làm một việc, nên khi sai có thể biết ngay lỗi nằm ở bước nào.</li>
<li><strong>Xử lý đứt gãy:</strong> trường <code>status</code> chạy xuyên chuỗi (<code>OK</code>/<code>PARTIAL</code>/<code>FAILED</code>). <code>FAILED</code> thì dừng và chuyển nhân viên, <code>PARTIAL</code> thì vẫn chạy nhưng gắn cờ.</li>
<li><strong>Chống bẫy dữ liệu:</strong> các câu lệnh IF-THEN bắt buộc AI báo lỗi thay vì tự đoán. Trong ví dụ, hệ thống bắt được 4 lỗi cùng lúc: SĐT sai, địa chỉ thiếu thành phố, vượt tồn kho và hết hàng.</li>
</ul>

