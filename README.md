# Souped Up — ระบบสมัครสมาชิกการแข่งขัน / Competition Registration System

ระบบสมัครสมาชิกและลงทะเบียนการแข่งขันรถยนต์ **Souped Up** โทนสีเข้มแนวแข่งรถ รองรับ 2 ภาษา (ไทย/อังกฤษ, ค่าเริ่มต้น: ไทย) ทำงานแบบ single-file ไม่ต้อง build

A dark, racing-themed registration flow for the **Souped Up** competition. Bilingual (TH/EN, default TH), single self-contained `index.html`, no build step.

## Public page
เปิดใช้งานผ่าน **GitHub Pages**: Settings → Pages → Deploy from branch `main` → root.
URL: `https://nidss.github.io/su/`

## ขั้นตอนการลงทะเบียน / Flow
1. **สมาชิก** — สมัครสมาชิก / เข้าสู่ระบบ (แบบ Tab)
2. **ข้อตกลงและเงื่อนไข** — อ่าน & ยอมรับเอกสาร 4 ฉบับ (มี popup)
3. **ข้อมูลผู้สมัคร** — ที่อยู่, ช่องทางติดต่อ, อัปโหลดบัตรประชาชน
4. **ข้อมูลนักแข่ง** — รถ, รุ่นแข่ง, เบอร์รถ (1–120), อัปโหลดรูป
5. **ที่จอดรถ** — เลือกช่องพีทจากผัง (แบบจองที่นั่ง)
6. **บัญชีคืนเงิน / รับเงินรางวัล**
7. **สรุปยอด** — กรอกโค้ดส่วนลด
8. **ชำระเงิน** — บัตรเครดิต / QR-โอนเงิน + ใบกำกับภาษี
9. **สำเร็จ** — หมายเลขอ้างอิง + ใบเสร็จ

## การรัน / Run
เปิดไฟล์ `index.html` ในเบราว์เซอร์ได้ทันที (ไม่มี dependency ที่ต้องติดตั้ง)

> หมายเหตุ: เป็นระบบ front-end demo — การสมัคร/ชำระเงินยังไม่เชื่อมต่อ backend จริง โค้ดส่วนลดตัวอย่าง: `SOUPEDUP`, `EARLYBIRD`, `VIP2026`
