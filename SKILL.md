---
name: slides
description: Dựng và sửa bản trình bày PowerPoint (.pptx) bằng generate_slides, inspect_slides, edit_presentation — spec JSON, layout, dùng file mẫu, sửa deck đã tạo, quy ước deck cho lãnh đạo NBC. Nạp trước khi gọi bất kỳ hàm slide nào: khi người dùng xin slide, deck, bản trình bày, file PowerPoint/.pptx, muốn sửa deck đã tạo, hoặc đính kèm .pptx làm mẫu.
---

# Bản trình bày PowerPoint (.pptx)

Ba hàm:
- generate_slides dựng bản trình bày .pptx native (biểu đồ Office, bảng, icon) từ một spec JSON và
  trả về link tải.
- inspect_slides đọc cấu trúc một tệp .pptx (file mẫu người dùng đính kèm, hoặc deck đã tạo).
- edit_presentation sửa một deck đã tạo mà không dựng lại.

Mọi quy tắc dữ liệu của system prompt vẫn áp dụng cho từng con số trên slide: <real_data>,
<counting_pitfalls>, <sanity_checks>, cách viết số ở <response_format>.

## Khi nào gọi

- Chỉ khi người dùng xin slide, deck, bản trình bày, file PowerPoint / .pptx hay tài liệu để trình
  chiếu, muốn sửa deck đã tạo, hoặc đính kèm .pptx làm mẫu. Không tự xuất khi không được xin.
- HIỂU → QUERY → PHÂN TÍCH xong mới gọi generate_slides, một lần, với deck hoàn chỉnh. Mọi con số
  trên slide lấy từ truy vấn đã chạy trong cuộc trò chuyện này, cùng kỳ, đã qua <sanity_checks>.
  Chưa có số thì truy vấn trước; thiếu dữ liệu thì không đưa lên slide.
- Câu trả lời là một deck thì biểu đồ nằm trong deck — không vẽ lại cùng số liệu trong chat.

## Spec của generate_slides

content là MỘT chuỗi JSON, không kèm chữ hay ``` bao ngoài. Ví dụ deck thường:

```json
{
  "title": "Hiệu quả kinh doanh lũy kế T01–T07/2026",
  "subtitle": "Báo cáo cho Tổng Giám đốc",
  "author": "Ban Kiểm soát",
  "theme": "midnight",
  "heading_font": "Calibri",
  "body_font": "Calibri",
  "footer": "NBC · Lũy kế T01–T07/2026",
  "slides": [
    {"layout": "cover", "eyebrow": "Lũy kế T01–T07/2026", "title": "Biên ròng 2,41%: ba nhóm kéo xuống dưới ngưỡng", "subtitle": "Kết quả kinh doanh theo nhóm và khách"},
    {"layout": "kpi_row", "eyebrow": "Lũy kế T01–T07/2026", "title": "Doanh thu tăng nhưng biên ròng mỏng",
     "stats": [{"value": "1.250,36 tỷ", "label": "Doanh thu", "change": "+4,25%"},
               {"value": "2,41%", "label": "Biên ròng", "change": "−0,32 điểm %"}]},
    {"layout": "chart", "eyebrow": "Lũy kế T01–T07/2026", "title": "SALE 3 có biên ròng thấp nhất",
     "chart_type": "bar", "labels": ["SALE 1", "SALE 3"],
     "datasets": [{"label": "Biên ròng (%)", "data": [4.12, -0.93]}],
     "insight": ["SALE 3 âm 0,93%: Báo động", "SALE 1 đạt 4,12%: Tốt"]},
    {"layout": "closing", "title": "Ba việc cần làm trong quý", "takeaways": ["…"]}
  ]
}
```

Trường chung của mỗi slide: `layout`, `title`, `eyebrow` (dòng nhỏ trên tiêu đề), `subtitle`.

Layout và trường chính:
- `cover`: title, subtitle, author, eyebrow, icon, date, chips[]
- `section`: number ("01"), eyebrow, title, lead — trang ngăn chương
- `title_bullets`: title, eyebrow, bullets[]
- `title_body`: title, eyebrow, body (các đoạn cách nhau bằng \n)
- `two_column_text`, `comparison_two`: left{} và right{}, hoặc columns[]; mỗi thẻ {heading, icon,
  subtitle, description, points[], highlight: true, badge}
- `kpi_row`: title, stats[] {value, label, change}
- `timeline_horizontal`, `process_flow`: steps[] {when, title, description}
- `icon_list_vertical`: items[] {icon, title, description}
- `icon_grid_2x2`, `icon_grid_3`, `pillars`: items[] {icon, title, description}
- `chart`: chart_type, labels[] kèm values[], hoặc datasets[] {label, data[]}; insight[] và
  insight_title là chữ bên cạnh biểu đồ
- `funnel`, `pyramid`, `cycle`, `bullseye`: nodes[] {label, description}
- `quadrant`: nodes[] {label, description}, x_axis, y_axis, points[]
- `alert`: title, level (info | tip | warning | danger), body hoặc bullets[]
- `table`: headers[], rows[] (danh sách hoặc danh sách object)
- `closing`: title, eyebrow, takeaways[]

Không dùng `text_image_right`, `image_left_text_right`, `image_full_caption` (tool không có nguồn
ảnh) và `quote` (không có trích dẫn thật); không điền contact ở closing.

Icon dùng tên như target, rocket, lightbulb, shield, users, trending-up, bar-chart, dollar-sign,
check-circle, award, layers, scale, workflow, globe, calendar, settings.

Tool trả về một dòng [TOOL_RESULT …] có link markdown để người dùng tải .pptx.

## Quy ước deck cho lãnh đạo NBC

Cấu trúc (6–12 slide): cover → kpi_row tóm tắt (tối đa 4 chỉ số) → alert điểm nóng → các slide phân
tích (chart / table) → khuyến nghị, mỗi khuyến nghị kèm số tỷ tác động → closing (takeaways ≤ 5).
- Tiêu đề slide là kết luận, không phải tên chủ đề: "Nhóm A kéo biên ròng toàn công ty xuống 1,82%"
  thay vì "Biên ròng theo nhóm". Tối đa khoảng 90 ký tự.
- Kỳ dữ liệu ở eyebrow của mọi slide có số, ví dụ "Lũy kế T01–T06/2026".
- Trong slide ghi trạng thái bằng chữ "Tốt / Cần theo dõi / Báo động", không dùng emoji. alert dùng
  level "danger" cho Báo động, "warning" cho Cần theo dõi.
- Kết luận dựa trên dữ liệu có giới hạn (<out_of_scope>) thì thêm slide "Giới hạn dữ liệu".
- Không xếp hai slide danh sách liền nhau; mỗi slide 3–6 ý ngắn; có số thì ưu tiên kpi_row, chart,
  table thay cho danh sách dài.

Spec bắt buộc:
- "theme": "midnight", "heading_font": "Calibri", "body_font": "Calibri" — font tiêu đề mặc định
  (Georgia) thiếu chữ có dấu tiếng Việt như Ư, ế, ổ, ự.
- "footer": "NBC · <kỳ dữ liệu>".
- Deck dựng theo file mẫu: xem mục Dùng file mẫu; theme / font / footer ở trên không áp cho slide có
  reuse.

Biểu đồ trong slide: chart_type chỉ có bar, bar_horizontal, line, area, pie, doughnut, stacked_bar;
một trục, màu theo theme — không có trục phụ, không tô màu theo ngưỡng, không tô đỏ số âm.
- values và datasets[].data là SỐ JSON, dấu chấm thập phân, dấu trừ ASCII: 57.54, -0.93. Không
  truyền chuỗi: "1.270,54" bị đọc thành 0, "−87" thành +87.
- Một đơn vị cho cả biểu đồ, làm tròn hai chữ số thập phân như QT-19 (57.54, 3.45 — không cắt
  bớt chữ số lẻ); đơn vị ghi ở tiêu đề hoặc eyebrow và ở tên series ("Doanh thu (tỷ VND)", "Biên
  ròng (%)").
- Không trộn tỷ và % trong một biểu đồ: tách hai slide, hoặc đưa đại lượng thứ hai vào insight[].
- Nhãn khách (labels[], ô bảng) là tên khách đầy đủ như ten_khach, không dùng mã EO-…; khách ở hai
  nhóm thì ghi kèm nhóm ("MANGO MNG, S.A. — SALE 1"). Tên dài thì bar_horizontal.
- Mỗi chỉ tiêu xếp hạng một biểu đồ hay một slide, sắp giảm dần theo chỉ tiêu của chính nó: hỏi
  doanh thu và lợi nhuận ròng thì hai biểu đồ, không dùng thứ tự doanh thu cho lợi nhuận.
- Tối đa 8 nhãn thì số mới hiện trên cột; nhiều hơn thì gom phần đuôi thành "Khác" hoặc dùng table.
- Màu và dấu không thể hiện được trạng thái, nên insight[] (2–4 ý, có số) phải nói rõ cột nào âm,
  cột nào dưới ngưỡng.
- Ánh xạ từ <chart_rules>:
  · Xu hướng tháng: line; biên đặt ở biểu đồ riêng hoặc trong insight[].
  · So sánh nhóm / khách: bar sắp giảm dần; bar_horizontal khi tên dài.
  · Thác lợi nhuận, bóc gộp − ròng: table các tầng (tỷ VND, % doanh thu), hoặc bar các khoản bị trừ
    sắp giảm dần.
  · Cơ cấu chi phí theo khoản mục hoặc tầng chi phí: stacked_bar; pie / doughnut chỉ khi là tỷ trọng
    của một tổng và không quá 5 phần.
  · Pareto: bar sắp giảm dần, tỷ trọng cộng dồn ghi trong insight[].
  · Khách lớn biên mỏng (thay biểu đồ phân tán) và nhóm × tháng (thay heatmap): table có cột trạng
    thái.

Bảng và layout:
- table: tối đa 8 dòng dữ liệu và 6 cột; số trong ô viết kiểu Việt, hai chữ số thập phân
  ("57,54 tỷ", "−1,20%").
- kpi_row tối đa 4 chỉ số; process_flow / timeline_horizontal tối đa 5 bước; comparison_two tối đa
  3 cột. Phần vượt bị cắt, không báo lỗi.

## Dùng file mẫu (.pptx người dùng đính kèm)

Khi người dùng đính kèm tệp .pptx và muốn làm theo nó ("dùng file này làm mẫu", "giữ giao diện file
đính kèm", "làm theo template này"):
- Gọi inspect_slides MỘT lần, để trống file_id — tool tự lấy tệp đang đính kèm. Kết quả là JSON nội
  bộ: không dán cho người dùng, không gọi lại với id khác. Người dùng chỉ hỏi cấu trúc tệp thì tóm
  tắt kết quả, không gọi generate_slides.
- Truy vấn số liệu như mọi deck, rồi gọi generate_slides một lần với "reference_file_id" là file_id
  của inspect, chép đúng từng ký tự. Đó là UUID (8-4-4-4-12 ký tự hex): không bịa, không thay bằng
  tên tệp hay số slide.
- Mỗi slide đầu ra dùng lại một slide mẫu:
  {"reuse": {"slide": <index của slide mẫu>, "text": {"<id>": "chữ mới"}}}
  · index lấy từ slides[].index của inspect (đếm từ 0); id lấy từ slides[].text.
  · Bảng (slides[].tables) điền {"headers": [...], "rows": [[...]]}: tool giữ định dạng ô của mẫu,
    tự thêm bớt dòng, cột. Điền chuỗi vào id của bảng là lỗi.
  · Điền MỌI id chữ của slide mẫu đó: id không điền giữ nguyên chữ mẫu; ô không cần thì "".
  · Không ghi keep_ids — hình trang trí, logo, nền tự được chép. Chỉ ghi drop_ids (id của inspect)
    khi người dùng muốn bỏ một hình của mẫu. Một slide mẫu dùng lại được nhiều lần.
- Slide có reuse lấy màu, font, nền của mẫu, nên không áp theme / font / footer của Spec bắt buộc.
  Tiêu đề là kết luận, kỳ dữ liệu ghi vào một ô chữ phù hợp, số liệu theo mọi quy tắc ở trên.
- Mẫu không có slide hợp với biểu đồ thì dùng slide chart như thường (không reuse): tool vẽ nó trên
  nền của mẫu.
- Tool trả lời id sai hoặc "not the attached template": gọi lại generate_slides với đúng file_id của
  inspect. Không chuyển sang deck thường, không báo người dùng là thiếu mẫu.
- Dùng file mẫu cần quản trị viên bật valve template_mode_enabled; tắt thì tool bỏ qua
  reference_file_id và ra deck thường — nói rõ điều đó với người dùng.
- Còn tệp mẫu đính kèm trong cuộc trò chuyện thì mọi deck mới đều theo mẫu đó; người dùng muốn deck
  không theo mẫu thì đề nghị mở cuộc trò chuyện mới.

## Sửa deck đã tạo (deck thường hay deck theo mẫu)

Có deck rồi, người dùng thường chat thêm để chỉnh. Phân loại yêu cầu trước:
- Chỉnh trên slide đang có (đổi câu chữ, tiêu đề to hay nhỏ, chữ tràn khung, sửa một ô bảng, thêm
  hoặc bớt một ý trong danh sách, tách tiêu đề khỏi nội dung, lấy ảnh người dùng gửi làm nền slide):
  dùng edit_presentation, KHÔNG gọi lại generate_slides.
- Thứ edit_presentation không sửa được (biểu đồ: số liệu, loại biểu đồ; thêm, bớt hay đổi thứ tự
  slide; đổi layout; đổi màu): gọi lại generate_slides với spec lần trước, chỉ đổi phần được xin, giữ
  nguyên các slide khác. Deck theo mẫu giữ reference_file_id và reuse như lần trước. Chữ đã sửa bằng
  edit_presentation phải đưa vào spec mới; cỡ chữ và ảnh nền đã chỉnh không mang sang được, nên báo
  người dùng điều đó.
- Chèn, thay hay xoá ảnh bên trong slide (ngoài ảnh nền), kéo hình sang chỗ khác, đổi màu một hình:
  không làm được qua chat. Nói thẳng như vậy, gợi ý sửa trong PowerPoint (logo công ty thì dùng file
  mẫu); không gọi tool.

Cách gọi edit_presentation:
- content là MỘT chuỗi JSON:
  {"presentation_file_id": "<id>", "title": "<tên deck>", "operations": [...]}.
  · presentation_file_id là UUID trong link tải gần nhất (/api/v1/files/<id>/content) của
    generate_slides hoặc edit_presentation, không bao giờ là id tệp mẫu. Mỗi lần sửa ra link mới;
    lần sau dùng id trong link mới.
  · title: tên deck như lúc tạo, thêm "(sửa)", để tên tệp dễ nhận.
- Lấy shape_id: gọi inspect_slides với file_id là id deck, mỗi deck một lần. id nằm trong
  slides[].text / slides[].tables; id các ô không đổi qua các lần sửa nên không cần inspect lại.
  Deck thường không có id trong spec đã gửi, nên luôn phải inspect.
- "slide" đánh số như PowerPoint, slide đầu là 1, tức là index + 1 của inspect.
- Chọn op:
  · đổi chữ: set_text {slide, shape_id, text}, giữ nguyên màu, cỡ, font của ô; danh sách nhiều ý thì
    các ý cách nhau bằng \n. Chữ mới dài, sợ tràn: replace_text_and_fit {slide, shape_id, text}
    (đổi chữ rồi co cỡ chữ vừa khung) — trên cùng một ô dùng op này thay cho set_text + shrink_font.
  · chữ nhỏ hoặc lớn hơn theo ý người dùng: set_font_pt {slide, shape_id} với font_pt (vd 28) hoặc
    font_pt_delta (vd -4), không ghi cả hai.
  · chữ tràn khung: shrink_font {slide, shape_id} co cỡ chữ đến khi vừa (min_pt, mặc định 14); hoặc
    enable_autofit {slide, shape_id} để PowerPoint tự co khi mở tệp. Khung quá thấp: resize_shape
    {slide, shape_id, delta_height_in} (vd 0.5, inch) cho khung cao thêm.
  · ô bảng: set_table_cell {slide, shape_id, row, col, text}, đếm từ 0, dòng 0 là dòng tiêu đề.
    Không dùng set_text cho bảng.
  · một ô chứa cả tiêu đề lẫn nội dung: split_text {slide, from_shape_id, to_shape_id,
    "mode": "first_line"} (hoặc "first_paragraph").
  · ảnh người dùng gửi làm nền ("slide 2 dùng background này"): set_background_image {slide,
    overlay, crop}, không có shape_id, mỗi slide một op. Tool tự lấy ảnh mới nhất người dùng gửi
    trong chat; chưa có ảnh thì xin người dùng gửi. overlay (0–0.95, mặc định 0.7) là độ đậm lớp màu
    nền cũ phủ lên ảnh để chữ dễ đọc: muốn ảnh rõ hơn thì giảm (0.4–0.5), chữ khó đọc thì tăng. Ảnh
    có chữ hay logo in sẵn ở mép thì cắt đi bằng crop, vd {"bottom": 0.25}; mỗi cạnh tối đa 0.45.
    Làm lại trên cùng slide thì thay nền cũ.
- Đổi số liệu trên slide: truy vấn lại trước; số mới qua <sanity_checks> như khi tạo deck.
- Một op sai thì cả lô bị từ chối, không tệp nào được lưu: đọc lỗi, sửa đúng op đó rồi gọi lại.
- Tool báo đang tắt sửa deck (presentation_edit_enabled) thì nói quản trị viên cần bật; không tự
  dựng lại deck.

## Sau khi gọi

Trả lời bằng link tải generate_slides hoặc edit_presentation trả về, giữ nguyên, kèm 1–2 câu nêu
deck gồm gì (lần sửa: đã sửa gì) và kỳ dữ liệu. Tool báo lỗi thì nói thẳng lỗi, không nói là đã tạo
file.
