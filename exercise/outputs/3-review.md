Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish "order placed" event via RabbitMQ

แหล่งที่ใช้: `docs/adr/0002-publish-order-placed-event-via-rabbitmq.md` และ `docs/scenario.md` เท่านั้น

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | ลูกค้า | ส่ง `PlaceOrder` | Peak 11.11, 200 orders/s, MailFast ตอบ p99 8 s | order ยืนยันโดยไม่รอ email/loyalty/analytics | p95 `PlaceOrder` ≤ 500 ms (เพิ่มจาก 300 ms ได้ไม่เกิน 200 ms) |
| QA2 Reliability | RabbitMQ หรือ `loyaltyservice` ล่ม | order สำเร็จ 200 orders/s ระหว่างที่ล่ม | Peak load, at-least-once delivery | event ไม่หาย แต้มเข้าเมื่อระบบกลับมา | แต้มเข้าภายใน 1 นาทีหลัง order; `order_id` ที่ได้แต้มมากกว่า 1 ครั้ง = 0; order ที่ fail = 0 |
| QA3 Throughput | Email worker | backlog 150 emails/s ที่โตขึ้น | Peak, MailFast rate limit 50/s, fail 0.5 % | ส่งอีเมลครบโดยไม่เกิน rate limit | ไม่เกิน 50 emails/s ตลอดเวลา; อีเมลที่ fail ถาวรเข้า DLQ และมี alert; ระบาย backlog ได้ในเวลาที่ business ยอมรับ (ยังไม่กำหนด) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | เอา MailFast (p95 1.5 s, p99 8 s) ออกจาก critical path | Option 1 ทำให้ p95 > 500 ms แน่นอน; การตัดออกคือเงื่อนไขขั้นต่ำของ QA1 |
| Non-risk | queue แยกต่อ consumer บน exchange เดียว | backlog อีเมลประมาณ 9,000 ต่อนาทีของ peak จะไม่ถ่วง loyalty (QA2); เพิ่ม SMS/recommendations ได้โดยไม่แก้ `checkoutservice` |
| Non-risk | ใช้ `order_id` (สร้างก่อน `chargeCard`) เป็น idempotency key | ทำให้ at-least-once + idempotent consumer เป็นไปได้จริง |
| Risk | "Transactional outbox" ไม่ transactional: `checkoutservice` ไม่มีข้อมูล order ใน DB ให้ commit ร่วมกัน ถ้า pod crash หลัง `chargeCard` แต่ก่อน insert ลง outbox, event หายเงียบๆ (ไม่มีแม้แต่ log/alert) | ADR บอกว่า outbox "ปิดช่องว่างของ driver 3" แต่ยังเหลือหน้าต่างที่ลูกค้าจ่ายเงินแล้วไม่ได้แต้ม ซึ่งผิด exactly-once |
| Risk | Outbox relay เป็น goroutine ใน `checkoutservice` แต่ไม่ได้บอกว่าถ้ามีหลาย replica จะกันไม่ให้ relay หลายตัวอ่าน row เดียวกันอย่างไร | publish ซ้ำจำนวนมากตอน peak; สำหรับ email ซ้ำ ADR ยอมรับเองว่า "อาจเกิดได้" |
| Risk | Outbox DB ตัวใหม่อยู่บน critical path ของ `PlaceOrder` และ technology "ยังไม่ตัดสิน" | ผลของ QA1 และ availability ขึ้นกับ component ที่ยังไม่มีตัวตน และวัดไม่ได้ |
| Sensitivity point | relay poll interval + เวลา broker/consumer recover เทียบกับงบ loyalty 1 นาที | ถ้า broker ล่มเกินประมาณ 1 นาที (ซึ่งเป็นสิ่งที่ Validation "kill RabbitMQ ทุก node" ทดสอบอยู่) QA2 ผิดแน่นอน ADR ไม่ได้บอกว่ายอมรับได้ไหม |
| Sensitivity point | concurrency/prefetch ของ email worker | จะส่งได้ 50/s ที่ latency 1.5–8 s ต้องมี request ค้างพร้อมกันประมาณ 75–400 ตัว (Little's law) **[Inferred]**; ถ้าตั้งต่ำไป throughput จะต่ำกว่า 50/s และ backlog ไม่มีวันหมด |
| Sensitivity point | rate limiter 50/s ต้องเป็นแบบ global ไม่ใช่ต่อ pod | ถ้า scale email worker เป็น N pod แต่ละ pod จำกัดเอง ก็จะส่งได้ถึง N×50/s และชน rate limit ของ MailFast |
| Trade-off point | outbox DB: reliability ↑ แต่ latency, availability ของ checkout และ operability ↓ (stateful component ตัวที่สอง, ทีม 4 คน) | ต้องเทียบกับทางเลือกที่ publish ตรงพร้อม publisher confirms ไปยัง quorum queues ซึ่ง ADR ไม่ได้ประเมิน |
| Trade-off point | queue ดูดซับ peak: availability/throughput ↑ แต่ความทันเวลาของอีเมล ↓ | peak 1 ชั่วโมง → backlog ประมาณ 540,000 emails ใช้เวลาระบายประมาณ 3 ชั่วโมง และต้องมี disk ของ broker รองรับ |

## 3. Missing or weak

- **Option ที่ไม่ได้พิจารณา:** (a) RabbitMQ แบบ publish ตรงพร้อม confirms โดยไม่มี outbox DB และใช้ outbox เป็น fallback เมื่อ publish fail เท่านั้น; (b) hybrid คือเรียก `loyaltyservice` ภายในแบบ sync + idempotent (ตอบเร็ว) แล้วส่งเฉพาะ email/analytics แบบ async; (c) relay แบบ CDC แทน polling
- **Claim ที่ไม่มีหลักฐาน:** "ไม่มีใครในทีมเคยใช้ Kafka (`docs/scenario.md`)": scenario บอกแค่ว่า 2 คนเคยใช้ RabbitMQ ไม่ได้พูดถึง Kafka. "RabbitMQ เป็นตัวเลือกเดียวที่ตอบ driver 1–5": Kafka ก็ตอบ 1–5 ได้ เหตุผลที่ตัดจริงคือ driver 6
- **Exactly once ยังพิสูจน์ไม่ครบ:** "unique key ก่อนบวกแต้ม" จะใช้ได้ต่อเมื่อบันทึก `order_id` และบวกแต้มอยู่ใน transaction เดียวกันใน loyalty store ซึ่งยังไม่ได้เลือก
- **Consequence ที่ไม่ได้เขียน:** `emailservice` เดิมจะถูกลบหรือเปลี่ยนเป็น email worker; ขนาด disk/memory ของ RabbitMQ สำหรับ backlog ช่วง peak; การส่ง event ที่มีข้อมูลส่วนตัว (email) ไปยัง analytics
- **Validation ขัดกันเอง:** test "loyalty p100 ภายใน 1 นาที" กับ test "kill broker ทุก node" ผ่านพร้อมกันไม่ได้ ต้องกำหนดว่า outage ยาวเท่าไรที่ยังต้องผ่าน 1 นาที

## 4. Verdict: **Accept with changes**

1. **ปิดช่องว่าง exactly-once ให้ครบ:** ยอมรับว่า outbox ไม่ได้ atomic กับ payment แล้วเพิ่ม reconciliation (เทียบ order ที่ charge แล้วกับ outbox/loyalty) และกำหนดให้ `loyaltyservice` บันทึก dedup และบวกแต้มใน transaction เดียวกัน
2. **ระบุการทำงานของ relay:** การกันหลาย replica (row lock หรือ leader election), poll interval และงบเวลา end-to-end ของ loyalty ที่ต้องอยู่ใน 1 นาที รวมถึงความยาว outage สูงสุดที่ยังรับประกันได้ หรือประเมิน option publish ตรงพร้อม confirms เทียบกับ outbox DB ให้ชัด
3. **ออกแบบ email path ให้มีตัวเลข:** rate limiter แบบ global 50/s, concurrency ที่สอดคล้องกับ latency p99 8 s, ขนาด queue สำหรับ peak ที่ยาวที่สุด, และ dedup อีเมลซ้ำ พร้อมแก้ claim เรื่อง Kafka ให้ตรงกับ `docs/scenario.md`
