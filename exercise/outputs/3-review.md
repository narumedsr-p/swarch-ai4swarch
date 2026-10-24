_เขียนโดย: Claude Code (รุ่น Claude Sonnet 5) — review `docs/adr/0002-adr.md` (context: `docs/adr/0002-adr.md`, `docs/scenario.md`)_

# Review ADR-0002: Event-driven ผ่าน RabbitMQ สำหรับงานหลังการสั่งซื้อ

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| 1 | ลูกค้า (load generator ช่วงเซล) | สั่งซื้อที่ 200 orders/s | ช่วงพีก 11.11, MailFast ตอบช้า p99 8 s | `checkoutservice` publish `OrderPlaced` แล้วตอบ client ทันทีโดยไม่รอ MailFast/loyalty/analytics | p95 ของ `PlaceOrder` ≤ 500 ms |
| 2 | RabbitMQ broker (redeliver หลัง consumer ack ไม่สำเร็จ/restart) | `loyaltyservice` ได้รับ `OrderPlaced` event ซ้ำ (at-least-once) | การทำงานปกติหรือช่วง consumer restart | `loyaltyservice` ตรวจ `orderID` ที่ประมวลผลแล้วก่อนให้แต้ม | ให้แต้มสะสม**ครั้งเดียว** และภายใน **1 นาที** จากเวลาสั่งซื้อ |
| 3 | MailFast / `loyaltyservice` / analytics collector | หนึ่งในสาม consumer ล่มหรือ RabbitMQ ไม่ตอบสนองชั่วคราว | ทุกช่วงเวลา รวมถึงช่วงพีก | `checkoutservice` ยังคืนผล order สำเร็จ ไม่ผูกกับสถานะของ consumer เหล่านี้ | 0% ของ order ล้มเหลวเพราะสามงานนี้ (ตรงกับ pattern เดิมของ `emailservice`) |

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Risk | MailFast rate limit 50/s ต่ำกว่า peak order rate 200/s ถึง 4 เท่า → backlog สะสม 150 orders/s ตลอดช่วงพีก แต่ ADR ไม่มีตัวเลขความยาวของช่วงพีกเพื่อคำนวณ queue depth สูงสุดที่ต้องใช้ | ถ้าช่วงพีกยาวหลายนาที backlog อาจเติบโตจนใช้ memory/disk ของ RabbitMQ มาก หรือ delay อีเมลยืนยันเกินกว่าที่ลูกค้า/ทีมการตลาดยอมรับได้ โดยไม่มีทางรู้ล่วงหน้าจากตัวเลขที่มี |
| Risk | ADR อ้างว่า "publish event ใช้เวลาสั้นมาก ไม่กระทบ latency" แต่ไม่มีตัวเลข publish-to-ack latency รองรับ claim นี้ โดยเฉพาะที่ peak 200 orders/s ซึ่ง budget ที่เหลือมีเพียง ~200 ms (500 − 300 ปัจจุบัน) | เป็น claim ที่ไม่ labeled ว่า **[Inferred]** ทั้งที่ไม่มีหลักฐานจาก `docs/scenario.md` หรือ measurement ใด ๆ รองรับ หาก publish+confirm ของ durable queue ช้ากว่าที่คาด driver #1 (performance) อาจไม่บรรลุ |
| Non-risk | ความถูกต้องของ "ให้แต้มครั้งเดียว" ไม่ได้พึ่ง RabbitMQ (ซึ่งเป็น at-least-once เท่านั้น) แต่พึ่ง idempotent consumer ที่ตรวจ `orderID` ก่อนให้แต้ม | เป็นการออกแบบที่ถูกต้องตามธรรมชาติของ message broker ทั่วไป และ ADR ระบุข้อจำกัดนี้อย่างชัดเจนด้วยตนเอง ไม่ได้หลงเชื่อว่า broker guarantee exactly-once |
| Sensitivity point | PlaceOrder latency อ่อนไหวสูงต่อ publish-to-broker latency เพราะ budget ที่เหลือเพียง ~200 ms เท่านั้น การเพิ่ม network hop หรือ RabbitMQ cluster ช้าลงเล็กน้อยภายใต้ peak load อาจกิน budget ไปมาก | หาก sensitivity นี้ไม่ถูกวัดแยกจาก end-to-end latency จะไม่รู้ว่าปัจจัยใดเป็นตัวกินเวลาจริงเมื่อ p95 เกิน 500 ms ในอนาคต |
| Trade-off point | เลือก async event-driven เพื่อ performance/availability/modifiability แต่แลกด้วย testability (eventual consistency ทำให้ debug ลำดับเหตุการณ์ยากขึ้น) และ operational cost (ต้องดูแล RabbitMQ cluster + replication โดยมีเพียง 2 จาก 4 คนที่มีประสบการณ์) | เป็น trade-off ที่ ADR ระบุไว้ตรงไปตรงมาในหัวข้อ Consequences แต่ไม่ได้ประเมินว่าทีมขนาด 4 คนรับภาระ operational นี้เพิ่มได้จริงหรือไม่ในระยะสั้น |
| Trade-off point | Durable queue + retry แก้ reliability ของแต่ละ consumer ได้ แต่ ADR ไม่พูดถึง ordering guarantee ของ event เลย ซึ่งอาจกระทบ analytics ที่ต้องการลำดับ "order placed" ที่ถูกต้องสำหรับทีมการตลาด | ถ้า 3 consumer ประมวลผล event ไม่เรียงตามลำดับ (เช่น retry ทำให้ event มาถึงช้า) analytics ที่นับยอดขายตามเวลาอาจผิดเพี้ยนโดยไม่มีการกล่าวถึงความเสี่ยงนี้เลย |

## 3. Missing or weak

- **Throttling mechanism ไม่เป็นรูปธรรม** — ADR บอกว่า MailFast consumer "ต้อง throttle ที่ 50 emails/s" แต่ไม่ระบุกลไกจริง (prefetch count, rate limiter, queue depth เพดาน, dead-letter policy) ทั้งที่นี่คือ risk หลักที่ระบุไว้เอง
- **Fallback เมื่อ RabbitMQ เองล่ม** ถูกเปิดไว้เป็น Open question แต่ decision drivers #2 (availability) ระบุชัดว่า order ต้องไม่ล้มเหลวเพราะงานหลังการสั่งซื้อ — RabbitMQ เป็น SPOF ใหม่ที่ไม่มีคำตอบว่า publish ล้มเหลวแล้วจะเกิดอะไรกับ order
- **ตัวเลือกลูกผสมไม่ถูกพิจารณาอย่างเป็นทางการ**: เก็บ `emailservice`/synchronous เดิมไว้เป็น fallback เฉพาะกรณี MailFast ล้มเหลว (0.5%) ถูกยกไปเป็น Open question เฉย ๆ แทนที่จะเป็น option ที่ 4 ที่มี pros/cons เทียบกับ option 2
- **Idempotency store ของ `loyaltyservice` ไม่มีรายละเอียด** — เก็บ `orderID` ที่ประมวลผลแล้วไว้ที่ไหน (DB อะไร), รองรับ throughput ที่ peak 200 orders/s ได้หรือไม่ ไม่ถูกกล่าวถึงทั้งที่ `loyaltyservice` เป็น service ใหม่ที่ยังไม่มี baseline
- **Validation ไม่ได้แยกวัด publish-to-ack latency** จาก end-to-end `PlaceOrder` latency ทั้งที่เป็น sensitivity point ที่สำคัญที่สุดของ decision นี้

## 4. Verdict

**Accept with changes.** โครงสร้างการตัดสินใจ (async publish + idempotent consumer) ตอบ decision drivers ได้ถูกทาง แต่ยังมี risk และ gap สำคัญที่ต้องปิดก่อนนำไป implement จริง สามการเปลี่ยนแปลงที่สำคัญที่สุด:

1. วัด publish-to-broker (รวม ack) latency แยกจาก end-to-end `PlaceOrder` latency ที่ peak 200 orders/s เพื่อยืนยันว่า sensitivity point ด้าน budget ~200 ms ไม่ถูกละเมิด
2. ระบุกลไก throttling ที่เป็นรูปธรรมสำหรับ MailFast consumer (prefetch/rate limit/queue depth เพดาน/dead-letter) พร้อมขอ SLA backlog delay ที่ยอมรับได้จากทีมการตลาดก่อนอนุมัติ
3. ตัดสินใจและระบุ fallback behavior เมื่อ publish ไปยัง RabbitMQ ล้มเหลวหรือ broker ไม่พร้อมใช้งาน ก่อนเปลี่ยน status จาก Proposed เป็น Accepted เพราะกระทบ availability driver ตรงๆ
