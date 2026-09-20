# แผนงาน: สมัครสมาชิก / เข้าสู่ระบบ (Authentication)

## 1. สรุปแนวทาง
- ฟีเจอร์นี้สร้างระบบสมัครสมาชิกและเข้าสู่ระบบสำหรับผู้ใช้ทุกบทบาท โดยมีการแยกผู้ใช้ตามบทบาทและสิทธิ์การใช้งาน เช่น นักศึกษา ผู้ชม วงดนตรี ผู้จัดงาน และ Admin
- ผู้ใช้จะเข้าถึงหน้า Auth สำหรับสมัครหรือเข้าสู่ระบบก่อนกดเข้าสู่หน้าหลัก โดยยึดหลักว่าไม่อนุญาตให้ผู้จัดงานและ Admin สมัครเองผ่านฟอร์มทั่วไป
- ระบบจะใช้ React + Vite สำหรับหน้าบ้านและ Python FastAPI สำหรับ API ของการสมัคร เข้าสู่ระบบ ตรวจสอบสิทธิ์ และจัดการ session/security
- การตรวจสอบข้อมูลจะเกิดทั้งฝั่ง frontend และ backend เพื่อให้มั่นใจว่า FR-AUTH-02, FR-AUTH-04, NFR-SEC-01 และ NFR-PERF-01 ทำงานตามเงื่อนไขที่ได้กำหนด
- แผนงานนี้จะออกแบบให้รองรับการเข้าสู่ระบบของ Admin ผ่านหน้า Auth เดียวกัน โดยมีการกำหนดสิทธิ์ผ่าน UC-14 และท้ายสุดนำไปหน้าหลักตามบทบาทที่มี

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React + Vite | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับหน้า Auth, Dashboard และ route ตามบทบาท |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับ API ของ register/login/session/permission |
| Session-based auth หรือ JWT option | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับยืนยันสถานะผู้ใช้และกำหนดสิทธิ์หลังเข้าสู่ระบบ |
| HTTPS TLS 1.2+ | NFR-SEC-01 | บังคับใช้ทุกหน้าเพื่อให้การส่งข้อมูลมีความปลอดภัย |
| Password hashing + salt | NFR-SEC-01 | ป้องกันการเก็บรหัสผ่านแบบ plain text |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR / NFR |
|---|---|---|
| User | id, email, password_hash, salt, full_name_or_band_name, role, status, created_at, last_login_at, failed_login_count, lock_until | FR-AUTH-02, FR-AUTH-03, FR-AUTH-05, NFR-SEC-01 |
| StudentProfile | user_id, student_id, university_code, created_at | FR-AUTH-02, FR-AUTH-03 |
| AudienceProfile | user_id, phone_number, created_at | FR-AUTH-02, FR-AUTH-03 |
| BandProfile | user_id, band_name, contact_name, phone_number, member_count, genre, performance_list, created_at | FR-AUTH-02, FR-AUTH-03, FR-AUTH-05 |
| RolePermission | role, permission_code, granted_at | FR-AUTH-05 |
| Session | session_id, user_id, issued_at, expires_at, last_seen_at | FR-AUTH-03, NFR-SEC-01 |

หมายเหตุ:
- จะไม่มีการเก็บรหัสผ่านแบบ plain text ตาม NFR-SEC-01
- ฟิลด์ที่เป็นข้อมูลเฉพาะบทบาทจะถูกเก็บภายใต้ corresponding profile table เพื่อให้แน่ใจว่าผู้ใช้แต่ละบทบาทมีโครงสร้างข้อมูลที่ชัดเจน
- การกรอกข้อมูลรายละเอียดวง เช่น สมาชิก แนวเพลง เพลงที่แสดง จะถูกจัดการภายหลังผ่าน UC-07 และ UC-06 ตาม ASM-02

## 4. API / หน้าจอ

- GET /auth | หน้าเลือกสมัครสมาชิกหรือเข้าสู่ระบบ | รองรับ FR-AUTH-01
- POST /api/auth/register | Input: email, password, confirmPassword, role, profileData | Output: success/error | รองรับ FR-AUTH-02, FR-AUTH-03, FR-AUTH-04
- POST /api/auth/login | Input: email, password | Output: authenticated session + redirect target | รองรับ FR-AUTH-03, FR-AUTH-04, FR-AUTH-06, NFR-SEC-01
- POST /api/auth/logout | Input: session_id | Output: clear session | รองรับ FR-AUTH-03
- GET /api/auth/me | Output: user info + role + permissions | รองรับ FR-AUTH-05
- GET /dashboard | หน้าแสดงหลักตามบทบาท | รองรับ FR-AUTH-03, FR-AUTH-05
- POST /api/auth/lockout-check | Input: user_id | Output: locked status | รองรับ NFR-SEC-01

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| PRE-AUTH-01 | ใช้ใน route /auth และ middleware ตรวจสอบสถานะเข้าสู่ระบบก่อนเข้าหน้า dashboard หรือฟังก์ชันภายในระบบ | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-AUTH-01 | test_AC_AUTH_01_register_success | สร้างผู้ใช้ใหม่ด้วยบทบาทนักศึกษา/ผู้ชม/วงดนตรีที่มีข้อมูลครบถ้วน ถูกต้อง แล้วตรวจว่า ระบบบันทึกข้อมูลและ redirect ไปหน้าหลักของผู้ใช้ |
| AC-AUTH-02 | test_AC_AUTH_02_login_success | Login ด้วยบัญชีมีอยู่และข้อมูลถูกต้อง แล้วตรวจว่า session ถูกสร้างและผู้ใช้เห็นหน้าหลักที่ถูกต้อง |
| AC-AUTH-03 | test_AC_AUTH_03_invalid_input_shows_error | ส่งข้อมูลที่ไม่ถูกต้อง เช่น email ผิด หรือ password สั้น แล้วตรวจว่า UI แจ้งเตือนและผู้ใช้สามารถแก้ไขได้ |
| AC-AUTH-04 | test_AC_AUTH_04_role_permissions | เข้าสู่ระบบด้วยบทบาทต่าง ๆ แล้วตรวจว่าเข้าถึงฟังก์ชันได้เฉพาะตามสิทธิ์ที่กำหนด |
| AC-AUTH-05 | test_AC_AUTH_05_db_unavailable_block_login | จำลองฐานข้อมูลหรือ auth service ไม่พร้อม แล้วตรวจว่า login ถูกปฏิเสธและแสดงข้อความไม่สำเร็จ |
| AC-AUTH-06 | test_AC_AUTH_06_response_time_under_3s | จำลอง 50 ผู้ใช้พร้อมกัน ส่ง request สมัคร/เข้าสู่ระบบ พร้อมวัด p95 ว่าไม่เกิน 3 วินาที |

## 7. ลำดับงาน

1. กำหนด route และ layout หน้า Auth พร้อม UX สำหรับสมัคร / login และการแสดงข้อความผิดพลาด (FR-AUTH-01, FR-AUTH-04)
2. กำหนด schema ของ User / Profile / Permission / Session และกฎ validation สำหรับบทบาทต่าง ๆ (FR-AUTH-02, FR-AUTH-05, NFR-SEC-01)
3. สร้าง API สมัครสมาชิกพร้อมตรวจข้อมูลซ้ำ ไม่ให้ผู้จัดงาน/Admin สมัครเอง และบันทึกผู้ใช้ใหม่ (FR-AUTH-02, FR-AUTH-03, ASM-02, ASM-03, ASM-04)
4. สร้าง API เข้าสู่ระบบ พร้อมตรวจ password, lockout, session, และ error message ที่ไม่ระบุชื่อผู้ใช้หรือรหัสผ่าน (FR-AUTH-03, FR-AUTH-06, NFR-SEC-01)
5. สร้าง middleware ตรวจสถานะผู้ใช้และสิทธิ์บทบาทเพื่อกำหนดหน้าหลักและเมนูที่แสดงตามบทบาท (FR-AUTH-03, FR-AUTH-05, ASM-07)
6. จัดการกรณีฐานข้อมูลไม่พร้อมหรือ auth service ล้มเหลว ให้ปฏิเสธการเข้าสู่ระบบและแจ้งผู้ใช้ (FR-AUTH-06)
7. ดำเนินการทดสอบประสิทธิภาพแบบ p95 สำหรับ 50 คน พร้อมตรวจเวลาการตอบสนอง (NFR-PERF-01, AC-AUTH-06)
8. ทดสอบ end-to-end ครบทุก AC และตรวจความสอดคล้องกับ spec ก่อนส่งให้ทีมตรวจ (AC-AUTH-01 ถึง AC-AUTH-06)

## 8. สิ่งที่ยังไม่ทำ
- ไม่มี Open Questions ที่ค้างอยู่ใน spec หลัง Draft v2
- ส่วนที่เกี่ยวข้องกับข้อสันนิษฐานหลังจากนี้จะยังไม่สร้างจนกว่า team จะตัดสินใจเพิ่มเติมเกี่ยวกับรายละเอียดเฉพาะด้านธุรกิจ เช่น จำนวนสมาชิกวงจริงหรือนโยบายรหัสผ่านเฉพาะของหน่วยงาน

