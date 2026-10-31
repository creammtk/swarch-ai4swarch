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

| ผู้เรียก → ผู้ถูกเรียก            | หลักฐานที่ AI อ้าง                          | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ                                                                                            |
| --------------------------------- | ------------------------------------------- | :----------------------------: | --------------------------------------------------------------------------------------------------------- |
| frontend → productcatalogservice  | Environment variable ใน Kubernetes manifest |               ✅               | อ่าน env ที่ frontend/main.go:132 และเรียก ListProducts / GetProduct ที่ frontend/rpc.go:44-54            |
| checkoutservice → paymentservice  | `PAYMENT_SERVICE_ADDR`                      |               ✅               | PlaceOrder เรียก cs.chargeCard ที่ main.go:252 ซึ่งเรียก PaymentService.Charge ที่ main.go:369-370        |
| checkoutservice → shippingservice | `SHIPPING_SERVICE_ADDR`                     |               ✅               | เรียก 2 ครั้ง: GetQuote ที่ main.go:314 และ ShipOrder ที่ main.go:258, main.go:386-387                    |
| checkoutservice → emailservice    | `EMAIL_SERVICE_ADDR`                        |               ✅               | cs.sendOrderConfirmation ที่ main.go:273 ซึ่งเรียก EmailService.SendOrderConfirmation ที่ main.go:379-380 |
| cartservice → redis-cart          | Redis connection configuration              |               ✅               | ใช้ Redis เฉพาะเมื่อตั้ง REDIS_ADDR ไว้ ดู Startup.cs:29-40                                               |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): shoppingassistantservice มีใน env ของ frontend แต่ไม่มี manifest
  loadgenerator ถูกวาดอยู่ใน System_Boundary ทั้งที่ไม่ได้ deploy
  redis-cart ใช้ emptyDir

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   Run A สามารถระบุ Service ได้ 9 รายการ แต่ไม่สามารถระบุ Dependency ระหว่าง Service ได้ เนื่องจากไฟล์ `.proto` แสดงเพียง Interface และ RPC ที่แต่ละ Service ให้บริการ ไม่ได้ระบุว่า Service ใดเป็นผู้เรียกใช้งาน ส่วน Run B สามารถระบุ Container ได้ 12 รายการและ Dependency 17 เส้นจาก Kubernetes manifests แต่ยังไม่สามารถยืนยันการเรียกใช้งานจริงได้หากไม่ตรวจสอบ Source Code เพิ่มเติม

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   ข้อผิดพลาดที่อาจทำให้สมาชิกใหม่เข้าใจผิดมากที่สุดคือการแสดง Dependency ที่อนุมานจาก Environment Variables ว่าเป็นการเรียกใช้งานจริงระหว่าง Service เพราะการมีค่าที่อยู่ของ Service ใน Configuration ไม่ได้หมายความว่ามีการเรียกใช้งานจริงเสมอไป หากนำแผนภาพนี้ไปใช้โดยไม่ตรวจสอบ อาจทำให้เข้าใจโครงสร้างและการทำงานของระบบผิดพลาด รวมถึงส่งผลต่อการวิเคราะห์ปัญหาและการแก้ไขระบบในอนาคต
