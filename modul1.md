<!--
  Halo semuanya! Selamat datang di demo live code hari ini.
  Kita akan membahas dasar-dasar HTML dan CSS styling.
  Pertama, kita mulai dengan deklarasi DOCTYPE untuk memberi tahu browser bahwa ini adalah dokumen HTML5.
-->
<!DOCTYPE html>
<html>
  <!--
    Di sini kita menggunakan tag <style> untuk mendefinisikan CSS internal.
    Ini akan mengatur tampilan elemen di seluruh halaman ini.
  -->
  <style>
    /* Kita atur warna latar belakang seluruh halaman menjadi 'powderblue' */
    body {
      background-color: powderblue;
    }
    /* Untuk semua heading 1 (h1), kita ubah font menjadi Courier dan ukurannya 300% */
    h1 {
      font-family: Courier;
      font-size: 300%;
    }
    /* Untuk semua paragraf (p), kita juga gunakan font courier */
    p {
      font-family: courier;
    }
  </style>
  <body>
    <!-- Bagian ini adalah konten utama yang akan terlihat di browser -->

    <!-- Contoh penggunaan heading h1 dan paragraf standar -->
    <h1>This is a heading</h1>
    <p>This is a paragraph.</p>

    <!-- Di sini kita menggunakan 'inline style' untuk mengubah ukuran font paragraf ini secara spesifik -->
    <p style="font-size: 160%;">This is a paragraph.</p>

    <!-- Contoh penggunaan inline style untuk meratakan teks ke tengah (center alignment) -->
    <h1 style="text-align: center;">Centered Heading</h1>
    <p style="text-align: center;">Centered paragraph.</p>

    <!-- Berikut adalah contoh manipulasi warna teks menggunakan inline style -->
    <p>I am normal</p>
    <p style="color: red;">saya berwarna red</p>
    <p style="color: blue;">I saya berwarna blue</p>

    <!-- Mengubah ukuran font menggunakan pixel -->
    <p style="font-size: 36px;"> saya berukuran big</p>

    <!--
      Selanjutnya, kita lihat elemen <blockquote>.
      Browser biasanya akan memberikan indentasi otomatis untuk elemen ini.
    -->
    <p>Browsers usually indent blockquote elements.</p>
    <blockquote>
      For 50 years, WWF has been protecting the future of nature.
      The world's leading conservation organization,
      WWF works in 100 countries and is supported by
      1.2 million members in the United States and
      close to 5 million globally.
    </blockquote>

    <!--
      Terakhir, kita bereksperimen dengan warna latar belakang (background-color)
      pada elemen heading 2 (h2). Perhatikan bagaimana kita bisa menggabungkan properti,
      seperti pada contoh warna biru di mana kita juga mengubah warna teks menjadi putih.
    -->
    <h2 style="background-color: red;">
      Background-color set by using red
    </h2>
    <h2 style="background-color: orange;">
      Background-color set by using orange
    </h2>
    <h2 style="background-color: yellow;">
      Background-color set by using yellow
    </h2>
    <h2 style="background-color: blue; color: white;">
      Background-color set by using blue
    </h2>
    <h2 style="background-color: cyan;">
      Background-color set by using cyan
    </h2>
  </body>
</html>
