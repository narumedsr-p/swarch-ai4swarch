_เขียนโดย: Claude Code (รุ่น Claude Sonnet 5) — Run B (context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`)_

# ADR-0002: ใช้ Message Broker (RabbitMQ) แบบ Event-driven สำหรับงานหลังการสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-23
- **Deciders:** <ระบุชื่อผู้ตัดสินใจ>

## Context and problem

ปัจจุบัน `checkoutservice.PlaceOrder` เรียก `paymentservice`, `shippingservice` และ `emailservice` แบบ **synchronous ผ่าน gRPC ทีละตัวตามลำดับ** (`system/online-boutique/src/checkoutservice/main.go:252-277`) หากการเรียก `emailservice` ล้มเหลว โค้ดจะ log คำเตือนเท่านั้นและคำสั่งซื้อยังสำเร็จ (`main.go:273-277`, `sendOrderConfirmation` ที่ `main.go:379-384`)

Requirement ใหม่ (R-12) ต้องการให้หลังจากสั่งซื้อสำเร็จ:

1. ส่งอีเมลยืนยันผ่าน **MailFast** (ผู้ให้บริการภายนอกรายใหม่) แทน `emailservice` เดิม
2. มอบ**แต้มสะสม**ผ่าน `loyaltyservice` ใหม่ (ยังต้องสร้าง)
3. บันทึก**เหตุการณ์ analytics "order placed"** ให้ทีมการตลาด

และต้องขยายรองรับงานหลังการสั่งซื้อเพิ่มอีกในปีหน้า (SMS, fraud scoring, การอัปเดตระบบแนะนำสินค้า) โดยไม่กระทบ `checkoutservice` ทุกครั้งที่เพิ่มงานใหม่

ข้อเท็จจริงด้านคุณภาพที่ต้องพิจารณา (`docs/scenario.md`):

| หัวข้อ | ข้อเท็จจริง |
|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ต้อง **≤ 500 ms** (ปัจจุบันประมาณ 300 ms) |
| Peak load | **200 orders/s** ช่วงเซล 11.11 |
| MailFast | p95 **1.5 s**, p99 **8 s**; ล้มเหลวประมาณ **0.5%**; rate limit **50 emails/s** |
| Loyalty points | ต้องให้ภายใน **1 นาที**หลังสั่งซื้อ และ**ครั้งเดียวเท่านั้น** (ห้ามให้ซ้ำ) |
| Availability | คำสั่งซื้อ**ต้องไม่ล้มเหลว**เพราะ email, loyalty หรือ analytics ล่ม |
| Team | นักพัฒนา 4 คน แข็งแรง Go และ gRPC; 2 คนเคยใช้ **RabbitMQ** |
| Operations | รันบน Kubernetes อยู่แล้ว; บริษัทต้องการ **open-source, self-hosted** มากกว่า managed cloud service |

**ข้อสังเกตสำคัญ:** ที่ peak 200 orders/s แต่ MailFast รับได้เพียง 50 emails/s — อัตราคำสั่งซื้อสูงกว่าความสามารถของ MailFast ถึง 4 เท่า จึงไม่สามารถเรียก MailFast แบบ synchronous ต่อคำสั่งซื้อได้โดยตรงที่ peak โดยไม่ throttle (ดู Consequences)

## Decision drivers

1. **Performance:** `PlaceOrder` latency p95 ≤ 500 ms ต้องคงอยู่ แม้ MailFast จะมี p99 สูงถึง 8 วินาที
2. **Availability:** คำสั่งซื้อต้องไม่ล้มเหลวเพราะงานหลังการสั่งซื้อล่ม (ตรงกับ pattern ที่ `emailservice` ใช้อยู่แล้วใน `main.go:273-277`)
3. **Reliability/idempotency:** แต้มสะสมต้องมาถึงภายใน 1 นาที และครั้งเดียวเท่านั้น แม้ event จะถูกส่งซ้ำ
4. **Modifiability:** ต้องเพิ่มงานหลังการสั่งซื้อใหม่ในปีหน้าได้โดยไม่แก้ `checkoutservice`
5. **Team skill และ cost:** ควรใช้เทคโนโลยีที่ทีมมีความรู้อยู่แล้ว (RabbitMQ) และเป็น open-source/self-hosted ตามนโยบายบริษัท ไม่ใช่ managed cloud service

## Considered options

1. **ขยาย synchronous orchestration เดิม** — เรียก MailFast, `loyaltyservice`, analytics แบบ synchronous ต่อจาก `shipOrder`/`emptyUserCart` เหมือนที่ `emailservice` ถูกเรียกอยู่ในปัจจุบัน
2. **Event-driven ผ่าน RabbitMQ** — `checkoutservice` publish event `OrderPlaced` (durable queue) หลัง payment/shipping สำเร็จ แล้วให้ 3 consumer อิสระ (MailFast adapter, `loyaltyservice`, analytics collector) subscribe และทำงานของตนเอง
3. **Fire-and-forget แบบ asynchronous โดยไม่มี broker** — `checkoutservice` ยิง gRPC call ไปยัง MailFast adapter, `loyaltyservice`, analytics แบบ goroutine ที่ไม่รอผลลัพธ์ (ไม่มี queue หรือ persistence)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous orchestration | เข้ากับโค้ดเดิมได้ง่าย; ไม่ต้องเพิ่ม infrastructure | MailFast p95 1.5 s / p99 8 s ทำให้ `PlaceOrder` เกิน budget 500 ms ทันที ขัดกับ driver #1; ที่ peak 200 orders/s เกิน MailFast rate limit 50/s จะทำให้ request ค้างหรือถูก MailFast ปฏิเสธ; ทุกงานใหม่ปีหน้าต้องแก้ `checkoutservice` อีก (ขัดกับ driver #4) |
| 2. Event-driven ผ่าน RabbitMQ | Publish event ใช้เวลาสั้นมาก ไม่กระทบ latency ของ `PlaceOrder` (ตอบ driver #1); consumer แต่ละตัวแยก failure domain จากกัน ตรงกับ pattern เดิมที่คำสั่งซื้อไม่ล้มเหลวเพราะ email ล่ม (driver #2); durable queue รองรับ retry และ idempotency key (`orderID` ที่สร้างแล้วใน `main.go:233`) เพื่อกันแต้มซ้ำ (driver #3); เพิ่ม consumer ใหม่ได้โดยไม่แก้ `checkoutservice` (driver #4); ทีมมีประสบการณ์ RabbitMQ อยู่แล้ว และเป็น open-source self-hosted ตรงนโยบายบริษัท (driver #5) | ต้องตั้งและดูแล RabbitMQ cluster เพิ่ม; ที่ peak 200 orders/s ยังเกิน MailFast 50/s ทำให้ email backlog สะสมใน queue ช่วงเซล (ต้อง throttle consumer และยอมรับ delay); ระบบเป็น eventual consistency ทำให้ debug ยากขึ้น |
| 3. Fire-and-forget ไม่มี broker | Implement ง่ายกว่า option 2 ไม่ต้องตั้ง broker; latency ของ `PlaceOrder` ไม่กระทบเช่นกัน | ไม่มี persistence — หาก `checkoutservice` pod ตายหลังตอบ client แต่ก่อน goroutine ทำงานเสร็จ event จะหายไปเฉย ๆ ขัดกับ "แต้มสะสมต้องมาถึงภายใน 1 นาที" (driver #3); ไม่มีกลไก retry/backoff สำหรับ MailFast ที่ล้มเหลว 0.5% ของ call; เพิ่มงานใหม่ปีหน้ายังต้องแก้ `checkoutservice` เพิ่ม goroutine ใหม่ (ขัดกับ driver #4) |

## Decision

เลือก **Option 2: Event-driven ผ่าน RabbitMQ** เพราะเป็นตัวเลือกเดียวที่ตอบ decision driver ทั้ง 5 ข้อพร้อมกัน — คง latency ของ `PlaceOrder` ไว้ได้ (publish เร็วกว่าการเรียก MailFast ตรง ๆ มาก), รักษา availability ตาม pattern ที่มีอยู่แล้วในโค้ด, รองรับ exactly-once ของแต้มสะสมด้วย `orderID` ที่มีอยู่แล้วเป็น idempotency key, ขยายงานใหม่ได้โดยไม่แก้ `checkoutservice`, และใช้ RabbitMQ ซึ่งทีมมีประสบการณ์อยู่แล้วและตรงกับนโยบาย open-source/self-hosted ของบริษัท

`checkoutservice` จะ publish event `OrderPlaced` (payload มี `orderID`, user, items, address เหมือนใน `OrderResult` ที่ `main.go:265-271`) ไปยัง durable queue ทันทีหลัง `shipOrder`/`emptyUserCart` สำเร็จ แทนที่การเรียก `sendOrderConfirmation` แบบ synchronous เดิม ส่วน MailFast adapter, `loyaltyservice`, และ analytics collector เป็น consumer อิสระ 3 ตัวที่ subscribe event เดียวกัน

## Consequences

- **Positive:** `PlaceOrder` latency ไม่ขึ้นกับความช้าของ MailFast (quality attribute: **performance**) — publish event เร็วกว่าการรอ MailFast ที่ p99 สูงถึง 8 วินาทีมาก
- **Positive:** คำสั่งซื้อไม่ล้มเหลวเพราะ MailFast, `loyaltyservice` หรือ analytics ล่ม สอดคล้องกับ pattern เดิมของ `emailservice` (quality attribute: **availability**)
- **Positive:** เพิ่มงานหลังการสั่งซื้อใหม่ปีหน้า (SMS, fraud scoring, recommendations) เป็นเพียงการเพิ่ม consumer ใหม่ (quality attribute: **modifiability**)
- **Negative:** ต้องตั้งและดูแล RabbitMQ cluster เพิ่มเติม เพิ่มภาระ operations (quality attribute: **cost**, **operability**) — แม้ทีมมีประสบการณ์ RabbitMQ แต่มีเพียง 2 จาก 4 คน
- **Negative:** ระบบกลายเป็น eventual consistency ทำให้การ debug และตรวจสอบลำดับเหตุการณ์ยากขึ้น (quality attribute: **testability**)
- **Risks and mitigations:**
  - **MailFast rate limit (50/s) ต่ำกว่า peak order rate (200/s) ถึง 4 เท่า** — MailFast consumer ต้อง throttle การส่ง ทำให้ email ค้างใน queue ช่วงเซล 11.11 (อาจส่งช้ากว่าที่ลูกค้าคาดหวัง) ต้องให้ทีมการตลาดยืนยันว่ารับ delay นี้ได้ หรือเจรจาเพิ่ม rate limit กับ MailFast — **[Inferred]** ไม่มีตัวเลข SLA ของอีเมลที่ลูกค้าต้องได้รับภายในเวลาใดใน scenario ที่ให้มา
  - **Event ซ้ำ** — RabbitMQ แบบ at-least-once อาจส่ง event ซ้ำ ต้องให้ `loyaltyservice` ตรวจสอบ `orderID` ที่ประมวลผลแล้วก่อนให้แต้ม (idempotent consumer) เพื่อรักษา "ครั้งเดียวเท่านั้น"
  - **RabbitMQ เป็นจุดเดียวที่ทุก consumer พึ่งพา** — ต้องตั้ง replication/mirrored queue เพื่อไม่ให้ broker ล่มกระทบทุกงานพร้อมกัน

## Validation

- Load test จำลอง 200 orders/s (peak 11.11) และตรวจว่า p95 ของ `PlaceOrder` ยังคง **≤ 500 ms**
- ทดสอบส่ง duplicate `OrderPlaced` event ไปยัง `loyaltyservice` แล้วตรวจว่าแต้มสะสมถูกให้**ครั้งเดียว**และภายใน **1 นาที**
- วัด backlog ของ MailFast consumer queue ที่ peak load เพื่อประเมินว่า delay ของอีเมลยืนยันอยู่ในระดับที่ทีมการตลาดยอมรับได้หรือไม่ (ต้องกำหนดเกณฑ์ร่วมกับทีมการตลาดก่อน — ดู Open questions)

## Open questions

- ทีมการตลาดยอมรับ email backlog/delay ที่เกิดจาก MailFast rate limit (50/s) ต่ำกว่า peak order rate (200/s) ได้มากน้อยเพียงใด — ไม่ได้ระบุใน scenario
- หาก RabbitMQ เองล่มหรือ queue เต็ม `checkoutservice` ควร fallback อย่างไร (บัฟเฟอร์ภายใน, ปฏิเสธ publish แล้ว log เหมือน `emailservice` เดิม, หรืออื่น) — ไม่ได้ระบุ
- Analytics event ต้องมี timing/SLA เฉพาะหรือไม่ (scenario ระบุเฉพาะ loyalty points ว่าต้องภายใน 1 นาที) — ไม่ได้ระบุ
- `emailservice` เดิมจะถูกถอดออกทั้งหมดหรือเก็บไว้เป็น fallback เมื่อ MailFast ล้มเหลว (0.5% ของ call) — ไม่ได้ระบุ
- Retention policy ของ queue (เก็บ event ที่ประมวลผลไม่สำเร็จไว้นานเท่าใดก่อน dead-letter) — ไม่ได้ระบุ
