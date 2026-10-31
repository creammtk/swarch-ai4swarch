# ADR-0002: Publish an OrderPlaced event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-31
- **Deciders:** creammtk (architect); product owner และ SRE ต้องยืนยัน (ดู Open questions)

## Context and problem

R-12 (`docs/scenario.md`) กำหนดให้ทำ 3 อย่างหลังจากสั่งซื้อ: ส่งอีเมลยืนยันผ่าน **MailFast** (แทน logic จำลองของ `emailservice`), ให้ **loyalty points** ผ่าน `loyaltyservice` ใหม่ และบันทึก **analytics event "order placed"** ปีหน้าจะมี SMS, fraud scoring และการอัปเดต recommendations เพิ่มเข้ามาอีก

**สถานะปัจจุบัน** (`system/online-boutique/src/checkoutservice/main.go`)

- `PlaceOrder` เรียก `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` ตามลำดับ แบบ synchronous gRPC ภายใน request เดียว
- ถ้าส่งอีเมลล้ม ระบบแค่ `log.Warnf` แล้ว order ยังสำเร็จ ส่วนผลของ `emptyUserCart` ถูกทิ้งไป
- dependency ทุกตัวผูกไว้ใน struct `checkoutService` และต้องมี env ครบผ่าน `mustMapEnv` การเพิ่ม action ใหม่จึงต้องแก้และ deploy checkout ใหม่ทุกครั้ง
- `checkoutservice` ไม่มี database ของตัวเอง และไม่มีที่เก็บ order แบบถาวร

**ข้อเท็จจริงที่บีบการตัดสินใจ** (`docs/scenario.md`)

| Topic | Fact | ผลต่อการออกแบบ |
|---|---|---|
| Checkout latency | p95 ≤ **500 ms** (ปัจจุบันประมาณ 300 ms) | เหลือ budget ประมาณ 200 ms แต่ MailFast p95 = **1.5 s** จึงเรียก MailFast แบบ synchronous ไม่ได้ |
| Peak load | **200 orders/s** (11.11) | MailFast รับได้ **50/s** → backlog เพิ่มประมาณ **150 อีเมล/วินาที** ระหว่าง peak |
| MailFast | p99 8 s, fail ประมาณ 0.5 % | ต้องมี timeout, retry และ rate limiting ฝั่งเรา |
| Loyalty | ภายใน **1 นาที**, **exactly once** | ต้องมี delivery ที่รับประกัน และ consumer ที่ idempotent |
| Availability | order **ต้องไม่ล้ม** เพราะ email/loyalty/analytics | action ทั้งหมดต้องอยู่นอก critical path |
| Team | 4 คน, เก่ง Go/gRPC, **2 คนเคยใช้ RabbitMQ** | broker ที่ทีมคุ้นเคยลดความเสี่ยงด้าน operation |
| Operations | Kubernetes, ชอบ **open-source self-hosted** | ตัด managed cloud queue ออก |

**ความขัดแย้งหลักที่ต้องตัดสิน:** "order ต้องไม่ล้ม" กับ "loyalty ต้อง exactly once" ดึงกันอยู่ ถ้า order สำเร็จแต่ event หาย ลูกค้าจะไม่ได้คะแนน ADR นี้จึงต้องระบุชัดว่าเมื่อ broker ล่มจะเกิดอะไรขึ้น

## Decision drivers

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s
3. **Reliability / correctness:** loyalty points ได้ครั้งเดียวต่อ order ภายใน 1 นาที และ event ต้องไม่หายแบบเงียบ ๆ
4. **Modifiability:** เพิ่ม post-order action ปีหน้าได้โดยไม่แก้ `checkoutservice`
5. **Operability / team fit:** self-hosted บน Kubernetes และทีม 4 คนดูแลได้
6. **Interoperability กับข้อจำกัดภายนอก:** ไม่ส่งเกิน 50 emails/s ไปยัง MailFast

## Considered options

1. **Extend synchronous gRPC:** เพิ่มการเรียก MailFast, `loyaltyservice` และ analytics ต่อจาก `sendOrderConfirmation` ใน `PlaceOrder`
2. **Goroutine fire-and-forget:** ใน `PlaceOrder` สั่ง `go` เรียกแต่ละ action แล้วตอบลูกค้าทันที
3. **RabbitMQ (self-hosted) + direct publish:** `checkoutservice` publish `OrderPlaced` ไปยัง topic exchange แล้วแต่ละ consumer มี queue ของตัวเอง
4. **Apache Kafka (self-hosted):** `checkoutservice` produce `OrderPlaced` ไปยัง topic แล้วแต่ละ consumer group อ่านเอง
5. **RabbitMQ + transactional outbox:** เพิ่ม database ให้ checkout เขียน order และ event ใน transaction เดียว แล้วมี relay ส่ง event ไป RabbitMQ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC | เปลี่ยนน้อยที่สุด ทีมถนัด gRPC | MailFast p95 1.5 s ทำให้ทะลุ 500 ms; ชน rate limit 50/s ตอน peak; loyalty ล่มแล้ว order ช้าหรือล้ม; action ใหม่ต้องแก้ `main.go` ทุกครั้ง |
| 2. Goroutine | latency ไม่เพิ่ม ไม่มี infra ใหม่ | งานหายเมื่อ pod restart/OOM จึงรับประกัน loyalty ไม่ได้; ไม่มี buffer รับ backlog 150/s; retry, rate limit, dedup ต้องเขียนเอง; coupling เท่าเดิม |
| 3. RabbitMQ direct publish | action ออกจาก critical path; durable queue เก็บ backlog ช่วง peak; consumer ใหม่ = queue ใหม่; มี publisher confirms, DLQ, prefetch; 2 คนในทีมเคยใช้; open-source บน K8s | stateful component ใหม่ (quorum queue ใช้ 3 nodes); at-least-once ต้องทำ idempotency เอง; ถ้า broker ล่มตอน publish ยังมีช่องให้ event หาย |
| 4. Kafka | throughput สูงมาก; replay event ได้ (ดีกับ analytics/recommendations) | ไม่มีใครในทีมเคยใช้; 200 events/s ต่ำเกินกว่าที่ Kafka จะคุ้ม; rate limit ต่อ consumer ต้องทำเอง |
| 5. RabbitMQ + outbox | ปิดช่อง event หายได้จริง (order กับ event commit พร้อมกัน) | ต้องเพิ่ม DB และ relay ให้ checkout ซึ่งตอนนี้ stateless; งานและ operation เพิ่มมากสำหรับทีม 4 คน; DB เข้ามาอยู่ใน critical path ของ checkout |

## Decision

เลือก **Option 3: RabbitMQ (self-hosted บน Kubernetes) แบบ direct publish** และกำหนด fallback ชัดเจนเมื่อ publish ล้ม ส่วน Option 5 (outbox) เก็บไว้เป็นขั้นต่อไปถ้าตัวชี้วัดใน Validation แสดงว่า event หายเกินที่ยอมรับได้

**ฝั่ง `checkoutservice`**

- หลัง `chargeCard` และ `shipOrder` สำเร็จ publish `OrderPlaced` (`order_id`, `user_id`, `email`, รายการสินค้า, ยอดเงิน) ไปยัง exchange `orders` แบบ persistent พร้อม **publisher confirms** โดยใช้ `order_id` เป็น message id
- รอ confirm ไม่เกิน **100 ms** [Assumption: ต้องวัดจริง ใช้ budget ที่เหลือประมาณ 200 ms ได้ไม่ถึงครึ่ง] แล้วตอบลูกค้า โดยไม่รอ consumer ใด ๆ
- **ถ้า publish ล้มหรือหมดเวลา:** order **ยังสำเร็จ** (driver 1 สำคัญที่สุด) แล้วเขียน event ทั้งก้อนเป็น structured log ระดับ error (มี `order_id`) และเพิ่ม metric `order_event_publish_failed_total` มี replay job อ่าน log นี้แล้ว publish ซ้ำ ซึ่งปลอดภัยเพราะทุก consumer dedup ด้วย `order_id`
- ลบการเรียก `sendOrderConfirmation` แบบ synchronous ออกจาก `PlaceOrder` และลบ dependency ของ `emailservice` ออกจาก `checkoutService`

**Consumers** (queue แยกกัน backlog ของตัวหนึ่งจึงไม่ทำให้อีกตัวช้า)

| Consumer | พฤติกรรม |
|---|---|
| `emailservice` (เขียนใหม่ให้เรียก MailFast) | token bucket รวมทุก replica **≤ 50/s**; timeout ต่อ call; retry แบบ exponential backoff; ส่ง DLQ เมื่อ retry ครบ; จำ `order_id` ที่ส่งสำเร็จแล้วเพื่อลดอีเมลซ้ำ |
| `loyaltyservice` (Go) | บันทึกคะแนนกับ `order_id` ใน transaction เดียว (unique constraint) แล้วจึง ack จึงเป็น **effectively exactly-once**; ไม่มี rate limit ภายนอก จึงตามทันภายใน 1 นาทีแม้ช่วง peak |
| analytics consumer | บันทึก event "order placed" ให้ marketing; ack หลังเขียนสำเร็จ |

**เหตุผล:** Option 1 ละเมิด driver 1, 2, 6 และ Option 2 ละเมิด driver 3 ส่วน Option 3, 4, 5 ตอบ driver 1–4 ได้ แต่ Option 3 เหมาะกับปริมาณ 200/s และทีมที่มีประสบการณ์อยู่แล้ว (driver 5) และ prefetch กับ queue แยกช่วยทำ rate limit ให้ MailFast ได้ง่าย (driver 6) Option 5 ปิดความเสี่ยงได้ดีกว่า แต่ต้องเพิ่ม DB เข้า critical path ของ checkout ซึ่งเป็นต้นทุนที่ยังไม่มีหลักฐานว่าจำเป็น จึงยอมรับความเสี่ยงที่เหลือแบบมี fallback และมีตัวชี้วัดแทน

## Consequences

- **Positive:** `PlaceOrder` ไม่เรียกระบบภายนอกที่ช้าอีกต่อไป เพิ่มแค่การ publish ใน cluster ซึ่งคาดว่าใช้ไม่กี่ ms [Inferred] จึงอยู่ใน budget 500 ms (quality attribute: performance)
- **Positive:** MailFast, loyalty หรือ analytics ล่มก็ไม่กระทบ order เพราะ event รออยู่ใน queue (quality attribute: availability)
- **Positive:** SMS, fraud scoring และ recommendations เพิ่มได้ด้วยการ bind queue ใหม่เข้ากับ exchange `orders` ไม่ต้องแก้ checkout (quality attribute: modifiability)
- **Positive:** queue เป็น buffer ช่วง 11.11 ส่งไป MailFast ไม่เกิน 50/s (quality attribute: interoperability)
- **Negative:** อีเมลช่วง peak จะมาช้า backlog เพิ่มประมาณ 150/s เช่น peak 10 นาที [Assumption] → ประมาณ 90,000 อีเมลค้าง ซึ่งใช้เวลาอย่างน้อย 30 นาทีกว่าจะส่งหมดที่ 50/s (ช้ากว่านี้ถ้ายังมี order เข้ามา) (quality attribute: performance ของการแจ้งเตือน, usability)
- **Negative:** RabbitMQ cluster 3 nodes เป็น stateful component ใหม่ ที่ทีมต้องดูแล upgrade, monitoring, storage และ on-call (quality attribute: operability, cost)
- **Negative:** ระบบเป็น eventual consistency ลูกค้าอาจเห็นหน้า "สั่งซื้อสำเร็จ" ก่อนคะแนนเข้า (สูงสุด 1 นาทีตามเป้า) (quality attribute: usability)
- **Negative:** ช่องที่ event อาจหายยังเหลืออยู่ เช่น pod ของ checkout crash ระหว่าง publish ล้มกับการเขียน log (quality attribute: reliability)
- **Risks and mitigations:**
  - *Event หายเมื่อ publish ล้ม:* quorum queue 3 replicas, publisher confirms, structured fallback log + replay job, alert เมื่อ `order_event_publish_failed_total` > 0; ถ้าเกินเกณฑ์ใน Validation ให้ทำ Option 5 เป็น ADR ใหม่
  - *Loyalty ได้คะแนนซ้ำจาก redelivery หรือ replay:* unique constraint บน `order_id`
  - *Loyalty เกิน 1 นาที:* monitor queue depth และ consumer lag ของ `loyalty` queue, alert ที่ 30 s, scale consumer ด้วย HPA
  - *MailFast fail 0.5 % หรือ p99 8 s:* timeout + retry + DLQ, alert ที่ DLQ depth
  - *Poison message:* DLQ แยกต่อ queue, version schema ของ `OrderPlaced` (เพิ่ม field ได้ ห้ามลบ)
  - *Queue ของอีเมลโตจนกิน disk ของ broker:* ตั้ง alert ที่ queue length และคำนวณ disk จาก backlog สูงสุดที่คาดไว้

## Validation

- Load test checkout ที่ **200 orders/s** (เช่นด้วย `loadgenerator`) ตรวจว่า p95 ของ `PlaceOrder` ≤ **500 ms**
- ระหว่าง load test วัดว่า p99 ของเวลาจาก order ถึงการให้ loyalty points ≤ **60 s** และทุก `order_id` มีคะแนนเท่ากับ 1 รายการพอดี
- Chaos test 1: ปิด `loyaltyservice`, `emailservice`/MailFast stub และ analytics ทีละตัว ตรวจว่า order สำเร็จ 100 % และ backlog drain หมดหลังระบบกลับมา
- Chaos test 2: ปิด RabbitMQ 1 node แล้วปิดทั้ง cluster ระหว่าง load test ตรวจว่า order ยังสำเร็จ, p95 ยังไม่เกิน 500 ms และ event ที่ publish ไม่สำเร็จถูก replay ครบ
- ตรวจว่า rate ที่ส่งไป MailFast ≤ 50/s ตลอดการทดสอบ
- เกณฑ์ไปทำ outbox: ถ้าใน production มี event ที่ replay ไม่ได้ > 0 ต่อเดือน [Assumption: product owner ต้องยืนยันเกณฑ์]
- ทบทวน ADR หลัง 11.11 ครั้งแรก หรือเมื่อเพิ่ม consumer ตัวที่ 4

## Open questions

- peak 200 orders/s ยาวนานแค่ไหน และธุรกิจยอมให้อีเมลยืนยันช้าได้ถึงเท่าไร
- MailFast เพิ่ม rate limit ได้ไหม (เช่น tier สูงขึ้นช่วง 11.11) และรองรับ idempotency key หรือไม่
- ธุรกิจยอมรับ order ที่ไม่ได้ loyalty points (เพราะ event หาย) ได้กี่รายการ ถ้าเป็นศูนย์ ต้องทำ Option 5 ตั้งแต่แรก
- `loyaltyservice` จะใช้ data store อะไร
- analytics event ไปเก็บที่ไหน และ marketing ต้องการ replay ย้อนหลังหรือไม่ (ถ้าต้องการ อาจพิจารณา RabbitMQ Streams หรือ Kafka)
- SRE มีคนพร้อม on-call ดูแล RabbitMQ cluster หรือไม่
