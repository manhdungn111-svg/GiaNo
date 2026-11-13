# Zalo Lite — nhắn tin, gọi thoại/video, sao lưu (web app cho iPhone)

Web app dạng **PWA**: mở bằng Safari, bấm *Thêm vào MH chính* là dùng như app thật.
Server chỉ dùng thư viện lõi của Node.js (không cần `npm install`).

## Có gì trong bản này

| Tính năng | Chi tiết |
|---|---|
| Tài khoản | Đăng ký/đăng nhập bằng tên đăng nhập (hoặc số điện thoại) + mật khẩu (băm scrypt) |
| Nhắn tin | 1-1 và nhóm (tối đa 200 người), realtime, "đang soạn tin", Đã gửi/Đã xem, link bấm được, tự kết nối lại, không mất/không trùng tin khi rớt mạng |
| Tệp & ảnh | Gửi ảnh (tự nén ≤1600px) và tệp ≤25 MB, xem ảnh toàn màn hình |
| Gọi | Thoại + video **1-1** (WebRTC), tắt mic/camera, đổi camera trước/sau, lưu lịch sử cuộc gọi vào chat |
| Sao lưu | Tự sao lưu `db.json` mỗi ngày (giữ 14 bản) + nút **Sao lưu ngay** (xuất JSON, lưu vào Tệp/iCloud) + **Khôi phục** |
| Đồng bộ | Đăng nhập thiết bị khác là có đủ lịch sử |
| Messenger | Nhập lịch sử Messenger cá nhân (chỉ xem + tìm kiếm không dấu) và cầu nối **Trang Messenger**: khách nhắn vào Page của bạn thì hiện trong app, trả lời từ app |
| Tìm kiếm | Tìm tin nhắn trong mọi cuộc trò chuyện, không cần gõ dấu, bấm vào kết quả để nhảy tới đúng tin |
| iPhone | Safe-area (tai thỏ), bàn phím không che ô nhập, chống tự phóng to, badge số tin chưa đọc |

## Chạy thử trên máy tính

```bash
node server.js          # mở http://localhost:3000
```

> Nhắn tin chạy được qua HTTP, nhưng **gọi thoại/video bắt buộc HTTPS** (trừ `localhost`) vì Safari chỉ cấp micro/camera cho trang HTTPS. Muốn thử trên iPhone, hãy deploy lên hosting có HTTPS (bên dưới).

## Deploy từ chính iPhone (không cần máy tính)

Bạn cần một nơi chạy Node.js/Docker **có HTTPS** và **ổ đĩa lưu trữ bền** (persistent volume) để dữ liệu không mất khi khởi động lại. Các dịch vụ như Railway, Render, Fly.io hay VPS riêng đều làm được — hãy xem trang giá hiện tại của từng nơi, vì gói miễn phí thường **không có ổ đĩa bền**.

1. **Giải nén**: trong app *Tệp*, chạm vào file `zalo-lite.zip` để giải nén.
2. **Đưa code lên GitHub**: mở Safari → github.com → *New repository* → *uploading an existing file* → chọn toàn bộ file/thư mục vừa giải nén (`server.js`, `package.json`, `Dockerfile`, thư mục `public`…).
   Lưu ý: Safari trên iPhone thường chỉ chọn được từng tệp chứ không chọn được cả thư mục. Nếu vậy, tải các tệp ở thư mục gốc lên trước, rồi dùng *Add file → Create new file* và gõ tên kiểu `public/app.js` (dấu `/` tự tạo thư mục) để dán nội dung từng tệp trong `public/`. Cách nhanh hơn là dùng máy tính hoặc một app Git trên iPhone để đẩy cả thư mục một lần.
3. **Tạo dịch vụ** trên nơi bạn chọn → *Deploy from GitHub repo* → chọn repo (nó sẽ tự dùng `Dockerfile`).
4. **Thêm ổ đĩa (volume)**, gắn vào đường dẫn `/data`.
5. **Biến môi trường**: `DATA_DIR=/data` (đã đặt sẵn trong Dockerfile), `PORT` nếu nơi đó yêu cầu.
6. Mở link HTTPS được cấp bằng Safari → đăng ký tài khoản.
7. **Thêm vào Màn hình chính**: nút Chia sẻ (□↑) → *Thêm vào MH chính*. Mở app từ biểu tượng, lần đầu gọi video bấm *Cho phép* micro/camera.

## Bật gọi ổn định trên 4G/Wi-Fi khác nhau: TURN server

Mặc định dùng STUN công cộng của Google. Khi hai người ở sau NAT "khó" (nhiều mạng 4G/5G, Wi-Fi công ty), cuộc gọi **cần TURN** để chuyển tiếp. Cài `coturn` trên một VPS (cần IP công cộng):

```
# /etc/turnserver.conf
listen-port=3478
fingerprint
use-auth-secret
static-auth-secret=ĐỔI_CHUỖI_BÍ_MẬT_DÀI
realm=turn.ten-mien-cua-ban.com
no-multicast-peers
total-quota=100
min-port=49152
max-port=65535
```

Mở cổng 3478 (UDP+TCP) và 49152–65535 (UDP). Rồi đặt biến môi trường cho app:

```
TURN_URLS=turn:turn.ten-mien-cua-ban.com:3478?transport=udp,turn:turn.ten-mien-cua-ban.com:3478?transport=tcp
TURN_SECRET=ĐỔI_CHUỖI_BÍ_MẬT_DÀI        # cùng giá trị static-auth-secret
```

Server tự cấp tài khoản TURN tạm thời (6 giờ) cho từng người dùng đang đăng nhập. Có thể thay bằng dịch vụ TURN thuê ngoài: đặt `TURN_URLS`, `TURN_USER`, `TURN_PASS`.

## Sao lưu dữ liệu

- **Phía server**: `DATA_DIR/backups/db-YYYY-MM-DD.json` tạo tự động mỗi ngày (giữ 14 bản). Hãy sao lưu cả thư mục `DATA_DIR` (gồm `db.json` và `files/`) ra nơi khác định kỳ (snapshot của volume, hoặc `rclone`/`rsync`).
- **Phía người dùng**: *Cá nhân → Sao lưu ngay* tải về file JSON lịch sử chat của họ.
- **Khôi phục**: *Cá nhân → Khôi phục từ tệp*. Chỉ khôi phục **tin nhắn do chính người dùng gửi**, trong các cuộc trò chuyện họ đang tham gia (để không ai giả mạo tin của người khác). Bản sao lưu chứa thông tin tệp đính kèm chứ **không chứa nội dung ảnh/tệp**.
- **Khôi phục toàn bộ server**: dừng app, chép `db.json` (hoặc một bản trong `backups/`) và thư mục `files/` về `DATA_DIR`, chạy lại.

## Đồng bộ với Messenger

**Điều cần biết trước:** Meta không cho web bên thứ ba đăng nhập tài khoản Messenger *cá nhân* để đồng bộ tin nhắn hay cuộc gọi. API chính thức của Messenger chỉ dành cho **Trang (Page)**. Các thư viện không chính thức dùng mật khẩu Facebook vi phạm điều khoản và hay bị khóa tài khoản nên bản này không dùng cách đó. Thay vào đó có hai cách chính thức:

### Cách 1 — Nhập lịch sử Messenger (tài khoản cá nhân)

Trong app: *Cá nhân → Nhập lịch sử từ Messenger* (có hướng dẫn từng bước). Tóm tắt:

1. Facebook: Cài đặt → Trung tâm tài khoản → Thông tin và quyền của bạn → Tải thông tin của bạn → chọn **Tin nhắn**, định dạng **JSON** (tên menu có thể khác tùy phiên bản).
2. Tải tệp `.zip` khi Facebook báo sẵn sàng, chạm vào để giải nén trong app *Tệp*.
3. Vào `messages/inbox/<tên cuộc trò chuyện>/` và chọn tất cả `message_1.json`, `message_2.json`… của cuộc trò chuyện đó. Lặp lại cho cuộc khác.

Kết quả: cuộc trò chuyện hiện trong danh sách với nhãn "Đã nhập", **chỉ xem lại và tìm kiếm**, không gửi tin mới. Nhập lại cùng tệp không bị trùng. Chữ tiếng Việt được sửa lỗi mã hóa của Facebook.
Giới hạn: ảnh/video/tệp chỉ hiện dạng `[Hình ảnh]`, `[Video]`… (nội dung không được nhập); không tự cập nhật (muốn thêm tin mới phải tải dữ liệu mới); dữ liệu rất lớn (hàng trăm nghìn tin) làm `db.json` nặng và chậm. Chỉ nhập dữ liệu của chính tài khoản bạn.

### Cách 2 — Cầu nối Trang Messenger (Facebook Page)

Khách nhắn vào Page của bạn thì hiện trong app (nhãn "Messenger"), bạn trả lời ngay trong app. Tin nhân viên gửi từ Page Inbox của Facebook cũng hiện ở đây.

1. Bạn phải là quản trị của một Facebook Page.
2. Vào developers.facebook.com → tạo **App** (loại Business) → thêm sản phẩm **Messenger**. Tạo **Page access token** cho Page, và lấy **App Secret** (Cài đặt → Cơ bản). Meta hay đổi tên menu, nếu khác hãy theo tài liệu "Messenger Platform" mới nhất.
3. Deploy app này có HTTPS, đặt các biến môi trường `MESSENGER_VERIFY_TOKEN`, `MESSENGER_APP_SECRET`, `MESSENGER_PAGE_TOKEN`, `MESSENGER_ADMINS` (xem bảng cấu hình bên dưới).
4. Đăng ký trong app một tài khoản có **tên đăng nhập trùng** với `MESSENGER_ADMINS`.
5. Ở Meta: Webhooks → chọn Page → *Callback URL* = `https://tên-miền-của-bạn/webhook/messenger` (cũng hiện sẵn trong *Cá nhân*), *Verify token* = `MESSENGER_VERIFY_TOKEN`. Đăng ký (subscribe) các trường `messages` và `message_echoes`, và gắn Page vào app.
6. Khi app còn ở chế độ phát triển, chỉ những người có vai trò trong app (admin/developer/tester) nhắn vào Page mới được chuyển tới đây. Để khách thật dùng được, Meta yêu cầu duyệt quyền `pages_messaging` (App Review).

Hành vi và quy tắc của Meta:
- **Chỉ trả lời được trong 24 giờ** kể từ tin nhắn cuối của khách. Quá hạn, app sẽ báo lỗi (trừ khi bạn có quyền Human Agent và đặt `MESSENGER_HUMAN_AGENT=1`, khi đó tới 7 ngày).
- Từ app chỉ gửi được **văn bản** cho khách. Ảnh khách gửi thì xem được.
- **Không gọi điện được** qua Messenger. Cuộc gọi trong app chỉ giữa các tài khoản của web này.
- Webhook kiểm tra chữ ký `X-Hub-Signature-256` bằng App Secret; không cấu hình thì từ chối. Chỉ tài khoản trong `MESSENGER_ADMINS` thấy tin nhắn của khách.
- Cầu nối đã được kiểm thử tự động bằng **máy chủ Meta giả lập**, chưa thử với Meta thật. Nếu Meta thay đổi API hay có bước duyệt khác, có thể cần chỉnh thêm.

## Cấu hình (biến môi trường)

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `PORT` | 3000 | Cổng HTTP |
| `DATA_DIR` | `./data` | Nơi lưu dữ liệu |
| `STUN_URLS` | stun của Google | Danh sách STUN, ngăn cách bằng dấu phẩy |
| `TURN_URLS`, `TURN_SECRET` | — | TURN dùng `use-auth-secret` |
| `TURN_URLS`, `TURN_USER`, `TURN_PASS` | — | TURN dùng tài khoản cố định |
| `MESSENGER_VERIFY_TOKEN` | — | Chuỗi bạn tự đặt, dán vào ô Verify token khi cài webhook ở Meta |
| `MESSENGER_APP_SECRET` | — | App Secret của Meta App (dùng kiểm tra chữ ký webhook) |
| `MESSENGER_PAGE_TOKEN` | — | Page access token (dùng gửi trả lời khách) |
| `MESSENGER_ADMINS` | — | Tên đăng nhập quản trị, cách nhau bằng dấu phẩy. **Chỉ những người này thấy tin nhắn của khách.** Bắt buộc để bật cầu nối |
| `MESSENGER_PAGE_NAME` | Trang | Tên hiển thị của Trang trong app |
| `MESSENGER_HUMAN_AGENT` | 0 | Đặt 1 nếu Meta đã cấp quyền Human Agent (cho trả lời tới 7 ngày) |
| `MESSENGER_GRAPH_VERSION` | v23.0 | Phiên bản Graph API |

## Giới hạn hiện tại (cần biết)

- **Chưa có thông báo đẩy** khi đóng app. iOS (16.4+) hỗ trợ Web Push cho app đã thêm vào Màn hình chính, nhưng chưa làm trong bản này (cần khóa VAPID + thư viện `web-push`). Khi mở app, tin mới vẫn tới ngay.
- **Gọi nhóm chưa có**; mới gọi 1-1. Gọi nhóm cần máy chủ SFU như LiveKit.
- **Chưa mã hóa đầu cuối**: máy chủ lưu tin nhắn dạng văn bản thường. Hãy dùng HTTPS và bảo vệ ổ đĩa.
- **Chưa xác thực OTP** qua SMS; đăng nhập bằng mật khẩu (có giới hạn đăng nhập sai).
- **Một server duy nhất**, lưu bằng file JSON: ổn cho nhóm bạn bè/đội nhỏ (hàng trăm người). Muốn lớn hơn nên chuyển sang PostgreSQL + Redis (pub/sub) để chạy nhiều instance.
- Mật khẩu quên **chưa có cách đặt lại** (cần thêm email/OTP).
- Đã kiểm thử tự động bằng Chrome mô phỏng iPhone (nhắn tin, nhóm, ảnh, gọi video 2 chiều, sao lưu/khôi phục, đồng bộ, tự nối lại), **chưa thử trên iPhone/Safari thật**. Nếu gặp lỗi riêng của Safari (ví dụ không có tiếng khi gọi), hãy báo lại để chỉnh.

## Cấu trúc

```
server.js            máy chủ: API, WebSocket, signaling WebRTC, sao lưu
public/index.html    vỏ ứng dụng
public/app.js        toàn bộ giao diện + logic client + WebRTC
public/style.css     giao diện (kiểu Zalo, hỗ trợ safe-area iPhone)
public/sw.js         service worker (mở nhanh, chạy được khi offline phần giao diện)
public/manifest.webmanifest, public/icons/   cài lên Màn hình chính
Dockerfile           đóng gói để deploy
```
