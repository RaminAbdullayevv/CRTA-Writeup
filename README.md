# CRTA Exam Writeup — Passed ✅ | CyberWarFare Labs

> **CRTA (Certified Red Team Analyst)** — CyberWarFare Labs-ın tam praktiki, black-box red team imtahanı.

---

## 🏆 Nəticə

| | |
|---|---|
| **Status** | ✅ PASSED |
| **Sertifikat** | CRTA — Certified Red Team Analyst |
| **Təşkilat** | CyberWarFare Labs (CWL) |
| **İmtahan növü** | Black-box, VPN, flag-based |
| **Hədəf** | `secret.xml` → Domain Controller |

---

## 📋 İmtahan Haqqında

CRTA real şəbəkə mühitinə qoşulub çox seçimli sual olmadan aparılan praktiki bir imtahandır. Hədəf: çox qatlı infrastrukturu kompromis etmək, imtiyazları artırmaq, lateral movement icra etmək və Domain Controller-dən `secret.xml` faylını əldə etmək.

**Hücum zənciri (qısaca):**

```
SSRF → Lokal fayl oxuma → SSH → sudo vi (GTFOBins) → ROOT
→ Docker → Daxili şəbəkə kəşfi → elfinder → AD Creds
→ DCSync → Pass-the-Hash → Domain Admin → secret.xml ✅
```

---

## 🗂️ Məzmun

```
CRTA-Writeup/
├── README.md           ← Bu fayl
└── crta_writeup.md     ← Tam writeup (Azərbaycan dilində)
```

---

## 🔍 Əhatə Edilən Texnikalar

| Kateqoriya | Texnikalar |
|---|---|
| **Kəşfiyyat** | Nmap, Gobuster |
| **Veb Hücumlar** | SSRF (file:// sxemi, Docker hostfs bypass) |
| **İlkin Giriş** | SSH (plaintext credentials via SSRF) |
| **Privilege Escalation** | GTFOBins — sudo vi |
| **Docker** | Konteyner analizi, hothost.json, MD5 flag |
| **Lateral Movement** | auth.log analizi, /elfinder, AD credentials |
| **Active Directory** | DCSync (impacket-secretsdump) |
| **Post-Exploit** | Pass-the-Hash (smbclient), secret.xml əldə etmə |

---

## 🖥️ Şəbəkə Mühiti

| Komponent | IP |
|---|---|
| VPN Scope | 172.26.10.0/24 |
| İlkin hədəf | 172.26.10.11 |
| Daxili şəbəkə | 10.10.10.0/24 |
| Domain Controller | 10.10.10.100 |

---

## ⚙️ İstifadə Edilən Alətlər

```
nmap · curl · ssh · gobuster · 
impacket-secretsdump · smbclient · docker
```

---

## 📄 Tam Writeup

Addım-addım hücum metodologiyası, bütün komandalar və izahatlar üçün **[crta_writeup.md](./crta_writeup.md)** faylına baxın.

---

## ⚠️ Etik Xəbərdarlıq

> Bu writeup **yalnız təhsil məqsədi** üçün hazırlanmışdır.  
> CyberWarFare Labs metodologiyanın paylaşılmasına açıq şəkildə icazə verir.  
> Heç bir tam credentials və ya flag burada açıqlanmır.  
> Bu texnikaları icazəsiz sistemlərə tətbiq etmək qanunsuz və etik deyildir.

---

## 👤 Müəllif

**Ramin Abdullayev**  
📧 raminabdullayev048@gmail.com  
🏅 CRTA — CyberWarFare Labs Certified Red Team Analyst

---

## 🔗 Digər Writeup-lar

- [eJPT Writeup](../eJPT-Writeup/) — INE Security Junior Penetration Tester

---

*© 2026 — Şəxsi təhsil məqsədi ilə hazırlanmışdır.*
