# Bộ máy "Làm đẹp slide cũ" — Implementation Plan (Kế hoạch 1/2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Một thư viện Python + lệnh `python -m autoslide restyle` nhận file `.pptx` cũ và một template, trả về `.pptx` mới đúng giao diện template, giữ nguyên chữ (hoặc dịch nếu được yêu cầu), không tràn chữ, có ảnh minh hoạ có giấy phép và báo cáo bằng tiếng Việt.

**Architecture:** Code làm toàn bộ việc máy móc (đọc, đổ, đo, chia, crop, render, kiểm chứng). AI chỉ trả JSON đã được kiểm tra theo schema Pydantic: vai trò từng khối chữ, layout, chỗ chia, từ khoá ảnh, bản dịch. Chữ gốc không bao giờ đi qua AI, trừ bước dịch. Lớp AI là một interface với ba bản cài đặt: Fake (test), Claude, Gemini/Gemma.

**Tech Stack:** Python ≥ 3.11, python-pptx, Pillow, Pydantic v2, anthropic, google-genai, httpx, pypdfium2, LibreOffice headless (`soffice`), pytest.

**Spec:** `docs/superpowers/specs/2026-11-12-auto-slide-design.md`. Kế hoạch 2 (trang web, hàng đợi, hướng dẫn lấy key, thư viện template, Docker) sẽ viết sau, dùng `autoslide.pipeline.restyle` của kế hoạch này.

## Global Constraints

- **Không bao giờ commit dữ liệu của bác sĩ** (CLAUDE.md quy tắc 8). Test chỉ dùng file `.pptx` do `tests/fixtures_builder.py` tự sinh. Không chép nội dung, tên file, tên hội nghị hay tên bệnh viện của bộ mẫu `D:\File\ppt` vào code, test hay commit message. `.gitignore` đã chặn `*.pptx`, `*.pdf`, `images/`.
- Chữ gốc → chữ mới phải được **code chép**. AI chỉ trả mã khối (`block_id`). Bước dịch (Task 10) là ngoại lệ duy nhất.
- Không thu nhỏ chữ: xoá `normAutofit`/`spAutoFit` khỏi `bodyPr` của mọi placeholder đã đổ chữ và không đặt cỡ chữ. Tràn thì chia slide.
- Chữ đổ vào giữ text và các thuộc tính đậm/nghiêng/gạch chân/chỉ số trên-dưới (`baseline`) của từng run. Bỏ font, cỡ, màu để template tự áp kiểu.
- Không sửa file đầu vào. Mọi đầu ra ghi vào thư mục `out_dir`.
- Chữ người dùng nhìn thấy (báo cáo, lỗi CLI) bằng **tiếng Việt**.
- Ảnh chỉ lấy từ Wikimedia Commons với giấy phép thuộc `{"Public domain", "CC0", "CC BY", "CC BY-SA"}` (mọi phiên bản). Ảnh nào cũng phải có dòng trong `credits.txt`.
- Không chèn ảnh vào slide có bảng, biểu đồ, nhóm hình, hoặc slide có tổng số ký tự > 400.
- Claude: model mặc định `claude-opus-5-5`, gọi `client.beta.messages.create` với `betas=["server-side-fallback-2026-07-01"]`, `fallbacks="default"`, `output_config={"effort": "medium", "format": {"type": "json_schema", "schema": ...}}`, `max_tokens=16000`. Phải kiểm tra `stop_reason == "refusal"` trước khi đọc `content`.
- Gemini: model đọc từ biến môi trường `AUTOSLIDE_GEMINI_MODEL`, mặc định `gemma-4-31b-it`. **Builder phải xác minh tên model này** bằng `client.models.list()` và ghi tên đúng vào README. Key đọc từ `GEMINI_API_KEY`, key Claude từ `ANTHROPIC_API_KEY`.
- Chữ chèn thêm khi chia slide: `" (tiếp)"` nếu ngôn ngữ đầu ra là tiếng Việt, `" (cont.)"` nếu là ngôn ngữ khác. Ngôn ngữ đầu ra là ngôn ngữ đích khi dịch, hoặc ngôn ngữ đoán được của slide gốc khi không dịch.

## Review Focus

1. **Chữ tiếng Việt khác dạng Unicode** (NFD so với NFC, hoặc có khoảng trắng không ngắt `\u00a0`, xuống dòng mềm `\v`): người dùng chờ kết quả kiểm chứng vẫn "đạt" vì chữ trông y hệt. Test nằm ở Task 5 (`test_verify_normalizes_unicode_and_spaces`).
2. **Template có layout tên lạ** (tiếng Việt, hoặc tên kiểu "Google Shape;12") hoặc thiếu hẳn một kiểu layout (không có 2 cột, không có khung ảnh): người dùng chờ kết quả vẫn ra slide đúng, vì layout được chọn theo cấu trúc placeholder chứ không theo tên. Test nằm ở Task 8 (`test_fallback_plan_ignores_layout_names`).
3. **Một đoạn văn dài đến mức đứng riêng một slide vẫn tràn**: người dùng chờ phần mềm chia theo câu. Nếu một câu riêng lẻ vẫn tràn thì giữ nguyên chữ và đánh dấu "cần xem lại", không thu nhỏ chữ. Test nằm ở Task 4 (`test_split_long_paragraph_by_sentence`).
4. **Slide cũ tỉ lệ 4:3 mà template 16:9** (hoặc ngược lại): người dùng chờ nhóm hình và bảng được co giãn giữ tỉ lệ vào vùng nội dung, không méo và không lòi ra ngoài slide. Test nằm ở Task 7 (`test_group_scaled_into_area_keeps_aspect`).
5. **File hỏng, file `.ppt` cũ đổi đuôi, hoặc template không có layout nào có placeholder thân bài**: người dùng chờ một câu báo lỗi tiếng Việt dễ hiểu, không phải traceback. Test nằm ở Task 13 (`test_cli_bad_file_vietnamese_error`).

---

## File Structure

```
pyproject.toml
README.md                      # cách chạy CLI, biến môi trường, tên model Gemma đã xác minh
src/autoslide/
  __init__.py
  __main__.py                  # CLI (Task 13)
  model.py                     # dataclass + schema Pydantic dùng chung (Task 1)
  extract.py                   # đọc slide cũ thành khối (Task 2)
  template.py                  # đọc layout, tạo bản trắng từ template (Task 3)
  fit.py                       # đo chữ vừa khung, chia đoạn (Task 4)
  verify.py                    # kiểm chứng chữ / bản dịch (Task 5)
  build.py                     # dựng slide: chữ, notes, chia, section (Task 6)
  build_media.py               # ảnh, bảng, nhóm hình, co giãn vùng (Task 7)
  analyze.py                   # gọi AI lập kế hoạch slide + kiểm tra + dự phòng (Task 8)
  prompts.py                   # nội dung prompt (Task 8, 10, 11)
  ai/base.py, ai/fake.py       # interface + bản giả cho test (Task 8)
  ai/claude.py, ai/gemini.py   # nhà cung cấp thật (Task 9)
  translate.py                 # dịch + file đối chiếu song ngữ (Task 10)
  images.py                    # Wikimedia Commons, crop, credits, kiểm tra ảnh (Task 11)
  render.py                    # LibreOffice → PDF → PNG (Task 12)
  pipeline.py                  # nối các bước + báo cáo tiếng Việt (Task 13)
tests/
  fixtures_builder.py          # sinh deck/template giả (Task 1)
  conftest.py
  test_<module>.py
```

---

### Task 1: Khung dự án, kiểu dữ liệu chung, bộ sinh file giả

**Files:**
- Create: `pyproject.toml`, `src/autoslide/__init__.py`, `src/autoslide/model.py`, `tests/fixtures_builder.py`, `tests/conftest.py`, `tests/test_fixtures.py`

**Interfaces:**
- Produces (`model.py`):
  - `@dataclass Run(text: str, bold: bool|None=None, italic: bool|None=None, underline: bool|None=None, baseline: int|None=None)`
  - `@dataclass Paragraph(runs: list[Run], level: int = 0)` với property `text -> str`
  - `BlockKind = Literal["text", "picture", "table", "group", "chart", "other"]`
  - `@dataclass Block(id: str, kind: BlockKind, paragraphs: list[Paragraph], left: int, top: int, width: int, height: int, font_pt: float|None, shape_xml: bytes|None, image_blob: bytes|None, image_ext: str|None)`. `id` có dạng `"s{slide_no}b{n}"`, `slide_no` bắt đầu từ 1, `n` theo thứ tự đọc (top rồi left).
  - `@dataclass SourceSlide(no: int, blocks: list[Block], notes: list[Paragraph], width: int, height: int)`
  - `@dataclass PlaceholderInfo(idx: int, kind: Literal["title","body","picture","table","chart","other"], left: int, top: int, width: int, height: int, font_pt: float)`
  - `@dataclass LayoutInfo(index: int, name: str, placeholders: list[PlaceholderInfo])`
  - Pydantic: `Role = Literal["title","subtitle","section_label","body","caption","citation","stat","footer","other"]`; `BlockRole(block_id: str, role: Role)`; `Placement(block_id: str, placeholder_idx: int)` (`-1` = vùng nội dung tự do); `SlidePlan(roles: list[BlockRole], layout_index: int, placements: list[Placement], split_after_block_id: str|None = None, image_query: str|None = None)`
- Produces (`tests/fixtures_builder.py`):
  - `make_template(path, *, layout_names: list[str]|None=None) -> Path`: lấy template mặc định của python-pptx, xoá hết slide. Nếu có `layout_names` thì đổi tên lần lượt các layout.
  - `make_old_deck(path, *, aspect: Literal["16:9","4:3"]="16:9") -> Path`: deck 7 slide **chỉ dùng text box tự do** (không placeholder), gồm: (1) tiêu đề + tên người trình bày; (2) nhãn chương `"01 · INTRO"` + tiêu đề + 3 bullet (có cấp 1) + footer `"CONF 2026 · 2"` + dòng trích dẫn `"Smith J. J Test 2020;1:1"`; (3) cùng nhãn chương + tiêu đề + 14 bullet dài (bắt buộc phải chia); (4) nhãn `"02 · METHODS"` + tiêu đề + bảng 3×3; (5) tiêu đề + 2 bullet + 1 ảnh PNG 400×300 (sinh bằng Pillow); (6) tiêu đề + nhóm hình (3 hình chữ nhật + 3 nhãn "40%", "25%", "10%"); (7) "Thank you". Mỗi slide có notes `"Notes {n}"`. Slide 2 có một run đậm và một run `baseline=30000` (chỉ số trên).
  - `conftest.py` cung cấp fixture `template_path`, `old_deck_path`, `old_deck_43_path` sinh vào `tmp_path`.

- [ ] **Step 1: Viết test hỏng** `tests/test_fixtures.py::test_fixtures_open`: mở hai file bằng `Presentation`, assert `len(old.slides) == 7`, `len(tpl.slides) == 0`, `len(tpl.slide_layouts) >= 5`, và slide 2 không có shape nào `is_placeholder`.
- [ ] **Step 2: Chạy** `pytest tests/test_fixtures.py -v` → FAIL (`ModuleNotFoundError`).
- [ ] **Step 3: Viết** `pyproject.toml` (package `autoslide` ở `src/`, deps như Tech Stack, extra `dev = ["pytest"]`), `model.py`, `fixtures_builder.py`, `conftest.py`. Cài bằng `pip install -e .[dev]`.
- [ ] **Step 4: Chạy** `pytest -v` → PASS.
- [ ] **Step 5: Commit** `git add pyproject.toml src tests && git commit -m "feat: khung dự án, kiểu dữ liệu và bộ sinh file giả"`

---

### Task 2: Đọc slide cũ thành khối

**Files:**
- Create: `src/autoslide/extract.py`, `tests/test_extract.py`

**Interfaces:**
- Consumes: `model.*`
- Produces: `extract_deck(path: Path) -> list[SourceSlide]`; `block_text(block: Block) -> str` (nối các đoạn bằng `"\n"`; với nhóm hình là chữ của mọi shape con theo thứ tự; với bảng là chữ các ô theo hàng).

Text box và placeholder → `kind="text"`, giữ đoạn, cấp và run. Picture → `kind="picture"` + `image_blob`/`image_ext`. Graphic frame có bảng → `"table"`, có biểu đồ → `"chart"`. Group → `"group"`. Với table/group/chart, gán `shape_xml = etree.tostring(shape._element)` và `paragraphs` = chữ bên trong (để kiểm chứng). `font_pt` = cỡ chữ lớn nhất tìm thấy trong run, `None` nếu không có. Shape rỗng không có chữ cũng không phải ảnh thì bỏ qua.

- [ ] **Step 1: Viết test hỏng**:
  - `test_extract_counts_and_ids`: 7 slide; id slide 2 bắt đầu `"s2b"`; khối được sắp theo (top, left).
  - `test_extract_keeps_levels_and_runs`: slide 2 có đoạn `level == 1`; có `Run(bold=True)`; có run `baseline == 30000`.
  - `test_extract_kinds`: slide 4 có đúng 1 khối `"table"` và `block_text` chứa tiêu đề cột; slide 5 có `"picture"` với `image_ext == "png"`; slide 6 có `"group"` và `block_text` chứa `"40%"`.
  - `test_extract_notes`: `slides[0].notes[0].text == "Notes 1"`.
- [ ] **Step 2: Chạy** `pytest tests/test_extract.py -v` → FAIL.
- [ ] **Step 3: Viết** `extract.py`.
- [ ] **Step 4: Chạy** → PASS.
- [ ] **Step 5: Commit** `git commit -am "feat: đọc slide cũ thành khối"` (nhớ `git add` file mới).

---

### Task 3: Đọc template, tạo bản trắng

**Files:**
- Create: `src/autoslide/template.py`, `tests/test_template.py`

**Interfaces:**
- Produces:
  - `read_layouts(path: Path) -> list[LayoutInfo]`. `kind` lấy từ `placeholder_format.type`: TITLE/CENTER_TITLE → `"title"`; BODY/OBJECT/SUBTITLE → `"body"`; PICTURE → `"picture"`; TABLE → `"table"`; CHART → `"chart"`; DATE/FOOTER/SLIDE_NUMBER → **loại khỏi danh sách**; còn lại → `"other"`. `font_pt` là cỡ chữ cấp 1 của placeholder, lần lượt tra ở `lstStyle` của placeholder trên layout, placeholder tương ứng trên master, rồi `titleStyle`/`bodyStyle` của master. Không tìm thấy thì title = 32, body = 18.
  - `blank_from_template(path: Path) -> Presentation`: mở template và xoá mọi slide (xoá cả `sldId` lẫn relationship để không còn part mồ côi).
  - `class TemplateError(Exception)`: ném ra khi template không có layout nào có placeholder `"body"`.

- [ ] **Step 1: Viết test hỏng**:
  - `test_read_layouts_kinds`: layout "Title and Content" có 1 `"title"` và 1 `"body"`, không có slide number.
  - `test_font_pt_resolved`: mọi placeholder có `font_pt > 0`.
  - `test_blank_from_template_has_no_slides`: dùng một template có 2 slide mẫu (thêm vào trong test), sau khi lưu và mở lại thì `len(slides) == 0` và không còn part `slides/slide*.xml` trong zip.
  - `test_template_without_body_raises`: xoá placeholder body khỏi mọi layout → `TemplateError`.
- [ ] **Step 2: Chạy** → FAIL. **Step 3: Viết** `template.py`. **Step 4: Chạy** → PASS.
- [ ] **Step 5: Commit** `"feat: đọc layout template"`.

---

### Task 4: Đo chữ vừa khung và chia đoạn

**Files:**
- Create: `src/autoslide/fit.py`, `tests/test_fit.py`

**Interfaces:**
- Produces:
  - `measure_height(paragraphs: list[Paragraph], width_emu: int, font_pt: float) -> int`: chiều cao EMU ước tính. Ngắt dòng theo từ, đo bằng `PIL.ImageFont.truetype` với font tìm thấy trong thư mục font hệ thống. Nếu không có font thì dùng DejaVuSans của Pillow. Tính thêm thụt lề 0.375 inch mỗi cấp, khoảng cách dòng 1.2 × cỡ chữ, cách đoạn 0.3 × cỡ chữ.
  - `fits(paragraphs, ph: PlaceholderInfo, safety: float = 0.9) -> bool`: `measure_height(..., ph.width - inset, ph.font_pt) <= ph.height * safety`, với inset = 0.2 inch.
  - `split_to_fit(paragraphs, ph) -> tuple[list[list[Paragraph]], bool]`: chia thành các trang vừa khung. Ưu tiên ngắt giữa các đoạn. Một đoạn đứng riêng mà vẫn tràn thì ngắt theo câu (`(?<=[.!?;])\s+`). Phần tử thứ hai trả về `True` nếu vẫn còn trang tràn (một câu riêng lẻ không vừa), khi đó giữ nguyên chữ.

- [ ] **Step 1: Viết test hỏng**:
  - `test_short_text_fits`: 3 đoạn ngắn, khung 6×4 inch, 18pt → `True`.
  - `test_long_text_does_not_fit`: 14 đoạn × 120 ký tự → `False`.
  - `test_split_keeps_all_text_in_order`: nối chữ các trang sau khi chia == nối chữ ban đầu; mọi trang `fits`; `flag is False`.
  - `test_split_long_paragraph_by_sentence` (Review Focus 3): 1 đoạn gồm 30 câu → ≥ 2 trang, chữ không mất. Thêm 1 "câu" 3000 ký tự không dấu chấm → `flag is True` và chữ vẫn đủ.
  - `test_wider_box_needs_less_height`.
- [ ] **Step 2–4:** Chạy → FAIL, viết `fit.py`, chạy → PASS.
- [ ] **Step 5: Commit** `"feat: đo chữ vừa khung và chia trang"`.

---

### Task 5: Kiểm chứng chữ và bản dịch

**Files:**
- Create: `src/autoslide/verify.py`, `tests/test_verify.py`

**Interfaces:**
- Produces:
  - `normalize(s: str) -> str`: chuẩn hoá NFC, đổi `\u00a0`/`\v`/tab thành khoảng trắng, gộp khoảng trắng liền nhau, strip.
  - `deck_paragraphs(path: Path) -> set[str]`: tập đoạn đã chuẩn hoá trong toàn bộ file (gồm shape, nhóm, ô bảng, notes), bỏ chuỗi rỗng.
  - `@dataclass TextCheck(missing: list[str], extra: list[str])` với `ok -> bool`.
  - `verify_text(source: Path, output: Path, *, dropped: set[str], allowed_suffixes: tuple[str, ...]) -> TextCheck`. `missing` là đoạn có ở nguồn mà không có ở đầu ra và không thuộc `dropped`. `extra` là đoạn có ở đầu ra mà không có ở nguồn, sau khi đã bỏ hậu tố thuộc `allowed_suffixes`. So theo **tập**, vì nhãn chương lặp lại nhiều lần được gộp vào slide section.
  - `check_translation(original: str, translated: str, keep_terms: list[str]) -> list[str]`: danh sách lỗi bằng tiếng Việt. Các con số (`\d+([.,]\d+)?`, kể cả `%`) của bản gốc phải có đủ, bỏ qua dấu phân cách hàng nghìn. Mỗi `keep_term` có trong bản gốc phải có nguyên văn trong bản dịch. Bản dịch rỗng là lỗi.

- [ ] **Step 1: Viết test hỏng**:
  - `test_verify_identical_ok`: tự so file với chính nó → `ok`.
  - `test_verify_detects_missing_and_extra`: sửa một đoạn ở bản sao → nó vừa thiếu vừa thừa.
  - `test_verify_allows_suffix_and_dropped`: thêm `" (tiếp)"` vào tiêu đề, xoá footer có trong `dropped` → `ok`.
  - `test_verify_normalizes_unicode_and_spaces` (Review Focus 1): `"Tổng quan"` dạng NFD + `\u00a0` so với dạng NFC → bằng nhau.
  - `test_check_translation_numbers`: `"12.5% of 1,234"` → `"12,5% trong 1.234"` không lỗi; thiếu `"12.5"` → 1 lỗi. `keep_terms=["XYZ"]` mà bản dịch thiếu → lỗi.
- [ ] **Step 2–4:** FAIL → viết → PASS.
- [ ] **Step 5: Commit** `"feat: kiểm chứng chữ và bản dịch"`.

---

### Task 6: Dựng slide: chữ, notes, chia slide, slide section

**Files:**
- Create: `src/autoslide/build.py`, `tests/test_build.py`

**Interfaces:**
- Consumes: `extract_deck`, `read_layouts`, `blank_from_template`, `fits`, `split_to_fit`, `SlidePlan`, `verify_text`
- Produces:
  - `@dataclass BuiltSlide(source_no: int, layout_index: int, part: int, parts: int, overflow_flag: bool)`
  - `@dataclass BuildResult(slides: list[BuiltSlide], dropped_texts: set[str], section_slides: list[int], flagged: list[tuple[int, str]])`. `flagged` là danh sách (số slide gốc, lý do tiếng Việt).
  - `build_deck(template: Path, source: list[SourceSlide], plans: list[SlidePlan], out: Path, *, texts: dict[str, list[Paragraph]] | None = None, notes: dict[int, list[Paragraph]] | None = None, cont_suffix: str = " (tiếp)", section_layout: int | None = None, media: "MediaPlacer | None" = None) -> BuildResult`. Có `texts`/`notes` thì thay đoạn của khối/notes tương ứng (dùng cho dịch).

Quy tắc:
- Mỗi `SlidePlan` tạo slide từ `layouts[layout_index]`. Các khối gán cùng một placeholder được đổ nối tiếp theo thứ tự `placements`, giữ cấp và run. Xoá `normAutofit`/`spAutoFit` khỏi `bodyPr`.
- Khối có vai trò `footer` không được đổ, chữ của nó đưa vào `dropped_texts`. Khối `section_label` không đổ vào slide nội dung. Khi nhãn chương khác slide trước và `section_layout` khác `None`, chèn trước một slide section có tiêu đề là chữ nhãn chương. Chữ nhãn chương nằm trên slide section nên không tính là mất.
- Chia slide: nếu có `split_after_block_id` thì chia ở đó trước. Sau đó với mỗi placeholder `body` vẫn chưa `fits`, dùng `split_to_fit`. Slide phần 2 trở đi lặp lại tiêu đề + `cont_suffix`. Notes đặt ở phần 1, phần sau có notes rỗng. `overflow_flag` → thêm vào `flagged` lý do `"Có đoạn quá dài, không chia nhỏ hơn được — cần xem lại"`.
- Placeholder không được dùng thì xoá khỏi slide, tránh để chữ mẫu "Click to add text".
- Khối `picture`/`table`/`group`/`chart` được giao cho `media.place(slide, block, ph_or_area)` (Task 7). Nếu `media is None` thì bỏ qua và ghi vào `flagged`. Mọi khối `group`/`chart` đều ghi `flagged` lý do `"Hình/biểu đồ vẽ tay được chép nguyên — nên xem lại"` (spec 6.3).
- Khối có `placeholder_idx == -1` (vùng tự do) cũng do `media` đặt. Với khối chữ `citation` không có placeholder thì tạo text box ở đáy vùng nội dung, cỡ chữ bằng `0.6 ×` cỡ body.

- [ ] **Step 1: Viết test hỏng** (dùng các `SlidePlan` viết tay trong test):
  - `test_build_text_roundtrip`: dựng slide 1, 2 và 7 → `verify_text(..., dropped={"CONF 2026 · 2"}, allowed_suffixes=(" (tiếp)",))` cho kết quả `ok`.
  - `test_build_keeps_runs_and_levels`: trong output vẫn có run đậm, run `baseline=30000` và đoạn `level == 1`. Không placeholder nào có `normAutofit`.
  - `test_build_splits_long_slide`: slide 3 thành ≥ 2 slide, slide sau có tiêu đề kết thúc bằng `" (tiếp)"`, notes chỉ ở phần 1, chữ đủ.
  - `test_build_inserts_section_slides`: `section_layout` đặt sẵn → có slide section `"01 · INTRO"` trước slide 2 và `"02 · METHODS"` trước slide 4. Không có section trước slide 3 vì cùng nhãn.
  - `test_build_removes_unused_placeholders`: không còn placeholder rỗng.
  - `test_build_does_not_modify_inputs`: hash của file nguồn và template không đổi.
- [ ] **Step 2–4:** FAIL → viết → PASS.
- [ ] **Step 5: Commit** `"feat: dựng slide từ kế hoạch, chia slide, chèn section"`.

---

### Task 7: Ảnh, bảng, nhóm hình và co giãn vùng

**Files:**
- Create: `src/autoslide/build_media.py`, `tests/test_build_media.py`

**Interfaces:**
- Produces:
  - `fit_box(src_w: int, src_h: int, area: tuple[int,int,int,int]) -> tuple[int,int,int,int]`: hộp giữ tỉ lệ, đặt giữa vùng `(left, top, width, height)`.
  - `crop_fractions(img_w: int, img_h: int, box_w: int, box_h: int) -> tuple[float,float,float,float]`: crop trái/trên/phải/dưới để ảnh lấp đầy khung mà không méo.
  - `class MediaPlacer` có `place(slide, block: Block, target: PlaceholderInfo | tuple[int,int,int,int]) -> None`:
    - **picture** vào placeholder ảnh: `insert_picture` rồi đặt `crop_*` theo `crop_fractions`. Vào vùng tự do: `add_picture` theo `fit_box`.
    - **table/chart/group:** deepcopy `shape_xml` vào `spTree`, rồi đặt `xfrm` theo `fit_box`. Với group thì scale `chOff`/`chExt` theo cùng hệ số. Với mọi `r:embed`/`r:link`/`r:id` bên trong: chép part ảnh/biểu đồ từ deck nguồn sang (`slide.part.relate_to(...)`) và đổi id. `MediaPlacer.__init__(source_pres: Presentation)` để tra part nguồn.
  - `content_area(layout: LayoutInfo, slide_w: int, slide_h: int) -> tuple[int,int,int,int]`: hộp bao của các placeholder body/picture/table/chart. Nếu layout không có thì lấy vùng bên dưới tiêu đề, cách lề 0.5 inch.

- [ ] **Step 1: Viết test hỏng**:
  - `test_crop_fractions_wide_into_square`: ảnh 400×200 vào khung 100×100 → crop trái = phải = 0.25, trên = dưới = 0.
  - `test_fit_box_centers`.
  - `test_place_picture_not_distorted`: ảnh slide 5 đặt vào placeholder ảnh → `(w·(1−cl−cr))/(h·(1−ct−cb))` của ảnh hiển thị bằng tỉ lệ khung ±1%.
  - `test_table_copied_with_text`: slide 4 → output có `graphicFrame` chứa bảng và chữ trong ô được giữ.
  - `test_group_scaled_into_area_keeps_aspect` (Review Focus 4): deck 4:3 → template 16:9. Tỉ lệ w/h của nhóm giữ ±1% và nằm gọn trong `content_area`.
  - `test_group_with_picture_inside_keeps_image`: nhóm có ảnh bên trong (tạo trong test) → mở output, ảnh vẫn hiển thị (blob tồn tại, rel hợp lệ).
- [ ] **Step 2–4:** FAIL → viết → PASS. Nối `MediaPlacer` vào `build_deck` và chạy lại `pytest tests/test_build.py`.
- [ ] **Step 5: Commit** `"feat: chép ảnh, bảng, nhóm hình vào slide mới"`.

---

### Task 8: Lớp AI, bản giả, lập kế hoạch slide kèm dự phòng

**Files:**
- Create: `src/autoslide/ai/__init__.py`, `ai/base.py`, `ai/fake.py`, `src/autoslide/analyze.py`, `src/autoslide/prompts.py`, `tests/test_analyze.py`

**Interfaces:**
- Produces:
  - `ai/base.py`: `class LLMError(Exception)`; `class LLM(Protocol)`: `name: str`; `supports_images: bool`; `complete_json(self, system: str, user: str, schema: type[T], images: list[bytes] = ()) -> T` (ném `LLMError` nếu sai schema hoặc bị từ chối).
  - `ai/fake.py`: `FakeLLM(responses: list[BaseModel | dict | Exception], supports_images=False)`. Mỗi lần gọi trả phần tử tiếp theo (dict thì `schema.model_validate`). `calls: list[tuple[str, str, type]]`.
  - `analyze.py`:
    - `slide_payload(slide: SourceSlide) -> dict`: chỉ gồm id, kind, vị trí (theo % slide), font_pt và **200 ký tự đầu** của chữ từng khối, để giảm token.
    - `validate_plan(plan: SlidePlan, slide: SourceSlide, layouts: list[LayoutInfo]) -> list[str]`: lỗi khi id không tồn tại; khi khối không phải footer/section_label mà không được đặt hoặc bị đặt hai lần; khi placeholder không thuộc layout; khi đặt khối ảnh/bảng/nhóm vào placeholder chữ. **Chốt chặn chống mất chữ:** chỉ được gắn `footer` cho khối ≤ 60 ký tự **và** chứa số slide `slide.no`. Ngược lại là lỗi.
    - `fallback_plan(slide: SourceSlide, layouts: list[LayoutInfo]) -> SlidePlan`: luật thuần code, **chỉ dựa vào cấu trúc placeholder, không dựa vào tên layout**. Tiêu đề là khối chữ có `font_pt` lớn nhất trong nửa trên. Nhãn chương khớp `^\d{1,2}\s*[·.\-–]\s*\S`. Footer theo đúng luật ở `validate_plan`. Dòng trích dẫn khớp năm `(19|20)\d\d` cùng `;`, `:` hoặc `et al`. Còn lại là body. Chọn layout: có ảnh và chữ thì layout có placeholder picture + body; có bảng thì layout có table, nếu không có thì layout title + body lớn nhất; chỉ có tiêu đề thì layout chỉ có title.
    - `plan_slide(llm: LLM, slide: SourceSlide, layouts: list[LayoutInfo], *, retries: int = 2) -> tuple[SlidePlan, bool]`: gọi AI, kiểm tra. Sai thì gửi lại kèm danh sách lỗi. Hết lượt thì trả `fallback_plan` và `True` (đã dùng dự phòng).
    - `pick_section_layout(layouts: list[LayoutInfo]) -> int | None`: layout có title cộng tối đa 2 body nhỏ, không có picture/table/chart, và tên chứa "section"/"chương"/"divider" nếu có nhiều ứng viên. Không có thì `None`.
  - `prompts.py`: `PLAN_SYSTEM: str` (tiếng Anh, nói rõ: chỉ trả JSON đúng schema, không viết lại chữ, chỉ dùng `block_id` có sẵn, quy tắc ảnh/chia slide trong spec mục 2), `plan_user(payload: dict, layouts: list[LayoutInfo]) -> str`.

- [ ] **Step 1: Viết test hỏng**:
  - `test_plan_slide_accepts_valid_ai_plan`: FakeLLM trả plan hợp lệ → dùng đúng plan đó, `used_fallback is False`.
  - `test_plan_slide_retries_then_succeeds`: lần 1 id bịa, lần 2 hợp lệ → 2 lần gọi, lần 2 có chuỗi lỗi trong `user`.
  - `test_plan_slide_falls_back_after_retries`: 3 lần sai hoặc `LLMError` → `used_fallback is True` và plan qua `validate_plan`.
  - `test_ai_cannot_drop_content_as_footer`: AI gắn `footer` cho một bullet dài → lỗi.
  - `test_fallback_plan_ignores_layout_names` (Review Focus 2): template đổi tên layout thành `["Bố cục 1", ..., "Google Shape;12"]` → plan cho slide 2, 4, 5 hợp lệ. Slide 4 đặt bảng vào table placeholder hoặc vùng tự do. Slide 5 chọn layout có picture.
  - `test_payload_truncates_text`.
- [ ] **Step 2–4:** FAIL → viết → PASS.
- [ ] **Step 5: Commit** `"feat: lập kế hoạch slide bằng AI, có kiểm tra và dự phòng"`.

---

### Task 9: Nhà cung cấp AI thật: Claude và Gemini/Gemma

**Files:**
- Create: `src/autoslide/ai/claude.py`, `src/autoslide/ai/gemini.py`, `tests/test_providers.py`

**Interfaces:**
- Produces:
  - `ClaudeLLM(api_key: str, model: str = "claude-opus-5-5", effort: str = "medium", client=None)`, `supports_images = True`. Gọi như trong Global Constraints, schema lấy từ `schema.model_json_schema()` (đặt `additionalProperties: false` ở mọi object). Ảnh gửi dạng block `{"type": "image", "source": {"type": "base64", ...}}` trước khối text. `stop_reason == "refusal"` → `LLMError("refusal")`. Đọc khối `text` đầu tiên → `schema.model_validate_json`. Bắt lỗi theo chuỗi `anthropic.RateLimitError` → `APIStatusError` → `APIConnectionError` và đổi thành `LLMError` (SDK đã tự thử lại 2 lần).
  - `GeminiLLM(api_key: str, model: str | None = None, client=None)`: model = tham số, hoặc `AUTOSLIDE_GEMINI_MODEL`, hoặc `"gemma-4-31b-it"`. **Model Gemma trên Gemini API có thể không hỗ trợ `system_instruction` và JSON mode/`response_schema`.** Builder kiểm tra tài liệu `google-genai` hiện hành (context7) và chạy thử một lần. Với model bắt đầu bằng `gemma`: ghép system vào đầu nội dung user, thêm schema JSON vào prompt, lấy JSON từ text (bỏ rào ```` ```json ````), rồi `model_validate_json`. Với model `gemini-*`: dùng `response_mime_type="application/json"` + `response_schema`. Gặp 429 thì chờ theo `retry-after` hoặc lần lượt 5, 15, 45 giây, tối đa 3 lần. `supports_images` = `True` nếu model nhận ảnh. Xác minh khi chạy thử; nếu không chắc thì để `False`.
  - `make_llm(kind: Literal["claude","gemini"]) -> LLM`: đọc key từ biến môi trường. Thiếu key → `LLMError("Chưa có key ... — xem README")`.

- [ ] **Step 1: Viết test hỏng** (client giả, không gọi mạng):
  - `test_claude_builds_request_and_parses`: client giả ghi lại kwargs → assert `model == "claude-opus-5-5"`, có `fallbacks == "default"`, `"server-side-fallback-2026-07-01" in betas`, `output_config["format"]["type"] == "json_schema"`. Trả về object có `stop_reason="end_turn"` và text JSON hợp lệ → được `SlidePlan`.
  - `test_claude_refusal_raises`.
  - `test_gemini_gemma_parses_fenced_json`: client giả trả `"```json\n{...}\n```"` → parse được. Assert không truyền `system_instruction` khi model là gemma.
  - `test_gemini_retries_on_429` (monkeypatch `time.sleep`).
  - `test_make_llm_missing_key_message_vietnamese`.
- [ ] **Step 2–4:** FAIL → viết → PASS.
- [ ] **Step 5: Kiểm tra thật (thủ công, không commit kết quả):** nếu có `GEMINI_API_KEY`, chạy `python -c "from autoslide.ai.gemini import GeminiLLM; ..."` với một `SlidePlan` nhỏ. Ghi tên model đã xác minh và việc model có nhận ảnh hay không vào `README.md`.
- [ ] **Step 6: Commit** `"feat: nhà cung cấp AI Claude và Gemini/Gemma"`.

---

### Task 10: Dịch và file đối chiếu song ngữ

**Files:**
- Create: `src/autoslide/translate.py`, `tests/test_translate.py`
- Modify: `src/autoslide/prompts.py` (thêm `TRANSLATE_SYSTEM`, `translate_user(...)`)

**Interfaces:**
- Produces:
  - Pydantic `TranslatedBlock(block_id: str, paragraphs: list[str])`, `TranslationBatch(blocks: list[TranslatedBlock])`.
  - `translate_deck(llm: LLM, slides: list[SourceSlide], target_lang: str, keep_terms: list[str]) -> tuple[dict[str, list[Paragraph]], dict[int, list[Paragraph]], list[tuple[int, str]]]`. Kết quả gồm: bản dịch theo `block_id`, notes đã dịch theo số slide, và `flagged`. Mỗi lần gọi dịch một slide (các khối chữ + notes). Notes dùng id giả `"s{n}notes"`. Số đoạn dịch phải bằng số đoạn gốc. Mỗi đoạn qua `check_translation`. Đoạn sai thì gửi lại một lần kèm lỗi. Vẫn sai thì **giữ chữ gốc** cho cả khối và thêm vào `flagged` lý do `"Không dịch được an toàn — giữ nguyên chữ gốc"`. Đoạn dịch giữ cấp của đoạn gốc. Nếu số run giống nhau thì giữ định dạng từng run, nếu không thì gộp thành một run mang định dạng của run đầu.
  - Chỉ dịch khối `text`. Bảng/nhóm hình giữ nguyên và được thêm vào `flagged` lý do `"Chữ trong bảng/hình chưa được dịch"` (bản sau mới làm).
  - `write_bilingual(path: Path, slides, translations, notes_tr) -> Path`: file HTML tự chứa, mỗi slide là một bảng 2 cột Gốc | Bản dịch, tiêu đề tiếng Việt, có câu `"Bản dịch do AI làm, bác sĩ cần đọc lại trước khi trình bày."`.
  - `TRANSLATE_SYSTEM`: dịch sát, không tóm tắt, không thêm/bỏ ý; giữ nguyên số, đơn vị, %, giá trị p, khoảng tin cậy, tên thuốc, tên người, dòng trích dẫn và `keep_terms`; trả đúng số đoạn.

- [ ] **Step 1: Viết test hỏng**:
  - `test_translate_happy_path`: FakeLLM trả bản dịch đúng số đoạn → có bản dịch, `flagged == []`.
  - `test_translate_wrong_paragraph_count_retries`.
  - `test_translate_missing_number_falls_back_to_original`: hai lần đều thiếu `"40%"` → chữ gốc được giữ, có trong `flagged`.
  - `test_bilingual_file_contains_both`.
  - `test_build_with_translations_places_translated_text`: `build_deck(..., texts=...)` → output chứa chữ đã dịch, không chứa chữ gốc của khối đó.
- [ ] **Step 2–4:** FAIL → viết → PASS.
- [ ] **Step 5: Commit** `"feat: dịch slide có kiểm chứng và file đối chiếu song ngữ"`.

---

### Task 11: Ảnh minh hoạ từ Wikimedia Commons

**Files:**
- Create: `src/autoslide/images.py`, `tests/test_images.py`, `tests/data/commons_search.json` (JSON giả tự viết theo đúng cấu trúc API, có 3 ảnh: một CC BY-SA, một "All rights reserved", một Public domain)
- Modify: `src/autoslide/prompts.py` (thêm `IMAGE_CHECK_SYSTEM`)

**Interfaces:**
- Produces:
  - `@dataclass ImageCandidate(title: str, file_url: str, page_url: str, author: str, license: str, width: int, height: int)`
  - `search_commons(query: str, limit: int = 8, http: httpx.Client | None = None) -> list[ImageCandidate]`: `GET https://commons.wikimedia.org/w/api.php` với `action=query&format=json&generator=search&gsrnamespace=6&gsrsearch=<query> filetype:bitmap&gsrlimit=<limit>&prop=imageinfo&iiprop=url|size|extmetadata&iiurlwidth=1600`. Gửi header `User-Agent: AutoSlide/0.1 (https://github.com/thanhpd123/auto-slide-pdf)`. `author` lấy từ `extmetadata.Artist` (bỏ thẻ HTML), `license` từ `LicenseShortName`. Chỉ giữ giấy phép hợp lệ theo Global Constraints. Bỏ ảnh < 800 px chiều rộng.
  - `license_ok(short_name: str) -> bool`.
  - Pydantic `ImageVerdict(relevant: bool, reason: str)`. Hàm `check_image(llm, img_bytes, slide_title, query) -> bool`: nếu `llm.supports_images` là `False` thì trả `True` và người gọi ghi `flagged` `"Ảnh chưa được AI xem lại — bác sĩ nên kiểm tra"`.
  - `slide_wants_image(slide: SourceSlide, plan: SlidePlan, layout: LayoutInfo) -> bool`: theo luật trong Global Constraints, cộng thêm điều kiện layout có placeholder picture và `plan.image_query` không rỗng.
  - `add_images(pptx: Path, slides, plans, layouts, llm, out_dir: Path, http=None) -> tuple[list[dict], list[tuple[int,str]]]`: với mỗi slide cần ảnh thì tìm, tải (giới hạn 5 MB), kiểm tra, crop bằng `crop_fractions`, chèn vào placeholder picture và thêm một dòng nguồn nhỏ (`"Ảnh: <author>, <license>, Wikimedia Commons"`) dưới ảnh. Lưu ảnh vào `out_dir/images/slideNN_<slug>.<ext>`. Trả danh sách credit.
  - `write_credits(path: Path, credits: list[dict]) -> Path`: mỗi ảnh có các dòng `Used on`, `Title`, `Source`, `File URL`, `Author`, `License`.

- [ ] **Step 1: Viết test hỏng** (mock HTTP bằng `httpx.MockTransport`):
  - `test_search_filters_licenses`: chỉ còn 2 ảnh hợp lệ, author không còn thẻ HTML.
  - `test_slide_wants_image_rules`: slide có bảng → `False`; slide > 400 ký tự → `False`; slide ngắn + layout có picture + có query → `True`.
  - `test_add_images_writes_credits_and_inserts`: dùng ảnh tải giả, output có ảnh trong placeholder picture, `credits.txt` có link nguồn và giấy phép.
  - `test_check_image_without_vision_returns_true_and_flags`.
- [ ] **Step 2–4:** FAIL → viết → PASS.
- [ ] **Step 5: Commit** `"feat: tìm và chèn ảnh minh hoạ có giấy phép"`.

---

### Task 12: Render ra PDF và ảnh xem trước

**Files:**
- Create: `src/autoslide/render.py`, `tests/test_render.py`

**Interfaces:**
- Produces:
  - `find_soffice() -> str | None`: tìm theo `AUTOSLIDE_SOFFICE`, `PATH`, rồi `C:\Program Files\LibreOffice\program\soffice.exe`.
  - `class RenderError(Exception)`
  - `to_pdf(pptx: Path, out_dir: Path, timeout: int = 180) -> Path`: chạy `soffice --headless --convert-to pdf --outdir <out_dir> <pptx>` với `-env:UserInstallation=file:///<tmp>` riêng cho mỗi lần chạy (để chạy song song được).
  - `pdf_to_pngs(pdf: Path, out_dir: Path, width_px: int = 1280) -> list[Path]` bằng pypdfium2, đặt tên `slide-01.png`, ...

- [ ] **Step 1: Viết test hỏng**: `test_render_roundtrip` (đánh dấu `pytest.mark.skipif(find_soffice() is None)`): dựng deck giả qua `build_deck` → PDF có số trang bằng số slide → số PNG bằng số trang. `test_missing_soffice_raises_vietnamese` (monkeypatch `find_soffice` trả `None`) → thông báo chứa `"Chưa cài LibreOffice"`.
- [ ] **Step 2–4:** FAIL → viết → PASS. Máy dev hiện **chưa có LibreOffice**: cài bằng `winget install TheDocumentFoundation.LibreOffice`, hoặc chạy test trong Docker `python:3.12` + `apt-get install libreoffice-impress`. Ghi cách làm vào README.
- [ ] **Step 5: Commit** `"feat: render PDF và ảnh xem trước"`.

---

### Task 13: Nối các bước, báo cáo tiếng Việt, lệnh CLI

**Files:**
- Create: `src/autoslide/pipeline.py`, `src/autoslide/__main__.py`, `README.md`, `tests/test_pipeline.py`, `tests/test_cli.py`

**Interfaces:**
- Consumes: mọi thứ ở trên.
- Produces:
  - `@dataclass Options(target_lang: str | None = None, keep_terms: list[str] = [], add_images: bool = True, export_pdf: bool = False, previews: bool = True)` (dùng `field(default_factory=list)`)
  - `@dataclass Report(split: list[tuple[int, int]], layouts: list[tuple[int, str]], flagged: list[tuple[int, str]], dropped: list[str], fallback_slides: list[int], text_check: TextCheck | None)`, với `to_vietnamese() -> str` để in báo cáo dễ đọc. `split` là (slide gốc, số phần), `layouts` là (slide gốc, tên layout).
  - `@dataclass Result(pptx: Path, pdf: Path | None, previews: list[Path], credits: Path | None, bilingual: Path | None, report: Report)`
  - `class InputError(Exception)`: thông báo tiếng Việt.
  - `detect_lang(slides: list[SourceSlide]) -> str`: trả `"vi"` nếu ≥ 5% chữ cái mang dấu tiếng Việt (ă â đ ê ô ơ ư và các dấu thanh), ngược lại trả `"other"`. Dùng để chọn `cont_suffix` khi không dịch.
  - `restyle(source: Path, template: Path, out_dir: Path, llm: LLM, options: Options = Options(), progress: Callable[[str], None] = lambda m: None) -> Result`. Thứ tự: kiểm tra đầu vào → `extract_deck` → `read_layouts` → `plan_slide` cho từng slide (progress `"Đang chọn bố cục slide {i}/{n}…"`) → dịch nếu có `target_lang` → `build_deck` → `add_images` → `verify_text` (chỉ khi không dịch) → render PDF/ảnh xem trước nếu có LibreOffice. Không có LibreOffice thì thêm `flagged` `(0, "Không tạo được ảnh xem trước vì máy chưa cài LibreOffice")` → ghi `report.txt`. **Nếu `text_check` không đạt thì vẫn trả file nhưng đưa từng đoạn thiếu/thừa vào `flagged`, và CLI trả mã thoát 2.**
  - Kiểm tra đầu vào: không mở được bằng python-pptx (zip hỏng, `.ppt` đổi đuôi) → `InputError("Không đọc được file '<tên>'. Hãy lưu lại dưới dạng .pptx rồi thử lại.")`. `TemplateError` → `InputError("Template không có bố cục nào có ô nội dung…")`.
  - CLI: `python -m autoslide restyle --source S --template T --out DIR --ai {gemini,claude} [--lang LANG] [--keep-term X ...] [--no-images] [--pdf]`. In tiến độ và báo cáo. `InputError`/`LLMError` → in thông báo tiếng Việt và mã thoát 1, không in traceback.
  - `README.md`: cài đặt, biến môi trường (`GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, `AUTOSLIDE_GEMINI_MODEL`, `AUTOSLIDE_SOFFICE`), ví dụ lệnh với đường dẫn chung chung, cảnh báo quyền riêng tư của Gemini free tier (spec 5.3). **Không** nhắc tới bộ mẫu thật.

- [ ] **Step 1: Viết test hỏng**:
  - `test_pipeline_end_to_end_fake_llm`: FakeLLM luôn ném `LLMError` → toàn bộ chạy bằng dự phòng, ra `.pptx`, `report.text_check.ok`, `fallback_slides == [1..7]`, slide 3 nằm trong `split`.
  - `test_pipeline_translate_mode_skips_text_check_and_writes_bilingual`.
  - `test_pipeline_no_images_option`.
  - `test_report_vietnamese_mentions_split_and_flagged`.
  - `test_cli_bad_file_vietnamese_error` (Review Focus 5): file `.pptx` thực chất là văn bản → mã thoát 1, stderr chứa `"Không đọc được file"`, không có `"Traceback"`.
- [ ] **Step 2–4:** FAIL → viết → PASS. Chạy toàn bộ `pytest -v` → PASS.
- [ ] **Step 5: Commit** `"feat: pipeline làm đẹp slide, báo cáo tiếng Việt và lệnh CLI"`.

---

### Task 14: Chạy nghiệm thu với bộ mẫu thật (thủ công, không commit đầu ra)

**Files:** không tạo file trong repo.

- [ ] **Step 1:** Chạy với bộ mẫu riêng, đầu ra để **ngoài repo**:
  `python -m autoslide restyle --source "<slide cũ trong D:\File\ppt>" --template "<template trong D:\File\ppt>" --out "D:\File\autoslide-out\gemma" --ai gemini --pdf`
  rồi chạy lại với `--ai claude` vào `...\claude`. Không ghi vào `D:\File\ppt`, không sửa file gốc.
- [ ] **Step 2:** Mở **từng** ảnh xem trước của cả hai bản, so với bản làm tay trong bộ mẫu. Ghi lại (chỉ trên máy, ngoài repo) những slide tràn chữ, lệch khung, chọn layout sai, ảnh sai chủ đề.
- [ ] **Step 3:** Với mỗi lỗi, viết **test tái hiện bằng dữ liệu tự sinh** (không chép chữ thật) vào test của module liên quan, sửa, rồi chạy `pytest -v`. Lặp lại từ Step 1 đến khi báo cáo không còn lỗi tràn và `text_check.ok`.
- [ ] **Step 4:** Thử chế độ dịch với `--lang vi` (hoặc `--lang en` nếu bản gốc tiếng Việt). Đọc file song ngữ, ghi lại chất lượng của Gemma so với Claude để báo cho chủ dự án.
- [ ] **Step 5:** Trước khi commit các bản sửa: `git status` không có file `.pptx/.pdf/.png`. `git grep` tên file, tên bài, tên hội nghị, tên bệnh viện của bộ mẫu → không có kết quả. Commit `"fix: sửa lỗi phát hiện khi nghiệm thu"`. **Hỏi chủ dự án trước khi push.**
