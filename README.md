Galeri Kita - Panduan Pemasangan
Ini adalah panduan untuk menghubungkan website index.html Anda dengan Google Apps Script (GAS) agar berfungsi sebagai backend penyimpanan file di Google Drive.
PENTING: Proyek ini menggunakan metode pengiriman file via Base64 ke Google Apps Script. Metode ini memiliki batasan ukuran file (biasanya sekitar 30-40 MB) karena batasan GAS. Ini cukup untuk foto dan file musik kecil, tetapi tidak akan berfungsi untuk file video besar.
Langkah 1: Siapkan Folder Google Drive
 * Buka Google Drive.
 * Buat folder baru. Beri nama apa saja, misalnya "GaleriWebsiteKita".
 * Buka folder tersebut. Lihatlah URL di browser Anda.
 * URL akan terlihat seperti: https://drive.google.com/drive/folders/1aBcDeFgHiJkLmNoPqRsTuVwXyZ
 * Salin bagian ID unik setelah folders/. Pada contoh di atas, ID-nya adalah 1aBcDeFgHiJkLmNoPqRsTuVwXyZ.
 * Simpan ID Folder ini. Anda akan membutuhkannya.
Langkah 2: Buat Google Apps Script (Backend)
 * Buka Google Apps Script.
 * Klik "Proyek baru".
 * Hapus semua kode yang ada di Code.gs.
 * Salin dan tempel seluruh kode di bawah ini ke dalam file Code.gs:
   // --- KONFIGURASI ---
// Ganti dengan ID Folder Google Drive yang Anda salin di Langkah 1
const DRIVE_FOLDER_ID = "YOUR_DRIVE_FOLDER_ID_HERE"; 
// -------------------

/**
 * Fungsi utama untuk menangani request POST (upload file).
 * Menerima payload JSON berisi: { filename, mimeType, data (base64) }
 */
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);

    const { filename, mimeType, data: base64Data } = data;

    if (!filename || !mimeType || !base64Data) {
      throw new Error("Payload tidak lengkap (filename, mimeType, data).");
    }

    const folder = DriveApp.getFolderById(DRIVE_FOLDER_ID);
    const blob = Utilities.newBlob(Utilities.base64Decode(base64Data), mimeType, filename);
    const file = folder.createFile(blob);

    // Mengembalikan respons sukses
    return ContentService
      .createTextOutput(JSON.stringify({ 
        success: true, 
        fileId: file.getId(), 
        name: file.getName() 
      }))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    Logger.log(error); // Catat error untuk debugging
    // Mengembalikan respons error
    return ContentService
      .createTextOutput(JSON.stringify({ 
        success: false, 
        error: error.message 
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

/**
 * Fungsi utama untuk menangani request GET (ambil daftar file, ambil info penyimpanan).
 * Menggunakan parameter URL: ?action=getFiles atau ?action=getStorage
 */
function doGet(e) {
  try {
    // Tentukan aksi berdasarkan parameter, default-nya 'getFiles'
    const action = e.parameter.action || 'getFiles';

    let responseData;

    if (action === 'getFiles') {
      responseData = getFilesInFolder();
    } else if (action === 'getStorage') {
      responseData = getStorageInfo();
    } else {
      throw new Error("Aksi tidak valid. Gunakan 'getFiles' atau 'getStorage'.");
    }

    // Mengembalikan data JSON
    return ContentService
      .createTextOutput(JSON.stringify(responseData))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    Logger.log(error); // Catat error untuk debugging
    // Mengembalikan respons error
    return ContentService
      .createTextOutput(JSON.stringify({ 
        success: false, 
        error: error.message 
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

/**
 * Fungsi helper untuk mengambil semua file di folder target
 */
function getFilesInFolder() {
  const folder = DriveApp.getFolderById(DRIVE_FOLDER_ID);
  const files = folder.getFiles();
  const fileList = [];

  while (files.hasNext()) {
    const file = files.next();

    // PENTING: Atur file agar bisa dilihat oleh "siapa saja dengan link"
    // Ini WAJIB agar file bisa di-preview (di-embed) di HTML
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);

    fileList.push({
      id: file.getId(),
      name: file.getName(),
      type: file.getMimeType(),
      size: file.getSize(),
      // webViewLink: Link untuk membuka di UI Google Drive
      previewUrl: file.getWebViewLink(), 
      // webContentLink: Link untuk download langsung / embed
      downloadUrl: file.getWebContentLink() 
    });
  }

  // Balik urutan agar file terbaru (yang baru di-upload) muncul di atas
  return fileList.reverse(); 
}

/**
 * Fungsi helper untuk mengambil info penyimpanan Google Drive
 */
function getStorageInfo() {
  const used = DriveApp.getStorageUsed(); // Byte terpakai
  const limit = DriveApp.getStorageLimit(); // Byte total
  return { 
    used, 
    limit, 
    available: (limit > 0) ? (limit - used) : 0 
  };
}

 * PENTING: Ganti YOUR_DRIVE_FOLDER_ID_HERE di baris ke-3 dengan ID Folder yang Anda salin di Langkah 1.
 * Beri nama proyek Anda (misal: "Backend Galeri"). Simpan proyek.
Langkah 3: Deploy Google Apps Script
 * Di editor Apps Script, klik tombol "Terapkan" (Deploy) di kanan atas.
 * Pilih "Deployment baru" (New deployment).
 * Di sebelah "Pilih jenis", klik ikon roda gigi dan pilih "Aplikasi Web" (Web app).
 * Isi deskripsi, misalnya "Backend Galeri v1".
 * Untuk "Jalankan sebagai" (Execute as), pilih "Saya" (Me). (Ini sangat penting!)
 * Untuk "Siapa yang memiliki akses" (Who has access), pilih "Siapa saja" (Anyone). (Ini penting agar file HTML bisa memanggilnya).
 * Klik "Terapkan" (Deploy).
 * PENTING: Otorisasi. Google akan meminta Anda untuk meninjau izin.
   * Klik "Tinjau izin".
   * Pilih akun Google Anda.
   * Anda mungkin melihat peringatan "Google belum memverifikasi aplikasi ini". Ini normal.
   * Klik "Lanjutan" (Advanced), lalu klik "Buka [nama proyek Anda] (tidak aman)" (Go to ... (unsafe)).
   * Klik "Izinkan" (Allow) untuk memberi skrip Anda akses ke Google Drive Anda.
 * Setelah otorisasi, Google akan menampilkan URL Aplikasi Web. Salin URL ini. URL ini terlihat seperti https://script.google.com/macros/s/..../exec.
Langkah 4: Konfigurasi File index.html
 * Buka file index.html yang saya berikan.
 * Cari baris ini di bagian bawah, di dalam tag <script>:
   const GAS_WEB_APP_URL = "YOUR_GAS_WEB_APP_URL_HERE"; 

 * Ganti YOUR_GAS_WEB_APP_URL_HERE dengan URL Aplikasi Web yang Anda salin di Langkah 3. Pastikan URL-nya ada di dalam tanda kutip "".
 * Simpan file index.html.
Selesai!
Sekarang Anda bisa membuka file index.html di browser Anda (cukup klik dua kali). Website akan memuat, mengambil data dari Google Drive Anda, dan siap menerima upload file.




live at: https://ardyproject.github.io/KingProject/
