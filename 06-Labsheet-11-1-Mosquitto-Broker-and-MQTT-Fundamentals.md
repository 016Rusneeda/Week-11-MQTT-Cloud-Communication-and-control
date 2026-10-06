# ใบงานการทดลองที่ 11.1
### การติดตั้ง Mosquitto Broker และการทดสอบโปรโตคอล MQTT ด้วย CLI (Pub/Sub, QoS, Retain, LWT)

> [!NOTE] **คำชี้แจง**
> ในใบงานนี้นักศึกษาจะได้เรียนรู้การติดตั้งและตั้งค่า **Eclipse Mosquitto MQTT Broker** บนเครื่องคอมพิวเตอร์เพื่อทำหน้าที่เป็นตัวกลางรับส่งข้อมูลของระบบ IoT จากนั้นใช้เครื่องมือบรรทัดคำสั่ง **`mosquitto_sub`** และ **`mosquitto_pub`** ในการทดลองคุณสมบัติสำคัญของโปรโตคอล MQTT ได้แก่ ระบบหัวข้อ (Topics & Wildcards), ระดับคุณภาพการให้บริการ (QoS 0, 1, 2), ข้อความคงค้าง (Retained Messages) และพินัยกรรมฉุกเฉิน (Last Will and Testament - LWT)

---

## 1. วัตถุประสงค์การทดลอง
1. สามารถติดตั้งและกำหนดค่าไฟล์คอนฟิกูเรชันของ Eclipse Mosquitto MQTT Broker ได้อย่างถูกต้อง
2. สามารถใช้คำสั่ง `mosquitto_sub` และ `mosquitto_pub` ในการบอกรับและเผยแพร่ข้อความผ่านเครือข่ายได้
3. เข้าใจความแตกต่างและสามารถใช้งานตัวกรองไวลด์การ์ดระดับเดียว (`+`) และหลายระดับ (`#`) ได้
4. สามารถเปรียบเทียบพฤติกรรมการรับส่งข้อมูลที่ระดับ QoS 0, QoS 1 และ QoS 2 พร้อมเข้าใจกลไก Packet Identifier และ ACKs
5. สามารถอธิบายและสาธิตการทำงานของ Retained Message และ Last Will and Testament (LWT) ได้

---

## 2. อุปกรณ์และเครื่องมือที่ใช้
* เครื่องคอมพิวเตอร์ระบบปฏิบัติการ Windows หรือ Linux/macOS
* ซอฟต์แวร์ **Eclipse Mosquitto** (พร้อมเครื่องมือ Client CLI) หรือ Docker Engine
* หน้าต่างเทอร์มินัล (PowerShell หรือ Bash Terminal) อย่างน้อย 3 หน้าต่าง

---

## 3. ขั้นตอนการทดลอง (Deconstructed Activities)

### กิจกรรมที่ 11-1.1: การติดตั้งและเริ่มต้น Mosquitto Broker

#### 1. การติดตั้งโปรแกรม
* **สำหรับ Windows**: ดาวน์โหลดตัวติดตั้ง `mosquitto-2.x.x-install-windows-x64.exe` จาก [mosquitto.org](https://mosquitto.org/download/) และติดตั้งลงในเครื่อง
* **สำหรับ Linux (Ubuntu/Debian)**:
  ```bash
  sudo apt update && sudo apt install -y mosquitto mosquitto-clients
  ```
* **หรือรันผ่าน Docker**:
  ```bash
  docker run -d --name mosquitto-test -p 1883:1883 eclipse-mosquitto:2
  ```

#### 2. การสร้างไฟล์คอนฟิกูเรชันเฉพาะ (`mosquitto.conf`)
สร้างโฟลเดอร์สำหรับแล็บ เช่น `C:\mqtt_lab\` (หรือ `~/mqtt_lab/`) แล้วสร้างไฟล์ชื่อ `mosquitto.conf`:

```text
# กำหนดพอร์ตและอนุญาตให้ทุกอินเทอร์เฟซเชื่อมต่อได้
listener 1883 0.0.0.0

# อนุญาตการเชื่อมต่อแบบ Anonymous (สำหรับการทดลองในห้องปฏิบัติการ)
allow_anonymous true

# แสดงบันทึกข้อความอย่างละเอียด
log_type all
```

#### 3. เริ่มต้นการทำงานของ Mosquitto Broker
เปิด **เทอร์มินัลที่ 1** แล้วสั่งรัน Broker ในโหมดแสดงผลละเอียด (Verbose):

```powershell
# บน Windows (ระบุ Path ไฟล์คอนฟิกที่สร้างไว้)
& "C:\Program Files\mosquitto\mosquitto.exe" -c "C:\mqtt_lab\mosquitto.conf" -v

# บน Linux
mosquitto -c ~/mqtt_lab/mosquitto.conf -v
```

**ผลลัพธ์ที่คาดหวังในเทอร์มินัลที่ 1**:
```text
1696500000: mosquitto version 2.0.18 starting
1696500000: Config loaded from C:\mqtt_lab\mosquitto.conf.
1696500000: Opening ipv4 listen socket on port 1883.
1696500000: mosquitto version 2.0.18 running
```

---

### กิจกรรมที่ 11-1.2: การทดสอบ Publish / Subscribe พื้นฐาน และการใช้ Wildcards

#### 1. การ Subscribe หัวข้อเฉพาะเจาะจง
เปิด **เทอร์มินัลที่ 2** เพื่อจำลองเป็นอุปกรณ์ Subscriber สังเกตการณ์:
```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "home/livingroom/light" -v
```

เปิด **เทอร์มินัลที่ 3** เพื่อจำลองเป็น Publisher ส่งข้อความสั่งงาน:
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "home/livingroom/light" -m "ON"
```
*สังเกตผล*: ในเทอร์มินัลที่ 2 จะได้รับข้อความ `home/livingroom/light ON` ทันที

#### 2. การทดสอบไวลด์การ์ดระดับเดียว (Single-Level Wildcard: `+`)
ใน **เทอร์มินัลที่ 2** กด `Ctrl + C` แล้วเปลี่ยนคำสั่ง Subscribe เป็น:
```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "home/+/light" -v
```

ใน **เทอร์มินัลที่ 3** ทดลอง Publish ข้อความ 3 ครั้ง:
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "home/livingroom/light" -m "ON"
mosquitto_pub -h 127.0.0.1 -p 1883 -t "home/kitchen/light" -m "OFF"
mosquitto_pub -h 127.0.0.1 -p 1883 -t "home/bedroom/balcony/light" -m "ON"
```

> [!NOTE]
> บันทึกผลการทดลอง: สังเกตว่าข้อความใดบ้างที่ปรากฏในเทอร์มินัลที่ 2 และข้อความใดที่ไม่ปรากฏ พร้อมอธิบายเหตุผลว่าทำไม `home/bedroom/balcony/light` จึงไม่แสดงผล

#### 3. การทดสอบไวลด์การ์ดหลายระดับ (Multi-Level Wildcard: `#`)
ใน **เทอร์มินัลที่ 2** เปลี่ยนคำสั่ง Subscribe เป็น:
```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "home/#" -v
```

ใน **เทอร์มินัลที่ 3** ทดลอง Publish ข้อความไปยังหัวข้อต่างๆ:
```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "home/livingroom/temp" -m "26.4"
mosquitto_pub -h 127.0.0.1 -p 1883 -t "home/bedroom/balcony/light" -m "ON"
```
*สังเกตผล*: ทุกข้อความที่ขึ้นต้นด้วย `home/` จะถูกส่งมาแสดงผลทั้งหมด

---

### กิจกรรมที่ 11-1.3: การทดสอบระดับคุณภาพการส่งข้อมูล (QoS 0, QoS 1, QoS 2)

ในกิจกรรมนี้ ให้สังเกตข้อความ Log ใน **เทอร์มินัลที่ 1 (Broker)** ควบคู่กับคำสั่งในเทอร์มินัลที่ 2 และ 3:

#### 1. ทดสอบ QoS 0 (At Most Once)
* ในเทอร์มินัล 2: `mosquitto_sub -h 127.0.0.1 -t "test/qos" -q 0 -v`
* ในเทอร์มินัล 3: `mosquitto_pub -h 127.0.0.1 -t "test/qos" -m "QoS 0 Message" -q 0`
* *สังเกต Log ฝั่ง Broker*: มีเพียงแพ็กเก็ต `PUBLISH` ส่งผ่านไปโดยไม่มีแพ็กเก็ตตอบรับใดๆ

#### 2. ทดสอบ QoS 1 (At Least Once)
* ในเทอร์มินัล 2: `mosquitto_sub -h 127.0.0.1 -t "test/qos" -q 1 -v`
* ในเทอร์มินัล 3: `mosquitto_pub -h 127.0.0.1 -t "test/qos" -m "QoS 1 Message" -q 1`
* *สังเกต Log ฝั่ง Broker*: จะปรากฏแพ็กเก็ต `PUBLISH (d0, q1, m1, ...)` ตามด้วยแพ็กเก็ต `PUBACK (Mid: 1)` เพื่อยืนยันการรับส่งข้อมูล

#### 3. ทดสอบ QoS 2 (Exactly Once)
* ในเทอร์มินัล 2: `mosquitto_sub -h 127.0.0.1 -t "test/qos" -q 2 -v`
* ในเทอร์มินัล 3: `mosquitto_pub -h 127.0.0.1 -t "test/qos" -m "QoS 2 Message" -q 2`
* *สังเกต Log ฝั่ง Broker*: จะปรากฏกระบวนการ 4-way Handshake อย่างชัดเจน:
  1. `Received PUBLISH ... q2`
  2. `Sending PUBREC`
  3. `Received PUBREL`
  4. `Sending PUBCOMP`

---

### กิจกรรมที่ 11-1.4: การทดสอบข้อความคงค้าง (Retained Messages)

1. ปิด Subscriber ในเทอร์มินัลที่ 2 ทั้งหมด (ไม่มีใคร Subscribe อยู่)
2. ใน **เทอร์มินัลที่ 3** ส่งข้อความธรรมดา (ไม่ใส่พารามิเตอร์ `-r`):
   ```powershell
   mosquitto_pub -h 127.0.0.1 -t "sensor/state" -m "Normal"
   ```
3. เปิด Subscriber ใน **เทอร์มินัลที่ 2**:
   ```powershell
   mosquitto_sub -h 127.0.0.1 -t "sensor/state" -v
   ```
   *ผลที่ได้*: หน้าจอว่างเปล่า ไม่ได้รับข้อความใดๆ เพราะข้อความถูกส่งผ่านไปแล้วก่อนหน้านี้

4. ใน **เทอร์มินัลที่ 3** สั่ง Publish โดยเพิ่มพารามิเตอร์ **`-r` (Retain)**:
   ```powershell
   mosquitto_pub -h 127.0.0.1 -t "sensor/state" -m "CRITICAL_ALERT" -r
   ```
   *ผลที่ได้*: ในเทอร์มินัลที่ 2 ได้รับข้อความ `sensor/state CRITICAL_ALERT` ทันที

5. ปิดเทอร์มินัลที่ 2 (`Ctrl + C`) แล้วเปิดใหม่อีกครั้ง:
   ```powershell
   mosquitto_sub -h 127.0.0.1 -t "sensor/state" -v
   ```
   *ผลที่ได้*: ทันทีที่เชื่อมต่อสำเร็จ เทอร์มินัลที่ 2 จะได้รับข้อความ `CRITICAL_ALERT` ในทันที แม้ Publisher จะไม่ได้ส่งข้อความใหม่เลยก็ตาม

> [!TIP]
> **วิธีลบ Retained Message ออกจาก Broker**: ส่งข้อความที่มีเนื้อหาว่างเปล่า (Empty Payload) พร้อมเปิดแฟล็ก `-r`:
> ```powershell
> mosquitto_pub -h 127.0.0.1 -t "sensor/state" -n -r
> ```

---

### กิจกรรมที่ 11-1.5: การทดสอบพินัยกรรมฉุกเฉิน (Last Will and Testament - LWT)

1. ใน **เทอร์มินัลที่ 2** เปิด Subscribe เพื่อรอเฝ้าดูสถานะของโหนด:
   ```powershell
   mosquitto_sub -h 127.0.0.1 -t "node/esp32/status" -v
   ```
2. ใน **เทอร์มินัลที่ 3** เปิด Client ที่จำลองเป็น ESP32 โดยกำหนดค่า **LWT** ไว้ล่วงหน้า:
   ```powershell
   mosquitto_sub -h 127.0.0.1 -i "Node_Client_01" -t "dummy/listen" --will-topic "node/esp32/status" --will-payload "LOST_CONNECTION_OFFLINE" --will-qos 1
   ```
3. ส่งสถานะเริ่มต้นว่า "ONLINE" จากอุปกรณ์:
   (เปิดเทอร์มินัลที่ 4 แล้วพิมพ์ `mosquitto_pub -h 127.0.0.1 -t "node/esp32/status" -m "ONLINE"`)
   *สังเกตผล*: เทอร์มินัลที่ 2 แสดงสถานะ `ONLINE`
4. **จำลองเหตุการณ์ไฟดับหรือเครือข่ายหลุดกะทันหัน**:  
   ในเทอร์มินัลที่ 3 ให้ทำการ **บังคับปิดหน้าต่างเทอร์มินัลทันที** หรือกดปิดโปรเซส เพื่อไม่ให้ Client ส่งแพ็กเก็ต `DISCONNECT`
5. *สังเกตผลในเทอร์มินัลที่ 2*:  
   เมื่อ Broker ตรวจพบว่า TCP Connection ขาดหาย Broker จะเป็นผู้ส่งข้อความ:
   ```text
   node/esp32/status LOST_CONNECTION_OFFLINE
   ```
   มายังเทอร์มินัลที่ 2 โดยอัตโนมัติ

---

## 4. การบันทึกและวิเคราะห์ผลการทดลอง

ให้นักศึกษาบันทึกภาพหน้าจอ (Screenshots) และตอบคำถามต่อไปนี้ลงในรายงาน:
1. ภาพหน้าจอการทำงานของ Mosquitto Broker Verbose Log ขณะที่มีการเชื่อมต่อและ Subscribe ข้อความ
2. ตารางเปรียบเทียบลำดับแพ็กเก็ตที่เกิดขึ้นระหว่าง QoS 0, QoS 1 และ QoS 2 ที่สังเกตได้จาก Broker Log
3. ภาพการทดสอบ Retained Message เมื่อเปิด Subscriber ขึ้นมาใหม่

---

## 5. คำถามท้ายการทดลอง (Post-Lab Questions)
1. เพราะเหตุใดสัญลักษณ์ `#` จึงต้องวางไว้ที่ตำแหน่งท้ายสุดของ Topic เสมอ? หากกำหนด Topic Filter เป็น `home/#/light` จะเกิดข้อผิดพลาดอย่างไร?
2. ในระบบงานติดตามยานพาหนะ (GPS Tracking) ที่ส่งพิกัดรถยนต์ทุกๆ 1 วินาที ท่านคิดว่าควรเลือกระดับ QoS ใดระหว่าง QoS 0, 1 หรือ 2 เพราะเหตุใด?
3. หาก Client ส่งแพ็กเก็ต `DISCONNECT` ไปยัง Broker อย่างถูกต้องก่อนปิดโปรแกรม ข้อความ Last Will and Testament (LWT) จะถูกเผยแพร่ออกไปหรือไม่ เพราะเหตุใด?
4. Retained Message มีประโยชน์อย่างไรในการออกแบบระบบแดชบอร์ด Smart Home สำหรับผู้ใช้งานที่เพิ่งเปิดแอปพลิเคชันขึ้นมาเป็นครั้งแรก?
