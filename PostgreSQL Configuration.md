# Lab 02: PostgreSQL Configuration and Memory Management

## วัตถุประสงค์
1. เรียนรู้การกำหนดค่า PostgreSQL สำหรับการปรับประสิทธิภาพ
2. ทำความเข้าใจกับ Memory Components ต่างๆ ของ PostgreSQL
3. ปฏิบัติการปรับแต่งพารามิเตอร์หลักเพื่อเพิ่มประสิทธิภาพ
4. เรียนรู้การใช้งานไฟล์ configuration ต่างๆ
5. เข้าใจหลักการคำนวณค่าที่เหมาะสมสำหรับแต่ละพารามิเตอร์

## ทฤษฎีก่อนการทดลอง

### PostgreSQL Memory Architecture

PostgreSQL ใช้สถาปัตยกรรมหน่วยความจำแบบ shared memory และ per-process memory เพื่อเพิ่มประสิทธิภาพในการประมวลผล

#### Shared Memory Components

**1. Shared Buffers**
- **นิยาม**: หน่วยความจำส่วนกลางที่ใช้แคชหน้าข้อมูล (data pages) จากดิสก์
- **วิธีการทำงาน**: เมื่อมีการอ่านข้อมูล PostgreSQL จะตรวจสอบใน shared buffers ก่อน หากไม่พบจึงไปอ่านจากดิสก์
- **ผลกระทบ**: การตั้งค่าที่เหมาะสมจะลดการเข้าถึงดิสก์และเพิ่มความเร็วในการประมวลผล
- **แนวทางการตั้งค่า**:
  - **ระบบขนาดเล็ก (< 1GB RAM)**: 25% ของ RAM
  - **ระบบขนาดกลาง (1-4GB RAM)**: 25-40% ของ RAM
  - **ระบบขนาดใหญ่ (> 4GB RAM)**: ไม่เกิน 8-16GB (เนื่องจากประสิทธิภาพจะลดลงหลังจากนี้)
  - **สูตรคำนวณ**: `RAM × 0.25` แต่ไม่เกิน 8GB

**2. WAL Buffer (Write-Ahead Log Buffer)**
- **นิยาม**: หน่วยความจำที่ใช้เก็บข้อมูล WAL ก่อนเขียนลงดิสก์
- **วิธีการทำงาน**: ทุกการเปลี่ยนแปลงข้อมูลจะถูกบันทึกใน WAL buffer ก่อน จากนั้นจึง flush ลงดิสก์
- **ความสำคัญ**: ช่วยให้การ recovery เป็นไปอย่างถูกต้องและรวดเร็ว
- **แนวทางการตั้งค่า**:
  - **Default**: 1/32 ของ shared_buffers (ขั้นต่ำ 64kB, สูงสุด 16MB)
  - **ระบบที่มี Write intensive**: 16-64MB
  - **ระบบ OLTP**: 16-32MB
  - **สูตรคำนวณ**: `shared_buffers / 32` หรือ 16MB

**3. CLog Buffer (Commit Log Buffer)**
- **นิยาม**: เก็บรายการสถานะของธุรกรรม (committed, aborted, in-progress)
- **วิธีการทำงาน**: ช่วยในการติดตามสถานะธุรกรรมและการจัดการ MVCC
- **การปรับแต่ง**: มักจะปรับตัวอัตโนมัติตาม workload

**4. Vacuum Buffer**
- **นิยาม**: หน่วยความจำที่ใช้ในกระบวนการ vacuum
- **วิธีการทำงาน**: ใช้ในการทำความสะอาดข้อมูลที่ไม่ใช้แล้ว (dead tuples)
- **ความสำคัญ**: ป้องกันการ bloat ของตารางและอินเด็กซ์

#### Per-Process Memory Components

**1. Work Memory**
- **นิยาม**: หน่วยความจำที่แต่ละ connection ใช้สำหรับการดำเนินการ sort, hash, merge
- **วิธีการทำงาน**: 
  - ใช้สำหรับ ORDER BY, GROUP BY, Hash joins
  - หากข้อมูลเกินขนาด work_mem จะเขียนลงดิสก์ (spill to disk)
- **แนวทางการตั้งค่า**:
  - **สูตรพื้นฐาน**: `(RAM × 0.25) / max_connections`
  - **ระบบ OLTP**: 4-16MB
  - **ระบบ OLAP**: 256MB-1GB
  - **Data Warehouse**: 1-8GB
- **ตัวอย่างการคำนวณ**:
  ```
  RAM = 8GB, max_connections = 100
  work_mem = (8GB × 0.25) / 100 = 20MB
  ```

**2. Maintenance Work Memory**
- **นิยาม**: หน่วยความจำสำหรับงานบำรุงรักษา เช่น VACUUM, CREATE INDEX, REINDEX
- **วิธีการทำงาน**: ใช้ครั้งละหนึ่งการดำเนินการ ไม่เหมือน work_mem ที่อาจใช้หลายครั้งพร้อมกัน
- **แนวทางการตั้งค่า**:
  - **ควรมากกว่า work_mem อย่างน้อย 4-8 เท่า**
  - **ระบบเล็ก**: 64-256MB
  - **ระบบใหญ่**: 1-4GB
  - **สูตรคำนวณ**: `work_mem × 8` หรือ `RAM × 0.05-0.10`

#### Cache และ Planning Memory

**1. Effective Cache Size**
- **นิยาม**: การประมาณขนาดหน่วยความจำที่ระบบปฏิบัติการและ PostgreSQL ใช้สำหรับ cache รวมกัน
- **วิธีการทำงาน**: Query planner ใช้ค่านี้ในการตัดสินใจเลือกแผนการประมวลผล
- **ไม่ใช่การจองหน่วยความจำ**: เป็นเพียงข้อมูลสำหรับ planner เท่านั้น
- **แนวทางการตั้งค่า**:
  - **แนะนำ**: 50-75% ของ RAM ทั้งหมด
  - **ระบบที่มี application อื่นร่วมด้วย**: 50-60%
  - **ระบบที่ PostgreSQL เป็นหลัก**: 70-75%

### Configuration Files

#### 1. postgresql.conf
- **ตำแหน่ง**: `$PGDATA/postgresql.conf`
- **หน้าที่**: กำหนดค่าหลักของเซิร์ฟเวอร์
- **โครงสร้าง**:
  ```ini
  # Memory
  shared_buffers = 256MB
  work_mem = 16MB
  
  # WAL
  wal_buffers = 16MB
  
  # Query Planner
  effective_cache_size = 2GB
  ```

#### 2. postgresql.auto.conf
- **ตำแหน่ง**: `$PGDATA/postgresql.auto.conf`
- **หน้าที่**: เก็บการตั้งค่าจาก ALTER SYSTEM commands
- **ความสำคัญ**: Override ค่าใน postgresql.conf
- **ข้อดี**: การจัดการแบบ programmatic, การ version control

#### 3. pg_hba.conf
- **ตำแหน่ง**: `$PGDATA/pg_hba.conf`
- **หน้าที่**: Host-Based Authentication
- **โครงสร้าง**:
  ```
  # TYPE  DATABASE  USER      ADDRESS         METHOD
  local   all       postgres                  peer
  host    all       all       127.0.0.1/32    md5
  ```

#### 4. pg_ident.conf
- **หน้าที่**: แมป OS users กับ database users
- **ใช้งานร่วมกับ**: pg_hba.conf (ident authentication method)

### Memory Sizing Strategies

#### Strategy 1: Conservative Approach (ระบบที่มี application อื่น)
```
Total RAM = 8GB
shared_buffers = 1.5GB (18.75%)
work_mem = 16MB
maintenance_work_mem = 128MB
effective_cache_size = 4GB (50%)
```

#### Strategy 2: Aggressive Approach (ระบบที่ PostgreSQL เป็นหลัก)
```
Total RAM = 8GB
shared_buffers = 2GB (25%)
work_mem = 32MB
maintenance_work_mem = 512MB
effective_cache_size = 6GB (75%)
```

#### Strategy 3: OLAP/Data Warehouse
```
Total RAM = 32GB
shared_buffers = 8GB (25%)
work_mem = 1GB
maintenance_work_mem = 4GB
effective_cache_size = 24GB (75%)
```

## การเตรียมความพร้อม

### สร้าง PostgreSQL Container พร้อมกำหนดค่า
```bash
# สร้าง Docker Volume สำหรับเก็บข้อมูล
docker volume create postgres-config-data

# สร้าง Container พร้อม Volume และจำกัด memory
docker run --name postgres-config \
  -e POSTGRES_PASSWORD=admin123 \
  -e POSTGRES_DB=configdb \
  -p 5432:5432 \
  -v postgres-config-data:/var/lib/postgresql/data \
  --memory=2g \
  --cpus=2 \
  -d postgres

# ตรวจสอบสถานะ container
docker ps
docker logs postgres-config
```

## ขั้นตอนการทดลอง

### Step 1: การวิเคราะห์ระบบและความต้องการ

#### 1.1 ตรวจสอบข้อมูลระบบ
```bash
# ตรวจสอบ memory ของ container
docker exec postgres-config free -h

# ตรวจสอบ CPU
docker exec postgres-config nproc

# ตรวจสอบ disk space
docker exec postgres-config df -h
```
### บันทึกผลการทดลอง
```
1. อธิบายหน้าที่คำสั่ง docker exec postgres-config free, docker exec postgres-config df

docker exec postgres-config free
หน้าที่: ดูสถานะหน่วยความจำ (RAM/SWAP) ภายในคอนเทนเนอร์
Mem: total / used / free / shared / buff/cache / available
Swap: เช่นเดียวกันสำหรับสว็อป
ใช้งานจริง (อ่านง่าย):

 docker exec postgres-config df
หน้าที่: ดูการใช้งานดิสก์ของไฟล์ซิสเต็มภายในคอนเทนเนอร์ (รวมถึง volume ที่ mount เข้าไป)
คอลัมน์สำคัญ:
Filesystem / Size / Used / Avail / Use% / Mounted on

ใช้งานจริง (อ่านง่าย):
```
docker exec -it postgres-config df -h
```
ใช้เมื่อไร: เวลา Postgres เขียนไฟล์ไม่ได้/ล่ม เพราะ “ดิสก์เต็ม” ให้เช็ค Use% ของพาร์ทิชันที่ Mounted on เช่น /var/lib/postgresql/data

2. option -h ในคำสั่งมีผลอย่างไร
option -h ที่ใช้กับคำสั่ง free และ df ใน Linux มีผลเหมือนกัน คือทำให้ ผลลัพธ์อ่านง่ายขึ้น (human-readable) โดยเปลี่ยนการแสดงหน่วยจาก byte ตัวเลขยาว ๆ ให้กลายเป็นหน่วยที่คนคุ้นเคย เช่น KB, MB, GB แทน

3. docker exec postgres-config nproc  แสดงค่าผลลัพธ์อย่างไร
<img width="473" height="58" alt="image" src="https://github.com/user-attachments/assets/74aba92c-84b2-4cb7-920b-f01d1acb5fff" />


```
#### 1.2 เชื่อมต่อและตรวจสอบสถานะปัจจุบัน
```bash
docker exec -it postgres-config psql -U postgres
```

```sql
-- ตรวจสอบเวอร์ชัน
SELECT version();

-- ดูตำแหน่งไฟล์ configuration
SHOW config_file;
SHOW hba_file;
SHOW data_directory;

### บันทึกผลการทดลอง
```
1. ตำแหน่งที่อยู่ของไฟล์ configuration อยู่ที่ตำแหน่งใด

<img width="355" height="97" alt="image" src="https://github.com/user-attachments/assets/c11d257d-a584-48c6-9094-e5933a51f84a" />
2. ตำแหน่งที่อยู่ของไฟล์ data อยู่ที่ตำแหน่งใด

<img width="242" height="87" alt="image" src="https://github.com/user-attachments/assets/177b05b0-e91c-4694-8ba2-e073a0cf7ba7" />

```
-- ตรวจสอบการตั้งค่าปัจจุบัน
SELECT name, setting, unit, category, short_desc 
FROM pg_settings 
WHERE name IN (
    'shared_buffers', 'work_mem', 'maintenance_work_mem',
    'wal_buffers', 'effective_cache_size', 'max_connections'
);
```
### บันทึกผลการทดลอง
```
บันทึกรูปผลของ configuration ทั้ง 6 ค่า 
```

### Step 2: การปรับแต่งพารามิเตอร์แบบค่อยเป็นค่อยไป

#### 2.1 ปรับแต่ง Shared Buffers (ต้อง restart)
```sql
-- ตรวจสอบค่าปัจจุบัน
SELECT name, setting, unit, source, pending_restart
FROM pg_settings 
WHERE name = 'shared_buffers';

### ผลการทดลอง
```
1.รูปผลการรันคำสั่ง

<img width="539" height="81" alt="image" src="https://github.com/user-attachments/assets/fb801e94-29d0-4480-a7bc-2259b949b9b3" />

2. ค่า  shared_buffers มีการกำหนดค่าไว้เท่าไหร่ (ใช้ setting X unit)

16384 × 8kB = 131072kB = 128MB

3. ค่า  pending_restart ในผลการทดลองมีค่าเป็นอย่างไร และมีความหมายอย่างไร

ผลลัพธ์ที่เห็นคือ f (false)
ความหมาย:
ถ้า pending_restart = f → ค่า shared_buffers ที่ตั้งอยู่ตอนนี้ มีผลแล้วทันที ไม่จำเป็นต้องรีสตาร์ท PostgreSQL ใหม่
ถ้าเป็น t (true) → หมายถึงเรามีการเปลี่ยนค่า parameter ที่ต้องการ restart PostgreSQL เพื่อให้ค่ามีผล (ไม่พอแค่ pg_reload_conf())

```
-- คำนวณและตั้งค่าใหม่
-- สำหรับระบบ 2GB: 512MB (25%)
ALTER SYSTEM SET shared_buffers = '512MB';

-- ตรวจสอบการเปลี่ยนแปลง
select pg_reload_conf();
SELECT name, setting, unit, source, pending_restart
FROM pg_settings 
WHERE name = 'shared_buffers';

```
-- ออกจาก postgres prompt (กด \q แล้ว enter) ทำการ Restart PostgreSQL ด้วยคำสั่ง แล้ว run docker อีกครั้ง หรือใช้วิธีการ stop และ run containner
docker exec -it -u postgres postgres-config pg_ctl restart -D /var/lib/postgresql/data -m fast

### ผลการทดลอง

รูปผลการเปลี่ยนแปลงค่า pending_restart

<img width="518" height="175" alt="image" src="https://github.com/user-attachments/assets/2c09f905-d9bc-4cf2-b562-2de7adb99609" />

รูปหลังจาก restart postgres

<img width="660" height="130" alt="image" src="https://github.com/user-attachments/assets/035e356a-4032-41f8-a001-15ba6c056310" />




#### 2.2 ปรับแต่ง Work Memory (ไม่ต้อง restart)
```sql
-- ตรวจสอบค่าปัจจุบัน
SHOW work_mem;

-- คำนวณและตั้งค่าใหม่
-- สำหรับ max_connections = 100 
ALTER SYSTEM SET work_mem = '20MB';
SELECT pg_reload_conf();

-- ตรวจสอบ
SHOW work_mem;

-- ดูว่าค่าเปลี่ยนแล้วหรือไม่
SELECT name, setting, unit, source
FROM pg_settings 
WHERE name = 'work_mem';
```
### ผลการทดลอง

รูปผลการเปลี่ยนแปลงค่า work_mem

<img width="422" height="222" alt="image" src="https://github.com/user-attachments/assets/2130cdc4-61cf-4447-a6a5-bfdb31e0dbfe" />


#### 3.3 ปรับแต่ง Maintenance Work Memory
```sql
-- ตรวจสอบค่าปัจจุบัน
SHOW maintenance_work_mem;

-- ตั้งค่าเป็น 8-12 เท่าของ work_mem
ALTER SYSTEM SET maintenance_work_mem = '256MB';
SELECT pg_reload_conf();

-- ตรวจสอบ
SHOW maintenance_work_mem;
```
### ผลการทดลอง

รูปผลการเปลี่ยนแปลงค่า maintenance_work_mem

<img width="470" height="226" alt="image" src="https://github.com/user-attachments/assets/8ecad133-0fc3-45f5-84e4-9b1e221bdbab" />


#### 3.4 ปรับแต่ง WAL Buffers
```sql
-- ตรวจสอบค่าปัจจุบัน
SHOW wal_buffers;

-- ตั้งค่าตาม shared_buffers/32 หรือ 16MB
ALTER SYSTEM SET wal_buffers = '16MB';
SELECT pg_reload_conf();

SELECT name, setting, unit, pending_restart 
FROM pg_settings 
WHERE name = 'wal_buffers';

-- restart postgres
docker restart postgres-config

docker exec -it postgres-config psql -U postgres
-- ตรวจสอบ
SHOW wal_buffers;
```
### ผลการทดลอง

รูปผลการเปลี่ยนแปลงค่า wal_buffers

<img width="356" height="201" alt="image" src="https://github.com/user-attachments/assets/55d040c1-65f6-4cc2-880a-ccf4970fd4e0" />


#### 3.5 ปรับแต่ง Effective Cache Size
```sql
-- ตรวจสอบค่าปัจจุบัน
SHOW effective_cache_size;

-- ตั้งค่าเป็น 75% ของ RAM
ALTER SYSTEM SET effective_cache_size = '1536MB';
SELECT pg_reload_conf();

-- ตรวจสอบ
SHOW effective_cache_size;
```
### ผลการทดลอง

รูปผลการเปลี่ยนแปลงค่า effective_cache_size

<img width="295" height="98" alt="image" src="https://github.com/user-attachments/assets/d2caf0bd-c39f-4127-8448-abf9bbd6a7c4" />


### Step 4: ตรวจสอบผล


```sql
-- สร้างรายงานการตั้งค่า
SELECT 
    name,
    setting,
    unit,
    CASE 
        WHEN name = 'shared_buffers' THEN pg_size_pretty(setting::bigint * 8192)
        WHEN unit = 'kB' THEN pg_size_pretty(setting::bigint * 1024)
        WHEN unit = 'MB' THEN setting || ' MB'
        ELSE setting || COALESCE(' ' || unit, '')
    END AS formatted_value,
    source,
    short_desc
FROM pg_settings 
WHERE name IN (
    'shared_buffers', 'work_mem', 'maintenance_work_mem',
    'wal_buffers', 'effective_cache_size'
)
ORDER BY name;
```
### ผลการทดลอง

รูปผลการลัพธ์การตั้งค่า

<img width="1049" height="204" alt="image" src="https://github.com/user-attachments/assets/f666e606-dd33-43bf-ab02-1cc787b3a173" />

### Step 5: การสร้างและทดสอบ Workload

#### 5.1 สร้างฐานข้อมูลทดสอบ
```sql
-- สร้างฐานข้อมูลสำหรับทดสอบประสิทธิภาพ
CREATE DATABASE performance_test;
\c performance_test

-- เปิด timing เพื่อวัดเวลา
\timing on

-- สร้างตารางใหญ่
CREATE TABLE large_table (
    id SERIAL PRIMARY KEY,
    data TEXT,
    number INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- สร้าง index
CREATE INDEX idx_large_table_number ON large_table(number);
CREATE INDEX idx_large_table_created_at ON large_table(created_at);
```

#### 5.2 การทดสอบ Work Memory
```sql
-- แทรกข้อมูลจำนวนมาก (ทดสอบ work_mem)
INSERT INTO large_table (data, number) 
SELECT 
    'Test data for performance ' || i,
    (random() * 1000000)::integer
FROM generate_series(1, 500000) AS i;

-- ทดสอบ Sort operation (จะใช้ work_mem)
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table 
ORDER BY data 
LIMIT 1000;
```
### ผลการทดลอง

1. คำสั่ง EXPLAIN(ANALYZE,BUFFERS) คืออะไร 
EXPLAIN: แสดง “แผนการทำงาน” (execution plan) ที่ Postgres เลือกใช้
ANALYZE: ให้รันคำสั่งจริง แล้วรายงานเวลาที่ใช้และจำนวนแถวจริง (ไม่ใช่แค่คาดคะเน)
BUFFERS: รายงานสถิติ buffer I/O เช่น shared hit/read/dirtied/written เพื่อดูว่าอ่านจากแคชแค่ไหน vs ต้องอ่านจากดิสก์

2. รูปผลการรัน

<img width="1048" height="388" alt="image" src="https://github.com/user-attachments/assets/145c5e37-2bf6-4e38-9d73-f64cb3b45b3a" />

3. อธิบายผลลัพธ์ที่ได้
   Limit (… actual rows=1000 …)
โหนดบนสุดตัดผลลัพธ์เหลือ 1000 แถวตามคำสั่ง

Gather Merge (Workers Planned/Launched: 2)
รวมผลจาก “คนงาน” 2 ตัว + ตัวหลัก (leader) โดย “merge” แบบเรียงลำดับเพื่อคงออร์เดอร์ ORDER BY data

Sort … Sort Method: top-N heapsort
แต่ละสตรีมทำ sort แบบ Top-N (เก็บเฉพาะรายการที่มีสิทธิ์ติด 1000 อันดับแรก)

ใช้หน่วยความจำเล็กมาก: ~242 kB (worker ~170–178 kB) → ไม่มีการ spill ลงดิสก์ แปลว่า work_mem เพียงพอสำหรับเคสนี้

Parallel Seq Scan on large_table (… rows=166667 loops=3)
สแกนทั้งตารางแบบขนาน 3 ลูป = leader + 2 workers แบ่งงานกันอ่านรวม ~500k แถวพอดี

Buffers

Buffers: shared hit=5059 (ใต้ Parallel Seq Scan) และ shared hit=5133 (โหนดบน ๆ)

ทั้งหมดคือ hit ใน shared buffers (8kB ต่อบล็อก) → ~40 MB (5133 × 8kB)

ไม่มี shared read ⇒ ไม่ต้องอ่านจากดิสก์จริงเลย (ข้อมูลอยู่ในแคชแล้ว) ⇒ งานเป็น CPU-bound

Planning Time: 0.270 ms / Execution Time: 158.966 ms
วางแผนเร็วมาก; เวลาหลักอยู่ที่สแกนและจัดเรียง
ค่า “Time: 160.369 ms” คือเวลาที่ psql รายงานรวม ๆ

```sql
-- ทดสอบ Hash operation
EXPLAIN (ANALYZE, BUFFERS)
SELECT number, COUNT(*) 
FROM large_table 
GROUP BY number 
HAVING COUNT(*) > 1
LIMIT 100;
```

### ผลการทดลอง

1. รูปผลการรัน

<img width="1152" height="326" alt="image" src="https://github.com/user-attachments/assets/d8fb3998-df00-482c-b2bc-ca30dc41a6fa" />

2. อธิบายผลลัพธ์ที่ได้ 
Limit … (actual rows=100)
ตัดผลลัพธ์เหลือ 100 แถวสุดท้ายที่ผ่านเงื่อนไข
GroupAggregate … Group Key: number
จัดกลุ่มตามคอลัมน์ number แล้วนับจำนวนแถวในแต่ละกลุ่ม
มี Filter: (count(*)) > 1 ⇒ กลุ่มที่มีสมาชิก ≤ 1 จะถูกตัดทิ้ง
Rows Removed by Filter: 319 หมายถึงมี 319 กลุ่มที่ถูกกรองออกเพราะนับได้ ≤ 1
Index Only Scan using idx_large_table_number on large_table
อ่านข้อมูลจาก ดัชนี เพียงอย่างเดียว (ไม่ต้องแตะตาราง) เพื่อป้อนให้ GroupAggregate
Heap Fetches: 0 ⇒ ไม่ต้องไปอ่านตารางจริงเลย เพราะ

ข้อมูลที่ต้องใช้สำหรับการรวมกลุ่ม/คัดกรองมีแค่คอลัมน์ number ซึ่งอยู่ในดัชนีครบ (ดัชนี “ครอบคลุม” งานนี้)
หน้า (pages) ในตารางถูกมาร์กว่า all-visible ใน visibility map ทำให้ Index-Only สามารถยืนยันความมองเห็นได้โดยไม่ต้องแตะ heap

3. การสแกนเป็นแบบใด เกิดจากเหตุผลใด
หตุผลที่ได้ Index Only Scan:
คำสั่งต้องใช้แค่คอลัมน์ number (สำหรับ GROUP BY และ COUNT(*)) ⇒ ดัชนีครอบคลุมคอลัมน์ที่ต้องใช้
Visibility map ของตารางบอกว่าหน้าเหล่านั้นเป็น all-visible ⇒ ไม่ต้องอ่าน heap เพื่อตรวจสิทธิ์การมองเห็นทุกรายการ (Heap Fetches: 0)

#### 5.3 การทดสอบ Maintenance Work Memory
```sql
-- ทดสอบ CREATE INDEX (จะใช้ maintenance_work_mem)
CREATE INDEX CONCURRENTLY idx_large_table_data 
ON large_table USING btree(data);
```

```sql
-- ทดสอบ VACUUM (จะใช้ maintenance_work_mem)
DELETE FROM large_table WHERE id % 10 = 0;

VACUUM (ANALYZE, VERBOSE) large_table;
```
### ผลการทดลอง

1. รูปผลการทดลอง จากคำสั่ง VACUUM (ANALYZE, VERBOSE) large_table;

<img width="1200" height="728" alt="image" src="https://github.com/user-attachments/assets/b154eac6-a8b2-42da-9241-b6f9c846372d" />

<img width="1031" height="292" alt="image" src="https://github.com/user-attachments/assets/829e6f1a-b41a-4d38-b429-436a3e87f327" />

2. อธิบายผลลัพธ์ที่ได้
1) ส่วนของตาราง (heap)

pages: 0 removed, 5059 remain, 5059 scanned (100.00% of total)
สแกนทั้งตารางครบทุกหน้า

dead item identifiers removed: 50000
เก็บกวาด dead tuples ที่เกิดจาก DELETE เรียบร้อย → พื้นที่ว่างกลับมาใช้ใหม่ได้ (ลด bloat)

index scan needed: 0 pages … (ขึ้นกับบรรทัดย่อย)
เป็นสถิติว่าต้องใช้สแกนดัชนีช่วยล้างบางมากน้อยแค่ไหนในรอบนี้

avg rate: … / WAL usage: …
ความเร็วและ WAL ที่เกิดขึ้น—เคสนี้เบามาก (WAL 1 record, ~188 bytes) เพราะ VACUUM ส่วนใหญ่เป็นงานภายใน

2) ส่วนของดัชนี

เห็นรายงานต่อดัชนีแต่ละตัว เช่น
idx_large_table_number, idx_large_table_created_at, และตัวใหม่ idx_large_table_data

ตัวอย่าง:
pages: 1743 in total, 0 newly deleted, 0 currently deleted, 0 reusable
แปลว่า “ยังไม่มีหน้าของดัชนีที่ลบทิ้งได้” ในรอบนี้—เป็นไปได้ว่าทuples ที่ถูกลบเป็น HOT-prunable หรือ VACUUM ยังไม่พบสิ่งที่ต้องลบในดัชนี

launched 2 parallel vacuum workers for index vacuuming (planned: 2)
ใช้ Parallel VACUUM เร่งจัดการดัชนี

3) ANALYZE (อัปเดตสถิติ)

analyzing "public.large_table" และบรรทัดสรุป:
scanned 5059 of 5059 pages, containing 450000 live rows and 0 dead rows; 30000 rows in sample, 450000 estimated total rows

ความหมาย: อัปเดตสถิติเรียบร้อย → Planner จะ “รู้ความจริงล่าสุด” ของขนาดตาราง/การกระจายข้อมูล ทำให้เลือกแผนเหมาะสม (เช่น ใช้ดัชนีตัวใหม่)
### Step 6: การติดตาม Memory Usage

#### 6.1 สร้างฟังก์ชันติดตาม Memory
```sql
-- สร้างฟังก์ชันดู memory usage
CREATE OR REPLACE FUNCTION get_memory_usage()
RETURNS TABLE(
    setting_name TEXT,
    current_value TEXT,
    bytes_value BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        pg_settings.name::TEXT,
        pg_settings.setting::TEXT,
        CASE 
            WHEN pg_settings.name = 'shared_buffers' THEN pg_settings.setting::bigint * 8192
            WHEN pg_settings.unit = 'kB' THEN pg_settings.setting::bigint * 1024
            WHEN pg_settings.unit = 'MB' THEN pg_settings.setting::bigint * 1024 * 1024
            WHEN pg_settings.unit = 'GB' THEN pg_settings.setting::bigint * 1024 * 1024 * 1024
            ELSE pg_settings.setting::bigint
        END as bytes_value
    FROM pg_settings 
    WHERE pg_settings.name IN (
        'shared_buffers', 'work_mem', 'maintenance_work_mem',
        'wal_buffers', 'effective_cache_size'
    );
END;
$$ LANGUAGE plpgsql;

-- ใช้งานฟังก์ชัน
SELECT 
    setting_name,
    current_value,
    pg_size_pretty(bytes_value) as formatted_size,
    ROUND((bytes_value::decimal / (1024*1024*1024)) * 100, 2) as percent_of_2gb
FROM get_memory_usage();
```
### ผลการทดลอง

รูปผลการทดลอง

<img width="555" height="273" alt="image" src="https://github.com/user-attachments/assets/6b737045-0c50-42ee-b4bd-cb37803e7520" />


#### 6.2 การติดตาม Buffer Hit Ratio
```sql
-- ตรวจสอบ buffer hit ratio (ควรอยู่เหนือ 95%)
SELECT 
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    CASE 
        WHEN heap_blks_read + heap_blks_hit = 0 THEN NULL
        ELSE ROUND((heap_blks_hit::decimal / (heap_blks_read + heap_blks_hit)) * 100, 2)
    END as hit_ratio_percent
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read + heap_blks_hit DESC;
```
### ผลการทดลอง

1. รูปผลการทดลอง

<img width="647" height="280" alt="image" src="https://github.com/user-attachments/assets/bd3eecb1-f5bf-4c06-8551-d37fa99491a7" />
2. อธิบายผลลัพธ์ที่ได้
 
   ตาราง large_table ถูกใช้งานจาก แคชทั้งหมด ในช่วงเวลาที่เก็บสถิตินี้ → ดีต่อความเร็ว (latency ต่ำ)

ตัวเลขเหล่านี้เป็น สะสม ตั้งแต่สถิติล่าสุดถูกรีเซ็ตหรือเซิร์ฟเวอร์เริ่มใหม่—not per query

#### 6.3 ดู Buffer Hit Ratio ทั้งระบบ
```sql
SELECT datname,
       blks_read,
       blks_hit,
       ROUND((blks_hit::decimal / (blks_read + blks_hit)) * 100, 2) as hit_ratio_percent
FROM pg_stat_database 
WHERE datname = current_database();
```
### ผลการทดลอง

1. รูปผลการทดลอง

<img width="586" height="235" alt="image" src="https://github.com/user-attachments/assets/cf242855-000c-45d7-8dc3-a5fa2e79c837" />

2. อธิบายผลลัพธ์ที่ได้
วิร์กโหลดที่ผ่านมา “วิ่งบนแคช” แทบจะทั้งหมด → latency ต่ำ, CPU-bound มากกว่า I/O-bound
blks_read มีบ้างเล็กน้อย อาจมาจากรอบแรกที่ยังไม่อุ่นแคช, ตอนสร้างดัชนี, หรือช่วงสแกนครั้งแรกก่อนข้อมูลเข้า shared buffers

#### 6.4 ดู Table ที่มี Disk I/O มาก
```sql
SELECT 
    schemaname,
    relname as tablename,
    heap_blks_read,
    heap_blks_hit,
    (heap_blks_read + heap_blks_hit) as total_access,
    ROUND((heap_blks_hit::decimal / NULLIF(heap_blks_read + heap_blks_hit, 0)) * 100, 2) as hit_ratio_percent,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as table_size
FROM pg_statio_user_tables
WHERE heap_blks_read > 0  -- เฉพาะที่มี disk read
ORDER BY heap_blks_read DESC
LIMIT 10;
```
### ผลการทดลอง

1. รูปผลการทดลอง

<img width="790" height="288" alt="image" src="https://github.com/user-attachments/assets/b8f3f096-7e02-4c43-a50f-78965d1f9351" />

2. อธิบายผลลัพธ์ที่ได้
   ไม่มีตารางผู้ใช้ (user tables) ใด ๆ ที่มี heap_blks_read > 0 ในรอบสถิติปัจจุบัน
แปลว่าในช่วงที่เก็บสถิตินี้ ทุกการเข้าถึงตารางถูกเสิร์ฟจาก shared buffers (แคชของ Postgres) ทั้งหมด → ไม่มีการต้อง “ดึงบล็อก 8KB จากระบบไฟล์” เข้าสู่ shared buffers เลย

### Step 7: การปรับแต่ง Autovacuum

#### 7.1 ทำความเข้าใจ Autovacuum Parameters
```sql
-- ดูการตั้งค่า autovacuum ทั้งหมด
SELECT name, setting, unit, short_desc 
FROM pg_settings 
WHERE name LIKE '%autovacuum%'
ORDER BY name;
```
### ผลการทดลอง

1. รูปผลการทดลอง

<img width="1085" height="674" alt="image" src="https://github.com/user-attachments/assets/1cf1a54f-0f6a-41b6-a48c-a20e50c34807" />

2. อธิบายค่าต่าง ๆ ที่มีความสำคัญ

เกณฑ์ “ถึงเวลา ANALYZE”

ใช้สูตร:
ถึงเวลาเมื่อ reltuples_changed ≥ autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × reltuples

autovacuum_analyze_threshold = 50
ค่าคงที่ฐาน
autovacuum_analyze_scale_factor = 0.1
สัดส่วนตามขนาดตาราง (10%)
ตาราง 1,000,000 แถว → เกณฑ์ ≈ 50 + 0.1×1,000,000 = 100,050 แถวเปลี่ยนแปลง จึง ANALYZE
ทิป: ตารางใหญ่/อัพเดตบ่อย → ลด scale_factor (หรือกำหนด per-table) เพื่อให้สถิติสดขึ้น

เกณฑ์ “ถึงเวลา VACUUM” (จาก UPDATE/DELETE)

ใช้สูตร:
ถึงเวลาเมื่อ dead_tuples ≥ autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × reltuples
autovacuum_vacuum_threshold = 50
ค่าคงที่ฐาน

autovacuum_vacuum_scale_factor = 0.2
สัดส่วนตามขนาดตาราง (20%)

ตาราง 1,000,000 แถว → เกณฑ์ ≈ 50 + 0.2×1,000,000 = 200,050 dead tuples จึง VACUUM
ทิป: ตาราง production ใหญ่ ๆ มัก “ลดแรง” ค่านี้ลง (เช่น 0.05 หรือ 0.01) เฉพาะบางตาราง ผ่าน ALTER TABLE ... SET (autovacuum_vacuum_scale_factor=...) เพื่อกันบวม
เกณฑ์ “VACUUM จากการ INSERT จำนวนมาก”

(ช่วยจัดการ visibility map/ฟื้นคืนพื้นที่เร็วขึ้นแม้ไม่มี DELETE/UPDATE)

ใช้สูตรคล้ายกัน:
ถึงเวลาเมื่อ inserted_tuples ≥ autovacuum_vacuum_insert_threshold + autovacuum_vacuum_insert_scale_factor × reltuples

autovacuum_vacuum_insert_threshold = 1000
autovacuum_vacuum_insert_scale_factor = 0.2
ทิป: เวิร์กโหลดที่ “เขียนหนัก” แม้ไม่ค่อยลบ/อัพเดต → ปรับสองค่านี้ให้ไวขึ้นเพื่อลดสแกน heap ที่ไม่จำเป็น

ควบคุมความแรง/การหน่วงของ VACUUM
autovacuum_vacuum_cost_delay = 2 ms
หน่วงเวลาต่อ cost unit เพื่อไม่ให้กิน I/O เกินไป (มากขึ้น = นุ่มนวลขึ้น แต่ช้าลง)
autovacuum_vacuum_cost_limit = -1
-1 = สืบทอด ค่าระดับ global vacuum_cost_limit (ปกติ ~200)

อยากให้เก็บกวาดเร็วขึ้น → เพิ่ม limit / ลด delay (เฉพาะตารางสำคัญทำผ่าน storage parameters จะปลอดภัยกว่า)
หน่วยความจำของ worker

autovacuum_work_mem = -1 (kB)
-1 = ใช้ค่า maintenance_work_mem แทน
งาน VACUUM/ANALYZE ขนาดใหญ่ → เพิ่ม maintenance_work_mem (หรือกำหนด autovacuum_work_mem ตรง ๆ) ให้พอ แต่หลีกเลี่ยงการตั้งสูงจนกิน RAM เกิน

ป้องกัน Transaction ID wraparound

autovacuum_freeze_max_age = 2,000,000,000
อายุ XID ที่ถึงแล้วจะบังคับทำ anti-wraparound VACUUM (งานนี้สำคัญมาก—ห้ามปล่อยเลยจุด)

autovacuum_multixact_freeze_max_age = 4,000,000,000
ขีดจำกัดสำหรับ MultiXact (ล็อกหลายผู้ใช้) ในทำนองเดียวกัน
การบันทึก Log

log_autovacuum_min_duration = 600000 ms (10 นาที)
ล็อกเฉพาะงาน autovacuum ที่ใช้เวลานานกว่า 10 นาที

ดีสำหรับ production: ถ้ากำลังจูน ให้ตั้ง ชั่วคราว เป็น 0 (ล็อกทุกครั้ง) เพื่อเก็บหลักฐาน แล้วปรับกลับเมื่อจบการทดสอบ

แนวทางจูนแบบสรุป

#### 7.2 การปรับแต่ง Autovacuum สำหรับประสิทธิภาพ
```sql
-- ปรับจำนวน autovacuum workers
-- แนะนำ: 1 worker ต่อ 2-4 GB RAM
ALTER SYSTEM SET autovacuum_max_workers = 3;

-- ปรับความถี่ของ autovacuum
-- แนะนำ: 30-60 วินาที สำหรับระบบที่มี write activity สูง
ALTER SYSTEM SET autovacuum_naptime = '45s';

-- ปรับ scale factor สำหรับ vacuum
-- แนะนำ: 0.1-0.2 (10-20% ของตารางเปลี่ยนแปลงแล้วจึง vacuum)
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.15;

-- ปรับ scale factor สำหรับ analyze
-- แนะนำ: 0.05-0.1 (5-10% ของตารางเปลี่ยนแปลงแล้วจึง analyze)
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.05;

-- ปรับ work_mem สำหรับ autovacuum
ALTER SYSTEM SET autovacuum_work_mem = '512MB';

-- โหลดการตั้งค่าใหม่
SELECT pg_reload_conf();
```
### ผลการทดลอง

รูปผลการทดลองการปรับแต่ง Autovacuum (Capture รวมทั้งหมด 1 รูป)

<img width="714" height="398" alt="image" src="https://github.com/user-attachments/assets/133ddfb6-269c-4489-b37c-8af3587afbc3" />


### Step 8: Performance Testing และ Benchmarking

#### 8.1 สร้างชุดทดสอบแบบ Comprehensive
```sql
-- สร้างตารางสำหรับเก็บผลการทดสอบ
CREATE TABLE performance_results (
    test_name VARCHAR(100),
    config_set VARCHAR(50),
    execution_time_ms INTEGER,
    buffer_hits BIGINT,
    buffer_reads BIGINT,
    test_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ฟังก์ชันสำหรับรันการทดสอบ
CREATE OR REPLACE FUNCTION run_performance_test(
    test_name TEXT,
    config_name TEXT
) RETURNS void AS $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    execution_time INTEGER;
BEGIN
    -- รีเซ็ต statistics
    SELECT pg_stat_reset();
    
    start_time := clock_timestamp();
    
    -- การทดสอบจริง (ปรับตามชุดทดสอบที่ต้องการ)
    IF test_name = 'large_sort' THEN
        PERFORM * FROM (
            SELECT * FROM large_table 
            ORDER BY data 
            LIMIT 10000
        ) t;
    ELSIF test_name = 'aggregation' THEN
        PERFORM number, COUNT(*) 
        FROM large_table 
        GROUP BY number;
    END IF;
    
    end_time := clock_timestamp();
    execution_time := EXTRACT(MILLISECONDS FROM (end_time - start_time));
    
    -- บันทึกผล
    INSERT INTO performance_results (
        test_name, config_set, execution_time_ms
    ) VALUES (
        test_name, config_name, execution_time
    );
END;
$$ LANGUAGE plpgsql;
```

#### 8.2 การทดสอบก่อนและหลังการปรับแต่ง
```sql
-- ทดสอบกับการตั้งค่าใหม่
SELECT run_performance_test('large_sort', 'optimized');
SELECT run_performance_test('aggregation', 'optimized');

-- ดูผลการทดสอบ
SELECT 
    test_name,
    config_set,
    execution_time_ms,
    AVG(execution_time_ms) OVER (PARTITION BY test_name) as avg_time
FROM performance_results
ORDER BY test_timestamp DESC;
```
### ผลการทดลอง

1. รูปผลการทดลอง

<img width="552" height="182" alt="image" src="https://github.com/user-attachments/assets/72208bde-95ca-4d8f-91f8-c4b9c90eab9e" />

2. อธิบายผลลัพธ์ที่ได้
  0 rows) หมายความว่า คำสั่ง SELECT ไม่พบแถวให้แสดง จากตาราง performance_results ครับ สาเหตุที่เป็นไปได้:

ตารางยัง ไม่มีข้อมูล (ว่าง)
อ่าน ผิดสคีมา (มีตารางชื่อเดียวกันในสคีมาอื่น)
เงื่อนไข/สิทธิ์ (เช่นมี RLS) ทำให้มองไม่เห็นแถว


### Step 9: การ Monitoring และ Alerting

#### 9.1 สร้างระบบ Monitoring แบบ Real-time
```sql
-- สร้าง view สำหรับติดตาม memory usage
CREATE VIEW memory_monitor AS
SELECT 
    'shared_buffers' as component,
    setting as current_setting,
    pg_size_pretty(setting::bigint * 8192) as size_formatted,
    ROUND((setting::bigint * 8192) / (1024.0*1024*1024) * 100 / 2, 2) as percent_of_2gb
FROM pg_settings WHERE name = 'shared_buffers'
UNION ALL
SELECT 
    'work_mem',
    setting,
    pg_size_pretty(setting::bigint * 1024),
    ROUND((setting::bigint * 1024) / (1024.0*1024*1024) * 100 / 2, 2)
FROM pg_settings WHERE name = 'work_mem'
UNION ALL
SELECT 
    'maintenance_work_mem',
    setting,
    pg_size_pretty(setting::bigint * 1024),
    ROUND((setting::bigint * 1024) / (1024.0*1024*1024) * 100 / 2, 2)
FROM pg_settings WHERE name = 'maintenance_work_mem';

-- ใช้งาน view
SELECT * FROM memory_monitor;
```
### ผลการทดลอง

รูปผลการทดลอง

<img width="628" height="129" alt="image" src="https://github.com/user-attachments/assets/e79d4588-0a34-4ada-aef1-ee530da816d8" />


### Step 10: การจำลอง Load Testing

#### 10.1 สร้าง Synthetic Workload
```sql
-- สร้างตารางสำหรับ load testing
CREATE TABLE load_test_orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    price DECIMAL(10,2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE load_test_customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- สร้างข้อมูล customers
INSERT INTO load_test_customers (first_name, last_name, email)
SELECT 
    'FirstName' || i,
    'LastName' || i,
    'user' || i || '@example.com'
FROM generate_series(1, 10000) AS i;

-- สร้างข้อมูล orders (จำนวนมาก)
INSERT INTO load_test_orders (customer_id, product_id, quantity, price)
SELECT 
    (random() * 10000 + 1)::integer,
    (random() * 1000 + 1)::integer,
    (random() * 10 + 1)::integer,
    (random() * 1000 + 10)::decimal(10,2)
FROM generate_series(1, 1000000) AS i;

-- สร้าง indexes
CREATE INDEX idx_orders_customer_id ON load_test_orders(customer_id);
CREATE INDEX idx_orders_product_id ON load_test_orders(product_id);
CREATE INDEX idx_orders_date ON load_test_orders(order_date);
```
### ผลการทดลอง

รูปผลการทดลอง การสร้าง FUNCTION และ INDEX

<img width="223" height="106" alt="image" src="https://github.com/user-attachments/assets/9e67307a-4ada-44a7-9585-f75e0ea6bf30" />


#### 10.2 การทดสอบ Query Performance
```sql
-- ฟังก์ชันสำหรับทดสอบ concurrent queries
CREATE OR REPLACE FUNCTION simulate_oltp_workload(
    iterations INTEGER DEFAULT 100
) RETURNS TABLE(
    operation_type TEXT,
    avg_time_ms NUMERIC,
    min_time_ms NUMERIC,
    max_time_ms NUMERIC,
    total_operations INTEGER
) AS $$
DECLARE
    i INTEGER;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    select_times NUMERIC[] := ARRAY[]::NUMERIC[];
    insert_times NUMERIC[] := ARRAY[]::NUMERIC[];
    update_times NUMERIC[] := ARRAY[]::NUMERIC[];
    delete_times NUMERIC[] := ARRAY[]::NUMERIC[];
    current_time_ms NUMERIC;
    random_customer_id INTEGER;
    random_order_id INTEGER;
BEGIN
    -- สร้างตารางทดสอบถ้ายังไม่มี
    CREATE TABLE IF NOT EXISTS load_test_customers (
        customer_id SERIAL PRIMARY KEY,
        first_name VARCHAR(50),
        last_name VARCHAR(50),
        email VARCHAR(100),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE TABLE IF NOT EXISTS load_test_orders (
        order_id SERIAL PRIMARY KEY,
        customer_id INTEGER REFERENCES load_test_customers(customer_id),
        price DECIMAL(10,2),
        quantity INTEGER,
        order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    -- สร้างข้อมูลตัวอย่างถ้าตารางว่าง
    IF NOT EXISTS (SELECT 1 FROM load_test_customers LIMIT 1) THEN
        INSERT INTO load_test_customers (first_name, last_name, email)
        SELECT 
            'Customer' || i,
            'LastName' || i,
            'customer' || i || '@example.com'
        FROM generate_series(1, 1000) i;
        
        INSERT INTO load_test_orders (customer_id, price, quantity, order_date)
        SELECT 
            (random() * 999 + 1)::INTEGER,
            (random() * 1000 + 10)::DECIMAL(10,2),
            (random() * 10 + 1)::INTEGER,
            CURRENT_TIMESTAMP - (random() * 60 || ' days')::INTERVAL
        FROM generate_series(1, 5000) i;
    END IF;
    
    -- ทดสอบ SELECT queries (JOIN + WHERE)
    FOR i IN 1..iterations LOOP
        start_time := clock_timestamp();
        
        PERFORM o.order_id, c.first_name, c.last_name, o.price
        FROM load_test_orders o
        JOIN load_test_customers c ON o.customer_id = c.customer_id
        WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days'
        LIMIT 10;
        
        end_time := clock_timestamp();
        current_time_ms := EXTRACT(MILLISECONDS FROM (end_time - start_time));
        select_times := array_append(select_times, current_time_ms);
    END LOOP;
    
    -- ทดสอบ INSERT operations
    FOR i IN 1..iterations LOOP
        start_time := clock_timestamp();
        
        random_customer_id := (random() * 999 + 1)::INTEGER;
        INSERT INTO load_test_orders (customer_id, price, quantity)
        VALUES (
            random_customer_id,
            (random() * 1000 + 10)::DECIMAL(10,2),
            (random() * 10 + 1)::INTEGER
        );
        
        end_time := clock_timestamp();
        current_time_ms := EXTRACT(MILLISECONDS FROM (end_time - start_time));
        insert_times := array_append(insert_times, current_time_ms);
    END LOOP;
    
    -- ทดสอบ UPDATE operations
    FOR i IN 1..iterations LOOP
        start_time := clock_timestamp();
        
        random_order_id := (SELECT order_id FROM load_test_orders ORDER BY random() LIMIT 1);
        UPDATE load_test_orders 
        SET price = price * 1.1,
            quantity = quantity + 1
        WHERE order_id = random_order_id;
        
        end_time := clock_timestamp();
        current_time_ms := EXTRACT(MILLISECONDS FROM (end_time - start_time));
        update_times := array_append(update_times, current_time_ms);
    END LOOP;
    
    -- ทดสอบ DELETE operations (soft delete - เพิ่ม column deleted_at)
    ALTER TABLE load_test_orders ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP;
    
    FOR i IN 1..iterations LOOP
        start_time := clock_timestamp();
        
        random_order_id := (SELECT order_id FROM load_test_orders 
                           WHERE deleted_at IS NULL 
                           ORDER BY random() LIMIT 1);
        UPDATE load_test_orders 
        SET deleted_at = CURRENT_TIMESTAMP
        WHERE order_id = random_order_id;
        
        end_time := clock_timestamp();
        current_time_ms := EXTRACT(MILLISECONDS FROM (end_time - start_time));
        delete_times := array_append(delete_times, current_time_ms);
    END LOOP;
    
    -- Return results
    RETURN QUERY 
    SELECT 
        'SELECT (JOIN + WHERE)'::TEXT,
        ROUND(AVG(t), 3),
        ROUND(MIN(t), 3),
        ROUND(MAX(t), 3),
        array_length(select_times, 1)
    FROM unnest(select_times) AS t
    
    UNION ALL
    
    SELECT 
        'INSERT'::TEXT,
        ROUND(AVG(t), 3),
        ROUND(MIN(t), 3),
        ROUND(MAX(t), 3),
        array_length(insert_times, 1)
    FROM unnest(insert_times) AS t
    
    UNION ALL
    
    SELECT 
        'UPDATE'::TEXT,
        ROUND(AVG(t), 3),
        ROUND(MIN(t), 3),
        ROUND(MAX(t), 3),
        array_length(update_times, 1)
    FROM unnest(update_times) AS t
    
    UNION ALL
    
    SELECT 
        'DELETE (soft)'::TEXT,
        ROUND(AVG(t), 3),
        ROUND(MIN(t), 3),
        ROUND(MAX(t), 3),
        array_length(delete_times, 1)
    FROM unnest(delete_times) AS t;
    
END;
$$ LANGUAGE plpgsql;
```

-- รัน load test ทดสอบเบาๆ
SELECT * FROM simulate_oltp_workload(25);


### ผลการทดลอง

<img width="609" height="239" alt="image" src="https://github.com/user-attachments/assets/1953f83a-cf95-402a-99ba-b0938e16e47b" />

รูปผลการทดลอง
```
-- ทดสอบปานกลาง  
SELECT * FROM simulate_oltp_workload(100);
### ผลการทดลอง
```
1. รูปผลการทดลอง

<img width="616" height="204" alt="image" src="https://github.com/user-attachments/assets/661ed22c-c718-407a-947b-e64d3f638477" />

2. อธิบายผลการทดลอง การ SELECT , INSERT, UPDATE, DELETE เป็นอย่างไร 

1) SELECT (JOIN + WHERE) — เร็วมาก (~0.05 ms/ครั้ง)
สะท้อนว่าเคสที่รัน มีดัชนีตรงกับเงื่อนไข (เช่น customer_id, product_id, order_date) และ/หรือข้อมูลอยู่ใน shared buffers (แคช) แล้ว
ไม่มี sort/spill และขนาดผลลัพธ์เล็ก (มี LIMIT หรือเลือกจุดเฉพาะ)
แทบไม่แตะดิสก์จริง ⇒ CPU-bound เล็กน้อย
ทิป: รักษาดัชนีที่สอดคล้องกับ pattern จริง (WHERE, JOIN, ORDER BY) และ ANALYZE สม่ำเสมอ

2) INSERT — เร็วมาก (~0.03 ms/ครั้ง)

เขียนบรรทัดเดียว/ไม่ซับซ้อน, ไม่มี FK/Trigger หนัก ๆ
ดัชนีบนตารางมีอยู่ แต่จำนวนไม่มาก จึงยังไม่เกิด write amplification สูง
อาจยังอยู่ในช่วง OS cache + WAL group commit ช่วยให้ latency ต่ำ
ทิป: bulk-insert ขนาดใหญ่ค่อยสร้าง index ทีหลังจะยิ่งเร็ว; งาน benchmark ปิด synchronous_commit = off ชั่วคราวได้ถ้ารับความเสี่ยงได้

3) UPDATE — ช้าที่สุด (~132 ms/ครั้ง)

สาเหตุที่พบบ่อย:

Write amplification: UPDATE = ลบเวอร์ชันเก่า + เขียนเวอร์ชันใหม่ → กระทบ ทุกดัชนี ที่ครอบคลุมคอลัมน์ที่เปลี่ยน
ถ้าแก้ คอลัมน์ที่อยู่ในดัชนี → บังคับอัปเดต entry ใน index (แพง)
หากหน้าเต็มจนใส่เวอร์ชันใหม่ไม่ได้ → เกิด page split หรือย้ายหน้า (ช้า)
ถ้าค่า HOT update ใช้ไม่ได้ (เพราะมีคอลัมน์ indexed ถูกแก้) → ต้องไปแตะ index เสมอ
WAL/ฟลัช (fsync) และการรอ autovacuum/visibility map


4) DELETE (soft) — ช้า (~152 ms/ครั้ง)

“soft delete” = อัปเดตคอลัมน์ deleted_at (จริง ๆ คือ UPDATE ชนิดหนึ่ง)
ช้าด้วยเหตุผลใกล้เคียง UPDATE:
การเซต deleted_at แตะคอลัมน์ใหม่ → กระทบดัชนีที่อาจครอบคลุมคอลัมน์นี้ (ถ้ามี)
หากมี partial index/constraint ที่อ้าง deleted_at → มีค่าใช้จ่ายดูเงื่อนไขเพิ่ม
ต้องรอให้ autovacuum ทำความสะอาดเวอร์ชันเก่า (dead tuples) ในภายหลัง


-- ทดสอบหนักขึ้น เครื่องใครไม่ไหวผ่านก่อน หรือเปลี่ยนค่า 500 เป็น 200 :)
SELECT * FROM simulate_oltp_workload(500);
### ผลการทดลอง

รูปผลการทดลอง

<img width="659" height="148" alt="image" src="https://github.com/user-attachments/assets/9e6a05bb-57e1-49ba-9367-50b25a76f783" />


### Step 11: การเปรียบเทียบประสิทธิภาพ

#### 11.1 การทดสอบแบบ Before/After
```sql
-- สร้างตารางเก็บผลการทดสอบ (ถ้ายังไม่มี)
CREATE TABLE IF NOT EXISTS benchmark_results (
    test_id SERIAL PRIMARY KEY,
    config_name VARCHAR(50),
    test_type VARCHAR(50),
    shared_buffers_mb INTEGER,
    work_mem_mb INTEGER,
    execution_time_ms NUMERIC,
    buffer_hit_ratio NUMERIC,
    test_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ฟังก์ชันบันทึกผลการทดสอบ
CREATE OR REPLACE FUNCTION record_benchmark(
    config_name TEXT,
    test_type TEXT
) RETURNS TABLE(
    config TEXT,
    test_name TEXT,
    exec_time_ms NUMERIC,
    buffer_hit_percent NUMERIC,
    shared_buffers_mb INTEGER,
    work_mem_mb INTEGER
) AS $$
DECLARE
    sb_mb INTEGER;
    wm_mb INTEGER;
    exec_time NUMERIC;
    hit_ratio NUMERIC;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
BEGIN
    -- ดึงค่า configuration ปัจจุบัน
    SELECT (setting::bigint * 8192) / (1024*1024) INTO sb_mb 
    FROM pg_settings WHERE name = 'shared_buffers';
    
    SELECT setting::INTEGER / 1024 INTO wm_mb 
    FROM pg_settings WHERE name = 'work_mem';
    
    -- Reset statistics เพื่อวัดผลแม่นยำ
    PERFORM pg_stat_reset();
    
    -- เริ่มจับเวลา
    start_time := clock_timestamp();
    
    -- รัน test ตาม type ที่กำหนด
    IF test_type = 'sort_heavy' THEN
        -- ทดสอบ sorting operations
        PERFORM * FROM (
            SELECT number, data, 
                   ROW_NUMBER() OVER (ORDER BY data, number) as rn
            FROM large_table 
            ORDER BY data, number
            LIMIT 50000
        ) t;
        
    ELSIF test_type = 'agg_heavy' THEN
        -- ทดสอบ aggregation operations  
        PERFORM customer_id, 
                COUNT(*) as order_count,
                SUM(price) as total_revenue,
                AVG(quantity) as avg_quantity,
                MAX(order_date) as latest_order
        FROM load_test_orders 
        WHERE order_date >= CURRENT_DATE - INTERVAL '90 days'
        GROUP BY customer_id
        HAVING COUNT(*) > 2
        ORDER BY total_revenue DESC;
        
    ELSIF test_type = 'join_heavy' THEN
        -- ทดสอบ complex joins
        PERFORM c.customer_id, c.first_name, c.last_name,
                COUNT(o.order_id) as total_orders,
                SUM(o.price) as total_spent,
                AVG(o.quantity) as avg_order_size
        FROM load_test_customers c
        LEFT JOIN load_test_orders o ON c.customer_id = o.customer_id
        WHERE o.order_date >= CURRENT_DATE - INTERVAL '60 days'
        GROUP BY c.customer_id, c.first_name, c.last_name
        ORDER BY total_spent DESC NULLS LAST
        LIMIT 1000;
        
    ELSIF test_type = 'index_scan' THEN
        -- ทดสอบ index scans
        PERFORM COUNT(*) 
        FROM large_table 
        WHERE number BETWEEN 1000 AND 5000;
        
    ELSE
        -- Default: simple select
        PERFORM COUNT(*) FROM large_table WHERE data > 0.5;
    END IF;
    
    -- จบการจับเวลา
    end_time := clock_timestamp();
    exec_time := EXTRACT(MILLISECONDS FROM (end_time - start_time));
    
    -- คำนวณ buffer hit ratio
    SELECT CASE 
        WHEN blks_read + blks_hit = 0 THEN 100.0
        ELSE ROUND((blks_hit::NUMERIC / (blks_read + blks_hit)) * 100, 2)
    END INTO hit_ratio
    FROM pg_stat_database 
    WHERE datname = current_database();
    
    -- บันทึกผลลง database
    INSERT INTO benchmark_results (
        config_name, test_type, shared_buffers_mb, work_mem_mb, 
        execution_time_ms, buffer_hit_ratio
    ) VALUES (
        record_benchmark.config_name, 
        record_benchmark.test_type, 
        sb_mb, 
        wm_mb, 
        exec_time, 
        COALESCE(hit_ratio, 0)
    );
    
    -- Return ผลลัพธ์
    RETURN QUERY SELECT 
        record_benchmark.config_name,
        record_benchmark.test_type,
        exec_time,
        COALESCE(hit_ratio, 0),
        sb_mb,
        wm_mb;
END;
$$ LANGUAGE plpgsql;

-- ฟังก์ชันสำหรับ benchmark หลายๆ configuration
CREATE OR REPLACE FUNCTION run_benchmark_suite()
RETURNS TABLE(
    config TEXT,
    test_name TEXT,
    exec_time_ms NUMERIC,
    buffer_hit_percent NUMERIC,
    performance_score NUMERIC
) AS $$
BEGIN
    -- ทดสอบกับ configuration ปัจจุบัน
    RETURN QUERY
    SELECT r.config, r.test_name, r.exec_time_ms, r.buffer_hit_percent,
           ROUND(1000.0 / NULLIF(r.exec_time_ms, 0), 2) as performance_score
    FROM (
        SELECT * FROM record_benchmark('current_config', 'sort_heavy')
        UNION ALL
        SELECT * FROM record_benchmark('current_config', 'agg_heavy') 
        UNION ALL
        SELECT * FROM record_benchmark('current_config', 'join_heavy')
        UNION ALL  
        SELECT * FROM record_benchmark('current_config', 'index_scan')
    ) r;
END;
$$ LANGUAGE plpgsql;

-- ฟังก์ชันดู benchmark history
CREATE OR REPLACE FUNCTION view_benchmark_history(days_back INTEGER DEFAULT 7)
RETURNS TABLE(
    test_date DATE,
    config_name TEXT,
    test_type TEXT,
    avg_exec_time NUMERIC,
    avg_hit_ratio NUMERIC,
    test_count BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        test_timestamp::DATE as test_date,
        benchmark_results.config_name,
        benchmark_results.test_type,
        ROUND(AVG(execution_time_ms), 2) as avg_exec_time,
        ROUND(AVG(buffer_hit_ratio), 2) as avg_hit_ratio,
        COUNT(*) as test_count
    FROM benchmark_results
    WHERE test_timestamp >= CURRENT_DATE - (days_back || ' days')::INTERVAL
    GROUP BY test_timestamp::DATE, benchmark_results.config_name, benchmark_results.test_type
    ORDER BY test_date DESC, avg_exec_time ASC;
END;
$$ LANGUAGE plpgsql;
```

```sql
-- ทดสอบกับ configuration ปัจจุบัน
SELECT * FROM run_benchmark_suite();
```
### ผลการทดลอง

รูปผลการทดลอง

<img width="643" height="166" alt="image" src="https://github.com/user-attachments/assets/92494666-92bf-4955-b9c8-9277a4a66585" />


-- ดูผลการทดสอบ
```
SELECT 
    config_name,
    test_type,
    shared_buffers_mb || 'MB' as shared_buffers,
    work_mem_mb || 'MB' as work_mem,
    ROUND(execution_time_ms, 2) as exec_time_ms,
    ROUND(buffer_hit_ratio, 2) as hit_ratio_percent,
    test_timestamp
FROM benchmark_results
ORDER BY test_timestamp DESC;
```
### ผลการทดลอง

รูปผลการทดลอง

<img width="890" height="189" alt="image" src="https://github.com/user-attachments/assets/172a8d15-12bd-49c9-ad67-db2e0516966f" />


### Step 12: การจัดการ Configuration แบบ Advanced

#### 12.1 การสร้าง Configuration Profiles
```sql
-- สร้างตาราง config templates
CREATE TABLE IF NOT EXISTS config_templates (
    template_id SERIAL PRIMARY KEY,
    template_name VARCHAR(50) UNIQUE,
    parameter_name VARCHAR(100),
    parameter_value VARCHAR(100),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- เพิ่ม configuration profiles สำหรับ workload ต่างๆ
INSERT INTO config_templates (template_name, parameter_name, parameter_value, description) VALUES
-- Small OLTP Profile (สำหรับระบบเล็ก < 2GB RAM)
('small_oltp', 'shared_buffers', '256MB', 'Buffer cache for small systems'),
('small_oltp', 'work_mem', '4MB', 'Conservative work memory'),
('small_oltp', 'maintenance_work_mem', '128MB', 'Maintenance operations memory'),
('small_oltp', 'effective_cache_size', '1GB', 'OS cache size estimation'),
('small_oltp', 'random_page_cost', '1.1', 'SSD optimized'),

-- Medium OLTP Profile (สำหรับระบบปานกลาง 2-8GB RAM)
('medium_oltp', 'shared_buffers', '512MB', 'Buffer cache for medium systems'),
('medium_oltp', 'work_mem', '8MB', 'Moderate work memory'),
('medium_oltp', 'maintenance_work_mem', '256MB', 'Maintenance operations memory'),
('medium_oltp', 'effective_cache_size', '3GB', 'OS cache size estimation'),
('medium_oltp', 'random_page_cost', '1.1', 'SSD optimized'),

-- Large OLTP Profile (สำหรับระบบใหญ่ > 8GB RAM)
('large_oltp', 'shared_buffers', '2GB', 'Buffer cache for large systems'),
('large_oltp', 'work_mem', '16MB', 'Higher work memory'),
('large_oltp', 'maintenance_work_mem', '512MB', 'Maintenance operations memory'),
('large_oltp', 'effective_cache_size', '6GB', 'OS cache size estimation'),
('large_oltp', 'random_page_cost', '1.05', 'NVMe SSD optimized'),

-- Analytics Profile (สำหรับ Data Warehouse)
('analytics', 'shared_buffers', '1GB', 'Buffer cache for analytics'),
('analytics', 'work_mem', '64MB', 'High work memory for complex queries'),
('analytics', 'maintenance_work_mem', '1GB', 'High maintenance memory'),
('analytics', 'effective_cache_size', '8GB', 'Large OS cache'),
('analytics', 'random_page_cost', '1.1', 'SSD optimized'),

-- Development Profile (สำหรับ Development)
('development', 'shared_buffers', '128MB', 'Small buffer cache'),
('development', 'work_mem', '2MB', 'Low work memory'),
('development', 'maintenance_work_mem', '64MB', 'Low maintenance memory'),
('development', 'effective_cache_size', '512MB', 'Small OS cache'),
('development', 'random_page_cost', '1.1', 'SSD optimized')
ON CONFLICT (template_name, parameter_name) DO NOTHING;

-- ฟังก์ชันสำหรับใช้ configuration profile
CREATE OR REPLACE FUNCTION apply_config_profile(
    profile_name TEXT
) RETURNS TEXT AS $$
DECLARE
    config_rec RECORD;
    result_text TEXT := '';
    profile_count INTEGER;
BEGIN
    -- ตรวจสอบว่า profile มีอยู่จริง
    SELECT COUNT(*) INTO profile_count
    FROM config_templates
    WHERE template_name = profile_name;
    
    IF profile_count = 0 THEN
        RETURN 'Error: Profile "' || profile_name || '" not found';
    END IF;
    
    -- ใช้ configuration จาก template
    FOR config_rec IN
        SELECT parameter_name, parameter_value, description
        FROM config_templates
        WHERE template_name = profile_name
        ORDER BY parameter_name
    LOOP
        EXECUTE format('ALTER SYSTEM SET %I = %L',
                      config_rec.parameter_name,
                      config_rec.parameter_value);
        
        result_text := result_text || config_rec.parameter_name || ' = ' ||
                      config_rec.parameter_value || ' (' || 
                      COALESCE(config_rec.description, '') || ')' || E'\n';
    END LOOP;
    
    -- Reload configuration
    PERFORM pg_reload_conf();
    
    RETURN 'Applied configuration profile: ' || profile_name || E'\n' ||
           'Changes applied:' || E'\n' || result_text ||
           E'\nNote: Some settings may require restart to take effect.';
END;
$$ LANGUAGE plpgsql;

-- ฟังก์ชันสำหรับ auto-tune based on workload
CREATE OR REPLACE FUNCTION auto_tune_memory()
RETURNS TEXT AS $$
DECLARE
    total_connections INTEGER;
    active_connections INTEGER;
    avg_query_time NUMERIC;
    buffer_hit_ratio NUMERIC;
    current_shared_buffers_mb INTEGER;
    current_work_mem_mb INTEGER;
    recommendation TEXT := '';
    temp_files_count BIGINT;
    temp_files_size BIGINT;
BEGIN
    -- ดึงสถิติปัจจุบัน
    SELECT COUNT(*) INTO total_connections
    FROM pg_stat_activity;
    
    SELECT COUNT(*) INTO active_connections
    FROM pg_stat_activity 
    WHERE state = 'active';
    
    -- ดู average query time (ถ้ามี pg_stat_statements)
    SELECT AVG(mean_exec_time) INTO avg_query_time
    FROM pg_stat_statements
    WHERE calls > 10;
    
    -- คำนวณ buffer hit ratio
    SELECT 
        ROUND((SUM(heap_blks_hit)::NUMERIC / 
               NULLIF(SUM(heap_blks_read + heap_blks_hit), 0)) * 100, 2)
    INTO buffer_hit_ratio
    FROM pg_statio_user_tables;
    
    -- ดึงค่า config ปัจจุบัน
    SELECT (setting::BIGINT * 8192) / (1024*1024) INTO current_shared_buffers_mb
    FROM pg_settings WHERE name = 'shared_buffers';
    
    SELECT setting::INTEGER / 1024 INTO current_work_mem_mb
    FROM pg_settings WHERE name = 'work_mem';
    
    -- ดู temp files usage
    SELECT SUM(temp_files), SUM(temp_bytes) INTO temp_files_count, temp_files_size
    FROM pg_stat_database;
    
    -- เริ่มสร้างคำแนะนำ
    recommendation := 'Current System Analysis:' || E'\n';
    recommendation := recommendation || '- Total connections: ' || total_connections || E'\n';
    recommendation := recommendation || '- Active connections: ' || active_connections || E'\n';
    recommendation := recommendation || '- Buffer hit ratio: ' || COALESCE(buffer_hit_ratio, 0) || '%' || E'\n';
    recommendation := recommendation || '- Current shared_buffers: ' || current_shared_buffers_mb || 'MB' || E'\n';
    recommendation := recommendation || '- Current work_mem: ' || current_work_mem_mb || 'MB' || E'\n';
    recommendation := recommendation || '- Avg query time: ' || COALESCE(ROUND(avg_query_time, 2), 0) || 'ms' || E'\n';
    recommendation := recommendation || '- Temp files: ' || COALESCE(temp_files_count, 0) || E'\n\n';
    
    recommendation := recommendation || 'Recommendations:' || E'\n';
    
    -- Logic สำหรับแนะนำการปรับแต่ง
    IF COALESCE(buffer_hit_ratio, 0) < 95 THEN
        recommendation := recommendation || '⚠️  Buffer hit ratio ต่ำ (' || 
                         COALESCE(buffer_hit_ratio, 0) || '%) - ควรเพิ่ม shared_buffers' || E'\n';
        recommendation := recommendation || '   แนะนำ: เพิ่ม shared_buffers เป็น ' || 
                         (current_shared_buffers_mb * 1.5)::INTEGER || 'MB' || E'\n';
    ELSE
        recommendation := recommendation || '✅ Buffer hit ratio ดี (' || 
                         COALESCE(buffer_hit_ratio, 0) || '%)' || E'\n';
    END IF;
    
    IF COALESCE(temp_files_count, 0) > 1000 THEN
        recommendation := recommendation || '⚠️  Temp files มาก (' || temp_files_count || 
                         ') - work_mem อาจไม่เพียงพอ' || E'\n';
        recommendation := recommendation || '   แนะนำ: เพิ่ม work_mem เป็น ' || 
                         (current_work_mem_mb * 2)::INTEGER || 'MB' || E'\n';
    END IF;
    
    IF COALESCE(avg_query_time, 0) > 1000 AND active_connections < 50 THEN
        recommendation := recommendation || '⚠️  Query ช้า (avg: ' || 
                         ROUND(avg_query_time, 2) || 'ms) - ควรเพิ่ม work_mem' || E'\n';
        recommendation := recommendation || '   แนะนำ: เพิ่ม work_mem เป็น ' || 
                         (current_work_mem_mb * 1.5)::INTEGER || 'MB' || E'\n';
    END IF;
    
    IF active_connections > 100 THEN
        recommendation := recommendation || '⚠️  Connections เยอะ (' || active_connections || 
                         ') - ควรลด work_mem เพื่อป้องกัน memory exhaustion' || E'\n';
        recommendation := recommendation || '   แนะนำ: ลด work_mem เป็น ' || 
                         (current_work_mem_mb / 2)::INTEGER || 'MB' || E'\n';
    END IF;
    
    -- ถ้าไม่มีปัญหา
    IF COALESCE(buffer_hit_ratio, 0) >= 95 AND 
       COALESCE(temp_files_count, 0) < 1000 AND
       active_connections <= 100 AND
       COALESCE(avg_query_time, 0) <= 1000 THEN
        recommendation := recommendation || '✅ Configuration ปัจจุบันดูเหมาะสมแล้ว' || E'\n';
    END IF;
    
    -- แนะนำ profile
    recommendation := recommendation || E'\nSuggested Profiles:' || E'\n';
    IF current_shared_buffers_mb < 256 THEN
        recommendation := recommendation || '- ลอง: SELECT apply_config_profile(''small_oltp'');' || E'\n';
    ELSIF current_shared_buffers_mb < 1024 THEN
        recommendation := recommendation || '- ลอง: SELECT apply_config_profile(''medium_oltp'');' || E'\n';
    ELSE
        recommendation := recommendation || '- ลอง: SELECT apply_config_profile(''large_oltp'');' || E'\n';
    END IF;
    
    RETURN recommendation;
END;
$$ LANGUAGE plpgsql;

-- ฟังก์ชันดู available profiles
CREATE OR REPLACE FUNCTION list_config_profiles()
RETURNS TABLE(
    profile_name TEXT,
    parameter_count BIGINT,
    description TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        template_name,
        COUNT(*) as param_count,
        CASE template_name
            WHEN 'small_oltp' THEN 'Small OLTP systems (<2GB RAM)'
            WHEN 'medium_oltp' THEN 'Medium OLTP systems (2-8GB RAM)'
            WHEN 'large_oltp' THEN 'Large OLTP systems (>8GB RAM)'
            WHEN 'analytics' THEN 'Analytics/Data Warehouse workloads'
            WHEN 'development' THEN 'Development environment'
            ELSE 'Custom profile'
        END as description
    FROM config_templates
    GROUP BY template_name
    ORDER BY template_name;
END;
$$ LANGUAGE plpgsql;

-- ฟังก์ชันเปรียบเทียบ current config กับ profile
CREATE OR REPLACE FUNCTION compare_with_profile(profile_name TEXT)
RETURNS TABLE(
    parameter_name TEXT,
    current_value TEXT,
    profile_value TEXT,
    status TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        ct.parameter_name,
        COALESCE(pg.setting, 'NOT SET') as current_value,
        ct.parameter_value as profile_value,
        CASE 
            WHEN pg.setting = ct.parameter_value THEN '✅ MATCH'
            WHEN pg.setting IS NULL THEN '❌ NOT SET'
            ELSE '⚠️  DIFFERENT'
        END as status
    FROM config_templates ct
    LEFT JOIN pg_settings pg ON ct.parameter_name = pg.name
    WHERE ct.template_name = profile_name
    ORDER BY ct.parameter_name;
END;
$$ LANGUAGE plpgsql;
```

-- ใช้งาน auto-tuning
SELECT auto_tune_memory();

### ผลการทดลอง

<img width="642" height="177" alt="image" src="https://github.com/user-attachments/assets/6dcb64f6-6392-477b-add4-6fb8cf9f39b4" />

รูปผลการทดลอง
```
```sql
-- ดูการเปลี่ยนแปลง buffer hit ratio
SELECT 
    relname,
    heap_blks_read,
    heap_blks_hit,
    ROUND((heap_blks_hit::decimal / NULLIF(heap_blks_read + heap_blks_hit, 0)) * 100, 2) as hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY hit_ratio;
```
### ผลการทดลอง

รูปผลการทดลอง

<img width="762" height="275" alt="image" src="https://github.com/user-attachments/assets/1bde3c3c-2bf4-4f6e-a457-2da19f557348" />

### การคำนวณ Memory Requirements

#### สูตรคำนวณพื้นฐาน
```
Total PostgreSQL Memory = shared_buffers + 
                         (work_mem × max_connections × sessions_per_connection) +
                         maintenance_work_mem +
                         wal_buffers +
                         other_buffers

แนะนำให้ไม่เกิน 75-80% ของ total system RAM
```

#### ตัวอย่างการคำนวณ
```
System RAM = 8GB
shared_buffers = 2GB
work_mem = 32MB
max_connections = 100
maintenance_work_mem = 512MB
wal_buffers = 64MB

Estimated Usage = 2GB + (32MB × 100 × 0.5) + 512MB + 64MB
                = 2GB + 1.6GB + 512MB + 64MB
                = 4.176GB (52% ของ system RAM) ← ปลอดภัย
```


## คำถามท้ายการทดลอง
1. หน่วยความจำใดบ้างที่เป็น shared memory และมีหลักในการตั้งค่าอย่างไร
shared_buffers → บล็อกข้อมูลของตาราง/ดัชนีที่แคช “ร่วมกัน” ทุกคอนเนกชัน
แนวทาง: เริ่มที่ 20–25% ของ RAM เครื่อง (OLTP ปกติ 4–8 GB ก็พอสำหรับ RAM 16–32 GB) แล้วดู pg_statio_*/hit ratio เพื่อปรับเพิ่ม-ลด

wal_buffers → บัฟเฟอร์ WAL ในหน่วยความจำ “ร่วมกัน”
แนวทาง: ตั้ง -1 (auto) ให้ PostgreSQLคำนวณเอง (≈ 1/32 ของ shared_buffers โดยมีเพดาน) ยกเว้นงาน write-heavy ขนาดใหญ่ค่อยตั้งเอง

(ภายในยังมี shared mem อื่น ๆ เช่น lock tables, proc arrays ฯลฯ แต่ผู้ดูแลปรับได้จริงหลัก ๆ คือ 2 ตัวบน)
ไม่ใช่ shared: work_mem, maintenance_work_mem (เป็น “ต่อปฏิบัติการ/ต่อ worker”)

2. Work memory และ maintenance work memory คืออะไร มีหลักการในการกำหนดค่าอย่างไร
work_mem: เพดาน RAM ต่อ 1 โหนด ของการทำงาน เช่น Sort, HashAggregate/HashJoin ต่อ worker/ต่อ parallel
แนวทาง:

OLTP เริ่ม 4–16 MB; Analytics/ETL อาจ 64–512 MB เฉพาะ session/job นั้น (SET work_mem='128MB';)
อย่าคูณตรง ๆ ด้วย max_connections อย่างเดียว—ให้คิด จำนวนโหนดที่อาจ active พร้อมกัน (มัก 1–3 ต่อคิวรี × จำนวนคิวรีพร้อมกัน)
maintenance_work_mem: เพดาน RAM ของ งานบำรุงรักษา เช่น VACUUM, CREATE INDEX, ALTER TABLE ADD FOREIGN KEY, CLUSTER
แนวทาง:
เซิร์ฟเวอร์ 16 GB: ตั้ง 512 MB–2 GB (ขึ้นกับงาน CREATE INDEX ใหญ่แค่ไหน)
autovacuum_work_mem ถ้าเป็น -1 จะใช้ค่าจาก maintenance_work_mem

3. หากมี RAM 16GB และต้องการกำหนด connection = 200 ควรกำหนดค่า work memory และ maintenance work memory อย่างไร
สมมติ shared_buffers = 4GB, wal_buffers = -1 (auto), maintenance_work_mem = 1GB
สมมติช่วงพีกมี คิวรีพร้อมกัน 50 ตัว และ เฉลี่ยมี 2 โหนดที่กิน work_mem ต่อคิวรี → โหนดพร้อมกัน ≈ 100
ถ้าตั้ง work_mem = 16MB → งบ work_mem ≈ 100 × 16MB = 1.6GB

4. ไฟล์ postgresql.conf และ postgresql.auto.conf  มีความสัมพันธ์กันอย่างไร
postgresql.conf = ไฟล์คอนฟิกหลัก (คุณแก้ตรง ๆ ได้)
postgresql.auto.conf = ไฟล์ที่ ถูกเขียนโดยคำสั่ง ALTER SYSTEM และ ถูก include “ท้ายสุด” → มีลำดับความสำคัญสูงกว่า ค่าที่ตั้งใน postgresql.conf
ถ้าค่าชนกัน จะใช้ค่าจาก postgresql.auto.conf
ถ้าไม่ต้องการ override ให้แก้ (หรือลบ) ใน postgresql.auto.conf แล้วรีสตาร์ท

5. Buffer hit ratio คืออะไร
hit_ratio = hits / (hits + reads)
ค่า สูง (เช่น 99%+) ⇒ ส่วนใหญ่เสิร์ฟจากแคช → เร็ว, I/O ต่ำ
ค่า ต่ำ มี disk read มาก → ตรวจ shared_buffers, แบบแผนคิวรี, ดัชนี, และหน่วยความจำของระบบโดยรวม

6. แสดงผลการคำนวณ การกำหนดค่าหน่วยความจำต่าง ๆ โดยอ้างอิงเครื่องของตนเอง
คิวรีดูค่าปัจจุบัน:
```
SELECT
  name, setting, unit
FROM pg_settings
WHERE name IN ('shared_buffers','work_mem','maintenance_work_mem','wal_buffers','max_connections');

-- แปลงเป็นหน่วยอ่านง่าย
SELECT
  name,
  CASE unit
    WHEN '8kB' THEN pg_size_pretty(setting::bigint * 8192)
    WHEN 'kB'  THEN pg_size_pretty(setting::bigint * 1024)
    WHEN 'MB'  THEN setting || ' MB'
    ELSE setting || COALESCE(' '||unit,'')
  END AS value
FROM pg_settings
WHERE name IN ('shared_buffers','work_mem','maintenance_work_mem','wal_buffers','max_connections');

```
ดู RAM ของเครื่อง/คอนเทนเนอร์:
```
docker exec -it postgres-config free -h
```

คำนวณงบประมาณอย่างเร็ว (สมมติ active_queries และ nodes_per_query ตามงานจริง):
```
Total ≈ shared_buffers
      + (work_mem × active_queries × nodes_per_query)
      + maintenance_work_mem
      + wal_buffers(+เผื่ออื่น ๆ)
เป้า ≤ 0.75–0.80 × RAM
```
จากผลก่อนหน้าในแล็บของคุณ: hit ratio = 100% หลายตาราง แปลว่า shared_buffers ปัจจุบัน “เอาอยู่” สำหรับเวิร์กโหลดทดลอง
7. การสแกนของฐานข้อมูล PostgreSQL มีกี่แบบอะไรบ้าง เปรียบเทียบการสแกนแต่ละแบบ
Seq Scan (สแกนทั้งตาราง)
ดียามต้องอ่านสัดส่วนมากของตาราง/ไม่มีดัชนีที่ช่วย; ไม่ต้องกระโดดดิสก์

Index Scan (เดินตามดัชนี แล้วไปอ่าน heap รายแถว)
ดีเมื่อคัดเลือกน้อย มีเงื่อนไขตรงกับดัชนี; อาจสุ่มอ่าน heap หลายที่

Index-Only Scan (อ่านจากดัชนีอย่างเดียว)
เร็วมากถ้า “คอลัมน์ที่ต้องใช้อยู่ในดัชนีทั้งหมด” และ หน้า heap เป็น all-visible (visibility map) → Heap Fetches ≈ 0

Bitmap Index Scan + Bitmap Heap Scan
เลือกได้ดีเมื่อ “แมตช์ปานกลางถึงมาก” → ดึงตำแหน่งจากดัชนีจำนวนมากมาก่อน แล้วรวม/เรียงตำแหน่ง ค่อยกระโดดไปอ่าน heap เป็นชุด (ลด random I/O)

(พบน้อย) Tid Scan (รู้ TID ตรง ๆ), Sample Scan (TABLESAMPLE), CTE/Materialize (ไม่ใช่สแกนชนิดฐานข้อมูลแต่มีผล I/O)

เลือกแบบไหนเร็วที่สุด?
ขึ้นกับ selectivity + ดัชนี + รูปร่างคิวรี:
น้อยมาก → Index Scan/Index-Only
ปานกลาง-มาก → Bitmap
มากมาก/ไม่มี index เหมาะ → Seq Scan

มี LIMIT + ORDER BY <indexed> → มัก Index(-Only) Scan + Limit (ไม่ต้อง sort)
