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
1.1docker exec postgres-config free
สั่งให้เข้าไป รันคำสั่ง free ใน container ที่ชื่อ postgres-config
free คือ การใช้ดู หน่วยความจำ (RAM)
1.2docker exec postgres-config df
สั่งให้เข้าไป รันคำสั่ง df ใน container
df ใช้ดู การใช้งานพื้นที่ดิสก์ (disk space) ของ filesystem และบอกว่า mount ไหนใช้ไปกี่ % และเหลือกี่ %
2. option -h ในคำสั่งมีผลอย่างไร
ทำให้แสดงผลในรูปแบบอ่านง่าย เช่น MB, GB
3. docker exec postgres-config nproc  แสดงค่าผลลัพธ์อย่างไร
แสดงเป็นตัวเลข
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
1. ตำแหน่งที่อยู่ของไฟล์ configuration อยู่ที่ตำแหน่งใด                                                                /var/lib/postgresql/data/postgresql.conf
3. ตำแหน่งที่อยู่ของไฟล์ data อยู่ที่ตำแหน่งใด                                                                                        /var/lib/postgresql/data
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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 05 41" src="https://github.com/user-attachments/assets/16d1a471-e8a5-44bd-abd2-3ceadb757b44" />

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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 08 38" src="https://github.com/user-attachments/assets/0f0cffb2-27b4-4b09-9214-663ae207ccc7" />

2. ค่า  shared_buffers มีการกำหนดค่าไว้เท่าไหร่ (ใช้ setting X unit)
setting 16384 unit8kB 
3. ค่า  pending_restart ในผลการทดลองมีค่าเป็นอย่างไร และมีความหมายอย่างไร
มีค่าเป็น f หมายถึง flag ที่บอกว่า การตั้งค่า (parameter) ใน PostgreSQL จำเป็นต้อง restart instance ถึงจะมีผลหรือไม่
ค่า f = false → ไม่ต้อง restart (เปลี่ยนแล้วมีผลทันที หรือ reload config ก็พอ)
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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 15 12" src="https://github.com/user-attachments/assets/7eef33f5-52c0-4fd0-80bb-e93496ed8e55" />

รูปหลังจาก restart postgres
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 17 40" src="https://github.com/user-attachments/assets/de546728-0117-4137-9c31-312eec1837df" />



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

<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 19 28" src="https://github.com/user-attachments/assets/5629e8fe-5608-49e9-ac13-ec761e974488" />


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
<img width="351" height="115" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 20 48" src="https://github.com/user-attachments/assets/491dd176-01c5-4836-9ac3-b6085c72a25d" />


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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 22 50" src="https://github.com/user-attachments/assets/390369de-521f-462c-99fd-4fc8319532fe" />


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
<img width="341" height="109" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 23 42" src="https://github.com/user-attachments/assets/0c2a08f2-e8a4-43b5-ba29-4d7ba1ee72e7" />


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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 24 31" src="https://github.com/user-attachments/assets/93fdf6eb-6615-428c-a4dc-827ec7a71f12" />


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
1.คำสั่ง EXPLAIN(ANALYZE,BUFFERS) คืออะไร
EXPLAIN = ใช้ดู Query Plan ว่า PostgreSQL จะเลือกใช้วิธีการไหนในการรัน query (เช่น Seq Scan, Index Scan, Join Method)
ANALYZE = ให้ PostgreSQL รัน query จริง (ไม่ใช่แค่คาดการณ์) แล้วแสดง เวลาจริง ที่ใช้ในแต่ละขั้นตอน
BUFFERS = แสดงข้อมูล การใช้หน่วยความจำ (shared buffers) เช่น อ่านจาก disk หรือ memory
2.รูปผลการรัน
<img width="2940" height="1912" alt="ภาพถ่ายหน้าจอ2568-09-16เวลา 14 37 01" src="https://github.com/user-attachments/assets/4ec29ab6-6a24-4bb2-9b8f-ddd3381e1f79" />
3.อธิบายผลลัพธ์ที่ได้

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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 41 38" src="https://github.com/user-attachments/assets/f0c541d5-fe89-42ff-a20c-122768bb3725" />
2. อธิบายผลลัพธ์ที่ได้ 
ผลลัพธ์นี้ PostgreSQL ใช้ Index Only Scan บน idx_large_table_number เพราะ query ใช้เฉพาะ column ที่อยู่ใน index (number) และมี visibility map ครอบคลุม
ไม่ต้องอ่านตารางจริง (Heap Fetches = 0) จึงทำงานเร็วมาก (0.412 ms)
4. การสแกนเป็นแบบใด เกิดจากเหตุผลใด
เป็นแบบ Index Only Scan
เกิดขึ้นเพราะ:
1.มี index บน column number
2.Query ใช้เพียง field ที่อยู่ใน index
3.ตารางมีค่า visibility map ที่บอกว่า tuple ใน heap ยัง valid → จึงไม่ต้องอ่าน heap
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
<img width="838" height="493" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 53 45" src="https://github.com/user-attachments/assets/3c255cc6-9a24-46aa-a90d-e8e646232d60" />
<img width="985" height="109" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 54 04" src="https://github.com/user-attachments/assets/b03191bb-e8d1-4926-b573-d8738593e408" />

2. อธิบายผลลัพธ์ที่ได้

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
<img width="628" height="297" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 58 22" src="https://github.com/user-attachments/assets/8f4e9296-a7c7-480f-86cb-12c5260d8997" />


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
1.รูปผลการทดลอง
<img width="718" height="309" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 14 59 10" src="https://github.com/user-attachments/assets/ba7813c6-49bf-4ec7-9ea0-26f74dea802e" />

2.อธิบายผลลัพธ์ที่ได้
หาข้อมูลในตารางแรกเอแต่ตารางสองไม่เจอ
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
1.รูปผลการทดลอง
<img width="692" height="205" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 00 26" src="https://github.com/user-attachments/assets/f7ff7a36-7cb8-4357-97f4-1a27fa17d411" />

2.อธิบายผลลัพธ์ที่ได้
แสดง สถิติประสิทธิภาพแคชโดยรวมของฐานข้อมูล ที่กำลังเชื่อมต่ออยู่ ซึ่งก็คือฐานข้อมูล postgres

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
1.รูปผลการทดลอง
<img width="875" height="318" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 01 30" src="https://github.com/user-attachments/assets/7aa389f4-af1e-4bbf-b37c-4427ab931c01" />

2.อธิบายผลลัพธ์ที่ได้
ไม่พบตารางใดๆ ที่มีการอ่านข้อมูลจากดิสก์ (Disk) เลย ในฐานข้อมูล postgres
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
1.รูปผลการทดลอง
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 02 42" src="https://github.com/user-attachments/assets/67fc0adc-0913-430e-b48d-daf05057c06b" />

2.อธิบายค่าต่าง ๆ ที่มีความสำคัญ
แสดงการตั้งค่าของ Autovacuum ซึ่งเป็นระบบบำรุงรักษาอัตโนมัติที่สำคัญที่สุดของ PostgreSQL 1: VACUUM (เก็บกวาดข้อมูลเก่า) 🧹 เมื่อคุณ UPDATE หรือ DELETE ข้อมูลใน PostgreSQL แถวเก่าจะไม่ได้ถูกลบออกไปทันที แต่จะถูกทำเครื่องหมายว่าเป็น "แถวที่ไม่ใช้แล้ว" (dead tuple) หน้าที่ของ VACUUM คือเข้ามาเก็บกวาดแถวเหล่านี้เพื่อนำพื้นที่กลับมาใช้ใหม่ 2: ANALYZE (อัปเดตสถิติ) 📊 เพื่อให้ Query ทำงานได้เร็ว ฐานข้อมูล (Query Planner) ต้องมีข้อมูลสถิติที่ทันสมัยเกี่ยวกับข้อมูลในตาราง (เช่น มีข้อมูลอะไรบ้าง, กระจายตัวอย่างไร) เพื่อจะได้เลือก "แผนการทำงาน" (Query Plan) ที่ดีที่สุด หน้าที่ของ ANALYZE ```
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
<img width="595" height="315" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 14 56" src="https://github.com/user-attachments/assets/aad4fb3b-42fc-447b-a4a6-496d99f9a6f9" />



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
1.รูปผลการทดลอง
<img width="585" height="189" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 17 47" src="https://github.com/user-attachments/assets/ee5d81e2-c06e-4bd1-bd89-c58ff8f7491d" />

2.อธิบายผลลัพธ์ที่ได้
มีข้อมูลอยู่ 1 row


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
<img width="607" height="133" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 18 59" src="https://github.com/user-attachments/assets/d5483e4b-c4ab-471e-b091-f11ccb44c927" />


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
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 21 42" src="https://github.com/user-attachments/assets/d157aaf2-329b-401d-b2ac-eefdb0e633d5" />


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

```
### ผลการทดลอง
```
<img width="1470" height="956" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 24 27" src="https://github.com/user-attachments/assets/ba438a6b-1b1b-4b2c-968b-80873cd16ffa" />

```
-- ทดสอบปานกลาง  
SELECT * FROM simulate_oltp_workload(100);
### ผลการทดลอง
```
1.รูปผลการทดลอง
<img width="721" height="214" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 28 14" src="https://github.com/user-attachments/assets/019e157b-70a7-4440-acdc-f8e44e036932" />

2.อธิบายผลการทดลอง การ SELECT , INSERT, UPDATE, DELETE เป็นอย่างไร 
SELECT (การอ่าน) �
      ถ้ามี Index: จะเร็วมาก เหมือนเปิดสารบัญหนังสือเจอหน้าที่ต้องการทันที
      ถ้าไม่มี Index: จะช้าลง เพราะต้องไล่อ่านทั้งตารางเหมือนอ่านหนังสือทั้งเล่ม
INSERT (การเพิ่ม) 
      การเพิ่มข้อมูล 1 แถว จะต้องเสียเวลาไป อัปเดต Index ทั้งหมด ที่มีอยู่ด้วยเล็กน้อย
UPDATE / DELETE (การแก้/ลบ) 
      ข้อมูลเก่า ไม่ได้ถูกลบทิ้งทันที แต่จะถูกเปลี่ยนสถานะเป็น "ขยะ" (Dead Tuples)
      "ขยะ" ที่สะสมมากๆ จะทำให้ตารางใหญ่ขึ้น (บวม) และทำงานช้าลง
      
```
-- ทดสอบหนักขึ้น เครื่องใครไม่ไหวผ่านก่อน หรือเปลี่ยนค่า 500 เป็น 200 :)
SELECT * FROM simulate_oltp_workload(500);
### ผลการทดลอง
```
<img width="713" height="214" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 30 27" src="https://github.com/user-attachments/assets/1def90ad-c918-4450-b5f2-162f2e29a58a" />

```

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

<img width="711" height="162" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 34 15" src="https://github.com/user-attachments/assets/f3a0047f-6935-4012-9191-a0f0e5db8f62" />



-- ดูผลการทดสอบ
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
```
<img width="918" height="328" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 34 47" src="https://github.com/user-attachments/assets/696a96fb-857a-4ce8-ae58-dc81a56c4f96" />

```

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
```
### ผลการทดลอง
```
<img width="380" height="259" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 40 16" src="https://github.com/user-attachments/assets/1fb0ae59-53c0-4880-a020-af0c867fdda2" />

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

<img width="821" height="296" alt="ภาพถ่ายหน้าจอ 2568-09-16 เวลา 15 40 49" src="https://github.com/user-attachments/assets/b1c95531-cf67-4517-ae44-c58e8eecdf7c" />



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
1.หน่วยความจำใดบ้างที่เป็น shared memory และมีหลักในการตั้งค่าอย่างไร
-shared_buffers ซึ่งทำหน้าที่เป็นแคชสำหรับเก็บข้อมูลจากดิสก์ โดยหลักการตั้งค่าเบื้องต้นที่แนะนำโดยทั่วไปคือ ประมาณ 25% ของ RAM ทั้งหมด สำหรับเซิร์ฟเวอร์ที่มี RAM      
ตั้งแต่ 1GB ขึ้นไป
2.Work memory และ maintenance work memory คืออะไร มีหลักการในการกำหนดค่าอย่างไร
-Work memory (work_mem) คือหน่วยความจำส่วนตัวที่ PostgreSQL ใช้สำหรับทำงานที่ซับซ้อนภายใน Query หนึ่งๆ เช่น การเรียงข้อมูล (Sort)
-Maintenance work memory (maintenance_work_mem) คือหน่วยความจำที่ใหญ่กว่าซึ่งสงวนไว้สำหรับงานบำรุงรักษาโดยเฉพาะ เช่น การสร้าง Index
หลักการกำหนดค่า
-work_mem: ตั้งค่าเริ่มต้นไว้ ไม่สูงนัก (เช่น 4MB - 16MB) เพราะทุกๆ connection ที่มีการทำงานซับซ้อนอาจขอใช้หน่วยความจำนี้พร้อมกันได้ การตั้งค่าสูงเกินไปอาจทำให้ RAM หมดได้
-maintenance_work_mem: ตั้งค่าให้ ใหญ่พอสมควร (เช่น 128MB - 2GB ขึ้นอยู่กับ RAM ทั้งหมด) เพื่อให้งานบำรุงรักษาระบบทำได้รวดเร็ว เนื่องจากงานเหล่านี้ไม่ได้ทำงานบ่อยและไม่ทำงานพร้อมกันหลายๆ โปรเซสเหมือน work_mem
3.หากมี RAM 16GB และต้องการกำหนด connection = 200 ควรกำหนดค่า work memory และ maintenance work memory อย่างไร
-work_mem = 16MB: เป็นการเริ่มต้นที่ปลอดภัย เพราะในสถานการณ์ที่แย่ที่สุด (200 connections ทำงานซับซ้อนพร้อมกัน) จะใช้ RAM สูงสุดที่ 200 * 16MB = 3.2GB
-maintenance_work_mem = 1GB ถึง 2GB: ตั้งค่าให้สูงได้เพื่อช่วยให้งานบำรุงรักษา เช่น VACUUM หรือ CREATE INDEX ทำงานได้เร็วขึ้น
4.ไฟล์ postgresql.conf และ postgresql.auto.conf  มีความสัมพันธ์กันอย่างไร
ค่าที่อยู่ใน postgresql.auto.conf จะมีผลบังคับใช้ทับค่าเดียวกันใน postgresql.conf เสมอ
5.Buffer hit ratio คืออะไร
-ค่าที่บ่งบอกประสิทธิภาพของแคช (Cache) ในฐานข้อมูล โดยคำนวณเป็นเปอร์เซ็นต์เพื่อแสดงว่า บ่อยแค่ไหนที่ฐานข้อมูลค้นหาข้อมูลเจอในหน่วยความจำ (RAM) ที่รวดเร็ว
เทียบกับการต้องไปเสียเวลาอ่านจากดิสก์ (Disk) ที่ช้ากว่า
6.แสดงผลการคำนวณ การกำหนดค่าหน่วยความจำต่าง ๆ โดยอ้างอิงเครื่องของตนเอง
shared_buffers = 2GB
การคำนวณ: 25% ของ RAM ทั้งหมด (8GB * 0.25)
เหตุผล: เป็นค่ามาตรฐานที่แบ่ง RAM ให้แคชหลักอย่างเหมาะสม โดยไม่กระทบกับหน่วยความจำของระบบปฏิบัติการ
work_mem = 8MB
การคำนวณ: เป็นค่าเริ่มต้นที่ปลอดภัยสำหรับ RAM ขนาดนี้ โดยคิดจาก 100 connections * 8MB = 800MB
เหตุผล: ป้องกันไม่ให้ RAM หมดเมื่อมีผู้ใช้หลายคนทำงานพร้อมกัน และยังเพียงพอสำหรับ Query ทั่วไป
maintenance_work_mem = 512MB
การคำนวณ: ประมาณ 6% ของ RAM ทั้งหมด
เหตุผล: ให้หน่วยความจำเพียงพอสำหรับงานบำรุงรักษาพื้นฐาน เช่น VACUUM บนตารางขนาดกลาง
effective_cache_size = 6GB
การคำนวณ: 75% ของ RAM ทั้งหมด (8GB * 0.75)
เหตุผล: บอกให้ PostgreSQL รู้ว่ามี RAM เหลือสำหรับ OS cache อีกมาก เพื่อช่วยให้ Planner เลือกแผนการทำงานที่ฉลาดขึ้น
7.การสแกนของฐานข้อมูล PostgreSQL มีกี่แบบอะไรบ้าง เปรียบเทียบการสแกนแต่ละแบบ
Sequential Scan: คือการอ่านข้อมูลทุกแถวในตาราง ตั้งแต่แถวแรกจนถึงแถวสุดท้าย เปรียบเสมือนการอ่านหนังสือทั้งเล่มเพื่อหาข้อมูล
Index Scan: คือการใช้ดัชนี (Index) เพื่อค้นหาตำแหน่งของข้อมูลแล้วกระโดดไปอ่านเฉพาะแถวที่ต้องการ เปรียบเสมือนการใช้สารบัญท้ายเล่มเพื่อเปิดไปที่หน้าๆ นั้นโดยตรง

การเปรียบเทียบ
Sequential Scan
วิธีการ: อ่านข้อมูลเรียงไปทีละแถวทั้งตาราง
ความเร็ว: เร็วเมื่อต้องการข้อมูลส่วนใหญ่ของตาราง
สถานการณ์ที่เหมาะ: Query ที่ไม่สามารถใช้ Index ได้ หรือ ทำงานบนตารางขนาดเล็กมาก

Index Scan
วิธีการ: ใช้ Index ค้นหาตำแหน่งแล้วจึงไปอ่านข้อมูล
ความเร็ว: เร็วมากเมื่อต้องการข้อมูลแค่ไม่กี่แถว
สถานการณ์ที่เหมาะ: Query ที่มีความเฉพาะเจาะจงสูง (Selective
