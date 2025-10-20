# Infrastructure-Checklist

## Bảo mật SSH

SSH là cổng vào chính của server. Nếu mình để nó mặc định thì không khác gì mở cửa mời trộm. Vài việc cần làm ngay:

Tắt quyền đăng nhập của user root: Trong file /etc/ssh/sshd_config, luôn đảm bảo dòng này được bật.

```bash
PermitRootLogin no
```

Dùng SSH key thay cho mật khẩu: Mật khẩu có thể bị dò ra, còn SSH key thì an toàn hơn nhiều. Lệnh này giúp mọi người copy public key qua server một cách đơn giản.

```bash
ssh-copy-id devops_admin@192.168.1.10
```

Đổi port mặc định: Port 22 là port quá nổi tiếng. Đổi nó sang một port khác ít người biết hơn, ví dụ 2222, để giảm thiểu các cuộc tấn công tự động.

Chỉ cho phép IP tin cậy truy cập: Dùng firewall để giới hạn IP được phép kết nối vào port SSH.

```bash
# Ví dụ với ufw
ufw allow from 118.70.15.99 to any port 2222
```

Giới hạn user được phép login: Không phải user nào trên server cũng cần quyền login qua SSH. Mình chỉ định rõ trong file /etc/ssh/sshd_config.

```bash
AllowUsers devops_admin service_account
```

## Phân quyền Sudo

Khi đã vào được bên trong, kẻ tấn công sẽ tìm cách leo thang đặc quyền lên root. Sudo là một công cụ rất mạnh, nên mình phải quản lý nó thật chặt.

Ở team mình, quy tắc là:

Chỉ các tài khoản admin mới có quyền sudo đầy đủ.
Mọi câu lệnh sudo đều phải được ghi log lại.
Với các tài khoản dịch vụ hoặc user thường, mình chỉ cấp quyền sudo cho những lệnh thực sự cần thiết.

```bash
# Ví dụ: cho phép user 'app_runner' chỉ được quyền đọc log của một service
app_runner ALL=(ALL) NOPASSWD: /usr/bin/tail -f /var/log/my_app/activity.log
```

## Quản lý User

Đây là lỗi khá phổ biến. Có lần team mình phát hiện tài khoản của một bạn nhân viên cũ vẫn còn active mấy tháng sau khi bạn nghỉ việc. Rủi ro rất lớn.

Thói quen của mình:

Xóa tài khoản không còn dùng tới và cả thư mục home của họ.

```bash
userdel -r user_cu
```

Thường xuyên kiểm tra danh sách user để xem có tài khoản nào lạ không.

```bash
cat /etc/passwd | cut -d: -f1
```

Sử dụng group để quản lý quyền thay vì gán trực tiếp cho từng user.

## Firewall (UFW / iptables)

Server production không nên mở toang các port ra ngoài internet. Nguyên tắc của mình là chặn tất cả, chỉ cho phép những gì cần thiết.

Với UFW cho các thiết lập nhanh:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 'Nginx Full' # Cho phép port 80 và 443
ufw allow from 10.0.1.0/24 to any port 5432 proto tcp # Cho phép subnet nội bộ truy cập DB
ufw enable
```

Với các rule phức tạp hơn, mình sẽ dùng iptables.

Dùng Faillock để chống Brute Force
Faillock là cơ chế để chống lại các cuộc tấn công dò mật khẩu. Nó sẽ tự động khóa tài khoản sau một số lần đăng nhập thất bại.

Cấu hình trong file /etc/security/faillock.conf:

```bash
deny = 3
unlock_time = 900
```

Với cấu hình này, tài khoản sẽ bị khóa trong 15 phút (900 giây) nếu nhập sai mật khẩu 3 lần.

## Chỉnh luật đăng nhập với PAM

PAM (Pluggable Authentication Modules) cho phép mình tùy chỉnh sâu hơn các quy tắc xác thực, đặc biệt là độ phức tạp của mật khẩu.

Ví dụ trong file /etc/pam.d/common-password để yêu cầu mật khẩu mạnh:

```bash
password requisite pam_pwquality.so retry=3 minlen=14 ucredit=-1 lcredit=-1 dcredit=-2 ocredit=-1
```

Rule này yêu cầu mật khẩu dài tối thiểu 14 ký tự, phải có ít nhất 1 chữ hoa, 1 chữ thường, 2 chữ số và 1 ký tự đặc biệt.

## Chính sách password

Một password yếu có thể phá vỡ mọi lớp phòng thủ. Ở team mình luôn yêu cầu:

Mật khẩu tối thiểu 14 ký tự.
Bao gồm chữ hoa, chữ thường, số, và ký tự đặc biệt.
Thay đổi mỗi 60 ngày với các hệ thống quan trọng.
Mình cấu hình điều này trong /etc/login.defs:

```bash
# Yêu cầu đổi mật khẩu sau 60 ngày
PASS_MAX_DAYS   60
# Không cho đổi mật khẩu trong 1 ngày sau lần đổi cuối
PASS_MIN_DAYS   1
# Cảnh báo trước 7 ngày
PASS_WARN_AGE   7
```

## Hạn chế truy cập Internet từ Server

Nhiều server production thực tế không cần kết nối internet toàn quyền. Việc giới hạn egress traffic giúp giảm bề mặt tấn công.

Ví dụ, mình chỉ cho phép server kết nối đến repo nội bộ để cập nhật package:

```bash
# Chặn mọi traffic đi ra mặc định
ufw default deny outgoing

# Cho phép kết nối DNS
ufw allow out 53

# Cho phép kết nối tới repo nội bộ
ufw allow out to 192.168.1.100 port 8081 proto tcp
```

## Bật Logging và Auditing

Để biết ai đã làm gì trên server, đặc biệt là các lệnh sudo, mình luôn bật ghi log chi tiết.

Trong file /etc/sudoers:

```bash
Defaults logfile="/var/log/sudo.log"
```

Sau đó mình sẽ dùng các công cụ như filebeat để đẩy log này về một hệ thống quản lý log tập trung như ELK Stack hoặc Graylog để tiện theo dõi và cảnh báo.

```bash
# Theo dõi log sudo trực tiếp trên server
tail -f /var/log/sudo.log
```

## Kích hoạt xác thực hai yếu tố (2FA)

Để tăng cường bảo mật cho các tài khoản quan trọng, mình thường tích hợp thêm 2FA. Sử dụng libpam-google-authenticator là một cách khá phổ biến để thêm một lớp OTP khi đăng nhập SSH.

Bảo mật không phải là một project làm một lần rồi thôi, mà là một thói quen, một văn hoá cần được duy trì hàng ngày. Việc gia cố kỹ càng ngay từ đầu có thể tốn thêm chút thời gian, nhưng nó sẽ giúp mình và cả team tránh được những sự cố không đáng có, và quan trọng là giúp chúng ta ngủ ngon hơn vào ban đêm
