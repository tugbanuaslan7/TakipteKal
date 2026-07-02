# Takipte Kal

Takipte Kal, bir grup insanın aynı rotada birlikte hareket etmesini sağlayan, gerçek zamanlı takip ve sesli navigasyon odaklı bir mobil uygulamadır. Fikir basit ama ihtiyaç gerçek: bir rehber gruba yol gösterirken, takipçilerin telefona bakıp "şimdi nereye dönüyoruz?" diye sormaması gerekiyor.

Uygulama lider-takipçi mimarisi üzerine kurulu. Lider rotayı başlatır, GPS koordinatları Firebase üzerinden anlık olarak akar, takipçiler ise Mapbox üzerinden adım adım navigasyon ve sesli yönlendirme alır.

Yalova Üniversitesi Bilgisayar Mühendisliği bitirme projesi olarak geliştirilmiştir.

---

## Nasıl Çalışır

Uygulamada iki farklı kullanıcı rolü var.

**Lider** rotayı başlatan kişidir. GPS konumu arka planda Firebase Firestore'a yazılır ve haritada tüm takipçilerin anlık konumu görünür. Gruba üç farklı şekilde katılım sağlanabilir: lider bir katılım kodu üretip paylaşabilir, QR kod oluşturup okutabilir ya da rehberinde kayıtlı kişileri doğrudan gruba ekleyebilir.

**Takipçi** bu yöntemlerden biriyle gruba dahil olur. Katılım kodu varsa ekrana manuel girer, QR kod varsa okuttur. Gruba bağlandıktan sonra liderin rotasını alır, Mapbox Directions API üzerinden kendi cihazında adım adım navigasyon oluşturur ve sesli yönlendirme eşliğinde ilerler. Haritada hem liderin hem de diğer takipçilerin konumu anlık olarak görünür.

Gruba geç katılan biri varsa bootstrap flush mekanizması devreye girer: o ana kadar Firestore'a yazılmış tüm rota noktaları toplu olarak çekilir, takipçi rotanın en başından değil mevcut konuma en yakın noktadan navigasyona devam eder.

---

## Navigasyon Motoru

Uygulamanın en kritik parçası `NavigationGuidanceEngine`. OsmAnd'ın açık kaynak navigasyon algoritmasından ilham alınarak Flutter'a uyarlandı.

Sesli duyurular üç aşamada çalışır:

| Aşama     | Ne zaman tetiklenir                        |
|-----------|--------------------------------------------|
| `PREPARE` | Manevra henüz uzakta, önceden haber verir  |
| `TURN_IN` | Manevra yaklaşıyor                         |
| `TURN_NOW`| Tam o an                                   |

Mesafe eşikleri hıza göre adaptif olarak ölçeklenir — yavaş yürürken ile araçla giderken aynı eşikler işe yaramaz.

Mapbox'ın windowed (pencereli) navigasyon mimarisi korundu. Tek seferlik tam rota çekme yaklaşımına geçilmedi; bu tercih bilinçli ve tasarımın temelinde yatıyor.

---

## Teknolojiler

| Katman               | Teknoloji                  |
|----------------------|----------------------------|
| Mobil Framework      | Flutter / Dart             |
| Harita               | Mapbox Maps SDK            |
| Navigasyon           | Mapbox Directions API      |
| Gerçek Zamanlı Veri  | Firebase Firestore         |
| Kimlik Doğrulama     | Firebase Authentication    |
| Sesli Yönlendirme    | Flutter TTS                |
| Konum                | Geolocator                 |
| Ayarlar              | SharedPreferences          |

---

## Veri Mimarisi

```
routes/
  {routeId}/
    points/              — Liderin GPS koordinatları, anlık olarak yazılır

users/
  {uid}/
    savedRoutes/         — Kullanıcıya ait kayıtlı rota referansları
```

---

## Proje Yapısı

```
lib/
├── main.dart
├── config/
│   └── app_config.dart                    — SharedPreferences ile uygulama ayarları
├── screens/
│   ├── login_screen.dart
│   ├── profile_screen.dart
│   ├── leader_screen.dart
│   ├── follower_screen.dart
│   └── follower_enter_code_screen.dart
├── services/
│   ├── navigation_guidance_engine.dart    — Yönlendirme motoru
│   ├── simulation_service.dart            — GPS simülasyonu
│   └── firestore_service.dart
└── widgets/
```

---

## Kurulum

Flutter ve Dart SDK'nın kurulu olduğunu varsayıyoruz.

```bash
git clone https://github.com/tugbanuraslan/TakipteKal.git
cd takipte-kal
flutter pub get
```

Firebase için `google-services.json` (Android) ve `GoogleService-Info.plist` (iOS) dosyalarını Firebase Console'dan indirip ilgili dizinlere eklemeniz gerekiyor. Mapbox erişim token'ını da `android/local.properties` içine tanımlayın.

```bash
flutter run
```

---

## Simülasyon Modu

Geliştirme sürecinde gerçek bir GPS hareketi olmadan test edebilmek için `SimulationService` kullanılıyor. `AppConfig` üzerinden açılıp kapatılabiliyor. Test sürecinde kullanıldı.

