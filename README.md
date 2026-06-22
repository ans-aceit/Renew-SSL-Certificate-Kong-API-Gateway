# SSL Certificate Renewal Guide for Kong API Gateway

## สภาพแวดล้อม
- **OS:** Ubuntu (Linux)
- **API Gateway:** Kong 2.8.1
- **Certificate Manager:** Let's Encrypt (Certbot)
- **Container:** Docker

---

## ปัญหาที่พบ
- Browser แสดง `NET::ERR_CERT_DATE_INVALID`
- Kong ACME Plugin ไม่สามารถ renew ได้อัตโนมัติเนื่องจาก `certificate chain too long`
- สาเหตุ: Let's Encrypt เปลี่ยน Root CA ใหม่ (Root YR) ใน Kong 2.8.1 รองรับไม่ได้

---

## ขั้นตอนการ Renew Certificate

### 1. ตรวจสอบ Certificate ที่หมดอายุ
```bash
# แก้ไข domain ให้ตรงกับของคุณ
# ตัวอย่าง: api.example.com system.example.com file.example.com app.example.com
for domain in <domain1> <domain2> <domain3> <domain4>; do
  echo "=== $domain ==="
  echo | openssl s_client -connect $domain:443 2>/dev/null \
    | openssl x509 -noout -dates
  echo ""
done
```

### 2. หยุด Kong เพื่อให้ Port 80 ว่าง
```bash
docker stop kong
```

### 3. Renew Certificate ด้วย Certbot
```bash
# แก้ไข <domain> ให้ตรงกับ domain ที่ต้องการ renew
# ตัวอย่าง: certbot certonly --standalone -d api.example.com --force-renewal
sudo certbot certonly --standalone \
  -d <domain> \
  --force-renewal
```

### 4. Start Kong กลับ
```bash
docker start kong
```

### 5. ลบ Certificate เก่าออกจาก Kong
```bash
# ดู Certificate ID ทั้งหมด
curl -s http://localhost:8001/certificates | python3 -m json.tool | grep -E '"id"|"snis"'

# ลบ cert เก่า โดยแก้ไข <old-cert-id> ให้ตรงกับ ID ที่ต้องการลบ
# ตัวอย่าง: curl -X DELETE http://localhost:8001/certificates/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
curl -X DELETE http://localhost:8001/certificates/<old-cert-id>
```

### 6. Import Certificate ใหม่เข้า Kong
```bash
# แก้ไข <domain> ให้ตรงกับ domain ที่ renew
# ตัวอย่าง: --data-urlencode "cert@/etc/letsencrypt/live/api.example.com/fullchain.pem"
curl -X POST http://localhost:8001/certificates \
  --data-urlencode "cert@/etc/letsencrypt/live/<domain>/fullchain.pem" \
  --data-urlencode "key@/etc/letsencrypt/live/<domain>/privkey.pem"
```

### 7. ผูก SNI กับ Certificate ใหม่
```bash
# แก้ไข <domain> และ <new-cert-id> ให้ตรงกับข้อมูลของคุณ
# ตัวอย่าง: curl -X POST http://localhost:8001/snis -d "name=api.example.com" -d "certificate.id=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
curl -X POST http://localhost:8001/snis \
  -d "name=<domain>" \
  -d "certificate.id=<new-cert-id>"
```

### 8. ตรวจสอบว่าสำเร็จ
```bash
# แก้ไข <domain> ให้ตรงกับ domain ที่ต้องการตรวจสอบ
echo | openssl s_client -connect <domain>:443 2>/dev/null \
  | openssl x509 -noout -dates
```

---

## Auto Renew Script

### สร้าง Script
```bash
sudo nano /usr/local/bin/renew-certs.sh
```

```bash
#!/bin/bash

# ============================================
# Auto Renew SSL Certificate for Kong
# ============================================

# แก้ไข DOMAINS ให้ตรงกับ domain ของคุณ
# ตัวอย่าง: DOMAINS=("api.example.com" "system.example.com")
DOMAINS=("<domain1>" "<domain2>")

# แก้ไข KONG_ADMIN ถ้า Kong Admin API อยู่ที่ port หรือ host อื่น
# ตัวอย่าง: KONG_ADMIN="http://localhost:8001"
KONG_ADMIN="http://localhost:8001"

# แก้ไข RENEW_THRESHOLD_DAYS ถ้าต้องการ renew ก่อนหมดอายุกี่วัน
# ตัวอย่าง: RENEW_THRESHOLD_DAYS=14
RENEW_THRESHOLD_DAYS=14

LOG_PREFIX="[$(date '+%Y-%m-%d %H:%M:%S')]"

echo "$LOG_PREFIX ===== Start Renew Certificate ====="

# Step 1: ตรวจสอบว่า cert ใกล้หมดอายุไหม
NEED_RENEW=false

for domain in "${DOMAINS[@]}"; do
  EXPIRY=$(echo | openssl s_client -connect $domain:443 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null \
    | cut -d= -f2)

  EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
  NOW_EPOCH=$(date +%s)
  DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

  echo "$LOG_PREFIX $domain: เหลืออีก $DAYS_LEFT วัน (หมดอายุ: $EXPIRY)"

  if [ $DAYS_LEFT -lt $RENEW_THRESHOLD_DAYS ]; then
    echo "$LOG_PREFIX $domain: ใกล้หมดอายุ! เริ่ม renew..."
    NEED_RENEW=true
  fi
done

if [ "$NEED_RENEW" = false ]; then
  echo "$LOG_PREFIX ทุก domain ยังไม่หมดอายุ ไม่ต้อง renew"
  echo "$LOG_PREFIX ===== End ====="
  exit 0
fi

# Step 2: หยุด Kong
echo "$LOG_PREFIX หยุด Kong..."
docker stop kong
sleep 3

# Step 3: Renew Certificate ด้วย Certbot
for domain in "${DOMAINS[@]}"; do
  EXPIRY=$(echo | openssl s_client -connect $domain:443 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null \
    | cut -d= -f2)

  EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
  NOW_EPOCH=$(date +%s)
  DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

  if [ $DAYS_LEFT -lt $RENEW_THRESHOLD_DAYS ]; then
    echo "$LOG_PREFIX Renew $domain..."
    certbot certonly --standalone \
      -d $domain \
      --force-renewal \
      --non-interactive \
      --quiet

    if [ $? -eq 0 ]; then
      echo "$LOG_PREFIX Renew $domain สำเร็จ"
    else
      echo "$LOG_PREFIX Renew $domain ล้มเหลว!"
    fi
  fi
done

# Step 4: Start Kong กลับ
echo "$LOG_PREFIX Start Kong..."
docker start kong
sleep 5

# Step 5: ลบ cert เก่าและ Import cert ใหม่เข้า Kong
for domain in "${DOMAINS[@]}"; do
  EXPIRY=$(cat /etc/letsencrypt/live/$domain/cert.pem \
    | openssl x509 -noout -enddate 2>/dev/null \
    | cut -d= -f2)

  EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
  NOW_EPOCH=$(date +%s)
  DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

  if [ $DAYS_LEFT -gt 89 ]; then
    echo "$LOG_PREFIX Import cert ใหม่สำหรับ $domain..."

    # ดึง cert id เก่าของ domain นี้จาก SNI
    OLD_CERT_ID=$(curl -s $KONG_ADMIN/snis/$domain \
      | python3 -c "import sys,json; print(json.load(sys.stdin)['certificate']['id'])" 2>/dev/null)

    # Import cert ใหม่
    RESPONSE=$(curl -s -X POST $KONG_ADMIN/certificates \
      --data-urlencode "cert@/etc/letsencrypt/live/$domain/fullchain.pem" \
      --data-urlencode "key@/etc/letsencrypt/live/$domain/privkey.pem")

    NEW_CERT_ID=$(echo $RESPONSE \
      | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])" 2>/dev/null)

    if [ -n "$NEW_CERT_ID" ]; then
      # อัปเดต SNI ให้ชี้ไป cert ใหม่
      curl -s -X PATCH $KONG_ADMIN/snis/$domain \
        -d "certificate.id=$NEW_CERT_ID" > /dev/null
      echo "$LOG_PREFIX อัปเดต SNI $domain -> $NEW_CERT_ID สำเร็จ"

      # ลบ cert เก่าออก
      if [ -n "$OLD_CERT_ID" ]; then
        curl -s -X DELETE $KONG_ADMIN/certificates/$OLD_CERT_ID
        echo "$LOG_PREFIX ลบ cert เก่า $OLD_CERT_ID สำเร็จ"
      fi
    else
      echo "$LOG_PREFIX Import cert ล้มเหลวสำหรับ $domain!"
    fi
  fi
done

# Step 6: Reload Kong
echo "$LOG_PREFIX Reload Kong..."
docker exec kong kong reload

echo "$LOG_PREFIX ===== End Renew Certificate ====="
```

### ติดตั้ง Script
```bash
# ให้สิทธิ์ execute
sudo chmod +x /usr/local/bin/renew-certs.sh

# ทดสอบรันก่อน
sudo /usr/local/bin/renew-certs.sh

# ตั้ง cron job รันทุกวันตี 3
sudo crontab -e
```

เพิ่มบรรทัดนี้ใน crontab:
```bash
0 3 * * * /usr/local/bin/renew-certs.sh >> /var/log/renew-certs.log 2>&1
```

### ดู Log
```bash
tail -f /var/log/renew-certs.log
```
