ผู้รีวิว: AI assistant using Copilot SDK in VS Code; model: ไม่ทราบ

## Quality-attribute scenarios

1. **Performance / availability:** ลูกค้าส่ง `PlaceOrder` ระหว่างยอดขาย 11.11 ที่ 200 orders/s ขณะที่ MailFast ช้าหรือล่ม (p95 1.5 s, p99 8 s; ล้มเหลวราว 0.5%) ระบบตอบรับโดยไม่รอ consumer และบันทึกงานไว้อย่างทนทาน; p95 ของ checkout ต้อง ≤ 500 ms และคำสั่งซื้อต้องไม่ล้มเหลวเพราะบริการปลายทางหยุดทำงาน (scenario: “Quality goals and constraints”; ADR: “Decision”).
2. **Correctness / performance:** `loyaltyservice` ได้รับเหตุการณ์ซ้ำหรือทำงานภายใต้โหลดสูงสุด 200 orders/s; ต้องบันทึกคะแนนด้วย idempotency key และประมวลผลซ้ำได้โดยไม่ให้คะแนนซ้ำ; คะแนนของแต่ละคำสั่งซื้อต้องถูกให้ภายใน 1 นาทีและไม่มีคำสั่งซื้อใดได้คะแนนซ้ำ (scenario: “Quality goals and constraints”; ADR: “Decision”).
3. **Availability / capacity:** ทุกคำสั่งซื้อที่โหลดสูงสุดสร้างอีเมลหนึ่งฉบับ แต่ MailFast รับได้ 50 emails/s; ระบบจำกัดอัตรารวม retry ไม่เกิน 50/s และเก็บงานค้างไว้ โดยอาจมี backlog เพิ่มราว 150 ฉบับ/s ตลอดช่วงที่โหลดสูงกว่าอัตราบริการ; ต้องกำหนดเกณฑ์อายุ/ขนาด backlog และระยะเวลาส่งอีเมลที่ยอมรับได้ (scenario: “Quality goals and constraints”; ADR: “Decision”).

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Non-risk** | RabbitMQ พร้อม durable outbox และ consumer แยกกันเหมาะกับการไม่ผูกเวลา checkout กับ MailFast และรองรับงาน post-order ใหม่ | เข้ากับเป้าหมาย checkout ≤ 500 ms และความต้องการเพิ่ม SMS, fraud scoring และ recommendations (scenario: “The new requirement”; ADR: “Decision”). |
| **Risk** | การรับประกัน order/outbox commit แบบ atomic ยังเป็นเงื่อนไข ไม่ใช่แบบที่ยืนยันได้; ADR ยังไม่ระบุฐานข้อมูลหรือ transaction boundary | หากบันทึก order สำเร็จแต่ event ไม่ถูกบันทึก หรือกลับกัน งานหลังสั่งซื้ออาจสูญหาย/ไม่สอดคล้อง จึงยังยืนยันความทนทานของแนวทางไม่ได้ (ADR: “Decision”, “Open questions”). |
| **Sensitivity point** | อัตราส่ง MailFast 50/s เทียบกับ peak 200 orders/s เป็นตัวกำหนด backlog โดยตรง; retry ยังใช้โควตาเดียวกัน | backlog โตประมาณ 150/s ในช่วง peak; ไม่ทราบระยะเวลา peak จึงยังประเมินพื้นที่จัดเก็บและเวลาระบาย backlog ไม่ได้ (scenario: “Quality goals and constraints”; ADR: “Decision”). |
| **Trade-off point** | จำกัด MailFast รวม retry ที่ 50/s ลดความเสี่ยงถูก throttle แต่เพิ่มเวลารออีเมลและ backlog | ADR ยอมรับความล่าช้าแต่ยังไม่มีเกณฑ์ยอมรับ, retention หรือ recovery capacity; การ buffer อย่างเดียวไม่รับประกันว่าจะระบายงานทัน (ADR: “Decision”, “Open questions”). |
| **Risk** | ข้อกำหนดคะแนนภายใน 1 นาทีไม่มีเป้าหมาย throughput, concurrency หรือพฤติกรรมเมื่อ `loyaltyservice` หยุดเกินหนึ่งนาที | idempotency ป้องกันคะแนนซ้ำ แต่ไม่รับประกัน deadline; ต้องพิสูจน์ทั้ง 200 orders/s และเวลาฟื้นตัว/ประมวลผล (scenario: “Quality goals and constraints”; ADR: “Validation”). |
| **Risk** | `paymentservice` และ `shippingservice` อาจทำงานสำเร็จก่อน order/outbox persistence ล้มเหลว; ADR ยังไม่กำหนด state transition หรือ compensation/reconciliation | เกิดผลข้างเคียงกับคำสั่งซื้อที่ไม่สามารถบันทึก/เผยแพร่ event ได้ และกระทบความถูกต้องของ checkout (scenario: “How it works today”; ADR: “Risks and mitigations”). |

## Missing or weak

- เปรียบเทียบเฉพาะ synchronous calls, RabbitMQ และ in-process worker; ยังไม่มีตัวเลือก durable แบบอื่นหรือเหตุผลเชิง trade-off ที่ทำให้ RabbitMQ เหนือตัวเลือกเหล่านั้น แม้ทีมมีผู้มีประสบการณ์ RabbitMQ 2 คนและต้องการ self-host/open-source (scenario: “Quality goals and constraints”; ADR: “Considered options”).
- กำหนดตัวเลขและเกณฑ์ปฏิบัติการสำหรับเวลาส่งอีเมลที่ยอมรับได้, peak duration, queue/outbox capacity, retention, backlog alert และอัตราระบายงานหลัง peak ให้ชัดเจน
- ระบุวิธีพิสูจน์ SLA คะแนนภายใน 1 นาที รวมโหลด 200 orders/s, retry/backoff, การหยุดชะงัก และกรณี outage ยาวกว่า deadline
- คำว่า “exactly once” ควรอธิบายว่าเป็นผลของการให้คะแนนที่เกิดครั้งเดียวด้วย durable deduplication แบบ atomic ภายใต้ at-least-once delivery ไม่ใช่การรับประกันการส่งข้อความครั้งเดียว
- ข้อความใน ADR ที่ให้ยืนยันประสบการณ์ RabbitMQ กับ source scenario อ่อน/คลาดเคลื่อน: scenario ระบุชัดว่าทีม 2 คนเคยใช้ RabbitMQ (scenario: “Quality goals and constraints”; ADR: “Decision drivers”).

## Verdict

**Accept with changes.** ก่อนอนุมัติสำหรับ implementation:

1. ยืนยันและระบุ transaction boundary ที่ทำให้ order กับ outbox อยู่ใน durable commit เดียว พร้อมกำหนด recovery เมื่อ payment/shipping สำเร็จแต่ persistence ล้มเหลว
2. กำหนด capacity และ SLO ของ backlog/อีเมล รวม peak duration, retention, recovery rate และ alert thresholds โดยคำนึงว่า backlog โตได้ 150/s
3. ระบุและทดสอบเกณฑ์ loyalty ภายใน 1 นาทีที่ 200 orders/s รวม outage, retry และ concurrent duplicate delivery
