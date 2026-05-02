# Reflection: Modul 8 High Level Networking
**Nama:** Izzudin Abdul Rasyid  
**NPM:** 2406495786  

**1: What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?**
 
*   **Unary RPC:** Klien mengirimkan satu permintaan ke server dan menunggu satu respons balasan. Ini cocok untuk operasi sederhana seperti mengambil satu entitas data, autentikasi pengguna, atau perhitungan matematis satu kali.
*   **Server Streaming RPC:** Klien mengirimkan satu permintaan, namun server merespons dengan aliran data (stream) secara berurutan. Ini ideal untuk mengirimkan pembaruan data secara *real-time* (seperti harga saham atau cuaca) atau mengirim file berukuran besar secara bertahap (chunk).
*   **Bi-directional Streaming RPC:** Klien dan server mengirimkan aliran pesan satu sama lain secara independen dan terus-menerus melalui koneksi yang sama. Sangat tepat untuk aplikasi interaktif dua arah, seperti aplikasi *chat real-time* atau analitik data *real-time*.

**2: What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?**
 
*   **Data Encryption:** Karena gRPC secara *default* berjalan di atas HTTP/2, komunikasi di lingkungan produksi wajib dibungkus dengan TLS agar *payload* dienkripsi dan tidak bisa disadap.
*   **Authentication:** gRPC mendukung autentikasi bawaan. Di Rust (Tonic), kita dapat menggunakan *Interceptors* untuk memvalidasi token (seperti JWT) yang disisipkan di dalam metadata *headers* pada setiap *reuest*.
*   **Authorization:** Setelah identitas diverifikasi, layanan gRPC harus memvalidasi hak akses (*Role-Based Access Control*) klien sebelum mengizinkan eksekusi logika bisnis.

**3: What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?**

*   **Manajemen Konkurensi & *State*:** Server harus melacak banyak klien secara bersamaan. Mengelola *shared state* (seperti daftar *user* aktif) di antara berbagai *task* asinkron tokio membutuhkan sinkronisasi yang aman (misal dengan `Arc<Mutex<T>>`) untuk menghindari *race conditions*.
*   **Koneksi Terputus:** Server harus proaktif mendeteksi dan menangani klien yang tiba-tiba terputus agar memori tidak bocor (*memory leak*).
*   ***Backpressure*:** Jika salah satu pihak mengirim pesan lebih cepat daripada yang bisa diproses pihak lain, diperlukan manajemen *buffer* pada *channel* agar sistem tidak kehabisan memori.

**4: What are the advantages and disadvantages of using the `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?**

*   **Advantages:** Sangat memudahkan integrasi antara *channel* asinkron bawaan Rust (`tokio::sync::mpsc`) dengan antarmuka *Stream* yang diwajibkan oleh Tonic. Hal ini memungkinkan pemisahan yang bersih antara *task* penghasil data (*producer*) dan *handler* gRPC (*consumer*) menggunakan `tokio::spawn`.
*   **Disadvantages:** Ada sedikit *overhead* performa karena data harus melewati *channel*. Selain itu, jika ukuran *buffer* tidak dikonfigurasi dengan tepat, *producer* bisa terblokir (jika penuh) atau memakan terlalu banyak memori.

**5: In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?**

*   **Separation of Concerns:** Memisahkan kode menjadi modul-modul spesifik. Contohnya: modul `proto` untuk kode yang di-*generate* oleh Protobuf, modul `handlers` untuk *routing* gRPC, dan modul `services` untuk inti logika bisnis.
*   **Dependency Injection:** Menghindari *hardcode* logika database atau API eksternal di dalam *struct* layanan (seperti `MyPaymentService`). Kita bisa melempar *connection pool* atau *state* aplikasi melalui *constructor* *struct* tersebut agar lebih mudah diuji (*unit testing*).

**6: In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?**

Implementasi saat ini hanya statis. Untuk logika yang kompleks, diperlukan:
1.  Validasi input (misalnya memastikan `amount` valid dan `user_id` ada).
2.  Integrasi asinkron dengan API *Payment Gateway* pihak ketiga (seperti Stripe atau Midtrans).
3.  Menyimpan riwayat dan status transaksi ke dalam database dengan menggunakan pola transaksi yang aman (ACID).
4.  Pemetaan *error* yang baik, yaitu mengonversi kegagalan sistem internal menjadi kode `tonic::Status` yang deskriptif bagi klien.

**7: What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?**

 gRPC sangat menguntungkan arsitektur *microservices*. Karena kontrak API didefinisikan secara terpusat menggunakan Protocol Buffers (*.proto*), layanan yang dibangun dengan berbagai bahasa pemrograman (Rust, Java, Python, dll.) bisa saling berkomunikasi seolah-olah memanggil fungsi lokal. Ini menghasilkan interoperabilitas yang tinggi, kode *boilerplate* yang minim berkat *code generation*, serta standarisasi tipe data yang ketat di seluruh ekosistem.

**8: What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?**

*   **Advantages:** HTTP/2 memecahkan masalah *head-of-line blocking* pada HTTP/1.1 dengan fitur *multiplexing*, memungkinkan banyak *reuest/response* berjalan bersamaan dalam satu koneksi TCP. HTTP/2 juga memiliki fitur HPack yang sangat efisien mengompresi *header*, memangkas *overhead* jaringan secara signifikan.
*   **Disadvantages:** Format *binary framing* pada HTTP/2 membuatnya sulit dibaca oleh manusia (*non-human-readable*) saat proses *debugging*, tidak seperti JSON/Teks di HTTP/1.1. Selain itu, tidak semua *web browser* mendukung penuh kapabilitas HTTP/2 gRPC secara *native* tanpa bantuan perantara (*proxy*).

**9: How does the reuest-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?**
REST API berpusat pada koneksi statis di mana klien harus terus-menerus bertanya kepada server (*polling*) untuk mendapatkan data terbaru, yang menghasilkan latensi dan beban jaringan yang tinggi. Sebaliknya, *bidirectional streaming* gRPC menjaga saluran koneksi TCP tetap terbuka secara dua arah. Klien dan server bisa saling mendorong (*push*) data secara reaktif dan bersamaan, memberikan tingkat responsivitas yang instan dan *real-time* sejati.

**10: What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?**
*   **Protocol Buffers (gRPC):** Mengharuskan struktur data didefinisikan di awal (berskema). Hal ini memberikan jaminan *type safety* sejak tahap kompilasi dan ukuran *payload* biner yang sangat kecil dan cepat untuk di-*parsing*.
*   **JSON (REST):** Bersifat fleksibel, *schema-less*, dan mudah dibaca. Namun, karena klien bisa saja mengirimkan data dengan format yang salah, REST API membutuhkan langkah validasi ekstra di sisi server untuk memastikan integritas data, dan teks JSON cenderung memakan ukuran *bandwidth* yang lebih besar.