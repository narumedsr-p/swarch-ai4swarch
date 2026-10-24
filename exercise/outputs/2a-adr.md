_เขียนโดย: Claude Code (รุ่น Claude Sonnet 5) — Run A (zero-shot, context: none)_

# ADR-0002: ใช้ Message Broker แบบ Event-driven สำหรับงานหลังการสั่งซื้อ (Post-order Actions)

- **Status:** Proposed
- **Date:** 2026-10-23
- **Deciders:** <ระบุชื่อผู้ตัดสินใจ>

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานเพิ่มเติม 3 อย่าง:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** ซึ่งเป็นผู้ให้บริการอีเมลภายนอกรายใหม่ แทนที่ logic จำลองเดิมใน `emailservice`
2. มอบ**แต้มสะสม (loyalty points)** ผ่าน `loyaltyservice` ใหม่ (ยังต้องสร้าง)
3. บันทึก**เหตุการณ์ analytics "มีการสั่งซื้อ"** ให้ทีมการตลาด

ทีมการตลาดคาดว่าปีหน้าจะมีงานหลังการสั่งซื้อเพิ่มขึ้นอีก (SMS, การให้คะแนนความเสี่ยงการทุจริต, การอัปเดตระบบแนะนำสินค้า) จึงต้องออกแบบให้ขยายได้ง่ายโดยไม่ต้องแก้ไข flow การสั่งซื้อหลักทุกครั้งที่เพิ่มงานใหม่

ตัวเลขด้านปริมาณงาน (จำนวนคำสั่งซื้อต่อวินาที, ช่วงพีก), ข้อจำกัดของ MailFast (rate limit, SLA), และกฎการให้แต้มสะสม **ไม่ได้ให้มาในความต้องการ** — ดูหัวข้อ Open questions

## Decision drivers

- **Modifiability** — ต้องเพิ่มงานหลังการสั่งซื้อใหม่ได้ในปีหน้า (SMS, fraud scoring, recommendations) โดยกระทบ flow การสั่งซื้อหลักน้อยที่สุด
- **Reliability** — ความล้มเหลวของงานหนึ่ง (เช่น MailFast ล่ม) ไม่ควรทำให้คำสั่งซื้อล้มเหลวหรือทำให้งานอื่นไม่เกิดขึ้น
- **Loose coupling** — flow การสั่งซื้อหลัก (checkout) ไม่ควรรู้รายละเอียดของผู้ให้บริการภายนอกหรือ business logic ของแต้มสะสม/analytics โดยตรง
- **Latency ของผู้ใช้** — ลูกค้าไม่ควรรอจนกว่าอีเมล/แต้มสะสม/analytics ทำงานเสร็จก่อนได้รับการยืนยันคำสั่งซื้อ (ตัวเลข latency ที่ยอมรับได้ไม่ได้ให้มา)
- **Cost** — ไม่ได้ระบุงบประมาณหรือข้อจำกัดด้านทีม

## Considered options

1. **Synchronous orchestration** — `checkoutservice` (หรือ service ที่รับผิดชอบหลังสั่งซื้อ) เรียก MailFast, `loyaltyservice`, และระบบ analytics ตามลำดับแบบ synchronous ในคำขอเดียว
2. **Event-driven ผ่าน message broker** — เมื่อสั่งซื้อสำเร็จ ให้ publish event เช่น `OrderPlaced` ไปยัง message broker แล้วให้แต่ละ consumer (email, loyalty, analytics) subscribe และทำงานของตัวเองแบบ asynchronous
3. **Batch/scheduled job** — บันทึกคำสั่งซื้อไว้ แล้วมี job ที่รันเป็นช่วง (เช่นทุกนาที) มาอ่านคำสั่งซื้อที่ยังไม่ประมวลผลและเรียกแต่ละงานตามลำดับ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous orchestration | ง่ายต่อการเข้าใจและ debug ในช่วงแรก; ไม่ต้องตั้ง infrastructure เพิ่ม | ผู้ใช้ต้องรอทุก dependency เสร็จ (latency สูงขึ้นตามจำนวนงาน); ความล้มเหลวของ MailFast อาจกระทบคำสั่งซื้อ; ทุกครั้งที่เพิ่มงานใหม่ต้องแก้ checkout flow (ขัดกับ modifiability) |
| 2. Event-driven ผ่าน message broker | เพิ่มงานใหม่ (consumer ใหม่) ได้โดยไม่แก้ checkout flow; แยก failure domain ของแต่ละงาน; ลด latency ที่ผู้ใช้เห็น เพราะยืนยันคำสั่งซื้อได้ก่อนงานย่อยเสร็จ | ต้องดูแล infrastructure เพิ่ม (broker); ระบบเป็น eventual consistency ต้องจัดการกรณี event ซ้ำ/หาย; debug ยากขึ้นเพราะงานกระจายหลาย consumer |
| 3. Batch/scheduled job | Implement ง่าย ไม่ต้องมี broker | มี delay ตามรอบ job (ไม่เหมาะกับอีเมลยืนยันที่ควรส่งทันที); การเพิ่มงานใหม่ยังต้องแก้ไข job กลางเดียว (ขัดกับ modifiability ในระยะยาว) |

## Decision

เลือก **Option 2: Event-driven ผ่าน message broker** เพราะตอบโจทย์ decision driver ด้าน modifiability ได้ดีที่สุด — ปีหน้าจะมีงานเพิ่ม (SMS, fraud scoring, recommendations) และสถาปัตยกรรมนี้ให้เพิ่ม consumer ใหม่ได้โดยไม่ต้องแก้ checkout flow เดิม นอกจากนี้ยังตอบโจทย์ reliability และ loose coupling เพราะ MailFast, `loyaltyservice`, และ analytics ทำงานอิสระจากกัน ความล้มเหลวของตัวใดตัวหนึ่งไม่กระทบคำสั่งซื้อหรือตัวอื่น ส่วน latency ที่ผู้ใช้เห็นก็ลดลงเพราะยืนยันคำสั่งซื้อได้ทันทีหลัง publish event โดยไม่ต้องรอ consumer ทำงานเสร็จ

**ยังไม่ได้ระบุ**: ชนิดของ message broker ที่จะใช้ (เช่น RabbitMQ, Kafka) เพราะไม่มีข้อมูลด้านทีม/infrastructure ที่มีอยู่แล้วให้ตัดสินใจ — ดู Open questions

## Consequences

- **Positive:** เพิ่มงานหลังการสั่งซื้อใหม่ในอนาคตได้ง่ายขึ้นโดยเพิ่ม consumer ใหม่ (quality attribute: **modifiability**); ลด latency ที่ผู้ใช้เห็นหลังสั่งซื้อ (quality attribute: **performance**); ความล้มเหลวของ MailFast หรือ `loyaltyservice` ไม่ทำให้คำสั่งซื้อล้มเหลว (quality attribute: **availability**)
- **Negative:** ต้องตั้งและดูแล message broker เพิ่ม เพิ่มความซับซ้อนของระบบและต้นทุนการ operate (quality attribute: **cost**, **operability**); ระบบกลายเป็น eventual consistency ทำให้การ debug และการรับประกันลำดับงานยากขึ้น (quality attribute: **modifiability/testability**)
- **Risks and mitigations:**
  - Event อาจถูกประมวลผลซ้ำ (เช่น แต้มสะสมถูกให้มากกว่าหนึ่งครั้ง) — ต้องออกแบบ consumer ให้ idempotent แต่รายละเอียดกฎการให้แต้มสะสมไม่ได้ให้มา (ดู Open questions)
  - MailFast อาจมีข้อจำกัดด้าน rate limit หรือ SLA ที่ไม่ทราบ — อาจต้องมี retry/backoff หรือ dead-letter queue แต่ไม่มีตัวเลขให้ออกแบบนโยบายที่เจาะจงได้
  - Broker เป็นจุดเดียวที่ทุกงานหลังการสั่งซื้อพึ่งพา — ต้องพิจารณาความพร้อมใช้งาน (availability) ของ broker เอง

## Validation

<ไม่มีข้อมูลเพียงพอให้กำหนดวิธี validation ที่เจาะจง เช่น เป้าหมาย latency หรือ SLA — ควรกำหนดเมตริก เช่น เวลาเฉลี่ยตั้งแต่สั่งซื้อถึงได้รับอีเมลยืนยัน, อัตราการส่ง event ซ้ำ/หาย, และทดสอบ load ที่จำลองช่วงพีก แต่ต้องมีตัวเลขเป้าหมายก่อนจึงจะตรวจสอบได้จริง>

## Open questions

- ปริมาณคำสั่งซื้อต่อวินาทีและช่วงพีกที่ระบบต้องรองรับ — ไม่ได้ระบุ
- ข้อจำกัดของ MailFast (rate limit, SLA, การยืนยันการส่งสำเร็จ) — ไม่ได้ระบุ
- เป้าหมาย latency ที่ยอมรับได้สำหรับอีเมลยืนยันและการให้แต้มสะสม — ไม่ได้ระบุ
- กฎการให้แต้มสะสม (เช่น ต้องให้ครั้งเดียวต่อคำสั่งซื้อหรือไม่ จะทำอย่างไรหาก event มาซ้ำ) — ไม่ได้ระบุ
- Message broker หรือเทคโนโลยีที่ทีมมีความสามารถ/infrastructure อยู่แล้ว — ไม่ได้ระบุ
- งบประมาณหรือข้อจำกัดด้าน cost สำหรับการเพิ่ม infrastructure ใหม่ — ไม่ได้ระบุ
- Analytics event ต้องส่งไปที่ระบบปลายทางใด (data warehouse, event stream ภายนอก ฯลฯ) — ไม่ได้ระบุ
