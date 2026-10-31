# oracle-snatch

Một script Python nhỏ chạy vòng lặp: liên tục dùng API chính thức của Oracle Cloud để xin cấp instance ARM miễn phí (VM.Standard.A1.Flex), cho đến khi gặp đúng lúc Oracle "nhả hàng" và API đồng ý.

## Vì sao có repo này

Hạn mức miễn phí A1.Flex của Always Free (tổng cộng 4 nhân / 24 GB) hiện là máy chủ miễn phí đáng giá nhất, không có đối thủ. Vấn đề là ở các region hot, console gần như lúc nào cũng báo "Out of host capacity" (hết hàng). Hàng không phải mất vĩnh viễn, mà thỉnh thoảng được nhả ra một đợt nhỏ, và không ai thông báo cho bạn. Ngồi F5 console cả tuần thì chỉ phí đời, thà để script hỏi thay bạn.

Tháng 10/2026, tôi chạy một bản thô ở region Tokyo trong vài ngày. Log toàn là "hết hàng", giữa chừng còn vài lần dính giới hạn 429 vì tôi đặt khoảng nghỉ quá dày. Cuối cùng tôi cũng giành được con máy 4 nhân 24 GB đó. Hiện giờ phần lớn dịch vụ self-host của tôi chạy trên nó, đến nay hóa đơn vẫn là 0. Repo này là bản đã dọn gọn của vòng lặp ấy, mọi cấu hình đều được tách ra riêng.

## Nó làm gì

- Chỉ dùng API chính thức của Oracle (SDK `oci` của Python), không tự động hóa console, không giả lập trình duyệt.
- Quét lần lượt mọi availability domain (AD) trong region của bạn, mỗi vòng thử mỗi AD một lần. Tokyo chỉ có một AD; region nhiều AD thì tỉ lệ trúng cao hơn một chút.
- Gặp "hết hàng" thì coi như "chưa đến lúc": nghỉ một chút, sang AD tiếp theo, rồi vòng tiếp theo.
- Gặp giới hạn 429 thì lùi (backoff) thật lâu (mặc định 300 giây). Cố lao qua 429 chỉ khiến giới hạn bám dai hơn.
- Gặp lỗi hạn mức (LimitExceeded) thì dừng ngay và ghi rõ lý do: nghĩa là hạn mức miễn phí của bạn đã dùng hết, thử lại một vạn lần cũng vô ích.
- Khi giành được, script chờ instance chuyển sang RUNNING, lấy IP công khai, ghi vào `snatched.json`, có thể gửi thông báo Telegram (tùy chọn), rồi thoát.
- `--dry-run` chỉ kiểm tra cấu hình: liệt kê các AD, xác nhận sẽ dùng image nào, không tạo máy thật. Lần đầu hãy chạy cái này trước.

Những việc nó không làm: không đăng ký tài khoản giúp bạn, không tạo VCN, không sinh API key, và càng không thể biến ra thứ hàng không tồn tại. Nó chỉ hỏi thay bạn không biết mệt.

## Cài đặt và sử dụng

Cần có tài khoản Oracle Cloud và file cấu hình OCI (`~/.oci/config`, trong đó đã cấu hình API key). Ai từng dùng OCI CLI hoặc SDK đều có file này.

Cần thêm OCID của subnet: trong console vào Networking -> Virtual Cloud Networks -> VCN của bạn -> Subnets. Muốn có IP công khai thì chọn subnet công khai (chắc chắn bạn muốn vậy).

```bash
git clone https://github.com/lucasforster1108-boop/oracle-snatch.git
cd oracle-snatch
pip install -r requirements.txt

cp config.example.env config.env
# Sửa config.env: chỉ bắt buộc điền SUBNET_ID và SSH_PUBLIC_KEY_FILE

python3 snatch.py --dry-run
python3 snatch.py
```

## Chạy treo lâu dài

Thứ này được thiết kế để chạy không người trông nom nhiều ngày, nên cứ treo trên bất kỳ máy nào bạn đang có và bật thường trực (VPS miễn phí ở nơi khác, server ở nhà, Raspberry Pi đều được). Repo có kèm một ví dụ dịch vụ systemd:

```bash
sudo cp oracle-snatch.service /etc/systemd/system/
sudo systemctl enable --now oracle-snatch
tail -f snatch.log
```

## Những cái bẫy đã gặp

- Hạn mức A1 miễn phí là tổng chung, không phải mỗi máy một phần: tổng cộng 4 nhân + 24 GB. Có thể tạo một máy 4/24, hoặc chia thành hai máy 2/12. Vượt quá sẽ báo LimitExceeded, script sẽ chủ động dừng, đừng trách nó.
- Nâng tài khoản lên PAYG (trả theo mức dùng) không làm mất hạn mức miễn phí; lượng A1 trong hạn mức vẫn không bị tính tiền, và cảm giác là hàng còn nhiều hơn một chút. Tài khoản của tôi là PAYG, hóa đơn đến giờ vẫn 0 đồng, ngoài ra tôi đặt thêm cảnh báo ngân sách để phòng hờ.
- Ổ khởi động cũng tính vào tổng 200 GB block storage miễn phí. 50 GB là đủ cho server nhỏ, đừng vì "dù sao cũng miễn phí" mà mặc định kéo lên 200; phần còn lại để dành cho máy thứ hai thì có lợi hơn.
- Thời điểm nhả hàng không có quy luật, thường vào nửa đêm. Ý nghĩa của vòng lặp chính là ở đây: 4 giờ sáng nó thức, bạn thì không cần.
- Đừng chỉnh khoảng nghỉ giữa các lần thử xuống còn vài giây. Bạn sẽ không giành được nhanh hơn, chỉ ăn 429, rồi một lúc sau đó chẳng giành được gì.

## Các file

- `snatch.py`: toàn bộ logic, gói trong một file
- `config.example.env`: tất cả tùy chọn cấu hình, có chú thích
- `oracle-snatch.service`: ví dụ dịch vụ systemd

## Giấy phép

MIT
