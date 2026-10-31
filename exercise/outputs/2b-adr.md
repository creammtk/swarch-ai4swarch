Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`). Run b (context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`)

# ADR-0002: Publish an OrderPlaced event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

R-12 (`docs/scenario.md` L7–14) กำหนดให้ทำ 3 อย่างหลังจากสั่งซื้อ: ส่งอีเมลยืนยันผ่าน **MailFast** (แทน logic จำลองของ `emailservice`), ให้ **loyalty points** ผ่าน `loyaltyservice` ใหม่ และบันทึก **analytics event "order placed"** นอกจากนี้ปีหน้าจะมี SMS, fraud scoring และการอัปเดต recommendations เพิ่มเข้ามา

**สถานะปัจจุบัน** (`system/online-boutique/src/checkoutservice/main.go`)
- `PlaceOrder` (L230–280) เรียก `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` ตามลำดับ แบบ synchronous gRPC ภายใน request เดียว
- ถ้าส่งอีเมลล้ม ระบบแค่ `log.Warnf` แล้ว order ยังสำเร็จ (L273–277) ส่วนผลของ `emptyUserCart` ถูกทิ้งไป (L263)
- dependency ทุกตัวถูกผูกไว้ใน struct `checkoutService` (L66–86) และต้องมี env ครบผ่าน `mustMapEnv` (L111–116) การเพิ่ม action จึงต้องแก้และ deploy checkout ใหม่
- `checkoutservice` ไม่มี database ของตัวเอง ไม่มีการเชื่อม data store ใน `main.go`

**ข้อเท็จจริงที่บีบการตัดสินใจ** (`docs/scenario.md` L22–30)

| Topic | Fact | ผลต่อการออกแบบ |
|---|---|---|
| Checkout latency | p95 ≤ **500 ms** (ปัจจุบันประมาณ 300 ms) | เหลือ budget ประมาณ 200 ms แต่ MailFast p95 อยู่ที่ **1.5 s** เกิน budget เกือบ 8 เท่า **จึงเรียก MailFast แบบ synchronous ไม่ได้** |
| Peak load | **200 orders/s** (11.11) | อีเมลเข้ามา 200/s แต่ MailFast รับได้ **50/s** → backlog เพิ่มขึ้นประมาณ **150 อีเมล/วินาที** ระหว่าง peak |
| MailFast | p99 8 s, fail ประมาณ 0.5 % | ต้องมี retry, timeout และ rate limiting |
| Loyalty | ภายใน **1 นาที**, **exactly once** | ต้องมี delivery ที่รับประกัน และ consumer ที่ idempotent |
| Availability | order **ต้องไม่ล้ม** เพราะ email/loyalty/analytics | action ทั้งหมดต้องอยู่นอก critical path |
| Team | 4 คน, เก่ง Go/gRPC, **2 คนเคยใช้ RabbitMQ** | broker ที่ทีมคุ้นเคยลดความเสี่ยง |
| Operations | Kubernetes, ชอบ **open-source self-hosted** | ตัด managed cloud queue ออก |

## Decision drivers

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s
3. **Correctness:** loyalty points exactly-once ภายใน 1 นาที
4. **Modifiability:** เพิ่ม post-order action ปีหน้าได้โดยไม่แก้ `checkoutservice`
5. **Operability / team fit:** self-hosted บน Kubernetes และทีม 4 คนดูแลได้
6. **Respect external limits:** ไม่ส่งเกิน 50 emails/s ไปยัง MailFast

## Considered options

1. **Extend synchronous gRPC:** เพิ่มการเรียก MailFast, `loyaltyservice` และ analytics ต่อจาก `sendOrderConfirmation` ใน `PlaceOrder`
2. **Goroutine fire-and-forget:** ใน `PlaceOrder` สั่ง `go` เรียกแต่ละ action หลังสร้าง `orderResult` แล้วตอบลูกค้าทันที
3. **RabbitMQ (self-hosted):** `checkoutservice` publish `OrderPlaced` ไปยัง topic exchange แล้วมี queue แยกสำหรับแต่ละ consumer
4. **Apache Kafka (self-hosted):** `checkoutservice` produce `OrderPlaced` ไปยัง topic แล้วแต่ละ consumer group อ่านเอง

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC | เปลี่ยนน้อยที่สุด ทีมถนัด gRPC อยู่แล้ว | MailFast p95 1.5 s ทำให้ p95 ทะลุ 500 ms; ที่ 200/s จะชน rate limit 50/s; loyalty ล่มแล้ว order ช้าหรือล้ม; ทุก action ใหม่ต้องแก้ `main.go` |
| 2. Goroutine | latency ไม่เพิ่ม ไม่มี infra ใหม่ | งานหายถ้า pod restart หรือ OOM จึงรับประกันการให้ loyalty ไม่ได้; ไม่มี buffer รองรับ backlog 150/s ของ MailFast; retry, rate limit และ dedup ต้องเขียนเอง; coupling เท่าเดิม |
| 3. RabbitMQ | ย้าย action ออกจาก critical path; durable queue เก็บ backlog ช่วง peak ได้; consumer ใหม่ = queue ใหม่ ไม่ต้องแก้ checkout; มี publisher confirms, DLQ และ prefetch ให้ทำ rate limit; 2 คนในทีมเคยใช้; open-source, self-hosted บน K8s ได้ | component stateful ใหม่ (quorum queue ควรใช้ 3 nodes); delivery เป็น at-least-once จึงต้องทำ idempotency เอง; eventual consistency |
| 4. Kafka | throughput สูงมาก; replay event ได้ (ดีกับ analytics และ recommendations ในอนาคต) | operate ยากกว่าสำหรับทีม 4 คนที่ไม่มีใครเคยใช้; 200 events/s ต่ำกว่าระดับที่คุ้มค่ากับ Kafka มาก; rate limit ต่อ consumer ต้องทำเอง |

## Decision

เลือก **Option 3: RabbitMQ (self-hosted บน Kubernetes)**

- หลัง `shipOrder` สำเร็จ `checkoutservice.PlaceOrder` จะ publish `OrderPlaced` (ข้อมูล `OrderResult`, `email`, `user_id`, โดยใช้ `order_id` เป็น message id) ไปยัง exchange `orders` แบบ persistent พร้อม **publisher confirms** แล้วจึงตอบลูกค้า โดยไม่รอ consumer ใด ๆ
- ถ้า publish ล้มหรือหมดเวลา (timeout สั้น เช่น 50 ms [Assumption]) order **ยังสำเร็จ** ตาม driver 1 แล้วบันทึก error ลง log พร้อม metric เพื่อนำไป replay (ดู Risks)
- consumer มี 3 ตัว แต่ละตัวมี queue ของตัวเอง
  - **`emailservice` (เขียนใหม่ให้เรียก MailFast):** จำกัด rate ไว้ที่ **≤ 50/s** รวมทุก replica, retry แบบ exponential backoff และส่งไป DLQ เมื่อ retry ครบ
  - **`loyaltyservice` (Go):** idempotent โดยเก็บ `order_id` ที่ประมวลผลแล้วไว้ใน store ของ loyalty และบันทึกคะแนนกับ `order_id` ใน transaction เดียวกัน (unique constraint) จึงเป็น effectively exactly-once แล้วจึง ack
  - **analytics consumer:** บันทึก event สำหรับ marketing
- ลบการเรียก `sendOrderConfirmation` แบบ synchronous (L273–277) ออกจาก `PlaceOrder`

**เหตุผล:** Option 1 ละเมิด driver 1, 2 และ 6 ส่วน Option 2 ละเมิด driver 3 เพราะงานหายได้ ส่วน RabbitMQ กับ Kafka ตอบ driver 1–4 ได้ทั้งคู่ แต่ RabbitMQ เหมาะกับปริมาณ 200/s และทีมที่มีประสบการณ์อยู่แล้ว (driver 5) และ prefetch/queue ช่วยทำ rate limit ให้ MailFast ได้ง่ายกว่า (driver 6)

## Consequences

- **Positive:** `PlaceOrder` ไม่มีการเรียกระบบภายนอกที่ช้าอีกต่อไป เพิ่มแค่ publish ที่ in-cluster เท่านั้น (คาดว่าไม่กี่ ms [Inferred]) จึงอยู่ใน budget 500 ms (performance)
- **Positive:** MailFast, loyalty หรือ analytics ล่มก็ไม่กระทบ order เพราะ event รออยู่ใน queue (availability)
- **Positive:** action ปีหน้า (SMS, fraud scoring, recommendations) เพิ่มได้ด้วยการ bind queue ใหม่เข้ากับ exchange `orders` ไม่ต้องแก้ checkout (modifiability)
- **Positive:** queue เป็น buffer ช่วง 11.11 ทำให้ส่งไป MailFast ไม่เกิน 50/s (interoperability กับ external limit)
- **Negative:** อีเมลช่วง peak จะช้า ระหว่าง peak backlog เพิ่มประมาณ 150/s (เช่น peak 10 นาที [Assumption] → ประมาณ 90,000 อีเมลค้าง) ถ้า traffic หลัง peak ต่ำมาก จะใช้เวลาประมาณ 30 นาทีกว่าจะส่งหมด (performance ของการแจ้งเตือน, user experience)
- **Negative:** RabbitMQ cluster เป็น stateful component ใหม่ที่ทีมต้องดูแล upgrade, monitor และ storage (operability, cost)
- **Negative:** checkout ไม่มี database จึงทำ transactional outbox ไม่ได้ทันที ถ้า RabbitMQ ล่มตอน publish อาจมี event หาย (reliability)
- **Risks and mitigations:**
  - *Event หายเมื่อ publish ล้ม:* ใช้ RabbitMQ quorum queue 3 replicas, publisher confirms และ log event ที่ publish ไม่สำเร็จแบบมีโครงสร้าง (มี `order_id`) เพื่อ replay; ระยะยาวพิจารณา outbox (ต้องเพิ่ม DB ให้ checkout แยกเป็น ADR)
  - *Loyalty ได้คะแนนซ้ำ (redelivery):* dedup ด้วย `order_id` ด้วย unique constraint
  - *Loyalty เกิน 1 นาที:* monitor queue depth และ consumer lag ของ `loyalty` queue และตั้ง alert ที่ 30 s; queue ของ loyalty แยกจาก email จึงไม่ติด backlog ของ MailFast
  - *MailFast fail 0.5 %:* retry + DLQ + alert ที่ DLQ depth
  - *Poison message:* DLQ แยกต่อ queue

## Validation

- Load test checkout ที่ **200 orders/s** (เช่นด้วย `loadgenerator`) แล้วตรวจว่า p95 ของ `PlaceOrder` ≤ **500 ms**
- ระหว่าง load test วัดว่า p99 ของเวลาตั้งแต่ order ถึงตอนให้ loyalty points ≤ **60 s** และนับว่าคะแนนต่อ `order_id` ต้องเท่ากับ 1 เสมอ (ต้องไม่มีคะแนนซ้ำ)
- Chaos test: ปิด `loyaltyservice`, `emailservice`/MailFast stub และ analytics ทีละตัว แล้วตรวจว่าอัตราการสั่งซื้อสำเร็จเป็น 100 % และ backlog ถูก drain จนหมดหลังระบบกลับมา
- ตรวจว่า rate ที่ส่งไป MailFast ≤ 50/s ตลอดการทดสอบ
- ทบทวน ADR หลัง 11.11 ครั้งแรก หรือเมื่อเพิ่ม consumer ตัวที่ 4

## Open questions

- peak 200 orders/s ยาวนานแค่ไหน และ traffic หลัง peak เป็นอย่างไร (ใช้คำนวณ backlog และเวลา drain ของอีเมล) ธุรกิจยอมให้อีเมลช้าได้ถึงเท่าไร
- MailFast เพิ่ม rate limit ได้ไหม (เช่น ซื้อ tier สูงขึ้นในช่วง 11.11) และรองรับ idempotency key หรือไม่ (เพื่อกันลูกค้าได้อีเมลซ้ำเมื่อ retry)
- ถ้า RabbitMQ ล่มตอน publish ธุรกิจยอมรับ event หายได้แค่ไหน หรือต้องเพิ่ม DB/outbox ให้ checkout
- `loyaltyservice` จะใช้ data store อะไร (ไม่มีหลักฐานในไฟล์ที่ให้มา)
- analytics event ไปเก็บที่ไหน และ marketing ต้องการ replay ย้อนหลังหรือไม่ (ถ้าต้องการ อาจต้องพิจารณา Kafka หรือ RabbitMQ Streams)
- ใครเป็น deciders
