# แผนงาน: จัดตารางการแสดง (Schedule Management)

## 1. สรุปแนวทาง
- ฟีเจอร์นี้ให้ผู้จัดงานจัดตารางการแสดงของวงที่ได้รับอนุมัติในอีเว้นต์ที่กำหนด โดยมีหลักการสำคัญว่า: บันทึกก่อนเป็น Draft และให้เผยแพร่เป็นขั้นตอนแยกจากการบันทึก
- ระบบจะต้องรองรับการเลือกวงที่ได้รับอนุมัติ กำหนดวัน เริ่มเวลา สถานที่ และลำดับ (order) ของแต่ละวง โดยลำดับเป็นฟิลด์แยกจากวันและเวลา
- ก่อนบันทึก ระบบต้องตรวจสอบ Time Conflict และ Venue Conflict อย่างอิสระต่อกัน และแสดง error ทั้งสองรายการพร้อมกันในคำตอบเดียว หากมีปัญหา
- เมื่อบันทึกสำเร็จ ระบบจะเก็บเป็น Draft เท่านั้น จนกว่าผู้จัดงานกดปุ่มเผยแพร่แยกอีกขั้นตอนหนึ่ง เพื่อให้ผู้จัดงานสามารถตรวจทานก่อนเปิดให้สาธารณะเห็น
- การเข้าถึงฟีเจอร์นี้จะถูกจำกัดเฉพาะผู้จัดงานที่ได้รับสิทธิ์ตาม CON-ACC-01
- แผนนี้จะใช้ React + Vite สำหรับส่วนหน้าบ้าน และ FastAPI สำหรับ API รวมถึง validation และเก็บข้อมูลตารางแบบ draft/published

## 2. เทคโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React + Vite | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับหน้าจอดู/แก้ไขตารางและข้อความแจ้งข้อผิดพลาด |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับ API ตรวจสอบ conflict, บันทึก Draft และเผยแพร่ |
| PostgreSQL | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้จัดเก็บข้อมูลอีเว้นต์ วง สถานที่ และตารางการแสดง |
| Draft/Published status model | ASM-03 | ใช้แยกสถานะเพื่อให้เกิดกระบวนการตรวจสอบก่อนเผยแพร่ |
| Order field separate from date/time | ASM-04 | จัดเก็บลำดับแยกจากวันเวลาเพื่อให้ผู้จัดงานปรับได้เอง |

## 3. โมเดิลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR / NFR |
|---|---|---|
| Event | id, title, start_date, end_date, status, created_by | PRE-SCM-01, FR-SCM-01 |
| ApprovedBand | id, event_id, band_id, name, approval_status, approved_at | FR-SCM-01 |
| Venue | id, name, capacity, status | FR-SCM-02, FR-SCM-06 |
| ScheduleItem | id, event_id, band_id, venue_id, schedule_date, start_time, end_time, order, status, created_by, created_at, published_at | FR-SCM-02, FR-SCM-04, ASM-03, ASM-04 |
| ConflictCheckResult | item_id, type, message, severity, source_value | FR-SCM-03, FR-SCM-05, FR-SCM-06 |
| ScheduleAuditLog | id, item_id, action, actor_id, old_status, new_status, created_at | FR-SCM-04, FR-SCM-07 |

หมายเหตุ:
- Order จะถูกเก็บแยกจาก schedule_date และ start_time เพื่อให้มีการ override แบบ manual
- status ของ ScheduleItem จะมีอย่างน้อย 2 ค่า: Draft และ Published
- ก่อนบันทึกหรือเผยแพร่ ระบบจะต้องตรวจสอบความถูกต้องของข้อมูลและ conflict ต่าง ๆ ตามลำดับที่กำหนด

## 4. ออกแบบฟังก์ชันและหน้าจอ

### 4.1 หน้าจอ
- หน้าเลือกวงได้รับอนุมัติสำหรับจัดตาราง
- ฟอร์มจัดตารางแต่ละวง: วัน, เริ่มเวลา, สถานที่, ลำดับ
- รายการ conflict summary ด้านล่างฟอร์ม บอกได้ว่า Time Conflict หรือ Venue Conflict มีอะไรบ้าง
- ปุ่ม Save as Draft และ Publish
- หน้าแสดงรายการ Draft และ Published ให้ผู้จัดงานตรวจสอบได้ก่อนเผยแพร่

### 4.2 API
- GET /api/events/{eventId}/approved-bands
  - คืนรายชื่อวงที่ได้รับอนุมัติ เพื่อให้เลือก
- POST /api/events/{eventId}/schedule/validate
  - รับข้อมูล date, start_time, venue_id, band_id, order
  - คืนค่า conflict list ที่ประกอบด้วย Time Conflict และ Venue Conflict พร้อมกันหากมี
- POST /api/events/{eventId}/schedule/draft
  - บันทึกเป็น Draft
  - ถ้ามี conflict ต้องปฏิเสธและคืน error list
- POST /api/events/{eventId}/schedule/publish
  - เปลี่ยน status จาก Draft เป็น Published
  - เปิดให้ผู้ชมและวงเห็นตารางเท่านั้นเมื่อกดขั้นตอนนี้
- GET /api/events/{eventId}/schedule
  - คืนตารางที่ Published สำหรับผู้ชม/วงดู
- GET /api/events/{eventId}/schedule/drafts
  - คืนรายการ Draft สำหรับผู้จัดงานตรวจสอบ

### 4.3 กระบวนการ validation
- ตรวจสอบสิทธิ์ผู้ใช้ก่อนเข้าถึง API และหน้าจัดตาราง
- ตรวจสอบข้อมูลบังคับ: วง, วัน, เริ่มเวลา, สถานที่, order
- ตรวจสอบ Time Conflict แบบแยกจาก Venue Conflict
- ถ้ามี conflict ให้รวบรวม error ทั้งหมดใน response เดียว ไม่หยุดที่ error แรก
- ถ้าไม่มี conflict ให้บันทึก Draft
- เฉพาะหลังจาก Save as Draft สำเร็จ และผู้จัดงานกด Publish เท่านั้น จึงจะเปลี่ยนเป็น Published

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| PRE-SCM-01 | ใช้ในการตรวจว่า event และ approved band มีอยู่ก่อนเปิดฟอร์มจัดตาราง | ใช้แล้ว |
| CON-ACC-01 | ใช้ใน middleware/authz และ route guard ของ schedule editor | ใช้แล้ว |
| ASM-02 | ใช้ใน validation logic ให้ตรวจ Time Conflict และ Venue Conflict พร้อมกัน | ใช้แล้ว |
| ASM-03 | ใช้ใน status flow Draft → Published | ใช้แล้ว |
| ASM-04 | ใช้ใน schema ScheduleItem และ UI form | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-SCM-01 | test_AC_SCM_01_save_draft_success | สร้างอีเว้นต์และวงที่ได้รับอนุมัติแล้ว กรอกวัน เริ่มเวลา สถานที่ และ order แล้วกด Save as Draft ตรวจว่าบันทึกสำเร็จและ status เป็น Draft |
| AC-SCM-02 | test_AC_SCM_02_time_conflict_blocked | จัดสรรเวลาให้วงใหม่ทับกับวงที่มีอยู่แล้ว ตรวจว่า response มี Time Conflict และไม่บันทึกข้อมูล |
| AC-SCM-03 | test_AC_SCM_03_combined_conflict_report | สร้างกรณีที่มี Time Conflict และ Venue Conflict พร้อมกัน ตรวจว่า response แสดงทั้งสองประเภทพร้อมกันในชุดเดียว |
| AC-SCM-04 | test_AC_SCM_04_not_visible_until_publish | บันทึก Draft แล้วเปิดตารางในมุมมองผู้ชม/วง ตรวจว่าที่ยังไม่เห็นข้อมูลจนกว่าจะกด Publish |
| AC-SCM-05 | test_AC_SCM_05_save_failure | จำลอง error จากฐานข้อมูลหรือ write failure ตรวจว่า UI แจ้ง “บันทึกไม่สำเร็จ” และไม่ประกาศ Published |
| AC-SCM-06 | test_AC_SCM_06_role_blocked | ลองเข้าถึงฟอร์มด้วยผู้ใช้ที่ไม่มีสิทธิ์ แล้วตรวจว่า API และ UI ปฏิเสธการเข้าถึง |

## 7. ลำดับงาน

1. กำหนด schema ของ Event, ApprovedBand, Venue, ScheduleItem และ status model Draft/Published
2. สร้าง authz guard สำหรับผู้จัดงานที่ได้รับสิทธิ์ก่อนเข้าถึงฟอร์มจัดตาราง
3. สร้างหน้า Schedule Editor พร้อมฟิลด์วัน/เวลา/สถานที่/order และปุ่ม Save as Draft / Publish
4. สร้าง validation service สำหรับ Time Conflict และ Venue Conflict แบบแยกแต่แสดงพร้อมกัน
5. สร้าง API save draft และ publish พร้อม workflow ที่ไม่เปิดให้สาธารณะเห็นจนกว่าจะ publish
6. สร้าง UI error handling และ banner รายการ conflict สำหรับผู้จัดงาน
7. ปรับ route/permission สำหรับผู้ชมและวงให้เห็นเฉพาะตารางที่ Published เท่านั้น
8. ดำเนินการทดสอบทุก AC และตรวจความสอดคล้องกับ spec ก่อนส่งให้ทีมตรวจ

## 8. สิ่งที่ยังค้างอยู่ / Open Questions
- Q-01 ยังไม่ได้ปิดความชัดเจนว่า Time Conflict ควรตรวจที่เงื่อนไขใด: สถานที่เดียวกันเวลาทับกัน วงเดียวกันเวลาทับกัน หรือทั้งสองอย่างพร้อมกัน
- ถ้าคำตอบ Q-01 เปลี่ยนแปลง แก้ไข logic validation และ test case ที่เกี่ยวข้องจะต้องอัปเดตตามนั้น
- NFR-PERF-01 ยังไม่ได้กำหนดค่าที่วัดได้ใน spec และยังต้องมีการตัดสินใจจากทีมก่อนเริ่มเขียนโค้ดจริง

## 9. ตัวชี้วัดความสำเร็จ
- ผู้จัดงานสามารถบันทึกตารางได้เป็น Draft และไม่เผยแพร่ก่อนกด Publish
- ระบบรายงานทั้ง Time Conflict และ Venue Conflict พร้อมกันเมื่อมีปัญหา
- ผู้ชมและวงเห็นตารางเฉพาะข้อมูลที่ Published เท่านั้น
- ผู้ใช้ที่ไม่มีสิทธิ์ไม่สามารถเข้าถึงฟอร์มหรือ API ได้
- ทุก Acceptance Criteria ของ UC-11 มี test case ที่ครอบคลุมแล้ว
