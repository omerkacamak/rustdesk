# RustDesk Kod Rehberi (Türkçe)

> Bu rehber, RustDesk açık kaynak kodunu anlamana yardımcı olmak için hazırlandı.
> Sıfırdan başlayıp, uygulamanın nasıl çalıştığını adım adım öğreneceksin.

---

## İçindekiler

1. [Proje Yapısı](#1-proje-yapısı)
2. [Uygulama Başlangıcı (Entry Point)](#2-uygulama-başlangıcı-entry-point)
3. [Flutter ↔ Rust Köprüsü (FFI)](#3-flutter--rust-köprüsü-ffi)
4. [Rendezvous / ID Sistemi](#4-rendezvous--id-sistemi)
5. [Bağlantı Akışı (Connection Flow)](#5-bağlantı-akışı-connection-flow)
6. [Server Tarafı (Kontrol Edilen)](#6-server-tarafı-kontrol-edilen)
7. [Client Tarafı (Kontrol Eden)](#7-client-tarafı-kontrol-eden)
8. [Video & Ekran Paylaşımı](#8-video--ekran-paylaşımı)
9. [Input (Klavye/Fare)](#9-input-klavye-fare)
10. [Flutter UI Katmanı](#10-flutter-ui-katmanı)
11. [Önemli Kütüphaneler (libs/)](#11-önemli-kütüphaneler-libs)
12. [Önerilen Okuma Sırası](#12-önerilen-okuma-sırası)
13. [Sık Kullanılan Kalıplar](#13-sık-kullanılan-kalıplar)

---

## 1. Proje Yapısı

```
rustdesk/
│
├── src/                      ← RUST BACKEND (ana uygulama)
│   ├── main.rs               ← Uygulama giriş noktası
│   ├── core_main.rs          ← Ortak başlangıç (Flutter & Sciter için)
│   ├── lib.rs                ← Kütüphane tanımları
│   ├── flutter_ffi.rs        ← Flutter ↔ Rust FFI köprüsü (çok kritik!)
│   ├── flutter.rs            ← Flutter session yönetimi
│   ├── rendezvous_mediator.rs ← Sunucu ile iletişim (ID, relay, peer bulma)
│   ├── server.rs             ← Server modülü (kontrol edilen taraf)
│   ├── client.rs             ← Client modülü (kontrol eden taraf)
│   ├── common.rs             ← Paylaşılan yardımcı fonksiyonlar
│   ├── server/               ← Server alt modülleri
│   │   ├── connection.rs     ← ★ BAĞLANTI YÖNETİMİ (6000+ satır, en önemlisi)
│   │   ├── video_service.rs  ← Video capture & encode
│   │   ├── audio_service.rs  ← Ses capture & stream
│   │   ├── input_service.rs  ← Gelen klavye/fare girdilerini işleme
│   │   ├── display_service.rs← Ekran listesi, çözünürlük
│   │   ├── clipboard_service.rs ← Pano paylaşımı
│   │   └── service.rs        ← Server servis şablonu
│   ├── client/               ← Client alt modülleri
│   │   ├── io_loop.rs        ← ★ BAĞLANTI DÖNGÜSÜ (client tarafı)
│   │   ├── helper.rs         ← FFI yardımcıları
│   │   ├── file_trait.rs     ← Dosya transferi trait'i
│   │   └── screenshot.rs     ← Client tarafı screenshot
│   ├── platform/             ← Platform-spesifik kod
│   │   ├── windows.rs        ← Windows: DXGI, session, vs.
│   │   ├── linux.rs          ← Linux: X11, Wayland
│   │   └── macos.rs          ← macOS
│   ├── clipboard.rs          ← Pano yönetimi
│   ├── keyboard.rs           ← Tuş haritalama
│   ├── ipc.rs                ← Süreçler arası iletişim
│   └── naming.rs             ← Servis adlandırma
│
├── flutter/                  ← FLUTTER UI (modern arayüz)
│   ├── lib/
│   │   ├── main.dart         ← Flutter girişi
│   │   ├── common.dart       ← Ortak Flutter yardımcıları
│   │   ├── consts.dart       ← Sabitler
│   │   ├── desktop/          ← Masaüstü sayfaları
│   │   │   ├── pages/
│   │   │   │   ├── desktop_home_page.dart  ← Ana sayfa (ID kutusu, liste)
│   │   │   │   ├── desktop_tab_page.dart   ← Tablı arayüz
│   │   │   │   ├── remote_page.dart        ← ★ Remote desktop ekranı
│   │   │   │   ├── connection_page.dart    ← Bağlantı ayarları
│   │   │   │   ├── desktop_setting_page.dart ← Ayarlar
│   │   │   │   ├── file_manager_page.dart  ← Dosya transferi UI
│   │   │   │   ├── terminal_page.dart      ← Terminal UI
│   │   │   │   └── server_page.dart        ← Server ayarları
│   │   │   ├── screen/       ← Multi-window ekranları
│   │   │   └── widgets/      ← Widget'lar
│   │   ├── mobile/           ← Mobil sayfalar
│   │   ├── models/           ← ★ State management (MODEL katmanı)
│   │   │   ├── model.dart    ← ★ ANA MODEL (FFI çağrılarını yönetir)
│   │   │   ├── peer_model.dart    ← Peer listesi
│   │   │   ├── server_model.dart  ← Server ayarları
│   │   │   ├── input_model.dart   ← Girdi işleme
│   │   │   ├── state_model.dart   ← Global state
│   │   │   ├── file_model.dart    ← Dosya transferi
│   │   │   ├── chat_model.dart    ← Sohbet
│   │   │   └── platform_model.dart← Platform FFI çağrıları
│   │   ├── common/           ← Paylaşılan Flutter kodu
│   │   │   ├── hbbs/hbbs.dart          ← Server'a API çağrıları
│   │   │   ├── shared_state.dart       ← Paylaşılan state
│   │   │   └── widgets/               ← Ortak widget'lar
│   │   └── utils/            ← Yardımcı araçlar
│
├── libs/                     ← RUST KÜTÜPHANELERİ
│   ├── hbb_common/           ← ★ Protobuf, config, socket, yardımcılar
│   │   ├── src/
│   │   │   ├── config.rs     ← ★ TÜM AYARLAR (çok önemli!)
│   │   │   ├── protobuf/     ← Protobuf mesaj tanımları
│   │   │   ├── socket_client.rs ← TCP/UDP socket işlemleri
│   │   │   ├── fs.rs         ← Dosya sistemi işlemleri
│   │   │   └── ...
│   │   └── proto/            ← .proto dosyaları
│   ├── scrap/                ← Ekran yakalama (capture)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── windows.rs    ← Windows: DXGI Duplication
│   │   │   ├── linux.rs      ← Linux: X11, Wayland, PipeWire
│   │   │   ├── macos.rs      ← macOS: CGDisplay, SCK
│   │   │   ├── codec/        ← VP8, VP9, H264, H265, AV1 kodlayıcılar
│   │   │   └── record/       ← Kayıt
│   ├── enigo/                ← Klavye/fare simülasyonu
│   └── clipboard/            ← Pano (cross-platform)
│
└── res/                      ← Görseller, ikonlar
```

---

## 2. Uygulama Başlangıcı (Entry Point)

### `src/main.rs`

Uygulamanın başladığı yer. `#[cfg]` ile **derleme anında** hangi main'in kullanılacağı belirlenir:

| Feature | Ne Zaman? |
|---|---|
| `flutter` | Flutter ile derlenmiş masaüstü |
| `cli` | Komut satırı |
| `android` / `ios` | Mobil |
| Hiçbiri | Sciter UI (eski, deprecated) |

**Flutter modunda (`#[cfg(feature = "flutter")]`)**:
```
main()
  → common::global_init()           // Log, config, crash handler başlat
  → common::test_rendezvous_server() // Rendezvous sunucusunu test et
  → common::test_nat_type()          // NAT tipini ölç
  → Flutter main() çalışır           // flutter/lib/main.dart
```

### `src/core_main.rs`

Flutter için `core_main()` çağrılır:
1. `global_init()` — global başlatma
2. `load_custom_client()` — özel client config yükle
3. Argümanları işle (`--connect`, `--elevate`, `--no-server`, vb.)
4. Server'ı başlat (gerekirse)
5. `Vec<String>` döndür — Flutter'ın başlaması için

### `flutter/lib/main.dart`

Flutter tarafı:
```
main()
  → WidgetsFlutterBinding.ensureInitialized()
  → runMobileApp() veya masaüstü çoklu pencere sistemi
  → GetMaterialApp() ile uygulama
```

---

## 3. Flutter ↔ Rust Köprüsü (FFI)

### `src/flutter_ffi.rs` (3134 satır) — ★★ EN KRİTİK DOSYA ★★

Bu dosya, Flutter'dan Rust'a yapılan **TÜM** çağrıların karşılığını içerir.
`flutter_rust_bridge` kullanılarak FFI fonksiyonları otomatik oluşturulur.

**FFI'da tanımlı ana fonksiyonlar:**

```rust
// === Başlatma ===
initialize(app_dir, custom_client_config)
start_global_event_stream(sink, app_type)

// === ID & Peer Yönetimi ===
peer_get_id()                    → Bu cihazın ID'sini döndür
peer_get_peer_info(id)           → Karşı cihaz ID'sine göre bilgi getir
peer_get_peer_list()             → Peer listesini al

// === Bağlantı ===
connect(id, password, ...)       → Remote bağlantı başlat
session_start_(session_id, ...)  → Session başlat
session_close(id)                → Session kapat

// === Video ===
video_start_display(session_id, display)
video_stop_display(session_id, display)
video_get_image(display)         → RGBA frame al

// === Input ===
input_keyboard(session_id, ...)  → Klavye tuşu gönder
input_mouse(session_id, ...)     → Fare hareketi/tıklama gönder

// === Ses ===
start_audio_thread(...)
stop_audio_thread(...)

// === Dosya Transferi ===
file_open(id, path)
file_read(id, name, offset, size)
file_write(id, name, offset, data)
file_close(id)

// === Clipboard ===
get_clipboard()
set_clipboard(data)

// === Config ===
get_option(key) / set_option(key, value)
load_options()
```

**Önemli:** `connect()` → `rendezvous_mediator.rs`'e gider.
`session_start_()` → `server/connection.rs`'e gider.

---

## 4. Rendezvous / ID Sistemi

### `src/rendezvous_mediator.rs` (992 satır)

RustDesk'in **merkezi sinir sistemi**. Her RustDesk cihazı rendezvous sunucusuna bağlanarak:

1. **ID'sini kaydeder** (register)
2. **Başka cihazları arar** (discover)
3. **Bağlantı kurulmasına yardımcı olur** (relay / peer-to-peer)

**Ana struct: `RendezvousMediator`**

```
RendezvousMediator {
    addr: TargetAddr,       // Sunucu adresi
    host: String,           // Sunucu host
    host_prefix: String,    // Host prefix
    keep_alive: i32,        // Keep-alive aralığı
}
```

**Ana metodlar:**

```
start_all()              → Her şeyi başlat
  ├── test_nat_type()    → NAT tipini ölç
  ├── hbbs_http/sync     → HTTP sync başlat
  └── loop:
       ├── register()        → ID'yi sunucuya kaydet
       ├── handle_message()  → Gelen mesajları işle
       └── keep_alive()      → Bağlantıyı canlı tut

register(id, password)       → ID + şifre ile kayıt
discover_peer(id)            → Bir peer'i bul
request_relay(id)            → Relay üzerinden bağlan
```

**Nasıl çalışır:**

```
Cihaz A (ID: 123)
    │
    │ UDP/TCP bağlantı
    ▼
┌──────────────────┐
│ Rendezvous Server│ (hbbs)
│   (rustdesk.com) │
└──────────────────┘
    ▲
    │
Cihaz B (ID: 456)
    │ "123'e bağlanmak istiyorum"
    ▼
Rendezvous Mediator ← → server/connection.rs
    "123 şu adreste, direkt bağlan veya relay kullan"
```

**Önemli mesaj tipleri (protobuf):**
- `RegisterPeer` — ID kaydı
- `PeerInfo` — Peer bilgisi
- `ConnectRequest` — Bağlantı isteği
- `RelayRequest` — Relay isteği
- `NatType` — NAT tipi bilgisi

---

## 5. Bağlantı Akışı (Connection Flow)

Bir RustDesk oturumu nasıl başlar, adım adım:

```
FLUTTER UI                        RUST BACKEND                     RENDEZVOUS SERVER
────────────                      ────────────                     ────────────────

1. Kullanıcı ID girer
   ve "Bağlan"a tıklar
         │
2. model.dart:                    connect(id, password)
   connect(id) ───────────────►   (flutter_ffi.rs)
                                       │
3.                                 rendezvous_mediator     ───────►  "123 nerede?"
                                       .discover_peer(id)
                                                                   
4.                                                               ◄─────── "123 şu IP:port"
                                                                        (veya "relay kullan")
                                       │
5.                                 TCP bağlantı kur
                                   (socket_client)
                                       │
6.                                 ┌──────────────────┐
                                   │ server/          │
                                   │ connection.rs    │ ← ★ BURASI ÇOK ÖNEMLİ
                                   └──────────────────┘
                                       │
7.                                 handle_message() döngüsü:
                                   - Login (şifre doğrulama)
                                   - OptionMessage (çözünürlük, codec, vs.)
                                   - VideoFrame (ekran görüntüsü)
                                   - AudioFrame (ses)
                                   - InputEvent (klavye/fare)
                                   - Clipboard (pano)
                                   - FileTransfer (dosya)
                                       │
8.                                 Video thread başlat
                                   Audio thread başlat
                                       │
9.  video_get_image() ◄──────────  video_service.rs
    (Flutter her frame'de çağırır)  (encodelanmış video)
                                       │
10. RemotePage ekranda gösterir
```

**Bağlantı türleri:**

| Tür | Ne işe yarar? |
|---|---|
| `DEFAULT_CONN` | Normal remote desktop |
| `FILE_TRANSFER` | Dosya transferi |
| `PORT_FORWARD` | Port yönlendirme |
| `TERMINAL` | Terminal erişimi |
| `VIEW_CAMERA` | Kamera görüntüsü |
| `RDP` | RDP oturumu |

---

## 6. Server Tarafı (Kontrol Edilen)

### `src/server/connection.rs` (6149 satır) — ★★ EN BÜYÜK VE EN ÖNEMLİ DOSYA ★★

Bu dosya **kontrol edilen bilgisayarda** çalışır. Gelen tüm bağlantıları yönetir.

**Ana yapılar:**

```rust
struct Session {
    // Her bağlantı için bir session
    conn: Connection,
    video_service: VideoService,
    audio_service: AudioService,
    input_service: InputService,
    clipboard_service: ClipboardService,
    ...
}

struct Connection {
    writer: Sender,          // Mesaj gönderme kanalı
    reader: Receiver,        // Mesaj alma kanalı
    stream: TcpStream,       // TCP bağlantısı
    ...
}
```

**Bağlantı kurulumu:**

```
run_session(conn)
  → login()           → Şifre/izin kontrolü
  → handle_message()  → Sonsuz döngü, mesajları işler
```

**`handle_message()` — Tüm mesaj tipleri:**

| Mesaj | Ne yapar? |
|---|---|
| `LoginRequest` | Şifre doğrula, izinleri ayarla |
| `PeerInfo` | Karşı tarafın bilgilerini al |
| `OptionMessage` | Çözünürlük, codec ayarları |
| `SwitchDisplay` | Hangi monitör gösterilecek |
| `KeyEvent` | Klavye tuşu → input_service'e yönlendir |
| `MouseEvent` | Fare hareketi → input_service'e yönlendir |
| `VideoFrame` | Video frame gönder (client → server değil, server → client) |
| `AudioFrame` | Ses gönder |
| `Clipboard` | Pano verisi |
| `FileTransfer` | Dosya transferi |
| `CursorData` | İmleç verisi |
| `CloseSession` | Oturumu kapat |

**Güvenlik katmanları:**
1. Şifre doğrulama (hash + salt)
2. IP kara liste/beyaz liste
3. İzin kontrolü (görüntüle, kontrol, dosya, ses)
4. İki faktörlü kimlik doğrulama (2FA)
5. Güvenilir cihaz yönetimi

### `src/server/video_service.rs` (1419 satır)

Ekran yakalama ve kodlama.

```
VideoService::start()
  → capturer = Capturer::new(display)  // scrap kütüphanesi
  → loop:
       frame = capturer.frame()        // Ham ekran görüntüsü
       encoded = encoder.encode(frame) // VP8/VP9/H264/H265/AV1
       send(encoded)                   // Client'a gönder
```

### `src/server/audio_service.rs`

Ses yakalama ve kodlama.

```
AudioService::start()
  → loop:
       audio = capture_audio()         // Mikrofondan ses al
       encoded = opus.encode(audio)    // Opus codec ile kodla
       send(encoded)                   // Client'a gönder
```

### `src/server/input_service.rs` (2373 satır)

**Klavye/Fare girdilerini işler.** Gelen mesajları alır, `enigo` kütüphanesi ile gerçek klavye/fare hareketlerine çevirir.

**İşlediği mesajlar:**
- `KeyEvent` → `enigo.key_click()` / `key_down()` / `key_up()`
- `MouseEvent` → `enigo.mouse_move_to()` / `mouse_click()`
- `PointerDeviceEvent` → Touch/pen girdileri
- `WheelEvent` → Scroll

### `src/server/clipboard_service.rs`

Pano paylaşımı. Her iki yönde de çalışır:
- Server → Client: Server'ın panosundaki değişikliği client'a bildir
- Client → Server: Client'ın panosundaki değişikliği server'a ilet

### `src/server/display_service.rs`

Monitör listesini yönetir:
- Hangi monitörler var?
- Çözünürlükleri ne?
- Hangi monitör şu anda paylaşılıyor?

---

## 7. Client Tarafı (Kontrol Eden)

### `src/client/io_loop.rs` (2506 satır)

Client tarafında **bağlantı döngüsü**. Server'a bağlandıktan sonra mesajları alır/işler.

```rust
struct Remote<T: InvokeUiSession> {
    handler: Session<T>,
    audio_sender: MediaSender,
    receiver: Receiver<Data>,
    sender: Sender<Data>,
    video_threads: HashMap<usize, VideoThread>,
    ...
}
```

**Ana döngü:**

```
run()
  → connect_to_server()
  → login()
  → loop:
       handle_message()  ← Gelen mesajları işle
         ├── VideoFrame   → video_thread'a gönder
         ├── AudioFrame   → ses çalma
         ├── Clipboard    → panoyu güncelle
         ├── CursorData   → imleci güncelle
         ├── FileTransfer  → dosya işle
         └── ChatMessage  → sohbet mesajı
```

### `src/client.rs` (4000+ satır)

Client modülünün ana dosyası. Hem FFI'ya servis verir hem de bağlantı mantığını içerir.

### `src/client/helper.rs`

FFI yardımcıları — Flutter'dan gelen çağrıları client mantığına bağlar.

---

## 8. Video & Ekran Paylaşımı

### `libs/scrap/` — Screen Capture Kütüphanesi

Platforma göre ekran yakalama:

| Platform | Teknoloji |
|---|---|
| Windows | `DXGI Output Duplication` |
| Linux X11 | `X11 ShmGetImage` / `XComposite` |
| Linux Wayland | `PipeWire` / `wlr-screencopy` |
| macOS | `CGDisplayStream` / `SCK` |
| Android | `MediaProjection API` |

**Codec'ler:**

| Codec | Ne zaman? |
|---|---|
| VP8 | Her yerde çalışır (varsayılan) |
| VP9 | Daha iyi sıkıştırma |
| H264 | Donanım hızlandırmalı (NVENC/QSV/VAAPI) |
| H265/HEVC | Daha iyi sıkıştırma, donanım desteği |
| AV1 | En iyi sıkıştırma (aom) |

**Kodlama zinciri:**

```
scrap::Capturer
  → frame() → EncodeInput
    → encoder::encode()
      → vpx / aom / hwcodec
        → EncodedFrame → client'a gönder
```

### `src/server/video_service.rs`

Kalite ayarları (QoS):
- Adaptive bitrate
- Frame drop
- Resolution scaling
- FPS yönetimi

---

## 9. Input (Klavye/Fare)

### Türkçe Q klavye → İngilizce Q klavye haritalama

```rust
// src/keyboard.rs
// TR gibi dillerde tuş haritalama
map_key_to_control_key(key_code, ...)
```

### Fare işleme

```rust
// src/server/input_service.rs
// Mutlak (absolute) ve bağıl (relative) fare modu
// Touch/Pen girdileri
// Scroll (tekerlek)
```

### Enigo kütüphanesi (`libs/enigo/`)

Platforma göre klavye/fare simülasyonu:
- `key_click()` — Tuşa bas/bırak
- `key_down()` / `key_up()` — Basılı tut
- `mouse_move_to(x, y)` — Fareyi taşı
- `mouse_click(MouseButton)` — Tıkla
- `mouse_down()` / `mouse_up()` — Basılı tut

---

## 10. Flutter UI Katmanı

### Sayfa Hiyerarşisi

```
main.dart
  │
  ├── Mobile: mobile/pages/home_page.dart
  │
  └── Desktop: desktop/pages/desktop_home_page.dart
        │
        ├── DesktopTabPage (tablı arayüz)
        │     ├── Remote tab (bağlı cihazlar)
        │     └── ...
        │
        ├── ConnectionPage (bağlantı ayarları)
        │
        ├── RemotePage (★ remote desktop ekranı)
        │     ├── Video render (texture)
        │     ├── Toolbar (araç çubuğu)
        │     ├── Input handling (klavye/fare)
        │     └── Overlay menüler
        │
        ├── FileManagerPage (dosya transferi)
        ├── TerminalPage (terminal)
        └── ServerPage (server ayarları)
```

### Model Katmanı (State Management)

```
model.dart (FFI ile iletişim)
  ├── peer_model.dart   → Peer listesi, durum
  ├── server_model.dart → Server ayarları
  ├── state_model.dart  → Global state (window, session)
  ├── input_model.dart  → Girdi yönetimi
  ├── file_model.dart   → Dosya transferi
  ├── chat_model.dart   → Sohbet
  └── platform_model.dart → Platform spesifik FFI çağrıları
```

**Veri akışı:**

```
Kullanıcı tıklaması
  → Widget (ör: RemotePage)
    → Model (ör: model.dart'daki metod)
      → FFI (flutter_ffi.rs)
        → Rust backend
          → Protobuf mesajı
            → TCP/UDP
              → Karşı taraftaki server/connection.rs
```

### Ana Ekran (`desktop_home_page.dart`)

1. ID gösterimi (kopyalanabilir)
2. Peer ID giriş kutusu
3. Peer listesi (geçmiş bağlantılar)
4. Bağlan butonu
5. Ayarlar butonu
6. Server başlat/durdur

### Remote Ekran (`remote_page.dart`)

1. Video görüntüleme (texture)
2. Toolbar:
   - Ekran görüntüsü al
   - Kayıt başlat/durdur
   - Tam ekran
   - Ctrl+Alt+Del gönder
   - Pano paylaşımı
   - Sohbet
   - Dosya transferi
   - Ses ayarları
3. Klavye/Fare girdi işleme

---

## 11. Önemli Kütüphaneler (libs/)

### `libs/hbb_common/` — ★ Ortak Kütüphane

| Dosya | Açıklama |
|---|---|
| `src/config.rs` | Tüm ayarlar (LocalConfig, PeerConfig, Config) |
| `src/socket_client.rs` | TCP/UDP soket yardımcıları |
| `src/fs.rs` | Dosya sistemi (transfer işlemleri) |
| `src/protobuf/` | Protobuf mesaj tanımları |
| `proto/rendezvous.proto` | Rendezvous mesajları |
| `proto/message.proto` | Data mesajları (video, audio, input, vs.) |

**Önemli protobuf mesajları:**

```protobuf
// rendezvous.proto
message RendezvousMessage {
    RegisterPeer register_peer = 1;
    PeerInfo peer_info = 2;
    ConnectRequest connect_request = 3;
    RelayRequest relay_request = 4;
    NatType nat_type = 5;
    ...
}

// message.proto
message Message {
    LoginRequest login_request = 1;
    LoginResponse login_response = 2;
    VideoFrame video_frame = 3;
    AudioFrame audio_frame = 4;
    KeyEvent key_event = 5;
    MouseEvent mouse_event = 6;
    Clipboard clipboard = 7;
    FileTransfer file_transfer = 8;
    OptionMessage option_message = 9;
    CursorData cursor_data = 10;
    CloseSession close_session = 11;
    ...
}
```

### `libs/scrap/` — Ekran Yakalama

```
Capturer::new(display)   → Belirtilen monitörü yakalamaya başla
  .frame()               → Bir sonraki frame'i al (blocking)
  .is_available()        → Yeni frame var mı?
  .set_display(display)  → Hangi monitör?
  .set_resolution(w, h)  → Çözünürlük değiştir
```

### `libs/enigo/` — Input Simülasyonu

```
Enigo::new()
  .key_click(Key::A)           → 'A' tuşuna bas
  .key_down(Key::Shift)        → Shift basılı tut
  .key_up(Key::Shift)          → Shift bırak
  .mouse_move_to(x, y)         → Fareyi taşı
  .mouse_click(MouseButton::Left) → Sol tık
  .mouse_scroll_x(amount)      → Yatay scroll
  .mouse_scroll_y(amount)      → Dikey scroll
```

---

## 12. Önerilen Okuma Sırası

### Aşama 1: Temel Akışı Anlama (Başlangıç)

```
1. src/main.rs                       ← Nereden başlıyor?
2. src/core_main.rs                  ← Ortak başlangıç
3. src/flutter_ffi.rs (ilk 200 satır)← Hangi FFI fonksiyonları var?
4. flutter/lib/main.dart             ← Flutter nasıl başlıyor?
5. flutter/lib/desktop/pages/desktop_home_page.dart ← Ana sayfa
6. flutter/lib/models/model.dart (ilk 200 satır)  ← State model
```

### Aşama 2: Bağlantı Mantığı (Orta Seviye)

```
7.  src/rendezvous_mediator.rs (ilk 200 satır)  ← ID sistemi
8.  libs/hbb_common/src/config.rs (ilk 300 satır) ← Ayarlar
9.  src/server/connection.rs (handle_message fonksiyonu) ← Ana bağlantı
10. src/client/io_loop.rs (handle_message fonksiyonu) ← Client mesajları
11. src/client.rs (login, connect kısımları)  ← Client mantığı
```

### Aşama 3: Spesifik Servisler (İleri Seviye)

```
12. src/server/video_service.rs       ← Video
13. src/server/audio_service.rs       ← Ses
14. src/server/input_service.rs       ← Input
15. src/server/clipboard_service.rs   ← Pano
16. src/server/display_service.rs     ← Ekran yönetimi
17. flutter/lib/desktop/pages/remote_page.dart ← Remote UI
18. flutter/lib/models/peer_model.dart         ← Peer model
19. flutter/lib/desktop/pages/connection_page.dart ← Bağlantı ayarları
```

### Aşama 4: Platform ve Kütüphaneler (Uzman)

```
20. libs/scrap/src/lib.rs             ← Screen capture
21. libs/enigo/src/lib.rs             ← Input simulation
22. src/platform/windows.rs           ← Windows spesifik
23. src/platform/linux.rs             ← Linux spesifik
24. src/keyboard.rs                   ← Tuş haritalama
25. src/privacy_mode.rs               ← Gizlilik modu
```

---

## 13. Sık Kullanılan Kalıplar

### Rust'ta Mesaj İşleme

```rust
// server/connection.rs içinden
async fn handle_message(&mut self, msg: Message) -> ResultType<()> {
    match msg.union {
        Some(message::Union::video_frame(frame)) => {
            self.video_service.handle_frame(frame).await?;
        }
        Some(message::Union::key_event(event)) => {
            self.input_service.handle_key(event).await?;
        }
        Some(message::Union::mouse_event(event)) => {
            self.input_service.handle_mouse(event).await?;
        }
        Some(message::Union::clipboard(clip)) => {
            self.clipboard_service.handle_clipboard(clip).await?;
        }
        Some(message::Union::file_transfer(ft)) => {
            self.handle_file_transfer(ft).await?;
        }
        Some(message::Union::close_session(_)) => {
            return Ok(()); // Session'ı sonlandır
        }
        _ => {
            log::debug!("Unknown message: {:?}", msg);
        }
    }
    Ok(())
}
```

### Flutter → FFI → Rust Çağrısı

```dart
// flutter/lib/models/model.dart
Future<void> connect(String id, String password) async {
  final result = await ffi.connect(id, password);
  // result Rust'tan döndü
}
```

```rust
// src/flutter_ffi.rs
pub fn connect(id: String, password: String) -> ResultType<String> {
    rendezvous_mediator::connect(id, password)
}
```

### Protobuf Mesaj Yaratma

```rust
use hbb_common::message_proto::{Message, KeyEvent, key_event::Union};

let mut msg = Message::new();
let mut key = KeyEvent::new();
key.set_union(Union::down(true));
key.set_name("a".into());
msg.set_union(message::Union::key_event(key));
// msg artık bir KeyEvent mesajı
```

### Config Okuma/Yazma

```rust
use hbb_common::config::{Config, LocalConfig, PeerConfig};

// Global config
let id = Config::get_id();
let option = Config::get_option("some_key");
Config::set_option("some_key", "value");

// Local config (cihaza özel)
LocalConfig::set_option("access_token", "token");

// Peer config (karşı cihaza özel)
let peer = PeerConfig::load("peer_id");
```

---

## 💡 Hızlı İpucu

Bir özelliğin kodunu arıyorsan:

1. Önce `flutter/lib/models/` veya `flutter/lib/desktop/pages/`'de Flutter tarafını bul
2. FFI çağrısını takip et → `ffi.fonksiyonAdi()`
3. `src/flutter_ffi.rs`'de aynı isimli fonksiyonu bul
4. O fonksiyonun çağırdığı Rust fonksiyonuna git
5. Genellikle `server/connection.rs` veya `client/io_loop.rs`'de sonlanır

Örnek: **"Dosya transferi nasıl çalışıyor?"**

```
flutter/lib/models/file_model.dart
  → ffi.file_open() → flutter_ffi.rs: file_open()
    → server/connection.rs: handle_file_transfer()
      → libs/hbb_common/src/fs.rs: dosya okuma/yazma
```

---

## 🔗 Faydalı Bağlantılar

| Kaynak | Link |
|---|---|
| Resmi Dokümantasyon | https://rustdesk.com/docs/ |
| GitHub | https://github.com/rustdesk/rustdesk |
| Server Kurulumu | https://github.com/rustdesk/rustdesk-server |
| Discord | https://discord.gg/nDceKgxnkV |
| Protobuf Tanımları | libs/hbb_common/proto/ |
| Config Seçenekleri | libs/hbb_common/src/config.rs |

---

*Hazırlayan: opencode AI — Soruların olursa çekinmeden sor!*
