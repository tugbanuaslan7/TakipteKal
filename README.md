# Takipte Kal — Dinamik Rota Takipli Mobil Navigasyon Sistemi

Lider bir sürücünün gerçek zamanlı olarak oluşturduğu güzergâhı, takipçilere anlık konum paylaşımı ve sesli navigasyonla aktaran bir Flutter mobil uygulaması.

## İçindekiler

- [Proje Hakkında](#proje-hakkında)
- [Özellikler](#özellikler)
- [Mimari](#mimari)
- [Kullanılan Teknolojiler](#kullanılan-teknolojiler)
- [Sistem Akışı](#sistem-akışı)
- [Veritabanı Yapısı](#veritabanı-yapısı)
- [Sesli Navigasyon Motoru](#sesli-navigasyon-motoru)
- [Kurulum](#kurulum)

## Proje Hakkında

Grup hâlinde yapılan yolculuklarda (örneğin konvoy sürüşü, motosiklet grupları, aile/arkadaş seyahatleri) sık karşılaşılan bir ihtiyaçtan yola çıkılmıştır: lider bir sürücü navigasyondan bağımsız olarak kendi rotasını oluştururken, takipçi araçların bu rotayı gerçek zamanlı olarak takip edip sesli yönlendirme alabilmesi.

Mevcut uygulamalar bu ihtiyacı tam karşılamamaktadır:
- **Navigasyon uygulamaları** (Google Maps vb.) yalnızca sabit bir hedefe rota üretir.
- **Konum paylaşım uygulamaları** (Life360 vb.) yalnızca anlık konumu gösterir, navigasyon desteği sunmaz.
- **Grup sürüş uygulamaları** (REVER, RISER, Strava vb.) grup konumu ve bazı durumlarda navigasyon sunar, ancak liderin *anlık oluşturduğu* rotayı takipçiye gerçek zamanlı aktarmaz.

Bu proje, liderin ürettiği rotayı canlı olarak takipçiye ileten ve bu rotayı sesli navigasyona dönüştüren bütünleşik bir sistem sunar.

## Özellikler

- **Gerçek zamanlı rota paylaşımı** — Lider hareket ettikçe ürettiği konum noktaları anlık olarak takipçilere iletilir.
- **Sesli navigasyon** — Türkçe, üç aşamalı (hazırlık / yaklaşma / son uyarı) hıza duyarlı sesli yönlendirme.
- **Çoklu katılım yöntemi** — Oturum kodu, QR kod veya rehber üzerinden davet ile oturuma katılım.
- **Oturum yönetimi** — Lider tarafında canlı rota, hız/süre/mesafe bilgisi ve takipçi durumu takibi.
- **Rota kaydetme** — Takip edilen rotaların kaydedilip daha sonra yeniden kullanılabilmesi.
- **GPS drift filtreleme** — Sahte konum sıçramalarının otomatik olarak ayıklanması.
- **Telefon doğrulamalı kayıt** — OTP tabanlı hesap doğrulama ve rehber üzerinden kullanıcı eşleştirme.

## Mimari

Sistem üç ana bileşenden oluşur:

```
Lider Cihaz  →  Cloud Firestore  →  Takipçi Cihaz
 (GPS + Firestore)   (merkezi         (Veri katmanı + Navigasyon katmanı)
                      haberleşme)
```

**Lider tarafı:** GPS noktalarını üretir, drift filtresinden geçirir ve Firestore'a yazar.

**Firestore:** Lider ve takipçi arasındaki tüm iletişim buradan geçer; taraflar birbiriyle doğrudan konuşmaz.

**Takipçi tarafı** iki alt katmana ayrılır:
- *Veri katmanı* (`FollowerService`) — Firestore'u dinler, geçmiş ve canlı noktaları senkronize eder, tamponlar.
- *Navigasyon katmanı* (`MapboxService` + `NavigationGuidanceEngine`) — koordinatları manevralara çevirir, filtreler ve sesli uyarı üretir.

`FollowerScreen` bu iki katmanı orkestra ederek haritayı, kamerayı ve TTS'i yönetir.

## Kullanılan Teknolojiler

| Katman | Teknoloji |
|---|---|
| Mobil geliştirme | Flutter, Dart |
| Kimlik doğrulama | Firebase Auth (telefon OTP + e-posta) |
| Veritabanı | Cloud Firestore (NoSQL) |
| Harita görselleştirme | Google Maps SDK |
| Rota ve manevra üretimi | Mapbox Directions API |
| Sesli yönlendirme | Flutter TTS (tr-TR) |
| Konum servisleri | Geolocator |

## Sistem Akışı

1. **Oturum başlatma** — Lider yeni bir takip oturumu başlatır; bu işlem veritabanında bir rota belgesi oluşturur ve bir oturum kodu üretir.
2. **Konum üretimi** — Lider hareket ettikçe ürettiği konum noktaları (enlem, boylam, hız, yönelim, sunucu zaman damgası) GPS drift filtresinden geçirilerek Firestore'a yazılır.
3. **Gerçek zamanlı dinleme** — Takipçi, Firestore'un snapshot mekanizmasıyla yeni noktaları anlık olarak alır; noktalar tamponlanıp gruplar hâlinde görünür rotaya eklenir.
4. **Manevra üretimi** — Takipçinin konumuna göre güzergâhın ilgili bölümü (en fazla 25 nokta, kayan pencere yaklaşımıyla) Mapbox Directions API'ye gönderilir.
5. **Standartlaştırma ve filtreleme** — Mapbox'ın döndürdüğü ham manevra türleri, uygulamanın kendi tanımladığı standart manevra kümesine eşlenir; gereksiz/tekrarlı adımlar ayıklanır.
6. **Sesli yönlendirme** — Filtrelenen manevralar, hıza duyarlı üç aşamalı bir sistemle (hazırlık → yaklaşma → son uyarı) Türkçe olarak seslendirilir.
7. **Oturum sonu** — Lider oturumu sonlandırdığında rota belgesi pasif hâle gelir; takipçinin izlediği güzergâh isteğe bağlı olarak kalıcı biçimde kaydedilebilir.

## Veritabanı Yapısı

Cloud Firestore üzerinde 5 koleksiyon bulunur:

| Koleksiyon | Açıklama |
|---|---|
| `users` | Sisteme kayıtlı kullanıcı bilgileri |
| `rotalar` | Lider kullanıcıların oluşturduğu oturumlar |
| `rotalar/{id}/rota_noktalari` | Oturuma ait konum noktaları (alt koleksiyon) |
| `rotalar/{id}/katilimcilar` | Oturuma katılan takipçiler (alt koleksiyon) |
| `saved_routes` | Kullanıcıların kalıcı olarak kaydettiği rotalar (kök koleksiyon) |

Tasarım ilkesi: oturuma bağlı, geçici veri alt koleksiyonlarda; uzun ömürlü kullanıcı verisi kök koleksiyonlarda tutulur.

## Sesli Navigasyon Motoru

`NavigationGuidanceEngine`, OsmAnd'ın açık kaynaklı sesli yönlendirme algoritmasından uyarlanmıştır ([referans](https://osmand.net/docs/technical/algorithms/voice-prompt-triggering/)).

Üç aşamalı uyarı sistemi:

| Aşama | Ne zaman | Süre katsayısı |
|---|---|---|
| Hazırlık (Prepare) | Manevradan çok önce, yalnızca büyük manevralarda | 115 sn × varsayılan hız |
| Yaklaşma (Advance) | Manevraya yakın mesafede | 22 sn × gerçek hız |
| Son uyarı (Near) | Tam manevra anında, mesafesiz | Dinamik (hız ve adım uzunluğuna göre) |

Uyarı mesafeleri sabit değildir; sürücünün anlık hızına göre ölçeklenir. Ayrıca aynı uyarının tekrar edilmesini önleyen (stage-lock, cooldown, GPS geri sıçraması tespiti) çeşitli denetimler bulunur.

## Kurulum

```bash
# Depoyu klonlayın
git clone <repo-url>
cd takipte-kal

# Bağımlılıkları yükleyin
flutter pub get

# Firebase yapılandırmasını ekleyin
# (google-services.json / GoogleService-Info.plist)

# Mapbox erişim anahtarınızı ekleyin
# (lib/config/ dizininde ilgili yapılandırma dosyasına)

# Uygulamayı çalıştırın
flutter run
```


**Geliştirici:** Tuğba Nur Aslan
**Danışman:** Öğr. Gör. Muhammed Tekin
