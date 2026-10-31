Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`). Run a (zero-shot, context: none)

# ADR-0002: Decouple post-order actions from checkout using asynchronous events

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ ระบบ Online Boutique ต้องทำงานเพิ่ม 3 อย่างตามความต้องการ R-12
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** ซึ่งเป็นผู้ให้บริการอีเมลภายนอกรายใหม่ แทน logic จำลองเดิมใน `emailservice`
2. ให้ **loyalty points** ผ่าน `loyaltyservice` ที่จะสร้างใหม่
3. บันทึก **analytics event "order placed"** ให้ทีม marketing

ทีม marketing คาดว่าปีหน้าจะมี post-order action เพิ่มอีก เช่น SMS, fraud scoring และการอัปเดต recommendations

ปัญหาคือจะเชื่อม action เหล่านี้เข้ากับขั้นตอนสั่งซื้ออย่างไร โดยไม่ทำให้การสั่งซื้อช้าลงหรือล้มเหลวเพราะระบบปลายทาง และเพิ่ม action ใหม่ได้ง่าย

ข้อมูลที่ไม่ได้ให้มา ได้แก่ โครงสร้างปัจจุบันของ checkout, latency budget, ปริมาณ order, SLA ของ MailFast, ข้อกำหนดด้านความถูกต้องของ loyalty points และทักษะของทีม ทั้งหมดนี้อยู่ใน **Open questions**

## Decision drivers

1. **Availability:** [Assumption] การสั่งซื้อไม่ควรล้มเหลวเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** [Assumption] การเรียกผู้ให้บริการภายนอก (MailFast) ไม่ควรเพิ่ม latency ของการสั่งซื้อ
3. **Modifiability / extensibility:** ต้องเพิ่ม post-order action ใหม่ (SMS, fraud scoring, …) ได้โดยไม่ต้องแก้ checkout ทุกครั้ง (มาจาก R-12)
4. **Correctness:** [Assumption] loyalty points ไม่ควรถูกให้ซ้ำหรือหายไป
5. **Operability / cost:** ควรใช้ component ที่ทีมดูแลได้

## Considered options

1. **Synchronous calls:** service ที่ทำหน้าที่ checkout เรียก MailFast, `loyaltyservice` และ analytics โดยตรงตามลำดับ ภายใน request สั่งซื้อ
2. **Asynchronous in-process:** checkout ตอบลูกค้าก่อน แล้วยิง call ไปยังแต่ละ action ใน background thread/goroutine (fire-and-forget)
3. **Event-driven ผ่าน message broker:** checkout publish event `OrderPlaced` ไปยัง broker แล้ว consumer แต่ละตัว (email/MailFast, loyalty, analytics) subscribe แยกกัน

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous | เข้าใจง่าย ไม่มี component ใหม่ รู้ผลทันที | latency ของ order เท่ากับผลรวมของทุก call; ถ้า dependency ล้มหรือช้า order ก็ได้รับผลกระทบ; ทุก action ใหม่ต้องแก้ checkout |
| 2. Async in-process | latency ต่ำ ไม่มี infrastructure ใหม่ | งานหายถ้า pod restart ระหว่างทำ; retry และ rate limiting ต้องเขียนเอง; checkout ยังต้องรู้จักทุก action (coupling) |
| 3. Message broker | แยก checkout ออกจาก action ปลายทาง; เพิ่ม consumer ใหม่ได้โดยไม่แก้ checkout; มี retry/buffer เมื่อปลายทางล่ม | มี component ใหม่ที่ต้อง operate; eventual consistency; ต้องจัดการ duplicate delivery และ ordering; debug ยากขึ้น |

## Decision

เลือก **Option 3: event-driven ผ่าน message broker** checkout publish event `OrderPlaced` หนึ่งครั้งหลังสั่งซื้อสำเร็จ แล้วให้ consumer อิสระ 3 ตัวทำงานต่อ
- **email consumer:** เรียก MailFast (มี retry และ backoff)
- **`loyaltyservice`:** ให้คะแนน แบบ idempotent โดยใช้ `order_id` เป็น key
- **analytics consumer:** บันทึก event

เหตุผล: เป็นทางเลือกเดียวที่ตอบ driver 1–3 ได้พร้อมกัน โดยเฉพาะการเพิ่ม action ในปีหน้า ส่วนชนิดของ broker (เช่น RabbitMQ, Kafka, NATS) ยังไม่เลือก เพราะไม่รู้ข้อจำกัดด้านการ operate และทักษะทีม (ดู Open questions)

## Consequences

- **Positive:** order ไม่ต้องรอ MailFast/loyalty/analytics (performance)
- **Positive:** ระบบปลายทางล่มไม่ทำให้การสั่งซื้อล้ม เพราะ broker buffer event ไว้ (availability)
- **Positive:** action ใหม่คือ consumer ใหม่ ไม่ต้องแก้ checkout (modifiability)
- **Negative:** broker เป็น component ใหม่ที่ต้อง deploy, monitor และ backup (operability, cost)
- **Negative:** อีเมลและคะแนนมาถึงช้าลงเล็กน้อย (eventual consistency) และ trace ข้าม service ยากขึ้น (observability)
- **Risks and mitigations:**
  - ถ้า publish event ล้มหลังจากรับเงินแล้ว event จะหาย → ใช้ transactional outbox หรือ retry ฝั่ง publisher
  - broker ส่ง message ซ้ำได้ (at-least-once) → consumer ต้อง idempotent โดยเฉพาะ loyalty
  - broker เองล่ม → รัน broker แบบ replicated

## Validation

- Load test checkout ทั้งก่อนและหลังเปลี่ยน แล้วเทียบ latency ของการสั่งซื้อ (ยังไม่มีเป้าตัวเลข ดู Open questions)
- Chaos test: ปิด MailFast/`loyaltyservice` แล้วตรวจว่ายังสั่งซื้อได้ และ event ถูกประมวลผลครบเมื่อระบบกลับมา
- ตรวจว่าไม่มี order ใดได้ loyalty points ซ้ำ
- ทบทวน ADR อีกครั้งเมื่อจะเพิ่ม post-order action ตัวที่ 4

## Open questions

- ตอนนี้ checkout ทำงานอย่างไร เรียก email แบบ synchronous หรือไม่ และถ้า email ล้ม order ล้มด้วยหรือไม่
- latency budget ของการสั่งซื้อ (เช่น p95) และ latency ปัจจุบันคือเท่าไร
- peak load (orders/s) เป็นเท่าไร
- latency, อัตราความล้มเหลว และ rate limit ของ MailFast เป็นอย่างไร
- loyalty points ต้องให้ภายในเวลาเท่าไร และต้อง exactly-once หรือไม่
- ทีมมีกี่คน ถนัดภาษาและเทคโนโลยีอะไร เคยใช้ message broker ตัวไหน
- โครงสร้างพื้นฐานเป็นแบบใด (Kubernetes? cloud?) และใช้ managed service ได้หรือไม่
- checkout มี database สำหรับทำ transactional outbox หรือไม่
- ใครเป็น deciders
