# แผนงาน: สร้างและแก้ไขอีเว้นต์ (Event Management)

## 1. สรุปแนวทาง
- ผู้จัดงานต้องสามารถสร้างอีเว้นต์ใหม่และแก้ไขข้อมูลอีเว้นต์เดิมผ่านฟอร์มที่มีการตรวจสอบความครบถ้วนของข้อมูลก่อนบันทึก
- ระบบต้องตรวจสอบสถานะสถานที่และช่วงเวลาแบบ atomic ก่อนบันทึก เพื่อป้องกันการจองซ้อนพร้อมกันและให้สอดคล้องกับ FR-EVM-03, FR-EVM-04 และ FR-EVM-07
- ผู้จัดงานเลือกสถานที่จากรายการที่ Admin จัดการผ่าน UC-15 เท่านั้น และระบบต้องไม่ยอมบันทึกเมื่อข้อมูลไม่ครบ หรือสถานที่ไม่พร้อมใช้งาน หรือตรวจสอบสถานที่ไม่สำเร็จ
- การยกเลิกก่อนบันทึกต้องไม่มีผลต่อข้อมูลเดิม และเมื่อมีการพยายามสร้าง/แก้ไขโดยผู้ใช้ที่ไม่มีสิทธิ์ ระบบต้องปฏิเสธด้วยข้อความสากลและบันทึก Log
- แบบฟอร์มต้นแบบจะใช้ React + Vite สำหรับ UI และ Python FastAPI สำหรับ API validation, authorization และ transaction handling

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React + Vite | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สร้างหน้า Create/Edit Event และแบบฟอร์มตรวจสอบข้อมูล |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้รับคำขอจัดการอีเว้นต์ ตรวจสิทธิ์ และจัดการ transaction |
| ฐานข้อมูลเชิงสัมพันธ์ | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้เก็บข้อมูลอีเว้นต์ สถานที่ และบันทึก log ตรวจสิทธิ์ |
| Transaction / optimistic locking หรือ equivalent | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้เพื่อป้องกัน race condition ระหว่างตรวจสถานที่และบันทึกอีเว้นต์ |
| Server-side authorization | NFR-ACC-01 | ตรวจสิทธิ์ทำฝั่งระบบเสมอ ไม่ใช่แค่ซ่อนเมนู |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR / NFR |
|---|---|---|
| Event | event_id, title, description, event_date, start_time, end_time, venue_id, is_registration_open, is_band_application_open, seat_capacity, created_by, updated_by, created_at, updated_at | FR-EVM-02, FR-EVM-04, ASM-02, ASM-04, ASM-05 |
| Venue | venue_id, name, status, availability, admin_owner, created_at, updated_at | FR-EVM-07, ASM-02, ASM-03 |
| EventConflictCheck | event_id, venue_id, check_time, result, reason | FR-EVM-03, FR-EVM-04, FR-EVM-07 |
| UserAccessLog | log_id, user_id, action_type, target_event_id, result, timestamp, request_payload_hash | NFR-ACC-01, ASM-06 |
| RegistrationSummary | event_id, registered_count, last_updated_at | ASM-05 |

หมายเหตุ:
- สถานที่จะถูกเลือกจากรายการที่ Admin จัดการผ่าน UC-15 เท่านั้น ดังนั้น UI จะใช้ dropdown/list จากข้อมูล Venue ที่สถานะเป็นเปิดใช้งาน
- จำนวนที่นั่งใช้เฉพาะการลงทะเบียนเข้าชม และต้องเป็นจำนวนเต็มตั้งแต่ 1 และต้องไม่ต่ำกว่าจำนวนผู้ลงทะเบียนที่ยังไม่ยกเลิก ตาม ASM-05
- การตรวจสอบสถานที่และการบันทึกข้อมูลอีเว้นต์จะเริ่มจาก transaction เดียวกัน เพื่อให้ห้ามมีการจองซ้อนพร้อมกัน

## 4. API / หน้าจอ

| รายการ | Input / Output หลัก | รองรับ |
|---|---|---|
| `GET /events/new` | หน้าฟอร์มสร้างอีเว้นต์ | FR-EVM-01 |
| `GET /events/:id/edit` | หน้าฟอร์มแก้ไขอีเว้นต์ | FR-EVM-01 |
| `GET /venues/available` | Output: รายการสถานที่พร้อมใช้งานและสถานะ | FR-EVM-02, FR-EVM-07, ASM-02 |
| `POST /events/validate` | Input: event payload + venue_id + date/time; Output: validation result + conflict details | FR-EVM-03, FR-EVM-06, FR-EVM-07 |
| `POST /events` | Input: event payload; Output: created event / validation error | FR-EVM-04, AC-EVM-01 |
| `PUT /events/:id` | Input: updated event payload; Output: updated event / validation error | FR-EVM-04, AC-EVM-02 |
| `POST /events/:id/cancel` หรือ `cancel` action ใน form | Output: ยกเลิกโดยไม่บันทึกการเปลี่ยนแปลง | FR-EVM-05, AC-EVM-04 |
| `GET /events/:id` | Output: ข้อมูลอีเว้นต์ที่บันทึกแล้ว | FR-EVM-08, AC-EVM-03 |
| `POST /events/:id/permission-check` | Output: permitted / denied | NFR-ACC-01 |
| `POST /admin/access-log` หรือ internal audit log | Input: user_id, action, target_event_id, result; Output: saved log | ASM-06 |

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| PRE-EVM-01 | ใช้ใน middleware ตรวจว่าผู้จัดงานเข้าสู่ระบบและมีสิทธิ์จัดการอีเว้นต์ก่อนเข้าหน้า create/edit event หรือเรียก API | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-EVM-01 | `test_AC_EVM_01_create_event_success` | สร้างอีเว้นต์ด้วยข้อมูลครบถ้วนและสถานที่พร้อมใช้งาน ตรวจว่าระบบบันทึกสำเร็จ |
| AC-EVM-02 | `test_AC_EVM_02_edit_event_success` | แก้ไขอีเว้นต์ที่มีอยู่แล้ว พร้อมข้อมูลใหม่ที่ถูกต้อง ตรวจว่าระบบยืนยันและบันทึกสำเร็จ |
| AC-EVM-03 | `test_AC_EVM_03_event_visible_to_users` | ดึงข้อมูลอีเว้นต์ที่บันทึกแล้วจาก API/หน้า UI ตรวจว่าผู้ใช้เห็นข้อมูลตรงตามที่บันทึก |
| AC-EVM-04 | `test_AC_EVM_04_cancel_before_save` | กรอกข้อมูลแบบไม่บันทึกและกดยกเลิก ตรวจว่าข้อมูลเดิมไม่เปลี่ยนแปลง |
| AC-EVM-05 | `test_AC_EVM_05_incomplete_data_rejected` | ส่งข้อมูลไม่ครบ เช่น ขาดวันหรือสถานที่ ตรวจว่าระบบแจ้งและไม่บันทึก |
| AC-EVM-06 | `test_AC_EVM_06_venue_unavailable_or_check_failed` | จำลองสถานที่ไม่พร้อมใช้งาน หรือฐานข้อมูลไม่ตอบ ตรวจว่าระบบไม่บันทึกและแสดงข้อความที่ถูกต้อง |
| AC-EVM-07 | `test_AC_EVM_07_unauthorized_user_denied` | ลองสร้าง/แก้ไขโดยผู้ใช้ที่ไม่มีสิทธิ์ ตรวจว่าระบบปฏิเสธและบันทึก log |

## 7. ลำดับงาน

1. สร้าง UI และ route สำหรับหน้า create/edit event พร้อม validation เบื้องต้น (FR-EVM-01, FR-EVM-02)
2. สร้าง schema ของ Event, Venue และ log บนฐานข้อมูลเชิงสัมพันธ์ พร้อม index ที่จำเป็น เช่น venue_id, event_date, start_time (FR-EVM-02, FR-EVM-04)
3. สร้าง API validate event payload: ตรวจความครบถ้วนของข้อมูล, ระบุวันที่/เวลา/สถานที่ และตรวจสถานะ venue (FR-EVM-03, FR-EVM-06, FR-EVM-07)
4. สร้าง transaction-based save flow ที่รวม validation + insert/update พร้อมการป้องกัน race condition เมื่อหลาย request บันทึกพร้อมกัน (FR-EVM-04, ASM-03)
5. สร้าง API และ UI สำหรับ cancel ก่อนบันทึก โดยไม่นำผลลัพธ์ไปเขียนฐานข้อมูล (FR-EVM-05, AC-EVM-04)
6. สร้าง middleware/guard สำหรับตรวจสิทธิ์ฝั่งระบบ และบันทึก log ทุกครั้งที่ผู้ใช้ไม่มีสิทธิ์พยายามสร้างหรือแก้ไข (NFR-ACC-01, ASM-06)
7. สร้างฟังก์ชัน lookup ของสถานที่จาก UC-15 และกำหนดเงื่อนไขพร้อมใช้งานตาม ASM-02
8. ทดสอบ end-to-end ครบทุก AC และตรวจความสอดคล้องกับ spec ก่อนส่งทีมตรวจ

## 8. สิ่งที่ยังไม่ทำ
- ไม่มี Open Questions ที่ค้างอยู่ใน Draft v2 ของ spec หลังทีมตอบ clarifying แล้ว
- ส่วนที่เกี่ยวข้องกับความละเอียดทางธุรกิจเพิ่มเติม เช่น ป้ายข้อความเฉพาะหน้า UI หรือรายละเอียด log schema แบบลึก จะยังไม่กำหนดจนกว่าทีมจะมีความต้องการชัดเจนในภายหลัง
