# postgresql-after-install
ถ้าสเปกเริ่มต้นเป็น 4 vCPU, RAM 8GB, Storage 200GB การรับมือกับ 500 Concurrent Connections ตรงๆ เข้า Database Node จะทำให้ RAM หมดทันที (Out of Memory - OOM) และ CPU ติดคอขวดที่ Context Switching
ดังนั้น การใช้ Connection Pooler (PgBouncer) จึงกลายเป็น ข้อบังคับ (Mandatory) ไม่ใช่แค่ออปชันเสริมสำหรับสเปกนี้

## 1. ปรับสถาปัตยกรรม Connection (PgBouncer + PostgreSQL)
**- ห้ามตั้ง max_connections บน PostgreSQL เกิน 60 - 80**
  - ด้วย 4 vCPU จำนวน active connections ที่ CPU สลับการทำงานได้ดีที่สุดโดยไม่ drop throughput อยู่ที่ประมาณ (4 x 2) ถึง 40-60 connections
  - ให้ PgBouncer รับ 500 connections จากฝั่ง Client แล้วส่งต่อเข้า DB เพียง 40–60 pool connections เท่านั้น (โหมด pool_mode = transaction)

## 2. การปรับแต่ง postgresql.conf สำหรับ RAM 8GB
สูตรคำนวณถูกบีบให้กระชับ เพื่อเหลือพื้นที่ให้ OS, Background Workers และ Query Sort/Hash:
Parameter,ค่าที่เหมาะสม (RAM 8GB),เหตุผล
max_connections,80,จำกัดไม่ให้แย่งกันใช้ CPU และกัน Memory บวม
shared_buffers,2GB,25% ของ RAM ทั้งหมดสำหรับบัฟเฟอร์หลัก
effective_cache_size,6GB,75% ของ RAM ให้ Query Planner คำนวณ Index scan แม่นยำ
work_mem,8MB – 16MB,คำนวณจาก: (8GB - 2GB) / (80 x 2-3 queries) กัน OOM ยามมี Sort ซับซ้อน
maintenance_work_mem,512MB,เพียงพอสำหรับ Index creation และ Autovacuum โดยไม่กิน RAM เครื่องจนค้าง
checkpoint_completion_target,0.9,เกลี่ย I/O ให้เรียบ ป้องกัน Disk spike บน Storage 200GB
max_wal_size,4GB – 8GB,ห้ามตั้งสูงเกินไป เพื่อไม่ให้กินพื้นที่ Storage 200GB เร็วเกินไป
min_wal_size,1GB,ขนาดพื้นฐานของ WAL
random_page_cost,1.1,เหมาะสำหรับ SSD/NVMe Cloud Volume
wal_compression,on (หรือ lz4),ประหยัดทั้ง I/O และพื้นที่ Storage 200GB
