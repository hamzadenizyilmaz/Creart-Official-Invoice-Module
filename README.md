# Creart Resmi Fatura Modulü

WHMCS 8.10+ / 9.x için geliştirilmiş, Türkiye odaklı resmi fatura PDF üretimi ve muhasebe provider entegrasyonu sunan addon modül.

## İçindekiler
1. Ürün Özeti
2. Kapsam ve Sınırlar
3. Mimari Genel Bakış
4. Yıllık Lisans Modeli
5. Teknik Gereksinimler
6. Kurulum
7. İlk Konfigürasyon
8. Provider Konfigürasyonu
9. PDF Üretim Kuralları
10. Lisans Güvenlik Modeli
11. Hook ve İş Akışı Matrisi
12. Veritabanı Şeması
13. Admin Arayüzü
14. API/Lisans Sunucusu Beklentisi
15. Test ve Kalite
16. Hata Senaryoları ve Troubleshooting
17. Production Checklist
18. Sürümleme ve Destek

## 1) Ürün Özeti
`ResmiFaturaModulu`, WHMCS faturalarını resmi görünümlü PDF çıktısına dönüştürür ve fatura verisini seçili provider’a (Paraşüt/Logo/Mikro) aktarır.

### Başlıca Yetkinlikler
- TCPDF ile resmi fatura düzeni.
- QR kod, kaşe/imza alanı, yasal metin bloğu.
- Provider abstraction (`FaturaProviderInterface`) ile genişletilebilir mimari.
- Gelişmiş lisans doğrulama (imza, fingerprint, şifreli cache).
- Hook tabanlı otomasyon (fatura oluşumu, ödeme, günlük cron).

## 2) Kapsam ve Sınırlar
- Bu modül **doğrudan GİB API entegrasyonu içermez**.
- Akış, WHMCS veri modeli + seçili muhasebe provider API’leri üzerinden ilerler.
- GİB tarafına direkt gönderim yerine “resmi formatlı PDF + provider entegrasyonu” odaklanır.

## 3) Mimari Genel Bakış

### Katmanlar
- `Entry`: `resmifaturamodulu.php`
- `Hooks`: `hooks.php`
- `License`: `src/License/LicenseValidator.php`
- `PDF`: `src/Pdf/InvoicePdfGenerator.php`
- `Providers`: `src/Providers/*`
- `Support`: `src/Support/*`
- `Templates`: `templates/admin/*`

### Tasarım Prensipleri
- `strict_types=1`
- PSR-4 autoload
- Interface + Factory tabanlı provider seçimi
- Güvenlikte fail-closed yaklaşımı

## 4) Yıllık Lisans Modeli
Bu ürün **yıllık lisans** ile satılır.

- Lisans süresi: `365 gün`.
- Lisans bağlama: `domain + IP + instance_id + fingerprint`.
- Süre sonu: otomatik akışlar kısıtlanabilir, yenileme gerekir.
- Yenileme kapsamı: sürüm güncellemeleri, güvenlik yamaları, teknik destek.
- Lisans devri: satıcı onayı ile kontrollü.

### Önerilen Ticari Politika
- Ana lisans: 1 production domain.
- Opsiyonel ek lisans: staging/dev domain.
- Yenileme gecikmesi sonrası grace period: satıcı politikasına göre tanımlanabilir.

## 5) Teknik Gereksinimler
- PHP `8.2+`
- IonCube Loader `14+`
- WHMCS `8.10+` / `9.x`
- Composer
- OpenSSL eklentisi (AES-256-GCM için önerilir)
- İnternet erişimi (lisans doğrulama ve provider API çağrıları için)

## 6) Kurulum
1. Modül klasörünü WHMCS kökünde aşağıya yerleştirin:
   - `modules/addons/resmifaturamodulu/`
2. Bağımlılıkları kurun:

```bash
composer install --no-dev
```

3. WHMCS Admin panelde `Addon Modules` bölümünden modülü aktive edin.
4. Aktivasyonda lisans anahtarını girin.
5. Provider ayarlarını kaydedin.
6. Gerekli client custom field’ları doğrulayın:
   - `Vergi Dairesi`
   - `VKN/TCKN`
   - `Ticari Unvan`

## 7) İlk Konfigürasyon
Admin ayar ekranındaki temel adımlar:
- Aktif provider seçin: `local_only | parasut | logo | mikro`
- Otomasyon bayraklarını belirleyin:
- `InvoiceCreated` sonrası gönderim
- `InvoicePaid` sonrası gönderim
- Firma bilgilerini doldurun:
- Ünvan, VKN, MERSIS, adres, telefon, e-posta, logo, IBAN
- Yasal metni HTML olarak kaydedin

## 8) Provider Konfigürasyonu

### Genel Parametreler
- `provider.api_url`
- `provider.company_id`
- `provider.client_id`
- `provider.client_secret`
- `provider.username`
- `provider.password`

### Provider Seçenekleri
- `local_only`: sadece yerel PDF üretimi, dış sistem gönderimi yok.
- `parasut`: OAuth/token akışı + satış faturası gönderimi.
- `logo`: token/basic auth tabanlı REST gönderimi.
- `mikro`: token tabanlı JSON REST gönderimi.

### Güvenlik
- Hassas bilgiler settings tablosunda şifreli tutulur.
- Provider credential’larını loglara düz metin yazmayın.

## 9) PDF Üretim Kuralları
`src/Pdf/InvoicePdfGenerator.php` aşağıdaki unsurları üretir:
- A4 dikey sayfa
- Türkçe karakter uyumlu font
- Firma başlık bloğu
- Alıcı bilgileri
- Satır bazlı ürün/hizmet tablosu
- KDV kırılımı ve toplamlar
- Tutarın yazıyla karşılığı
- Yasal metin alanı
- Kaşe/imza boşlukları
- QR kod
- Trial watermark

### Çıktı Dizini
- Varsayılan: `storage/pdfs/`
- Dosya adı örneği: `invoice_123_20260218_001501.pdf`

## 10) Lisans Güvenlik Modeli
`LicenseValidator` katmanı:
- Request/response imza doğrulama (HMAC-SHA256)
- `nonce + timestamp` replay koruması
- Cihaz/kurulum fingerprint bağlama
- Runtime bütünlük hash kontrolü
- PBKDF2 ile anahtar türetme
- AES-256-GCM ile şifreli cache
- DB fallback (offline tolerans)

### Lisans Geçersizse
- Hook akışları işlevsel olarak durdurulur.
- Admin tarafta uyarı gösterilir.
- Olay loglanır.

## 11) Hook ve İş Akışı Matrisi
Uygulanan kritik hook’lar:
- `InvoiceCreated`: PDF üret + opsiyonel provider push
- `InvoicePaid`: opsiyonel tekrar push/senkron
- `AdminAreaPageInvoice`: sidebar ve lisans uyarısı
- `DailyCronJob`: lisans check + temizlik işleri
- `AfterCronJob`: status sync için uygun nokta
- `AdminInvoicesControlsOutput`: admin badge/aksiyon alanı
- `InvoiceCancelled`, `InvoiceRefunded`, `InvoiceDeleted`: audit logging
- `AddInvoicePayment`: ödeme hareket logu
- `ClientAdd`, `ClientEdit`: müşteri veri değişimi takibi

## 12) Veritabanı Şeması
- `tbl_resmifatura_settings`
- `tbl_resmifatura_logs`
- `tbl_resmifatura_manuel`
- `tbl_resmifatura_lisans`

Migration dosyası:
- `database/migrations/20260217_create_tables.php`

## 13) Admin Arayüzü
- `templates/admin/config.tpl`
- `templates/admin/invoice_sidebar.tpl`

### Arayüz İşlevleri
- Provider/Firma/Lisans/Yasal metin ayarları
- Manuel ETTN ve resmi no girişi
- Faturadan tek tık PDF üretimi
- Provider’a manuel gönderim
- Durum senkronlama

## 14) API/Lisans Sunucusu Beklentisi
Lisans endpoint’i şu yapıyı desteklemelidir:

Request:
- `payload`:
- `key, ip, domain, instance_id, whmcs_version, timestamp, nonce, fingerprint, integrity_hash`
- `signature`

Response:
- `payload`:
- `valid, message, expiry, max_instances, allowed_ips, allowed_domains, timestamp, nonce, integrity_hash`
- `signature`

Not: Response imzası doğrulanamazsa lisans `invalid` kabul edilir.

## 15) Test ve Kalite

### Test Komutları
```bash
composer test
```

### Test Dosyaları
- `tests/LicenseValidatorTest.php`
- `tests/ProviderFactoryTest.php`
- `tests/InvoicePdfGeneratorTest.php`

### Öneri
- CI’de en az PHP 8.2 + static analysis + unit test koşun.

## 16) Hata Senaryoları ve Troubleshooting

### Sık Görülen Problemler
- Lisans API timeout
- Provider 401/403
- Provider 429 rate limit
- Geçersiz VKN/TCKN formatı
- PDF yazma izni hatası
- QR/merge sırasında kütüphane uyumsuzluğu

### Hızlı Kontrol Adımları
1. `tbl_resmifatura_logs` tablosunu kontrol edin.
2. Lisans tarihini ve domain/IP eşleşmesini doğrulayın.
3. `storage/pdfs` yazma izinlerini kontrol edin.
4. Provider API URL ve credential’ları doğrulayın.
5. Cron çalışmasını doğrulayın.

## 17) Production Checklist
1. Lisans anahtarı ve doğrulama endpoint’i aktif.
2. Provider credentials production bilgileri girildi.
3. Firma bilgileri ve yasal metin tamamlandı.
4. PDF klasör izinleri doğrulandı.
5. Cron job günlük çalışıyor.
6. İlk test faturası `local_only` ile üretildi.
7. Provider’a test gönderimi başarılı.
8. Log retention politikası belirlendi.

## 18) Sürümleme ve Destek
- Sürüm: `1.2.0`
- Teknik notlar: `IMPLEMENTATION_NOTES.md`
- Destek ve yıllık lisans yenileme: satış/destek kanalınız

---

## Lisans ve Hukuki
Bu depodaki kod örnekleri proje gereksinimine göre düzenlenmiştir. Ticari kullanımda yıllık lisans koşulları sözleşme ile belirlenmelidir.
