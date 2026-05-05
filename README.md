# Module 08 - Rust gRPC Reflection

## 1. Perbedaan unary, server streaming, dan bi-directional streaming RPC

Unary RPC adalah pola komunikasi paling sederhana, yaitu client mengirim satu request dan server mengembalikan satu response. Contohnya pada `PaymentService`, client mengirim data pembayaran, lalu server cukup mengembalikan status berhasil atau gagal. Pola ini cocok untuk operasi yang hasilnya langsung dan tidak perlu banyak data, seperti login, validasi pembayaran, atau mengambil detail satu data.

Server streaming RPC adalah pola ketika client mengirim satu request, tetapi server mengirim banyak response secara bertahap. Pada tutorial ini contohnya adalah `TransactionService`, karena riwayat transaksi bisa berisi banyak data. Pola ini cocok untuk fitur seperti list data besar, log aktivitas, progress update, atau notifikasi yang dikirim terus dari server ke client.

Bi-directional streaming RPC berarti client dan server sama-sama bisa mengirim banyak message dalam satu koneksi. Contohnya adalah `ChatService`, di mana client bisa terus mengirim pesan, dan server bisa langsung membalas pesan tersebut. Pola ini cocok untuk chat, multiplayer game, live collaboration, atau sistem yang butuh komunikasi real-time dua arah.

## 2. Pertimbangan keamanan pada gRPC service di Rust

Beberapa hal penting dari sisi keamanan adalah authentication, authorization, dan encryption. Authentication dibutuhkan supaya server tahu siapa client yang sedang mengakses service. Misalnya bisa memakai token seperti JWT, API key, atau integrasi OAuth.

Authorization juga penting karena user yang sudah login belum tentu boleh mengakses semua method. Contohnya, user biasa mungkin hanya boleh melihat transaksi miliknya sendiri, sedangkan admin boleh melihat data lebih banyak. Jadi server tetap harus mengecek permission pada setiap request.

Untuk data encryption, gRPC sebaiknya berjalan di atas TLS agar data yang lewat jaringan tidak mudah dibaca pihak lain. Ini penting terutama untuk data sensitif seperti pembayaran, user ID, atau isi chat. Selain itu, server juga perlu validasi input, rate limiting, logging yang aman, dan tidak menampilkan error internal secara berlebihan ke client.

## 3. Tantangan pada bidirectional streaming di Rust gRPC

Tantangan utama bidirectional streaming adalah mengatur banyak proses asynchronous yang berjalan bersamaan. Pada chat, client bisa mengirim pesan kapan saja, sementara server juga bisa mengirim response kapan saja. Kalau pengelolaan task dan channel tidak hati-hati, bisa muncul masalah seperti deadlock, task yang tidak berhenti, atau memory terus bertambah.

Masalah lain adalah disconnect. Client bisa tiba-tiba menutup koneksi, jaringan bisa putus, atau server gagal mengirim balasan. Karena itu kode perlu menangani error dari stream dan channel dengan baik. Pada tutorial, penggunaan `unwrap_or_else` dan pengecekan `is_err()` membantu supaya service tidak langsung panic ketika koneksi bermasalah.

Selain itu, untuk aplikasi chat yang lebih serius, perlu dipikirkan urutan pesan, penyimpanan chat, autentikasi user, dan cara membatasi jumlah pesan agar server tidak kewalahan.

## 4. Kelebihan dan kekurangan `ReceiverStream`

Kelebihan `tokio_stream::wrappers::ReceiverStream` adalah mudah digunakan bersama `tokio::sync::mpsc`. Kita bisa membuat channel, menjalankan task producer, lalu mengubah receiver menjadi stream yang bisa dikirim oleh tonic. Ini membuat implementasi server streaming dan chat menjadi lebih sederhana.

Kelebihan lainnya adalah pemisahan logic menjadi lebih jelas. Bagian producer bertugas membuat data, sedangkan gRPC response cukup membaca dari stream. Ini cocok untuk kasus seperti mengirim riwayat transaksi atau balasan chat.

Kekurangannya, kita tetap harus mengatur buffer channel dengan benar. Kalau buffer terlalu kecil, producer bisa sering menunggu. Kalau terlalu besar, memory bisa lebih boros. Selain itu, error handling perlu diperhatikan karena ketika receiver atau sender ditutup, task harus bisa berhenti dengan rapi.

## 5. Struktur kode agar lebih reusable dan modular

Kode gRPC akan lebih maintainable kalau dipisah berdasarkan tanggung jawab. Misalnya definisi module generated proto bisa diletakkan di satu tempat, lalu implementasi `PaymentService`, `TransactionService`, dan `ChatService` dibuat dalam file atau module terpisah.

Selain itu, logic bisnis sebaiknya tidak langsung ditulis semuanya di method gRPC. Contohnya, validasi pembayaran bisa dibuat di service layer sendiri, lalu gRPC handler hanya menerima request, memanggil service layer, dan mengembalikan response. Dengan cara ini, logic lebih mudah dites dan lebih mudah dikembangkan.

Untuk project yang lebih besar, struktur yang bisa dipakai misalnya:

```text
src/
  grpc_server.rs
  grpc_client.rs
  services/
    payment.rs
    transaction.rs
    chat.rs
  domain/
    payment_processor.rs
```

Dengan struktur seperti ini, jika nanti ada service baru, kita tidak perlu membuat satu file server menjadi terlalu panjang.

## 6. Langkah tambahan untuk payment processing yang lebih kompleks

Implementasi `MyPaymentService` pada tutorial masih sangat sederhana karena langsung mengembalikan `success: true`. Untuk sistem pembayaran nyata, perlu ada validasi input seperti memastikan `user_id` valid, amount tidak negatif, dan format request benar.

Setelah itu, server perlu mengecek saldo atau metode pembayaran, berkomunikasi dengan payment gateway, menangani timeout, dan menyimpan transaksi ke database. Error juga harus dibedakan, misalnya pembayaran ditolak, saldo kurang, gateway error, atau request tidak valid.

Selain itu, sistem pembayaran juga perlu idempotency key agar request yang terkirim dua kali tidak membuat pembayaran dobel. Logging dan audit trail juga penting karena pembayaran berhubungan dengan data finansial.

## 7. Dampak penggunaan gRPC pada arsitektur distributed system

Penggunaan gRPC membuat komunikasi antar-service menjadi lebih terstruktur karena semua service didefinisikan lewat file `.proto`. Ini membantu dalam distributed system karena contract antara client dan server menjadi jelas. Service yang dibuat di Rust juga bisa dipanggil dari bahasa lain seperti Go, Java, Python, atau Node.js selama mereka memakai schema protobuf yang sama.

gRPC juga cocok untuk arsitektur microservices karena performanya bagus dan mendukung beberapa jenis komunikasi, termasuk streaming. Namun, tim juga perlu menyiapkan tooling seperti load balancing, service discovery, observability, dan versioning schema protobuf.

Dampaknya, desain sistem menjadi lebih contract-first. Perubahan pada `.proto` harus dipikirkan dengan hati-hati agar tidak merusak client lama.

## 8. Kelebihan dan kekurangan HTTP/2 untuk gRPC dibanding HTTP/1.1 atau WebSocket

HTTP/2 punya beberapa kelebihan seperti multiplexing, header compression, dan koneksi yang lebih efisien. Dengan multiplexing, beberapa request bisa berjalan dalam satu koneksi tanpa harus menunggu satu sama lain selesai. Ini membuat gRPC cocok untuk komunikasi service-to-service yang butuh performa tinggi.

Dibanding HTTP/1.1 biasa, gRPC lebih efisien karena payload protobuf lebih kecil daripada JSON dan HTTP/2 lebih modern. Dibanding WebSocket, gRPC punya schema yang jelas dan tooling code generation, jadi lebih enak untuk RPC antar-service.

Kekurangannya, gRPC tidak selalu semudah REST untuk dipakai langsung dari browser atau debugging manual. REST bisa dites mudah lewat browser atau curl, sedangkan gRPC biasanya butuh tool khusus seperti grpcurl. WebSocket juga lebih familiar untuk beberapa use case real-time di frontend.

## 9. REST request-response vs bidirectional streaming gRPC

REST API biasanya memakai model request-response. Client mengirim request, server menjawab, lalu koneksi selesai. Model ini sederhana dan cocok untuk operasi biasa seperti CRUD. Namun untuk real-time communication, REST sering membutuhkan polling, long polling, atau tambahan WebSocket.

gRPC dengan bidirectional streaming lebih responsif untuk komunikasi real-time karena client dan server bisa terus bertukar message dalam satu koneksi. Pada chat service, client tidak perlu membuat request baru setiap kali ingin menerima balasan. Server juga bisa langsung mengirim response lewat stream.

Jadi, REST lebih sederhana dan cocok untuk banyak aplikasi web umum, sedangkan gRPC streaming lebih cocok untuk aplikasi yang butuh komunikasi cepat, terus-menerus, dan dua arah.

## 10. Schema-based gRPC dengan Protocol Buffers vs JSON REST

gRPC memakai Protocol Buffers yang schema-based. Artinya struktur data harus didefinisikan dulu di file `.proto`. Kelebihannya, client dan server punya contract yang jelas, tipe data lebih aman, dan code bisa di-generate otomatis. Payload protobuf juga lebih ringkas dibanding JSON.

Di sisi lain, JSON pada REST lebih fleksibel karena tidak harus punya schema yang ketat. Ini membuat REST lebih mudah untuk prototyping atau komunikasi dengan aplikasi web sederhana. JSON juga lebih mudah dibaca manusia.

Namun, fleksibilitas JSON bisa menjadi kekurangan ketika project semakin besar. Jika tidak ada schema yang jelas, perubahan payload bisa menyebabkan bug yang baru ketahuan saat runtime. Dengan protobuf, perubahan data lebih terkontrol, tetapi developer harus lebih disiplin dalam mengelola versi `.proto`.
