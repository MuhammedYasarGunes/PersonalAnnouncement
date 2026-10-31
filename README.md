# Duyuru & Halka Arz Takip

İki kaynağı saatte bir kontrol edip yeni içerik bulduğunda telefonunuza push
bildirim gönderen bir sistem:

- **Fırat Üniversitesi Bilgisayar Müh.** duyuru sayfası
  (`bilgisayarmf.firat.edu.tr/announcements-all`)
- **halkarz.com** halka arz listesi

## Mimari: 0 maliyet, kredi kartı gerekmez

```
[ GitHub Actions ]  --saatte bir çalışır, sunucu YOK-->  iki siteyi tarar
        |                                                        |
        | diff için bu repodaki data/state.json'u kullanır       |
        | (kendi commit'iyle günceller)                          |
        v                                                        v
[ Firebase Firestore ]  <--cihaz token'ı yazar--  [ Mobil uygulama ]
        |
        | yeni içerik varsa Firebase Cloud Messaging (FCM)
        v
   telefonunuza push bildirim
```

Hiçbir yerde 7/24 açık tutmanız gereken bir sunucu yok:

- **GitHub Actions**: zamanlanmış görevleri (cron) ücretsiz çalıştırır.
  Public repo'larda sınırsız, private repo'larda ayda 2000 dakika ücretsiz
  — bu iş ayda ~700-1000 dakika kullanır, rahat sığar.
- **Firebase**: Cloud Messaging (push bildirim) her zaman ücretsizdir.
  Firestore (sadece cihaz token'larını saklamak için) ücretsiz "Spark"
  planında — **kredi kartı istemez**, günlük 50.000 okuma/20.000 yazma
  limiti var, bu proje için fazlasıyla yeterli.
- **"Veritabanı"**: `data/state.json` dosyasının kendisi — GitHub Actions
  her taramadan sonra bu dosyayı otomatik commit'ler. Mobil uygulama da
  listeleri göstermek için bu dosyayı GitHub'ın ücretsiz "raw" adresinden
  okur, ayrı bir API sunucusu gerekmez.

Toplam maliyet: **0 TL**. Tek gereken: bir GitHub hesabı ve bir Firebase
projesi (ikisi de ücretsiz, kart istemiyor).

> Not: Eskiden hazırladığım `backend/` klasöründeki FastAPI sunucusu hâlâ
> duruyor — kendi VPS'inizde çalıştırmak isterseniz (ör. ileride daha
> gelişmiş özellikler eklemek için) o da kullanılabilir, ama artık
> **gerekli değil**. Bu README aşağıda sadece ücretsiz GitHub Actions
> yolunu anlatıyor.

---

## 1. GitHub reposu oluşturun

1. https://github.com/new adresinden yeni bir repo oluşturun (ör.
   `duyuru-takip`). Public veya private fark etmez (public'te Actions
   dakika limiti yok, önerilir).
2. Bu klasördeki tüm dosyaları (bu zip'in içeriğini) o reponun içine
   kopyalayıp push'layın:
   ```bash
   cd duyuru-bildirim-app
   git init
   git add .
   git commit -m "ilk kurulum"
   git branch -M main
   git remote add origin https://github.com/KULLANICI_ADINIZ/duyuru-takip.git
   git push -u origin main
   ```

---

## 2. Firebase Kurulumu

1. https://console.firebase.google.com adresinden yeni bir proje oluşturun
   (ör. "duyuru-takip"). Google hesabı yeterli, **kredi kartı istemez**
   (varsayılan "Spark" ücretsiz plan).
2. Sol menüden **Build > Firestore Database** açın, "Create database"
   deyip **Production mode** ve size yakın bir bölge (ör. `eur3`) seçin.
3. Firestore güvenlik kurallarını düzenleyin (Firestore > Rules sekmesi),
   aşağıdakini yapıştırıp yayınlayın — bu, sadece cihazların kendi
   token'ını yazmasına izin verir, okumaya kapalıdır (okuma zaten sadece
   GitHub Actions'daki Admin SDK ile yapılıyor, o kurallardan etkilenmez):
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /device_tokens/{token} {
         allow read: if false;
         allow write: if request.resource.data.keys().hasOnly(['platform', 'updated_at']);
       }
     }
   }
   ```
4. **Proje Ayarları (⚙️) > Servis Hesapları (Service accounts)** sekmesine
   gidin, "Generate new private key" ile bir JSON dosyası indirin.
   - Bu dosyanın **tüm içeriğini** kopyalayın (bir sonraki adımda GitHub
     secret'ı olarak kullanacaksınız).
   - **Bu dosyayı asla repoya commit'lemeyin / paylaşmayın.**
5. Android uygulaması eklemek için "Add app > Android" seçin, paket adı
   olarak birazdan kullanacağınız adı girin (ör.
   `com.eksedra.duyurutakip`). İndirilen `google-services.json`'a şimdilik
   gerek yok — `flutterfire configure` (adım 4) bunu otomatik halledecek.

---

## 3. GitHub Actions secret'ını ekleyin

1. GitHub reponuzda **Settings > Secrets and variables > Actions** açın.
2. "New repository secret" deyip:
   - Name: `FIREBASE_SERVICE_ACCOUNT_JSON`
   - Value: adım 2.4'te indirdiğiniz JSON dosyasının **tüm içeriği**
     (`{` ile başlayıp `}` ile biten kısmın tamamı)
3. Kaydedin.

Artık workflow'u elle tetikleyip test edebilirsiniz: GitHub reponuzda
**Actions** sekmesi > "Duyuru ve Halka Arz Kontrolü" > "Run workflow".
Birkaç saniye içinde çalışıp `data/state.json` dosyasını güncelleyerek
commit atmalı (ilk çalıştırmada bildirim gönderilmez, sadece mevcut
kayıtlar kaydedilir — bkz. aşağıdaki "Nasıl çalışıyor" notu).

Sorun olursa Actions sekmesindeki çalıştırma loglarına bakın; en sık
hata Firestore kurallarının henüz yayınlanmamış olması veya secret'ın
yanlış kopyalanmış olmasıdır.

---

## 4. Mobil Uygulama Kurulumu (Flutter)

`mobile/` klasöründe uygulamanın tüm Dart kaynak kodu (`lib/`) hazır.
Android/iOS platform klasörleri (gradle, Xcode projesi gibi Flutter'ın
`flutter create` ile ürettiği, projeye özel dosyalar) bu paketin içinde
**yok** — bunları aşağıdaki adımla siz oluşturacaksınız (tek seferlik,
~1 dakika sürer).

### Ön koşul

Flutter SDK kurulu olmalı: https://docs.flutter.dev/get-started/install
Kurulumu doğrulamak için: `flutter doctor`

### Adımlar

```bash
cd mobile
./setup.sh com.sizin.paket.adi   # ör: ./setup.sh com.eksedra.duyurutakip
```

Bu script `flutter create` ile android/ios klasörlerini oluşturur ve
hazırladığım `lib/` kodunu + `pubspec.yaml`'ı bunların üzerine geri koyar.

Ardından:

```bash
dart pub global activate flutterfire_cli
flutterfire configure
```

Bu komut, 2. adımda oluşturduğunuz Firebase projesini seçmenizi ister ve
`lib/firebase_options.dart` dosyasını gerçek proje bilgilerinizle otomatik
olarak yeniden yazar (elle düzenlemeniz gerekmez).

### GitHub adresinizi girin

`mobile/lib/services/data_service.dart` dosyasını açın:

```dart
static const String stateUrl =
    'https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/YOUR_REPO/main/data/state.json';
```

satırındaki `YOUR_GITHUB_USERNAME/YOUR_REPO` kısmını 1. adımda
oluşturduğunuz gerçek GitHub kullanıcı adı/repo adıyla değiştirin.

### Android minimum sürüm

`mobile/android/app/build.gradle` içinde `minSdkVersion`'ın en az 21
olduğundan emin olun (Firebase gereksinimi; genelde varsayılan zaten
yeterlidir).

### Test edin

Bir Android cihaz/emülatör bağlıyken:

```bash
flutter run
```

Uygulama açılınca FCM token'ını otomatik olarak Firestore'a kaydeder.
GitHub Actions'da "Run workflow" ile taramayı elle tetikleyip (veya bir
sonraki saatlik çalışmayı bekleyip) yeni bir kayıt bulunduğunda
telefonunuza bildirim gelmeli.

### APK oluşturma (kendi telefonunuza kurmak için)

```bash
flutter build apk --release
```

Çıktı: `build/app/outputs/flutter-apk/app-release.apk` — bu dosyayı
telefonunuza aktarıp kurabilirsiniz (Android "bilinmeyen kaynaklardan
yükleme" iznini açmanız gerekebilir). Play Store'a yayınlamak isterseniz
`flutter build appbundle --release` ile `.aab` oluşturup Play Console'a
yükleyin (imzalama anahtarı oluşturmanız gerekir — Flutter dokümantasyonu:
https://docs.flutter.dev/deployment/android).

iOS için `flutter build ios --release` sonrası Xcode üzerinden
imzalama/TestFlight/App Store süreci izlenir (Apple Developer hesabı
gerektirir — bu tek ücretli/opsiyonel kısım, sadece App Store'a
yayınlamak isterseniz gerekir; kendi iPhone'unuza kurmak için
gerekmeyebilir).

---

## Nasıl çalışıyor (özet)

1. Her saat başı GitHub Actions `scripts/check_and_notify.py`'yi çalıştırır.
2. Script iki siteyi tarar, `data/state.json`'daki önceki kayıtlarla
   karşılaştırır.
3. Yeni kayıt varsa: Firestore'daki `device_tokens` koleksiyonundan tüm
   cihaz token'larını okur, her birine FCM ile push bildirim gönderir.
4. `data/state.json` güncellenip repoya commit'lenir — bir sonraki
   çalıştırmada aynı kayıtlar tekrar "yeni" sayılmaz.
5. **İlk çalıştırma özel durumdur**: state dosyası boşsa, mevcut tüm
   kayıtlar bildirim gönderilmeden kaydedilir (yoksa geçmişteki onlarca
   duyuru/halka arz için anlık bildirim yağmuru olurdu).
6. Mobil uygulama açıldığında FCM token'ını Firestore'a yazar; liste
   ekranı için `data/state.json`'u GitHub'ın ücretsiz raw-content
   adresinden okur.

## Site yapısı değişirse

`scripts/scraper.py` içindeki iki fonksiyonu (`fetch_firat_announcements`,
`fetch_halkarz_ipos`) güncel HTML yapısına göre güncellemeniz yeterli.
Yerelde test etmek için:

```bash
cd scripts
pip install -r requirements.txt
python scraper.py
```

Her iki kaynaktan da başlık/tarih/link listesi görmelisiniz.

## Sınırlamalar

- GitHub Actions'ın zamanlanmış görevleri yoğunlukta birkaç dakika
  gecikebilir — "tam saat başı" garantisi yoktur ama pratikte yeterince
  yakındır.
- Bir repoda **60 gün boyunca hiç commit olmazsa** GitHub zamanlanmış
  workflow'u otomatik devre dışı bırakabilir. Script her çalıştığında
  `last_checked` alanını güncelleyip commit attığından bu normalde
  sorun olmaz; yine de ara sıra Actions sekmesinden görevin hâlâ aktif
  olduğunu kontrol etmenizde fayda var.
- Firestore güvenlik kuralları herkesin `device_tokens` koleksiyonuna
  yazabilmesine izin veriyor (okuma kapalı). Kişisel kullanım için
  yeterlidir; uygulamayı çok sayıda kişiye dağıtacaksanız daha sıkı bir
  doğrulama (App Check gibi) eklemeyi düşünebilirsiniz.
# PersonalAnnouncement
