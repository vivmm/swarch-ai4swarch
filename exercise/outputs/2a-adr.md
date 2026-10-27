> Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ post-order actions ใหม่ 3 อย่าง (ตาม requirement):

| # | Action | ผู้รับผิดชอบ / ปลายทาง |
|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | **MailFast** (external email provider ใหม่) แทน logic mock ใน `emailservice` |
| 2 | ให้ loyalty points | `loyaltyservice` (ยังไม่มี ต้องสร้างใหม่) |
| 3 | บันทึก analytics event "order placed" | ระบบของทีม marketing |

- ทีม marketing คาดว่า **ปีหน้าจะมี post-order actions เพิ่ม** (SMS, fraud scoring, recommendations update) ดังนั้นจำนวนผู้ที่ต้องรู้ว่า "มี order เกิดขึ้น" จะเพิ่มขึ้นเรื่อย ๆ
- MailFast เป็น external system ซึ่งเราควบคุม availability และ latency ไม่ได้
- **[Inferred]** ใน Online Boutique service ที่รับผิดชอบการสั่งซื้อน่าจะเป็น `checkoutservice` และน่าจะเรียก `emailservice` แบบ synchronous ใน flow เดียวกับการสั่งซื้อ — Not evidenced in the provided files (run นี้ไม่ได้อ่าน source code)

**ปัญหาที่ต้องตัดสินใจ:** service ที่ทำ order ควรแจ้ง post-order actions เหล่านี้อย่างไร เพื่อให้เพิ่ม action ใหม่ได้ง่าย และความล้มเหลวของ action ใด action หนึ่ง (โดยเฉพาะ MailFast) ไม่ทำให้การสั่งซื้อล้มเหลวหรือช้าลง

## Decision drivers

1. **Modifiability / extensibility** — เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ service ที่ทำ order
2. **Availability ของ checkout** — การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม
3. **Performance (latency ของ checkout)** — ลูกค้าไม่ควรต้องรอ post-order actions ทั้งหมดเสร็จ
4. **Reliability** — แต่ละ action ต้องเกิดขึ้นจริงสำหรับทุก order (ไม่หาย) และไม่ซ้ำจนเป็นปัญหา (เช่น ได้ points สองครั้ง)
5. **Cost / operational complexity** — จำนวน infrastructure ใหม่ที่ทีมต้องดูแล

## Considered options

1. **Option A — Synchronous orchestration ใน order service:** service ที่ทำ order เรียก MailFast adapter, `loyaltyservice` และ analytics ทีละตัวโดยตรง (เช่น gRPC/HTTPS) ก่อนตอบลูกค้า
2. **Option B — Event-driven publish/subscribe ผ่าน message broker:** service ที่ทำ order publish event `OrderPlaced` ครั้งเดียวไปยัง message broker; แต่ละ consumer (email adapter → MailFast, `loyaltyservice`, analytics) subscribe และทำงานแบบ asynchronous ของตัวเอง
3. **Option C — Dedicated post-order orchestrator service:** service ที่ทำ order เรียก `postorderservice` ใหม่ตัวเดียว (asynchronous หรือ fire-and-forget) ซึ่งรับผิดชอบเรียกปลายทางทั้งหมดตามลำดับ/พร้อมกัน

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Synchronous orchestration | เรียบง่าย ไม่มี infrastructure ใหม่; debug ง่ายเพราะเป็น call chain เดียว; รู้ผลทันทีว่า action สำเร็จหรือไม่ | checkout latency = ผลรวมของทุก call; ถ้า MailFast ล่ม checkout อาจล้มเหลวหรือค้าง (availability ต่ำลง); ทุก action ใหม่ต้องแก้และ deploy order service (modifiability ต่ำ); order service coupling กับทุกปลายทาง |
| B — Event-driven pub/sub | เพิ่ม consumer ใหม่ได้โดยไม่แตะ order service (modifiability สูง); checkout ไม่ต้องรอ consumer (latency ต่ำ); consumer ล่มได้โดย checkout ยังทำงาน และ broker เก็บ event ไว้ retry ได้ (availability); แต่ละ consumer scale และ deploy แยกกันได้ | ต้องดูแล message broker ใหม่ (cost, operational complexity); eventual consistency — อีเมล/points มาถึงช้ากว่า order เล็กน้อย; ต้องจัดการ duplicate delivery (idempotency) และ dead-letter; tracing ข้าม async ยากขึ้น; ต้องกำหนดและ version schema ของ event |
| C — Post-order orchestrator | order service coupling กับ service เดียว; logic ของ post-order รวมอยู่ที่เดียว ดูภาพรวมง่าย; ไม่ต้องมี broker | orchestrator กลายเป็น single point of failure และจุดที่ต้องแก้ทุกครั้งที่เพิ่ม action (ย้าย coupling ไป ไม่ได้ลบ); ถ้าเรียกแบบ fire-and-forget โดยไม่มีที่เก็บถาวร event อาจหายเมื่อ orchestrator ล่ม; ต้องเขียน retry/state เองซึ่งซ้ำกับสิ่งที่ broker ให้มาอยู่แล้ว |

## Decision

เลือก **Option B — Event-driven publish/subscribe ผ่าน message broker**

- service ที่ทำ order publish event `OrderPlaced` (เช่น order id, customer contact, รายการสินค้า, ยอดเงิน) หลังบันทึก order สำเร็จ แล้วตอบลูกค้าทันที
- consumers ช่วงแรก:
  - **email consumer** (แทน logic mock ของ `emailservice`) → เรียก MailFast
  - **`loyaltyservice`** → คำนวณและบันทึก loyalty points
  - **analytics consumer** → บันทึก event "order placed" ให้ทีม marketing
- action ในอนาคต (SMS, fraud scoring, recommendations update) เพิ่มเป็น subscriber ใหม่ โดยไม่ต้องแก้ publisher
- เทคโนโลยีของ broker (เช่น RabbitMQ, Kafka, Google Cloud Pub/Sub) ยังไม่เลือก — ดู Open questions

**เหตุผล:** Option B ตอบ driver อันดับ 1–3 ได้ดีที่สุด: แยก order service ออกจากจำนวน action ที่เพิ่มขึ้น (modifiability), กันความล้มเหลวของ MailFast และ consumer อื่นไม่ให้กระทบ checkout (availability) และตัด post-order work ออกจาก critical path (performance) ส่วนต้นทุนของ broker และ eventual consistency ยอมรับได้เพราะ action ทั้งสามไม่จำเป็นต้องเสร็จก่อนตอบลูกค้า **[Inferred]**

> หมายเหตุ: ถ้า fraud scoring ในอนาคตต้อง **block** order ก่อนยืนยัน (synchronous) action นั้นไม่เหมาะกับ subscriber แบบ asynchronous และต้องตัดสินใจแยกเป็น ADR ใหม่

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ได้โดย deploy consumer ใหม่อย่างเดียว ไม่แก้ order service (quality attribute: modifiability)
- **Positive:** checkout ยังสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม; event รอใน broker จนกว่า consumer กลับมา (quality attribute: availability)
- **Positive:** latency ของ checkout ไม่ขึ้นกับ latency ของ MailFast หรือ consumer อื่น (quality attribute: performance)
- **Positive:** consumer แต่ละตัว scale แยกกันได้ตามโหลดของตัวเอง (quality attribute: scalability)
- **Negative:** ต้อง deploy และดูแล message broker ใหม่ รวมถึง monitoring และ capacity (quality attribute: cost / operability)
- **Negative:** อีเมลยืนยันและ points เกิดขึ้นช้ากว่า order (eventual consistency) และลูกค้าอาจเห็น points ยังไม่ขึ้นทันที (quality attribute: consistency / usability)
- **Negative:** การ trace ปัญหาข้าม async boundary ยากกว่า call chain เดียว (quality attribute: testability / observability)
- **Risks and mitigations:**
  - *Event หาย* ถ้า order บันทึกแล้วแต่ publish ไม่สำเร็จ → ใช้ transactional outbox หรือ publish พร้อม retry และ acknowledgement ของ broker
  - *Duplicate delivery* (at-least-once) ทำให้อีเมลซ้ำหรือ points ซ้ำ → ให้ consumer เป็น idempotent โดยใช้ order id เป็น deduplication key
  - *MailFast ล่มหรือ rate-limit* → retry แบบ exponential backoff, dead-letter queue และ alert
  - *Schema ของ event เปลี่ยนแล้ว consumer พัง* → กำหนด schema ที่ version ได้ (เช่น protobuf) และเปลี่ยนแบบ backward-compatible เท่านั้น
  - *ข้อมูลส่วนบุคคล (อีเมลลูกค้า) อยู่ใน event ที่ทุก consumer อ่านได้* → ส่งเฉพาะ field ที่จำเป็น หรือให้ consumer ดึงข้อมูลเพิ่มเอง; ทบทวนกับข้อกำหนด privacy (ดู Open questions)

## Validation

- **Load test:** วัด p95/p99 latency ของ checkout ก่อนและหลังเปลี่ยน; latency ไม่ควรเพิ่มขึ้นเมื่อเพิ่ม consumer (เป้าหมายตัวเลข: ดู Open questions)
- **Fault injection:** ปิด MailFast (หรือ mock), `loyaltyservice` และ analytics consumer ทีละตัว → checkout ต้องยังสำเร็จ และเมื่อกลับมา action ที่ค้างต้องทำครบ
- **Duplicate test:** ส่ง event `OrderPlaced` ซ้ำ → ลูกค้าต้องได้อีเมลหนึ่งฉบับและ points หนึ่งครั้ง
- **Metrics:** consumer lag, จำนวนข้อความใน dead-letter queue, อัตราสำเร็จของ MailFast, เวลาตั้งแต่ order ถึงส่งอีเมลสำเร็จ
- **Modifiability check:** เพิ่ม consumer ตัวอย่าง (เช่น SMS stub) โดยไม่แก้ order service — วัดว่าไม่ต้องมีการเปลี่ยน code ใน publisher
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มทำ post-order actions ของปีหน้า (วันที่แน่นอน: ดู Open questions)

## Open questions

- ใครเป็น deciders ของ ADR นี้?
- service ใดใน Online Boutique ที่ทำ order และเรียก `emailservice` อยู่ในปัจจุบัน และเรียกด้วย protocol อะไร? (Not evidenced in the provided files)
- เป้าหมาย latency ของ checkout และจำนวน order ต่อวินาที (ปัจจุบันและที่คาดในปีหน้า) คือเท่าไร?
- อีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาที/นาที? points ต้องขึ้นภายในเวลาเท่าไร?
- MailFast มี API แบบใด (HTTPS/SMTP), SLA, rate limit, และรองรับ idempotency key หรือไม่?
- ทีม marketing ต้องการ analytics event ในรูปแบบใดและส่งไปที่ระบบไหน (มีระบบ analytics อยู่แล้วหรือไม่)?
- ทีมมีประสบการณ์กับ message broker ตัวใด และมีข้อจำกัดด้าน platform/cost (เช่น ต้องรันบน Kubernetes เอง หรือใช้ managed service ได้)?
- fraud scoring ในอนาคตต้อง block order ก่อนยืนยันหรือไม่ (ถ้าใช่ ต้องใช้ synchronous call แยก)?
- มีข้อกำหนด privacy/compliance เรื่องข้อมูลลูกค้าใน event หรือไม่?
- `loyaltyservice` จะเก็บข้อมูล points ไว้ที่ data store ใด และใครเป็นเจ้าของ?
- ยอมรับได้หรือไม่ถ้าอีเมลซ้ำในกรณีที่หายาก (at-least-once) หรือต้องการ exactly-once?
