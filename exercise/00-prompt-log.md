**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** นฤเมศร์ พิทยจำรัส / 6731328921
**agent ที่ใช้:** Claude Code
**โมเดล:** Sonnet 5
**แพ็กเกจที่ใช้:** Claude Pro $20

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Sonnet 5 | `prompts/1-…` run A |system/online-boutique/protos/demo.proto |อยู่ในขอบเขต | 5| |
| 2 | 1-B | Claude Code, Sonnet 5 | `prompts/1-…` run B |demo.proto + ทั้ง 10 ไฟล์ manifest ใน kubernetes-manifests/ (frontend.yaml, cartservice.yaml, checkoutservice.yaml, currencyservice.yaml,emailservice.yaml, adservice.yaml, paymentservice.yaml, productcatalogservice.yaml, recommendationservice.yaml, shippingservice.yaml, loadgenerator.yaml) + README.md, kustomization.yaml | อยู่ในขอบเขต | 5 | |
| 3 | 2-a | Claude Code, Sonnet 5 | `prompts/2-write-adr.md` run a (zero-shot) | docs/adr/template.md เท่านั้น (context: none) | อยู่ในขอบเขต | 3 | ไม่มี context เลย ADR จึงใส่ตัวเลข/ข้อจำกัดแบบทั่วไป ไม่ตรงกับระบบจริง |
| 4 | 2-b | Claude Code, Sonnet 5 | `prompts/2-write-adr.md` run b (with context) | docs/adr/template.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go | อยู่ในขอบเขต | 5 | มี context แล้ว ADR อ้างเลขจาก scenario และโค้ด `main.go` ได้ตรงจุด (เช่น `main.go:252-277`) |
| 5 | 3 | Claude Code, Sonnet 5 | `prompts/3-…` review `docs/adr/0002-adr.md` | docs/adr/0002-adr.md, docs/scenario.md | อยู่ในขอบเขต | 5 | ความประทับใจแรก: AI ชี้ risk และ sensitivity point ที่เจาะจงกับตัวเลขใน ADR (เช่น MailFast 50/s vs peak 200/s, latency budget ~200 ms) ได้ตรงประเด็นและเป็นรูปธรรม ไม่ใช่คำแนะนำทั่วไป |
| 6 | 4 | Claude Code, Sonnet 5 | "review adr 0002-adr.md แล้วตอบไฟล์ 04-review-board.md" → "ทำคนเดียวไปเลย แล้ว random ให้เราเปนสัก row แล้วให้ AI อีกตัวก็คือตัวนี้รีวิว" | exercise/04-review-board.md, docs/review-board/README.md + card-P/S/X ทั้ง 3 ใบ, docs/adr/0002-adr.md, docs/scenario.md, exercise/outputs/3-review.md | **นอกขอบเขตปกติของส่วน B** — ตาม README ปกติ AI ไม่ควรเห็นการ์ดบทบาทเลย แต่ทำตามคำขอของนิสิตให้ Claude จำลองทุกบทบาท (สถาปนิก/AI analyst/SRE/security) คนเดียวแทนกลุ่ม 5 คน จึงต้องอ่านการ์ดทั้งหมดเอง | 4 | ข้อค้นพบที่การ์ดให้มา (PDPA, คำโฆษณา "แต้มทันที", นโยบาย ops) ล้วนเป็นสิ่งที่ AI ไม่มีทางรู้ได้จากไฟล์ปกติ ยืนยัน insight ของ task นี้ได้ตรง แต่ความน่าเชื่อถือของ "การประชุม" ต่ำกว่าถ้ามีมนุษย์จริงเล่นแต่ละบทบาท |
| 7 | 5 | Claude Code, Sonnet 5 | "ทำ reflection ต่อด้วย" | exercise/05-reflection.md | อยู่ในขอบเขต (เขียนไฟล์ output ที่ระบุ) | — | งานนี้ควรเป็นความเห็นส่วนตัวของนิสิตเอง AI ช่วยร่างจากตัวอย่างในที่ประชุมข้อ 4 แต่นิสิตควรอ่านทวนและแก้ให้เป็นน้ำเสียง/ความเห็นของตัวเองก่อนส่ง |
