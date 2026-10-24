**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
|frontend → cartservice |frontend.yaml env CART_SERVICE_ADDR และ demo.proto service CartService |✅ |src/frontend/main.go:134 อ่าน env CART_SERVICE_ADDR; main.go:143 เปิด connection ด้วย mustConnGRPC(..., svc.cartSvcAddr); src/frontend/rpc.go:58,63,68 เรียก
  pb.NewCartServiceClient(fe.cartSvcConn).GetCart/EmptyCart/AddItem(...) จริง ยืนยันว่าเป็น gRPC |
|checkoutservice → paymentservice |checkoutservice.yaml env PAYMENT_SERVICE_ADDR และ demo.proto service PaymentService |✅ |src/checkoutservice/main.go:116 mustMapEnv(&svc.paymentSvcAddr, "PAYMENT_SERVICE_ADDR"); main.go:370 เรียก
  pb.NewPaymentServiceClient(cs.paymentSvcConn).Charge(...) |
|checkoutservice → emailservice |checkoutservice.yaml env EMAIL_SERVICE_ADDR และ demo.proto service EmailService |✅ |main.go:115 mustMapEnv(&svc.emailSvcAddr, "EMAIL_SERVICE_ADDR"); main.go:380 เรียก pb.NewEmailServiceClient(cs.emailSvcConn).SendOrderConfirmation(...) |
|cartservice → redis-cart |cartservice.yaml env REDIS_ADDR และ readiness/liveness probe แบบ tcpSocket |✅ |ซอร์สของ cartservice เป็น C#/.NET และมีไฟล์ src/cartservice/src/cartstore/RedisCartStore.cs โดยตรง ยืนยันว่ามีการใช้ Redis client จริง |
|recommendationservice → productcatalogservice |recommendationservice.yaml env PRODUCT_CATALOG_SERVICE_ADDR และ demo.proto service ProductCatalogService |✅ |recommendation_server.py:131-136 อ่าน env แล้วสร้าง demo_pb2_grpc.ProductCatalogServiceStub(channel); บรรทัด 73 เรียก product_catalog_stub.ListProducts(...) จริง |
|frontend → shoppingassistantservice |ระบุไว้ใน Gaps ของ Run B ว่า "ไม่มี manifest ยืนยัน" |❓ |src/shoppingassistantservice/ มีซอร์สโค้ดจริง (shoppingassistantservice.py) แต่ grep ในทั้ง kubernetes-manifests/ และ kustomize/ ไม่พบไฟล์ manifest ของ service
  นี้เลย แสดงว่า service นี้มีอยู่จริงในระบบ แต่ไม่ได้ถูก deploy ผ่านไฟล์ evidence ที่ AI อ่านได้  |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ไม่พบ dependency ที่ไม่มีจริงเลยจากการสุ่มตรวจ 5 รายการ (และตรวจเพิ่ม frontend→cartservice อีก 1) ทุกรายการมีหลักฐานในซอร์สโค้ดตรงกับที่ Run B อ้าง ส่วน container ที่น่าสงสัยคือ shoppingassistantservice — Run B ระบุไว้ถูกต้องแล้วว่าเป็น Gap แต่การตรวจซอร์สโค้ดยืนยันว่า service นี้มีอยู่จริงในระบบ เพียงแค่ไม่ได้ถูก deploy ผ่าน manifest ที่ให้เป็น evidence ทำให้ไดอะแกรม C4 ที่ AI สร้างจาก evidence ที่จำกัดขาด container นี้ไปจากภาพรวมของระบบจริง เป็นตัวอย่างว่าขอบเขตของ evidence ส่งผลต่อความสมบูรณ์ของสถาปัตยกรรมที่ recover ได้

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
- ใครเรียกใคร (caller→callee) — เพราะ .proto มีแค่ signature ของ rpc ไม่มี client stub ที่ import กันจริง การใช้ message type ร่วมกัน (เช่น CartItem) เป็นแค่ Inferred ไม่ใช่หลักฐานว่าเรียกกันจริง
- ภาษา/เทคโนโลยีที่ implement จริงของแต่ละ service — go_package บอกแค่ชื่อ package ของ stub ที่ generate ร่วมกัน ไม่ได้บอกภาษาที่ implement จริง (Run B พบว่าจริงๆคนละภาษากันหมด)
- การมีอยู่ของ frontend/end user, data store (เช่น redis), message broker, หรือ external system จริง — เพราะ .proto อธิบายแค่ interface ของ rpc ไม่อธิบาย infrastructure
หรือ dependency ที่อยู่ "หลัง" การ implement
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
- Run A มี 9 containers แต่ไม่มี Rel เลยแม้แต่เส้นเดียว ทำให้สมาชิกใหม่เข้าใจผิดว่า services เป็นอิสระจากกันโดยสิ้นเชิง
- ความจริงจาก Run B คือ checkoutservice และ frontend เรียกออกไปรวมกันถึง 13 dependencies ผ่าน gRPC ทำให้ระบบ coupling กันแน่นมาก ไม่ใช่อิสระต่อกันแบบที่ Run A สื่อ
- ถ้านำแผนภาพ Run A ไปใช้ประเมิน blast-radius หรือผลกระทบจากการแก้ไข/ปิด service ใดๆ จะสรุปผิดว่าไม่กระทบใคร ซึ่งอาจนำไปสู่การ deploy ที่ทำให้ระบบอื่นล้มโดยไม่รู้ตัว
