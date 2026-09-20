# แผนงาน: จัดการข้อมูลระบบ (System Data Management)

## 1. สรุปแนวทาง
- ฟีเจอร์นี้ให้ผู้ดูแลระบบจัดการ system master/reference data ที่ฟีเจอร์อื่นต้องพึ่งพา เช่น venues, event categories/types, ประเภท/บทบาทการแสดง และ announcement categories เมื่อมีการใช้งาน โดยข้อมูลเหล่านี้ต้องไม่ hardcode ในโค้ด (FR-SYS-01, ASM-02)
- ระบบรองรับการสร้าง แก้ไข และปิดใช้งานข้อมูล reference เมื่อข้อมูลถูกอ้างอิงอยู่ โดยคงข้อมูลเก่าให้ใช้งานได้และไม่ให้รายการใหม่เลือกข้อมูลที่ปิดใช้งาน (FR-SYS-02, FR-SYS-04, NFR-DAT-01, ASM-03)
- การบันทึกต้องผ่าน validation และสิทธิ์ของผู้ดูแลระบบ พร้อมรองรับการยกเลิกก่อนบันทึกและการคงข้อมูลเดิมเมื่อบันทึกล้มเหลว (FR-SYS-03 ถึง FR-SYS-07, NFR-ACC-01)
- ระบบต้องมี backup อย่างน้อยวันละครั้ง เก็บอย่างน้อย 30 วัน RPO ไม่เกิน 24 ชั่วโมง และ RTO ไม่เกิน 4 ชั่วโมง (NFR-BKP-01, ASM-04)
- แผนนี้ไม่กำหนดแทนทีมในเรื่องฟิลด์/validation, ความหมายของ “ทันที”, ขอบเขต authorization และ rollback เพราะยังเป็น Q-04 ถึง Q-07

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| Web application frontend | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับเลือกประเภทข้อมูล รายการ ฟอร์ม และข้อความแจ้งผล |
| Backend API/service | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้ตรวจสิทธิ์ validation dependency และบันทึกข้อมูล |
| Reference-data storage | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้เก็บข้อมูล master/reference หลายประเภทและสถานะ active/deactivated |
| Reference-usage check | FR-SYS-02, NFR-DAT-01, ASM-03 | ตรวจจำนวน/รายการที่อ้างอิงก่อนป้องกัน hard delete และเสนอ deactivate |
| Backup and restore mechanism | NFR-BKP-01, ASM-04 | ทีมเลือกวิธีดำเนินการ โดยต้องวัด daily backup, retention, RPO และ RTO ได้ |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR / NFR |
|---|---|---|
| SystemReferenceData | id, reference_type, fields ตามประเภทข้อมูล, status, created_at, updated_at | FR-SYS-01, FR-SYS-02, FR-SYS-04, ASM-02, ASM-03 |
| ReferenceUsage | reference_data_id, referencing_type, referencing_id, usage_status | FR-SYS-02, NFR-DAT-01, ASM-03 |
| BackupRecord | backup_at, retention_until, result, recovery_point | NFR-BKP-01, ASM-04 |
| RestoreRecord | restore_started_at, restore_completed_at, recovery_point, result | NFR-BKP-01, ASM-04 |

หมายเหตุ:
- ฟิลด์เฉพาะของแต่ละ reference type และกฎ validation ยังรอ Q-04 จึงไม่กำหนดรายละเอียดแทนทีม
- ReferenceUsage ต้องทำให้ระบบแจ้งจำนวน/รายการที่อ้างอิงอยู่ก่อน deactivate และต้องรักษาการอ้างอิงของข้อมูลเก่า
- BackupRecord และ RestoreRecord เป็นแบบจำลองสำหรับตรวจสอบ SLA ในแผน ไม่ได้กำหนดรูปแบบ storage หรือเครื่องมือเฉพาะ

## 4. หน้าที่สำคัญ / workflow

### 4.1 หน้าจอและการจัดการ reference data
- หน้าเลือกประเภทข้อมูล master/reference ที่ผู้ดูแลระบบมีสิทธิ์จัดการ (FR-SYS-01, NFR-ACC-01)
- หน้ารายการข้อมูลของประเภทที่เลือก พร้อมสถานะใช้งาน/ปิดใช้งาน (FR-SYS-01, FR-SYS-04)
- ฟอร์มสร้างและแก้ไขข้อมูล พร้อม validation ก่อนบันทึก (FR-SYS-02, FR-SYS-03)
- การปิดใช้งานข้อมูลที่ถูกอ้างอิง พร้อมข้อความจำนวน/รายการที่อ้างอิง และการป้องกัน hard delete (FR-SYS-02, NFR-DAT-01, ASM-03)
- ปุ่มยกเลิกก่อนบันทึกที่ไม่เปลี่ยนข้อมูลเดิม (FR-SYS-05)
- ข้อความแจ้งข้อมูลไม่ถูกต้องหรือบันทึกไม่สำเร็จ โดยคงข้อมูลเดิม (FR-SYS-06, FR-SYS-07)

### 4.2 กฎข้อมูลและ dependency
1. เก็บ reference data ในระบบกลางที่ฟีเจอร์อื่นเรียกใช้ได้ และไม่ฝังค่าไว้ในโค้ด (FR-SYS-01, ASM-02)
2. ตรวจ validation ของประเภทข้อมูลก่อนเขียนข้อมูล (FR-SYS-03)
3. ตรวจ ReferenceUsage ก่อนการลบ/ปิดใช้งาน (FR-SYS-02, NFR-DAT-01)
4. หากมีการอ้างอิง ให้ป้องกัน hard delete แจ้งจำนวน/รายการ และใช้ deactivate แทน (ASM-03)
5. รายการเก่ายังคงอ้างอิงข้อมูลที่ปิดใช้งานได้ แต่ตัวเลือกสำหรับรายการใหม่ต้องกรองเฉพาะข้อมูลที่ active (FR-SYS-04, AC-SYS-01, AC-SYS-03)

### 4.3 Authorization และ transaction
1. ตรวจสิทธิ์ผู้ดูแลระบบก่อนเข้าถึงและแก้ไขข้อมูลระบบ (PRE-SYS-01, NFR-ACC-01)
2. ขอบเขตการป้องกัน menu/URL/API และสิทธิ์แยกตาม action ให้ดำเนินการตาม Q-06 โดยไม่เดาในแผนนี้
3. บันทึกข้อมูลเมื่อ validation ผ่านและผู้ดูแลระบบยืนยันเท่านั้น (FR-SYS-03, FR-SYS-04)
4. หากบันทึกล้มเหลว ต้องแจ้งผลและคงข้อมูลเดิมตาม FR-SYS-07; พฤติกรรม rollback เมื่อมีหลายรายการให้รอ Q-07
5. “ใช้งานได้ทันที” ให้ implement ตามคำตอบ Q-05 ก่อนกำหนดวิธีตรวจสอบจริง

### 4.4 Backup และ restore
- ตั้งรอบ backup อย่างน้อยวันละครั้ง และเก็บ backup ย้อนหลังอย่างน้อย 30 วัน (NFR-BKP-01, ASM-04)
- ตรวจ recovery point ให้ข้อมูลสูญหายไม่เกิน 24 ชั่วโมงตาม RPO (NFR-BKP-01, ASM-04)
- ทดสอบ restore ให้ระบบกลับมาใช้งานได้ภายใน 4 ชั่วโมงตาม RTO (NFR-BKP-01, ASM-04)
- บันทึกผล backup/restore เพื่อใช้ตรวจสอบว่า SLA ทำได้จริง โดยไม่กำหนดผู้ให้บริการหรือรูปแบบเครื่องมือในแผนนี้

## 5. ตารางตรวจ Constraints และ Assumptions

| Constraint / Assumption ID | ถูกนำไปใช้ในแผน | สถานะ |
|---|---|---|
| PRE-SYS-01 | ตรวจสิทธิ์ผู้ดูแลระบบก่อนเข้าถึงและแก้ไข system data | ใช้แล้ว |
| ASM-01 | ใช้ error handling ที่แจ้งผู้ดูแลระบบและคงข้อมูลเดิมเมื่อเกิด exception | ใช้แล้ว |
| ASM-02 | ใช้ reference-data model กลางและไม่ hardcode ข้อมูล | ใช้แล้ว |
| ASM-03 | ใช้ dependency check, hard-delete protection และ deactivate flow | ใช้แล้ว |
| ASM-04 | ใช้กำหนด backup/retention/RPO/RTO และการตรวจผล restore | ใช้แล้ว |
| Q-04 ถึง Q-07 | ไม่กำหนดพฤติกรรมแทนทีม; ต้องปิดคำถามก่อนรายละเอียด implementation | รอคำตอบ |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | วิธีทดสอบ |
|---|---|---|
| AC-SYS-01 | test_AC_SYS_01_reference_data_crud_and_deactivate | ใช้ Admin สร้าง แก้ไข และปิดใช้งาน reference data ตรวจว่าข้อมูลปิดใช้งานไม่ถูกเลือกในรายการใหม่ และเมื่อมีการอ้างอิงระบบไม่ hard delete พร้อมแจ้งจำนวน/รายการ |
| AC-SYS-02 | test_AC_SYS_02_invalid_reference_data_blocked | กรอกข้อมูลไม่ถูกต้องตาม validation ที่ทีมกำหนด ตรวจว่าระบบแจ้งก่อนบันทึกและข้อมูลไม่เปลี่ยน |
| AC-SYS-03 | test_AC_SYS_03_latest_data_and_legacy_reference | บันทึกข้อมูลใหม่แล้วเรียกใช้จากฟังก์ชันอื่น ตรวจว่าเห็นข้อมูลล่าสุดตามความหมายของ “ทันที” ที่จะปิดใน Q-05 และข้อมูลเก่ายังอ้างอิงรายการที่ปิดใช้งานได้ |
| AC-SYS-04 | test_AC_SYS_04_cancel_before_save | เริ่มแก้ไขข้อมูลแล้วกดยกเลิก ตรวจว่าข้อมูลเดิมไม่เปลี่ยน |
| AC-SYS-05 | test_AC_SYS_05_save_failure_preserves_old_data | จำลองบันทึกล้มเหลว ตรวจว่าระบบแจ้งบันทึกไม่สำเร็จและข้อมูลเดิมยังคงอยู่ |
| AC-SYS-06 | test_AC_SYS_06_unauthorized_system_data_access | ใช้ผู้ไม่มีสิทธิ์พยายามเข้าถึงเมนู/ช่องทางที่กำหนดตาม Q-06 ตรวจว่าระบบปฏิเสธ |

หมายเหตุ: NFR-BKP-01 มีเกณฑ์ daily/30 วัน/RPO/RTO แล้ว แต่ยังไม่มี AC เฉพาะใน spec จึงไม่สร้าง test ID ใหม่ในแผนนี้ ต้องให้ทีมปรับ spec ก่อน implementation หากต้องการ traceability ระดับ acceptance criteria

## 7. ลำดับงาน

1. ยืนยันประเภท reference data ที่ฟีเจอร์อื่นใช้งานและ mapping ของแต่ละประเภท (FR-SYS-01, ASM-02)
2. ออกแบบ schema กลางและสถานะ active/deactivated พร้อม reference usage tracking (FR-SYS-01, FR-SYS-02, NFR-DAT-01)
3. ปิด Q-04 เพื่อกำหนดฟิลด์บังคับและ validation ของแต่ละประเภท
4. สร้างหน้ารายการ ฟอร์มสร้าง/แก้ไข และ deactivate flow พร้อม dependency message (FR-SYS-01 ถึง FR-SYS-04)
5. สร้าง cancel-before-save และ error handling ที่คงข้อมูลเดิม (FR-SYS-05 ถึง FR-SYS-07)
6. ปิด Q-05 ถึง Q-07 แล้วกำหนด read propagation, authorization scope และ rollback behavior
7. สร้าง backup/restore workflow และตรวจ daily schedule, retention, RPO และ RTO (NFR-BKP-01, ASM-04)
8. ทดสอบ AC-SYS-01 ถึง AC-SYS-06 และตรวจ traceability ก่อนส่งให้ทีมตรวจ

## 8. สิ่งที่ยังไม่ทำ / Open Questions

- Q-04 ยังไม่ระบุฟิลด์บังคับและ validation ของ reference data แต่ละประเภท
- Q-05 ยังไม่ระบุขอบเขตของคำว่า “ใช้งานได้ทันที”
- Q-06 ยังไม่ระบุขอบเขตการป้องกัน menu/URL/API และสิทธิ์แยกตาม action
- Q-07 ยังไม่ระบุ rollback เมื่อการบันทึกหลายรายการล้มเหลว
- NFR-BKP-01 ยังไม่มี AC เฉพาะสำหรับตรวจ backup/restore SLA

## 9. ตัวชี้วัดความสำเร็จ

- ฟีเจอร์อื่นเรียกใช้ system master/reference data จากระบบกลางได้โดยไม่ hardcode
- Admin สร้าง แก้ไข และปิดใช้งาน reference data ได้ตามสิทธิ์
- ข้อมูลที่ถูกอ้างอิงไม่ถูก hard delete มีข้อความจำนวน/รายการอ้างอิง และข้อมูลเก่ายังใช้งานได้
- ข้อมูลที่ปิดใช้งานไม่ปรากฏให้เลือกใช้กับรายการใหม่
- การ validation, cancel, save failure และ authorization ทำงานตาม AC
- ระบบทำ backup daily เก็บ 30 วัน และผ่าน RPO 24 ชั่วโมง/RTO 4 ชั่วโมงตาม NFR-BKP-01
