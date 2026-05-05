# Tutorial Rust gRPC – Refleksi

Jadi setelah ngerjain tutorial ini, ini beberapa hal yang aku pelajari dan pikirkan.

## 1. Bedanya unary, server streaming, sama bi-directional streaming itu apa sih?

Anggap aja kayak gini:

- **Unary** itu kayak nanya ke Google: ngetik 1 pertanyaan, dapet 1 jawaban. Cocok buat hal-hal sederhana kayak login, ambil data user, atau proses pembayaran. Singkat, padat, jelas.
- **Server streaming** itu kayak nonton YouTube live: kita cuma klik 1 kali (request), tapi server terus-terusan ngasih data ke kita. Cocok buat notifikasi real-time, harga saham yang update terus, atau download file gede yang dipecah jadi chunk.
- **Bi-directional streaming** itu kayak nelpon temen: dua-duanya bisa ngomong kapan aja, bareng-bareng juga bisa. Pas banget buat aplikasi chat, game multiplayer, atau dashboard live yang butuh interaksi dua arah.

Di tutorial ini ketiganya kena semua: PaymentService (unary), TransactionService (server streaming), ChatService (bi-di streaming). Lumayan dapet feel-nya beda-beda.

## 2. Soal keamanan gRPC di Rust

Kalau gRPC mau dipake di production, ada beberapa hal yang ga boleh dilewatin:

- **Authentication**: harus ada cara buat ngecek "kamu siapa". Biasanya pake token (JWT, OAuth) yang dikirim di metadata gRPC. Tonic punya fitur interceptor yang bisa dipake buat validasi token sebelum request masuk ke handler.
- **Authorization**: setelah tau siapa, harus tau juga "kamu boleh ngapain aja". Misalnya cuma admin yang boleh hapus data. Ini biasanya dicek di tiap method.
- **Encryption (TLS)**: ini wajib banget kalau jalan di internet. Tanpa TLS, traffic gRPC bisa di-sniff orang. Tonic udah support TLS via `ServerTlsConfig` dan `ClientTlsConfig`.

Intinya: jangan deploy gRPC tanpa TLS dan auth, kecuali emang internal banget di dalam network privat.

## 3. Tantangan bikin chat (bidirectional streaming) di Rust

Awalnya keliatan simpel, tapi pas diimplementasi ada beberapa hal yang bikin saya agak tertantang antara lain :

- **Banyak orang chat bareng** = harus ati-ati sama race condition. Kalau ada shared state (misalnya list user online), harus dibungkus `Mutex` atau `RwLock`.
- **Koneksi bisa putus tiba-tiba** entah karena WiFi mati, app di-close, atau apalah. Server harus handle dengan rapi, ga boleh leak resource.
- **Backpressure**: kalau client kirim pesan kecepetan sampe buffer penuh, gimana? Mau di-drop? Ditunggu? Ini perlu dipikirin.
- **Error di satu sisi ga otomatis nutup sisi lain**, jadi harus dihandle manual biar ga zombie connection.

## 4. `tokio_stream::wrappers::ReceiverStream` itu enak dipake ga?

**Enaknya:**
- Tinggal pakai, ga perlu mikir banyak. Ubah `mpsc::Receiver` jadi `Stream` cuma 1 baris.
- Nyambung mulus sama tonic.
- Pas banget buat pola producer-consumer.

**Ga enaknya:**
- Buffernya ukurannya tetap. Kalau producer kebut-kebutan, bisa keblokir.
- Cuma single-consumer. Kalau mau broadcast 1 pesan ke banyak penerima, ga bisa pake ini—harus pake `tokio::sync::broadcast`.

Tapi buat use case kebanyakan, udah lebih dari cukup harusnya.

## 5. Cara biar kode gRPC-nya ga berantakan dan gampang di-maintain

Beberapa hal yang menurutku bisa diterapin:

- **Pisah-pisah file**: jangan numpuk semua service di `grpc_server.rs`. Bikin module sendiri kayak `payment.rs`, `transaction.rs`, `chat.rs`. Server.rs cuma jadi entry point doang.
- **Pakai dependency injection**: service struct-nya jangan langsung create database connection di dalemnya. Inject dari luar lewat constructor `new()`.
- **Trait abstraction**: bikin trait kayak `PaymentRepository` biar service ga terikat sama implementasi storage tertentu. Gampang testing dan ganti backend nanti.
- **Interceptor buat hal-hal common**: logging, auth check, metrics—semua bisa dijadiin interceptor biar ga duplikat di tiap method.

## 6. Kalau MyPaymentService dipake beneran, butuh apa lagi?

Banyak banget. Yang sekarang masih super basic, di real world butuh:

- **Validasi input**: cek amount harus positif, user_id beneran ada, mata uang valid.
- **Integrasi ke payment gateway** beneran (Midtrans, Stripe, Xendit) — yang artinya juga harus handle timeout, retry, idempotency.
- **Idempotency key** biar kalau client retry karena network glitch, ga jadi double charge.
- **Database transaction** biar pencatatan pembayaran atomic.
- **Error handling proper**: jangan asal return generic error. Pakai `Status` dengan code yang sesuai (`InvalidArgument`, `Unavailable`, `PermissionDenied`).
- **Observability**: logging yang structured, trace ID, metrics. Biar pas ada masalah di production, gampang dilacak.

## 7. Dampak gRPC ke arsitektur sistem terdistribusi

Plus minus banget sebenernya.

**Plusnya**: kontrak API jelas dan strict (Protobuf), performa kenceng (binary + HTTP/2), dukung streaming native, dan bisa lintas bahasa pemrograman. Mantep buat komunikasi antar microservice di dalam sistem.

**Minusnya**: butuh tooling tambahan (protoc, build script ribet), susah debug karena binary (ga bisa langsung curl kayak REST), dan support browser-nya terbatas (perlu gRPC-Web atau proxy).

Kesimpulanku: gRPC pas banget buat **internal communication** antar service. Tapi kalau bikin public API yang dipake banyak third-party developer, REST masih lebih friendly.

## 8. HTTP/2 vs HTTP/1.1 (atau HTTP/1.1 + WebSocket)

**HTTP/2 menang di:**
- Multiplexing (banyak request paralel di 1 koneksi, ga ada head-of-line blocking).
- Header compression — bandwidth lebih hemat.
- Binary framing yang lebih efisien daripada text-based.

**HTTP/2 kalah di:**
- Lebih ribet di-debug. Kalau di HTTP/1.1 bisa baca traffic langsung pake Wireshark, di HTTP/2 perlu tools khusus.
- Browser umumnya cuma pake HTTP/2 di TLS, jadi perlu sertifikat.

**WebSocket** sebenernya bisa juga buat full-duplex, tapi ga punya struktur message bawaan kayak gRPC. Semua harus didefinisikan manual—formatnya, kapan kirim apa, dll. Lebih ribet.

## 9. Request-response REST vs bidirectional gRPC, real-time-nya gimana?

REST itu kerjanya 1-1: kirim request, tunggu response. Selesai. Buat real-time, harus polling terus (boros) atau pake SSE/WebSocket (tambahan kompleksitas).

gRPC bidirectional streaming udah real-time native: pesan ngalir terus dua arah, ga ada overhead handshake baru tiap pesan. Latency rendah, cocok banget buat chat, IoT, live data.

Kalau aplikasi butuh interaktif real-time, gRPC streaming lebih masuk akal. Kalau cuma CRUD biasa, REST udah cukup.

## 10. Schema-based (Protobuf) vs schema-less (JSON)

Ini debat klasik sebenernya.

**Protobuf** kerasnya:
- Type-safe — error ketauan pas compile, bukan pas user pertama nge-bug di production.
- Payload kecil, parsing cepet.
- File `.proto`-nya itu sendiri jadi dokumentasi.
- Schema bisa di-evolve (tambah field baru) tanpa break client lama.

**Protobuf belajarnya:**
- Butuh build step (compile `.proto` dulu).
- Ga human-readable. Ga bisa langsung di-curl terus dilihat hasilnya.
- Tooling-nya lebih ribet daripada JSON.

**JSON** enaknya:
- Bisa langsung dilihat manusia. Debugging gampang banget.
- Ga butuh code generation.
- Native di JavaScript dan udah jadi standar di web.

**JSON repotnya:**
- Ga ada validasi tipe bawaan. Field bisa kosong, bisa salah tipe—ga ketauan sampai runtime.
- Payload lebih gede, parsing lebih lambat.
- Schema implisit, gampang inkonsisten antar tim kalau ga didokumentasi rapi.

Akhirnya pilihan tergantung konteks. Buat internal high-perf → Protobuf. Buat public API atau prototyping cepet → JSON.

Yaa segitu dulu refleksi dari tutorialnya. Lumayan dapet insight banyak soal gimana komunikasi antar service di sistem distributed bisa lebih efisien pake gRPC.