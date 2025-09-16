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
ตอบ : docker exec postgres-config free เป็นคำสั่งเข้าไปในคอนเทนเนอร์ postgres-config และเรียกใช้คำสั่ง free ซึ่งเป็นคำสั่งมาตรฐานของระบบปฏิบัติการ Linux/Unix เพื่อแสดงข้อมูลเกี่ยวกับการใช้งานหน่วยความจำ (RAM) ทั้งหมด, หน่วยความจำที่ใช้งานอยู่, และหน่วยความจำที่ว่างอยู่
      docker exec postgres-config df  เป็นคำสั่งเข้าไปในคอนเทนเนอร์ postgres-config และเรียกใช้คำสั่ง df (disk free) ซึ่งเป็นคำสั่งมาตรฐานของระบบปฏิบัติการ Linux/Unix เพื่อแสดงข้อมูลเกี่ยวกับการใช้งานพื้นที่บนดิสก์ (Disk space) ของระบบไฟล์ต่างๆ ภายในคอนเทนเนอร์

2. option -h ในคำสั่งมีผลอย่างไร
ตอบ : การใช้ option -h ในคำสั่งต่างๆ จะทำให้ผลลัพธ์ที่แสดงออกมาดูเข้าใจง่ายขึ้น โดยเฉพาะตัวเลขที่เกี่ยวกับขนาดไฟล์, พื้นที่ดิสก์, หรือหน่วยความจำ
3. docker exec postgres-config nproc  แสดงค่าผลลัพธ์อย่างไร
ตอบ : 12
<img width="898" height="46" alt="image" src="https://github.com/user-attachments/assets/78288803-5b3b-4b76-aecd-0efefe953abe" />

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
   ตอบ :  /var/lib/postgresql/data/postgresql.conf
   
2. ตำแหน่งที่อยู่ของไฟล์ data อยู่ที่ตำแหน่งใด
   ตอบ : /var/lib/postgresql/data
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

บันทึกรูปผลของ configuration ทั้ง 6 ค่า
<img width="1424" height="392" alt="image" src="https://github.com/user-attachments/assets/53672104-39c4-4944-bf61-95431dd4ad50" />


### Step 2: การปรับแต่งพารามิเตอร์แบบค่อยเป็นค่อยไป

#### 2.1 ปรับแต่ง Shared Buffers (ต้อง restart)
```sql
-- ตรวจสอบค่าปัจจุบัน
SELECT name, setting, unit, source, pending_restart
FROM pg_settings 
WHERE name = 'shared_buffers';

### ผลการทดลอง
```
<img width="725" height="208" alt="image" src="https://github.com/user-attachments/assets/4a870fe1-6028-4669-a099-a0732a67dfc4" />


1.รูปผลการรันคำสั่ง
2. ค่า  shared_buffers มีการกำหนดค่าไว้เท่าไหร่ (ใช้ setting X unit)
3. ค่า  pending_restart ในผลการทดลองมีค่าเป็นอย่างไร และมีความหมายอย่างไร
```
-- คำนวณและตั้งค่าใหม่
-- สำหรับระบบ 2GB: 512MB (25%)
ALTER SYSTEM SET shared_buffers = '512MB';
<img width="507" height="60" alt="image" src="https://github.com/user-attachments/assets/2bf27a47-dd65-42c4-8444-0f52f422719d" />

-- ตรวจสอบการเปลี่ยนแปลง
select pg_reload_conf();
SELECT name, setting, unit, source, pending_restart
FROM pg_settings 
WHERE name = 'shared_buffers';

```
-- ออกจาก postgres prompt (กด \q แล้ว enter) ทำการ Restart PostgreSQL ด้วยคำสั่ง แล้ว run docker อีกครั้ง หรือใช้วิธีการ stop และ run containner
docker exec -it -u postgres postgres-config pg_ctl restart -D /var/lib/postgresql/data -m fast

### ผลการทดลอง
```
รูปผลการเปลี่ยนแปลงค่า pending_restart
รูปหลังจาก restart postgres
```
<img width="834" height="378" alt="image" src="https://github.com/user-attachments/assets/06a4bb11-7db1-47ea-ac53-cb50c85ed926" />


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
<img width="908" height="212" alt="image" src="https://github.com/user-attachments/assets/a6d9942d-d4fa-4c4f-9c13-abc99171e0bf" />


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
<img width="778" height="341" alt="image" src="https://github.com/user-attachments/assets/2df3c686-46d3-44be-9b1d-ecdc096af537" />


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
<img width="689" height="162" alt="image" src="https://github.com/user-attachments/assets/7c6877cc-7601-4dcc-be47-2410152ef2f7" />


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
<img width="619" height="160" alt="image" src="https://github.com/user-attachments/assets/4facf7fb-cb2c-4f99-9bdd-6cd6fc3a00be" />


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
<img width="1439" height="222" alt="image" src="https://github.com/user-attachments/assets/24c800ef-9cd6-43b5-a669-5e22375cdd7f" />


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
ตอบ : EXPLAIN ฐานข้อมูลวางแผนจะทำอะไร? 
      ANALYZE สิ่งที่ทำจริงเป็นอย่างไร? 
      BUFFERS การทำงานนั้นต้องใช้การอ่านข้อมูลจากดิสก์มากแค่ไหน? 
การดูข้อมูลเหล่านี้จะช่วยให้เราเห็นจุดอ่อนของคำสั่ง SQL และสามารถปรับปรุงให้ทำงานได้เร็วขึ้น

2. รูปผลการรัน

3. อธิบายผลลัพธ์ที่ได้
ตอบ :
PostgreSQL ดึงข้อมูลมาแค่ 1000 แถวแรก
actual time=181.720..194.056 ms → ใช้เวลาประมาณ 12.3 ms ในขั้นตอนนี้
rows=1000 → ได้ผลลัพธ์ 1000 แถว
Buffers: shared hit=5133 → มีการดึงข้อมูลจาก shared buffer 5,133 ครั้ง

Gather Merge
PostgreSQL ใช้งาน parallel query โดยมี worker 2 ตัว (เพื่อเพิ่มความเร็ว)
Gather Merge คือการรวมผลลัพธ์จาก worker หลายตัวเข้าด้วยกัน โดย merge ให้เรียงลำดับตาม ORDER BY data
ใช้เวลาประมาณ 12 ms (181.703 → 193.923)

Sort
มีการ sort ข้อมูลจากแต่ละ worker (ทำ 3 รอบ → loops=3 เพราะมี 2 worker + main)
ใช้เทคนิค top-N heapsort ซึ่งจะหยุดทันทีเมื่อได้ข้อมูลครบตามที่ต้องการ (1000 แถว)
ใช้หน่วยความจำในการ sort ประมาณ 232 KB
ใช้เวลาเฉลี่ย 0.04 ms ต่อ loop ในการ sort

Parallel Seq Scan on large_table (การอ่านข้อมูล)
PostgreSQL อ่านข้อมูลจากตาราง large_table โดย ไม่ใช้ index (sequential scan)
ใช้การอ่านแบบขนาน (parallel) → loop 3 รอบ แปลว่าแบ่งข้อมูลให้ thread ทั้งหมด 3 ตัว (main + 2 workers)
rows=166667 ต่อ loop → รวมกันเป็นประมาณ 500,000 แถว (ครบทั้งหมด)
ใช้เวลาเร็วมากในการอ่านข้อมูล (0.015 → 32.811 ms)
```
<img width="1451" height="534" alt="image" src="https://github.com/user-attachments/assets/7b83b516-09f7-4369-9ddd-997a32256982" />

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
ตอบ :
- เวลาที่ใช้: การวางแผน (Planning) ใช้เวลา 1.748 ms ส่วนการทำงานจริง (Execution) ใช้เวลาแค่ 0.460 ms ซึ่งถือว่าเร็วมากครับ
- การอ่านข้อมูล: ฐานข้อมูลใช้การอ่านข้อมูลจากหน่วยความจำ (cache) ทั้งหมด 6 บล็อก (Buffers: shared hit=6) ซึ่งน้อยมาก แสดงให้เห็นว่าการทำงานมีประสิทธิภาพสูง
- ผลลัพธ์ที่ได้: คำสั่งนี้คืนค่ากลับมา 14 แถว (14 rows)

3. การสแกนเป็นแบบใด เกิดจากเหตุผลใด
ตอบ : การสแกนเป็นแบบ Index Only Scan
เหตุผล: การสแกนแบบนี้เกิดขึ้นได้เพราะฐานข้อมูลสามารถหาข้อมูลที่ต้องการได้ จาก Index ทั้งหมด โดยไม่ต้องกลับไปอ่านข้อมูลในตารางหลักเลย ซึ่งทำให้การทำงานเร็วมาก
```
<img width="1444" height="521" alt="image" src="https://github.com/user-attachments/assets/28a4d0af-836a-48dd-a7b6-bfa26ede1d38" />

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
ตอบ : ผลลัพธ์ที่ได้จะรายละเอียดจำนวนแถวที่ถูกลบหรือ reclaimed อัตโนมัติ และค่าอื่น ๆ ที่เกี่ยวข้องกับการทำ vacuum
จะแสดงข้อมูลว่าสถิติอะไรในแต่ละคอลัมน์ถูกอัปเดตและประเมินค่าต่าง ๆ 
```
<img width="1410" height="761" alt="image" src="https://github.com/user-attachments/assets/bdbef09b-2f54-45f1-a9de-446b531ec808" />

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
<img width="924" height="378" alt="image" src="https://github.com/user-attachments/assets/11332118-6c54-4728-acec-4d4e34c03994" />


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
ตอบ : heap_blks_read = 0 → ไม่มีการอ่านข้อมูลจาก disk เลย
      heap_blks_hit = 620839 → ข้อมูลทั้งหมดถูกอ่านจาก memory (buffer cache)
      hit_ratio_percent = 100.00 → หมายถึง การเข้าถึงข้อมูลทั้งหมดเป็น cache hit (เร็วที่สุด เพราะไม่ต้องอ่านจาก disk)
```
<img width="955" height="425" alt="image" src="https://github.com/user-attachments/assets/9d88b6e5-1f37-4328-9d55-336bafed9bfc" />

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
ตอบ : blks_read → อ่านจาก disk = 3481 blocks 
      blks_hit→ อ่านจาก memory = 4,052,764 blocks เยอะมากเมื่อเทียบกับ disk
      hit_ratio_percent = 99.91% → แทบทั้งหมดมาจาก memory
```
<img width="883" height="294" alt="image" src="https://github.com/user-attachments/assets/6d64104b-6660-42a5-8cce-9716a4370193" />


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
ตอบ : มี เงื่อนไข WHERE heap_blks_read > 0 → หมายถึงเอาเฉพาะตารางที่ มีการอ่านจาก disk จริง ๆ เท่านั้น
ซึ่งได้ผลลัพธ์ได้ 0 rows → แสดงว่า ไม่มีตารางใดที่มีการอ่านจาก disk เลยในรอบสถิติที่เก็บ
```
<img width="1079" height="402" alt="image" src="https://github.com/user-attachments/assets/f4c5ca92-4a2f-4515-b076-3ede291a93f2" />

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
autovacuum                      | on        | เปิดใช้งาน autovacuum daemon (ควรเปิดเสมอ)
autovacuum_analyze_scale_factor | 0.1       | ANALYZE เมื่อมี row เปลี่ยน 10% ของตาราง
autovacuum_analyze_threshold    | 50        | ANALYZE เมื่อมี row เปลี่ยนเกิน 50 แถวขึ้นไป
autovacuum_freeze_max_age       | 200000000 | ป้องกัน transaction ID wraparound (บังคับ vacuum)
autovacuum_max_workers          | 3         | จำนวน worker autovacuum ที่รันพร้อมกันได้สูงสุด
autovacuum_multixact_freeze_max_age | 400000000 | ป้องกัน multixact wraparound
autovacuum_naptime              | 60s       | ช่วงเวลาพักก่อน autovacuum รอบถัดไป
autovacuum_vacuum_cost_delay    | 2ms       | หน่วงเวลาแต่ละรอบ vacuum เพื่อลดผลกระทบ workload
autovacuum_vacuum_cost_limit    | -1        | ค่า cost limit (ใช้ default ของระบบ)
autovacuum_vacuum_insert_scale_factor | 0.2 | VACUUM insert เมื่อ row insert เปลี่ยน 20%
autovacuum_vacuum_insert_threshold   | 1000 | VACUUM insert เมื่อมี insert เกิน 1000 แถว
autovacuum_vacuum_scale_factor  | 0.2       | VACUUM เมื่อมี row update/delete 20% ของตาราง
autovacuum_vacuum_threshold     | 50        | VACUUM เมื่อมี row update/delete เกิน 50 แถว
autovacuum_work_mem             | -1        | memory สูงสุดที่ใช้ต่อ worker (ค่า -1 = ใช้ default)
log_autovacuum_min_duration     | 600000ms  | log ถ้า autovacuum ใช้เวลานานกว่า 10 นาที
```
<img width="1419" height="554" alt="image" src="https://github.com/user-attachments/assets/550dbbb1-8f3f-419d-bd79-0bbcf845f2db" />


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
<img width="965" height="510" alt="image" src="https://github.com/user-attachments/assets/1ee26cd9-e81e-44e1-8267-65bd2f4379d7" />


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

<img width="694" height="286" alt="image" src="https://github.com/user-attachments/assets/23d1daac-5de0-4230-9615-d38ac45b6481" />
```
1. รูปผลการทดลอง
2. อธิบายผลลัพธ์ที่ได้
ตอบ : 
test_name → ชื่อการทดสอบ 
config_set → ค่า config ที่ใช้ตอนรันทดสอบ 
execution_time_ms → เวลาในการ execute test นั้น ๆ 
AVG(...) OVER (PARTITION BY test_name) → ใช้ window function เพื่อหา average time ของแต่ละ test_name โดยไม่ต้อง GROUP BY
ORDER BY test_timestamp DESC → เรียงผลลัพธ์ตามเวลาล่าสุด
```

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
<img width="879" height="222" alt="image" src="https://github.com/user-attachments/assets/5f9a4fd0-bedd-43eb-be85-542d2d50a421" />


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
<img width="885" height="224" alt="image" src="https://github.com/user-attachments/assets/2dfccda3-85db-4116-82ab-64025c5a5c0e" />
<img width="943" height="221" alt="image" src="https://github.com/user-attachments/assets/d6d94f3b-76bf-42a8-aa92-c9a12759c6da" />


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

<img width="851" height="288" alt="image" src="https://github.com/user-attachments/assets/5717b81a-3e53-46ec-b42e-1f6a4f106807" />


-- ทดสอบปานกลาง  
SELECT * FROM simulate_oltp_workload(100);
### ผลการทดลอง

1. รูปผลการทดลอง
<img width="890" height="318" alt="image" src="https://github.com/user-attachments/assets/92dd649c-5bec-4f71-8c2f-c4081aa19670" />


2. อธิบายผลการทดลอง การ SELECT , INSERT, UPDATE, DELETE เป็นอย่างไร
   🔹 SELECT (JOIN + WHERE)
avg_time = 0.050 ms → เร็วมาก แทบจะทันที
ค่า min/max อยู่ระหว่าง 0.036 – 0.743 ms → แทบไม่มี outlier
แสดงว่า query read มีประสิทธิภาพดีมาก (น่าจะ cache hit สูง, index ใช้งานได้ดี)
  🔹 INSERT
avg_time = 0.025 ms → เร็วที่สุดใน 4 operations
INSERT โดยทั่วไปใน PostgreSQL ค่อนข้างเบา เพราะแค่เพิ่ม row เข้า heap และ WAL
ค่า min/max อยู่ระหว่าง 0.012 – 0.355 ms → ยังถือว่าคงที่
  🔹 UPDATE
avg_time = 130.385 ms → ช้ากว่า SELECT/INSERT อย่างมาก
UPDATE ใน PostgreSQL ไม่ได้แก้ไข row ตรง ๆ แต่ทำ MVCC → สร้าง row ใหม่ + mark row เก่าเป็น dead tuple
ค่า min/max กว้าง (109.6 – 202.2 ms) → อาจมี table bloat หรือ index update หลายตัว
  🔹 DELETE (soft)
avg_time = 166.604 ms → ช้าที่สุด
ค่า min/max อยู่ระหว่าง 120.7 – 251.5 ms → แกว่งมากกว่าตัวอื่น
DELETE ใน PostgreSQL ไม่ลบ row จริง แต่ mark เป็น dead tuple (และเพิ่มค่า deleted_at สำหรับ soft delete) → ต้องการ vacuum/autovacuum ตามมา
ช้ากว่า UPDATE เพราะอาจกระทบ index + trigger constraint


-- ทดสอบหนักขึ้น เครื่องใครไม่ไหวผ่านก่อน หรือเปลี่ยนค่า 500 เป็น 200 :)
SELECT * FROM simulate_oltp_workload(500);
### ผลการทดลอง
<img width="867" height="309" alt="image" src="https://github.com/user-attachments/assets/a4ee0603-ac2b-4bde-9288-07b3c4c534d0" />


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
<img width="966" height="247" alt="image" src="https://github.com/user-attachments/assets/3f932039-801d-4ced-b6e6-53c9f0dadae8" />

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
<img width="1162" height="441" alt="image" src="https://github.com/user-attachments/assets/f9491435-d8e2-43c4-bcb4-57af3ac627f9" />


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
<img width="1028" height="409" alt="image" src="https://github.com/user-attachments/assets/7fb90d89-c943-4000-b71b-85b0247ccf2e" />


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
```
1. หน่วยความจำใดบ้างที่เป็น shared memory และมีหลักในการตั้งค่าอย่างไร
  ตอบ :
    - shared_buffers หน่วยความจำส่วนนี้ถูกใช้โดยทุกกระบวนการ (process) ของ PostgreSQL เพื่อจัดเก็บข้อมูลที่ถูกเรียกใช้บ่อย เช่น หน้าข้อมูล (data pages) และล็อก (log)
    - มีหลักในการตั้งค่าโดยแนะนำให้ตั้งค่าเป็น 25% ของ RAM ทั้งหมด ของเซิร์ฟเวอร์ ถ้า RAM ของคุณมีมากกว่า 1GB  การตั้งค่าที่สูงเกินไปอาจทำให้ระบบปฏิบัติการไม่มี RAM เพียงพอสำหรับแคชไฟล์ (file cache)

2. Work memory และ maintenance work memory คืออะไร มีหลักการในการกำหนดค่าอย่างไร
  ตอบ :
    - Work Memory (work_mem) คือหน่วยความจำที่ใช้สำหรับงานชั่วคราวแต่ละรายการ (per-operation) เช่น การจัดเรียง (sort) และการรวมกลุ่ม (hash joins) ถ้างานมีขนาดใหญ่กว่าค่าที่กำหนด ข้อมูลจะถูกเขียนลงดิสก์ชั่วคราว ซึ่งทำให้ช้าลง 
    - Maintenance Work Memory (maintenance_work_mem) คือหน่วยความจำที่ใช้สำหรับงานบำรุงรักษาฐานข้อมูล เช่น VACUUM, CREATE INDEX และ ALTER TABLE. งานเหล่านี้ไม่ได้เกิดขึ้นบ่อยและใช้หน่วยความจำค่อนข้างมาก

 หลักการตั้งค่า:
- work_mem: ควรตั้งค่าให้ต่ำเข้าไว้และค่อยๆ ปรับขึ้นตามความจำเป็น โดยค่าเริ่มต้นอยู่ที่ 4MB
- maintenance_work_mem: ควรตั้งค่าให้สูงพอสมควร เพื่อให้งานบำรุงรักษาทำได้เร็วขึ้น แนะนำให้ตั้งค่าเป็น 10-15% ของ RAM ทั้งหมด หรือประมาณ 1GB สำหรับเครื่องเซิร์ฟเวอร์ที่ใหญ่ขึ้น

3. หากมี RAM 16GB และต้องการกำหนด connection = 200 ควรกำหนดค่า work memory และ maintenance work memory อย่างไร
  ตอบ :
    - work_mem: ควรเริ่มต้นที่ค่ามาตรฐาน เช่น 4-8MB. ค่านี้จะถูกใช้ต่อหนึ่งการจัดเรียงหรือ hash join. ถ้าตั้งค่าสูงไปอาจทำให้ RAM หมดได้ เนื่องจากจำนวนการเชื่อมต่อที่เยอะ (สูงสุด 200 connections x work_mem). หากผู้ใช้งาน 200 คนรันคำสั่งที่ต้องใช้ work_mem พร้อมกัน โดยตั้งค่าไว้ที่ 8MB จะใช้ RAM ส่วนนี้ไปสูงสุดถึง 1.6GB
    - maintenance_work_mem: ควรตั้งค่าให้สูงพอสำหรับงานบำรุงรักษา แนะนำประมาณ 1GB (หรือ 1/16 ของ RAM ทั้งหมด). ค่านี้ไม่ได้ใช้พร้อมกันหลายงานเหมือน work_mem จึงสามารถตั้งค่าได้สูงกว่า

4. ไฟล์ postgresql.conf และ postgresql.auto.conf  มีความสัมพันธ์กันอย่างไร
  ตอบ :
    - postgresql.conf คือไฟล์การตั้งค่าหลักของ PostgreSQL ที่มีพารามิเตอร์ทั้งหมดและค่าเริ่มต้น
    - postgresql.auto.conf คือไฟล์ที่เก็บค่าการตั้งค่าที่ถูกเปลี่ยนผ่านคำสั่ง ALTER SYSTEM เท่านั้น
ดังนั้น เมื่อ PostgreSQL เริ่มทำงาน มันจะอ่านค่าจาก postgresql.conf ก่อน แล้วจึงอ่านค่าจาก postgresql.auto.conf ซึ่งถ้ามีการตั้งค่าซ้ำกันในสองไฟล์นี้ ค่าใน postgresql.auto.conf จะถูกใช้เป็นหลัก

5. Buffer hit ratio คืออะไร
  ตอบ : Buffer Hit Ratio คืออัตราส่วนที่บอกว่าข้อมูลที่ถูกเรียกใช้ สามารถหาได้จากแคชในหน่วยความจำ (shared_buffers) โดยตรง
 

6. แสดงผลการคำนวณ การกำหนดค่าหน่วยความจำต่าง ๆ โดยอ้างอิงเครื่องของตนเอง
  ตอบ :
    - RAM ทั้งหมด: 16GB
    - shared_buffers: 16GB x 0.25 = 4GB
    - maintenance_work_mem: 256 MB
    - work_mem: 20 MB
    - max_connections: 200

7. การสแกนของฐานข้อมูล PostgreSQL มีกี่แบบอะไรบ้าง เปรียบเทียบการสแกนแต่ละแบบ
  ตอบ : หลักๆ มี 3 แบบ
    1. Sequential Scan (การสแกนแบบลำดับ): อ่านทุกแถวในตารางตั้งแต่ต้นจนจบ
- เหมาะสำหรับ: ตารางเล็กๆ หรือเมื่อต้องอ่านข้อมูลส่วนใหญ่ของตาราง
- ข้อเสีย: ช้ามากสำหรับตารางขนาดใหญ่

    2. Index Scan (การสแกนแบบใช้ดัชนี): ใช้ดัชนี (index) เพื่อหาตำแหน่งของข้อมูลที่ต้องการโดยตรง
- เหมาะสำหรับ: การค้นหาข้อมูลจำนวนน้อยๆ จากตารางขนาดใหญ่
- ข้อเสีย: ต้องเสียเวลาในการสร้างและบำรุงรักษาดัชนี

    3. Index Only Scan (การสแกนแบบใช้ดัชนีเท่านั้น): คล้ายกับ Index Scan แต่จะอ่านข้อมูลทั้งหมดที่ต้องการจากดัชนีโดยไม่ต้องไปอ่านข้อมูลจากตารางหลักเลย ซึ่งทำให้เร็วที่สุด
- เหมาะสำหรับ: เมื่อข้อมูลที่ต้องการอยู่ในดัชนีอยู่แล้ว
- ข้อเสีย: เกิดขึ้นได้เฉพาะในกรณีที่ข้อมูลที่ต้องการอยู่ในดัชนีทั้งหมดเท่านั้น
```
