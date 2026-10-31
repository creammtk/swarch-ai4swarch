Reviewer: Claude Code (model: Claude Opus 5.5, `claude-opus-5-5`)

# Architecture review (ATAM-style): ADR-0002 Publish `OrderPlaced` to RabbitMQ

ADR ที่รีวิว: `docs/adr/0002-publish-order-events-to-rabbitmq.md` · ข้อเท็จจริงอ้างอิง: `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | ลูกค้าช่วง 11.11 | `PlaceOrder` **200 ครั้ง/วินาที** | peak, RabbitMQ ทำงานแต่ confirm ช้า | ตอบลูกค้าหลังได้ confirm หรือหมดเวลา 100 ms โดยไม่รอ consumer | p95 ของ `PlaceOrder` ≤ **500 ms** |
| S2 Availability + reliability | RabbitMQ ทั้ง cluster | ล่ม 10 นาที [Assumption] | peak 200 orders/s | order ยังสำเร็จ, event ที่ publish ไม่ได้ถูก replay หลังระบบกลับมา | order สำเร็จ **100 %**, event หาย **0** รายการ |
| S3 Correctness | broker redelivery หรือ replay job | `OrderPlaced` เดิมถูกส่งซ้ำ 2 ครั้ง | `loyaltyservice` scale out ด้วย HPA | dedup ด้วย unique constraint บน `order_id` | คะแนน **1 รายการพอดี** ต่อ order ภายใน **60 s** |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | fallback เมื่อ publish ล้มคือ "เขียน log แล้วมี replay job อ่าน log" แต่ log เป็นช่องทางที่ไม่รับประกันการส่ง (log pipeline, retention และ format ไม่ถูกระบุ) | S2 ล้มเหลว: event หายเงียบ ๆ = ลูกค้าไม่ได้คะแนน ซึ่งขัดกับ "exactly once" (หายไป = zero ครั้ง) |
| Risk | `checkoutservice` ไม่มีที่เก็บ order แบบถาวร (ADR บรรทัด 16 ยอมรับเอง) จึง **ไม่มี source of truth** ไว้ reconcile ว่า order ไหนยังไม่ได้คะแนน | ตัวชี้วัด "event ที่ replay ไม่ได้ > 0 ต่อเดือน" วัดใน production ไม่ได้ จึงไม่มีทางรู้ว่าถึงเกณฑ์ไปทำ outbox แล้วหรือยัง |
| Risk | `emailservice` ทำ token bucket "รวมทุก replica ≤ 50/s" แต่ไม่บอกว่าใช้ shared state อะไร และอ้างว่า prefetch ช่วย rate limit ซึ่ง prefetch จำกัดแค่ in-flight ไม่ใช่ rate | ตาม Little's law ที่ 50/s × p95 1.5 s ต้องมีประมาณ 75 call ค้างพร้อมกัน (p99 8 s ยิ่งมากกว่า) ถ้าตั้ง prefetch ผิด อาจได้ throughput ต่ำกว่า 50/s หรือเกิน rate limit จน MailFast ปฏิเสธ |
| Risk | HPA scale `loyaltyservice` ตาม queue depth ต้องใช้ external metrics และใช้เวลา scale-up หลายสิบวินาที | เป้า 60 s เหลือ margin น้อยมากหลัง alert ที่ 30 s |
| Sensitivity point | timeout รอ publisher confirm (**100 ms**) บน quorum queue (ต้อง replicate ก่อน confirm) | สั้นไป → fallback บ่อย (reliability แย่ลง); ยาวไป → p95 เพิ่มขึ้นตรง ๆ ทุก request (S1) |
| Sensitivity point | ระยะเวลา peak (ADR สมมติ 10 นาที) | backlog อีเมล = 150/s × ระยะเวลา; ทั้งเวลาที่ลูกค้ารออีเมลและ disk ของ broker แปรตามค่านี้ |
| Trade-off point | direct publish (Option 3) แทน outbox (Option 5) | ได้ availability และ latency ของ checkout (ไม่มี DB ใน critical path) แลกกับ reliability ของ event (ช่อง crash ระหว่าง publish ล้มกับเขียน log) |
| Trade-off point | order สำเร็จเสมอแม้ publish ล้ม | availability ดีขึ้น แต่ correctness ของ loyalty ลดลง ADR เลือกฝั่ง availability อย่างชัดเจน ซึ่งดี แต่ต้องให้ product owner ยืนยัน |
| Non-risk | `loyaltyservice` บันทึกคะแนนพร้อม unique constraint บน `order_id` ใน transaction เดียวแล้วค่อย ack | at-least-once + idempotent consumer = effectively exactly-once ที่ถูกต้อง รองรับทั้ง redelivery และ replay (S3) |
| Non-risk | queue แยกต่อ consumer + DLQ ต่อ queue | backlog อีเมล 90,000 ฉบับไม่ทำให้ loyalty ช้า; poison message ไม่บล็อก queue อื่น |
| Non-risk | ตัด Kafka ด้วยเหตุผล team fit และปริมาณ 200 events/s | สอดคล้องกับ scenario (2 คนเคยใช้ RabbitMQ, ไม่มีใครเคยใช้ Kafka) |

## 3. Missing or weak

- **Option ที่ไม่ได้พิจารณา:** (a) fallback แบบ durable ที่เบากว่า outbox เช่น เขียน event ที่ publish ไม่ได้ลง PersistentVolume หรือ local spool แล้ว retry เอง แทนการพึ่ง log; (b) ให้ checkout มี order store (ซึ่งยังไม่มี) เพื่อใช้ทั้ง reconcile และ outbox ในขั้นต่อไป; (c) RabbitMQ Streams สำหรับ analytics/recommendations ที่ต้อง replay (ถูกพูดถึงแค่ใน Open questions)
- **Consequence ที่ไม่ได้ระบุ:** `OrderPlaced` และ fallback log มี `email` ของลูกค้า แต่ไม่พูดถึง security/privacy (PII ใน log, สิทธิ์อ่าน queue); อีเมลซ้ำได้ถ้า consumer crash หลังส่ง MailFast แต่ก่อน ack และไม่บอกว่า "จำ `order_id`" เก็บไว้ที่ไหน
- **Claim ที่อ่อนหรือน่าจะผิด:** "budget เหลือประมาณ 200 ms" ไม่ได้นับว่าการลบ `sendOrderConfirmation` แบบ synchronous จะลด latency ปัจจุบัน 300 ms ลง (budget จริงน่าจะมากกว่านี้ [Inferred]); "publish ใช้ไม่กี่ ms" ยังไม่มีการวัด และไม่กล่าวถึงกรณี broker ล่มทั้ง cluster ที่การต่อ connection ใหม่อาจค้างนานกว่า 100 ms ถ้าไม่มี circuit breaker
- **Validation:** Chaos test 2 ไม่ได้ทดสอบกรณี checkout pod crash ระหว่าง publish ล้มกับการเขียน log ซึ่งเป็นช่องที่ ADR ยอมรับว่ามีอยู่

## 4. Verdict

**Accept with changes.** ทางเลือกหลัก (RabbitMQ + queue แยก + idempotent consumer) เหมาะกับ scenario และ trade-off ถูกเขียนไว้ตรงไปตรงมา การเปลี่ยนแปลงที่สำคัญที่สุด 3 ข้อ:

1. **เปลี่ยน fallback จาก log เป็นที่เก็บแบบ durable** (local spool บน PV หรือ table เล็ก ๆ) พร้อม retry อัตโนมัติ และระบุ circuit breaker เมื่อ broker ล่ม เพื่อให้ S2 ผ่านโดยไม่ต้องพึ่ง log pipeline
2. **เพิ่มกลไก reconcile** (เช่น บันทึก `order_id` ที่ charge สำเร็จไว้ที่ใดที่หนึ่ง) เพื่อให้เกณฑ์ "event หาย > 0 ต่อเดือน → ทำ outbox" วัดได้จริง และให้ product owner ตอบ Open question เรื่องจำนวน order ที่ยอมให้ไม่ได้คะแนน **ก่อน** อนุมัติ
3. **ออกแบบ rate limit ของ MailFast ให้ชัด:** กลไก token bucket แบบ distributed (หรือจำกัด consumer เป็น 1 ตัวที่ concurrency ประมาณ 75–400), ที่เก็บ dedup ของอีเมล และการจัดการ PII ใน event และ log
