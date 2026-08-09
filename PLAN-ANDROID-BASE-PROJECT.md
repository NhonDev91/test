# Kế hoạch xây dựng Android Base Project (Cách B)

> **Dành cho Claude Code**: File này là playbook đã được chốt từ trước. Khi người dùng yêu cầu
> "thực hiện kế hoạch trong PLAN-ANDROID-BASE-PROJECT.md", hãy đọc kỹ toàn bộ file và thực thi
> theo đúng các giai đoạn bên dưới. Mọi quyết định kiến trúc trong file này đã được thảo luận
> và chốt — không cần hỏi lại, trừ các mục đánh dấu ❓ (cần người dùng cung cấp).

## 1. Mục tiêu

Xây dựng một **Android base project chuẩn** (Kotlin + Jetpack Compose, multi-module,
Clean Architecture) để:

- Làm nền cho **toàn bộ project mới của công ty**.
- Mỗi khi cần màn hình mới: mở session Claude Code mới, đưa link Figma, Claude implement
  **đúng khuôn** của base project.
- Codebase phải sạch, nhất quán, có feature mẫu đầy đủ để Claude "soi bài" khi code.

## 2. Quyết định đã chốt: Cách B — "Template + NiA tham chiếu"

- **Nền tảng**: dùng [`android/architecture-templates`](https://github.com/android/architecture-templates)
  (branch `multimodule`) làm khung xương — sạch, nhẹ, không nợ domain demo.
- **Tham chiếu**: clone [`android/nowinandroid`](https://github.com/android/nowinandroid) (NiA)
  làm **thư mục anh em** (KHÔNG nằm trong project) để tham khảo cách giải bài toán
  (designsystem, offline-first, testing...) trong giai đoạn dựng nền.
- **Luật vàng**: kiến trúc/convention của **repo này** là chuẩn duy nhất. NiA chỉ để tham khảo
  *cách giải quyết vấn đề* — mọi code viết ra phải theo khuôn của repo này, **không copy
  hạ tầng của NiA** (convention plugins phức tạp, protobuf DataStore, flavor demo/prod,
  Firebase... trừ khi kế hoạch này yêu cầu rõ).
- **Sau khi dựng nền xong**: cắt NiA khỏi context. Các session thường ngày chỉ nhìn thấy
  base project.

Lý do đã phân tích: "cộng dần" từ template sạch an toàn hơn "trừ dần" từ NiA (nợ domain news
dính sâu, build nặng, dạy hư Claude ở các session sau). Rủi ro của Cách B (lệch khuôn khi
tham khảo NiA) là rủi ro một lần, kiểm soát được bằng luật vàng ở trên.

## 3. Kiến trúc đích

- **Ngôn ngữ/UI**: Kotlin, Jetpack Compose, Material 3.
- **Pattern**: MVVM + UDF (UI → ViewModel expose `StateFlow<UiState>` → UseCase → Repository
  → DataSource). Một chiều dữ liệu, sealed `UiState` (Loading / Success / Error).
- **DI**: Hilt. **Async**: Coroutines + Flow. **Network**: Retrofit + OkHttp + kotlinx-serialization.
  **Local**: Room + DataStore (Preferences, KHÔNG dùng protobuf). **Navigation**: Compose Navigation.
- **Cấu trúc module**:
  ```
  :app                      — điểm vào, navigation graph tổng
  :core:designsystem        — theme, màu, typography, component dùng chung
  :core:ui                  — UI util, component gắn business nhẹ
  :core:common              — util thuần Kotlin, Result wrapper, dispatcher
  :core:network             — Retrofit, DTO, API service
  :core:database            — Room, entity, DAO
  :core:datastore           — DataStore preferences
  :core:data                — repository (gộp network + database)
  :core:domain              — use case
  :core:testing             — test util, fake, rule dùng chung
  :feature:<tên>            — mỗi feature một module
  ```
- **Chất lượng**: detekt + ktlint (bắt buộc, chạy trong CI nếu có), unit test cho ViewModel
  và UseCase của mọi feature mẫu.
- ❓ **Package name**: hỏi người dùng (ví dụ `com.<company>.<app>`). Nếu chưa có câu trả lời,
  tạm dùng `com.company.baseproject` và ghi chú lại để đổi sau.

## 4. Các giai đoạn thực thi

### Giai đoạn 0 — Chuẩn bị (người dùng làm hoặc yêu cầu Claude làm)

1. Clone hai repo thành thư mục anh em:
   ```bash
   git clone -b multimodule https://github.com/android/architecture-templates.git <thư-mục-base-project>
   git clone --depth 1 https://github.com/android/nowinandroid.git   # cạnh bên, KHÔNG nằm trong base project
   ```
2. Mở session Claude Code trỏ vào `<thư-mục-base-project>`, thêm NiA làm thư mục tham chiếu:
   `/add-dir ../nowinandroid` (hoặc `claude --add-dir ../nowinandroid`).
3. Xóa lịch sử git của template, khởi tạo repo mới của công ty (`rm -rf .git && git init`),
   giữ ghi chú license Apache 2.0 của template/NiA cho phần code kế thừa.

### Giai đoạn 1 — Dựng nền

1. Đổi package name + tên app theo mục ❓ ở trên; dọn placeholder của template.
2. Chuẩn hóa cấu trúc module theo sơ đồ ở mục 3 (template multimodule đã gần đúng,
   bổ sung module còn thiếu: `:core:designsystem`, `:core:testing`...).
3. Version catalog (`libs.versions.toml`) đầy đủ, cập nhật các dependency lên bản stable mới.
4. Thiết lập detekt + ktlint, cấu hình rule cơ bản, đảm bảo `./gradlew detekt ktlintCheck` pass.
5. Convention plugin Gradle **mức tối giản** (một plugin cho android-library + compose,
   một cho feature module) — tham khảo ý tưởng từ NiA nhưng viết gọn hơn nhiều.
6. Dựng `:core:designsystem`: theme, màu, typography, shape + vài component nền
   (Button, TextField, TopBar, LoadingIndicator, ErrorView...) — tham khảo cách tổ chức
   của NiA `:core:designsystem` nhưng chỉ lấy cấu trúc, không copy branding.
7. Build xanh: `./gradlew assembleDebug` + `./gradlew test` pass.

### Giai đoạn 2 — Xây bộ feature mẫu (harvest từ NiA có kỷ luật)

Mỗi feature mẫu phải **đầy đủ chiều sâu**: Compose UI → ViewModel (StateFlow UiState) →
UseCase → Repository → DataSource, kèm xử lý Loading/Error/Empty và **unit test** cho
ViewModel + UseCase. Tất cả cùng MỘT khuôn — đây là yêu cầu quan trọng nhất.

| # | Feature mẫu | Đại diện cho dạng màn hình | Ghi chú |
|---|---|---|---|
| 1 | `:feature:login` | Form + validation + gọi API + điều hướng khi thành công | Tự viết (NiA không có), auth giả lập qua fake API |
| 2 | `:feature:checklist` | Danh sách + CRUD + empty/loading/error state | Tham khảo pattern list của NiA (feed) |
| 3 | `:feature:settings` | Màn tĩnh + đọc/ghi DataStore | Tham khảo NiA settings dialog, chuyển thành màn hình |
| 4 | `:feature:home` | ViewPager/Tab, nhiều màn con chia sẻ state | HorizontalPager + TabRow |

Sau mỗi feature: chạy build + test + detekt, tự review lại xem có đúng khuôn feature trước không
rồi mới sang feature tiếp theo.

### Giai đoạn 3 — Viết CLAUDE.md (bắt buộc, là linh hồn của base project)

Tạo `CLAUDE.md` ở root với các phần:

1. **Tổng quan kiến trúc**: mô tả module graph, luồng dữ liệu, các thư viện đã chốt.
2. **Quy ước**: đặt tên, tổ chức file trong một feature module (liệt kê cụ thể các file
   cần có khi tạo feature mới), style Kotlin, quy tắc string resource/theme.
3. **Bảng chỉ đường mẫu** (quan trọng nhất):
   - Màn hình dạng form/nhập liệu → làm theo `:feature:login`
   - Màn hình danh sách → làm theo `:feature:checklist`
   - Màn hình settings/preferences → làm theo `:feature:settings`
   - Màn hình nhiều tab/pager → làm theo `:feature:home`
   - Tạo feature mới: các bước cụ thể (tạo module bằng convention plugin, khai báo
     trong `settings.gradle.kts`, nối navigation...)
4. **Luật**: "Kiến trúc và convention của repo này là chuẩn duy nhất. Không tự ý đổi
   thư viện/pattern. Mọi màn hình mới phải có UiState sealed + unit test ViewModel.
   Chạy `./gradlew detekt ktlintCheck test` trước khi coi là xong."
5. **Lệnh thường dùng**: build, test, lint, chạy app.

### Giai đoạn 4 — Cắt NiA & quy trình vận hành

1. Gỡ NiA khỏi context (không `/add-dir` nữa). Base project giờ tự đủ mẫu.
2. Quy trình hàng ngày cho mọi màn hình mới:
   - Mở session Claude Code mới trỏ vào base project (hoặc project con sinh ra từ base).
   - Prompt mẫu: *"Implement màn hình X theo design Figma <link>, tuân thủ kiến trúc trong
     CLAUDE.md, làm theo pattern của `:feature:<mẫu-phù-hợp>`."*
   - Claude dùng Figma MCP đọc design context, code theo khuôn, chạy build + test + lint.
3. Khi nhân rộng cho công ty: mỗi project mới = copy base project (hoặc dùng làm template
   repo trên GitHub), đổi package name, giữ nguyên CLAUDE.md và bộ feature mẫu trong
   giai đoạn đầu để Claude soi, xóa dần mẫu khi feature thật thay thế đủ.

## 5. Checklist nghiệm thu base project

- [ ] `./gradlew assembleDebug` xanh
- [ ] `./gradlew test` xanh (có test thật ở ViewModel/UseCase của cả 4 feature mẫu)
- [ ] `./gradlew detekt ktlintCheck` xanh
- [ ] 4 feature mẫu cùng một khuôn (đối chiếu chéo cấu trúc file, cách viết UiState)
- [ ] `CLAUDE.md` đầy đủ 5 phần ở Giai đoạn 3
- [ ] Không còn tham chiếu nào tới domain demo của template/NiA
- [ ] Smoke test: mở session Claude Code MỚI, yêu cầu một màn hình đơn giản,
      kiểm tra Claude có tự tuân thủ khuôn mà không cần nhắc thêm không

## 6. Các cạm bẫy cần tránh (đã phân tích trước)

- **KHÔNG** clone NiA vào trong thư mục base project — gây nhiễu search, phình context, rối git.
- **KHÔNG** copy hạ tầng NiA (flavor demo/prod, Firebase, protobuf DataStore, baseline
  profile, benchmark module) — base project không cần trong giai đoạn này.
- **KHÔNG** để hai feature mẫu lệch khuôn nhau — 3 mẫu nhất quán tốt hơn 6 mẫu lệch.
- **KHÔNG** giữ NiA trong context sau khi dựng nền xong — tốn token và tăng rủi ro lệch khuôn.
- Feature mẫu **phải đủ layer + có test** — mẫu chỉ có UI sẽ khiến Claude tự chế phần thiếu.
