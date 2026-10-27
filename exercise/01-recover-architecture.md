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

| ผู้เรียก → ผู้ถูกเรียก               | หลักฐานที่ AI อ้าง                                                                    | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
| ------------------------------------ | ------------------------------------------------------------------------------------- | ------------------------------ | -------------- |
| Shopper → `frontend`                 | `frontend.yaml` (Service `frontend-external`, `type: LoadBalancer`, port name `http`) | ✅                             | มีไฟล์จริง     |
| `frontend` → `productcatalogservice` | `frontend.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`)           | ✅                             | มีไฟล์จริง     |
| `frontend` → `currencyservice`       | `frontend.yaml` (`CURRENCY_SERVICE_ADDR=currencyservice:7000`)                        | ✅                             | มีไฟล์จริง     |
| `frontend` → `cartservice`           | `frontend.yaml` (`CART_SERVICE_ADDR=cartservice:7070`)                                | ✅                             | มีไฟล์จริง     |
| `frontend` → `recommendationservice` | (`RECOMMENDATION_SERVICE_ADDR=recommendationservice:8080`)                            | ✅                             | มีไฟล์จริง     |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - frontend.yaml อ้างถึง shoppingassistantservice แต่แผนภาพไม่ได้แสดงไว้

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   Answer: ไฟล์ .proto บอกเพียงว่าแต่ละ service ให้บริการอะไร (interface ฝั่งผู้ให้บริการ) แต่ไม่ได้บอกว่า ใครเป็นผู้เรียก ข้อมูลการเชื่อมต่อจริงอยู่ในไฟล์ deploy
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   Answer:loadgenerator.yaml ไม่ได้ถูก deploy โดยค่าเริ่มต้น และเป็นเพียงเครื่องมือทดสอบที่สร้าง traffic จำลอง สมาชิกใหม่อาจเข้าใจผิดว่า traffic นี้มาจากผู้ใช้จริงที่อยู่บน production
