# Ghi chú: Base project iOS (triển khai SAU khi hoàn thiện Android)

> Đây là ghi chú khảo sát, chưa phải kế hoạch thực thi. Khi nào bắt đầu làm iOS, dùng file này
> làm đầu vào để viết `PLAN-IOS-BASE-PROJECT.md` theo đúng khuôn của
> `PLAN-ANDROID-BASE-PROJECT.md`.

## 1. Kết luận khảo sát (tra cứu ngày 2026-08-15)

**Không tồn tại "Now in Android phiên bản iOS"** — không có repo chính chủ Apple nào vừa
multi-module, vừa có kiến trúc rõ ràng, vừa có test đầy đủ. Apple **cố tình không đưa ra
kiến trúc chính thức** cho app; sample code của Apple (Landmarks, Food Truck, Backyard Birds,
Destination Video) chỉ dạy idiom SwiftUI/SwiftData, không dùng làm base được.

Hệ quả: bên iOS phải **ghép từ nhiều nguồn**, không fork được một repo duy nhất.

## 2. Các repo tham chiếu đã khảo sát

| Repo | Vai trò | Mạnh | Yếu / cảnh báo |
|---|---|---|---|
| [Dimillian/IceCubesApp](https://github.com/Dimillian/IceCubesApp) | Gần vai NiA nhất | ~7k sao, maintain tích cực (~2.657 commit), thuần SwiftUI, multiplatform, chia SPM packages (Status/Timeline, Notifications, Explore, Conversations, Account, Env, Network), domain feed nội dung rất giống NiA | ⚠️ **License AGPL-3.0** (xem mục 3). MVVM đơn giản, không có tầng UseCase, ít test |
| [pointfreeco/isowords](https://github.com/pointfreeco/isowords) | Học modular hóa + test | **86 module**, kỷ luật modular hóa và testing tốt nhất giới iOS, có preview app riêng cho từng feature | Khóa chặt vào TCA (cam kết lớn); domain là game |
| [kudoleh/iOS-Clean-Architecture-MVVM](https://github.com/kudoleh/iOS-Clean-Architecture-MVVM) | Bộ xương phân tầng | Đúng dạng template: DIContainer, FlowCoordinator, DTO, caching, tách Domain/Data/Presentation rõ | Chủ yếu UIKit (chỉ 1 view SwiftUI), quy mô nhỏ, hơi cũ |
| [nalexn/clean-architecture-swiftui](https://github.com/nalexn/clean-architecture-swiftui) | Bộ xương Clean Architecture thuần SwiftUI | License permissive | App demo nhỏ, không multi-module thật |
| Apple sample code | Tra cứu idiom API mới | Chính chủ, luôn cập nhật | Demo một màn, không kiến trúc |

## 3. ⚠️ Khác biệt pháp lý quan trọng so với Android

- Android: NiA + architecture-templates dùng **Apache 2.0** → fork/copy thoải mái.
- iOS: Ice Cubes dùng **AGPL-3.0** (copyleft mạnh) → **đọc để học thì được, copy code vào sản
  phẩm đóng của công ty là rủi ro pháp lý thật sự**.

→ Với iOS, mô hình **"tham chiếu rồi tự viết"** không chỉ là lựa chọn tốt hơn mà là **bắt buộc**.
Ý tưởng kiến trúc không bị bảo hộ bản quyền, nhưng code thì có. Nên hỏi ý kiến pháp lý một lượt
trước khi áp cho toàn công ty.

## 4. Stack dự kiến cho base iOS

- **UI**: SwiftUI + Observation (`@Observable`)
- **Concurrency**: async/await + AsyncSequence (lưu ý Swift 6 strict concurrency: actor,
  `Sendable`, data-race safety — đây là chỗ tốn công nhất và AI hay sai nhất)
- **Modular hóa**: local SPM packages (tương đương Gradle modules)
- **Sinh project**: **Tuist** (hoặc XcodeGen) — đóng vai trò như convention plugins của Gradle
- **Lint/format**: SwiftLint + SwiftFormat (tương đương detekt/ktlint)
- **Test**: Swift Testing (Xcode 16+) hoặc XCTest
- **DI**: init injection thủ công hoặc `swift-dependencies` (Swift **không có** tương đương Hilt)

## 5. Cạm bẫy riêng của iOS

1. **File `.xcodeproj`**: XML khổng lồ, AI sửa dễ hỏng, conflict git liên miên.
   → Bắt buộc chốt ngay từ đầu: dùng **Tuist/XcodeGen** (chỉ sửa manifest Swift), hoặc
   **synchronized folders** của Xcode 16+ kết hợp cấu trúc SPM.
2. **Vòng lặp build/chạy simulator** qua `xcodebuild` rất khổ với AI agent.
   → Cần **XcodeBuildMCP** (MCP server cộng đồng) để Claude build, chạy simulator, đọc log,
   chụp màn hình — đóng vai trò như `./gradlew` bên Android. Gần như bắt buộc.
3. **Đừng dịch quá sát từ Android** → sẽ ra "code Android viết bằng Swift" (anti-pattern).
   SwiftUI không có Activity/Fragment, vòng đời View khác hẳn; nhiều trường hợp iOS làm gọn
   hơn (MV thay vì MVVM đầy đủ). Lấy **cấu trúc và phân tầng**, viết lại theo idiom Swift.

## 6. Chuyển đổi từ base Android — cái gì dùng lại được

**Tái dùng gần như 100%** (giá trị cao nhất, đã làm xong bên Android):
- Cấu trúc `CLAUDE.md` + bảng chỉ đường "màn hình dạng X → theo feature Y"
- Danh sách 4 feature mẫu (login / list / settings / tabs)
- Quy trình review hai lớp, checklist nghiệm thu, quy trình Figma → code
- Module graph và phân tầng (Clean Architecture): `:core:*` → SPM packages cùng tên
- `UiState` sealed class → `enum` với associated values (trùng khít)

**Phải viết lại từ đầu** (0% tái dùng):
- Build system (Gradle → Tuist), DI (Hilt → không có tương đương), Room → SwiftData/GRDB,
  Compose Navigation → NavigationStack, JUnit → Swift Testing, và toàn bộ code UI
  (Compose ↔ SwiftUI nhìn giống nhưng copy-paste được 0%)

**Ước lượng**: tiết kiệm ~40–50% công sức, chủ yếu ở phần ra quyết định và lập kế hoạch.
Phần code gần như viết mới hoàn toàn. Coi base Android như **bản đặc tả (spec)**,
không phải mã nguồn để dịch.

## 7. Công thức triển khai (khi bắt đầu làm)

1. Dựng khung bằng **Tuist** + local SPM packages theo module graph của base Android.
2. Trỏ Claude vào **base Android làm thư mục tham chiếu** (đúng pattern NiA đã dùng),
   kèm luật vàng: *"lấy cấu trúc, phân tầng và tên gọi; viết lại hoàn toàn theo idiom
   Swift/SwiftUI; không bê hạ tầng Android sang"*.
3. Đọc Ice Cubes để học cách chia package + idiom SwiftUI — **chỉ đọc, không copy** (AGPL).
4. Xây 4 feature mẫu cùng khuôn, có test, đúng như Giai đoạn 2 của kế hoạch Android.
5. Viết `CLAUDE.md` cho iOS, chốt review hai lớp tại từng giai đoạn.
6. Cắt các repo tham chiếu khỏi context sau khi dựng nền xong.
