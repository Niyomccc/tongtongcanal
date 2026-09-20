# แผนงาน: ประกาศข่าวสาร (Announcements)

## 1. สรุปแนวทาง
- ฟีเจอร์นี้ออกแบบให้ผู้จัดงานสามารถสร้างประกาศแบบร่างหรือเผยแพร่ได้ โดยประกาศแต่ละฉบับต้องมีข้อมูลพื้นฐานที่จำเป็น เช่น หัวข้อ รายละเอียด และ event_id หรือ target_group เพื่อกำหนดกลุ่มผู้อ่าน
- ระบบจะให้ผู้ใช้เห็นประกาศเฉพาะกลุ่มที่เกี่ยวข้องเท่านั้น และเมื่อมีการเผยแพร่หรือเปลี่ยนแปลง จะต้องมี in-app notification เป็นขั้นต่ำเพื่อให้ผู้ใช้ทราบทันที
- กลุ่มผู้ใช้ที่เกี่ยวข้องอาจเป็นผู้เข้าร่วมอีเว้นต์หรือสมาชิกใน target_group ตามที่ผู้จัดงานกำหนด และการส่ง email/SMS ใช้เป็นทางเลือกเสริมตามความสำคัญของการเปลี่ยนแปลง
- เมื่อประกาศเผยแพร่แล้ว ผู้จัดงานสามารถแก้ไข ซ่อน หรือ soft delete ได้ โดยต้องมี edit log เพื่อส่งเสริมความโปร่งใสและรักษาความถูกต้องของข้อมูล
- แผนนี้ยึดมั่นว่า spec.md คือแหล่งความจริง และจะไม่เพิ่มฟีเจอร์ที่ไม่ได้ระบุไว้ เช่น ระบบจดหมายข่าวหรือการคัดกรองข่าวตามความชอบส่วนตัว ที่อยู่นอกขอบเขต

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| Web application frontend | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับฟอร์มสร้างประกาศ, รายการข่าว, และการแจ้งเตือนในแอป |
| Backend API | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับสร้าง/แก้ไข/เผยแพร่ประกาศและจัดการ notification |
| Database schema แบบ relational | ทีมเลือกเอง ไม่ได้มาจาก spec | เหมาะสำหรับบันทึกประกาศ, กลุ่มผู้รับ, และประวัติการแก้ไข |
| In-app notification service | FR-ANN-03 | เป็นขั้นต่ำที่บังคับให้ส่งทันทีเมื่อประกาศเผยแพร่ |
| Optional email/SMS provider | FR-ANN-03 | ใช้ในกรณีที่ว่าจ้างหรือความสำคัญสูงตามนโยบายของทีม |
| Access control middleware | FR-ANN-06, NFR-ACC-01, CON-ACC-01 | ตรวจสิทธิ์ก่อนให้สร้าง/เผยแพร่ประกาศ |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ ID |
|---|---|---|
| Announcement | id, title, content, status, created_by, created_at, published_at, hidden_at, deleted_at, event_id, target_group | FR-ANN-01, FR-ANN-02, FR-ANN-04, FR-ANN-05 |
| AnnouncementAudience | announcement_id, user_id, group_id, is_targeted, viewed_at | FR-ANN-03, NFR-DAT-01 |
| Notification | id, announcement_id, recipient_user_id, channel, sent_at, read_at, status | FR-ANN-03 |
| AnnouncementEditLog | id, announcement_id, edited_by, old_values, new_values, edited_at, reason | FR-ANN-04 |
| UserRolePermission | user_id, role, permission_code | FR-ANN-06, NFR-ACC-01, CON-ACC-01 |

หมายเหตุ:
- status ควรมีค่า เช่น draft, published, hidden, soft_deleted เพื่อรองรับ FR-ANN-04
- event_id หรือ target_group เป็นฟิลด์บังคับเพื่อให้การกำหนดผู้รับข้อมูลชัดเจนตาม FR-ANN-01 และ FR-ANN-02
- การบันทึก edit log จะต้องไม่ลบข้อมูลเดิมก่อนเผยแพร่ เพื่อให้สามารถแสดงประวัติการเปลี่ยนแปลงได้ตาม FR-ANN-04
- การเก็บ audience ควรแยกจากเนื้อหา announcement เพื่อให้สามารถแสดงเฉพาะผู้ใช้ที่เกี่ยวข้องตาม NFR-DAT-01

## 4. หน้าที่สำคัญ / API / UI

### 4.1 หน้าและ workflow
- GET /announcements/new | หน้าแบบร่าง/สร้างประกาศ | รองรับ FR-ANN-01, FR-ANN-02
- POST /api/announcements | สร้างประกาศแบบ draft | รองรับ FR-ANN-01, FR-ANN-02, FR-ANN-04
- PUT /api/announcements/:id | แก้ไขประกาศที่ยังเป็น draft หรือ published | รองรับ FR-ANN-04
- POST /api/announcements/:id/publish | เผยแพร่ประกาศให้กลุ่มที่เกี่ยวข้องเห็น | รองรับ FR-ANN-03, FR-ANN-05
- POST /api/announcements/:id/hide | ซ่อนประกาศแบบ soft hide | รองรับ FR-ANN-04
- DELETE /api/announcements/:id | ทำ soft delete แทนลบถาวร | รองรับ FR-ANN-04
- GET /api/announcements?event_id=... หรือ target_group=... | ดึงประกาศที่เกี่ยวข้องสำหรับผู้ใช้ | รองรับ NFR-DAT-01
- GET /api/notifications | ดูรายการ notification ที่ส่งแล้ว | รองรับ FR-ANN-03

### 4.2 การทำงานหลัก
1. ผู้จัดงานเข้าสู่ระบบและมีสิทธิ์ประกาศ (PRE-ANN-01, FR-ANN-06)
2. ระบบตรวจสอบข้อมูลพื้นฐานก่อนอนุญาตให้เผยแพร่ เช่น หัวข้อและรายละเอียดไม่ว่าง, event_id หรือ target_group ชัดเจน
3. เมื่อกดบันทึกเป็นแบบร่าง ระบบจะบันทึก status = draft และไม่เปิดให้ผู้ใช้เห็น
4. เมื่อกดเผยแพร่ ระบบคำนวณ audience จาก event_id หรือ target_group แล้วส่ง in-app notification ให้ผู้ใช้ที่เกี่ยวข้อง
5. ถ้าเผยแพร่ไม่สำเร็จ ระบบจะหยุดการเผยแพร่และแสดงข้อความแจ้งผู้จัดงาน พร้อมไม่เปิดเผยประกาศให้ผู้ใช้เห็น
6. หากมีการแก้ไข/ซ่อน/soft delete หลังเผยแพร่ จะบันทึกประวัติไว้ใน AnnouncementEditLog

## 5. ตารางตรวจ Constraints

| Constraint / Requirement ID | ถูกนำไปใช้ในแผน | สถานะ |
|---|---|---|
| PRE-ANN-01 | ผู้จัดงานต้องเข้าสู่ระบบก่อนเข้าหน้าและก่อนเรียก API สร้าง/เผยแพร่ประกาศ | ใช้แล้ว |
| CON-ACC-01 | Middleware ตรวจสิทธิ์ก่อนอนุญาตให้สร้าง/เผยแพร่ประกาศ | ใช้แล้ว |
| FR-ANN-01 | ฟอร์มประกาศและ schema ต้องมีหัวข้อ รายละเอียด event_id/target_group | ใช้แล้ว |
| FR-ANN-02 | validation rules ก่อน publish | ใช้แล้ว |
| FR-ANN-03 | audience targeting + in-app notification + optional email/SMS | ใช้แล้ว |
| FR-ANN-04 | draft / hidden / soft delete + edit log | ใช้แล้ว |
| FR-ANN-05 | publish failure handling | ใช้แล้ว |
| FR-ANN-06 | role-based access control | ใช้แล้ว |
| NFR-DAT-01 | query logic แสดงข่าวสารที่ถูกต้องและเฉพาะกลุ่มที่เกี่ยวข้อง | ใช้แล้ว |
| NFR-ACC-01 | permission check | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | วิธีทดสอบ |
|---|---|---|
| AC-ANN-01 | test_AC_ANN_01_create_announcement_success | ผู้จัดงานเข้าสู่ระบบแล้วกรอกหัวข้อ รายละเอียด และ event_id หรือ target_group แล้วตรวจว่าระบบบันทึกประกาศสำเร็จ |
| AC-ANN-02 | test_AC_ANN_02_publish_targets_correct_users | สร้างประกาศที่ข้อมูลถูกต้องและระบุ target_group หรือ event_id ให้ชัดเจน แล้วตรวจว่า announcement ถูกเผยแพร่สำหรับผู้ใช้ที่เกี่ยวข้อง และส่ง in-app notification ทันที |
| AC-ANN-03 | test_AC_ANN_03_view_only_relevant_users | โหลดประกาศด้วยผู้ใช้ที่เกี่ยวข้องและผู้ใช้ที่ไม่เกี่ยวข้อง ตรวจว่าผู้ใช้ที่เกี่ยวข้องเห็นประกาศแต่ไม่เกี่ยวข้องไม่เห็น |
| AC-ANN-04 | test_AC_ANN_04_draft_and_edit_log | บันทึกเป็นแบบร่างและหลังเผยแพร่เลือกแก้ไข/ซ่อนหรือ soft delete แล้วตรวจว่า status อัปเดตและมี edit log ถูกบันทึก |
| AC-ANN-05 | test_AC_ANN_05_publish_failure_blocked | จำลองระบบไม่สามารถเผยแพร่ได้ แล้วตรวจว่า API คืน error และไม่มีประกาศปรากฏแก่ผู้ใช้ |
| AC-ANN-06 | test_AC_ANN_06_unauthorized_user_blocked | ใช้บัญชีที่ไม่ใช่ผู้จัดงานที่มีสิทธิ์พยายามสร้างหรือเผยแพร่ประกาศ แล้วตรวจว่าระบบปฏิเสธ |

## 7. ลำดับงาน

1. กำหนด schema ของ Announcement, AnnouncementAudience, Notification, AnnouncementEditLog และ role/permission (FR-ANN-01, FR-ANN-02, FR-ANN-03, FR-ANN-04)
2. สร้างหน้าแบบร่าง/ฟอร์มสร้างประกาศ พร้อม validation สำหรับหัวข้อ รายละเอียด และ event_id/target_group (FR-ANN-01, FR-ANN-02)
3. สร้าง API บันทึกแบบร่างและจัดการสถานะ draft (FR-ANN-04)
4. สร้าง API เผยแพร่ประกาศพร้อม audience targeting และ in-app notification (FR-ANN-03, NFR-DAT-01)
5. เพิ่มตัวเลือกส่ง email/SMS ตามความสำคัญของการเปลี่ยนแปลง โดยเป็นทางเลือกเสริม ไม่ใช่ข้อบังคับ (FR-ANN-03)
6. สร้าง logic สำหรับแก้ไข/ซ่อน/soft delete และบันทึก edit log (FR-ANN-04)
7. สร้าง error handling สำหรับการเผยแพร่ไม่สำเร็จ และยืนยันไม่แสดงประกาศให้ผู้ใช้เห็น (FR-ANN-05)
8. สร้าง middleware และ permission enforcement สำหรับผู้จัดงานเท่านั้น (FR-ANN-06, NFR-ACC-01, CON-ACC-01)
9. ทดสอบ end-to-end ทุก AC และตรวจความสอดคล้องกับ spec ก่อนส่งให้ทีมตรวจ

## 8. สิ่งที่ยังไม่ทำ / Open Questions
- ไม่มี Open Questions เหลืออยู่ใน spec หลังการตอบคำถามและปรับ Draft v2
- ไม่ได้สร้างฟีเจอร์ที่อยู่นอก scope เช่น การสร้าง/แก้ไขอีเว้นต์เอง หรือการแจ้งผลการพิจารณาใบสมัครให้วง
- ทุกชิ้นงานในแผนนี้จะต้อง map กลับไปหา ID ใน spec อย่างชัดเจน เช่น function, route, schema, test จะต้องระบุว่ารองรับ FR-ANN-XX หรือ AC-ANN-XX

## 9. ข้อสรุปด้านการตรวจสอบ
- แผนนี้จะมุ่งเน้นที่ “ประกาศและกลุ่มผู้อ่าน” เป็นศูนย์กลางของฟีเจอร์ และคงการตัดสินใจที่สำคัญจาก spec ไว้ครบทุกประเด็น เช่น
  - ต้องมี event_id หรือ target_group
  - in-app notification เป็นขั้นต่ำ
  - email/SMS เป็นทางเลือกเสริม
  - post-publish edit/hide/soft delete ถูกอนุญาตพร้อม edit log
- การปฏิบัติตาม spec ยังคงเป็นเงื่อนไขสำคัญก่อนเริ่มพัฒนาใด ๆ ต่อไป
