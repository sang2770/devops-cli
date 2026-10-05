# devops-cli

CLI setup nhanh **nginx + HTTPS (certbot / Let's Encrypt)** cho domain, không cần tạo file cấu hình thủ công.

- Một file Python 3, chỉ dùng stdlib (không cần `pip install`)
- Hệ điều hành: Ubuntu / Debian (dùng `apt`, `sites-available/sites-enabled`, `systemd`)
- Cần quyền root (`sudo`)

## Cài đặt

```bash
git clone <repo> && cd devops-cli
chmod +x devops
sudo ln -s "$PWD/devops" /usr/local/bin/devops   # tuỳ chọn: gọi `devops` ở mọi nơi
sudo devops init                                  # cài nginx + certbot
```

`init` sẽ: cài `nginx`, `certbot`, `python3-certbot-nginx`; bật nginx và `certbot.timer` (tự gia hạn); mở firewall `Nginx Full` nếu có `ufw`.

## Chế độ menu (step by step)

```bash
sudo devops
```

```
1) Cài đặt nginx + certbot
2) Thêm site (domain → port/thư mục, + HTTPS)
3) Danh sách site
4) Cấp HTTPS cho site đã có
5) Bật / tắt site
6) Xoá site
7) Xem chứng chỉ SSL & hạn
8) Test gia hạn SSL
0) Thoát
```

Mục "Thêm site" hỏi lần lượt: domain → loại site (proxy / static / SPA) → port hoặc thư mục → `www` → giới hạn upload → HTTPS + email → xem tóm tắt → xác nhận.

## Chế độ lệnh (cho script / CI)

```bash
# Reverse proxy tới app ở port 3000, kèm www và HTTPS
sudo devops site add app.example.com --proxy 3000 --www --ssl --email me@example.com

# Proxy tới URL khác
sudo devops site add api.example.com --proxy http://10.0.0.5:8080 --ssl --email me@example.com

# Static site / SPA (fallback về index.html)
sudo devops site add blog.example.com --root /var/www/blog --spa --ssl --email me@example.com

sudo devops site list
sudo devops site enable|disable <domain>
sudo devops site remove <domain> [--cert]     # --cert: xoá luôn chứng chỉ

sudo devops ssl issue <domain> --email me@example.com [--www] [--staging]
sudo devops ssl status
sudo devops ssl renew [--test]
```

### Tuỳ chọn của `site add`

| Option | Ý nghĩa |
|---|---|
| `--proxy <port\|url>` | Reverse proxy (có sẵn header chuẩn + WebSocket) |
| `--root <dir>` | Static site, tự tạo thư mục nếu chưa có |
| `--spa` | Static: `try_files` fallback về `/index.html` |
| `--www` | Thêm `www.<domain>` vào `server_name` và cert |
| `--ssl --email <e>` | Cấp HTTPS bằng certbot, tự redirect HTTP → HTTPS |
| `--staging` | Dùng Let's Encrypt staging để test, không tốn rate limit |
| `--max-body <size>` | `client_max_body_size` (mặc định `20m`) |
| `--force` | Ghi đè site đã tồn tại (file cũ được backup `.bak`) |

Thêm `--dry-run` (đứng trước lệnh) để chỉ in cấu hình và lệnh sẽ chạy, không ghi file hay thực thi:

```bash
devops --dry-run site add app.example.com --proxy 3000 --ssl --email me@example.com
```

## Cách hoạt động

1. Sinh file `/etc/nginx/sites-available/<domain>` từ template, symlink sang `sites-enabled`.
2. Chạy `nginx -t`. Nếu lỗi: **tự rollback** (khôi phục file cũ hoặc xoá file mới).
3. `systemctl reload nginx`.
4. Nếu `--ssl`: `certbot --nginx --non-interactive --agree-tos --redirect -d <domain> ...`.

## Điều kiện để cấp HTTPS thành công

- DNS A/AAAA của domain (và `www.` nếu dùng) đã trỏ về IP server.
- Port 80 và 443 mở (firewall, security group của cloud).
- Domain chưa vượt rate limit của Let's Encrypt (dùng `--staging` khi thử nghiệm).

## Giới hạn hiện tại

- Chưa hỗ trợ wildcard cert (cần DNS challenge), template PHP/Docker, hay cấu hình nhiều domain từ file YAML.
- Chỉ Debian/Ubuntu.
