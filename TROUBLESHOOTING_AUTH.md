# Troubleshooting: Google Sheets MCP Auth & Connection Refused

## ปัญหาที่พบ
เมื่อเปิดใช้งาน Antigravity ใหม่ ตัวระบบพยายามจะยืนยันตัวตน (Authentication) เข้าถึง Google Sheets แต่แสดงหน้าจอ Error สีขาว หรือขึ้นคำว่า `ERR_CONNECTION_REFUSED` บนหน้าเบราว์เซอร์ `localhost` และในฝั่ง Antigravity เกิด Error `invalid character 'L' looking for beginning of value`. 

ปัญหานี้มักจะเกิดซ้ำๆ ทุกๆ 7 วัน

## สาเหตุ
1. **Token หมดอายุ (7 วัน):** แอป OAuth ใน Google Cloud ถูกตั้งค่า Publishing status เป็น **"Testing"** ทำให้ Token มีอายุการใช้งานเพียง 7 วัน และต้องล็อกอินใหม่เสมอ
2. **การทำงานเบื้องหลัง (Background Process):** เมื่อ Token หมดอายุ, MCP server พยายามจะรันคำสั่งล็อกอินและเปิดเว็บเบราว์เซอร์ แต่เนื่องจากมันทำงานเป็นแบบ Background ใน Antigravity พอร์ต `localhost` ที่เปิดรอรับ Token จึงปิดไปก่อน หรือเกิด Time out ทำให้การล็อกอินไม่สมบูรณ์
3. **ไฟล์ Token เก่าตกค้าง:** เมื่อระบบบันทึก Token ไม่สำเร็จ แต่ยังมีไฟล์ `.gsheets-server-credentials.json` (ที่หมดอายุ) ค้างอยู่ในโฟลเดอร์ `dist/` ทำให้เซิร์ฟเวอร์พยายามโหลดและใช้งานไฟล์เดิมเรื่อยๆ

## วิธีการแก้ไขอย่างถาวร

### 1. แก้ปัญหา Token หมดอายุทุก 7 วัน (ทำที่ Google Cloud)
1. ไปที่ [Google Cloud Console](https://console.cloud.google.com/) โดยใช้บัญชีที่ผูกโปรเจกต์ไว้
2. เข้าไปที่เมนู **APIs & Services** > **OAuth consent screen** (หน้าจอความยินยอม OAuth)
3. หากแอปมี **User type** เป็น **External**: เลื่อนหาหัวข้อ **Publishing status** และคลิกปุ่ม **PUBLISH APP** เพื่อเปลี่ยนสถานะจาก "Testing" เป็น **"In production"** 
*(หมายเหตุ: หาก User type เป็น **Internal** อยู่แล้ว โทเคนจะไม่มีวันหมดอายุ สามารถข้ามขั้นตอนนี้ได้)*

### 2. สร้างไฟล์ Token ใหม่ด้วยตัวเอง (Manual Auth)
เพื่อให้การล็อกอินเสร็จสมบูรณ์โดยไม่ถูกขัดจังหวะจาก Background process ของ Antigravity ให้ทำตามขั้นตอนดังนี้ผ่าน Terminal:

1. เปิดแอป **Terminal**
2. เข้าสู่โฟลเดอร์ของ Google Sheets MCP:
   ```bash
   cd /Users/Boss/Documents/Antigravity_Projects/_Global_MCP/google-sheets-mcp
   ```
3. **ลบไฟล์ Token เก่า** (ที่หมดอายุหรือมีปัญหา) ทิ้งก่อน เพื่อบังคับให้ระบบสร้างใหม่:
   ```bash
   rm dist/.gsheets-server-credentials.json
   ```
   *(หากขึ้น No such file or directory แสดงว่าไม่มีไฟล์ค้างอยู่ ให้ทำข้อถัดไปได้เลย)*
4. **รันคำสั่งสร้าง Token ใหม่ด้วยตัวเอง:**
   ```bash
   npm run build && node dist/index.js
   ```
5. ระบบจะเปิดหน้าต่างเบราว์เซอร์ ให้ทำการล็อกอินบัญชี Google และกด **Allow** 
6. เมื่อหน้าเว็บแจ้งว่าสำเร็จ (The authentication flow has completed) ให้กลับมาที่ Terminal 
7. กดปุ่ม `Control + C` เพื่อหยุดการทำงานของเซิร์ฟเวอร์ใน Terminal
8. กลับไปที่ Antigravity และกดปุ่ม **Reload** (Command + R) เพื่อให้ระบบโหลด Token ใหม่ไปใช้งาน

จากนั้นตัวเซิร์ฟเวอร์จะจดจำ Token นี้ไว้ในเครื่อง และสามารถใช้งานได้ตลอดไป (จนกว่าจะมีการเปลี่ยนรหัสผ่าน Google หรือยกเลิกสิทธิ์)
