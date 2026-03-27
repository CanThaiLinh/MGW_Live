# Hướng dẫn Tích hợp MWG_Live SDK cho iOS

## 1. Giới thiệu
**MWG_Live** là SDK cung cấp giải pháp toàn diện để tích hợp tính năng **Livestream** và **phát video trực tiếp** vào ứng dụng iOS của bạn một cách nhanh chóng và tối ưu.

**Lưu ý hiện tại:**
- SDK hiện chỉ hỗ trợ cài đặt thông qua **CocoaPods**.
- SDK **chỉ hỗ trợ build và chạy trên thiết bị thật (Real Device)**, hiện chưa hỗ trợ Simulator do phụ thuộc vào trình phát native bên dưới.

---

## 2. Yêu cầu hệ thống
- **iOS Target**: Tối thiểu iOS 13.0 (hoặc phiên bản mục tiêu cao hơn do dự án của bạn quy định).
- **Công cụ**: Xcode 14+ / Swift 5+.
- **Môi trường chạy**: **Thiết bị thật (iPhone/iPad)**. Việc build trên Simulator sẽ phát sinh lỗi do không có slice kiến trúc tương ứng.

---

## 3. Hướng dẫn cài đặt bằng CocoaPods
Thêm `MWG_Live` vào file `Podfile` của dự án:

```ruby
target 'TênAppCủaBạn' do
  use_frameworks!
  
  # Thêm dòng dưới đây để cài đặt MWG_Live SDK
  pod 'MWG_Live'
end

# Thêm đoạn cấu hình sau vào cuối file Podfile để tránh lỗi thiếu module hoặc mismatch với SohaPlayer
post_install do |installer|

  installer.pods_project.targets.each do |target|
    
    target.build_configurations.each do |config|
      # Fix Swift runtime compatibility
      config.build_settings['BUILD_LIBRARY_FOR_DISTRIBUTION'] = 'YES'
      
      # Optional: giảm lỗi linker
      config.build_settings['DEAD_CODE_STRIPPING'] = 'YES'
    end

    # ✅ Fix MWG_Live thành module để resolve dependency SohaPlayer
    if target.name == 'MWG_Live'
      target.build_configurations.each do |config|
        config.build_settings['DEFINES_MODULE'] = 'YES'
      end
    end

    # ✅ Fix module name mismatch SohaPlayerV2 -> SohaPlayer
    if target.name == 'SohaPlayerV2'
      target.build_configurations.each do |config|
        config.build_settings['PRODUCT_MODULE_NAME'] = 'SohaPlayer'
      end
    end

  end

  # ✅ Inject modulemap cho SohaPlayer nếu thiếu
  soha_framework = File.join(installer.sandbox.root, 'SohaPlayerV2/SohaPlayerV2-1.4.9-o/SohaPlayer.framework')
  modules_dir = File.join(soha_framework, 'Modules')
  FileUtils.mkdir_p(modules_dir)

  File.write(File.join(modules_dir, 'module.modulemap'), <<~MODULEMAP)
    framework module SohaPlayer {
      header "SHAdManager.h"
      header "SHPlayerConfig.h"
      header "SHViewController.h"
      header "SohaPlayerManager.h"
      export *
    }
  MODULEMAP

end
```

Sau đó, tiến hành cài đặt bằng cách mở Terminal ở thư mục chứa file Podfile và chạy lệnh:
```bash
pod install
```

**Lưu ý:** Sau khi chạy lệnh thành công, hãy đảm bảo bạn luôn mở dự án bằng file **`.xcworkspace`** (không dùng `.xcodeproj`).

---

## 4. Cấu hình Info.plist (Bắt buộc)
SDK sử dụng native player bên dưới nên yêu cầu ứng dụng phải khai báo các key định danh cũng như cho phép tải giao thức HTTP. 

Mở file `Info.plist` của bạn dưới dạng Source Code (hoặc Edit trực tiếp Property List) và bổ sung các mã cấu hình sau:

```xml
<key>appkey</key>
<string>YOUR_APP_KEY_HERE</string>

<key>playerid</key>
<string>YOUR_PLAYER_ID_HERE</string>

<key>secretkey</key>
<string>YOUR_SECRET_KEY_HERE</string>

<!-- Cho phép HTTP Request -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```
**Giải thích các key (các key này viết thường toàn bộ):**
- **`appkey`**: Khóa định danh ứng dụng của bạn để giao tiếp với hệ thống MWG.
- **`playerid`**: Mã ID của player được cấp phát riêng cho nền tảng video của bạn.
- **`secretkey`**: Khóa bí mật dùng để xác thực hệ thống.

---

## 5. Cấp quyền truy cập thiết bị
Để tính năng Livestream (phát sóng, ghi hình, thu âm) hoạt động, ứng dụng cần xin quyền truy cập Camera và Microphone:

```xml
<key>NSCameraUsageDescription</key>
<string>Ứng dụng cần quyền truy cập Camera để thực hiện quay và phát Livestream.</string>

<key>NSMicrophoneUsageDescription</key>
<string>Ứng dụng cần quyền truy cập Microphone để thu âm thanh trong quá trình Livestream.</string>
```

---

## 6. Khai báo các mô hình dữ liệu (Models)
SDK quy định các Protocol (kết thúc bằng đuôi `Representable`) cho dữ liệu. Tránh phụ thuộc cứng vào Class, SDK cho phép bạn tuỳ ý ánh xạ dữ liệu theo Models hiện có của app. 

Trước tiên, hãy tạo các Struct hoặc Class tuân thủ các protocol trên. Ví dụ:

```swift
import MWG_Live

// Cấu hình thông số một Video / Stream (để xem Player)
struct DemoPlayerConfig: MWGPlayerConfigRepresentable {
    var videoId: String
    var url: String
}

// Cấu hình thông tin User sử dụng SDK (tuỳ chọn)
struct DemoUser: MWGUserRepresentable {
    var id: String
    var name: String
    var avatar: String?
    var isPremium: Bool
}

// Cấu hình ví tiền (nếu tích hợp in-app purchase hoặc tặng quà live)
struct DemoWallet: MWGWalletRepresentable {
    var walletID: String
    var amount: Double
}
```

---

## 7. Hướng dẫn Khởi tạo & Sử dụng tính năng

### 7.1. Khởi tạo SDK (Bắt buộc)
Tại file `AppDelegate.swift`, hãy import thư viện và gọi hàm khởi tạo ngay ở đầu chu trình sống của ứng dụng:

```swift
import UIKit
import MWG_Live

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        // Khởi tạo MWG_Live SDK
        MWGSDK.initialize()
        
        return true
    }
}
```

### 7.2. Bật Log SDK (Tuỳ chọn)
Mặc định SDK **không in log** ra console. Nếu bạn cần theo dõi hoạt động nội bộ của SDK trong quá trình phát triển, hãy gọi `enableLogSDK()` **trước** `initialize()`:

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    
    // Bật log SDK (tuỳ chọn — chỉ dùng khi cần debug)
    MWGSDK.enableLogSDK()
    
    // Khởi tạo MWG_Live SDK
    MWGSDK.initialize()
    
    return true
}
```

Khi bật log, output sẽ có format:
```
🔍 [MWG_Live SDK] [15:15:15 26/03/2026] [DEBUG] Player initialized successfully
ℹ️ [MWG_Live SDK] [15:15:15 26/03/2026] [INFO] Stream connected
⚠️ [MWG_Live SDK] [15:15:15 26/03/2026] [WARNING] Low bandwidth detected
❌ [MWG_Live SDK] [15:15:15 26/03/2026] [ERROR] Connection lost
```

**Các mức log và icon tương ứng:**

| Icon | Level     | Mô tả                                      |
| :--- | :-------- | :------------------------------------------ |
| 🔍   | `DEBUG`   | Thông tin chi tiết dùng để gỡ lỗi          |
| ℹ️   | `INFO`    | Thông tin trạng thái hoạt động bình thường  |
| ⚠️   | `WARNING` | Cảnh báo các vấn đề tiềm ẩn                |
| ❌   | `ERROR`   | Lỗi nghiêm trọng cần xử lý                |

> **Lưu ý:** Không nên bật log trên bản production, chỉ sử dụng khi phát triển và gỡ lỗi.

### 7.3. Mở màn hình Livestream (Broadcast)
Khi người dùng (Streamer) bấm vào nút phát sóng, bạn tạo ra màn hình Broadcast từ SDK:

```swift
import MWG_Live

class DemoVC: UIViewController {
    
    @objc private func startLive() {
        // 1. Khởi tạo đối tượng Dependency (Truyền RTMP và Stream Key)
        let dependency = MWGBroadcastDependency(
            rtmpURL: "rtmp://ovp-rtmp.sohatv.vn/...",
            streamKey: "...",
            delegate: self // Ánh xạ vào MWGBroadcastDelegate
        )
        
        // 2. Lấy ViewController từ MWGSDK
        let broadcastVC = MWGSDK.createBroadcastVC(dependency: dependency)
        
        // 3. Hiển thị màn hình Livestream
        navigationController?.pushViewController(broadcastVC, animated: true)
    }
}
```

### 7.4. Mở màn hình Xem Live / VOD (Player)
Khi người dùng bấm vào xem 1 luồng Live đang phát hoặc Video (VOD):

```swift
class DemoVC: UIViewController {
    
    @objc private func watchLive() {
        // 1. Cấu hình cấu trúc Models cho Danh sách phát
        let samplePlayers: [any MWGPlayerConfigRepresentable] = [
            DemoPlayerConfig(videoId: "liveId", url: "https://your-domain/live.m3u8"),
            DemoPlayerConfig(videoId: "vodId", url: "https://your-domain/video.m3u8")
        ]
        
        // 2. Khởi tạo Dependency
        let playerDependency = MWGPlayerDependency(
            players: samplePlayers,
            delegate: self // Ánh xạ vào MWGPlayerDelegate
        )

        // 3. Lấy ViewController trình phát
        let playerVC = MWGSDK.createPlayerVC(dependency: playerDependency)

        // 4. Hiển thị Trình phát video
        navigationController?.pushViewController(playerVC, animated: true)
    }
}
```

### 7.5. Chi tiết về Cấu hình Trình phát (Player Dependency)

Để khởi tạo một màn hình Player bằng `MWGSDK.createPlayerVC(dependency:)`, bạn cần truyền vào một đối tượng `MWGPlayerDependency`. Đối tượng này đóng gói toàn bộ dữ liệu và cấu hình hiển thị:

```swift
public struct MWGPlayerDependency {
    public var media: (any MWGPlayerRepresentable)?
    public var viewType: MWGViewType?
    public weak var delegate: (any MWGPlayerDelegate)?
}
```

#### 1. MWGPlayerRepresentable (Dữ liệu Media)
Định nghĩa thông tin của một video hoặc livestream. Thay vì buộc bạn dùng Model của SDK, bạn chỉ cần cho Model của app tuân thủ Protocol này:

- `id` (Int?): ID duy nhất của nội dung media.
- `name` (String?): Tên hoặc tiêu đề của video/livestream.
- `description` (String?): Mô tả chi tiết nội dung.
- `embed` (String?): URL HLS (`.m3u8`) dùng để phát nội dung (quan trọng nhất để player có thể chạy).
- `strkey` (String?): Stream key (dùng cho luồng live).
- `thumbnailUrl` (String?): Đường dẫn ảnh bìa hiển thị trước khi phát.
- `stateLive` (String?): Trạng thái của luồng live (ví dụ: đang live, đã kết thúc, v.v.).
- `rtmpSV` (String?): Máy chủ RTMP (nếu cần thiết cho cấu hình luồng).

#### 2. MWGViewType (Kiểu hiển thị UI)
Quyết định giao diện của trình phát sẽ được tối ưu cho loại nội dung nào:
- `.live`: Tối ưu giao diện cho Livestream (có chat realtime, thả tim, thông tin người đang xem).
- `.video`: Tối ưu giao diện cho Video On Demand - VOD (có thanh tiến trình, nút tua tới/lui).
- `.tab`: Giao diện dạng danh sách lướt (dành cho trải nghiệm giống TikTok/Shorts). Mặc định nếu không truyền.

#### 3. MWGPlayerDelegate (Nhận sự kiện từ Player)
Protocol giúp ứng dụng của bạn biết được trạng thái hiện tại của trình phát video thông qua phương thức `func player(didChangeEvent event: MWGPlayerEvent)`.

Các trạng thái của **MWGPlayerEvent**:
- `ready`: Trình phát đã tải xong dữ liệu, sẵn sàng phát.
- `playing`: Video/Livestream đang được phát.
- `pause`: Video đã bị tạm dừng.
- `stalled`: Quá trình tải dữ liệu bị nghẽn (mạng chậm, đang buffering).
- `finished`: Đã phát xong video (đạt đến cuối file, không áp dụng cho luồng live liên tục).
- `removed`: Trình phát đã bị gỡ bỏ khỏi bộ nhớ (người dùng tắt màn hình).
- `error`: Xảy ra lỗi trong quá trình phát (link hỏng, mất mạng liên tục).

---

## 8. Xử lý các Sự kiện (Delegates)

Để nhận phản hồi từ View của SDK, bạn cần trỏ các ViewController của mình đến các giao thức của SDK:

### 8.1. MWGBroadcastDelegate (Dùng khi tính năng Live)
```swift
extension DemoVC: MWGBroadcastDelegate {
    func broadcastDidStart() {
        print("Trạng thái: Đã bắt đầu phát sóng")
    }
    
    func broadcastDidStop() {
        print("Trạng thái: Đã dừng phát sóng")
    }
    
    func broadcastDidFail(error: any Error) {
        print("Trạng thái: Lỗi phát sóng - \(error.localizedDescription)")
    }
}
```

### 8.2. MWGPlayerDelegate (Dùng khi tính năng Xem)
```swift
extension DemoVC: MWGPlayerDelegate {
    func player(didChangeEvent event: MWGPlayerEvent) {
        print("Sự kiện Player: \(event)")
    }
}
```

### 8.3. MWGModuleDelegate (Sự kiện toàn khối & Tương tác UI đặc biệt)
Delegate này cung cấp các hook mạnh mẽ để bạn tương tác sâu hơn với SDK, chẳng hạn như đóng màn hình `mwgDidClose()` và nạp tiền `mwgWalletUpdated(:)`:

```swift
extension DemoVC: MWGModuleDelegate {
    func mwgDidClose() {
        // Hành động mong muốn khi user nhấn nút x hoặc Thoát khỏi SDK
        navigationController?.popViewController(animated: true)
    }

    func mwgWalletUpdated(walletID: String, amount: Double) {
        // Có thể mở một Alert / Bottom Sheet UI nạp tiền từ phía Native...
        // ... Sau đó cập nhật lại số dư cho UI của SDK
        
        // Ví dụ: Update lại UI của SDK sau khi nạp tiền thành công!
        let newAmount = amount + 50.0 // Giả sử user nạp 50k
        
        if let sdkVC = self.navigationController?.topViewController as? MWGWalletUpdatable {
            sdkVC.updateWalletUI(amount: newAmount)
        }
    }
}
```

---

## 9. Tóm tắt API `MWGSDK`

| Phương thức | Mô tả |
| :--- | :--- |
| `MWGSDK.initialize()` | **Bắt buộc.** Khởi tạo SDK, chuẩn bị player engine và cấu hình runtime. |
| `MWGSDK.enableLogSDK()` | Bật chế độ in log ra console. Mặc định SDK không in log. |
| `MWGSDK.setUser(_:)` | Lưu thông tin người dùng và access token cho phiên SDK. |
| `MWGSDK.clearSession()` | Xoá phiên người dùng đã lưu khi đăng xuất. |
| `MWGSDK.createPlayerVC(dependency:)` | Tạo ViewController cho trình phát video / xem livestream. |
| `MWGSDK.createBroadcastVC(dependency:)` | Tạo ViewController cho phát sóng livestream. |

---

## 10. Troubleshooting (Xử lý lỗi)

| Hiện tượng lỗi | Nguyên nhân & Khắc phục |
| :--- | :--- |
| **Báo lỗi thiếu file header / `No such module` sau khi chạy `pod install`** | Mở Xcode với `xcworkspace` thay vì `xcodeproj`. Nếu vẫn lỗi, đoạn script `post_install` ở Config Podfile của mục 3 sẽ ép Xcode Build thư viện với `DEFINES_MODULE=YES`. |
| **Crash ứng dụng ngay khi mở màn hình Player** | Thường do thiếu một trong 3 key `appkey`, `playerid`, `secretkey` trong **`Info.plist`**. Hãy kiểm tra kỹ. |
| **Crash ứng dụng ngay khi thao tác mở Camera (Broadcast)** | Do iOS cơ chế bảo mật ngắt ép ứng dụng do thiếu khai báo `NSCameraUsageDescription` hoặc `NSMicrophoneUsageDescription` trong `Info.plist`. |
| **Lỗi `Could not build module MWG_Live` trên Simulator** | SDK không build thành công đối với kiến trúc `simulator`. Chỉ test trên **Thiết bị thật**. |
| **Video Play không được (Loading hoài) theo HTTP** | Trong **`Info.plist`** của bạn có thể đang thiếu cụm `NSAppTransportSecurity` / `NSAllowsArbitraryLoads = true` để hỗ trợ link HTTP thuần. |

---

> **Kết Luận:** 
> Tóm gọn lại, cốt lõi luồng tính năng nằm ở việc bạn cung cấp các cấu hình Data thích hợp (thông qua `MWG...Representable`) => Khởi tạo Dependency tương ứng => Giao quyền cho `MWGSDK` => Gọi hàm `MWGSDK.createPlayerVC(...)` hoặc `MWGSDK.createBroadcastVC(...)` để push ra View hiển thị rồi bắt sự kiện ngược lại ở Client. Chúc bạn tích hợp thành công!
