# Hướng dẫn Tích hợp MWG_Live SDK cho iOS
## 1. Giới thiệu
**MWG_Live** là SDK cung cấp giải pháp toàn diện để tích hợp tính năng Livestream và phát video trực tiếp vào ứng dụng iOS của bạn một cách nhanh chóng và tối ưu.
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
# Thêm đoạn cấu hình sau vào cuối file Podfile nếu gặp lỗi "No such module 'MWG_Live'"
post_install do |installer|
  installer.pods_project.targets.each do |target|
    if target.name == 'MWG_Live'
      target.build_configurations.each do |config|
        config.build_settings['DEFINES_MODULE'] = 'YES'
      end
    end
  end
end
```
Sau đó, tiến hành cài đặt bằng cách mở Terminal ở thư mục chứa file Podfile và chạy lệnh:
```bash
pod install
```
**Lưu ý:** Sau khi chạy lệnh thành công, hãy đảm bảo bạn luôn mở dự án bằng file **`.xcworkspace`** (không dùng `.xcodeproj`).
---
## 4. Cấu hình Info.plist (Bắt buộc)
SDK sử dụng native player bên dưới nên yêu cầu ứng dụng phải định danh rõ ràng. Nếu thiếu các key này, SDK sẽ không hoạt động đúng.
Mở file `Info.plist` của bạn dưới dạng Source Code (hoặc Edit trực tiếp Property List) và bổ sung các mã cấu hình sau:
```xml
<key>AppKey</key>
<string>YOUR_APP_KEY_HERE</string>
<key>PlayerId</key>
<string>YOUR_PLAYER_ID_HERE</string>
<key>SecretKey</key>
<string>YOUR_SECRET_KEY_HERE</string>
```
**Giải thích các key:**
- **`AppKey`**: Khóa định danh ứng dụng của bạn để giao tiếp với hệ thống MWG.
- **`PlayerId`**: Mã ID của player được cấp phát riêng cho nền tảng video của bạn.
- **`SecretKey`**: Khóa bí mật dùng để xác thực hệ thống.
---
## 5. Cấp quyền truy cập thiết bị
Để tính năng Livestream (phát sóng, ghi hình, thu âm) hoạt động và tránh việc ứng dụng bị crash khi mở trình phát sóng, ứng dụng cần phải được cấp quyền truy cập Camera và Microphone. 
Tiếp tục thêm vào `Info.plist` các khai báo sau:
```xml
<key>NSCameraUsageDescription</key>
<string>Ứng dụng cần quyền truy cập Camera để thực hiện quay và phát Livestream.</string>
<key>NSMicrophoneUsageDescription</key>
<string>Ứng dụng cần quyền truy cập Microphone để thu âm thanh trong quá trình Livestream.</string>
```
---
## 6. Lưu ý quan trọng
- **Không hỗ trợ Simulator:** Nhắc lại lần nữa, dependency cốt lõi của native player không chứa kiến trúc dành cho Simulator (x86_64 / arm64-simulator). Việc build trên Simulator sẽ trực tiếp báo lỗi (Undefined symbols hoặc Could not build module).
- **Luôn kiểm thử trên thiết bị thật:** Mọi thao tác tích hợp, khởi tạo player hay phát sóng livestream đều cần build thẳng vào iPhone hoặc iPad thật của bạn.
---
## 7. Troubleshooting (Xử lý lỗi thường gặp)
| Hiện tượng lỗi | Nguyên nhân & Cách khắc phục |
| :--- | :--- |
| **Báo lỗi thiếu file header sau khi chạy `pod install`** | Bạn có thể đã lỡ mở file `.xcodeproj`. Hãy đóng Xcode và mở lại project bằng **`.xcworkspace`**. |
| **Crash ứng dụng ngay khi mở màn hình Player** | Thường do thiếu một trong 3 key `AppKey`, `PlayerId`, `SecretKey` trong **`Info.plist`**. |
| **Crash ứng dụng ngay khi thao tác mở Camera** | Thiếu khai báo `NSCameraUsageDescription` hoặc `NSMicrophoneUsageDescription` trong file **`Info.plist`**. |
| **Lỗi `Build Failed` (Could not find module...) khi chạy Simulator** | SDK không hỗ trợ Simulator. Hãy cắm iPhone vào Mac và chọn đúng thiết bị đó làm Destination để build. |
---
## 8. Hướng dẫn Khởi tạo và Sử dụng SDK
Để bắt đầu tương tác với SDK, hãy thiết lập theo trình tự dưới đây:
### Bước 8.1: Khởi tạo SDK (Bắt buộc)
Tại file `AppDelegate.swift`, hãy import thư viện và gọi hàm khởi tạo ngay ở đầu chu trình sống của ứng dụng:
```swift
import UIKit
import MWG_Live
@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        // Khởi tạo MWG_Live SDK
        MWGModule.initialize()
        
        return true
    }
}
```
### Bước 8.2: Mở màn hình Livestream (Broadcast)
Khi người dùng bấm vào nút phát sóng, bạn tạo ra màn hình Broadcast từ SDK:
```swift
import MWG_Live
class DemoVC: UIViewController, MWGBroadcastDelegate {
    
    @objc private func startLive() {
        // 1. Chuẩn bị dependency cấu hình Live
        let dependency = MWGBroadcastDependency(
            rtmpURL: "rtmp://your-rtmp-server.url/path",
            streamKey: "your-stream-auth-key",
            delegate: self
        )
        
        // 2. Lấy ViewController từ MWGModule
        let broadcastVC = MWGModule.createBroadcast(dependency: dependency)
        
        // 3. Hiển thị
        navigationController?.pushViewController(broadcastVC, animated: true)
    }
    
    // MARK: - MWGBroadcastDelegate
    func broadcastDidStart() {
        print("Đã bắt đầu phát sóng")
    }
    
    func broadcastDidStop() {
        print("Đã dừng phát sóng")
    }
    
    func broadcastDidFail(error: Error) {
        print("Lỗi phát sóng:", error.localizedDescription)
    }
}
```
### Bước 8.3: Mở màn hình Trình phát Video / Xem Live (Player)
Khi người dùng bấm vào xem 1 luồng Live hoặc VOD:
```swift
import MWG_Live
// Chú ý: Ở hệ thống của bạn, bạn cần định nghĩa các Struct/Class thoả mãn các giao thức của SDK như MWGSDKConfigRepresentable, MWGPlayerConfigRepresentable,...
class DemoVC: UIViewController, MWGPlayerDelegate {
    
    @objc private func watchLive() {
        // 1. Cấu hình danh sách Player truyền vào hệ thống
        let playerDependency = MWGPlayerDependency(
            config: DemoSDKConfig(
                tmpDomain: "your-tmp-domain",
                secretKey: "your-secret-key"
            ),
            players: [
                DemoPlayerConfig(type: .live, videoId: "live_123", url: "https://your-domain/live.m3u8"),
                DemoPlayerConfig(type: .video, videoId: "vod_123", url: "https://your-domain/video.m3u8")
            ],
            delegate: self
        )
        // 2. Lấy ViewController trình phát
        let playerVC = MWGModule.createPlayerView(dependency: playerDependency)
        // 3. Hiển thị 
        navigationController?.pushViewController(playerVC, animated: true)
    }
    // MARK: - MWGPlayerDelegate
    func player(didChangeEvent event: MWGPlayerEvent) {
        print("Sự kiện Player:", event)
    }
}
```
---
## 9. Kết luận
Tổng kết lại, vòng đời tích hợp SDK **MWG_Live** đòi hỏi bạn thực hiện chuẩn xác 5 bước:
1. Include thư viện qua **CocoaPods** và build bằng `.xcworkspace`.
2. Truyền 3 tham số `AppKey`, `PlayerId`, và `SecretKey` vào **`Info.plist`**.
3. Mở khóa quyền **Camera & Microphone** cũng trong **`Info.plist`**.
4. Khởi chạy `MWGModule.initialize()` bên trong **`AppDelegate`**.
5. Chọn đúng **Thiết bị thật** (iPhone/iPad) khi build dự án.
Nếu bạn tuân thủ đúng 5 bước trên, ứng dụng sẽ chạy mượt mà không gặp rắc rối nào với SDK. Chúc bạn thành công!
