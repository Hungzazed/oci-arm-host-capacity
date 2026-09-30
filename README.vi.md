# OCI ARM Host Capacity Hunter

Tự động gọi lại API `LaunchInstance` của Oracle Cloud Infrastructure cho tới khi săn được máy ARM free-tier ở home region.

Fork từ [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity), đã tinh gọn setup, thêm thông báo Telegram, hỗ trợ boot volume và workflow GitHub Actions dùng sẵn.

<p align="center">
  <a href="https://github.com/Hungzazed/oci-arm-host-capacity/actions/workflows/hunt.yml"><img src="https://github.com/Hungzazed/oci-arm-host-capacity/actions/workflows/hunt.yml/badge.svg" alt="Hunt ARM"></a>
</p>

> 📖 English version: [README.md](README.md)

> Mỗi tenancy được miễn phí 3.000 OCPU giờ + 18.000 GB giờ / tháng cho `VM.Standard.A1.Flex` (tối đa 4 OCPUs / 24 GB RAM). Oracle thỉnh thoảng mới nhả thêm capacity — script này sẽ poll `LaunchInstance` liên tục cho tới khi tạo được máy.

**Mẹo (2024+):** Nhiều người khuyên nâng lên Pay-As-You-Go (PAYG) để được ưu tiên tạo máy free-tier. PAYG vẫn giữ quyền lợi Always Free, ít gặp lỗi `Out of host capacity` hơn và mở thêm nhiều dịch vụ. Nhớ bật budget alert và kiểm soát tài nguyên đã tạo.

---

- [Tính năng](#tính-năng)
- [Yêu cầu](#yêu-cầu)
- [Bắt đầu nhanh](#bắt-đầu-nhanh)
- [Cấu hình](#cấu-hình)
- [Lấy OCI_SUBNET_ID và OCI_IMAGE_ID](#lấy-oci_subnet_id-và-oci_image_id)
- [Chạy script](#chạy-script)
  - [Khuyên dùng: cron-job.org](#khuyên-dùng-cron-joborg)
  - [Local / cron](#local--cron)
  - [GitHub Actions (dự phòng, bị delay)](#github-actions-dự-phòng-bị-delay)
  - [Nhiều cấu hình](#nhiều-cấu-hình)
- [Thông báo Telegram](#thông-báo-telegram)
- [Cách hoạt động](#cách-hoạt-động)
- [Gán public IP](#gán-public-ip)
- [Xử lý lỗi](#xử-lý-lỗi)
- [Credits](#credits)

## Tính năng

- Thử lại qua từng Availability Domain khi gặp `Out of host capacity` (HTTP 500 + `InternalError`).
- Bỏ qua nếu đã đủ `OCI_MAX_INSTANCES` cùng shape (kiểm tra qua `ListInstances`).
- Cache `ListAvailabilityDomains` vào `oci_cache.json` khi `CACHE_AVAILABILITY_DOMAINS=1`.
- Nghỉ khi bị HTTP 429 / `TooManyRequests` qua `TOO_MANY_REQUESTS_TIME_WAIT` (lưu trạng thái ở `too_many_requests_waiter.txt`).
- Hỗ trợ dung lượng boot volume tùy chỉnh (`OCI_BOOT_VOLUME_SIZE_IN_GBS`) hoặc dùng lại boot volume cũ (`OCI_BOOT_VOLUME_ID`).
- Nhắn Telegram khi tạo máy thành công (nếu có cấu hình).
- Chạy local, qua cron, qua [cron-job.org](https://cron-job.org) (khuyên dùng, không cần treo VPS), hoặc qua GitHub Actions (`.github/workflows/hunt.yml`, mỗi 5 phút — có delay, xem bên dưới).

## Yêu cầu

- PHP >= 7.0 < 9.0 kèm `ext-curl`, `ext-json`
- `composer`
- Tài khoản OCI Always Free (hoặc PAYG) đã tạo API key

## Bắt đầu nhanh

```bash
git clone https://github.com/Hungzazed/oci-arm-host-capacity.git
cd oci-arm-host-capacity
composer install
cp .env.example .env
# sửa file .env, rồi chạy:
php ./index.php
```

Khi chưa có capacity sẽ báo lỗi như sau (bình thường):

```json
{
    "code": "InternalError",
    "message": "Out of host capacity."
}
```

Khi thành công sẽ in ra JSON của máy mới và (nếu có) gửi tin nhắn Telegram.

## Cấu hình

Copy `.env.example` thành `.env`. **Tuyệt đối không commit `.env` — file này chứa secret.**

### 1. Thông tin API

Tạo API key trong OCI Console: click avatar -> User Settings -> Resources -> API keys -> Add API Key -> chọn Generate Key Pair -> Download Private Key -> Add. Copy các giá trị hiện ra vào `.env`:

| Biến | Mô tả |
|---|---|
| `OCI_REGION` | ví dụ `eu-frankfurt-1` |
| `OCI_USER_ID` | `ocid1.user.oc1...` |
| `OCI_TENANCY_ID` | `ocid1.tenancy.oc1...` |
| `OCI_KEY_FINGERPRINT` | ví dụ `b3:a5:90:...` |
| `OCI_PRIVATE_KEY_FILENAME` | Đường dẫn tuyệt đối hoặc URL public tới file `.pem`, ví dụ `"/path/to/oci.pem"` |

![User Settings](images/user-settings.png)

### 2. Tham số máy ảo

| Biến | Bắt buộc | Mặc định | Mô tả |
|---|---|---|---|
| `OCI_SUBNET_ID` | Có | — | Xem [bên dưới](#lấy-oci_subnet_id-và-oci_image_id) |
| `OCI_IMAGE_ID` | Có* | — | Xem bên dưới. *Không cần nếu đã set `OCI_BOOT_VOLUME_ID` |
| `OCI_SSH_PUBLIC_KEY` | Có | — | Nội dung `~/.ssh/id_rsa.pub`, để trong ngoặc kép, 1 dòng duy nhất, không xuống dòng |
| `OCI_SHAPE` | Có | — | `VM.Standard.A1.Flex` (ARM) hoặc `VM.Standard.E2.1.Micro` (AMD) |
| `OCI_OCPUS` | Có | `4` | ARM: `1/2/3/4` (AMD: `1`) |
| `OCI_MEMORY_IN_GBS` | Có | `24` | ARM: `6/12/18/24` (AMD: `1`). Image Oracle Linux Cloud Developer cần >= 8 |
| `OCI_MAX_INSTANCES` | Không | `1` | Số máy cùng shape tối đa thì bỏ qua, không tạo thêm |
| `OCI_AVAILABILITY_DOMAIN` | ARM không bắt buộc | rỗng | Để trống để tự dò tất cả AD. **Bắt buộc** với AMD `E2.1.Micro` (phải là AD Always Free Eligible) và khi dùng `OCI_BOOT_VOLUME_ID` |
| `OCI_BOOT_VOLUME_SIZE_IN_GBS` | Không | rỗng | Dung lượng boot volume tùy chỉnh, 50–200 GB cho Always Free (tối thiểu 47 AMD / 50 ARM) |
| `OCI_BOOT_VOLUME_ID` | Không | rỗng | Dùng lại boot volume cũ theo OCID. Không dùng chung với size ở trên. Phải set `OCI_AVAILABILITY_DOMAIN` trùng AD của volume |
| `CACHE_AVAILABILITY_DOMAINS` | Không | `1` | `1` = cache danh sách AD vào `oci_cache.json` để giảm gọi API |
| `TOO_MANY_REQUESTS_TIME_WAIT` | Không | `600` | Số giây nghỉ khi bị HTTP 429. `0` / rỗng = tắt |
| `TELEGRAM_BOT_API_KEY` | Không | rỗng | Xem [Telegram](#thông-báo-telegram) |
| `TELEGRAM_USER_ID` | Không | rỗng | Xem [Telegram](#thông-báo-telegram) |

Ví dụ AMD Always Free:

```bash
OCI_SHAPE=VM.Standard.E2.1.Micro
OCI_OCPUS=1
OCI_MEMORY_IN_GBS=1
OCI_AVAILABILITY_DOMAIN=FeVO:EU-FRANKFURT-1-AD-2
```

Lấy SSH public key:

```bash
cat ~/.ssh/id_rsa.pub
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... user@example.com
```

## Lấy OCI_SUBNET_ID và OCI_IMAGE_ID

1. Vào OCI Console -> Menu -> Compute -> Instances -> Create Instance.
2. Chọn image + shape (`VM.Standard.A1.Flex`, 4/24). Với AMD phải có nhãn `Always Free Eligible`.
3. Phần Networking chọn VCN/subnet có sẵn (nếu chưa có thì tạo trước 1 máy `VM.Standard.E2.1.Micro`). Tạm bỏ tick public IP.
4. Mở DevTools trình duyệt -> tab Network, bấm Create, đợi báo lỗi `Out of capacity`.
5. Tìm request `/instances` màu đỏ, chuột phải -> Copy as cURL, dán vào editor, đọc `subnetId`, `imageId`, `availabilityDomain` trong `--data-binary`.
6. Điền `OCI_SUBNET_ID`, `OCI_IMAGE_ID` tương ứng.

![Dev Tools](images/dev-tools.png)

## Chạy script

### Khuyên dùng: cron-job.org

Dùng [cron-job.org](https://cron-job.org) (miễn phí) để ping script đã host mỗi phút — đáng tin hơn GitHub Actions khi capacity chỉ mở trong vài phút ngắn.

1. Host project này ở 1 URL public (VPS rẻ / shared hosting có PHP đều được, ví dụ `https://your-domain.com/oci-arm-host-capacity/index.php`).
   - Set `OCI_PRIVATE_KEY_FILENAME` trong `.env` thành URL public của file `.pem` (hoặc pre-authenticated URL của OCI Object Storage), vì host web phải tải được key.
   - Mở URL đó trên trình duyệt phải ra JSON giống `php ./index.php` (`Out of host capacity` cho tới khi thành công).
2. Đăng ký [cron-job.org](https://cron-job.org) -> Create cronjob:
   - URL: `https://your-domain.com/oci-arm-host-capacity/index.php`
   - Interval: mỗi 1 phút (hoặc 2–5 phút để tránh bị OCI rate-limit).
   - Request timeout: 30s, bật logging / thông báo khi lỗi.
3. Xong. Khi tạo máy thành công sẽ nhận tin nhắn Telegram (nếu có cấu hình). Tạo được máy rồi thì tắt/xóa cron-job đi.

Vì sao không dùng GitHub Actions? Xem [bên dưới](#github-actions-dự-phòng-bị-delay) — workflow lên lịch thường bị queue delay 10–30+ phút, rất dễ hụt mất capacity chỉ mở vài phút.

### Local / cron

```bash
php ./index.php
# dùng file env riêng:
php index.php .env.my_acc1
```

Cron (mỗi phút):

```bash
touch /path/to/oci-arm-host-capacity/oci.log
chmod 644 /path/to/oci-arm-host-capacity/oci.log
which php  # thường là /usr/bin/php
EDITOR=nano crontab -e
```

```
* * * * * /usr/bin/php /path/to/oci-arm-host-capacity/index.php >> /path/to/oci-arm-host-capacity/oci.log 2>&1
```

Dùng đường dẫn tuyệt đối. Nếu lỗi quyền thì chạy `sudo crontab -e` hoặc sửa ownership thay vì `chmod 777`.

### GitHub Actions (dự phòng, bị delay)

> ⚠️ **Sẽ bị delay:** GitHub chạy workflow `cron` theo kiểu best-effort. Dù `hunt.yml` để `*/5 * * * *`, job vẫn phải xếp hàng, giờ cao điểm có thể trễ 10–30+ phút, mất thêm 1–2 phút để dựng `ubuntu-latest` + PHP + `composer install`, và có thể bị bỏ qua hẳn nếu repo không hoạt động (GitHub tắt schedule sau 60 ngày không có commit). Capacity free của OCI thường hết trong vài phút nên rất dễ hụt. Hãy ưu tiên [cron-job.org](#khuyên-dùng-cron-joborg) hoặc cron local để săn thật; chỉ dùng Actions để test.

`hunt.yml` chạy mỗi 5 phút + bấm tay `workflow_dispatch`. Không cần file `.env` — mọi giá trị lấy từ Secrets / Variables của repo.

1. Fork repo này.
2. Vào Settings -> Secrets and variables -> Actions -> New repository secret, thêm từng cái (tên giống trong `.env`, **không thêm ngoặc kép**):
   `OCI_REGION`, `OCI_USER_ID`, `OCI_TENANCY_ID`, `OCI_KEY_FINGERPRINT`, `OCI_SUBNET_ID`, `OCI_IMAGE_ID`, `OCI_OCPUS`, `OCI_MEMORY_IN_GBS`, `OCI_SHAPE`, `OCI_MAX_INSTANCES`, `OCI_AVAILABILITY_DOMAIN`, `OCI_SSH_PUBLIC_KEY`, `CACHE_AVAILABILITY_DOMAINS`, `OCI_BOOT_VOLUME_SIZE_IN_GBS`, `OCI_BOOT_VOLUME_ID`, `TOO_MANY_REQUESTS_TIME_WAIT`, `TELEGRAM_USER_ID`, `TELEGRAM_BOT_API_KEY`, cộng thêm:
   - `OCI_PRIVATE_KEY_CONTENT` — toàn bộ nội dung file `.pem` (workflow sẽ ghi ra `/tmp/oci.pem`).
3. Push / vào Actions -> Hunt ARM -> Run workflow để test.
4. Tạo được máy rồi thì disable hoặc xóa workflow để dừng poll.

> Đừng dùng GitHub-hosted runner để poll liên tục ngoài việc test repo của chính bạn — có thể vi phạm [điều khoản GitHub Actions](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#actions). Muốn săn lâu dài hãy dùng [cron-job.org](#khuyên-dùng-cron-joborg) hoặc cron/VPS.

### Nhiều cấu hình

Truyền tên file env riêng làm tham số CLI cho nhiều tài khoản:

```bash
php index.php .env.my_acc1
```

Web SAPI (Apache/nginx) không dùng `$argv` — nếu cần thì tự map `$_GET` vào `$envFilename` trong `index.php`.

## Thông báo Telegram

1. Tạo bot qua [@BotFather](https://core.telegram.org/bots), lấy token -> `TELEGRAM_BOT_API_KEY`.
2. Lấy chat ID số của bạn (ví dụ qua [@userinfobot](https://t.me/userinfobot)) -> `TELEGRAM_USER_ID`.
3. Điền cả 2 vào `.env` (hoặc GitHub Secrets). Khi tạo máy thành công sẽ nhận JSON máy mới trong Telegram.

## Cách hoạt động

`index.php` -> `OciApi`:

1. `ListInstances` trong compartment. Nếu số máy khác `TERMINATED` cùng shape >= `OCI_MAX_INSTANCES` thì in `Already have an instance(s)...` và dừng.
2. `ListAvailabilityDomains` (hoặc dùng `OCI_AVAILABILITY_DOMAIN` / cache) để lấy danh sách AD cần thử.
3. `LaunchInstance` cho từng AD với `shapeConfig` (`ocpus`/`memoryInGBs`), `sourceDetails` (image hoặc boot volume), VNIC `assignPublicIp: false`.
4. Gặp `Out of host capacity` (500) thì sleep 16s rồi thử AD tiếp theo. Gặp 429 thì bật waiter nghỉ `TOO_MANY_REQUESTS_TIME_WAIT` giây. Lỗi khác thì dừng luôn.

## Gán public IP

Script tạo máy không kèm public IP (giới hạn 2 ephemeral/compartment). Sau khi thành công: OCI Console -> Instance Details -> Attached VNICs -> chọn VNIC -> IPv4 Addresses -> Edit -> Ephemeral -> Update.

![Attached VNICs](images/attached-vnics.png)

Đăng nhập (user mặc định `opc`):

```bash
ssh -i ~/.ssh/id_rsa opc@<public-ip>
# hoặc qua private DNS từ máy khác cùng VCN:
ssh -i ~/.ssh/id_rsa opc@instance-20210714-xxxx.subnet.vcn.oraclevcn.com
```

## Xử lý lỗi

| Hiện tượng | Nguyên nhân / cách sửa |
|---|---|
| `PrivateKeyFileNotFoundException` / file does not exist | Sai `OCI_PRIVATE_KEY_FILENAME`. Kiểm tra bằng `cat /path/to/oci.pem`. Dạng URL phải để trong ngoặc kép và `curl "https://..."` phải trả về key, không redirect/login |
| `Permission denied` khi đọc `.pem` | Sai ownership / chạy `chmod 600`. Đảm bảo user cron đọc được file |
| `InvalidParameter: Unable to parse message body` | `OCI_SSH_PUBLIC_KEY` bị xuống dòng. Phải là 1 dòng trong ngoặc kép |
| `InvalidParameter: Invalid ssh public key; must be in base64 format` | Paste sai key. Copy lại `~/.ssh/id_rsa.pub` hoặc tạo lại cặp key |
| `LimitExceeded: standard-a1-...` | Hết quota chứ không phải hết capacity — đã đủ máy tối đa hoặc cần xin tăng limit |
| `TooManyRequests` lặp lại / `Will retry after N seconds` | Bị OCI rate-limit. Tăng `TOO_MANY_REQUESTS_TIME_WAIT` hoặc giãn cron/schedule ra |
| `OCI_BOOT_VOLUME_ID and OCI_BOOT_VOLUME_SIZE_IN_GBS cannot be used together` | Chỉ được set 1 trong 2 |
| `OCI_AVAILABILITY_DOMAIN must be specified...` khi dùng boot volume | Boot volume gắn với AD — phải set `OCI_AVAILABILITY_DOMAIN` trùng AD của volume |

## Credits

- Project gốc + thư viện ký request OCI: [Alexander Hitrov](https://github.com/hitrov/oci-arm-host-capacity)
- [Bài Medium](https://hitrov.medium.com/resolving-oracle-cloud-out-of-capacity-issue-and-getting-free-vps-with-4-arm-cores-24gb-of-6ecd5ede6fcc) và [package signer](https://github.com/hitrov/oci-api-php-request-sign)
- Fork này: workflow GitHub Actions `hunt.yml`, cấu hình qua secrets, viết lại docs. MIT License — xem [LICENSE](LICENSE).
