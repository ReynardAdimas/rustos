# Freestanding Rust Binary

## Apa itu Freestanding Binary?

Freestanding binary adalah executable yang dibuat untuk berjalan tanpa bergantung pada lingkungan sistem operasi atau runtime standar.

Pada aplikasi Rust biasa, kita bergantung pada beberapa komponen, seperti:

Rust Standard Library (std).

C Runtime (crt0).

Runtime initialization.

Entry point standar (main).

Namun, ketika mengembangkan kernel sistem operasi, kita tidak bisa mengandalkan komponen-komponen tersebut karena kernel justru merupakan bagian dari sistem yang menyediakan lingkungan eksekusi.

Oleh karena itu, kita perlu membuat executable yang dapat berjalan secara independen dari runtime sistem operasi.

Salah satu pendekatan yang digunakan Rust adalah #![no_std] dan #![no_main].

## Memahami no_std

Secara default, Rust menggunakan Standard Library (std) yang menyediakan berbagai fasilitas, seperti:

- Dynamic memory allocation
- File system access
- Networking
- Thread management
- I/O operations
- Error handling

Ketika membuat kernel, kita tidak bisa menggunakan fasilitas tersebut secara langsung karena kernel belum memiliki sistem operasi yang menyediakan layanan-layanan berikut.

Kita bisa menggunakan :

```Rust
#![no_std]
```

Atribut ini memberitahu compiler bahwa program tidak menggunakan Rust Standard Library.
Sebagai gantinya, kita bisa menggunakan `core`, yaitu library fundamental Rust yang tidak bergantung pada sistem operasi.
Contoh:

```rust
#![no_std]

use core::panic::PanifInfo;
```

Perbedaan:

| Library | Keterangan                                                             |
| ------- | ---------------------------------------------------------------------- |
| std     | Standard Library yang memiliki fasilitas sistem operasi                |
| core    | Fasilitas fundamental Rust yang tidak bergantung pada OS               |
| alloc   | Fasilitas alokasi dinamis yang dapat digunakan jika tersedia allocator |

## Stack Unwinding dan Panic Handler

Secara default, Rust menggunakan mekanisme stack unwinding ketika terjadi panic.
Stack unwinding merupakan proses menelusuri kembali call stack dan menjalankan destructor dari variabel-variabel yang masih aktif.
Destructor bertanggung jawab membersihkan resource, misalnya melalui trait Drop.
Contohnya:

```rust
struct MyResource;

impl Drop for MyResource {
  fn drop(&mut self) {
    println!("Resource cleaned up")
  }
}
```

Ketika terjadi panic, Rust dapat menjalankan destructor sebelum menghentikan eksekusi atau meneruskan panic ke pemanggil.

Namun, stack unwinding membutuhkan dukungan runtime dan library khusus sistem operasi, misalnya:

- `libundwind` pada linux
- Structured Exception Handling (SEH) pada Windows
  Hal ini menjadi masalah bagi kernel no_std karena kernel tidak memiliki runtime standar tersebut.

**Panic = Abort**

untuk mengatasi hal ini, kita bisa menggunakan strategi:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

Strategi abort membuat program menghentikan eksekusi ketika terjadi panic tanpa melakukan stack unwinding.
Kita juga perlu menyediakan panic handler sendiri:

```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
  loop {}
}
```

#[panic_handler] merupakan implementasi penanganan panic untuk program no_std. Dalam satu executable no_std, harus ada tepat satu implementasi panic handler.

## Entry Point

Kita mungkin berpikir bahwa fungsi main() merupakan fungsi pertama yang dieksekusi ketika program berjalan.

Namun, pada aplikasi Rust biasa, terdapat beberapa tahapan inisialisasi sebelum main() dipanggil.

**Alur Program Rust Biasa**

Ketika menjalankan aplikasi Rust yang menggunakan std, prosesnya kurang lebih:

1. Operating System menjalankan executable.
2. C Runtime (crt0) mempersiapkan lingkungan program.
3. Rust Runtime melakukan inisialisasi.
4. Rust Runtime memanggil fungsi main()

C Runtime bertanggung jawab melakukan beberapa hal, seperti:

- Membuat stack
- Menyiapkan argumen program
- Menyiapkan lingkungan awal aplikasi

Rust Runtime menangani beberapa hal tambahan, seperti:

- Stack overflow protection
- Backtrace ketika terjadi panic

**Mengapa kernel tidak menggunakan main()?**

Pada kernel no_std, kita tidak memiliki Rust Runtime maupun C Runtime (crt0).
Oleh karena itu, kita harus menentukan sendiri entry point program.
Contohnya:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
  loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
  loop {}
}
```

## Memahami Atribut dan Fungsi Entry Point

**A. #![no_main]**

menonaktifkan mekanisme entry point standar Rust yang menggunakan fungsi main().
Kita akan menentukan entry point sendiri.

**B. #[unsafe(no_mangle)]**

Secara default, Rust menggunakan mekanisme name mangling. Name mangling adalah proses mengubah nama fungsi menjadi simbol unik.

Misalnya:

```
_start
```

Dapat diubah menjadi simbol seperti:

```
_ZN3blog_os4_start7hb173fedf945531caE
```

Hal ini dilakukan agar compiler dapat membedakan simbol-simbol yang berasal dari module, generic, dan konteks lainnya.

Namun linker membutuhkan nama simbol _start agar dapat menemukan entry point yang kita tentukan.

Dengan menggunakan:

```
#[unsafe(no_mangle)]
```

Kita meminta compiler untuk mempertahankan nama simbol _start.

**C. extern "C"**

```
pub extern "C" fn _start() -> !
```

Atribut extern "C" memberitahu compiler menggunakan C calling convention.

Calling convention merupakan aturan tentang bagaimana fungsi dipanggil dan bagaimana data dipertukarkan antara pemanggil dan fungsi.

Aturan tersebut mencakup:

- Pengiriman argumen melalui register atau stack
- Pengembalian nilai fungsi
- Penggunaan register
- Pengelolaan stack

Kita menggunakan "C" agar fungsi memiliki calling convention yang sesuai dengan ABI C, bukan Rust ABI yang tidak dijamin stabil.

**D -> ! (Never Type)**
Tipe ! disebut Never Type.
Artinya, fungsi tersebut tidak akan pernah mengembalikan kontrol kepada pemanggilannya.
Contoh:

```rust
fn infinite_loop() -> ! {
  loop {}
}
```

Entry point kernel tidak memiliki fungsi pemanggil normal yang menerima kembali kontrol eksekusinya.
Karena itu, fungsi tersebut harus terus berjalan atau melakukan mekanisme penghentian sistem. Untuk sementara, kita menggunakan:

```
loop {}
```

## Apa itu Linker?

Linker adalah program yang bertugas menghubungkan berbagai object file dan library untuk menghasilkan executable.

Proses kompilasi secara sederhana:

```

Rust Source Code
       |
       v
   rustc
       |
       v
   Object Code
       |
       v
    Linker
       |
       v
   Executable
```

Perbedaan compiler dan linker:

| Compiler                                 | Linker                               |
| ---------------------------------------- | ------------------------------------ |
| Mengubah source code menjadi object code | Menghubungkan object code dan simbol |
| Memeriksa sintaks dan tipe data          | Menyelesaikan referensi antar simbol |
| Menghasilkan kode mesin                  | Menghasilkan executable              |

**Symbol**
Symbol adalah identitas yang digunakan untuk merujuk pada fungsi atau variabel dalam object file. Contohnya

```
main
add
_start
```

Linker menggunakan simbol untuk menemukan fungsi dan menghubungkan referensi antar object file. Dalam pengembangan kernel, linker perlu mengetahui simbol entry point yang akan digunakan. Itulah alasan kita mempertahankan nama _start menggunakan #[unsafe(no_mangle)]

## Mengapa Terjadi Linker Error?

Ketika kita menjalankan:

```
cargo build --target x86_64-unknown-linux-gnu
```

Rust menggunakan target Linux biasa. Target tersebut mengasumsikan bahwa executables akan berjalan sebagai aplikasi Linux normal. Akibatnya, linker menyertakan C Runtime startup file, salah satunya:

```
Scrt1.o
```

File tersebut sudah menyediakan simbol _start milik Linux.
Sementara itu, kita juga mendefinisikan:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
  loop {}
}
```

Akibatnya terdapat dua simbol _start.

Error yang muncul:

```
rust-lld: error: duplicate symbol: _start

defined at Scrt1.o: (_start)
defined at main.rs:7
```

Artinya:

- Scrt1.o menyediakan _start milik Linux.
- main.rs menyediakan _start miliki kernel kita.

Keduanya bertabrakan ketika proses linking berlangsung.

**Solusi sementara: -nostartfiles**
Kita bisa memberi tahu linker agar tidak menggunakan startup files standar.
Command:

```
cargo rustc -- -C link-arg=-nostartfiles
```

Penjelasan:

| Argumen       | Fungsi                                             |
| ------------- | -------------------------------------------------- |
| cargo rustc   | Menjalankan Cargo dengan opsi tambahan untuk rustc |
| -C            | Mengatur compiler option                           |
| link-arg      | Mengirim argumen kepada linker                     |
| -nostartfiles | Tidak menggunakan startup files standar            |

Dengan demikian, linker tidak lagi menggunakan _start dari Scrt1.o.

Namun, pendekatan ini hanya solusi alternatif untuk host Linux, bukan pendekatan utama yang akan digunakan untuk kernel.

## Host System, Target System, dan Target Triple

**A. Host System**
Host system adalah lingkungan komputer tempat kita melakukan kompilasi.
Untuk melihat informasi host:

```
rustc --version --verbose
```

Contoh output:

```
rustc 1.98.0
host: x86_64-unknown-linux-gpu
```

Artinya:

- x86_64: Arsitektur CPU
- unknown: Vendor
- linux: Sistem operasi
- gnu: ABI

**B. Target System**
Target system adalah lingkungan tempat executable yang dihasilkan akan dijalankan. Secara default, Rust mencoba mengompilasi program untuk host system. Namun, kita bisa menentukan target yang berbeda menggunakan:

```
cargo build --target TARGET_TRIPLE
```

**C. Cross Compilation**
Cross compilation adalah proses mengompilasi program untuk lingkungan target yang berbeda dari host system.

Contohnya:

```
Host:
Ubuntu Linux x86_64

Target:
Bare Metal x86_64
```

Kita menggunakan komputer Ubuntu untuk mengompilasi program yang nantinya akan berjalan tanpa sistem operasi yang mendasarinya.

**D. Target Bare Metal**
Salah satu contoh target embedded ARM:

```
thumbv7em-none-eabihf
```

Kita dapat menambahkan target tersebut menggunakan:

```
rustup target add thumbv7em-none-eabihf
```

Kemudian:

```
cargo build --target thumbv7em-none-eabihf
```

Target ini tidak memiliki sistem operasi yang mendasarinya.
Namun, karena proyek kita menggunakan arsitektur x86_64, kita tidak menggunakan target ARM tersebut.

Sebagai alternatif awal untuk kernel x86_64, kita dapat menggunakan:

```
x86_64-unknown-none
```

Instalasi:

```
rustup target add x86_64-unknown-none
```

Build:

```
cargo build --target x86_64-unknown-none
```

Target tersebut ditujukan untuk lingkungan freestanding x86_64.

Untuk kernel sungguhan, kita tetap membutuhkan konfigurasi linker, bootloader, dan mekanisme boot yang sesuai.

## Dua arti Target dalam Cargo

Cargo menggunakan istilah target dalam 2 konteks berbeda.
**A. Compilation Target (--target)**
Contoh:

```
cargo build --target x86_64-unknown-none
```

**B. Package Target (--all-targets)**
Menentukan jenis artefak dalam package Cargo yang ingin dibangun atau diperiksa.

Contohnya:

| Package Target | Fungsi                      |
| -------------- | --------------------------- |
| Library        | Membangun library           |
| Binary         | Membangun executable        |
| Examples       | Membangun contoh program    |
| Tests          | Membangun program pengujian |
| Benchmarks     | Membangun program benchmark |

Contoh:

```
cargo check --all-targets
```

Perintah tersebut meminta Cargo memeriksa seluruh package targets.

**Penting**: --target dan --all-targets memiliki fungsi berbeda.

## rust-analyzer dan Duplicate panic_impl

rust-analyzer adalah tool yang menyediakan fitur bantuan pemrograman Rust di editor, seperti:

- Autocomplete
- Go to definition
- Find references
- Type checking
- Error diagnostics
  Salah satu fiturnya adalah checkOnSave.

Ketika fitur ini aktif, rust-analyzer secara default dapat menjalankan:

```
cargo check --all-targets
```

Hal ini dapat menyebabkan masalah pada kernel no_std.

**Mengapa terjadi duplicate panic_impl?**

Ketika Cargo membangun binary dalam mode testing, crate test dapat ikut digunakan. Crate test bergantung pada Standard Library (std). Sementara itu, kita sudah menyediakan panic handler sendiri:

```rust
#[panic handler]
fn panic(_info: &PanicInfo) -> ! {
  loop {}
}
```

Akibatnya:

1. std menyediakan implementasi panic_impl
2. Kernel kita juga menyediakan implementasi panic_impl
3. Rust menemukan dua implementasi dalam proses kompilasi yang sama

Error:

```
error[E0152]: found duplicate lang item 'panic_impl'
```

Solusi:
Tambahahkan konfigurasi berikut pada Cargo.toml:

```toml
[[bin]]
name = "rustos"
test = false
bench = false
```

Penjelasan:

- test = false: Binary tidak dibangun dalam mode testing
- bench = false: Binary tidak dibangun dalam mode benchmarking

Nama `rustos` harus sesuai dengan nama binary project.

Dengan demikian, Cargo tidak lagi mencoba membangun binary kernel dalam mode testing atau benchmarking.

Catatan: konfigurasi ini tidak menghilangkan seluruh kemungkinan depedency terhadap std. Kita tetap perlu memastikan depedency kernel sesuai dengan lingkungan no_std.
