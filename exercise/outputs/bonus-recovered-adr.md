Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`)

> **หมายเหตุเรื่องขอบเขตหลักฐาน:** งานนี้ทำใน session เดียวกับที่ agent เคยอ่านไฟล์อื่นของระบบมาก่อน (Kubernetes manifests และ source ของ service อื่น เพื่อตรวจผลงานที่ 1) จึงไม่ใช่ session ใหม่ตามที่ใบงานกำหนด ในไฟล์นี้อ้างอิงหลักฐานเฉพาะจากไฟล์ evidence สามไฟล์เท่านั้น สิ่งที่รู้จากไฟล์อื่นจะไม่นับเป็น Evidenced

# ADR-R001: ใช้ gRPC แบบ synchronous ผ่าน `demo.proto` ไฟล์เดียวสำหรับการสื่อสารระหว่าง services

- **Status:** Reconstructed
- **Date:** 2026-10-31 (วันที่กู้คืน ส่วนวันที่ตัดสินใจจริง: Not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files (license header ระบุเพียง "Copyright 2018/2020 Google LLC")

## Context and problem

- ระบบแบ่งเป็น 9 backend services ที่นิยามใน `demo.proto` ไฟล์เดียว (`package hipstershop`, L17) และมี `frontend` เป็น HTTP server สำหรับผู้ใช้ (`frontend/main.go` L171 `http.ListenAndServe`)
- ทุก RPC ใน `demo.proto` เป็น **unary** (request → response) และไม่มี `stream` เลย (L24–244)
- `frontend` ต้องเรียก 7 services (`frontend/main.go` L141–147) และ `checkoutservice` ต้องเรียก 6 services (`checkoutservice/main.go` L118–123) จึงจะทำงานของตัวเองได้
- `PlaceOrder` ต้องได้ผลลัพธ์จาก service อื่นก่อนตอบกลับผู้ใช้ ได้แก่ ราคาสินค้า ค่าส่ง การตัดบัตร และ tracking ID (`checkoutservice/main.go` L230–280; `OrderResult` ใน `demo.proto` L208)
- services เขียนด้วยหลายภาษา: Not evidenced in the provided files (ไฟล์ที่ให้มาเป็น Go ทั้งสองไฟล์)

**ปัญหา:** services ควรสื่อสารกันแบบใด และควรนิยามสัญญา (contract) ของ API อย่างไร

## Decision drivers

- ความง่ายในการพัฒนาและอ่านโค้ด (modifiability / understandability) **[Inferred]** เพราะเป็น demo application
- สัญญา API ที่มี type ชัดเจนและสร้าง stub ให้ได้หลายภาษา (interoperability)
- ผู้ใช้ต้องได้ผลลัพธ์ทันที เช่น หน้า checkout ต้องแสดง order และ tracking ID (performance / usability)
- ต้องตามรอยคำขอข้าม services ได้ (observability): `otelgrpc` อยู่ทั้งฝั่ง client และ server

## Considered options

ไม่มีหลักฐานว่าผู้ออกแบบเคยพิจารณาทางเลือกใดบ้าง ทางเลือกด้านล่างกู้คืนขึ้นมาเพื่อเปรียบเทียบ **[Inferred]**

1. gRPC แบบ synchronous (unary) กับ `.proto` ไฟล์เดียวที่ทุก service ใช้ร่วมกัน (เลือกใช้)
2. REST/JSON over HTTP โดยแต่ละ service มี OpenAPI ของตัวเอง
3. Asynchronous messaging ผ่าน message broker (เช่น Pub/Sub, Kafka) สำหรับขั้นตอนหลังตัดบัตร
4. gRPC แต่แยก `.proto` ตามแต่ละ service

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC sync + proto เดียว | สัญญามี type ชัดเจน (proto3); สร้าง stub ได้ทันที; เห็นทุก API ในไฟล์เดียว; message ใช้ร่วมกันได้ (`Money`, `Address`, `CartItem`); มี gRPC health protocol และ `otelgrpc` ใช้ได้เลย | coupling ทางเวลาสูง: service ปลายทางล่มหนึ่งตัวทำให้ `PlaceOrder` ล้มทั้งคำขอ; latency สะสมเป็นลำดับ; แก้ proto ไฟล์เดียวกระทบทุก service; ผู้ใช้เปิดดูด้วยเบราว์เซอร์ตรง ๆ ไม่ได้ ต้องมี frontend แปลงให้ |
| 2. REST/JSON | debug ง่ายด้วย curl หรือเบราว์เซอร์; ทุกภาษารองรับ | ไม่มี schema บังคับในตัว; payload ใหญ่กว่า; ต้องเขียน client เอง |
| 3. Async broker | ลด coupling; email และ shipping ทำทีหลังได้; ทนต่อ service ที่ล่มชั่วคราว | ต้องดูแล broker เพิ่ม; ต้องจัดการ eventual consistency และ idempotency; demo เข้าใจยากขึ้น; ผู้ใช้ไม่ได้ tracking ID ทันที |
| 4. gRPC + proto แยก | แต่ละทีมเปลี่ยนสัญญาได้อิสระ; ownership ชัดเจน | message ที่ใช้ร่วมกัน (`Money`, `Address`) ต้องซ้ำกันหรือแยกเป็นแพ็กเกจกลาง; ดูภาพรวมทั้งระบบยากขึ้น |

## Decision

เลือก **Option 1**: ทุก service คุยกันด้วย gRPC แบบ unary ที่นิยามใน `demo.proto` ไฟล์เดียว ส่วน `frontend` แปลงคำขอ HTTP จากผู้ใช้เป็น gRPC calls

เหตุผลที่กู้คืนได้: สัญญาที่มี type และเปรียบเทียบได้ในไฟล์เดียว, message ที่ใช้ร่วมกันข้าม service, ใช้ gRPC health check และ OpenTelemetry ได้ทันที **[Inferred]** และตรงกับเป้าหมายของ demo ที่ต้องการให้เห็นทุก API ในที่เดียว

## Consequences

- **Positive:** สร้าง client ได้จาก stub ที่ generate (`pb.NewXxxServiceClient`) จึงเพิ่มการเรียก service ใหม่ได้ง่าย (modifiability)
- **Positive:** trace ต่อเนื่องข้าม services ผ่าน `otelgrpc` และ `propagation.TraceContext` (observability)
- **Positive:** ทุก service ใช้ gRPC health protocol แบบเดียวกัน (`healthpb.RegisterHealthServer`) (operability)
- **Negative:** `PlaceOrder` เรียก services เป็นลำดับ และเรียก `GetProduct` + `Convert` แยกต่อสินค้าแต่ละชิ้น จำนวน call จึงโตตามจำนวนสินค้า latency สะสมขึ้นเรื่อย ๆ (performance)
- **Negative:** ถ้า shipping ล้มหลังตัดบัตรแล้ว คำขอจะคืน error โดยไม่มีการคืนเงิน (availability / correctness)
- **Negative:** แก้ `demo.proto` แล้วต้อง regenerate stub ของทุก service (modifiability)
- **Negative:** เชื่อมต่อแบบ `insecure.NewCredentials()` ไม่มี TLS ระหว่าง services (security)
- **Risks and mitigations:**
  - service ปลายทางช้าหรือล่ม → ใส่ deadline หรือ timeout และ retry ต่อ call ซึ่ง**ไม่พบในโค้ด**: `mustConnGRPC` สร้าง timeout 3 วินาทีแต่ทิ้ง context นั้น (`_, cancel := ...`) และ `convertCurrency` ใช้ `context.TODO()` จึงไม่ได้รับ deadline และ trace จากคำขอต้นทาง
  - ตัดบัตรแล้วส่งของไม่สำเร็จ → ใช้ saga หรือ compensation เช่น refund
  - email ล้ม → ปัจจุบันทำแค่ log warning (L273–277) ถ้าย้ายไปส่งแบบ async ผ่าน queue จะลด latency ได้

## Validation

Not evidenced in the provided files ไม่มี load test, SLO หรือ metric ในไฟล์ evidence **[Inferred]** วิธีตรวจที่เหมาะสมคือวัด p95 latency ของ `PlaceOrder` เทียบกับจำนวนสินค้าในตะกร้า และทดสอบด้วยการปิด `shippingservice` หรือ `emailservice` ทีละตัว

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | ทุก backend service นิยามใน `demo.proto` ไฟล์เดียว package `hipstershop` | Evidenced | `demo.proto` L17; `service ...` L23–243 |
| 2 | ทุก RPC เป็น unary และไม่มี streaming | Evidenced | `demo.proto` บรรทัด `rpc` ทั้งหมด (L24–244) ไม่มีคำว่า `stream` |
| 3 | frontend เชื่อมต่อ 7 services ผ่าน gRPC | Evidenced | `frontend/main.go` L141–147 `mustConnGRPC` |
| 4 | checkoutservice เชื่อมต่อ 6 services ผ่าน gRPC | Evidenced | `checkoutservice/main.go` L118–123 |
| 5 | frontend รับคำขอจากผู้ใช้ทาง HTTP | Evidenced | `frontend/main.go` L149–171 (`mux`, `http.ListenAndServe`) |
| 6 | `PlaceOrder` เรียก services เป็นลำดับแบบ synchronous: cart → catalog/currency → shipping quote → currency → payment → shipping → cart → email | Evidenced | `checkoutservice/main.go` L238–273, L288–311 |
| 7 | เรียก `GetProduct` และ `Convert` แยกต่อสินค้าแต่ละชิ้น | Evidenced | `checkoutservice/main.go` `prepOrderItems` L343–355 |
| 8 | ตัดบัตรก่อนส่งของ และไม่มี compensation เมื่อ shipping ล้ม | Evidenced | `checkoutservice/main.go` L252–261 |
| 9 | email ล้มได้โดยไม่ทำให้ order ล้ม (best effort) แต่ยังเรียกแบบ synchronous | Evidenced | `checkoutservice/main.go` L273–277 |
| 10 | error ของ `EmptyCart` ถูกละเลย | Evidenced | `checkoutservice/main.go` L263 `_ = cs.emptyUserCart(...)` |
| 11 | การเชื่อมต่อระหว่าง services ไม่มี TLS | Evidenced | `insecure.NewCredentials()` ใน `mustConnGRPC` (`checkoutservice/main.go` L215, `frontend/main.go` L230) |
| 12 | timeout 3 วินาทีใน `mustConnGRPC` ไม่มีผลจริง | Evidenced | `_, cancel := context.WithTimeout(...)` L212 / L227 (context ที่ได้ไม่ถูกใช้) |
| 13 | `convertCurrency` ไม่ส่งต่อ deadline หรือ trace ของคำขอต้นทาง | Evidenced | `checkoutservice/main.go` L360 `context.TODO()` |
| 14 | มี distributed tracing ผ่าน OpenTelemetry บน gRPC | Evidenced | `otelgrpc.NewServerHandler` L139, `NewClientHandler` L216, `propagation.TraceContext` L135–137 |
| 15 | ใช้ gRPC health protocol | Evidenced | `checkoutservice/main.go` L143–144, L222–228 |
| 16 | message ที่ใช้ร่วมกัน (`Money`, `Address`, `CartItem`) เป็นเหตุผลที่รวมไว้ใน proto ไฟล์เดียว | Inferred | `CartItem` L29 ถูกใช้ทั้งใน cart และ shipping แต่ไม่มีเอกสารบอกเหตุผล |
| 17 | เลือก gRPC เพราะสัญญามี type และรองรับหลายภาษา | Inferred | ไฟล์ evidence เป็น Go ทั้งหมด และไม่มีข้อความระบุเหตุผล |
| 18 | ผู้ใช้ต้องได้ tracking ID ทันที จึงใช้แบบ synchronous | Inferred | `OrderResult.shipping_tracking_id` (`demo.proto` L208) อยู่ใน response แต่ไม่มีหลักฐานว่าเป็นความต้องการตั้งต้น |
| 19 | ความง่ายของ demo เป็น decision driver หลัก | Inferred | ไม่มีในไฟล์ evidence |
| 20 | ผู้ออกแบบเคยพิจารณา REST, broker หรือ proto แยก | Inferred | ไม่มีหลักฐานว่าพิจารณาทางเลือกใด |
| 21 | แต่ละ service generate stub ของตัวเองจาก `demo.proto` | Inferred | checkout import `src/checkoutservice/genproto` (L34) ซึ่งแสดงว่ามี stub ในโมดูลของตัวเอง แต่ไม่มีหลักฐานว่า generate มาจาก `demo.proto` |
| 22 | shopping assistant ไม่ได้ใช้ gRPC ซึ่งเป็นข้อยกเว้นของ "ทุก service ใช้ gRPC" | Inferred | `frontend/main.go` L88, L139 เก็บ address ไว้แต่ไม่เรียก `mustConnGRPC`; โปรโตคอลจริงไม่ปรากฏในไฟล์ evidence |
| 23 | gRPC (HTTP/2 + protobuf) มี payload เล็กและเร็วกว่า REST/JSON | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 24 | การเรียกแบบ synchronous เป็นลำดับทำให้ availability รวมเท่ากับผลคูณของ availability ของแต่ละ service | Assumption | ความรู้ทั่วไปเรื่อง distributed systems |
| 25 | broker ช่วยลด temporal coupling แต่ต้องรองรับ eventual consistency และ idempotency | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 26 | saga หรือ compensation เป็นวิธีแก้กรณีตัดบัตรแล้วส่งของไม่สำเร็จ | Assumption | pattern ทั่วไป |
