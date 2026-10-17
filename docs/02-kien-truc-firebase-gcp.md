# LittleTalk – Kiến trúc & chi phí trên Firebase / GCP

> Trạng thái: **Đề xuất – chờ xác nhận**. Bổ sung cho `01-phan-tich-giai-phap-va-chi-phi.md`: thay hạ tầng Cloudflare bằng Firebase/GCP theo yêu cầu.
> Domain: **littletalk.vhub.id.vn**. Ngày lập: 2026-10-17. Tỷ giá: 1 USD ≈ 26.500 VND.
> ✅ = nguồn chính thức/nhiều nguồn khớp · ⚠️ = nguồn thứ ba, cần kiểm tra lại trên firebase.google.com/pricing trước khi bật billing.

## 1. Executive Summary
- **Firebase/GCP phù hợp**: Hosting có CDN, deploy kèm preview channel (rất hợp cho kiểm duyệt nội dung), Auth/Firestore sẵn cho giai đoạn 2, Cloud TTS + Vertex AI (Imagen/Veo) cùng một billing account.
- **Chi phí vận hành MVP: ~0 – 30.000 VND/tháng** – không đổi so với phương án Cloudflare. Domain đã có → bỏ chi phí tên miền.
- **Khác biệt quan trọng nhất: băng thông có tính phí.** Cloudflare R2 miễn phí egress; Firebase Hosting chỉ miễn phí ~10 GB/tháng, sau đó ~$0,15/GB ⚠️. Ở 10.000 MAU, nếu **không cache tốt** → ~4,8 triệu VND/tháng; nếu **cache PWA đúng** → ~0,2–0,7 triệu VND/tháng.
- **Bắt buộc dùng gói Blaze** nếu cần Cloud Storage (Spark không còn truy cập bucket ✅) hoặc gọi Cloud TTS/Vertex AI. Blaze **không có hạn mức chi cứng** → phải cài budget alert + cơ chế tự ngắt billing.

## 2. Ánh xạ dịch vụ

| Nhu cầu | Cloudflare (bản 01) | **Firebase/GCP (đề xuất)** | Ghi chú |
|---|---|---|---|
| Web app PWA | Pages | **Firebase Hosting** + custom domain `littletalk.vhub.id.vn` | SSL tự cấp; preview channel cho mỗi PR |
| Media (ảnh, audio, hoạt ảnh) | R2 | **MVP: đặt chung trên Firebase Hosting** → khi > ~50 GB/tháng chuyển sang **Cloud Storage for Firebase** | Hosting: CDN toàn cầu, deploy nguyên khối, rollback 1 click |
| Tiến độ học | IndexedDB (local) | **IndexedDB (local)** – giữ nguyên | Không cần backend ở MVP |
| Đồng bộ phụ huynh (GĐ2) | Workers + D1 | **Firebase Auth (email link/Google) + Firestore** | Firestore region `asia-southeast1` (Singapore) |
| TTS sinh sẵn | Azure F0 / Kokoro | **Google Cloud TTS** (Neural2/Chirp 3 HD) chạy trong CI; Kokoro làm phương án dự phòng | Tổng ký tự rất nhỏ → vài nghìn VND |
| Ảnh/video AI | Gemini API | **Vertex AI / Gemini API**: Imagen 4 Fast, Gemini Flash Image, Veo 3.1 Lite/Fast | Cùng billing GCP |
| CI/CD | GitHub Actions | GitHub Actions + **Workload Identity Federation** (không lưu key JSON) | Action chính thức của Firebase Hosting |
| Analytics | CF Web Analytics | **Không dùng Google Analytics for Firebase ở MVP** | Ứng dụng hướng tới trẻ em – xem mục 6 |

## 3. Bảng giá liên quan

| Dịch vụ | Miễn phí | Vượt mức | Độ tin cậy |
|---|---|---|---|
| Firebase Hosting – lưu trữ | 10 GB | ~$0,026/GB | ⚠️ |
| Firebase Hosting – truyền tải | ~10 GB/tháng | ~$0,15/GB | ⚠️ (trang chính thức hiển thị mức tương tự cho App Hosting: 10 GiB free, $0,15 cached / $0,20 uncached) |
| Cloud Storage for Firebase (bucket mới `*.firebasestorage.app`) | 5 GB-tháng, **100 GB tải xuống/tháng**; chỉ trên Blaze | Theo giá Cloud Storage | ✅ (có thể chỉ áp dụng region US – cần xác nhận) |
| Firestore | 1 GiB, 50k đọc/ngày, 20k ghi/ngày | Theo lượt | ⚠️ |
| Firebase Auth (email, Google) | 50.000 MAU | — | ⚠️ |
| Cloud TTS | Có free tier nhưng nguồn mâu thuẫn | Neural2 ~$16/1M ký tự, Chirp 3 HD ~$30/1M | ⚠️ |
| Imagen 4 Fast / Gemini Flash Image | — | ~$0,02 / ~$0,039 mỗi ảnh | ⚠️ |
| Veo 3.1 Lite / Fast | — | ~$0,05 / ~$0,10–0,15 mỗi giây | ⚠️ |

## 4. Dự toán chi phí vận hành (VND/tháng)

**Giả định băng thông thực tế có cache PWA:** trẻ tải mỗi gói bài 1 lần, các phiên sau dùng cache → ~30 MB/trẻ/tháng (app shell + gói bài mới). Kịch bản xấu (không cache): 8 MB × 20 phiên = 160 MB/trẻ/tháng.

| Hạng mục | MVP (≤500 MAU) | Mở rộng 10k MAU – có cache | 10k MAU – không cache (tránh) |
|---|---|---|---|
| Băng thông | 15 GB → ~5 GB vượt × $0,15 ≈ **20.000** | 300 GB: chuyển media sang Cloud Storage (100 GB free), phần còn lại theo giá egress ⚠️ → **~200.000 – 800.000** | 1,6 TB × ~$0,12–0,15 → **~5 – 6 triệu** |
| Hosting storage | 0 (< 1 GB, giới hạn số phiên bản lưu giữ) | 0 | 0 |
| Firestore + Auth | 0 (chưa dùng) | 0 – nếu ghi **1 document/phiên** (~7k ghi/ngày < 20k free) | Ghi mỗi sự kiện → vượt free tier |
| Cloud TTS / AI | 0 (chỉ khi sản xuất nội dung) | 0 – 800.000 (nếu thêm bài đều đặn) | — |
| Domain | 0 (đã có) | 0 | 0 |
| **Tổng** | **~0 – 30.000** | **~200.000 – 1,6 triệu** | **> 5 triệu** |

**Chi phí tạo nội dung ban đầu** giữ như bản 01: **~0,4 – 3,5 triệu VND** + 80–150 giờ công. TTS bằng Google Cloud: ~15.000 ký tự × $30/1M (Chirp 3 HD) ≈ **$0,45 (~12.000 VND)**.

### Điều kiện bắt buộc để giữ chi phí thấp trên Firebase
1. **Cache-Control dài hạn + tên file theo content-hash** (`apple.3f9a1c.webp`, `max-age=31536000, immutable`) trong `firebase.json`.
2. **Service Worker tải gói bài một lần**, phiên sau đọc từ cache.
3. Ảnh WebP/AVIF ≤ 60 KB, audio Opus ~32–48 kbps, không video dài.
4. Firestore: gộp tiến độ thành **1 lần ghi/phiên**.
5. Đặt **giới hạn số phiên bản Hosting lưu giữ** (ví dụ 10) để không vượt 10 GB storage.
6. Khi băng thông > ~50 GB/tháng: chuyển media sang Cloud Storage (100 GB free) hoặc đặt Cloudflare CDN phía trước (nếu DNS `vhub.id.vn` nằm trên Cloudflare).

## 5. Bảo vệ ngân sách trên Blaze
| Biện pháp | Mục đích |
|---|---|
| Budget alert 50% / 90% / 100% (ví dụ ngân sách 100.000 VND/tháng) | Cảnh báo sớm – **chỉ gửi thông báo, không tự chặn chi** |
| Cloud Function nhận Pub/Sub budget → tắt billing khi vượt ngưỡng (mẫu chính thức "cap costs" của GCP) | Chặn chi cứng; đánh đổi: app ngừng hoạt động đến khi bật lại |
| Firebase App Check (khi có Firestore/Storage) | Chặn bot gọi API, tránh "hoá đơn bất ngờ" |
| Security Rules từ chối mặc định | Không lộ dữ liệu, không cho ghi tuỳ ý |
| Service account riêng cho CI, quyền tối thiểu; API key giới hạn domain | Giảm rủi ro rò rỉ thông tin xác thực |

## 6. Dữ liệu trẻ em – lưu ý riêng cho Firebase
- **Không dùng Google Analytics for Firebase / AdMob** ở MVP: chúng thu định danh thiết bị; với ứng dụng hướng tới trẻ em cần cấu hình child-directed và đối chiếu chính sách Google – chưa xác minh đủ, nên tránh. Nếu cần số liệu: tự ghi sự kiện tổng hợp ẩn danh (đếm lượt hoàn thành bài) vào Firestore, không gắn định danh trẻ.
- **Không dùng Firebase Auth cho trẻ**; chỉ tài khoản phụ huynh (GĐ2), có màn hình đồng ý theo Luật 91/2025/QH15.
- Firestore đặt tại **asia-southeast1**; Security Rules: phụ huynh chỉ đọc/ghi document của mình.
- Ghi âm luyện nói **không upload** lên Storage.

## 7. Cấu hình domain `littletalk.vhub.id.vn`
1. Firebase Console → Hosting → Add custom domain → nhập `littletalk.vhub.id.vn`.
2. Thêm bản ghi DNS Firebase yêu cầu (TXT xác minh + A/CNAME) tại **nhà cung cấp DNS đang quản lý `vhub.id.vn`**.
3. Chờ cấp SSL tự động; bật redirect `*.web.app` → domain chính.
4. Nếu DNS nằm trên Cloudflare: để chế độ **DNS only (mây xám)** trong lúc xác minh/cấp SSL, tránh xung đột chứng chỉ.

## 8. Kiến trúc & quy trình nội dung (Firebase)
```
GitHub repo (content/*.yaml + assets nguồn)
   │  PR
   ▼
GitHub Actions (xác thực bằng Workload Identity Federation)
   ├─ validate schema
   ├─ sinh audio thiếu bằng Cloud TTS (cache theo hash text+voice+rate)
   ├─ tối ưu ảnh/audio, đặt tên theo content-hash, tạo manifest
   └─ deploy Firebase Hosting PREVIEW CHANNEL  → link để kiểm duyệt trên tablet
   │  merge vào main (sau khi duyệt)
   ▼
Firebase Hosting LIVE: littletalk.vhub.id.vn  (PWA + media, CDN)
   └─ (GĐ2) Firebase Auth + Firestore asia-southeast1: đồng bộ tiến độ phụ huynh
```
Mô hình dữ liệu nội dung (YAML), activity templates, quy trình kiểm duyệt: **giữ nguyên như bản 01 (mục 5–7)**. Preview channel thay cho bước "kiểm tra thủ công trước khi publish".

## 9. So sánh nhanh: Firebase/GCP vs Cloudflare

| Tiêu chí | Firebase/GCP | Cloudflare |
|---|---|---|
| Chi phí MVP | ~0 | ~0 |
| Chi phí khi lớn (10k MAU) | 0,2–1,6 triệu (có cache) | 0,15–0,9 triệu |
| Rủi ro hoá đơn bất ngờ | Cao hơn (Blaze không cap, egress tính phí) → cần budget + kill switch | Thấp (egress miễn phí) |
| Trải nghiệm dev | Tốt: preview channel, Auth/Firestore, TTS/AI cùng billing | Tốt nhưng ghép nhiều dịch vụ |
| Phù hợp hệ sinh thái của bạn | ✅ (lựa chọn của bạn) | — |

**Kết luận:** Đi theo Firebase/GCP là hợp lý; chênh lệch chi phí chỉ đáng kể khi > vài nghìn MAU và kiểm soát được bằng cache + chuyển media sang Cloud Storage/CDN. Không cần đổi lựa chọn.

## 10. Lộ trình (cập nhật)
| Giai đoạn | Việc chính trên Firebase/GCP | Acceptance gate |
|---|---|---|
| 0 – Setup (1–2 ngày) | Tạo project, Blaze + budget alert + kill switch, Hosting + domain, WIF cho GitHub Actions, bật Cloud TTS API | `littletalk.vhub.id.vn` có HTTPS; CI deploy preview từ PR; budget alert test được |
| 0 – PoC (1–2 tuần) | 1 bài, 3 activity, audio Cloud TTS | Như bản 01 + Lighthouse PWA đạt, header cache đúng |
| 1 – MVP (4–6 tuần) | 12 bài, offline, local progress | Như bản 01 + băng thông/trẻ/tháng đo thực tế ≤ 30 MB |
| 2 – Mở rộng | Auth phụ huynh + Firestore, App Check, chuyển media sang Storage khi cần | Chi phí/MAU đo thực tế |

## 11. Cần bạn xác nhận
1. Chấp nhận **bật Blaze** (bắt buộc cho Cloud TTS/Vertex AI/Storage) kèm budget alert + tự ngắt billing? Có / Không
2. Ngân sách trần đề xuất: **100.000 VND/tháng** cho giai đoạn MVP? Có / Khác
3. DNS của `vhub.id.vn` đang quản lý ở đâu? (Cloudflare / nhà đăng ký .vn / khác)
4. Giọng TTS: Google **Chirp 3 HD** (tự nhiên hơn, ~$30/1M) hay **Neural2** (~$16/1M)? Chênh lệch thực tế < 10.000 VND cho cả giáo trình → khuyến nghị nghe thử 2 giọng rồi chọn.

## Nguồn (truy cập 2026-10-17)
- Firebase pricing: https://firebase.google.com/pricing
- Thay đổi Cloud Storage for Firebase (Blaze, no-cost usage): https://firebase.google.com/docs/storage/faqs-storage-changes-announced-sept-2024
- Spark vs Blaze (tổng hợp, 06/2026): https://dev.to/androve2k/spark-vs-blaze-the-firebase-pricing-guide-i-wish-id-read-sooner-onb
- Firestore/Auth quota (tổng hợp): https://supertokens.com/blog/firebase-pricing · https://www.verdent.ai/guides/devtools/firebase-pricing
- Google Cloud TTS: https://cloud.google.com/text-to-speech/pricing · https://costbench.com/software/voice-apis/google-tts-api/
- Gemini/Imagen: https://ai.google.dev/pricing · https://deploybase.ai/articles/gemini-api-pricing-2026
- Veo pricing (tổng hợp): https://invideo.io/blog/ai-video-model-pricing
- Child-directed treatment (Google ads): https://firebase.google.com/docs/admob/android/targeting
