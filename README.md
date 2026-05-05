1. What are the key differences between unary, server streaming, and bi-directional streaming RPC methods, and in what scenarios would each be most suitable?

Unary: Klien mengirim satu request dan menunggu satu response dari server (mirip pemanggilan fungsi biasa). Sangat cocok untuk operasi standar seperti CRUD data atau inisiasi pembayaran.

Server Streaming: Klien mengirim satu request, tetapi server merespons dengan mengirimkan serangkaian data secara bertahap (stream). Cocok untuk skenario di mana data yang diminta sangat besar atau panjang, seperti mengunduh file besar, menampilkan riwayat transaksi.

Bi-directional Streaming: Klien dan server sama-sama dapat mengirim dan menerima aliran data secara independen dan bersamaan dalam satu koneksi persisten. Sangat ideal untuk aplikasi real-time yang interaktif, seperti aplikasi chat, multiplayer gaming, atau pelacakan lokasi real-time.


2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

Untuk menjaga keamanan, koneksi gRPC sebaiknya dienkripsi menggunakan TLS (Transport Layer Security) atau mTLS agar data tetap aman dan tidak mudah disadap selama proses transmisi. Untuk authentication, kita bisa menyisipkan token seperti JWT ke dalam metadata gRPC, yang mekanismenya mirip dengan HTTP headers. Sementara itu, untuk authorization, kita perlu mengimplementasikan middleware (misalnya menggunakan layer dari library tower di Rust) untuk memvalidasi apakah pengguna yang mengirim token memiliki hak akses yang sesuai untuk menjalankan metode RPC tertentu.


3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?

Tantangan utamanya adalah state management dan concurrency. Dalam aplikasi chat, server harus melacak siapa saja yang sedang online dan mempertahankan koneksi (stream) mereka (misalnya menggunakan HashMap yang dibungkus Arc<Mutex>). Selain itu, penanganan error saat koneksi terputus tiba-tiba (client disconnect) harus dikelola dengan baik agar tidak menyebabkan memory leak atau thread yang menggantung. Mengatur antrean pesan dengan channel MPSC juga rawan deadlock jika tidak dikonfigurasi dengan kapasitas buffer yang tepat.


4. What are the advantages and disadvantages of using the tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services?

Kelebihan: tokio_stream::wrappers::ReceiverStream sangat praktis karena dapat menjembatani mpsc::channel milik Tokio dengan trait Stream yang dibutuhkan oleh Tonic pada gRPC streaming. Dengan begitu, implementasi asynchronous menjadi lebih sederhana karena server cukup mengirim data melalui transmitter (tx), sementara receiver (rx) secara otomatis dapat digunakan sebagai stream respons gRPC.

Kekurangan: Penggunaan ReceiverStream menambahkan sedikit overhead karena adanya proses pembungkusan channel menjadi stream. Selain itu, karena mekanismenya bergantung pada MPSC channel, kita juga perlu memperhatikan backpressure. Jika client membaca data lebih lambat dibanding server mengirim data, buffer channel dapat penuh sehingga operasi send menjadi tertunda atau bahkan gagal.


5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?

Agar kode lebih mudah di-maintain dan dikembangkan, struktur proyek sebaiknya dipisahkan ke dalam beberapa layer sesuai prinsip modularitas dan clean architecture. Misalnya, proyek dapat dibagi menjadi modul khusus untuk kode hasil generate Protobuf, modul Handler untuk menangani request dan response gRPC, modul Service untuk logika bisnis utama, serta modul Repository untuk interaksi dengan database. Dengan pemisahan tanggung jawab seperti ini, perubahan pada satu bagian (misalnya struktur database) tidak akan terlalu memengaruhi bagian lain seperti handler gRPC. Selain meningkatkan maintainability, pendekatan ini juga mempermudah code reuse, testing, dan pengembangan fitur baru di masa depan.


6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?

Untuk sistem finansial atau aplikasi patungan (split bill) yang sebenarnya, kita tidak bisa sekadar mengembalikan nilai true. Kita harus mengimplementasikan validasi bisnis (seperti mengecek apakah saldo mencukupi), menerapkan transaksi database yang memenuhi sifat ACID agar tidak ada data yang inkonsisten saat potong saldo, serta mengintegrasikan service ini dengan API Payment Gateway eksternal. Selain itu, penanganan error yang lebih spesifik sangat penting, misalnya dengan mereturn status gRPC seperti FailedPrecondition atau Internal jika transaksi gagal diproses.


7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?

Adopsi gRPC mendorong penggunaan pendekatan contract-first melalui file .proto yang mendefinisikan service dan struktur data secara terstandarisasi. Pendekatan ini membuat arsitektur sistem terdistribusi, seperti microservices, menjadi lebih rapi, konsisten, dan mengurangi risiko ketidaksesuaian tipe data antar layanan. Dari sisi interoperability, gRPC memiliki keunggulan besar karena mendukung berbagai bahasa pemrograman. Sebagai contoh, microservice yang ditulis menggunakan Rust dapat berkomunikasi dengan layanan lain yang dibuat menggunakan Java, Python, atau Go, karena Protobuf dapat secara otomatis menghasilkan kode client dan server (stub/skeleton) untuk berbagai platform dan bahasa tersebut.


8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

Kelebihan: HTTP/2 mendukung multiplexing, sehingga banyak request dan response dapat dikirim secara simultan melalui satu koneksi TCP tanpa perlu membuka banyak koneksi seperti pada HTTP/1.1. Selain itu, fitur kompresi header (HPACK) dan penggunaan format biner membuat komunikasi menjadi lebih efisien dalam penggunaan bandwidth dan memiliki performa yang lebih baik. Dibandingkan kombinasi HTTP/1.1 dengan WebSocket, HTTP/2 juga menyediakan dukungan streaming yang lebih terintegrasi dan terstandarisasi.

Kekurangan: Karena menggunakan format biner, data pada HTTP/2 tidak mudah dibaca secara langsung oleh manusia. Akibatnya, proses debugging dan monitoring menjadi lebih kompleks dibanding REST API berbasis HTTP/1.1 yang menggunakan format teks seperti JSON. Beberapa alat sederhana seperti curl atau inspeksi biasa pada Network Tab browser juga memiliki keterbatasan untuk menganalisis traffic gRPC tanpa bantuan tool tambahan.


9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?

Model request-response pada REST API cenderung kurang efisien untuk komunikasi real-time karena client perlu secara berkala mengirim request ke server (polling) untuk memeriksa apakah ada data baru. Pendekatan ini dapat meningkatkan penggunaan bandwidth dan menambah latensi komunikasi. Sebaliknya, bidirectional streaming pada gRPC mempertahankan koneksi dua arah yang tetap terbuka, sehingga client dan server dapat saling mengirim data kapan saja tanpa harus membuat request baru berulang kali. Dengan mekanisme ini, komunikasi menjadi lebih responsif dan sangat cocok untuk aplikasi real-time seperti chat, live notification, atau multiplayer game.


10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

Pendekatan berbasis skema pada gRPC menggunakan Protocol Buffers mengharuskan client dan server mengikuti struktur data yang telah didefinisikan di file .proto. Pendekatan ini menciptakan kontrak yang jelas antar layanan, sehingga proses serialisasi dan parsing data menjadi lebih cepat, ukuran payload lebih kecil, serta risiko kesalahan tipe data saat runtime dapat dikurangi. Hal ini dimungkinkan karena Protobuf menggunakan representasi biner dan tag numerik, bukan nama field dalam bentuk teks seperti JSON. Sebaliknya, payload JSON pada REST API bersifat lebih fleksibel karena struktur datanya dapat berubah dengan lebih mudah tanpa definisi skema yang ketat. Namun, fleksibilitas ini memiliki konsekuensi berupa ukuran payload yang lebih besar, performa parsing yang lebih lambat, dan kemungkinan terjadinya error apabila format atau tipe data yang diterima tidak sesuai dengan yang diharapkan aplikasi.