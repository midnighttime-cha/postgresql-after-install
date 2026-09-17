# ตั้งค่า PostgrSQL สำหรับรองรับการทำงาน
> นิยามว่าหากเรามี Server spec
> - 4 vCPU
> - RAM 8GB
> - Storage 200GB
> ต้องรองรับกับ 500 Concurrent Connections

การที่จะต้องให้ PostgreSQL รองรับ 500 concurrent แบบตรงๆ เข้า Database Node อาจจะทำให้ RAM หมดทันที (Out of Memory - OOM) และ CPU ติดคอขวดที่ Context Switching
ดังนั้น การใช้ Connection Pooler (PgBouncer) จึงกลายเป็น ข้อบังคับ (Mandatory) ไม่ใช่แค่ออปชันเสริมสำหรับสเปกนี้

## 1. ปรับสถาปัตยกรรม Connection (PgBouncer + PostgreSQL)
***ห้ามตั้ง max_connections บน PostgreSQL เกิน 60 - 80***
  - ด้วย 4 vCPU จำนวน active connections ที่ CPU สลับการทำงานได้ดีที่สุดโดยไม่ drop throughput อยู่ที่ประมาณ (4 x 2) ถึง 40-60 connections
  - ให้ PgBouncer รับ 500 connections จากฝั่ง Client แล้วส่งต่อเข้า DB เพียง 40–60 pool connections เท่านั้น (โหมด pool_mode = transaction)

## 2. การปรับแต่ง postgresql.conf สำหรับ RAM 8GB
สูตรคำนวณถูกบีบให้กระชับ เพื่อเหลือพื้นที่ให้ OS, Background Workers และ Query Sort/Hash:
- `max_connections` 80,จำกัดไม่ให้แย่งกันใช้ CPU และกัน Memory บวม
- `shared_buffers` 2GB,25% ของ RAM ทั้งหมดสำหรับบัฟเฟอร์หลัก
- `effective_cache_size` 6GB,75% ของ RAM ให้ Query Planner คำนวณ Index scan แม่นยำ
- `work_mem` 8MB – 16MB,คำนวณจาก: (8GB - 2GB) / (80 x 2-3 queries) กัน OOM ยามมี Sort ซับซ้อน
- `maintenance_work_mem` 512MB,เพียงพอสำหรับ Index creation และ Autovacuum โดยไม่กิน RAM เครื่องจนค้าง
- `checkpoint_completion_target` 0.9,เกลี่ย I/O ให้เรียบ ป้องกัน Disk spike บน Storage 200GB
- `max_wal_size` 4GB – 8GB,ห้ามตั้งสูงเกินไป เพื่อไม่ให้กินพื้นที่ Storage 200GB เร็วเกินไป
- `min_wal_size` 1GB,ขนาดพื้นฐานของ WAL
- `random_page_cost` 1.1,เหมาะสำหรับ SSD/NVMe Cloud Volume
- `wal_compression` on (หรือ lz4),ประหยัดทั้ง I/O และพื้นที่ Storage 200GB
## 3. แผนรับมือข้อจำกัด Storage 200GB (สำหรับเป้าหมายใช้งานยาว)
พื้นที่ 200GB ถือว่าจำกัดมากสำหรับการใช้งานระยะยาวหลายปี หากไม่มีการจัดการ:
- แยกที่เก็บ WAL Archive / Backup ออกจากเครื่องทันที:
  - ห้ามเก็บไฟล์ Backup (pg_dump หรือ pgBackRest stanza) ไว้บน Disk เดียวกันกับ Data Directory
  - ตั้งค่าส่ง WAL files และ Snapshot ไปยัง Cloud Object Storage (เช่น S3/MinIO) ทันที
- ตั้ง Autovacuum ให้ดักเก็บพื้นที่บ่อยขึ้น:
```bash
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_limit = 1000
autovacuum_vacuum_cost_delay = 2ms
```
- ควบคุม Data Growth:
  - ตาราง Logs/Audit ต้องมี Retention Policy (เช่น ลบหรือ Move ออกทุก 3–6 เดือน)
  - หลีกเลี่ยงการเก็บภาพหรือไฟล์ Binary (BLOB/bytea) ลงในตารางโดยตรง ให้เก็บเป็น Object Storage แล้วอ้างอิงด้วย URL/Path

## 4. ตรวจสอบการใช้งานจริง (Verification)
หลัง Deploy และปล่อยโหลด สามารถเช็กสถานะการใช้ Resource ได้จากคำสั่ง:
```bash
# ดู Memory และ CPU บน OS
free -m
top -b -n 1 | head -n 15
```

```sql
-- ตรวจสอบ Memory work_mem ต่อ process ว่าไม่มี Query ที่ทำ Temp File ลง Disk บ่อยเกินไป
SELECT query, temp_bytes, calls 
FROM pg_stat_statements 
WHERE temp_bytes > 0 
ORDER BY temp_bytes DESC LIMIT 5;
```
(ถ้าพบ temp_bytes สูงมาก แสดงว่าค่า work_mem = 8MB อาจเล็กไปสำหรับบาง Complex Query ให้พิจารณาจูน Query นั้นๆ หรือเพิ่มเป็น 16MB)

## 5. ตั้งค่า pgBouncer
ตั้งค่า pgbouncer.ini และ userlist.txt ที่ปรับแต่งสำหรับการรับ 500 Concurrent Clients โดยส่งต่อไปยัง PostgreSQL บนเครื่อง 4 vCPU / RAM 8GB หลักการสำคัญคือการใช้ pool_mode = transaction เพื่อให้แอปพลิเคชัน 500 session สลับกันใช้ physical connection สู่ Database จริงเพียง 40–60 connections (ช่วยประหยัด RAM และ CPU ไม่ติด Wait/Context Switch)
1. ไฟล์ /etc/pgbouncer/pgbouncer.ini
```bash
[databases]
; กำหนดชื่อฐานข้อมูลและปลายทางไปยัง PostgreSQL เครื่อง Local
; รูปแบบ: <dbname> = host=<pg_host> port=<pg_port> dbname=<dbname>
* = host=127.0.0.1 port=5432 auth_user=postgres

[pgbouncer]
;; -------------------------------------------------------------
;; Administrative settings
;; -------------------------------------------------------------
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
admin_users = postgres
stats_users = postgres, monitoring

;; -------------------------------------------------------------
;; Pool Configuration (สำหรับ 500 Clients บน RAM 8GB / 4 vCPU)
;; -------------------------------------------------------------
; โหมด Transaction: คืน Connection เข้า Pool ทันทีที่คำสั่ง Transaction จบ
pool_mode = transaction

; จำนวน Client Connection สูงสุดที่ PgBouncer ยอมให้ต่อเข้ามาได้
max_client_conn = 1000

; จำนวน Connection จริงสู่ PostgreSQL ต่อ 1 User/Database pair
; บน 4 vCPU ค่า 40-50 เหมาะสมที่สุดในการรีด Throughput สูงสุด
default_pool_size = 40

; จำนวน Connection ขั้นต่ำที่จะเปิดทิ้งไว้สู่ PostgreSQL
min_pool_size = 10

; Connection สำรองกรณีเกิด Spike โหลดสั้นๆ (รวมกับ default_pool_size = 60)
reserve_pool_size = 20
reserve_pool_timeout = 5.0

; เพดานรวมของ Connection จริงสู่ PostgreSQL จากทุก Database รวมกัน
; ต้องตั้งให้น้อยกว่า max_connections ใน postgresql.conf (ตั้งไว้ 80)
max_db_connections = 65

;; -------------------------------------------------------------
;; Connection Timeouts & Lifecycle
;; -------------------------------------------------------------
; หาก Pool เต็ม Client จะรอคิวนานสุดกี่วินาทีก่อน Error (ไม่ให้ค้างข้ามนาที)
query_wait_timeout = 30.0

; ตัด Connection ฝั่ง Client ที่ค้างว่างเกิน 10 นาที
client_idle_timeout = 600.0

; ปิด Connection ฝั่ง Server (PostgreSQL) ที่ไม่ได้ใช้งานเกิน 5 นาที
server_idle_timeout = 300.0

; รีเฟรช Connection สู่ Database ทุกๆ 1 ชั่วโมง เพื่อป้องกัน Connection Stale
server_lifetime = 3600.0

; ป้องกัน Client ที่เปิด Transaction ทิ้งไว้แล้วหายไป (หน่วยวินาที)
server_idle_transaction_timeout = 60.0

;; -------------------------------------------------------------
;; Low-level & Memory tuning
;; -------------------------------------------------------------
; ขนาด Buffer ต่อ Connection (ค่ามาตรฐาน 4KB ช่วยประหยัด RAM บนเครื่อง 8GB)
pkt_buf = 4096
listen_backlog = 512
```
2. ไฟล์ /etc/pgbouncer/userlist.txt
ไฟล์นี้เก็บ Username และ Hash รหัสผ่าน (ใช้ SCRAM-SHA-256):
```bash
"postgres" "SCRAM-SHA-256$4096:..."
"app_user" "SCRAM-SHA-256$4096:..."
```
> คำแนะนำในการดึง Hash รหัสผ่าน:
> ไม่จำเป็นต้องพิมพ์เอง ให้รัน Query นี้บน PostgreSQL แล้วคัดลอกผลลัพธ์มาใส่ใน userlist.txt:
> 
```sql
SELECT concat('"', usename, '" "', passwd, '"') 
FROM pg_shadow 
WHERE usename IN ('postgres', 'app_user');
```
3. การประสานงานกับ postgresql.conf
เพื่อให้ PgBouncer ทำงานสอดคล้องกัน ให้ตรวจสอบค่าใน postgresql.conf ของ Database หลัก:
```bash
# ต้องมีค่ามากกว่า max_db_connections ของ PgBouncer เสมอ (เผื่อ admin/monitoring)
max_connections = 80

# แนะนำให้เปิดใช้งานเพื่อรองรับ Prepared Statements ข้าม Session ใน Transaction Mode
track_commit_timestamp = off
```
4. ข้อจำกัดและข้อควรระวังในโหมด transaction
เนื่องจากเราใช้ pool_mode = transaction (ซึ่งจำเป็นมากสำหรับเครื่อง RAM 8GB):

  1. **Prepared Statements ฝั่ง Client:**
    หากใช้งานผ่าน Driver เช่น Go pgx, Node.js pg, หรือ Java JDBC ต้องตั้งค่าโหมด Connection ใน Library ให้เป็น prefer_simple_protocol = true หรือปิด Client-side prepared statement caching (หรือใช้ PgBouncer เวอร์ชัน 1.21+ ขึ้นไปที่มีฟีเจอร์     max_prepared_statements ในตัว)

  **2. Session-level Features ที่ห้ามใช้:**
    คำสั่งอย่าง SET TIME ZONE ..., LISTEN / NOTIFY, หรือ Temporary Tables จะไม่คงสถานะข้าม Transaction
ตรวจสอบการทำงาน (Verification)
เชื่อมต่อเข้าคอนโซลแอดมินของ PgBouncer เพื่อเช็กปริมาณ Queue และ Pool:
```bash
psql -h 127.0.0.1 -p 6432 -U postgres -d pgbouncer
```

```bash
-- 1. ดูสถานะของแต่ละ Pool (ตรวจสอบ cl_waiting ว่ามีคิวค้างนานหรือไม่)
SHOW POOLS;

-- 2. ดูจำนวน Connection ฝั่ง Client และ Server จริง
SHOW CLIENTS;
SHOW SERVERS;

-- 3. ดูสถิติ Throughput และ Latency
SHOW STATS;
```
(ถ้าคอลัมน์ cl_waiting มีค่ามากกว่า 0 ต่อเนื่องเป็นเวลานาน แสดงว่า Query ช้าหรือขนาด default_pool_size เริ่มตึงมือ)

## 6. วิธีตั้งค่า PgBouncer 1.21+ ให้รองรับ Prepared Statements ใน Transaction Mode พร้อมตัวอย่างโค้ดแอป
ตั้งแต่ PgBouncer 1.21 เป็นต้นมา มีการเพิ่มฟีเจอร์ Built-in Prepared Statement Tracking ทำให้ Client สามารถใช้ Named/Server-side Prepared Statements ร่วมกับ pool_mode = transaction ได้ทันที โดย PgBouncer จะคอย map statement ข้าม connection ให้เองโดยที่ Transaction ไม่ชนกัน
1. การตั้งค่าใน pgbouncer.ini
เพิ่มพารามิเตอร์ max_prepared_statements ภายใต้เซกชัน [pgbouncer]:
```bash
[pgbouncer]
pool_mode = transaction

; กำหนดจำนวน Prepared Statements สูงสุดที่ PgBouncer จะจำไว้ต่อ connection
; แนะนำ 100 - 200 สำหรับเครื่อง RAM 8GB (ใช้ Memory น้อยมาก แต่ช่วยให้ Cache hit สูง)
max_prepared_statements = 100

; เปิดใช้งาน protocol-level statement cache
; (ปกติ default เป็น 0 ในเวอร์ชันเก่า แต่ 1.21+ ต้องระบุค่าตัวเลขเพื่อเปิดใช้)
```
> คำเตือนเวอร์ชัน: ตรวจสอบว่า PgBouncer เป็นเวอร์ชัน 1.21 หรือใหม่กว่าโดยรันคำสั่ง pgbouncer --version
2. ตัวอย่างการคอนฟิกฝั่ง Application Drivers
แม้ PgBouncer 1.21+ จะรองรับ Prepared Statement แล้ว แต่ Driver บางตัวอาจมี Behavior การตั้งชื่อ Statement ชนกัน หรือมี Session Cache เฉพาะตัว จึงควรคอนฟิกตามตัวอย่างด้านล่าง
**Go (pgx/v5)**
pgx รองรับฟีเจอร์นี้ได้ดีมาก โดยสามารถตั้งค่าใน Connection String หรือ pgxpool.Config:
```go
package main

import (
	"context"
	"log"

	"github.com/jackc/pgx/v5/pgxpool"
)

func main() {
	// เชื่อมต่อไปที่ Port 6432 ของ PgBouncer
	connStr := "postgres://app_user:secret@127.0.0.1:6432/mydb?sslmode=disable"
	
	config, err := pgxpool.ParseConfig(connStr)
	if err != nil {
		log.Fatalf("Config parse error: %v", err)
	}

	// แนะนำสำหรับ Transaction Mode: ใช้ Named Statement Cache ปกติได้เลยบน v1.21+
	// หากพบปัญหาชื่อชนกัน ให้ fallback มาใช้ Describe Cache:
	// config.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeDescribeExec

	pool, err := pgxpool.NewWithConfig(context.Background(), config)
	if err != nil {
		log.Fatalf("Unable to connect to PgBouncer: %v", err)
	}
	defer pool.Close()

	// Query ด้วย Prepared Statement อัตโนมัติ
	var id int
	var username string
	err = pool.QueryRow(context.Background(), 
		"SELECT id, username FROM users WHERE id = $1", 1).Scan(&id, &username)
	if err != nil {
		log.Fatalf("Query failed: %v", err)
	}
}
```
***Node.js / TypeScript (pg / TypeORM / Prisma)***
สำหรับไลบรารี pg (node-postgres):
```TypeScript
import { Pool } from 'pg';

const pool = new Pool({
  host: '127.0.0.1',
  port: 6432, // PgBouncer port
  user: 'app_user',
  password: 'secret_password',
  database: 'mydb',
  max: 20, // client-side pool size
});

async function getUser(userId: number) {
  // ส่งคำสั่งแบบ Parameterized Query
  // บน PgBouncer 1.21+ สามารถใส่ name เพื่อทำ Server-side Prepared Statement ได้
  const query = {
    name: 'fetch-user-by-id',
    text: 'SELECT id, username, email FROM users WHERE id = $1',
    values: [userId],
  };

  const res = await pool.query(query);
  return res.rows[0];
}
```
> กรณีใช้ Prisma: Prisma engine มี Client-side pooler handling ในตัว หากต่อผ่าน PgBouncer แนะนำให้ต่อด้วย Query Parameter ?pgbouncer=true ใน DATABASE_URL
***Java (Spring Boot + HikariCP)***
ใน application.yml หรือ application.properties ให้ปรับ Driver property:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://127.0.0.1:6432/mydb?prepareThreshold=5&preparedStatementCacheQueries=100
    username: app_user
    password: secret_password
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      # ปิด auto-commit เพื่อให้รอบ Transaction ชัดเจน
      auto-commit: false
```
> prepareThreshold=5: ไดรเวอร์จะสร้าง Server-side Prepared Statement เมื่อ Query ตัวเดิมถูกเรียกซ้ำตั้งแต่ 5 ครั้งขึ้นไป ช่วยลดภาระการเตรียม Statement ที่ไม่จำเป็น

3. การตรวจสอบบน PgBouncer (Verification)
หลังรันแอปพลิเคชันและมีการยิง Prepared Queries สามารถตรวจสอบว่า PgBouncer ทำการแคช Statement สำเร็จหรือไม่ผ่าน Administrative Console:
```bash
psql -h 127.0.0.1 -p 6432 -U postgres -d pgbouncer
```
```sql
-- 1. ดูรายการและจำนวน Prepared Statements ที่ถูกแคชไว้
SHOW PREPARED_STATEMENTS;

-- 2. สังเกตคอลัมน์ prepared_statements ในคำสั่ง SHOW POOLS
SHOW POOLS;
```
หากตารางแสดงรายการ Query String และ Hash ชัดเจน แสดงว่า PgBouncer จัดการ Prepared Statement ข้าม Client Connection ได้อย่างสมบูรณ์โดยไม่มี Error prepared statement "..." does not exist อีกต่อไป

## 7. ขั้นตอนการตั้งค่า Systemd Service และสิทธิ์ของไฟล์ (File Permissions) สำหรับ PgBouncer ให้ทำงานเป็น Background Service บน Linux (รองรับทั้ง Ubuntu/Debian และ RHEL/Rocky/AlmaLinux)
1. สร้าง User และไดเรกทอรีที่จำเป็น
หากติดตั้งผ่าน Package Manager ตัวระบบมักสร้าง user postgres หรือ pgbouncer มาให้อัตโนมัติ หากยังไม่มี ให้สร้าง user และไดเรกทอรีสำหรับ Log, PID, และ Runtime Configuration:
```bash
# สร้าง Directory สำหรับ Config, Log และ PID
sudo mkdir -p /etc/pgbouncer
sudo mkdir -p /var/log/postgresql
sudo mkdir -p /var/run/postgresql

# กำหนด Owner เป็น postgres (หรือ pgbouncer ตามระบบที่ใช้)
sudo chown -R postgres:postgres /var/log/postgresql
sudo chown -R postgres:postgres /var/run/postgresql
```
วิธีตรวจสอบ: รัน ls -ld /var/run/postgresql จะต้องเห็น Owner และ Group เป็น postgres:postgres
2. กำหนด File Permissions อย่างปลอดภัย
ไฟล์ userlist.txt บรรจุ Hash รหัสผ่าน จึงต้องจำกัดสิทธิ์อ่าน/เขียนเฉพาะ Service User เท่านั้น:
```bash
# กำหนด Owner ทั้งหมดใน /etc/pgbouncer
sudo chown -R postgres:postgres /etc/pgbouncer

# pgbouncer.ini ให้อ่านได้เฉพาะ owner และ group
sudo chmod 640 /etc/pgbouncer/pgbouncer.ini

# userlist.txt ล็อกสิทธิ์ให้อ่าน/เขียนได้เฉพาะ owner เท่านั้น
sudo chmod 600 /etc/pgbouncer/userlist.txt
```
> วิธีตรวจสอบ: รัน ls -la /etc/pgbouncer ตรวจสอบว่า userlist.txt มีสิทธิ์เป็น -rw-------
3. สร้าง Systemd Unit File
สร้างไฟล์ Unit ที่ /etc/systemd/system/pgbouncer.service:
```bash
sudo nano /etc/systemd/system/pgbouncer.service
```
ใส่เนื้อหา Configuration ด้านล่าง:
```bash
[Unit]
Description=PgBouncer connection pooler for PostgreSQL
Documentation=man:pgbouncer(1) https://www.pgbouncer.org/
After=network.target local-fs.target
# หากรัน PostgreSQL บนเครื่องเดียวกัน ให้รอ DB สตาร์ทก่อน
After=postgresql.service

[Service]
Type=notify
User=postgres
Group=postgres

# รัน PgBouncer โดยชี้ไปที่ไฟล์คอนฟิกหลัก
ExecStart=/usr/bin/pgbouncer /etc/pgbouncer/pgbouncer.ini

# Reload ค่าคอนฟิกโดยไม่ต้อง Restart process (SIGHUP)
ExecReload=/bin/kill -HUP $MAINPID

# การจัดการ Lifecycle และ Auto-restart
KillSignal=SIGINT
Restart=on-failure
RestartSec=5s

# ปรับเพิ่มขีดจำกัด File Descriptors เพื่อรองรับ 500+ Connections
LimitNOFILE=65536

# สร้าง Directory ชั่วคราวอัตโนมัติหาก Reboot (systemd tmpfiles handler)
RuntimeDirectory=postgresql
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
```
> (หมายเหตุ: บน RHEL บางเวอร์ชัน binary อาจอยู่ที่ /usr/sbin/pgbouncer ให้ตรวจสอบด้วยคำสั่ง which pgbouncer ก่อนบันทึก)
4. เปิดใช้งานและ Start Service
Reload systemd daemon เพื่ออ่าน Unit file ใหม่ และสั่งเปิด Service:
```bash
# โหลดคอนฟิก systemd ใหม่
sudo systemctl daemon-reload

# เปิดใช้งานให้ Start อัตโนมัติตอน Boot เครื่อง
sudo systemctl enable pgbouncer

# สั่ง Start Service ทันที
sudo systemctl start pgbouncer
```
วิธีตรวจสอบ: รัน sudo systemctl status pgbouncer สถานะต้องขึ้นเป็น active (running) สีเขียว

5. ตรวจสอบการเชื่อมต่อและการ Reload Config
   - ทดสอบการ Reload คอนฟิก (Zero-downtime): เมื่อมีการแก้ไข userlist.txt หรือปรับขนาด Pool ใน pgbouncer.ini สามารถสั่ง Reload ได้ทันทีโดย Connection ไม่หลุด:
     ```bash
     sudo systemctl reload pgbouncer
     ```
   - ตรวจสอบ Port ที่เปิดรับ:
     ```bash
     ss -tlpn | grep 6432
     ```
     ต้องพบ process pgbouncer ฟังอยู่ที่พอร์ต 0.0.0.0:6432 หรือ 127.0.0.1:6432 ตามที่ตั้งไว้

## 8. ขั้นตอนการตั้งค่า Logrotate สำหรับตัดและบีบอัดไฟล์ Log ของ PgBouncer แบบอัตโนมัติ เพื่อป้องกันไม่ให้ Log โตจนกินพื้นที่ Storage 200GB จนเต็ม
1. สร้างไฟล์คอนฟิก Logrotate
สร้างไฟล์คอนฟิกใหม่ในไดเรกทอรี /etc/logrotate.d/:
```bash
sudo nano /etc/logrotate.d/pgbouncer
```
ใส่เนื้อหาด้านล่างลงในไฟล์:
```bash
/var/log/postgresql/pgbouncer.log {
    # ตัดรอบไฟล์ Log ทุกวัน
    daily

    # เก็บ Log ย้อนหลัง 14 ไฟล์ (ประมาณ 2 สัปดาห์) เก่ากว่านั้นจะถูกลบอัตโนมัติ
    rotate 14

    # บีบอัด Log เก่าเป็น .gz เพื่อประหยัดเนื้อที่ Disk
    compress

    # ชะลอการบีบอัดไฟล์ล่าสุดไว้ 1 รอบ ป้องกันปัญหาไฟล์กำลังถูกเขียน
    delaycompress

    # ข้ามการทำงานเงียบๆ หากไม่พบไฟล์ Log (ไม่แจ้ง Error กวนระบบ)
    missingok

    # ไม่ตัดไฟล์หากไฟล์มีขนาดเป็น 0 Byte
    notifempty

    # กำหนดสิทธิ์และ Owner ให้ไฟล์ Log ที่สร้างขึ้นใหม่
    create 0640 postgres postgres

    # รวมคำสั่งหลังหมุนไฟล์ให้ยิง signal เพียงครั้งเดียว
    sharedscripts

    # สั่งให้ PgBouncer ปิดไฟล์เดิมแล้วเปิดเขียนไฟล์ใหม่ (Reopen Log)
    postrotate
        if [ -f /var/run/postgresql/pgbouncer.pid ]; then
            kill -HUP $(cat /var/run/postgresql/pgbouncer.pid) 2>/dev/null || true
        fi
    endscript
}
```
> จุดสำคัญ: PgBouncer เมื่อได้รับสัญญาณ SIGHUP จะทำการ reload config และ Reopen logfile ใหม่ทันที ทำให้สามารถเขียนลงไฟล์ใหม่ได้ต่อเนื่องโดยไม่ต้อง restart service
2. ตรวจสอบสิทธิ์ของไฟล์คอนฟิก
Logrotate กำหนดความปลอดภัยเข้มงวด ไฟล์ใน /etc/logrotate.d/ ต้องเป็นของ root และห้ามมีสิทธิ์เขียนสำหรับกลุ่มอื่น:
```bash
sudo chown root:root /etc/logrotate.d/pgbouncer
sudo chmod 0644 /etc/logrotate.d/pgbouncer
```
> วิธีตรวจสอบ: รัน ls -l /etc/logrotate.d/pgbouncer ต้องได้สิทธิ์ -rw-r--r-- root root
3. ทดสอบคำสั่งการหมุน Log (Dry Run)
ทดสอบประเมินเงื่อนไขโดยไม่ตัดไฟล์จริง (Debug mode):
```bash
sudo logrotate -d /etc/logrotate.d/pgbouncer
```
> วิธีตรวจสอบ: บรรทัดล่างสุดของ output ต้องแสดงแผนการทำงาน เช่น rotating pattern: /var/log/postgresql/pgbouncer.log after 1 days... โดยไม่มีข้อผิดพลาดแจ้งเตือนเรื่อง syntax หรือ permission
4. ทดสอบตัดไฟล์จริง (Force Run)
สั่ง Force rotation เพื่อดูว่าไฟล์ถูกแยกและสร้างใหม่ถูกต้อง:
```bash
sudo logrotate -f /etc/logrotate.d/pgbouncer
```
***วิธีตรวจสอบ:***
รัน ls -la /var/log/postgresql/ จะต้องเห็นไฟล์ใหม่และไฟล์เก่าที่ถูก rotate:
```bash
-rw-r----- 1 postgres postgres     0 Sep 17 12:00 pgbouncer.log
-rw-r----- 1 postgres postgres 15234 Sep 17 11:59 pgbouncer.log.1
```
และตรวจสอบว่า PgBouncer ยังคงทำงานปกติ:
```bash
-rw-r----- 1 postgres postgres     0 Sep 17 12:00 pgbouncer.log
-rw-r----- 1 postgres postgres 15234 Sep 17 11:59 pgbouncer.log.1
```
และตรวจสอบว่า PgBouncer ยังคงทำงานปกติ:
```bash
systemctl status pgbouncer
```
ข้อแนะนำเพิ่มเติมสำหรับ Storage 200GB
หากในอนาคตพบว่าปริมาณ Traffic สูงจน Log โตเร็วเกินไปในแต่ละวัน (ก่อนจะถึงรอบ daily):
- สามารถเพิ่มบรรทัด maxsize 500M ไว้ในบล็อกคอนฟิก เพื่อสั่งให้ตัดไฟล์ทันทีที่ขนาดแตะ 500MB โดยไม่ต้องรอหมดวัน
- ไฟล์ที่ถูกบีบอัด (.gz) จะมีขนาดเล็กลงจากเดิมประมาณ 80–90% ช่วยประหยัดพื้นที่บน Disk 200GB ได้อย่างมีประสิทธิภาพ
