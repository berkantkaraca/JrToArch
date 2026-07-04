# 📘 ADR-5001: Eğitim — Online Sınav Sistemi için Dağıtık ve Güvenli Mimari Seçimi

### Status

**Accepted** — 2026-07-04

### Context

Eğitim platformumuzda online sınav sistemi, özellikle dönem sonu sınavları, deneme sınavları ve canlı sınav oturumlarında yüksek trafik ve güvenlik gereksinimiyle karşı karşıyadır. Mevcut yapı, sınav süresince öğrenci girişlerini, cevap bloklamasını, zamanlamayı, oturum doğrulamasını ve sonuçlandırma süreçlerini tek bir uygulama içinde yürütmektedir.

Bu yapı şu sorunları yaratmaktadır:

- Sınav saatlerinde sistem yavaşlıyor ve kullanıcı deneyimi bozuluyor.
- Sınav sırasında yoğun erişim nedeniyle servis performansı düşüyor.
- Kimlik doğrulama ve anti-cheat kontrolleri tek noktada yoğunlaşarak kırılgan hale geliyor.
- Sınav verilerinin güvenliği ve denetlenebilirliği artan bir gereklilik haline geliyor.
- Özellikle çok sayıda öğrenci aynı anda sınava girdiğinde, sistemin yüksek erişilebilirlik ve doğruluk seviyesinde çalışması zorunlu hale geliyor.
- Sınav başlangıcında anlık yük patlaması (thundering herd) ve 40.000+ adayın aynı saniyede sisteme yüklenmesi operasyonel bir risk oluşturur.
- Dağıtık mimarilerde ağ bölünmesi (network partition) durumunda veri tutarlılığı ve sistem davranışı için CAP Teoremi çerçevesinde AP/CP tercihleri netleştirilmelidir.

#### Mevcut Durum Metrikleri

| Metric | Mevcut | Hedef |
|--------|--------|-------|
| Eşzamanlı sınav katılımcısı | 12.000 | 40.000+ |
| Sınav giriş P99 latency | 6.8 sn | < 2 sn |
| Sınav sırasında hata oranı | %2.1 | < %0.5 |
| Oturum doğrulama başarısı | %96 | > %99.5 |
| Log tutarlılığı | Kısmi | Tam ve denetlenebilir |
| Güvenlik denetim geçerliliği | Geçmedi | Geçmeli |

#### Kısıtlar

- Öğrenci sayısı sezonluk olarak ani artış gösterebiliyor.
- Sınavın sürekliliği kritik; herhangi bir kesinti, hak kaybına yol açabilir.
- Kimlik doğrulama, yetkilendirme ve oturum güvenliği zorunlu.
- KVKK ve eğitim verisi koruma gereksinimleri uygulanmalı.
- Sınav sistemi, uzaktan erişimli olduğu için DDOS, bot saldırısı ve sahte oturum riskine açık.
- Ekip mevcut olarak monolith yaklaşımına aşina; yeni mimariye geçişte operasyonel yük artacaktır.
- Sınav girişindeki eşzamanlı yük artışı, aynı anda çok sayıda istemcinin aynı kaynaklara erişmesi nedeniyle servis seviyesinde performans düşüşü ve kaynak tükenmesi riski yaratır.

### Decision

Online sınav sistemini, monolith yapının dışında ayrı bir “dağıtık ve güvenli mimari” ile işletmeye karar verdik. Yeni mimari aşağıdaki bileşenlerden oluşacaktır:

1. **Sınav Gateway** — tüm giriş isteklerini karşılayacak, rate limiting ve bot koruması sağlayacak.
2. **Authentication & Session Service** — öğrenci kimlik doğrulama, oturum yönetimi ve token lifecycle işlemlerini yürütecek.
3. **Exam Orchestrator** — sınav akışını koordine edecek; zamanlayıcı, cevap kayıt, durum yönetimi ve devamlılık süreçlerini yönetecek.
4. **Submission Service** — cevapların güvenli şekilde kaydedilmesini ve teslim edilmesini sağlayacak.
5. **Security & Audit Service** — oturum analizi, ip/cihaz kontrolü, olay kayıtları ve denetim loglarını yönetecek.
6. **Event Bus (Kafka)** — servisler arası olay iletişimi için kullanılacak.
7. **Cache Layer (Redis)** — oturum, kısa süreli sınav durumu ve sık erişilen veriler için kullanılacak.

Ek mimari patternler ve güvenlik mekanizmaları şunlardır:

- **Virtual Waiting Room** — sınav başlama anındaki anlık yük patlamasını eritebilmek için Gateway önüne konumlandırılacak, adayların eşzamanlı erişimini kontrollü bir havuza alan mekanizma.
- **Client-Side Resilience** — öğrenci cevaplarının internet kesintilerine karşı tarayıcı tarafında (IndexedDB / LocalStorage) şifreli olarak yedeklenmesi ve bağlantı yeniden kurulduğunda retry edilmesi.
- **Transactional Outbox Pattern / Write-Ahead Logging (WAL)** — Submission Service’in cevap kaybını önlemek için Kafka’ya olay yazma aşamasında uygulayacağı güvenilir kayıt mekanizması.
- **Token Fingerprinting & Heartbeat** — çoklu cihaz girişini engellemek için oturum token’larının cihaz parmak izine bağlanması ve düzenli WebSocket / HTTP heartbeat takibi.

Mimari genel akış:

```text
[Student Browser]
        │
        ▼
   Exam Gateway
        │
        ├── Auth & Session Service
        ├── Exam Orchestrator
        ├── Submission Service
        └── Security & Audit Service
                │
                ▼
           Kafka / Redis
                │
                ▼
         Database / Object Storage
```

Bu yapı sayesinde:

- Sınav giriş ve oturum yönetimi ayrı bir servis olarak çalışır.
- Cevap akışı ve zamanlayıcı ayrı bir modül olarak izole edilir.
- Güvenlik ve denetim hizmetleri merkezi bir yerden yönetilir.
- Yük artışı sırasında servisler bağımsız olarak ölçeklenebilir.
- Virtual Waiting Room ile giriş yoğunluğu kontrollü şekilde yönetilir; Client-Side Resilience ile internet kopukluklarında cevap kaybı azaltılır.

### Alternatives Considered

#### A) Mevcut Monolith Yapıda İyileştirme

- (+) Düşük başlangıç maliyeti.
- (+) Ekip için geçiş kolaydır.
- (–) Sınav saatlerindeki yoğunlukta performans yine darboğaza girebilir.
- (–) Güvenlik ve denetim mekanizmaları tek noktada yoğunlaşır.
- (–) Yüksek ölçekleme gereksiniminde yeterli olmaz.

**Karar:** Reddedildi.

#### B) Tamamen Yeni Microservice Platformu

- (+) Maksimum esneklik.
- (+) Her servis bağımsız ölçeklenebilir.
- (–) Çok yüksek operasyonel karmaşıklık.
- (–) Eğitim ekibi için ilk aşamada ağır gelir.
- (–) Sınav sistemi gibi kritik iş akışında gereğinden fazla karmaşıklık doğurur.

**Karar:** Reddedildi.

#### C) Dağıtık ve Güvenli Mimari (Seçilen Yöntem)

- (+) Yük altında daha dayanıklı olur.
- (+) Güvenlik kontrolleri ayrıştırılmış şekilde uygulanır.
- (+) Sınav akışının izlenebilirliği artar.
- (+) İleride yeni özellikler eklemek daha kolay olur.
- (–) İlk kurulum maliyeti ve operasyonel yük artar.

**Karar:** Seçildi.

### Consequences

#### Pozitif

- Sınav akışı daha güvenli ve daha izlenebilir hale gelir.
- Yoğun saatlerde servisler bağımsız şekilde ölçeklenebilir.
- Olay bazlı mimari sayesinde güvenlik ihlali ve oturum anormallikleri daha hızlı tespit edilir.
- Sınav sonuçları ve audit log'lar daha düzenli ve denetlenebilir olur.
- Gelecekte canlı ders, deneme sınavı ve analitik modülleri bu mimariye eklenebilir.

#### Negatif

- İlk kurulumda daha fazla mühendislik ve altyapı çalışması gerekir.
- Operasyonel karmaşıklık artar; log, monitoring, tracing ve incident response süreçleri güçlenmelidir.
- Ekipte yeni mimari kavramlarına alışma süreci gerekir.
- Dağıtık sistemlerde veri tutarlılığı, eventual consistency ve retry / idempotency mekanizmaları dikkatli yönetilmelidir.

#### Risk Mitigation

- Öncelikle sınav giriş ve oturum yönetimi servislerine odaklanılır.
- Canlı sınavın ilk aşamalarında shadow mode ile yeni mimari test edilir.
- Rate limiting, bot detection ve IP risk analizi öncelikli güvenlik adımlarıdır.
- Her servis için ayrı log ve metric pipeline kurulmalıdır.
- OpenTelemetry, Jaeger / Zipkin ile distributed tracing kurulmalı; böylece geçiş sürecinde gözlemlenebilirlik artırılmalıdır.
- Eventual consistency senaryoları ve idempotent consumer kuralları test edilerek veri kaybı ve tekrar işleme riski azaltılmalıdır.
- Failover senaryoları test edilerek kritik saatlerde sistemin düşmesi azaltılmalıdır.

### Guardrails

- ❌ Sınav sırasında tek bir servis kapanırsa tüm sistem düşürülmeyecek; fallback mekanizması olacaktır.
- ❌ Öğrenci cevapları yalnızca tek bir noktadan değil, doğrulama ve yazma adımlarından geçecektir.
- ❌ Güvenlik logları silinemez veya elle değiştirilemez.
- ❌ Sınav başlatma ve teslim işlemleri idempotent olacaktır.
- ❌ İnterneti kesilen öğrencinin o ana kadar verdiği cevaplar local cache'ten sunucuya başarıyla iletilene kadar sınavı sonlandırılamaz.
- ❌ Aynı kullanıcı hesabı, aynı anda birden fazla aktif cihaz parmak izi (fingerprint) ile sınav oturumu yürütemez.
- ❌ Sınav zamanlayıcısı (Timer) istemci (browser) saatine güvenemez; tüm zaman doğrulamaları güvenli NTP sunucuları ve sunucu imzalı timestamp ile yapılır.
- ❌ Kimlik doğrulama başarısız olan oturumlar doğrudan sınava alınmayacaktır.
- ❌ Bot tespiti başarısız olan isteklerin sınav akışına girmesi engellenecektir.
- ❌ Üretim ortamına geçmeden önce yük testi ve güvenlik testi zorunlu olacaktır.

### Evidence & Review

**Review tarihi:** 2026-09-01

| SLI | Hedef | Ölçüm Yöntemi |
|-----|-------|---------------|
| Sınav giriş P99 latency | < 2 sn | Distributed tracing |
| Sınav hata oranı | < %0.5 | Application metrics |
| Oturum doğrulama başarı oranı | > %99.5 | Auth service logs |
| Güvenlik olay tespiti | < 1 dk | SIEM / audit pipeline |
| Sistem erişilebilirliği | %99.9+ | Uptime monitoring |

**Otomatik geri alma kriteri:**
- Sınav giriş latency 5 dakikada 3 sn'yi geçerse sistem otomatik olarak canary düşürülür.
- Güvenlik olayları belirli bir eşik aşıyorsa ilgili sınav oturumu durdurulur.

### Date / Owner

- **Date:** 2026-07-04
- **Owner:** Architect - Berkant Karaca
- **Reviewers:** Product Team, Security Team, SRE Lead
- **Stakeholders:** CTO, Head of Education Products, Compliance
