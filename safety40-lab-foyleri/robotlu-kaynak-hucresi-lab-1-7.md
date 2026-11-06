# Safety 4.0 Lab Föyleri: Robotlu Kaynak Hücresi

**Hazırlayan:** Büşra Erdem

## Makine: Hürmaksan Robotlu MIG/MAG Kaynak Hücresi

Seçilen makine, [Hürmaksan Otomasyon Makina](https://hurmaksan.com/)'nın (Esenyurt, İstanbul) robotlu MIG/MAG kaynak hücresidir. Firmanın sitesinde belirtilenler: robot kaynak hücreleri, Siemens S7-1500 / Beckhoff PLC'ler, Pilz safety PLC'li CE güvenlik sistemleri ve devreye alma sonrası uzaktan bağlantı desteği. Firma model detayı yayınlamadığı için aşağıdaki konfigürasyon bu bilgilere dayanan **örnek** bir hücredir.

**Örnek hücre: varlıklar**

| Kategori | Varlıklar |
| --- | --- |
| Donanım | Kaynak robotu + robot kontrolörü, MIG/MAG güç kaynağı, Siemens S7-1500 PLC, Pilz safety PLC, operatör HMI'ı, 2 istasyonlu döner tabla, güvenlik çiti + kapı kilidi, yükleme tarafında ışık perdesi, acil stop butonları, duman emiş ünitesi, dikiş kontrol kamerası + IPC, endüstriyel switch |
| Yazılım / firmware | PLC programı, safety PLC konfigürasyonu, robot programları (kaynak reçeteleri), HMI yazılımı, IPC işletim sistemi + YZ görüntü modeli, güç kaynağı firmware'i |
| Hesaplar | Operatör, bakım teknisyeni, Hürmaksan servis hesabı, robot kontrolörü yönetici hesabı, IPC yerel yönetici |
| Sertifika / anahtar | Uzak erişim VPN anahtarı, IPC ile bulut arasındaki API anahtarı |
| Dış bağlantı | Uzak servis bağlantısı (tedarikçi), MES/ERP bağlantısı, bakımcının USB'si, servis dizüstü bilgisayarı |
| Kritik veri akışı | Kaynak parametreleri (akım, voltaj, hız) → robot; kamera görüntüsü → YZ → OK/RED; üretim verileri → MES |

**Neden bu makine:** İnsan için gerçek bir tehlike var (robot kolu, ark, sıcak parça, duman). Normal kontrol yolu ile safety yolu net biçimde ayrışıyor. Uzak erişim mevcut. Kamera + YZ olduğu için Lab 7 bu makineye doğal olarak oturuyor.

---

## LAB 1 · Safety Architecture → Safety Function Map

- **Normal kontrol yolu:** Parça sensörleri → S7-1500 PLC → robot kontrolörü + güç kaynağı + döner tabla. Uzak servis bu yola bağlı.
- **Safety yolu (ayrı):** Kapı kilidi + ışık perdesi + E-stop → Pilz safety PLC → robot STO + güç kaynağı kontaktörü → safe state.
- **Dijital kesişim noktası:** Robot kontrolörü ile safety PLC'nin aynı switch üzerinde olması ve uzak servisin safety konfigürasyonuna erişebilmesi.

1. **Tehlike nedir?** Robot kolunun operatöre çarpması veya onu sıkıştırması; kaynak arkı (göz/cilt yanığı, UV); sıcak parça ve sıçrantı (yanık, yangın); kaynak dumanı (soluma). En büyük tehlike, operatör döner tablaya parça yüklerken robotun hareket etmesidir.
2. **Safety function:** Yükleme tarafındaki ışık perdesi kesilirse veya çit kapısı açılırsa robot hareketi ve kaynak arkı güvenli şekilde durdurulur. Buna ek olarak E-stop ile acil durdurma yapılabilir.
3. **Safety-related control system nerede?** Kapı kilidi, ışık perdesi, E-stop, Pilz safety PLC, robot STO girişi ve güç kaynağı kontaktörü.
4. **Normal PLC ile Safety PLC arasındaki fark:** Normal PLC üretimi yönetir. Hata yaparsa ürün bozulur. Safety PLC ise yalnızca insanı korur. Çift kanallı, kendini test eden ve sertifikalıdır. Bir hata oluştuğunda makineyi her zaman güvenli tarafa çeker.
5. **PLr / SIL neden var?** Koruma riskle orantılı olmalıdır. Robot çarpması ağır yaralanmaya yol açar (S2), operatör her çevrimde tehlike bölgesine yaklaşır (F2) ve tehlikeden kaçınmak zordur (P2). Bu yüzden bu fonksiyon için PL d / SIL 2 civarında bir seviye gerekir. Bu güvenilirlik safety PLC konfigürasyonuna bağlıdır.
6. **Safe state:** Robot STO ile durmuş ve tork kesilmiş, kaynak arkı kapalı, döner tabla hareketsizdir. Yeniden başlatma yalnızca kapı kapanıp operatör reset verdiğinde mümkündür.
7. **Dijital bağımlılıklar:** Safety PLC konfigürasyon dosyası ve parolası, robot kontrolöründeki güvenli hız/bölge ayarları, PROFIsafe haberleşmesi, ışık perdesi parametreleri (muting/blanking) ve bunları değiştiren mühendislik dizüstü bilgisayarı.
8. **Siber müdahale nereden bozar?** Saldırgan uzak servis bağlantısı veya servis dizüstü bilgisayarı üzerinden robot kontrolörüne ya da safety PLC'ye ulaşabilir. Oradan ışık perdesinde muting açabilir veya robotun güvenli bölge sınırlarını genişletebilir. Sonuçta operatör yükleme yaparken robot hareket eder.

---

## LAB 2 · IEC 62443 Mini Workshop → Zone / Conduit Diagram

**Görev A**

1. **Bölgeler:**
    - Zone 1 – Safety: Pilz safety PLC, ışık perdesi, kapı kilidi, E-stop zinciri.
    - Zone 2 – Kontrol: S7-1500 PLC, robot kontrolörü, güç kaynağı, döner tabla sürücüsü, HMI.
    - Zone 3 – Görüntü/YZ: Dikiş kamerası + IPC.
    - Zone 4 – DMZ: Jump server, yama/güncelleme sunucusu.
    - Zone 5 – Kurumsal IT: Ofis ağı, ERP/MES, mühendislik istasyonu.
2. **Safety PLC nereye?** Kendi ayrı bölgesine (Zone 1), en yüksek güvenlik seviyesinde konur. Kontrol bölgesi ele geçirilse bile safety mantığı değiştirilememelidir.
3. **Conduit'ler:**
    - Zone 2 ↔ Zone 1: Yalnızca PROFIsafe ve durum okuma. Konfigürasyon yalnızca yerinde, fiziksel anahtarla değiştirilebilir.
    - Zone 3 → Zone 2: IPC yalnızca OK/RED sonucu gönderir, robot parametresi yazamaz.
    - Zone 4 ↔ Zone 2: Uzak servis yalnızca jump server üzerinden bağlanır. MFA, kişisel hesap ve oturum kaydı zorunludur.
    - Zone 5 ↔ Zone 4: Firewall'dan yalnızca MES verisi ve onaylı güncellemeler geçer.
    - USB / servis dizüstü: Yalnızca taranmış ve kayıtlı cihazlar kullanılır, USB portları kilitlidir.

**Görev B: Uzak erişim kararı**

| Karar | Seçim | Nedeni |
| --- | --- | --- |
| Aracı | Jump server | Tedarikçi DMZ'deki tek kontrollü noktadan girer. |
| Tünel | VPN | Trafik şifrelenir, cihaz portları internete açık kalmaz. |
| Kimlik | MFA | Parola çalınsa bile giriş yapılamaz. |
| Hesap | Kişisel | Kim ne yaptı izlenebilir. |
| Kayıt | Tam oturum kaydı | Bir olay olursa kanıt sağlar. |
| Zamanlama | Varsayılan kapalı | Bağlantı yalnızca onayla ve süreli açılır. |

---

## LAB 3 · Threat & Risk + Risk Matrix → Risk Register

Skor = L × S × E (her biri 1–3). Kritik 18–27 · Yüksek 9–17 · Orta 4–8 · Düşük 1–3.

| # | Senaryo | L | S | E | Skor | Öncelik |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Dış saldırgan → internete açık uzak servis (paylaşılan hesap) → robot güvenli bölgesini genişletir / ışık perdesine muting verir → operatör yükleme yaparken robot hareket eder, yaralanma | 3 | 3 | 3 | 27 | Kritik |
| 2 | Tedarik zinciri → enfekte servis dizüstü bilgisayarı → PLC'ye zararlı yazılım → duman emiş durur, HMI "çalışıyor" gösterir → zehirlenme / yangın | 2 | 3 | 2 | 12 | Yüksek |
| 3 | İçeriden → ortak HMI hesabı → kaynak akımı/voltajı yükseltilir → sıçrantı, yanık, yangın; zayıf dikişli parça müşteride kırılır | 2 | 2 | 1 | 4 | Orta |

1. **En yüksek öncelik:** Senaryo 1. Saldırgan internete açık bir kapıdan doğrudan safety fonksiyonuna ulaşabiliyor.
2. **S=3 olduğu halde skoru düşük kalan:** Senaryo 2. Skoru 12'de kalıyor ama S=3 olduğu için kırmızı işaretlenir ve 90 günlük plana zorunlu olarak girer.

---

## LAB 4 · Secure Remote Access → Secure Architecture

| | MEVCUT | HEDEF |
| --- | --- | --- |
| Yol | Tedarikçi → internet → 4G modem → robot/PLC | Tedarikçi → VPN → jump server → kontrol bölgesi |
| Ara nokta | Yok | Jump server |
| Hesap | Ortak "service" | Kişisel |
| Kimlik | Parola | Parola + MFA |
| Zamanlama | 7/24 açık | Varsayılan kapalı, onayla süreli açılır |
| Kayıt | Yok | Tam oturum kaydı |

1. **Saldırgan içeri girerse emniyet fonksiyonu etkilenir mi?** Mevcut durumda etkilenir. Hedef mimaride ise safety yolu bağımsız olduğu için saldırgan kapı açıkken robotu hareket ettiremez.
2. **Gereksinim:** "Işık perdesi / kapı kilidi / E-stop → robot STO emniyet fonksiyonu, normal kontrol sistemi ve uzak erişim yolu tamamen ele geçirilmiş olsa bile tasarlanan PL seviyesinde çalışmaya devam etmelidir. Emniyet parametreleri ağ üzerinden değiştirilememelidir."

---

## LAB 5 · SBOM / CVE / Patch → Vulnerability Decision

Verilen kayıt: Component X (PLC firmware kütüphanesi) · sürüm 2.4 · CVE-XXXX-XXXX · Critical (CVSS 9.8) · yama mevcut.

1. **Etkiliyor mu?** Etkiliyor. Bileşen S7-1500 firmware'inde var, kurulu sürüm 2.4 etkilenen aralıkta ve PLC'ye uzak servis üzerinden ulaşılabiliyor.
2. **Hemen durdurulur mu?** Hayır. Önce uzak servis kapatılır ve conduit kısıtlanır. Safety ayrı çalıştığı için insan korunmaya devam eder. Açık safety PLC'yi de etkiliyorsa üretim durdurulur.
3. **Doğrudan mı, test mi?** Önce test edilir. Yama test düzeneğinde ya da üretici onaylı ortamda denenir. Ardından yedek alınır ve yama planlı duruşta uygulanır.
4. **Safety validation etkilenir mi?** Yama safety PLC'ye, PROFIsafe'e veya robotun güvenli bölgesine dokunuyorsa validation tekrarlanır: ışık perdesi, kapı, E-stop ve durma süresi test edilir.
5. **Yama yapılamazsa:** Uzak servis kapalı tutulur, PLC portu firewall'da kapatılır, PLC ayrı alt bölgeye alınır, ağ izleme eklenir, USB kilitlenir ve durum risk register'a yazılır.

---

## LAB 6 · CRA Incident Simulation → Incident Workflow

Senaryo: Saat 10:15'te servis hesabında yetkisiz erişim tespit edildi. PLC konfigürasyonu değişmiş olabilir ve hücre hâlâ çalışıyor.

1. **10:15 (ilk 15 dk):** OT, IT güvenlik, iş güvenliği ve Hürmaksan servis aranır. Hücre kontrollü durdurulur. Uzak bağlantı fiziksel olarak kesilir ve hesap kilitlenir. Ekranlar fotoğraflanır, hiçbir şey resetlenmez.
2. **İlk 2 saat:** Safety fiziksel olarak test edilir ve checksum karşılaştırılır. Programlar değiştirilmeden yedeklenir, loglar dışa alınır. Servis hesabı ve aynı parolayı kullanan tüm hesaplar kapatılır.
3. **24 saat (erken uyarı):** Tespit zamanı, etkilenen ürün ve sürüm, aktif istismar şüphesi, ilk önlemler ve diğer müşterilerin durumu bildirilir.
4. **72 saat (bildirim):** Kök neden, etki ve önlemler raporlanır. OT teknik analizi, IT logları, safety sorumlusu doğrulamayı, yönetim de bildirimi üstlenir.
5. **14 gün (son rapor):** Jump server, VPN ve firewall logları, program kopyaları, checksum kayıtları, HMI alarmları, değişiklik kayıtları ve safety test tutanakları sunulur.

---

## LAB 7 · AI Safety Challenge → AI Fail-Safe Decision

Akış: Dikiş kamerası → YZ modeli (IPC) → OK / RED → kaynak parametresi → robot. YZ yalnızca kalite kararı verir. Robot hareketi ve safety üzerinde yetkisi yoktur.

| Başarısızlık | Güvenli davranış | Kavram |
| --- | --- | --- |
| Model yanlış sınıflandırdı | Kritik parçada örneklemeli insan kontrolü yapılır, YZ önerileri güvenli sınırlarla kırpılır | Human override + fail-safe |
| Girdi manipüle edildi | Görüntü kalitesi ve bütünlüğü izlenir. Şüpheli durumda parça karantinaya alınır, son onaylı parametreye dönülür, alarm verilir | Monitoring + fallback |
| Confidence düştü | %90 altında insana sorulur. Art arda düşük sonuç veya %70 altı → hücre durur, son onaylı reçeteye dönülür, bakım çağrılır | Confidence threshold + fallback |

## Kaynak

- [Hürmaksan Otomasyon Makina](https://hurmaksan.com/)
- Ders materyali: Safety 4.0 Bootcamp sunumu ve lab föyleri
