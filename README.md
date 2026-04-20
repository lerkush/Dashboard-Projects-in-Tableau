# Dashboard-Projects-in-Tableau
All of projects in this repository are created by using Tableau. Also they are available on https://public.tableau.com/app/profile/lerkush/vizzes

# Docker Workshop 2026 — Eğitimci Scripti (Oturum 2)

> **Format:** Online, ekran paylaşımı
> **Süre:** 60 dakika
> **Ön Hazırlık:** Eğitimden 10 dk önce `docker compose up -d` çalıştır,
> `docker compose ps` ile 4 servisin healthy olduğunu doğrula.

---

## Zaman Çizelgesi

| Bölüm | Konu | Süre | Kümülatif |
|-------|------|------|-----------|
| 0 | Açılış ve Recap | 5 dk | 05:00 |
| 1 | Docker Volumes | 12 dk | 17:00 |
| 2 | Docker Networking | 10 dk | 27:00 |
| 3 | Docker Compose | 25 dk | 52:00 |
| 4 | Best Practices | 5 dk | 57:00 |
| 5 | Kapanış ve Q&A | 3 dk | 60:00 |

---

---

## BÖLÜM 0 — Açılış ve Recap `[00:00 → 05:00]`

*Slayt 1 — Kapak*

> "Merhaba herkese. Docker eğitimimizin ikinci oturumuna hoş geldiniz. Geçen hafta güçlü bir temel attık. Bugün o temelin üstüne gerçek hayata yakın bir mimari inşa edeceğiz."

Terminalde `docker version` çalıştır — ortamın hazır olduğunu göster.

---

*Slayt 2 — Program*

> "Bugün üç ana kavram var: Volumes, Networking ve Docker Compose. Ama bunlar birbirinden kopuk konular değil. Birinin üstüne diğerini inşa edeceğiz ve sonunda dört servisli bir stack çalıştıracağız."

---

*Slayt 3 — Geçen Hafta*

> "Geçen haftayı hızlıca hatırlayalım. Image'ın bir blueprint olduğunu, container'ın bu blueprint'ten çalışan bir süreç olduğunu gördük. Dockerfile ile kendi image'larımızı tarif ettik."

Bir an dur, soru sor:

> "Şimdi şunu düşünelim: PostgreSQL çalıştırıyorsunuz, kullanıcı tablosuna kayıtlar yazıyorsunuz. Sonra o container'ı siliyorsunuz. Veriye ne oluyor?"

3 saniye bekle.

> "Tamamen gidiyor. Container silindiğinde yazılabilir katmanı da silinir. Bu bir hata değil, tasarım gereği. Container'lar stateless olmak zorunda — bu onların ölçeklenebilirliğinin kaynağı. Ama uygulama verisi? Onun kalması lazım. İşte bugünün ilk konusu tam olarak bu."

---

---

## BÖLÜM 1 — Docker Volumes `[05:00 → 17:00]`

*Slayt 4 — Problem*

> "Önce sorunu canlı gösterelim."

Terminale geç:

```bash
docker run --name demo-db \
  -e POSTGRES_PASSWORD=secret \
  -d postgres:17-alpine

docker exec -it demo-db psql -U postgres \
  -c "CREATE TABLE kullanicilar (id serial, isim text);"

docker exec -it demo-db psql -U postgres \
  -c "INSERT INTO kullanicilar(isim) VALUES ('Ayse'), ('Mehmet');"

docker exec -it demo-db psql -U postgres \
  -c "SELECT * FROM kullanicilar;"
```

> "Güzel. İki kayıt var. Şimdi container'ı silelim."

```bash
docker rm -f demo-db
```

> "Aynı image'dan yeni bir container:"

```bash
docker run --name demo-db2 \
  -e POSTGRES_PASSWORD=secret \
  -d postgres:17-alpine

docker exec -it demo-db2 psql -U postgres \
  -c "SELECT * FROM kullanicilar;"
```

Hata gelecek: `relation "kullanicilar" does not exist`

> "Yok. Container'ın yazılabilir katmanı geçiciydi. Silindi. İşte volumes'un çözdüğü problem bu."

Temizle:

```bash
docker rm -f demo-db2
```

---

*Slayt 5 — Volume Tipleri*

> "Docker'ın üç farklı veri kalıcılık mekanizması var. Birincisi ve production'da kullandığımız: named volume. Docker'ın host'ta kendi yönettiği kalıcı alan. Container'dan tamamen bağımsız."

> "İkincisi bind mount: host'unuzdaki bir dizini container'a doğrudan bağlarsınız. Geliştirme ortamında çok kullanılır — kodu değiştirince container'ı yeniden build etmek zorunda kalmazsınız, dosya anında görünür."

> "Üçüncüsü tmpfs: RAM'de tutulur. Container kapanınca uçup gider. Çok hızlıdır. Session token gibi geçici hassas veriler için idealdir."

---

*Slayt 6 — Komutlar / Terminal*

Named volume demosu:

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
# Mountpoint: /var/lib/docker/volumes/pgdata/_data
```

> "'Mountpoint' alanına bakın. Docker bu volume'u host'ta bu konumda tutuyor. Siz bu dizine doğrudan müdahale etmiyorsunuz — Docker yönetiyor."

```bash
docker run --name kalici-db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:17-alpine

docker exec -it kalici-db psql -U postgres \
  -c "CREATE TABLE ayarlar (anahtar text, deger text);"

docker exec -it kalici-db psql -U postgres \
  -c "INSERT INTO ayarlar VALUES ('tema','koyu'),('dil','tr');"

docker rm -f kalici-db

# Aynı volume ile yeni container
docker run --name kalici-db2 \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:17-alpine

docker exec -it kalici-db2 psql -U postgres \
  -c "SELECT * FROM ayarlar;"
```

> "Veri orada. Container gitti, volume kaldı. İkisi arasında hiçbir doğrudan bağlantı yok — tek ortakları aynı volume'u kullanmaları."

Temizle:

```bash
docker rm -f kalici-db2
docker volume rm pgdata
```

Geçiş cümlesi:

> "Volumes'u çözdük. Şimdi birden fazla container çalışırken bunlar nasıl birbirleriyle konuşuyor? Bir container başka bir container'ı nasıl buluyor?"

---

---

## BÖLÜM 2 — Docker Networking `[17:00 → 27:00]`

*Slayt 7 — Neden Networking?*

> "Gerçek uygulamalar tek container'dan oluşmuyor. API ayrı, veritabanı ayrı, cache ayrı. Bu servislerin birbiriyle güvenli ve güvenilir iletişim kurması gerekiyor. Ama container'lar birbirini nasıl buluyor?"

---

*Slayt 8 — Network Driver'ları*

> "Docker dört farklı network driver'ı destekliyor. Ama günlük hayatta neredeyse hep 'custom bridge' kullanıyoruz."

> "Neden varsayılan bridge yetmiyor? Çünkü varsayılan bridge'de Docker DNS yok. Container'lar birbirini IP adresiyle bulabilir ama IP değişkendir. Custom network'te Docker otomatik DNS sağlar — servis adını yazarsınız, Docker IP'yi çözümler."

Terminal'de farkı göster:

```bash
# Varsayılan bridge — isimle ping çalışmaz
docker run -d --name servis-a alpine sleep 3600
docker run -d --name servis-b alpine sleep 3600

docker exec servis-a ping -c 2 servis-b
# ping: bad address 'servis-b'
```

> "Çalışmadı. Şimdi custom network:"

```bash
docker network create workshop-net

docker run -d --name api-srv   --network workshop-net alpine sleep 3600
docker run -d --name redis-srv --network workshop-net alpine sleep 3600

docker exec api-srv ping -c 3 redis-srv
```

> "Çalıştı. 'redis-srv' ismini yazdım, IP yazmadım. Docker custom network'te servis adını otomatik çözümledi. Docker Compose'un arka planda yaptığı da tam olarak bu."

Temizle:

```bash
docker rm -f servis-a servis-b api-srv redis-srv
docker network rm workshop-net
```

Geçiş cümlesi:

> "Volumes ve networking'i anladık. Şimdi bu iki kavramı birleştirip tüm stack'i tek bir dosyadan yönetmenin yoluna bakacağız."

---

---

## BÖLÜM 3 — Docker Compose `[27:00 → 52:00]`

*Slayt 9 — Compose Nedir?*

> "Docker Compose, birden fazla container'ı YAML formatında tanımlamanızı ve tek komutla yönetmenizi sağlayan araç. Infrastructure as Code — altyapınız artık kod, versiyon kontrolünde, code review'a açık."

> "Compose olmadan: dört ayrı docker run komutu, her birinin network ve volume parametrelerini elle belirtmek, sırayı doğru ayarlamak. Compose bunu bir kez tanımlayıp tekrar kullanılabilir hale getiriyor."

---

*Slayt 10 — YAML Anatomisi*

> "docker-compose.yml'in temel yapısına bakalım."

```bash
cat demo/docker-compose.yml
```

Dosyayı ekranda açıp bölüm bölüm anlat:

> "'services' altında her container bir servis olarak tanımlanır. Servis adı aynı zamanda Docker DNS adı — başka bir container bu isme 'postgresql://db:5432' gibi bağlanabilir."

> "'env_file: .env' satırı kritik. Şifreler YAML'a yazılmaz, .env dosyasından okunur. Bu dosya .gitignore'da olacak — birazdan neden önemli olduğunu göstereceğiz."

> "'depends_on' ile startup sırasını kontrol ediyoruz. Ama dikkat — sadece 'condition: service_healthy' kullanırsak güvenli. Bunu bir slide'da ayrıca işleyeceğiz."

---

*Slayt 11 — Demo Proje Mimarisi*

> "Demo projemizde dört servis var. Her birinin burada olmasının bir nedeni var:"

> "**Nginx**: Gerçek hayatta API'yi internete doğrudan açmazsınız. Nginx rate limiting uygular, güvenlik başlıklarını ekler, isteği arkaya iletir. Egitimde 'api:8000' satırını göstereceğiz — IP değil, servis adı."

> "**FastAPI**: İş mantığı burada çalışıyor. Not okurken önce Redis'e bakıyor, cache miss'te PostgreSQL'den çekiyor ve Redis'e yazıyor. Bu akışı demoda X-Response-Time-Ms header'ı ile somut göstereceğiz."

> "**PostgreSQL**: Volume kalıcılığını göstereceğimiz servis. notes_db named volume'u kullanıyor. Container silinse bile veri diskte kalıyor."

> "**Redis**: Cache katmanı. İlk istekte DB'den geliyor, ikinci istekte Redis'ten. Hit rate'i /cache/info endpoint'i ile canlı takip edeceğiz."

> "Güvenlik açısından da dikkat edin: frontend network sadece nginx ile API arasında. Backend network 'internal: true' — dış dünya doğrudan db ya da redis'e erişemiyor. Tek giriş noktası nginx."

---

*Slayt 12 — depends_on + healthcheck*

> "Production'da en sık karşılaşılan hatalardan birine gelelim."

> "Senaryo: docker compose up yapıyorsunuz. PostgreSQL container'ı başlıyor ama bağlantı kabul etmesi birkaç saniye alıyor. Bu sürede API bağlanmaya çalışıyor — 'connection refused' hatası alıyor ve crash ediyor."

> "Çözüm: sadece 'depends_on: - db' yazmak yetmez. 'condition: service_healthy' kullanın. Bu, healthcheck komutunun başarılı dönmesini bekler. PostgreSQL için pg_isready mükemmel bir healthcheck."

```bash
# Healthcheck durumlarını göster
docker compose ps
# Status: running (healthy) görülmeli
```

> "'running (healthy)' görüyorsunuz. 'Healthy' ibaresi healthcheck'in geçtiğini gösteriyor. Sadece 'running' olsaydı healthcheck tanımlı değil demekti."

---

### Canlı Demo `[32:00 → 50:00]`

*Slayt 13 — Demo Başlatma*

> "Hadi çalıştıralım. Demo klasörüne gelin."

```bash
cd demo/
cat .env              # Yapılandırmayı göster
```

> "Şifrelerin YAML'da değil .env'de olduğuna dikkat edin. .gitignore'da da var — şifreler hiçbir zaman GitHub'a gitmeyecek."

```bash
docker compose up -d
docker compose logs -f
```

> "İlk çalıştırmada API image'ı build ediliyor, diğerleri Hub'dan çekiliyor. Bir dakika sürebilir — bu normal. Logda pg_isready'nin geçtiği anı izleyin — o anda API başlıyor."

```bash
# Ctrl+C ile logdan çık, sonra:
docker compose ps
```

> "4 servis, hepsi 'running (healthy)'. Sağlık kontrolü:"

```bash
curl http://localhost/health | python3 -m json.tool
```

> "postgresql:ok ve redis:ok görüyorsunuz. API her iki alt sisteme ulaşabiliyor."

---

#### Cache Demo — Egitimin En Çarpıcı Anı 1

```bash
# 1. istek — cache miss, DB'den gelir
curl -i http://localhost/notes/1 2>&1 | grep -E "X-Response|source"
```

> "X-Response-Time-Ms: ~20 ve source:'db'. Redis'te yoktu, PostgreSQL'den geldi, Redis'e yazıldı."

```bash
# 2. istek — cache hit, Redis'ten gelir
curl -i http://localhost/notes/1 2>&1 | grep -E "X-Response|source"
```

> "X-Response-Time-Ms: ~2 ve source:'cache'. 10 kat daha hızlı. Redis'ten geldi, PostgreSQL'e hiç gidilmedi."

```bash
# Cache istatistikleri
curl http://localhost/cache/info | python3 -m json.tool
```

> "Hit rate görüyorsunuz. Production'da bu oran %85 üzerinde olmalı — veritabanı yükünü ciddi ölçüde düşürür."

---

#### Volume Demo — Egitimin En Çarpıcı Anı 2

```bash
# Kendi notunuzu ekleyin
curl -X POST http://localhost/notes \
  -H "Content-Type: application/json" \
  -d '{"title":"Volume Kaniti","content":"Bu not kalici mi?","author":"Demo"}'

# Listeyi doğrula
curl http://localhost/notes | python3 -m json.tool
```

> "Şimdi tüm container'ları silelim."

```bash
docker compose down
docker compose ps    # Boş
```

```bash
docker volume ls
# notes_db hala listede!
```

> "Container'lar gitti. Ama volume'a bakın — notes_db hala diskte. Volume container'dan bağımsız yaşıyor."

```bash
docker compose up -d
sleep 8

curl http://localhost/notes | python3 -m json.tool
```

> "Az önce eklediğiniz not orada. Container silinip yeniden oluşturuldu, ama volume aynı veriye sahip. İşte production'da veri kalıcılığı böyle sağlanır."

---

#### Compose Komut Turu

*Slayt 16 — Komut Referansı*

```bash
# Servis yeniden başlatma
docker compose restart api

# Sadece API logları
docker compose logs -f api

# API container'ına shell
docker compose exec api bash
  # İçeriden: curl http://db:5432 — DB'ye isim ile erişim
  exit

# DB'ye direkt bağlantı
docker compose exec db psql -U workshop notes \
  -c "SELECT title, author FROM notes;"

# Kaynak kullanımı
docker compose stats --no-stream
```

---

---

## BÖLÜM 4 — Best Practices `[52:00 → 57:00]`

*Slayt 17 — Best Practices*

> "Son beş dakikada production'da mutlaka uygulamanız gereken pratikleri geçelim. Bunlar gerçek projelerden, gerçek hatalardan öğrenilen şeyler."

**1. .env hiçbir zaman Git'e gitmez**

> "2026'da GitHub'ta her gün production şifresi sızdırılıyor. Sebebi basit: biri yanlışlıkla .env'i commit ediyor. .gitignore'a '.env' satırını ekleyin, .env.example'ı commit edin. Production'da GCP Secret Manager veya AWS Secrets Manager kullanın."

**2. Multi-stage Dockerfile**

> "Demo projemizdeki Dockerfile'a bakın — iki aşama var. 'deps' aşamasında gcc ve build araçları var. 'runtime' aşamasında sadece çalışma zamanı. Final image ~3 kat küçük, saldırı yüzeyi çok dar. 2026'da tek aşamalı Dockerfile yazmak kabul görmez."

**3. Non-root user**

> "Container içinde root çalıştırmak güvenlik açığı. Dockerfile'da adduser ile kısıtlı kullanıcı oluşturduk, USER komutuyla geçtik. Bir güvenlik ihlalinde saldırgan root yerine kısıtlı kullanıcı olarak çalışır."

**4. Image tag'ini sabitle**

> "'postgres:latest' yazmayın. Bugün latest PostgreSQL 17, altı ay sonra 18 olabilir ve uygulamanız patlar. 'postgres:17-alpine' yazın. Dependabot veya Renovate ile güncellemeleri otomatik takip edin."

**5. Healthcheck**

> "Her production servisine healthcheck tanımlayın. 'Process çalışıyor' yeterli değil. pg_isready, redis-cli ping gibi anlamlı kontroller yazın. Bunu yapmadan depends_on condition'ı çalışmaz."

**6. internal: true network**

> "Veritabanı ve cache servisleri dışarıya açık olmamalı. Backend network'ü internal:true yapın. Dışarıdan sadece nginx görünsün. Veritabanına doğrudan erişim production'da felakete yol açar."

---

---

## BÖLÜM 5 — Kapanış `[57:00 → 60:00]`

*Slayt 18 — Kapanış*

> "Bugün şunları öğrendik:"

> "Container'lar stateless. Named volume ile PostgreSQL verisi container'dan bağımsız diskte yaşıyor. Container silin, volume kalsın."

> "Custom network'te Docker otomatik DNS sağlıyor. 'db', 'redis' gibi servis adlarını yazıyoruz, IP adresine ihtiyaç yok."

> "Docker Compose tüm stack'i YAML'da tanımla, tek komutla yönet. Versiyon kontrolünde, code review'a açık altyapı."

> "internal:true ile backend ağını dış dünyaya kapattık. Tek giriş noktası nginx."

> "depends_on + healthcheck ile startup sırasını ve hazırlığı güvence altına aldık."

> "Eğer üçüncü oturum yapacak olursak, doğal adım şu: GitHub Actions ile bu stack'i otomatik deploy etmek. Kod push edildiğinde Actions image build eder, Container Registry'e push eder, Cloud Run'a deploy eder. Bu projede tam olarak bunu yapıyoruz."

---

## Troubleshooting Notları

| Belirti | Olası Sebep | Çözüm |
|---------|-------------|-------|
| Port 80 meşgul | Başka bir servis kullanıyor | `lsof -i :80` ile bul, NGINX_PORT değiştir |
| api container unhealthy kalıyor | DB henüz hazır değil | `docker compose logs db` — pg_isready geçiyor mu? |
| nginx başlamıyor | api healthy olmadı | depends_on zinciri: redis/db → api → nginx |
| Cache istatistikleri sıfır | İstek atılmadı | Önce GET /notes/{id} at, sonra /cache/info |
| Volume silinmedi | `down` volume korur | Silmek için `down -v` (dikkat: veri gider!) |
| Image değişikliği görünmüyor | Cache kullanıldı | `docker compose up -d --build` |
| `no space left on device` | Docker disk doldu | `docker system prune -f` |
| Container ismiyle ping çalışmıyor | Aynı network'te değil | `docker network inspect` ile kontrol et |

---

## Eğitim Sonrası Temizlik

```bash
# Stack'i durdur, veri koru
docker compose down

# Her şeyi tamamen sil (eğitim bitince)
docker compose down -v
docker system prune -f
docker volume prune -f
```
