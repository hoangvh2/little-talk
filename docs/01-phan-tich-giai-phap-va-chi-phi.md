# LittleTalk – Play, Speak, Grow
## Phân tích yêu cầu, phương án giải pháp & dự toán chi phí

> Trạng thái: **Đề xuất – chờ quyết định**. Chưa lập trình.
> Ngày lập: 2026-10-17. Tỷ giá dùng: **1 USD ≈ 26.500 VND** (VCB bán ~26.070 ngày 10/10/2026 + phí thanh toán thẻ quốc tế ~1–3%).
> Ký hiệu độ tin cậy giá: ✅ nguồn chính thức/nhiều nguồn khớp · ⚠️ nguồn thứ ba, cần kiểm tra lại trên trang giá chính thức trước khi chi tiền.

---

## 1. Executive Summary

- **Khả thi về chi phí: CÓ**, với điều kiện **toàn bộ media được sản xuất trước (pre-generated) và phục vụ dạng file tĩnh**. Không gọi AI lúc runtime.
- **Chi phí vận hành MVP: ~0 – 30.000 VND/tháng** (chỉ tên miền nếu muốn; hosting, storage, băng thông nằm trong free tier).
- **Khi mở rộng 5.000–10.000 MAU: ~150.000 – 900.000 VND/tháng**.
- **Chi phí tạo nội dung ban đầu (MVP 12 bài): ~1 – 3,5 triệu VND tiền mặt** + **80–150 giờ công** (đây là chi phí thật lớn nhất).
- **Rủi ro chi phí lớn nhất** nằm ở 3 lựa chọn cần tránh: TTS/AI chạy runtime theo lượt dùng, video AI số lượng lớn, hội thoại LLM với trẻ.
- **Khuyến nghị: Phương án Hybrid chi phí thấp** – AI tạo ảnh (trả theo ảnh, vài chục USD) để có nhân vật đồng nhất → tái sử dụng bằng hoạt ảnh vector (Rive/Lottie) → TTS neural free tier/mã nguồn mở sinh sẵn → "video tình huống" thay bằng **cảnh tương tác ghép từ nhân vật + bối cảnh + audio** → PWA tĩnh trên Cloudflare, **local-first, không lưu dữ liệu trẻ trên server**.

### Thách thức giả định trong yêu cầu
| Yêu cầu | Vấn đề | Đề xuất |
|---|---|---|
| "Video tình huống" | Video AI tốn tiền theo giây, khó giữ nhân vật đồng nhất, không tương tác được, nặng băng thông | Dùng **cảnh hoạt hình tương tác** (nhân vật rigged + audio thoại). Video AI chỉ cho vài clip "hero" nếu thật sự cần |
| "Hội thoại thực tế" / luyện nói | Nhận dạng giọng trẻ 3–4 tuổi kém chính xác; Web Speech API trên Chrome gửi audio ra server bên thứ ba → rủi ro dữ liệu trẻ em | MVP: **hội thoại kịch bản rẽ nhánh** + "nghe – nhắc lại – ghi âm & tự nghe lại trên máy". Không chấm điểm phát âm tự động |
| AI tạo hội thoại động (LLM) | Không dự đoán được nội dung, chi phí theo lượt, rủi ro an toàn cho trẻ | Không đưa vào MVP |
| "3,5 tuổi" | Trẻ chưa biết đọc, tập trung 5–10 phút | UI **audio-first**, không chữ cho trẻ, nút to, phiên ngắn, phụ huynh đồng hành |

---

## 2. Phân tích yêu cầu

### 2.1 Người dùng
| Vai trò | Nhu cầu chính | Ràng buộc |
|---|---|---|
| Trẻ ~3,5 tuổi (mở rộng 3–6) | Chơi, nghe, chạm, bắt chước, nhận phản hồi vui | Chưa đọc; vận động tinh hạn chế (chạm/kéo đơn giản); tập trung ngắn |
| Phụ huynh | Biết con học gì, giới hạn thời gian, gợi ý áp dụng ngoài đời | Ít thời gian; cần giao diện tiếng Việt |
| Người soạn nội dung (bạn) | Thêm bài không sửa code; kiểm duyệt nhanh | Nguồn lực 1 người, part-time |

### 2.2 Nguyên tắc sư phạm (định hướng thiết kế)
- **Nghe hiểu trước, nói sau** (receptive → productive): Stage 1 chủ yếu nghe & chạm.
- **Cụm từ ngắn trong ngữ cảnh** ("More, please", "Bye-bye", "I'm hungry") thay vì từ đơn rời rạc.
- **Lặp lại có khoảng cách** (spaced repetition nhẹ): mỗi bài ôn 30–40% từ cũ.
- **TPR** (Total Physical Response – học qua vận động: "Clap your hands", "Touch your nose").
- **Áp dụng ngoài đời**: mỗi bài có "Nhiệm vụ ở nhà" cho phụ huynh (1–2 câu dùng trong bữa ăn, lúc đi ngủ…).
- **Tiến độ theo năng lực, không chỉ theo tuổi**: mở bài theo mức thành thạo (mastery).

| Stage | Độ tuổi tham chiếu | Mục tiêu | Dạng hoạt động chính |
|---|---|---|---|
| 1 – Listen & Point | 3–4 | Hiểu ~100 từ, ~30 cụm | Nghe & chạm, tìm đồ vật trong cảnh, bài hát, TPR |
| 2 – Say & Play | 4–5 | Nói từ/cụm 1–3 từ | Nhắc lại, ghép cặp, hội thoại kịch bản 2 lượt |
| 3 – Talk & Share | 5–6 | Trao đổi 2–4 lượt | Hội thoại rẽ nhánh, kể chuyện tranh, role-play |

### 2.3 Yêu cầu phi chức năng
- Chạy tốt trên **tablet/điện thoại tầm trung**, mạng yếu → **PWA offline** (tải gói bài về máy).
- Thời gian tải bài đầu < 3 s trên 4G; audio phản hồi < 100 ms sau khi chạm.
- **An toàn trẻ em**: không quảng cáo, không link ra ngoài, không chat, cổng phụ huynh (parent gate).
- **Mở rộng nội dung không sửa code**.

---

## 3. So sánh phương án tạo media

### 3.1 Tổng quan 5 phương án
| Tiêu chí | A. Tài nguyên miễn phí | B. Tự tạo bằng công cụ OSS | C. AI tạo nội dung (trả theo lượt) | D. Dịch vụ AI trả phí (subscription) | **E. Hybrid (khuyến nghị)** |
|---|---|---|---|---|---|
| Tiền mặt | 0 | 0 | Thấp (vài chục USD) | Trung bình–cao, định kỳ | Thấp |
| Công sức | Trung bình (tìm, lọc license) | **Rất cao** | Trung bình | Thấp | Trung bình |
| Nhân vật đồng nhất | ❌ Khó (nhiều tác giả) | ✅ | ⚠️ Cần ảnh tham chiếu + chỉnh tay | ⚠️ | ✅ |
| Chất lượng phát âm | ⚠️ Không đủ kho | ✅ (Kokoro) | ✅ | ✅ | ✅ |
| Rủi ro bản quyền | ⚠️ License hỗn hợp (CC BY-SA, attribution) | Thấp | ⚠️ Bản quyền ảnh AI chưa rõ ràng; tránh mô phỏng nhân vật có thương hiệu | ⚠️ | Thấp–TB |
| Tái sử dụng | Thấp | Cao | Trung bình | Trung bình | **Cao** |
| Phù hợp | Icon, hiệu ứng âm thanh, nhạc nền | Rig nhân vật, hoạt ảnh | Ảnh từ vựng, concept nhân vật | Không cần thiết | Toàn dự án |

### 3.2 Theo từng loại media

| Loại | Phương án rẻ nhất đạt yêu cầu | Phương án dự phòng | Không khuyến nghị |
|---|---|---|---|
| **Nhân vật** (3–4 nhân vật + 1 mascot) | AI ảnh tạo character sheet → vector hoá/chỉnh trong **Inkscape** → tách lớp (đầu, tay, miệng, mắt) | Vẽ tay Krita/Inkscape | Mỗi cảnh sinh AI riêng (mất đồng nhất) |
| **Bối cảnh** (nhà, bếp, công viên, lớp học…) ~8–10 | AI ảnh theo style guide cố định, tái dùng cho nhiều bài | Kenney/OpenMoji (CC0/CC BY-SA) cho đồ vật | — |
| **Ảnh từ vựng** ~250 | AI ảnh (Imagen 4 Fast ~$0,02/ảnh ⚠️; Gemini Flash Image ~$0,039/ảnh ⚠️, hỗ trợ ảnh tham chiếu để giữ style) | OpenMoji / Kenney | Ảnh stock trả phí |
| **Phát âm** (từ, cụm, lời dẫn) | **Azure Neural TTS free tier 0,5M ký tự/tháng** ✅ – đủ cho toàn bộ giáo trình; hoặc **Kokoro-82M** (Apache-2.0, chạy local, miễn phí) | Google Cloud TTS (Neural2 ~$16/1M, Chirp 3 HD ~$30/1M ⚠️) | Gọi TTS runtime; ElevenLabs Free (không được dùng thương mại) |
| **Bài hát / chant** | Tự thu (giọng người bản ngữ) hoặc bài public domain (Twinkle, Old MacDonald) + nhạc nền CC0 | TTS + nhạc nền | Nhạc có bản quyền |
| **Hoạt ảnh tương tác** | **Lottie** (OSS, editor Glaxnimate miễn phí) hoặc CSS/Web Animations trên lớp SVG | **Rive** (editor free, runtime MIT; export cần gói Cadet ~$9/tháng ⚠️ – chỉ trả trong tháng sản xuất, file đã export dùng vĩnh viễn) | Tự viết engine hoạt hình |
| **Video tình huống** | **Cảnh tương tác** = bối cảnh + nhân vật rigged + audio thoại + khẩu hình đơn giản (2–3 frame miệng) | Vài clip AI video: Veo 3.1 Lite ~$0,05/s, Veo 3.1 Fast ~$0,10–0,15/s ⚠️ | Video AI hàng loạt; quay dựng thật |
| **Hiệu ứng âm thanh** | Kenney audio (CC0), freesound CC0 | — | — |

**Vì sao không mặc định AI trả phí:** khối lượng text của giáo trình rất nhỏ (ước tính toàn bộ 3 stage ~60.000 ký tự), nên kể cả TTS trả phí cao cấp cũng chỉ tốn ~$1–2 nếu sinh sẵn một lần. Subscription (ElevenLabs Creator ~$22/tháng ⚠️, Midjourney…) là chi phí định kỳ không cần thiết.

---

## 4. Dự toán chi phí

### 4.1 Giả định
| Hạng mục | MVP | Mở rộng |
|---|---|---|
| MAU (trẻ hoạt động/tháng) | ≤ 500 (gia đình, bạn bè, nhóm thử) | 5.000 – 10.000 |
| Phiên/trẻ/tháng | 20 (5–10 phút/phiên) | 20 |
| Dữ liệu tải/phiên (chưa tính cache) | ~8 MB (ảnh WebP, audio Opus/MP3, hoạt ảnh); có video ~+7 MB | ~8 MB |
| Nội dung | 3 chủ đề, 12 bài, ~150 từ, ~60 cụm, ~300 file audio, ~250 ảnh | 6–10 chủ đề, 40–60 bài |
| Tổng media lưu trữ | < 100 MB | < 2 GB |
| Ghi dữ liệu tiến độ | Local (IndexedDB), không server | Đồng bộ tuỳ chọn: ~10 ghi/phiên |

Băng thông: MVP ~500 × 20 × 8 MB ≈ **80 GB/tháng**; mở rộng 10.000 × 20 × 8 MB ≈ **1,6 TB/tháng** (trước cache trình duyệt/PWA – thực tế thấp hơn nhiều vì gói bài được cache offline).

### 4.2 Chi phí vận hành hàng tháng (VND)

| Hạng mục | Dịch vụ | MVP | Mở rộng (10k MAU) | Cơ sở giá |
|---|---|---|---|---|
| Hosting web (PWA tĩnh) | Cloudflare Pages | 0 | 0 | Free: 500 build/tháng, không công bố giới hạn băng thông ✅/⚠️ |
| Lưu trữ + truyền tải media | Cloudflare R2 | 0 | 0 – 30.000 | Free 10 GB, 1M Class A, 10M Class B, **egress miễn phí**; vượt: $0,015/GB-tháng ✅ |
| Backend API | Cloudflare Workers | 0 (không cần) | 0 – 133.000 | Free 100k request/ngày ✅; gói Paid ~$5/tháng ⚠️ |
| Database | Không (local) → D1 hoặc Supabase | 0 | 0 – 663.000 | D1 free: 5M đọc, 100k ghi/ngày, 5 GB ⚠️ (chặn cứng khi vượt từ 01/09/2026 ✅). Supabase Free 500 MB, tự pause sau 1 tuần không dùng; Pro $25 ⚠️ |
| Auth phụ huynh | Không → Supabase Auth / tự làm magic link | 0 | 0 (trong gói) | Supabase Free 50k MAU ⚠️ |
| TTS | Sinh sẵn (Azure F0 / Kokoro) | 0 | 0 | Không gọi runtime |
| AI ảnh/video | Chỉ khi thêm bài mới | 0 | 100.000 – 800.000 (nếu thêm 4–8 bài/tháng) | Xem 4.3 |
| API AI (LLM, ASR) | Không dùng | 0 | 0 | — |
| Analytics | Cloudflare Web Analytics (không cookie) | 0 | 0 | — |
| Tên miền | `*.pages.dev` miễn phí, hoặc .com | 0 – ~30.000 | ~30.000 | Ước lượng phân bổ theo tháng; giá tuỳ nhà đăng ký – chưa kiểm tra |
| **Tổng** | | **~0 – 30.000** | **~150.000 – 900.000** (chủ yếu nếu chọn Supabase Pro + sản xuất nội dung đều đặn) | |

**Chi phí cố định vs biến đổi**
- Cố định: tên miền; (khi mở rộng) Workers Paid ~$5, Supabase Pro $25 nếu dùng.
- Biến đổi theo người dùng: gần như **0** nhờ egress R2 miễn phí + cache PWA. Biến đổi thật nằm ở **tốc độ sản xuất nội dung**, không phải số người dùng.

### 4.3 Chi phí tạo nội dung ban đầu (MVP 12 bài, một lần)

| Hạng mục | Khối lượng | Cách tính | VND |
|---|---|---|---|
| Ảnh AI (nhân vật, bối cảnh, từ vựng) | ~250 ảnh × ~3 lần thử | $0,02–0,04/ảnh ⚠️ → $15–30 | 400.000 – 800.000 |
| TTS | ~10.000–15.000 ký tự | Azure F0 / Kokoro | 0 |
| Hoạt ảnh | 4 nhân vật rig + ~20 animation | Lottie/Glaxnimate: 0; Rive Cadet 2–3 tháng ~$9/tháng ⚠️ | 0 – 720.000 |
| Video AI (tuỳ chọn) | ≤10 clip × 8 s × 3 lần thử | $0,05–0,15/s ⚠️ → $12–36 | 0 – 950.000 |
| Kiểm duyệt phát âm/nội dung bởi người bản ngữ hoặc GV | ~2–4 giờ | **Giả định**, chưa khảo sát giá freelancer | 0 – 1.000.000 |
| **Tổng tiền mặt** | | | **~0,4 – 3,5 triệu** |
| Công sức (ẩn) | Thiết kế giáo trình, prompt, chỉnh ảnh, rig, QA | | **80–150 giờ** |

### 4.4 Đối chứng: nếu dùng AI runtime (để thấy lý do không chọn)
| Kịch bản (10k MAU) | Phép tính | VND/tháng |
|---|---|---|
| TTS gọi mỗi phiên (~400 ký tự/phiên, Neural $16/1M ⚠️) | 10.000 × 20 × 400 = 80M ký tự → ~$1.280 | **~34 triệu** |
| Cùng nội dung nhưng sinh sẵn 1 lần | ~60.000 ký tự → ~$1 | **~0** (một lần ~27.000) |

### 4.5 Phương án chi phí thấp nhất có thể (0 VND tiền mặt)
- Kokoro TTS local + OpenMoji/Kenney + Lottie/Glaxnimate + Cloudflare Pages/R2 + `*.pages.dev` + local-first.
- **Giới hạn:** style ảnh không đồng nhất, nhân vật đơn giản, tốn nhiều giờ công hơn (~+50%); Kokoro đôi khi đọc sai từ/ngữ điệu → bắt buộc nghe kiểm tra từng file.
- **Rủi ro:** free tier có thể đổi chính sách (đã thấy D1 bắt đầu chặn cứng từ 09/2026); Supabase Free tự pause → không nên dùng làm phụ thuộc chính cho MVP.

---

## 5. Kiến trúc kỹ thuật đề xuất

```
[Nguồn nội dung trong Git]
  content/*.yaml (bài học, từ vựng, hội thoại)  +  assets nguồn (SVG, Lottie/Rive)
          │
          ▼  CI (GitHub Actions – free cho repo public/giới hạn phút cho private)
  1. Validate schema (thiếu asset, sai tham chiếu, chưa duyệt)
  2. Sinh TTS còn thiếu (key = hash(text + voice + rate))
  3. Chuẩn hoá audio (ffmpeg loudnorm, Opus/MP3), nén ảnh (WebP/AVIF)
  4. Đóng gói content pack + manifest.json (tên file theo content-hash)
          │
          ▼
  Cloudflare R2 (media, cache lâu dài)   Cloudflare Pages (PWA)
          │                                   │
          └──────────► Trình duyệt / Tablet ◄─┘
                       - Service Worker: cache gói bài offline
                       - IndexedDB: tiến độ, cài đặt (local-first)
                       - (Giai đoạn 2) Workers + D1: đồng bộ tài khoản phụ huynh
```

| Lớp | Lựa chọn | Lý do |
|---|---|---|
| Frontend | Vite + React (hoặc Svelte) + TypeScript, PWA (Workbox) | Phổ biến, nhẹ, offline tốt |
| Audio | Howler.js / Web Audio API | Độ trễ thấp, sprite audio |
| Hoạt ảnh | lottie-web hoặc @rive-app/canvas; SVG + CSS cho hiệu ứng đơn giản | Runtime OSS, file nhỏ, tương tác được |
| Kéo/thả | Pointer Events thuần | Tránh phụ thuộc nặng |
| Nội dung | YAML/JSON + JSON Schema (zod) trong Git; giai đoạn 2: Decap CMS (git-based, miễn phí) | Thêm bài = thêm file, không sửa code |
| Pipeline | Script Node/Python: TTS, ảnh, ffmpeg | Tự động hoá, chạy lại được |
| Backend | Không cho MVP → Cloudflare Workers + D1 | Chi phí 0, cùng hệ sinh thái |
| Test | Vitest (logic), Playwright (E2E trên Chromium có sẵn) | Kiểm tra mỗi activity template |

**Nguyên tắc mở rộng:** *Activity template là code (cố định ~6–8 loại), bài học là dữ liệu.* Thêm bài mới chỉ cần dữ liệu + asset; chỉ khi cần **loại trò chơi mới** mới sửa code.

---

## 6. Mô hình dữ liệu nội dung học tập

```yaml
# content/characters/mia.yaml
id: mia
name: Mia
role: child            # child | parent | teacher | mascot
voice: { provider: azure, voiceId: en-US-AnaNeural, rate: -10% }
rig: assets/characters/mia.lottie.json
states: [idle, talk, happy, wave, point]

# content/vocab/apple.yaml
id: apple
text: apple
tags: [food, fruit]
image: assets/vocab/apple.webp
audio: auto            # pipeline tự sinh
stage: 1
review: { status: approved, by: hoang, date: 2026-11-01 }

# content/lessons/food-01.yaml
id: food-01
theme: food
stage: 1
title: { en: "I'm hungry!", vi: "Con đói rồi!" }
objectives:
  understand: [apple, banana, milk, "more, please"]
  say: ["more, please"]
scene: kitchen
activities:
  - type: dialogue-scene          # cảnh tương tác thay video
    lines:
      - { speaker: mom, text: "Are you hungry, Mia?" }
      - { speaker: mia, text: "Yes! Apple, please." }
  - type: listen-and-tap
    prompt: "Where is the apple?"
    items: [apple, banana, milk]
    target: apple
  - type: say-it                  # nghe – nhắc lại – ghi âm cục bộ & nghe lại
    text: "More, please."
  - type: song
    ref: songs/yum-yum
review: [hello, bye-bye]         # ôn từ cũ
homeMission:
  vi: "Trong bữa tối, khuyến khích con nói 'More, please' khi muốn thêm đồ ăn."
```

| Thực thể | Trường chính | Ghi chú |
|---|---|---|
| Character | id, role, voice, rig, states | Dùng chung cho mọi bài |
| Scene | id, background, hotspots | Tái dùng nhiều bài |
| VocabItem / Phrase | id, text, image, audio, tags, stage | Audio tự sinh từ text |
| Activity | type, tham số theo template | ~6–8 loại: listen-and-tap, find-in-scene, drag-match, dialogue-scene, say-it, song, TPR, story |
| Lesson | theme, stage, objectives, activities, review, homeMission | Thuần dữ liệu |
| Asset metadata | path, hash, source, license, generator, prompt, reviewStatus | Truy vết bản quyền & kiểm duyệt |
| Progress (local) | itemId, lastSeen, correct/attempts, mastery | IndexedDB; SRS nhẹ |
| ParentSettings (local) | thời lượng tối đa/ngày, giọng US/UK, ngôn ngữ phụ huynh | Sau parent gate |

---

## 7. Quy trình sản xuất & kiểm duyệt nội dung

1. **Style guide** (1 lần): bảng màu, nét vẽ, tỉ lệ nhân vật, prompt mẫu, ảnh tham chiếu nhân vật.
2. **Soạn bài** bằng YAML (hoặc CMS) → trạng thái `draft`.
3. **Sinh asset tự động**: TTS cho text thiếu audio; ảnh AI theo prompt template + ảnh tham chiếu.
4. **Kiểm tra tự động (CI)**: thiếu file, sai tham chiếu, kích thước file vượt ngưỡng, âm lượng chuẩn (loudnorm), license trống, bài có asset chưa `approved`.
5. **Kiểm duyệt người** (checklist):
   - Phát âm đúng, tốc độ chậm vừa, ngữ điệu tự nhiên (người bản ngữ/GV duyệt mẫu).
   - Ảnh: không lỗi chữ/ngón tay AI, không bạo lực/đáng sợ, phù hợp văn hoá, không giống nhân vật thương hiệu (Peppa, Bluey…).
   - Phù hợp tuổi: câu ≤ 4–5 từ ở Stage 1–2.
6. **Thử nghiệm với trẻ thật** (con bạn): quan sát trẻ có hiểu yêu cầu không cần đọc, có hứng thú ≥ 5 phút.
7. `approved` → merge → CI deploy.

---

## 8. Bảo vệ dữ liệu trẻ em

Căn cứ: **Luật Bảo vệ dữ liệu cá nhân số 91/2025/QH15, hiệu lực 01/01/2026** (Điều 24 về dữ liệu trẻ em; xử lý dữ liệu trẻ cần sự đồng ý của người đại diện theo pháp luật). Nếu phục vụ người dùng Mỹ: COPPA. *Cần đối chiếu văn bản gốc và nghị định hướng dẫn trước khi mở công khai.*

| Biện pháp | MVP | Giai đoạn 2 |
|---|---|---|
| Tối thiểu hoá dữ liệu | Không thu thập tên thật, ảnh, giọng nói của trẻ; chỉ biệt danh + avatar chọn sẵn | Tài khoản = email phụ huynh |
| Lưu trữ | **Local-first** (IndexedDB trên máy) | Đồng bộ mã hoá đường truyền, xoá/xuất dữ liệu theo yêu cầu |
| Ghi âm luyện nói | Chỉ trong bộ nhớ, nghe lại tại chỗ, **không upload**, xoá khi rời màn hình | Giữ nguyên |
| Bên thứ ba | Không quảng cáo, không SDK tracking, không Web Speech API (gửi audio ra ngoài) | Analytics tổng hợp không cookie |
| Cổng phụ huynh | Phép tính đơn giản trước cài đặt, link ngoài | Đồng ý (consent) rõ ràng khi tạo tài khoản |
| Nội dung | Không chat, không UGC, không LLM tương tác với trẻ | Giữ nguyên trừ khi có đánh giá an toàn riêng |
| Minh bạch | Chính sách quyền riêng tư ngắn bằng tiếng Việt | Quy trình xử lý yêu cầu xoá dữ liệu |

---

## 9. Top 3 phương án tổng thể

| # | Phương án | Vận hành/tháng | Tạo nội dung MVP | Chất lượng | Rủi ro chính |
|---|---|---|---|---|---|
| **1 (Khuyến nghị)** | **Hybrid chi phí thấp**: AI ảnh trả theo lượt + vector/Lottie/Rive + TTS free tier/Kokoro + cảnh tương tác thay video + Cloudflare static + local-first | ~0–30k (MVP); ~150–900k (10k MAU) | ~0,4–3,5 triệu + 80–150 giờ | Cao, đồng nhất | Free tier đổi chính sách; công rig hoạt ảnh |
| 2 | **Miễn phí hoàn toàn**: chỉ OSS + tài nguyên CC0 | 0 | 0 + 120–220 giờ | Trung bình, style không đồng nhất | Tốn thời gian, trẻ kém hứng thú |
| 3 | **AI trả phí đầy đủ**: subscription TTS/ảnh/video, video AI cho mỗi bài, AI runtime | ~1–2 triệu (subscription) → **~34 triệu+** nếu runtime ở 10k MAU | ~5–15 triệu | Cao nhưng khó đồng nhất nhân vật | Chi phí không kiểm soát, an toàn nội dung |

**Vì sao chọn #1:** đạt chất lượng học tập (nhân vật đồng nhất, phát âm chuẩn, tương tác) với tiền mặt thấp hơn #3 hàng chục lần; ít giờ công hơn #2; chi phí gần như không tăng theo số người dùng.

---

## 10. Lộ trình MVP (1 người, part-time)

| Giai đoạn | Thời gian | Đầu ra | Cổng nghiệm thu (acceptance gate) |
|---|---|---|---|
| **0 – PoC** | 1–2 tuần | Style guide, 1 nhân vật + 1 mascot, 1 bối cảnh, **1 bài** với 3 activity (dialogue-scene, listen-and-tap, say-it), pipeline TTS | Con bạn tự chơi hết bài không cần đọc chữ; audio đúng chuẩn qua nghe kiểm tra; tải < 3 s trên 4G |
| **1 – MVP** | 4–6 tuần | 3 chủ đề × 4 bài = 12 bài; 6 activity template; PWA offline; tiến độ local + SRS nhẹ; màn hình phụ huynh (thời lượng, nhiệm vụ ở nhà); CI validate + deploy | Thêm 1 bài mới chỉ bằng YAML + asset, không sửa code; E2E pass cho 6 template; Lighthouse PWA đạt; 2–3 tuần dùng thật, trẻ quay lại ≥ 3 lần/tuần |
| **2 – Mở rộng** | Sau 4 tuần dùng thật | Stage 2, thêm 4–6 chủ đề, Decap CMS, đồng bộ tài khoản phụ huynh (Workers + D1), giọng US/UK | Dựa trên số liệu sử dụng thực |
| **3 – Tuỳ chọn** | — | Luyện phát âm có phản hồi (chỉ khi có giải pháp xử lý trên thiết bị/đạt yêu cầu riêng tư), clip video hero | ROI chứng minh được |

---

## 11. Rủi ro & nợ kỹ thuật cần theo dõi

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Ảnh AI lệch style / nhân vật không đồng nhất | Cao | Ảnh tham chiếu cố định; nhân vật chính vector hoá 1 lần rồi tái dùng |
| TTS đọc sai/ngữ điệu không tự nhiên cho trẻ | TB | Duyệt mẫu bằng tai; cho phép ghi đè file audio thu tay; SSML điều chỉnh tốc độ |
| Free tier thay đổi | TB | Kiến trúc tĩnh, dễ chuyển nhà cung cấp; không phụ thuộc DB ở MVP |
| Bản quyền ảnh AI/tài nguyên CC | TB | Lưu metadata nguồn/license/prompt cho từng asset |
| Trẻ mất hứng thú | Cao | PoC thử với trẻ thật trước khi sản xuất hàng loạt |
| Thời lượng màn hình | TB | Giới hạn phiên mặc định 10–15 phút, khuyến khích phụ huynh cùng chơi |

---

## 12. Quyết định cần bạn xác nhận

1. **Chọn Phương án #1 (Hybrid)?** Có / Không
2. **Phạm vi người dùng giai đoạn đầu:** (a) chỉ con bạn & người quen – *khuyến nghị* · (b) công khai
3. **Giọng chuẩn:** (a) US – *khuyến nghị (nhiều voice TTS hơn)* · (b) UK · (c) cả hai
4. **Thiết bị chính:** (a) tablet ngang – *khuyến nghị* · (b) điện thoại
5. **Chấp nhận bỏ video AI và chấm phát âm tự động khỏi MVP?** Có / Không

---

## Nguồn tham khảo giá & pháp lý (truy cập 2026-10-17)
- Cloudflare R2 pricing: https://developers.cloudflare.com/r2/pricing/ · tổng hợp: https://filebase.com/blog/cloudflare-r2-pricing-costs-savings-and-alternatives-in-2026/
- Cloudflare Pages limits: https://developers.cloudflare.com/pages/platform/limits
- Cloudflare D1 pricing & enforcement: https://developers.cloudflare.com/d1/platform/pricing/ · https://developers.cloudflare.com/changelog/post/2026-09-01-d1-free-tier-limit-enforcement/ · https://freetier.co/articles/cloudflare-d1-free-tier-limits-pricing-and-alternatives
- Cloudflare free limits tổng hợp: https://eastondev.com/blog/en/posts/dev/20260526-cloudflare-free-limits/
- Supabase pricing (tổng hợp): https://makerkit.dev/blog/md/saas/supabase-pricing · https://www.jetadmin.io/blog/supabase-pricing-2026-guide-to-plans-limits-and-real-world-costs/
- Azure Speech pricing: https://azure.microsoft.com/en-au/pricing/details/cognitive-services/speech-services/
- Google Cloud TTS pricing: https://cloud.google.com/text-to-speech/pricing · https://costbench.com/software/voice-apis/google-tts-api/
- ElevenLabs pricing (tổng hợp): https://bigvu.tv/blog/elevenlabs-pricing-2026-plans-credits-commercial-rights-api-costs/
- OpenAI image/TTS pricing (tổng hợp): https://pricepertoken.com/gpt-image-pricing
- Gemini/Imagen pricing: https://ai.google.dev/pricing · https://deploybase.ai/articles/gemini-api-pricing-2026
- AI video pricing: https://invideo.io/blog/ai-video-model-pricing
- Rive pricing: https://rive.app/blog/new-pricing · https://rive.app/docs/account-admin/pricing
- Kokoro vs Piper (license): https://www.promptquorum.com/power-local-llm/piper-vs-kokoro-tts
- Tỷ giá VCB: https://webgia.com/ty-gia/vietcombank/10-10-2026.html
- Luật BVDLCN 2025: https://luatvietan.vn/nhung-diem-moi-cua-luat-bao-ve-du-lieu-ca-nhan-2025-so-voi-nghi-dinh-13-2023-nd-cp.html · https://lsvn.vn/cong-bo-du-lieu-ca-nhan-cua-tre-tu-07-tuoi-phai-duoc-nguoi-dai-dien-dong-y-a167752.html

> Lưu ý: nhiều trang giá chính thức không truy cập trực tiếp được từ môi trường phân tích; các mục ⚠️ lấy từ nguồn thứ ba và cần kiểm tra lại trước khi thanh toán. Kết luận khả thi **không thay đổi** kể cả khi các giá ⚠️ lệch ±50%, vì phương án khuyến nghị không phụ thuộc chi phí runtime.
