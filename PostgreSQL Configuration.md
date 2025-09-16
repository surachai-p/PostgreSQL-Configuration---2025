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
2. option -h ในคำสั่งมีผลอย่างไร
3. docker exec postgres-config nproc  แสดงค่าผลลัพธ์อย่างไร
```
<img width="1336" height="454" alt="image" src="https://github.com/user-attachments/assets/aeda8c21-4e2e-47bf-9a4e-1d1e78444aa1" />
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
<img width="1850" height="846" alt="image" src="https://github.com/user-attachments/assets/8d894669-bc4d-48dc-96b5-739604b6ee54" />

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
<img width="2682" height="476" alt="image" src="https://github.com/user-attachments/assets/297fc8c6-3dcf-4ba1-b8da-3688fc9bd725" />
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
2. ค่า  shared_buffers มีการกำหนดค่าไว้เท่าไหร่ (ใช้ setting X unit)
3. ค่า  pending_restart ในผลการทดลองมีค่าเป็นอย่างไร และมีความหมายอย่างไร
<img width="1256" height="228" alt="image" src="https://github.com/user-attachments/assets/abe2a9da-f7d0-4cf7-b61a-3c2f37308efa" />
ค่า shared_buffers = 128MB
ค่า pending_restart = false → ไม่ต้อง restart, ค่าใช้งานแล้ว
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
```
รูปผลการเปลี่ยนแปลงค่า pending_restart
รูปหลังจาก restart postgres

```
<img width="1456" height="86" alt="image" src="https://github.com/user-attachments/assets/ad2f20e4-a71e-4e2a-b5d8-711234e9a8ae" />
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
<img width="718" height="418" alt="image" src="https://github.com/user-attachments/assets/45c6972a-75ac-46f0-a84d-c87ba4faf540" />
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
<img width="994" height="578" alt="image" src="https://github.com/user-attachments/assets/3eb21dc8-d49b-4d07-b485-2f3a3ddf665b" />
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
<img width="1314" height="966" alt="image" src="https://github.com/user-attachments/assets/0ab070f8-9e44-4eee-9da3-89d16f34887f" />
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
<img width="1118" height="558" alt="image" src="https://github.com/user-attachments/assets/243533f0-2ca4-47e1-930e-95b4abed3e11" />
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
<img width="2396" height="784" alt="image" src="https://github.com/user-attachments/assets/023843bb-c7a3-4ace-b370-af5306b74e04" />
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
2. รูปผลการรัน
3. อธิบายผลลัพธ์ที่ได้
```
1.EXPLAIN (ANALYZE, BUFFERS) คือ คำสั่งใน PostgreSQL ที่ใช้ดูแผนการประมวลผลของ SQL query พร้อมรันจริงและแสดงรายละเอียดการใช้เวลาและหน่วยความจำ (buffer) ของแต่ละขั้นตอน
EXPLAIN → แสดงแผนการประมวลผล query
ANALYZE → รัน query จริงและวัดเวลา
BUFFERS → แสดงการใช้ memory และ disk ของ query
<img width="2976" height="1564" alt="image" src="https://github.com/user-attachments/assets/c6bbc024-8713-40ce-ad9c-9075169218d9" />
3.การใช้ Index มีประสิทธิภาพ: การทดสอบนี้แสดงให้เห็นว่าการ SELECT ข้อมูลจำนวน 10 แถว จากตารางที่มี 10 ล้านแถว สามารถทำได้อย่างรวดเร็ว โดยใช้เวลาเพียง 187.248 ms

การทำงานแบบ Parallel ช่วยได้: PostgreSQL เลือกใช้การสแกนตารางแบบขนาน (parallel scan) ซึ่งหมายความว่าได้แบ่งการทำงานไปให้ worker หลายตัวช่วยกันประมวลผล ทำให้การดึงข้อมูลจากตารางขนาดใหญ่ทำได้เร็วกว่าการสแกนแบบปกติ (single process)

การเรียงลำดับข้อมูลทำได้อย่างมีประสิทธิภาพ: PostgreSQL เลือกใช้ top-N heapsort ซึ่งเหมาะสมกับสถานการณ์ที่ต้องการข้อมูลเพียง 10 แถวแรกจากทั้งหมด ทำให้ใช้หน่วยความจำน้อยและทำงานได้รวดเร็ว

ข้อมูล Buffers: ข้อมูล Buffers แสดงให้เห็นว่าฐานข้อมูลใช้พื้นที่หน่วยความจำร่วม (shared buffers) ในการอ่านข้อมูล ซึ่งช่วยลดการอ่านจากดิสก์ (disk I/O) ทำให้การทำงานโดยรวมเร็วขึ้น
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
3. การสแกนเป็นแบบใด เกิดจากเหตุผลใด
```
<img width="2532" height="706" alt="image" src="https://github.com/user-attachments/assets/d91fa1d8-8e4e-4ec0-9abd-faade4699e66" />
2.จากผลลัพธ์ที่ได้ สามารถสรุปได้ดังนี้:

ประสิทธิภาพสูงด้วย Index: ฐานข้อมูลใช้ Index Only Scan ซึ่งเป็นวิธีที่รวดเร็วที่สุด เนื่องจากสามารถดึงข้อมูลที่ต้องการ (ค่า number และจำนวน) ได้จาก index โดยตรงโดยไม่ต้องสแกนข้อมูลในตารางจริง

การกรองข้อมูลที่ซ้ำกัน: ขั้นตอน GroupAggregate ทำงานได้อย่างมีประสิทธิภาพ โดยสามารถกรองกลุ่มข้อมูลที่มีจำนวนแถวมากกว่า 1 ออกไปตามเงื่อนไข HAVING

ความเร็วโดยรวม: คำสั่งนี้ใช้เวลาในการประมวลผลจริงเพียง 0.801 มิลลิวินาที ซึ่งถือว่าเร็วมากสำหรับคำสั่งที่ต้องมีการจัดกลุ่มและกรองข้อมูลจากตารางขนาดใหญ่

การที่ Heap Fetches มีค่าเป็น 0 และการที่ Buffers: shared hit มีค่าต่ำ แสดงให้เห็นว่าคำสั่งนี้ทำงานได้อย่างมีประสิทธิภาพและใช้ทรัพยากรน้อยมาก เพราะสามารถใช้ประโยชน์จาก index ได้อย่างเต็มที่

3.การสแกนแบบ Index Only Scan เกิดขึ้นเนื่องจาก:

คำสั่ง Query: คำสั่ง SELECT ที่ใช้ประกอบด้วย COUNT(*) และ GROUP BY number ซึ่งต้องการข้อมูลแค่ 2 อย่างคือ ค่าจากคอลัมน์ number และ จำนวน (count) ของข้อมูลในแต่ละกลุ่ม

มี Index ที่เหมาะสม: ฐานข้อมูลมีดัชนี (index) ชื่อ idx_large_table_number ซึ่งสร้างขึ้นจากคอลัมน์ number

ข้อมูลครบใน Index: ในกรณีนี้ ข้อมูลที่จำเป็นทั้งหมดสำหรับคำสั่ง SELECT (คือค่าของ number และข้อมูลที่จำเป็นสำหรับการนับ) สามารถหาได้จากตัว index โดยตรง ฐานข้อมูลจึงไม่จำเป็นต้องไปอ่านข้อมูลจาก "ตารางจริง" (heap) เลย ทำให้การทำงานรวดเร็วและมีประสิทธิภาพสูงมาก ดังที่เห็นจากค่า Heap Fetches: 0 ในผลลัพธ์
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
```
<img width="2118" height="1148" alt="image" src="https://github.com/user-attachments/assets/13b04735-b4ad-48e7-bfb3-84823ed6408f" />
 2. 1. การสร้างดัชนี (Create Index)

CREATE INDEX CONCURRENTLY idx_large_table_data ON large_table USING btree(data);: เป็นคำสั่งสร้างดัชนีชื่อ idx_large_table_data บนคอลัมน์ data ของตาราง large_table โดยใช้โครงสร้างแบบ B-tree การสร้างดัชนีช่วยให้การค้นหาข้อมูลในคอลัมน์นั้นรวดเร็วขึ้นมาก

CONCURRENTLY: ตัวเลือกนี้ทำให้การสร้างดัชนีไม่ไปขัดขวางการทำงานอื่น ๆ บนตาราง ผู้ใช้งานยังคงสามารถอ่านหรือเขียนข้อมูลได้ตามปกติ

Time: 1932.697 ms: ใช้เวลาประมาณ 1.93 วินาทีในการสร้าง

2. การลบข้อมูล (Delete)

DELETE FROM large_table WHERE id % 10 = 0;: คำสั่งนี้เป็นการลบข้อมูล 50,000 แถว (rows) จากตาราง large_table โดยลบเฉพาะแถวที่ค่าในคอลัมน์ id หารด้วย 10 ลงตัว

DELETE 50000: ผลลัพธ์ยืนยันว่ามีข้อมูลถูกลบไป 50,000 แถว

3. การทำ Vacuum และ Analyze

หลังจากลบข้อมูลแล้ว พื้นที่ในฐานข้อมูลที่เคยถูกใช้โดยแถวที่ถูกลบจะยังคงอยู่และถูกทำเครื่องหมายว่าเป็น "dead tuples" หรือแถวที่ตายแล้ว การทำ VACUUM จะช่วยกู้คืนพื้นที่เหล่านี้ให้กลับมาใช้งานได้ใหม่

VACUUM (ANALYZE, VERBOSE) large_table;: ดำเนินการ Vacuum และวิเคราะห์ข้อมูลบนตาราง large_table

tuples 500000 removed, 450000 remain: ในช่วงแรกของการทำ Vacuum พบว่ามีแถวที่ถูกทำเครื่องหมายว่าถูกลบไปแล้ว 500,000 แถว และเหลือแถวที่ยังใช้งานอยู่ 450,000 แถว

containing 450000 live rows and 0 dead rows: ผลลัพธ์สุดท้ายแสดงให้เห็นว่าการทำ Vacuum ประสบความสำเร็จในการกู้คืนพื้นที่ที่ถูกลบไปได้อย่างสมบูรณ์ โดยตอนนี้มีแต่แถวที่ใช้งานอยู่ 450,000 แถว และไม่มีแถวที่ตายแล้ว (0 dead rows)

analyzing "public.large_table": ส่วนนี้เป็นการเก็บสถิติของตารางเพื่อช่วยให้ตัววางแผนการทำงาน (Query Planner) ของ PostgreSQL สามารถเลือกวิธีการประมวลผลคำสั่ง SELECT ในอนาคตได้อย่างมีประสิทธิภาพสูงสุด

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
<img width="1834" height="1278" alt="image" src="https://github.com/user-attachments/assets/ae84736d-39e8-455f-b1a3-991282a478d3" />
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
```
<img width="1560" height="510" alt="image" src="https://github.com/user-attachments/assets/6239fe9e-acd9-4d3d-beea-cef3b502d80e" />
2.คำสั่ง SQL นี้ถูกออกแบบมาเพื่อตรวจสอบประสิทธิภาพการเข้าถึงตารางในฐานข้อมูล PostgreSQL โดยเฉพาะอย่างยิ่งการดูว่าฐานข้อมูลสามารถอ่านข้อมูลจากแคชในหน่วยความจำได้ดีแค่ไหน (Heap Blks Hit) เทียบกับการต้องอ่านข้อมูลจากดิสก์ (Heap Blks Read) ซึ่งจะช้ากว่ามาก

รายละเอียดแต่ละคอลัมน์

schemaname: แสดงชื่อสกีมาของตาราง ในที่นี้คือ public

relname: แสดงชื่อตาราง ในที่นี้คือ large_table

heap_blks_read: จำนวนบล็อกข้อมูลที่ฐานข้อมูลต้องอ่านจาก ดิสก์ ในการเข้าถึงตารางนี้ ค่าที่แสดงคือ 0 ซึ่งหมายความว่าไม่มีการอ่านข้อมูลจากดิสก์เลย

heap_blks_hit: จำนวนบล็อกข้อมูลที่ฐานข้อมูลสามารถอ่านได้จาก แคช (หน่วยความจำ) ค่าที่แสดงคือ 620839 ซึ่งหมายความว่าการเข้าถึงทั้งหมดสามารถทำได้จากแคช

hit_ratio_percent: อัตราส่วนการเข้าถึงแคช คำนวณจาก (heap_blks_hit / (heap_blks_read + heap_blks_hit)) * 100 ค่าที่ได้คือ 100.00 
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
```
<img width="1540" height="356" alt="image" src="https://github.com/user-attachments/assets/3516ad44-bc22-4cac-b838-b70e3a1cc165" />
2.ผลลัพธ์ที่คุณเห็นเป็นการแสดงประสิทธิภาพการทำงานของฐานข้อมูลโดยรวม ซึ่งตรวจสอบจากแคช (cache) ของฐานข้อมูล PostgreSQL ฐานข้อมูลจะพยายามดึงข้อมูลจากหน่วยความจำ (RAM) ก่อน เพราะเร็วกว่าการอ่านข้อมูลจากดิสก์ (HDD/SSD) มาก หากข้อมูลอยู่ในหน่วยความจำ จะเรียกว่า "Hit" แต่ถ้าต้องไปดึงจากดิสก์ จะเรียกว่า "Read"

datname: ชื่อของฐานข้อมูล ซึ่งในที่นี้คือ performance_test

blks_read: จำนวนบล็อกข้อมูลทั้งหมดที่ถูกอ่านจาก ดิสก์ มีค่าเท่ากับ 3481 บล็อก

blks_hit: จำนวนบล็อกข้อมูลทั้งหมดที่ถูกดึงจาก หน่วยความจำ (แคช) มีค่าเท่ากับ 4051106 บล็อก

hit_ratio_percent: อัตราส่วนการเข้าถึงข้อมูลจากแคช ซึ่งคำนวณจาก (blks_hit / (blks_read + blks_hit)) * 100 ผลลัพธ์ที่ได้คือ 99.91%
####6.4 ดู Table ที่มี Disk I/O มาก
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
```
<img width="1942" height="476" alt="image" src="https://github.com/user-attachments/assets/979d8ee4-bb7a-460c-ba3a-947221b2d563" />
2.
0 rows: ผลลัพธ์นี้บอกว่าไม่มีตารางไหนที่ตรงกับเงื่อนไข WHERE heap_blks_read > 0

Time: 13.854 ms: เป็นเวลาที่ใช้ในการประมวลผลคำสั่งนี้ ซึ่งถือว่ารวดเร็วมาก
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
```
<img width="2508" height="680" alt="image" src="https://github.com/user-attachments/assets/804a3326-696a-4cb8-86e8-94df289dbbbd" />
2.ค่าการตั้งค่าที่สำคัญและคำอธิบาย

autovacuum:

setting: on

คำอธิบาย: การตั้งค่านี้จะระบุว่าระบบ autovacuum ทำงานอยู่หรือไม่ ถ้าตั้งเป็น on แปลว่าระบบทำงานอยู่ ถ้าเป็น off แปลว่าต้องรัน VACUUM ด้วยตัวเองเป็นประจำ

<br>

autovacuum_analyze_scale_factor:

setting: 0.1

คำอธิบาย: กำหนดสัดส่วนของข้อมูลที่ถูกเปลี่ยนแปลง (เพิ่ม/ลบ/อัปเดต) เมื่อเทียบกับจำนวนแถวทั้งหมดในตาราง (rel-tuples) โดยจะเริ่มรัน ANALYZE เมื่อถึงสัดส่วนนี้ (ในที่นี้คือ 10%)
<br>

autovacuum_analyze_threshold:

setting: 50

คำอธิบาย: กำหนดจำนวนขั้นต่ำของแถวที่ถูกเปลี่ยนแปลงที่จะกระตุ้นการรัน ANALYZE โดยจะทำงานเมื่อจำนวนแถวที่เปลี่ยนแปลงถึงค่านี้ และตรงกับเงื่อนไข autovacuum_analyze_scale_factor
<br>

autovacuum_vacuum_scale_factor:

setting: 0.2

คำอธิบาย: กำหนดสัดส่วนของข้อมูลที่ถูกเปลี่ยนแปลง (เพิ่ม/ลบ) ที่จะเริ่มรัน VACUUM (ในที่นี้คือ 20%)
<br>

autovacuum_vacuum_threshold:

setting: 50

คำอธิบาย: กำหนดจำนวนขั้นต่ำของแถวที่ถูกเปลี่ยนแปลงที่จะกระตุ้นการรัน VACUUM โดยจะทำงานเมื่อจำนวนแถวที่เปลี่ยนแปลงถึงค่านี้ และตรงกับเงื่อนไข autovacuum_vacuum_scale_factor
<br>

autovacuum_vacuum_cost_delay:

setting: 2

unit: ms (มิลลิวินาที)

คำอธิบาย: เป็นการหน่วงเวลาการทำงาน (sleep) ของ autovacuum เพื่อไม่ให้ใช้ทรัพยากรระบบมากเกินไป โดยจะหยุดพักเป็นเวลาที่กำหนดหลังจากประมวลผลไปจำนวนหนึ่ง
<br>

autovacuum_vacuum_cost_limit:

setting: -1

คำอธิบาย: กำหนดขีดจำกัดสูงสุดของปริมาณงานที่ autovacuum สามารถทำได้ก่อนที่จะหยุดพัก การตั้งค่าเป็น -1 หมายถึงจะใช้ค่าเริ่มต้นของระบบ vacuum_cost_limit
<br>

autovacuum_max_workers:

setting: 3

คำอธิบาย: จำนวนกระบวนการ autovacuum ที่สามารถทำงานพร้อมกันได้สูงสุด การเพิ่มค่านี้จะช่วยให้การทำความสะอาดข้อมูลเกิดขึ้นได้เร็วขึ้นเมื่อมีตารางที่ต้องจัดการจำนวนมาก

ค่าอื่นๆ ที่ควรทราบ

autovacuum_freeze_max_age:

setting: 200000000

คำอธิบาย: อายุสูงสุดของ Transaction ID ก่อนที่ autovacuum จะถูกบังคับให้รันเพื่อป้องกันปัญหาการวนซ้ำของ Transaction ID (Transaction ID wraparound) ซึ่งอาจทำให้ข้อมูลเสียหายได้

<br>

log_autovacuum_min_duration:

setting: 600000

unit: ms (มิลลิวินาที)

คำอธิบาย: กำหนดระยะเวลาการทำงานขั้นต่ำที่ autovacuum จะต้องใช้ก่อนที่จะบันทึกข้อความลงใน log การตั้งค่านี้มีประโยชน์สำหรับการติดตามและปรับปรุงประสิทธิภาพของ autovacuum 
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
<img width="1414" height="700" alt="image" src="https://github.com/user-attachments/assets/fc890574-796a-4353-9d08-46c8d7406c69" />
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
```
<img width="1700" height="908" alt="image" src="https://github.com/user-attachments/assets/615021ca-f8ab-4e8a-9fd3-d2b4fac16670" />
2.สรุปผลลัพธ์

การเรียกใช้ฟังก์ชัน run_performance_test ล้มเหลวเนื่องจากใช้คำสั่ง SELECT แทน PERFORM ทำให้เกิดข้อผิดพลาด "query has no destination for result data"

การเรียกดูข้อมูลจากตาราง performance_results ได้ผลลัพธ์เป็น (0 rows) แสดงว่าการทดสอบประสิทธิภาพที่รันไป ไม่ได้ถูกบันทึกลงในตารางนี้เลย

หากคุณต้องการแก้ไขปัญหานี้ คุณอาจต้องตรวจสอบโค้ดของฟังก์ชัน run_performance_test เพื่อให้แน่ใจว่ามีการบันทึกผลลัพธ์ลงในตาราง performance_results อย่างถูกต้อง และอาจลองเปลี่ยนการเรียกใช้จาก SELECT เป็น PERFORM### Step 9: การ Monitoring และ Alerting

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
<img width="1712" height="962" alt="image" src="https://github.com/user-attachments/assets/8c335061-c05a-47f1-81b5-8838096b65bf" />
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
<img width="1418" height="1402" alt="image" src="https://github.com/user-attachments/assets/36d77d20-559f-4272-b448-942351812c60" />
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

รูปผลการทดลอง
<img width="1806" height="350" alt="image" src="https://github.com/user-attachments/assets/de096603-5201-4c96-b6cc-bf28b902978e" />
```

-- ทดสอบปานกลาง  
SELECT * FROM simulate_oltp_workload(100);
### ผลการทดลอง
```
1. รูปผลการทดลอง
2. อธิบายผลการทดลอง การ SELECT , INSERT, UPDATE, DELETE เป็นอย่างไร
<img width="1472" height="370" alt="image" src="https://github.com/user-attachments/assets/210b4e6b-4d81-4a69-ab8b-cb2121b079ba" />
2.SELECT: การดึงข้อมูล

คำอธิบาย: คำสั่ง SELECT ใช้สำหรับ ดึงข้อมูล จากตารางที่ระบุ โดยไม่เปลี่ยนแปลงข้อมูลใดๆ ในตารางนั้น

ผลการทดลอง:

ถ้าเงื่อนไขที่ระบุ (เช่น WHERE) ตรงกับข้อมูล ในตาราง คำสั่งจะแสดง แถว (row) หรือ คอลัมน์ (column) ที่ตรงตามเงื่อนไข

ถ้าเงื่อนไข ไม่ตรงกับข้อมูล หรือไม่มีข้อมูลในตารางนั้นเลย ผลลัพธ์ที่ได้จะเป็น ชุดผลลัพธ์ที่ว่างเปล่า (empty result set)

สรุป: เป็นคำสั่งที่ปลอดภัยที่สุด เพราะเป็นเพียงการ อ่าน ข้อมูลเท่านั้น

INSERT: การเพิ่มข้อมูล

คำอธิบาย: คำสั่ง INSERT ใช้สำหรับ เพิ่มข้อมูลแถวใหม่ เข้าไปในตาราง

ผลการทดลอง:

ถ้าคำสั่ง สำเร็จ ข้อมูลใหม่จะถูกเพิ่มเข้าไปในตารางอย่างถาวร (permanently added) ซึ่งจะเพิ่มจำนวนแถวในตารางขึ้นหนึ่งแถว

ถ้าคำสั่ง ไม่สำเร็จ (เช่น ข้อมูลที่พยายามใส่ไม่ตรงกับชนิดข้อมูลของคอลัมน์ หรือละเมิดข้อจำกัดบางอย่าง เช่น Primary Key ที่ซ้ำกัน) จะเกิด ข้อผิดพลาด (error) และข้อมูลจะไม่ถูกเพิ่ม

สรุป: เป็นการ เพิ่ม ข้อมูลเข้าไปในฐานข้อมูล

UPDATE: การแก้ไขข้อมูล

คำอธิบาย: คำสั่ง UPDATE ใช้สำหรับ แก้ไขข้อมูล ในแถวที่มีอยู่แล้วในตาราง

ผลการทดลอง:

ต้องระบุเงื่อนไข WHERE เพื่อเลือกแถวที่จะแก้ไขอย่างถูกต้อง

ถ้าคำสั่ง สำเร็จ ข้อมูลในคอลัมน์ที่ระบุของแถวนั้นจะถูก เปลี่ยนแปลง ตามค่าใหม่ที่ให้ไว้

ถ้า ลืมใส่ คำสั่ง WHERE คำสั่ง UPDATE จะทำการ แก้ไขข้อมูลในทุกแถว ของตาราง ซึ่งอาจก่อให้เกิดความเสียหายอย่างร้ายแรงได้

สรุป: เป็นการ แก้ไข ข้อมูลที่มีอยู่แล้ว

DELETE: การลบข้อมูล

คำอธิบาย: คำสั่ง DELETE ใช้สำหรับ ลบข้อมูลแถว ออกจากตาราง

ผลการทดลอง:

ต้องระบุเงื่อนไข WHERE เพื่อเลือกแถวที่จะลบอย่างถูกต้อง

ถ้าคำสั่ง สำเร็จ แถวที่ตรงตามเงื่อนไขจะถูก ลบ ออกจากตารางอย่างถาวร ซึ่งจะทำให้จำนวนแถวในตารางลดลง

ถ้า ลืมใส่ คำสั่ง WHERE คำสั่ง DELETE จะทำการ ลบข้อมูลในทุกแถว ของตาราง ทำให้ตารางว่างเปล่า

สรุป: เป็นการ ลบ ข้อมูลออกจากฐานข้อมูล
```
SELECT * FROM simulate_oltp_workload(500);
### ผลการทดลอง
```
รูปผลการทดลอง
<img width="2052" height="428" alt="image" src="https://github.com/user-attachments/assets/5bea637f-5a24-43f5-a2f0-14468a89bd58" />
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
<img width="1270" height="346" alt="image" src="https://github.com/user-attachments/assets/7071e79d-a26a-42e9-b8f0-de0781a77330" />
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
<img width="2140" height="206" alt="image" src="https://github.com/user-attachments/assets/42f8dfde-05a5-4938-b5ca-af1477074367" />
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
<img width="1708" height="514" alt="image" src="https://github.com/user-attachments/assets/0a7df1ca-f75a-4ec8-bcf9-a6df6e485ba0" />
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
2. Work memory และ maintenance work memory คืออะไร มีหลักการในการกำหนดค่าอย่างไร
3. หากมี RAM 16GB และต้องการกำหนด connection = 200 ควรกำหนดค่า work memory และ maintenance work memory อย่างไร
4. ไฟล์ postgresql.conf และ postgresql.auto.conf  มีความสัมพันธ์กันอย่างไร
5. Buffer hit ratio คืออะไร
6. แสดงผลการคำนวณ การกำหนดค่าหน่วยความจำต่าง ๆ โดยอ้างอิงเครื่องของตนเอง
7. การสแกนของฐานข้อมูล PostgreSQL มีกี่แบบอะไรบ้าง เปรียบเทียบการสแกนแต่ละแบบ
คำตอบ
1. หน่วยความจำใดบ้างที่เป็น shared memory และมีหลักในการตั้งค่าอย่างไร

Shared memory ใน PostgreSQL คือส่วนของหน่วยความจำที่ถูกใช้ร่วมกันระหว่างกระบวนการต่างๆ ของ PostgreSQL ได้แก่:

shared_buffers: เป็นส่วนที่สำคัญที่สุดสำหรับ shared memory ใช้เก็บข้อมูลจากดิสก์ (data pages) ที่ถูกใช้งานบ่อยๆ เพื่อลดการอ่านจากดิสก์ซ้ำๆ ค่าเริ่มต้นมักจะถูกตั้งไว้ที่ 128MB หรือ 25% ของ RAM ทั้งหมดในเครื่อง

wal_buffers: ใช้เก็บข้อมูลที่กำลังจะถูกเขียนลงใน WAL (Write-Ahead Log) เป็นพื้นที่พักข้อมูลชั่วคราวช่วยลดจำนวนการเขียนลงดิสก์โดยตรง

backend_flush_buffers: เป็นพื้นที่สำหรับเก็บ dirty buffers ที่จะถูกเขียนลงดิสก์ในภายหลัง

maintenance_work_mem: ใช้สำหรับงานดูแลระบบ เช่น VACUUM, ANALYZE, และการสร้างดัชนี (index)

temp_buffers: ใช้สำหรับตารางชั่วคราว (temporary tables) ที่ถูกสร้างขึ้นในเซสชัน

หลักการตั้งค่า shared memory มักขึ้นอยู่กับปริมาณ RAM ที่มีในเครื่องและลักษณะการใช้งานของฐานข้อมูล โดยทั่วไปแล้ว เราจะปรับค่า shared_buffers ให้เหมาะสมกับจำนวน RAM ในเครื่อง โดยเฉพาะหากมี RAM เยอะๆ จะช่วยเพิ่มประสิทธิภาพได้มาก

<br>

2. Work memory และ maintenance work memory คืออะไร มีหลักการในการกำหนดค่าอย่างไร

work_mem: เป็นหน่วยความจำที่ใช้สำหรับแต่ละ session หรือ backend process สำหรับการเรียงลำดับ (sort) และการเชื่อมต่อ (hash join) ข้อมูล หากข้อมูลที่ต้องประมวลผลมีขนาดเกิน work_mem PostgreSQL จะต้องใช้พื้นที่ในดิสก์ชั่วคราว (temporary files) ซึ่งจะทำให้ประสิทธิภาพลดลง

maintenance_work_mem: เป็นหน่วยความจำที่ใช้สำหรับงานดูแลรักษาระบบ เช่น VACUUM, ANALYZE, CREATE INDEX และ ALTER TABLE การตั้งค่าที่สูงขึ้นจะช่วยให้งานเหล่านี้ทำงานได้เร็วขึ้น เพราะไม่ต้องใช้พื้นที่ในดิสก์ชั่วคราวมากนัก

หลักการในการกำหนดค่า:

work_mem: ควรตั้งค่าให้เหมาะสมกับแต่ละ session โดยไม่ให้สูงเกินไปจนกิน RAM ทั้งหมดของระบบ โดยปกติค่าเริ่มต้นจะอยู่ที่ 4MB อาจปรับเพิ่มขึ้นเป็น 16MB หรือ 32MB หากพบว่ามี temporary files เกิดขึ้นบ่อยๆ

maintenance_work_mem: ควรตั้งค่าให้สูงพอสำหรับงานบำรุงรักษา อาจตั้งค่าเป็น 1/4 หรือ 1/8 ของ RAM ทั้งหมด แต่ไม่ควรเกิน 1GB หรือ 2GB เพื่อป้องกันไม่ให้กินหน่วยความจำมากเกินไป

<br>

3. หากมี RAM 16GB และต้องการกำหนด connection = 200 ควรกำหนดค่า work memory และ maintenance work memory อย่างไร

จากข้อมูล RAM 16GB และ connection 200 สามารถคำนวณการกำหนดค่าได้ดังนี้:

shared_buffers: ควรกำหนดเป็น 25% ของ RAM ทั้งหมด (16GB) ซึ่งเท่ากับ 4GB

work_mem: หากกำหนด work_mem ไว้ที่ 16MB และมี connection พร้อมกัน 200 sessions จะใช้หน่วยความจำสูงสุดที่ $16MB \* 200 = 3,200MB$ หรือประมาณ 3.2GB ซึ่งยังเหลือพื้นที่สำหรับระบบและส่วนอื่นๆ อีกมาก

maintenance_work_mem: สามารถกำหนดค่าได้สูงขึ้นเพื่อช่วยให้งานบำรุงรักษาทำงานได้เร็วขึ้น เช่น 512MB หรือ 1GB เนื่องจากงานเหล่านี้มักจะไม่ทำงานพร้อมกัน

ดังนั้น การกำหนดค่าที่เหมาะสมสำหรับ RAM 16GB และ connection 200 ควรเป็น:

shared_buffers: 4GB

work_mem: 16MB

maintenance_work_mem: 512MB - 1GB

effective_cache_size: ควรกำหนดเป็น 50% - 75% ของ RAM ทั้งหมด ซึ่งเท่ากับ 8GB - 12GB

<br>

4. ไฟล์ postgresql.conf และ postgresql.auto.conf มีความสัมพันธ์กันอย่างไร

postgresql.conf: เป็นไฟล์การตั้งค่าหลักของ PostgreSQL มีค่าเริ่มต้นที่กำหนดไว้สำหรับ work_mem, shared_buffers และค่าอื่นๆ

postgresql.auto.conf: เป็นไฟล์ที่เก็บค่าการตั้งค่าที่ถูกปรับเปลี่ยนโดยใช้คำสั่ง ALTER SYSTEM เช่น ALTER SYSTEM SET work_mem = '32MB'; ค่าในไฟล์นี้จะมีลำดับความสำคัญสูงกว่าค่าใน postgresql.conf

ความสัมพันธ์ของทั้งสองไฟล์คือ PostgreSQL จะอ่านค่าการตั้งค่าจาก postgresql.conf ก่อน จากนั้นจึงอ่านและใช้ค่าจาก postgresql.auto.conf เพื่อ override ค่าเดิมที่ตั้งไว้ ทำให้ง่ายต่อการจัดการและตรวจสอบการเปลี่ยนแปลงที่เกิดขึ้นกับระบบ

<br>

5. Buffer hit ratio คืออะไร

Buffer hit ratio คืออัตราส่วนของจำนวน blocks ที่ถูกอ่านจาก shared_buffers (หรือ cache) ต่อจำนวน blocks ทั้งหมดที่ถูกร้องขอ ค่านี้บ่งชี้ถึงประสิทธิภาพของการใช้ shared_buffers ในการลดการอ่านข้อมูลจากดิสก์

Buffer hit ratio สูง (ใกล้ 100%) หมายถึง PostgreSQL สามารถดึงข้อมูลที่ต้องการจาก shared_buffers ได้บ่อยครั้ง ทำให้การทำงานรวดเร็วขึ้น

Buffer hit ratio ต่ำ หมายถึง PostgreSQL ต้องอ่านข้อมูลจากดิสก์บ่อยครั้ง ซึ่งจะทำให้ประสิทธิภาพของระบบลดลง

สูตรการคำนวณ Buffer hit ratio คือ:
$Buffer Hit Ratio = \frac{Blocks Read From Cache}{Blocks Read From Cache + Blocks Read From Disk} \* 100$

<br>

6. แสดงผลการคำนวณ การกำหนดค่าหน่วยความจำต่าง ๆ โดยอ้างอิงเครื่องของตนเอง

หมายเหตุ: สมมติว่าเครื่องมี RAM 16GB

shared_buffers: $16GB \* 0.25 = 4GB$

effective_cache_size: $16GB \* 0.75 = 12GB$

maintenance_work_mem: 16GB/16=1GB (หรืออาจตั้งเป็น 512MB)

work_mem: หากใช้ connection 200, เรากำหนด work_mem ที่ 16MB/connection ดังนั้นรวมเป็น $16MB \* 200 = 3.2GB$

<br>

7. การสแกนของฐานข้อมูล PostgreSQL มีกี่แบบอะไรบ้าง เปรียบเทียบการสแกนแต่ละแบบ

การสแกนของฐานข้อมูล PostgreSQL มีหลักๆ 3 แบบ ได้แก่:

Sequential Scan (การสแกนแบบลำดับ)

Index Scan (การสแกนด้วยดัชนี)

Bitmap Heap Scan (การสแกนแบบ Bitmap)

การเปรียบเทียบ:

ประเภทการสแกน	ลักษณะการทำงาน	ข้อดี	ข้อเสีย
Sequential Scan	อ่านข้อมูลทั้งหมดจากตารางตั้งแต่ต้นจนจบ	เหมาะสำหรับตารางขนาดเล็ก หรือเมื่อต้องการเข้าถึงข้อมูลส่วนใหญ่ของตาราง	ไม่มีประสิทธิภาพสำหรับตารางขนาดใหญ่ หรือเมื่อต้องการข้อมูลเพียงส่วนน้อย
Index Scan	ใช้ Index เพื่อค้นหาตำแหน่งของข้อมูลที่ต้องการ และไปดึงข้อมูลจากตารางจริง	เหมาะสำหรับตารางขนาดใหญ่ที่ต้องการข้อมูลเฉพาะเจาะจง	ไม่มีประสิทธิภาพเมื่อดึงข้อมูลจำนวนมาก เพราะต้องทำ 2 ขั้นตอน (หา Index -> ดึงข้อมูล)
Bitmap Heap Scan	ใช้ Index เพื่อหาตำแหน่งของข้อมูลที่ต้องการ (เก็บในรูปแบบ Bitmap) จากนั้นจึงเรียงลำดับและเข้าถึง Heap pages ทีละชุดเพื่อดึงข้อมูล	มีประสิทธิภาพเมื่อต้องดึงข้อมูลจำนวนมาก แต่ไม่ถึงกับต้องอ่านทั้งตาราง	ซับซ้อนกว่า Index Scan และต้องใช้หน่วยความจำในการสร้าง Bitmap
