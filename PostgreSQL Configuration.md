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
or
docker run --name postgres-config -e POSTGRES_PASSWORD=admin123 -e POSTGRES_DB=configdb -p 5432:5432 -v postgres-config-data:/var/lib/postgresql/data --memory=2g --cpus=2 -d postgres

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
ใช้เพื่อเรียกดูสถานะการใช้หน่วยความจำ (memory usage) ภายใน container ที่ชื่อ postgres-config โดยคำสั่ง free เป็นคำสั่งในระบบลินุกซ์ที่แสดงข้อมูลเกี่ยวกับหน่วยความจำ เช่น จำนวนหน่วยความจำทั้งหมด หน่วยความจำที่ใช้ หน่วยความจำว่าง และ swap memory

2. option -h ในคำสั่งมีผลอย่างไร
ให้แสดงผลลัพธ์เป็นหน่วยที่อ่านง่าย (human-readable) เช่น แสดงเป็น MB, GB แทนที่จะเป็นหน่วยบล็อกหรือไบต์ดิบ ซึ่งช่วยให้เข้าใจข้อมูลได้ง่ายขึ้น

3. docker exec postgres-config nproc  แสดงค่าผลลัพธ์อย่างไร
แสดงจำนวน CPU cores ที่ container postgres-config สามารถใช้งานได้ คำสั่ง nproc ในลินุกซ์จะแสดงจำนวนโปรเซสเซอร์ที่พร้อมใช้งานในระบบ ค่าผลลัพธ์คือจำนวนเต็มที่ระบุจำนวน CPU cores ที่ระบบหรือ container นั้นเห็นและสามารถใช้งานได้
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
2. ตำแหน่งที่อยู่ของไฟล์ data อยู่ที่ตำแหน่งใด
<img width="536" height="145" alt="image" src="https://github.com/user-attachments/assets/925c99df-d0ba-4235-b22b-51da89d22e83" />    
<img width="498" height="148" alt="image" src="https://github.com/user-attachments/assets/02453e90-8623-47df-8f60-8c4b7ac5bb57" />    
<img width="399" height="136" alt="image" src="https://github.com/user-attachments/assets/f2958dcb-6875-459d-9e6c-2393c4d9f7a0" />    

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
<img width="1310" height="402" alt="image" src="https://github.com/user-attachments/assets/9e59a468-aea1-4a53-a639-666cefcbe38e" />

### Step 2: การปรับแต่งพารามิเตอร์แบบค่อยเป็นค่อยไป

#### 2.1 ปรับแต่ง Shared Buffers (ต้อง restart)
```sql
-- ตรวจสอบค่าปัจจุบัน
SELECT name, setting, unit, source, pending_restart
FROM pg_settings 
WHERE name = 'shared_buffers';

### ผลการทดลอง
```
<img width="712" height="299" alt="image" src="https://github.com/user-attachments/assets/493587e6-aa6a-4f14-a250-ac4f6a7475de" />

1.รูปผลการรันคำสั่ง
2. ค่า  shared_buffers มีการกำหนดค่าไว้เท่าไหร่ (ใช้ setting X unit)
3. ค่า  pending_restart ในผลการทดลองมีค่าเป็นอย่างไร และมีความหมายอย่างไร
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
<img width="774" height="381" alt="image" src="https://github.com/user-attachments/assets/d8bd8ef2-b8be-4125-842f-5bfd7bbc1eaa" />

-- ออกจาก postgres prompt (กด \q แล้ว enter) ทำการ Restart PostgreSQL ด้วยคำสั่ง แล้ว run docker อีกครั้ง หรือใช้วิธีการ stop และ run containner
docker exec -it -u postgres postgres-config pg_ctl restart -D /var/lib/postgresql/data -m fast

### ผลการทดลอง
```
รูปผลการเปลี่ยนแปลงค่า pending_restart
รูปหลังจาก restart postgres

```
<img width="693" height="327" alt="image" src="https://github.com/user-attachments/assets/65f0b3b4-59de-4adb-ad55-1be981053736" />


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
```
รูปผลการเปลี่ยนแปลงค่า work_mem
```
<img width="525" height="192" alt="image" src="https://github.com/user-attachments/assets/5af68938-1d1f-4671-8828-65ab150423f0" />

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
```
รูปผลการเปลี่ยนแปลงค่า maintenance_work_mem
```
<img width="507" height="144" alt="image" src="https://github.com/user-attachments/assets/c6e029f7-cf85-4b51-ac4b-217e5825bcbe" />

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
```
รูปผลการเปลี่ยนแปลงค่า wal_buffers
```
<img width="359" height="144" alt="image" src="https://github.com/user-attachments/assets/466d7df7-3ea2-4086-bf40-a4eb77ba375d" />

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
```
รูปผลการเปลี่ยนแปลงค่า effective_cache_size
```
<img width="451" height="155" alt="image" src="https://github.com/user-attachments/assets/3217e9aa-f6e0-4a4f-94e5-b7f01dad68df" />

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
```
รูปผลการลัพธ์การตั้งค่า
```
<img width="1312" height="284" alt="image" src="https://github.com/user-attachments/assets/0a0df04c-e0d3-48d8-a243-855b106ea2bd" />

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
```
1. คำสั่ง EXPLAIN(ANALYZE,BUFFERS) คืออะไร
คำสั่ง EXPLAIN ใช้ใน PostgreSQL เพื่อแสดงแผนการดำเนินการ (execution plan) ของคิวรี SQL การเพิ่ม option ANALYZE จะทำให้ PostgreSQL รันคำสั่งจริงและแสดงเวลาที่ใช้ในการประมวลผลแต่ละขั้นตอน ส่วน BUFFERS จะรายงานการใช้งานแคชข้อมูล (buffer usage) โดยบอกว่าข้อมูลโหลดจากแคชหรือดิสก์ ทำให้เห็นละเอียดว่าคิวรีใช้ทรัพยากรอย่างไรและมีประสิทธิภาพแค่ไหน

2. รูปผลการรัน

3. อธิบายผลลัพธ์ที่ได้
Limit: ใช้เพื่อจำกัดจำนวนผลลัพธ์ที่แสดง (1000 แถว) พร้อมค่า cost และเวลาดำเนินการจริง (actual time)
Gather Merge: เป็นการนำข้อมูลจาก worker หลายตัวมารวมและเรียงลำดับ โดยใช้ parallel query processing มี worker จริง 2 ตัว
Sort: การ sort ใช้วิธี top-N heapsort ซึ่งใช้หน่วยความจำ 239kB, 179kB, 168kB สำหรับ worker แต่ละคน ใช้งาน work_mem ตามค่าเหล่านี้
Parallel Seq Scan on large_table: การสแกนตารางด้วย parallel scan เพื่อดึงข้อมูลจากตาราง large_table มี buffer hit 5059 ครั้ง แสดงว่าข้อมูลส่วนใหญ่ถูกอ่านจาก cache
Buffers: แสดงจำนวน page ที่อ่านผ่าน shared buffer ซึ่งช่วยวิเคราะห์การใช้ cache ใน query นี้
Planning Time: เวลาที่ใช้ในการวางแผน query เพียง 0.151 ms
```
<img width="1305" height="503" alt="image" src="https://github.com/user-attachments/assets/1d010214-0f19-41ae-938c-e8ef35df592c" />

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
```
1. รูปผลการรัน

2. อธิบายผลลัพธ์ที่ได้
Limit: จำกัดผลลัพธ์ที่แสดงเพียง 100 แถว ใช้เวลาแค่ 0.201 ms
GroupAggregate: ดำเนินการ group ข้อมูลตาม number พร้อมใช้ filter (นับเฉพาะ group ที่ count > 1) และใช้เวลารันจริงที่รวดเร็วมาก (0.191 ms)
Index Only Scan: PostgreSQL เลือกใช้ index idx_large_table_number แทนการอ่านข้อมูลจากตารางโดยตรง (Heap Fetches: 0) ส่งผลให้ query ทำงานเร็ว เพราะข้อมูลที่ต้องใช้ถูกเก็บครบใน index แล้ว
Buffers: มีแต่ shared hit (อ่านจาก cache) ไม่ต้องเข้าถึงบนดิสก์ และไม่มีการ fetch ข้อมูลจาก heap จริง
Planning Time / Execution Time: ใช้เวลาเตรียมแผน 6.080 ms และใช้เวลารวมทั้งคิวรีเพียง 0.260 ms

3. การสแกนเป็นแบบใด เกิดจากเหตุผลใด
Index Only Scan: PostgreSQL ใช้วิธีสแกนเฉพาะดัชนี (Index Only Scan) เพราะต้องการแค่ข้อมูลในคอลัมน์ number ซึ่งอยู่ใน index แล้ว ทำให้ไม่ต้องอ่านข้อมูลจริงทั้งแถว (Heap Fetches: 0)
เหตุผล: เลือกสแกนแบบนี้เพราะ query ใช้เฉพาะคอลัมน์ที่มี index ครบ และค่าใน index เพียงพอต่อการสร้างผลลัพธ์ จึงประหยัดเวลาและ IO อีกทั้งข้อมูลถูกอ่านจาก buffer cache ทั้งหมด (shared hit) เพิ่มความเร็วในการประมวลผลอย่างมาก
```
<img width="1323" height="506" alt="image" src="https://github.com/user-attachments/assets/af327acb-f09e-4c3c-86fe-b539e0f397b8" />

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
```
1. รูปผลการทดลอง จากคำสั่ง VACUUM (ANALYZE, VERBOSE) large_table;
2. อธิบายผลลัพธ์ที่ได้
ระบบจะรายงานรายละเอียดจำนวนแถวที่ถูกลบหรือ reclaimed อัตโนมัติ และค่าอื่น ๆ ที่เกี่ยวข้องกับการทำ vacuum
จะแสดงข้อมูลว่าสถิติอะไรในแต่ละคอลัมน์ถูกอัปเดตและประเมินค่าต่าง ๆ เพื่อช่วยวางแผน query ในอนาคต
ช่วยให้ฐานข้อมูลทำงานเร็วขึ้น และมีประสิทธิภาพในการจัดการพื้นที่จัดเก็บข้อมูล
การใช้ VERBOSE ทำให้รู้ขั้นตอนและสถานะของกระบวนการ vacuum และ analyze ช่วยสำหรับตรวจสอบ และแก้ไขปัญหาได้ดีขึ้น
```
<img width="1289" height="323" alt="image" src="https://github.com/user-attachments/assets/38a75f29-8580-452e-a702-bfeee2a2706d" />
<img width="1316" height="419" alt="image" src="https://github.com/user-attachments/assets/229a1417-b121-4e7e-b48f-385feca07bbb" />

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
```
รูปผลการทดลอง
```
<img width="878" height="386" alt="image" src="https://github.com/user-attachments/assets/3690f9b4-025b-4683-b056-1f6499f63a96" />

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
```
1. รูปผลการทดลอง
2. อธิบายผลลัพธ์ที่ได้
แสดงข้อมูลของดาต้าเบสปัจจุบัน
จำนวน heap_blks_read คือการอ่านข้อมูลจากดิสก์จริงซึ่งช้ากว่า
จำนวน heap_blks_hit คือการอ่านข้อมูลจากแคชในหน่วยความจำที่เร็วกว่ามาก
อัตรา hit_ratio_percent บอกว่าในช่วงเวลาที่เก็บสถิติ query มีการดึงข้อมูลจาก cache กี่เปอร์เซ็นต์ ยิ่งสูงแปลว่าฐานข้อมูลทำงานได้เร็วขึ้นเพราะไม่ต้องเข้าถึงดิสก์บ่อย
การจัดเรียง ORDER BY heap_blks_read + heap_blks_hit DESC คือเรียงลำดับตารางที่ถูกอ่านบ่อยที่สุดก่อน
```
<img width="1099" height="439" alt="image" src="https://github.com/user-attachments/assets/e1f6c265-b591-49ea-85f5-5345a5e110de" />

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
```
1. รูปผลการทดลอง
2. อธิบายผลลัพธ์ที่ได้
แสดงข้อมูลรายละเอียดของการอ่านข้อมูลในแต่ละตารางของดาต้าเบสปัจจุบัน
heap_blks_read คือจำนวนบล็อกที่ถูกอ่านจากดิสก์สำหรับตารางนั้น
heap_blks_hit คือจำนวนบล็อกที่ถูกอ่านจากแคช (shared buffer) สำหรับตารางนั้น
hit_ratio_percent เป็นเปอร์เซ็นต์การอ่านข้อมูลจากแคชในแต่ละตาราง
การเรียงลำดับตามจำนวนการอ่านข้อมูลรวม (ดิสก์+แคช) เพื่อแสดงตารางที่ถูกใช้งานมากที่สุดก่อน
```
<img width="963" height="300" alt="image" src="https://github.com/user-attachments/assets/ccaac25b-9dc8-4cc9-9254-12921b687d2b" />

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
```
1. รูปผลการทดลอง
2. อธิบายผลลัพธ์ที่ได้
แสดงผลสถิติของการเข้าถึงข้อมูลในตารางที่ถูกอ่านจากดิสก์ในฐานข้อมูลปัจจุบัน โดยเฉพาะแสดง 10 ตารางที่มีการอ่านข้อมูลจากดิสก์มากที่สุด พร้อมขนาดของตารางแต่ละตัว รายละเอียดดังนี้:
schemaname: ชื่อ schema ที่ตารางนั้นอยู่
tablename: ชื่อตาราง
heap_blks_read: จำนวนบล็อกข้อมูลที่ถูกอ่านจากดิสก์ (disk reads) ในตารางนั้น
heap_blks_hit: จำนวนบล็อกข้อมูลที่ถูกอ่านจาก cache ใน shared buffers
total_access: ผลรวมการเข้าถึงข้อมูลทั้งจากดิสก์และ cache
hit_ratio_percent: เปอร์เซ็นต์ของการอ่านจาก cache เทียบกับทั้งหมด (ค่าประสิทธิภาพ cache hit rate)
table_size: ขนาดของตารางโดยรวม (ข้อมูล + indexes + TOAST) แสดงแบบอ่านง่าย (เช่น MB, GB)
```
<img width="1037" height="409" alt="image" src="https://github.com/user-attachments/assets/5ff81e15-7736-4b62-bf41-7a5f9dc49b02" />

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
```
1. รูปผลการทดลอง
2. อธิบายค่าต่าง ๆ ที่มีความสำคัญ
autovacuum: ควบคุมว่าเปิดหรือปิด autovacuum (on = เปิดใช้งาน) เป็นค่าที่ต้องเปิดไว้เสมอเพื่อความเสถียรและประสิทธิภาพ
autovacuum_analyze_scale_factor: อัตราส่วนของจำนวนแถวที่เปลี่ยนแปลง (insert/update/delete) เมื่อเทียบกับจำนวนแถวในตารางที่ทำให้เกิดการวิเคราะห์สถิติใหม่ (analyze) เช่น 0.1 หมายถึงเปลี่ยนแปลง 10% ของตารางจะ trigger analyze
autovacuum_analyze_threshold: จำนวนขั้นต่ำของแถวที่เปลี่ยนแปลงก่อนเริ่ม analyze (เช่น 50 แถว)
autovacuum_freeze_max_age: อายุของ transaction ID ที่จะทำให้ autovacuum ทำการ freeze tuples เพื่อป้องกัน transaction ID wraparound (ค่าปกติสูงมากเช่น 200000000)
autovacuum_max_workers: จำนวน autovacuum worker ที่รันพร้อมกันสูงสุด (เช่น 3)
autovacuum_naptime: เวลาที่จะพัก (sleep) ระหว่างแต่ละรอบของ autovacuum (เช่น 60 วินาที)
autovacuum_vacuum_cost_delay: การหน่วงเวลา (delay) ในหน่วย millisecond ระหว่างขั้นตอน vacuum เพื่อลดผลกระทบต่อ performance ระบบ (เช่น 2 ms)
autovacuum_vacuum_cost_limit: จำนวนคะแนนสูงสุดที่อนุญาตในแต่ละรอบ vacuum ก่อนจะต้องหยุดพัก (ค่า -1 หมายถึงใช้ค่าเริ่มต้น)
autovacuum_vacuum_insert_scale_factor และ autovacuum_vacuum_insert_threshold: กำหนดเกณฑ์การ trigger autovacuum เมื่อตารางมีการแทรกข้อมูลใหม่ เป็นสัดส่วนกับจำนวนแถวทั้งหมดและจำนวนแถวขั้นต่ำ
autovacuum_vacuum_scale_factor และ autovacuum_vacuum_threshold: กำหนดเกณฑ์การ trigger autovacuum เมื่อตารางมีการแก้ไขหรือลบข้อมูล เป็นสัดส่วนกับจำนวนแถวทั้งหมดและจำนวนแถวขั้นต่ำ
autovacuum_work_mem: หน่วยความจำสูงสุดที่ autovacuum worker แต่ละตัวใช้ในการทำงาน (ค่า -1 หมายถึงใช้ค่าเริ่มต้นของระบบ)
```
<img width="1311" height="503" alt="image" src="https://github.com/user-attachments/assets/956f8675-b5cc-48fb-95c4-197b26795362" />

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
```
รูปผลการทดลองการปรับแต่ง Autovacuum (Capture รวมทั้งหมด 1 รูป)
```
<img width="842" height="366" alt="image" src="https://github.com/user-attachments/assets/725e63c5-b13e-42cd-9848-89dcb61f2dd7" />

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
```
1. รูปผลการทดลอง
2. อธิบายผลลัพธ์ที่ได้
แสดงผลข้อมูลการทดสอบประสิทธิภาพที่เก็บในตาราง performance_results โดยมีข้อมูลสำคัญ คือ
test_name: ชื่อของการทดสอบ (เช่น ชื่อ workload หรือ test case)
config_set: การตั้งค่าหรือชุดการตั้งค่าที่ใช้ทดสอบในแต่ละครั้ง
execution_time_ms: เวลาที่ใช้รันทดสอบในหน่วยมิลลิวินาที
avg_time: ค่าเฉลี่ยเวลาที่ใช้ทดสอบของแต่ละ test_name โดยคำนวณจากข้อมูลทดสอบทั้งหมดในกลุ่มนั้น (ใช้ฟังก์ชัน window AVG() OVER (PARTITION BY))
```
<img width="809" height="295" alt="image" src="https://github.com/user-attachments/assets/7fb18a95-748c-4093-a3c2-516791bbaee5" />

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
```
รูปผลการทดลอง
```
<img width="914" height="437" alt="image" src="https://github.com/user-attachments/assets/cf5eeb43-11e1-4443-a34b-02f6df72e62b" />

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
```
รูปผลการทดลอง การสร้าง FUNCTION และ INDEX
```
<img width="870" height="217" alt="image" src="https://github.com/user-attachments/assets/b4978456-e37c-4927-b47b-d1b0785537d5" />
<img width="906" height="221" alt="image" src="https://github.com/user-attachments/assets/e2122e42-1f3c-40dc-9624-94b20811acc4" />

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
<img width="872" height="380" alt="image" src="https://github.com/user-attachments/assets/76329441-b20a-49ba-990b-c452833e4131" />

รูปผลการทดลอง
```
-- ทดสอบปานกลาง  
SELECT * FROM simulate_oltp_workload(100);
### ผลการทดลอง
```
<img width="906" height="412" alt="image" src="https://github.com/user-attachments/assets/a7ed5b6b-5074-431b-99c7-b4c96b8d775b" />

1. รูปผลการทดลอง
2. อธิบายผลการทดลอง การ SELECT , INSERT, UPDATE, DELETE เป็นอย่างไร
SELECT (JOIN + WHERE): ใช้เวลาโดยเฉลี่ยต่ำสุดในบรรดา operations เพราะเป็นการอ่านข้อมูลแบบคัดกรอง ส่วนที่ทำการ join และ where กับจำนวนแถวจำกัด (limit 10) ทำให้ประสิทธิภาพดี
INSERT: ใช้เวลามากกว่า SELECT เพราะมีการเพิ่มข้อมูลใหม่ลงในตาราง orders และต้องอัพเดต index ด้วย
UPDATE: ใช้เวลามากกว่ากลุ่มอื่น ๆ เล็กน้อย เพราะต้องอ่านข้อมูลขึ้นมาแก้ไขและเขียนกลับ รวมถึงต้องรักษาความถูกต้องของ index และ constraints
DELETE (soft delete): ทำโดย update คอลัมน์ deleted_at แทนการลบจริง ทำให้ใช้เวลาใกล้เคียงกับ update
```

-- ทดสอบหนักขึ้น เครื่องใครไม่ไหวผ่านก่อน หรือเปลี่ยนค่า 500 เป็น 200 :)
SELECT * FROM simulate_oltp_workload(500);
### ผลการทดลอง
```
<img width="916" height="413" alt="image" src="https://github.com/user-attachments/assets/8ca6109f-a67c-4cd8-a3bf-ed3e071010e9" />

รูปผลการทดลอง
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
```
รูปผลการทดลอง
```
<img width="828" height="241" alt="image" src="https://github.com/user-attachments/assets/12a4affd-c3b6-4626-a106-55c6ccb4fcfc" />

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
รูปผลการทดลอง
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
รูปผลการทดลอง
<img width="799" height="191" alt="image" src="https://github.com/user-attachments/assets/4299174e-eb45-4d11-ad55-80620c574530" />

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
```
รูปผลการทดลอง
```
<img width="963" height="420" alt="image" src="https://github.com/user-attachments/assets/a0502afd-8293-4f4c-a1b5-222430e15d35" />

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
shared_buffers คือหน่วยความจำหลักที่ PostgreSQL ใช้เป็น shared memory สำหรับ cache data blocks ที่อ่านจากดิสก์
ระดับการตั้งค่าแนะนำคือ 15%-40% ของ RAM ทั้งหมด (เช่น RAM 16GB ตั้ง shared_buffers ประมาณ 4GB)
ต้องรีสตาร์ทเซิร์ฟเวอร์หลังเปลี่ยนค่า
ค่าอื่นที่เกี่ยวข้องกับ shared memory เช่น max_connections ต้องตั้งให้อยู่ในขอบเขตที่ OS รองรับ

2. Work memory และ maintenance work memory คืออะไร มีหลักการในการกำหนดค่าอย่างไร
work_mem คือหน่วยความจำที่ PostgreSQL จัดสรรให้แต่ละ operation ภายใน query เช่น การ sort, hash join ก่อนใช้ disk temporary file ถ้าไม่พอ
maintenance_work_mem คือหน่วยความจำที่ใช้กับงานบำรุงรักษาอย่าง vacuum, create index, alter table
ค่าที่แนะนำคือ
work_mem ควรตั้งให้อยู่ในระดับที่เหมาะสมกับจำนวน connection สูงสุด เพื่อไม่ให้เกิดการใช้หน่วยความจำเกินระบบ
maintenance_work_mem มักตั้งสูงกว่า work_mem เพราะงานเหล่านี้ใช้เวลานานและต้องประสิทธิภาพสูง

3. หากมี RAM 16GB และต้องการกำหนด connection = 200 ควรกำหนดค่า work memory และ maintenance work memory อย่างไร
สมมติแบ่ง RAM 25% ให้ shared_buffers = 4GB
เหลือ 12GB สำหรับ work_mem และอื่น ๆ
คำนวณ work_mem = (16GB × 0.25) / 200 = 0.02GB = 20MB (โดยประมาณ)
maintenance_work_mem ควรตั้งสูงกว่า เช่น 256MB หรือมากกว่า ขึ้นกับ workload

4. ไฟล์ postgresql.conf และ postgresql.auto.conf  มีความสัมพันธ์กันอย่างไร
postgresql.conf คือไฟล์หลักสำหรับตั้งค่าฐานข้อมูล ที่ผู้ดูแลสามารถแก้ไขได้
postgresql.auto.conf คือไฟล์ที่ระบบ PostgreSQL ใช้เก็บค่าที่ถูกตั้งผ่านคำสั่ง ALTER SYSTEM อัตโนมัติ ไม่ควรแก้โดยตรง
เมื่อตั้งค่าผ่าน ALTER SYSTEM ค่าจะถูกบันทึกใน postgresql.auto.conf และจะถูกโหลดรวมกับ postgresql.conf
ช่วยแยกการตั้งค่าที่เปลี่ยนผ่านคำสั่งกับไฟล์ตั้งค่าหลัก

5. Buffer hit ratio คืออะไร
เป็นอัตราการอ่านข้อมูลจาก shared buffer cache เทียบกับการอ่านข้อมูลทั้งหมดจากดิสก์และ cache รวม
ค่า hit ratio สูง หมายถึงฐานข้อมูลใช้แคชได้ดี ลดการอ่านดิสก์บ่อย ช่วยเพิ่มประสิทธิภาพ
คำนวณจากค่า pg_stat หรือ pg_statio

6. แสดงผลการคำนวณ การกำหนดค่าหน่วยความจำต่าง ๆ โดยอ้างอิงเครื่องของตนเอง
<img width="928" height="312" alt="image" src="https://github.com/user-attachments/assets/e10cb837-612d-4b79-866d-6b80c564872f" />

7. การสแกนของฐานข้อมูล PostgreSQL มีกี่แบบอะไรบ้าง เปรียบเทียบการสแกนแต่ละแบบ
Sequential Scan (Seq Scan): อ่านข้อมูลทีละแถวจากดิสก์ หรือไฟล์ข้อมูลเต็มตาราง เหมาะกับตารางเล็กหรือต้องการอ่านข้อมูลทั้งหมด
Index Scan: ใช้ index เพื่อค้นหาแถวที่ต้องการโดยตรง ลดจำนวนข้อมูลที่อ่าน เหมาะกับเงื่อนไขการคิวรีที่ตรงกับ index
Index Only Scan: เช่น Index Scan แต่ข้อมูลทั้งหมดจะถูกเก็บใน index ไม่ต้องอ่านข้อมูลหลัก ช่วยเพิ่มความเร็ว
Bitmap Index Scan: สร้าง bitmap แสดงตำแหน่งข้อมูลที่ต้องอ่าน จากนั้น อ่านข้อมูลเหมาะกับ query ที่คืนข้อมูลจำนวนมาก
Tid Scan: อ่านข้อมูลโดยใช้ tuple id ตรง ๆ จาก index เหมาะกับบางกรณีพิเศษ
Parallel Scan: การแบ่งงานสแกนข้อมูลตารางให้หลาย process ช่วยเพิ่มความเร็วในเครื่อง multi-core
