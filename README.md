# Commit 1 Reflection

Ada dua fungsi dalam kode Rust ini, yaitu main dan handle_connection. Fungsi main memulai program dan bertanggung jawab untuk mengatur TCP listener dan menangani koneksi masuk. Fungsi itu menggunakan TcpListener::bind() untuk membuat sebuah instance TcpListener, dan menentukan alamat IP lokal dan port yang akan diikat untuk instance tersebut. Metode unwrap() digunakan untuk menangani kesalahan apa pun yang mungkin terjadi selama proses pengikatan dan panik jika terjadi kesalahan.

Setelah mengonfigurasi sebuah listener, fungsi main memasuki sebuah loop yang menggunakan listener.incoming(), melakukan iterasi untuk koneksi yang masuk. Setiap koneksi memiliki type Result<TcpStream, std::io::Error>. Terjadi panggilan metode unwrap() pada setiap Result untuk mengambil TcpStream dan panik jika terjadi kesalahan. Untuk setiap koneksi yang berhasil dibuat, fungsi handle_connection dipanggil, menerima stream dengan type TcpStream sebagai parameternya.

Fungsi handle_connection bertanggung jawab untuk menangani setiap koneksi individual. Dibutuhkan mutable TcpStream sebagai satu parameter yang memungkinkan membaca dan menulis ke koneksi. Di dalam fungsinya di baris pertama, BufReader::new() digunakan untuk membuat satu instance BufReader yang menggabungkan mutable reference ke TcpStream. BufReader melakukan pembacaan buffer.

Fungsi handle_connection kemudian menggunakan serangkaian iterator adapter seperti map, take_while, collect untuk melakukan parsing request HTTP. Fungsi tersebut memanggil metode lines() dari objek BufReader, yang melakukan iterasi untuk baris-baris buffered reader tersebut. Setiap baris diwakili oleh type Result<String, std::io::Error>. Metode map() kemudian digunakan untuk mengubah setiap Result menjadi nilai suksesnya (baris itu sendiri) menggunakan unwrap(), panik hanya jika terjadi kesalahan. Metode take_ while() digunakan untuk mengumpulkan setiap baris hingga menemukan baris kosong, mengikuti konvensi protokol HTTP untuk memisahkan header dan isi. Terakhir, metode Collect() dipanggil untuk mengumpulkan semua baris yang sudah diproses ke dalam Vec<String> dan menyimpannya dalam variabel http_request. Baris terakhir dari fungsi ini melakukan print http_request menggunakan println!. Bagian ({:#?}) ada untuk meningkatkan keterbacaan dari hasil print tersebut.

# Commit 2 Reflection

Kode untuk fungsi handle_connection sudah diubah untuk commit 2 sehingga bisa membuat respons http dan tidak mencetak request http seperti di commit 1. Setelah memproses HTTP request, akan dibuat respons dalam variabel response yang dikembalikan melalui sebuah instance TCPStream yang merupakan parameter fungsi handle_connection.

Variabel status_line yang memiliki tipe &str, referensi ke string statis "HTTP/1.1 200 OK", berarti status dari respons HTTP sukses.

Fungsi fs::read_to_string lalu menggunakan file "hello.html" dan memasukkannya ke dalam variabel contents. Seperti biasa, unwrap untuk menangani kesalahan, secara khusus yang bisa terjadi saat membaca file.  

Panjang dari contents disimpan dalam variabel length.

Konten untuk HTTP digunakan dengan format! makro untuk memformat string (mirip dengan printf dalam C). Status line, header Content-Length yang menentukan panjang contents, baris kosong untuk memisahkan header dari body, dan konten itu sendiri.

Terakhir, write_all membangun respons HTTP ke koneksi TCP. Metode as_bytes melakukan konversi string ke bytes agar dapat ditulis ke stream. Lalu, klien bisa melihat isi hello.html muncul di browser nya saat 127.0.0.1:7878.

![Commit 2 screen capture](/assets/images/commit2.PNG)

# Commit 3 Reflection 

Sebelumnya, dengan mengikuti bab 20 buku Rust tersebut, terdapat blok if-else yang besar dengan logika serupa untuk menangani dua kasus: jika request_line sama dengan "GET / HTTP/1.1" dan jika bukan. Dalam kedua kasus tersebut, kode untuk membaca isi file HTML, menghitung panjang contents, dan membuat respons HTTP berformat hampir sama,  berbeda hanya pada nilai variabel status_line dan file_name yang digunakan.

Untuk menghindari duplikasi, dilakukan refactoring untuk blok if-else yang besar sehingga menjadi satu ekspresi if ringkas untuk (status_line, filename) yang bernilai ("HTTP/1.1 200 OK", "hello.html") jika request_line sama dengan "GET / HTTP/1.1" dan ("HTTP/1.1 404 NOT FOUND", "404.html") jika tidak.

Setelah menentukan status_line dan filename yang sesuai berdasarkan request_line, logika untuk membaca file, menghitung panjang contents, dan memformat dan menuliskan respons HTTP dalam bytes ke stream tetap sama seperti sebelumnya menurut buku. Kode tersebut dalam proses refactoring dipindahkan ke luar blok if-else karena berlaku untuk kedua kasus.

Sekarang, server kita akan menunjukkan konten 404.html jika bukan http://127.0.0.1:7878 .

![Commit 3 screen capture](/assets/images/commit3.PNG)

# Commit 4 Reflection

Ketika request untuk url /sleep diterima, server memanggil thread::sleep(Duration::from_secs(10)) yang menyebabkan thread yang sedang menangani request tersebut tidur selama 10 detik sebelum melanjutkan pemrosesan dan mengirimkan respons.

Karena server berjalan pada hanya satu thread, saat server menerima request "GET /sleep HTTP/1.1", thread tidur selama 10 detik sebelum mengirimkan respons. Selama sepuluh detik tersebut, thread server akan diblokir untuk request lain. Hal ini menyebabkan request lain, seperti "GET / HTTP/1.1" pada jendela browser kedua, harus menunggu hingga request "/sleep" selesai diproses. Masalah ini akan semakin buruk jika banyak pengguna mencoba mengakses server secara bersamaan karena setiap request  memakan waktu lama dan memblokir satu thread server dan menyebabkan request lain tertunda atau bahkan timeout.

## Commit 5 Reflection

ThreadPool  struktur dalam kode Rust ini untuk mengelola satu kelompok thread yang siap untuk menjalankan tugas-tugas secara konkuren. Ide utama di balik ThreadPool adalah menciptakan sejumlah thread dalam jumlah tertentu dan menjalankan tugas-tugas yang berbeda, alih-alih menciptakan thread baru untuk setiap tugas. Hal ini meningkatkan performa dengan menghindari overhead dari pembuatan dan penghancuran thread secara terus-menerus. ThreadPool menerima tugas-tugas (dalam bentuk closure) melalui metode execute, lalu mendistribusikan tugas-tugas tersebut ke thread-thread worker yang tersedia. Thread-thread worker mengambil tugas-tugas dari antrian dan menjalankannya secara konkuren, sehingga memungkinkan pemrosesan tugas-tugas secara efficient dan scalable.
