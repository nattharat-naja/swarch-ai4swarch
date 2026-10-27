Agent: AI assistant using Copilot SDK in VS Code; model: not disclosed.

# ADR-0002: แยกงานหลังวางคำสั่งซื้อด้วย RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ยังไม่ได้ระบุในข้อมูลที่ให้มา

## Context and problem

ข้อกำหนด R-12 ระบุว่าเมื่อวางคำสั่งซื้อแล้ว ระบบต้องส่งอีเมลยืนยันผ่าน MailFast, ให้ `loyaltyservice` มอบแต้ม และบันทึก event "order placed" ให้ทีมการตลาด อีกทั้งคาดว่าจะมีงานหลังสั่งซื้อเพิ่ม เช่น SMS, fraud scoring และ recommendations update ([`docs/scenario.md`](../../docs/scenario.md), R-12)

ปัจจุบัน `checkoutservice.PlaceOrder` เรียก payment และ shipping ก่อนเรียก `emailservice` แบบ synchronous; หากส่งอีเมลไม่สำเร็จจะบันทึก warning แต่ยังคืนผลคำสั่งซื้อสำเร็จ ([`system/online-boutique/src/checkoutservice/main.go`](../../system/online-boutique/src/checkoutservice/main.go), `PlaceOrder` และ `sendOrderConfirmation`). การเรียก MailFast แบบ synchronous ไม่สอดคล้องกับเป้าหมาย p95 ของ checkout ที่ไม่เกิน 500 ms เพราะ MailFast มี p95 1.5 s และ p99 8 s. ปริมาณสูงสุด 200 orders/s ยังสูงกว่า rate limit ของ MailFast ที่ 50 emails/s ([`docs/scenario.md`](../../docs/scenario.md), How it works today และ Quality goals and constraints).

## Decision drivers

1. **Availability:** คำสั่งซื้อต้องไม่ล้มเหลวเมื่อ email, loyalty หรือ analytics ใช้งานไม่ได้
2. **Performance:** รักษา p95 ของ `PlaceOrder` ไม่เกิน 500 ms
3. **Correctness:** มอบ loyalty points ภายใน 1 นาที และไม่ให้แต้มซ้ำ
4. **Scalability and modifiability:** รองรับงานหลังสั่งซื้อเพิ่มเติมโดยไม่เพิ่มการเรียกแบบ synchronous ใน checkout
5. **Operational fit:** ใช้ส่วนประกอบ open-source ที่ self-host ได้บน Kubernetes; ทีมมีนักพัฒนาสองคนที่เคยใช้ RabbitMQ

## Considered options

1. เรียกบริการทั้งหมดแบบ synchronous จาก `checkoutservice`
2. ส่ง event ผ่าน RabbitMQ ไปยัง consumer แยกตามงาน พร้อม durable outbox และการประมวลผลซ้ำอย่างปลอดภัย
3. เริ่ม goroutine หรือ background worker ภายใน `checkoutservice` โดยไม่เพิ่ม message broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| เรียกแบบ synchronous | โฟลว์ตรงไปตรงมาและเห็นผลการเรียกทันที | MailFast p95 1.5 s เกินงบ p95 checkout 500 ms; การเรียกต่อเนื่องผูกความพร้อมใช้งานของงานเข้ากับ checkout; rate limit 50 emails/s ต่ำกว่า peak 200 orders/s |
| RabbitMQ + durable outbox | แยก checkout ออกจากเวลาตอบสนองและความพร้อมใช้งานของ consumer; queue รองรับการหน่วงและ retry; เพิ่ม consumer สำหรับงานใหม่ได้; เป็นเทคโนโลยี self-hosted ที่ทีมบางส่วนคุ้นเคย | เพิ่มส่วนประกอบและภาระปฏิบัติการ; ต้องดูแล outbox, queue, retry และ dead-letter; ต้องทำ consumer ให้รองรับข้อความซ้ำ |
| goroutine/background worker ใน checkoutservice | เปลี่ยนโค้ดได้ง่ายและไม่ต้องดูแล broker เพิ่ม | งานที่อยู่ในหน่วยความจำอาจสูญหายเมื่อ process หยุด; retry และการกระจายงานระหว่าง replicas ซับซ้อน; การเพิ่ม consumer อิสระทำได้ยาก |

## Decision

เลือกส่ง event `OrderPlaced` แบบ asynchronous ผ่าน RabbitMQ และให้แต่ละงานมี consumer ของตนเอง ได้แก่ consumer ที่เรียก MailFast, `loyaltyservice` และ analytics. ให้ `checkoutservice` บันทึก event ลง durable outbox และให้ publisher ส่ง event ไปยัง durable queue โดยไม่รอให้ consumer ทำงานเสร็จก่อนตอบ `PlaceOrder`. การเปลี่ยนขอบเขตความคงทนของข้อมูลและรายละเอียด storage ยังต้องยืนยันก่อน implementation; ปัจจุบันไฟล์โค้ดที่ให้มาไม่แสดงการ persist คำสั่งซื้อหรือ outbox.

ใช้การส่งแบบ at-least-once และกำหนด `orderId` เป็น idempotency key สำหรับ `loyaltyservice` เพื่อให้การส่งซ้ำไม่เพิ่มแต้มซ้ำ. Consumer ควร retry ความล้มเหลวแยกจาก checkout และส่งข้อความที่ retry ไม่สำเร็จไป dead-letter queue เพื่อให้ตรวจสอบและนำกลับมาประมวลผลได้. จำกัดการเรียก MailFast ไม่เกิน 50 ครั้งต่อวินาทีตามข้อกำหนด provider. ที่ peak 200 orders/s จะมีอีเมลเข้าคิวสุทธิโดยประมาณ 150 รายการต่อวินาที หากทุกคำสั่งซื้อมีอีเมลหนึ่งฉบับและไม่มีการระบายคิวเพิ่มเติม; จึงต้องติดตาม backlog และกำหนดแผนระบายคิว.

## Consequences

- **Positive:** ความล้มเหลวหรือ latency ของ MailFast, loyalty และ analytics ไม่ต้องรออยู่ใน request ของ checkout (availability, performance).
- **Positive:** consumer แต่ละตัวเพิ่มหรือลดกำลังแยกกันได้ และเพิ่ม action ใหม่โดยเพิ่ม consumer แทนการขยาย synchronous flow (scalability, modifiability).
- **Positive:** การบันทึกแต้มแบบ idempotent ป้องกันแต้มซ้ำเมื่อมีการส่ง event ซ้ำ (correctness).
- **Negative:** ต้องดูแล RabbitMQ, durable outbox, queue, retry และ dead-letter operationally (operability, cost).
- **Negative:** การประมวลผลเป็น eventual; อีเมลและ analytics อาจล่าช้า โดยเฉพาะเมื่ออัตราคำสั่งซื้อเกินข้อจำกัด MailFast (timeliness).
- **Risks and mitigations:** หาก publisher หรือ broker หยุดทำงาน event อาจค้างใน outbox; ใช้ durable storage, publisher confirms, retry และ alert สำหรับ outbox/queue backlog. หาก consumer ได้รับ event ซ้ำ ให้ใช้ `orderId` สำหรับ deduplication โดยเฉพาะการมอบแต้ม.
- **Risks and mitigations:** ข้อมูลที่ให้มายังไม่กำหนดวิธีผูกการยืนยันคำสั่งซื้อกับ durable outbox; ระบุ persistence และ failure semantics ให้ชัดก่อนเปิดใช้งาน เพื่อไม่ให้ตอบสำเร็จโดยไม่มี event ที่กู้คืนได้.

## Validation

- ทดสอบโหลดที่ 200 orders/s และยืนยันว่า p95 ของ `PlaceOrder` ไม่เกิน 500 ms ขณะ consumer ทำงานช้าและ MailFast ตอบช้าหรือขัดข้อง.
- ทดสอบ MailFast throttling ให้ไม่เกิน 50 emails/s; ตรวจว่าคิวรับภาระช่วง peak ได้และมี metric/alert สำหรับ backlog กับ dead-letter.
- จำลองการส่ง `OrderPlaced` ซ้ำและยืนยันว่า `loyaltyservice` มอบแต้มหนึ่งครั้งต่อ order และทำได้ภายใน 1 นาที.
- ปิดหรือทำให้ consumer แต่ละตัวใช้งานไม่ได้ชั่วคราว แล้วยืนยันว่า checkout ยังสำเร็จและงานถูกประมวลผลภายหลังเมื่อ consumer กลับมา.
- ทดสอบการหยุด publisher และ RabbitMQ ระหว่างมี outbox event แล้วตรวจว่าข้อความถูกส่งต่อได้หลังระบบกลับมา โดยไม่สูญหายและไม่ทำให้เกิดแต้มซ้ำ.

## Open questions

- ใครเป็นผู้ตัดสินใจ และใครรับผิดชอบปฏิบัติการ RabbitMQ กับ durable outbox?
- จะเก็บ outbox ที่ใด และจะกำหนดขอบเขต durable transaction อย่างไร เมื่อ `PlaceOrder` ในโค้ดปัจจุบันยังไม่แสดงการ persist คำสั่งซื้อ?
- หากการบันทึก outbox ล้มเหลวหลัง payment หรือ shipping สำเร็จ จะกำหนดสถานะคำสั่งซื้อและการกู้คืนอย่างไร?
- อีเมลยืนยันล่าช้าได้นานเท่าใด และ backlog/retention สูงสุดที่ยอมรับได้คือเท่าใด เมื่อ MailFast จำกัด 50 emails/s แต่ peak อยู่ที่ 200 orders/s?
- analytics ต้องการ schema, retention และ delivery guarantee ระดับใด?
