# Báo cáo tổng quan dự án & khả thi nâng cấp

> Repo: `baobd/qr-code-scanner` (fork của `code-kotis/qr-code-scanner`) · Ngày lập: 2026-10-24

## TL;DR

- **Dự án không sử dụng Laravel.** Không có PHP, `composer.json`, `artisan` hay backend nào. Đây là một
  **Progressive Web App (PWA) tĩnh viết bằng JavaScript thuần**, build bằng **webpack 4**, deploy lên Netlify.
- Vì vậy "nâng cấp lên Laravel mới nhất" **không phải là một bản nâng cấp** mà là **viết lại / thêm hẳn một
  backend mới**. Không khả thi theo nghĩa nâng cấp; chỉ hợp lý nếu có yêu cầu nghiệp vụ cần server.
- Thứ thực sự cần nâng cấp là **toolchain front-end** (webpack 4 → Vite hoặc webpack 5, Workbox 3 → 7,
  bỏ Google Analytics cũ). Việc này **khả thi cao, công sức nhỏ (~2–4 ngày)**.

---

## 1. Tổng quan dự án

### 1.1 Mục đích

Ứng dụng web quét mã QR từ camera (hoặc từ ảnh chọn trong máy), chạy offline, có thể "Add to Home Screen".
Live: <https://qrcodescan.in>.

### 1.2 Công nghệ

| Hạng mục | Hiện trạng |
|---|---|
| Ngôn ngữ | JavaScript ES2015 (vanilla, không framework) |
| Bundler | webpack 4.47 + webpack-cli 3, webpack-dev-server 3 |
| CSS | css-loader 1, extract-text-webpack-plugin 4 beta **và** mini-css-extract-plugin (dùng song song, dư thừa) |
| Service worker | workbox-webpack-plugin 3.6 (`GenerateSW`, `staleWhileRevalidate` cho mọi URL) |
| Giải mã QR | `app/decoder.js` — thư viện **ZBar biên dịch bằng Emscripten (asm.js)**, ~3,5 MB, chạy trong Web Worker |
| Dependency runtime | Chỉ `is-url` |
| Chất lượng code | Prettier 1, husky 1 + lint-staged 7. **Không có test, không có linter, không có CI đang chạy** |
| Deploy | Netlify (`_redirects`, `CNAME` → qrcodescan.in) |
| Node engine khai báo | `>=4.1.1` |

### 1.3 Cấu trúc

```
app/
  index.html          # App shell, nhúng Google Analytics (analytics.js) + pwacompat từ CDN
  manifest.json       # Web App Manifest
  decoder.js          # ZBar/Emscripten (3,5 MB), chạy trong Worker
  css/styles.css
  js/main.js          # Khởi tạo SW, UI (dialog, chọn ảnh), gọi QRReader
  js/snackbar.js      # Toast thông báo
  js/vendor/qrscan.js # Lấy luồng camera (getUserMedia), vẽ canvas, gửi frame cho decoder Worker
webpack.config.js
```

Luồng xử lý: `main.js` → `QRReader.init()` mở camera → mỗi frame vẽ lên `<canvas>` → `postMessage` ImageData
sang Worker `decoder.js` → ZBar trả kết quả → hiện dialog, nếu là URL thì cho nút "Open".

### 1.4 Tình trạng hiện tại

- Commit cuối: **25/05/2020** — dự án đã không được bảo trì ~6 năm.
- Build thử trên Node 22 / npm 10: `npm install` + `npm run build` **vẫn chạy được** (webpack 4.47 đã vá lỗi
  OpenSSL/md4), nhưng **không có lockfile** nên kết quả build phụ thuộc thời điểm cài.
- `npm audit`: **88 lỗ hổng (2 critical, 30 high, 51 moderate, 5 low)** — chủ yếu trong devDependencies
  (webpack-dev-server, loader cũ…), ảnh hưởng máy dev/CI nhiều hơn người dùng cuối.

### 1.5 Vấn đề kỹ thuật phát hiện được

| # | Vấn đề | Vị trí | Mức độ |
|---|---|---|---|
| 1 | Google Analytics `analytics.js` (Universal Analytics) đã ngừng hoạt động từ 07/2023; ngoài ra `ga('create', 'pageview')` thiếu tracking ID nên vốn không ghi nhận gì | `app/index.html` | Trung bình (code chết, tải script thừa) |
| 2 | Hai plugin trích CSS chạy song song (`extract-text-webpack-plugin` beta + `mini-css-extract-plugin`) | `webpack.config.js` | Thấp |
| 3 | Workbox 3: cache `staleWhileRevalidate` cho mọi URL `/`, SW cũ không còn được hỗ trợ | `webpack.config.js` | Trung bình |
| 4 | `snackbar.js` trộn `exports.` (CommonJS) với `import` (ESM) — không tương thích khi chuyển sang Vite/ESM thuần | `app/js/snackbar.js` | Trung bình (chặn nâng cấp) |
| 5 | Phát hiện iOS bằng `navigator.platform` (deprecated; iPadOS báo là Mac) | `app/js/main.js` | Thấp |
| 6 | Decoder asm.js 3,5 MB — nặng, chậm so với lựa chọn hiện đại (`BarcodeDetector` API, ZXing/zbar WASM ~ vài trăm KB) | `app/decoder.js` | Trung bình (hiệu năng) |
| 7 | `_redirects` dòng 2 thiếu TLD (`https://qrcodescan/*`) nên không bao giờ khớp | `_redirects` | Thấp |
| 8 | `engines.node >=4.1.1` không phản ánh thực tế; không có lockfile | `package.json` | Thấp |

---

## 2. Khả thi nâng cấp lên Laravel mới nhất

### 2.1 Kết luận

**Không áp dụng được theo nghĩa "nâng cấp".** Không có mã Laravel/PHP nào để nâng cấp. Phiên bản Laravel
mới nhất (dòng 13.x, yêu cầu PHP ≥ 8.3 — cần kiểm tra lại tại laravel.com/docs trước khi triển khai) chỉ có
thể được đưa vào bằng cách **xây mới một backend**.

### 2.2 Nếu vẫn muốn đưa Laravel vào — đánh giá

Toàn bộ tính năng hiện tại (quét QR, offline) chạy **100% phía client**; Laravel không cải thiện được
tính năng nào trong số đó. Laravel chỉ có giá trị nếu có yêu cầu mới cần server, ví dụ:

- Tài khoản người dùng, lưu **lịch sử quét** đồng bộ nhiều thiết bị
- **Tạo mã QR** động / rút gọn link / thống kê lượt quét
- Kiểm tra URL độc hại (gọi Safe Browsing API phía server)
- Trang quản trị

| Tiêu chí | Đánh giá |
|---|---|
| Khả thi kỹ thuật | Có — Laravel + Vite (mặc định của Laravel) có thể phục vụ lại front-end hiện tại trong `resources/js` |
| Tái sử dụng code | ~90% front-end (JS/CSS/decoder) giữ được, nhưng phải chuyển build sang Vite (xem §3) |
| Công sức ước tính | Khung Laravel + chuyển front-end: 3–5 ngày; mỗi tính năng backend thêm: tuỳ phạm vi (tuần) |
| Hạ tầng | **Thay đổi lớn**: từ hosting tĩnh miễn phí (Netlify) sang server PHP 8.3+ + DB (VPS, Forge, Laravel Cloud…), phát sinh chi phí & vận hành |
| Rủi ro | PWA offline phải giữ nguyên hành vi SW; tăng bề mặt tấn công (auth, DB); tăng chi phí |
| Khuyến nghị | **Không làm** trừ khi đã có yêu cầu nghiệp vụ cụ thể cần backend |

---

## 3. Đề xuất thay thế: hiện đại hoá toolchain front-end (khuyến nghị)

### 3.1 Phương án

| Phương án | Ưu | Nhược | Công sức |
|---|---|---|---|
| **A. Vite 7 + vite-plugin-pwa (Workbox 7)** — khuyến nghị | Cấu hình gọn (~20 dòng), dev server nhanh, cùng toolchain với Laravel nếu sau này thêm backend | Phải sửa `snackbar.js` sang ESM, xử lý `decoder.js` như asset tĩnh trong `public/` | 1–2 ngày |
| B. webpack 5 + Workbox 7 | Thay đổi ít, giữ cấu trúc cũ | Phải thay gần hết plugin (clean, copy, css, html, optimize-css → css-minimizer) | 1–2 ngày |

### 3.2 Lộ trình đề xuất (phương án A)

1. **Chuẩn hoá môi trường**: `engines.node >=20`, thêm `.nvmrc`, commit `package-lock.json`.
2. **Chuyển build sang Vite**: `index.html` ra gốc, `decoder.js` + `images/` + `manifest.json` vào `public/`,
   sửa `snackbar.js` thành `export { snackbar }`.
3. **Service worker**: `vite-plugin-pwa` (`generateSW`), precache app shell + decoder, bỏ rule cache mọi URL.
4. **Dọn dẹp**: bỏ Google Analytics cũ (hoặc thay GA4 / Plausible), sửa `_redirects`, bỏ `navigator.platform`.
5. **Tooling**: Prettier 3, ESLint 9 (flat config), husky 9 + lint-staged 15, GitHub Actions chạy lint + build.
6. **(Tuỳ chọn) Hiệu năng**: dùng `BarcodeDetector` API khi trình duyệt hỗ trợ, fallback sang decoder WASM
   (ví dụ `zxing-wasm` / `@undecaf/zbar-wasm`) thay asm.js 3,5 MB.
7. **Kiểm thử**: test thủ công trên Chrome Android, Safari iOS, desktop (camera + chọn ảnh + offline);
   Lighthouse PWA audit.

Kỳ vọng sau nâng cấp: `npm audit` về ~0 lỗ hổng, build/dev nhanh hơn rõ rệt, SW cập nhật đúng cách,
(nếu làm bước 6) giảm ~3 MB tải lần đầu.

### 3.3 Đánh giá khả thi tổng thể

| | Nâng cấp lên Laravel | Hiện đại hoá front-end |
|---|---|---|
| Áp dụng được? | Không (không có Laravel) — chỉ có thể xây mới | Có |
| Công sức | 3–5 ngày + tính năng backend | 2–4 ngày (gồm cả dọn dẹp & CI) |
| Chi phí vận hành | Tăng (server PHP + DB) | Không đổi (Netlify tĩnh) |
| Rủi ro | Trung bình–cao | Thấp |
| Khuyến nghị | Chỉ khi có yêu cầu backend | **Nên làm** |
