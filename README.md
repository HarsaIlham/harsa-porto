# 📋 Implementation Plan — Refactor GET & POST
### Proyek: 2012_Harsa_Modul4 | Windows Forms C#

---

## 🎯 Tujuan

Melakukan refactor method **GET All** dan **POST (Create)** pada proyek Windows Forms
dengan menerapkan arsitektur berlapis (Layered Architecture):

```
UI Layer    → Form1.cs          (hanya event handler & tampilan)
Service     → ProductService.cs (logika HTTP & parsing JSON)
Client      → ApiClient.cs      (konfigurasi HttpClient tunggal)
Config      → AppConfig.cs      (konstanta URL API)
Helper      → InputValidator.cs (validasi input dari user)
```

---

## 📁 File Yang Perlu Dibuat / Dimodifikasi

| Status     | File                          | Keterangan                      |
|------------|-------------------------------|---------------------------------|
| 🆕 BARU   | `AppConfig.cs`                | Menyimpan Base URL API          |
| 🆕 BARU   | `Services/ApiClient.cs`       | Singleton HttpClient            |
| 🆕 BARU   | `Services/ProductService.cs`  | Method GetAllAsync & CreateAsync|
| 🆕 BARU   | `Helpers/InputValidator.cs`   | Validasi input sebelum POST     |
| ✏️ UBAH   | `Form1.cs`                    | Hapus logika HTTP, pakai service|

---

## ✅ Langkah-Langkah Implementasi

---

### LANGKAH 1 — Buat `AppConfig.cs`
📍 Lokasi: Root project (sejajar dengan `Form1.cs`)

> ⚠️ **WAJIB dibuat PERTAMA** sebelum file lain, karena ApiClient.cs bergantung pada class ini.

```csharp
namespace _2012_Harsa_Modul4
{
    public static class AppConfig
    {
        public const string BaseUrl = "https://praktikum-paa.vercel.app/api/";
    }
}
```

**Ceklis setelah selesai:**
- [ ] File `AppConfig.cs` sudah ada di root project
- [ ] Namespace sesuai dengan project (`_2012_Harsa_Modul4`)
- [ ] Build project → tidak ada error

---

### LANGKAH 2 — Buat Folder & File `Services/ApiClient.cs`
📍 Lokasi: Buat folder `Services` di dalam project, lalu buat file `ApiClient.cs` di dalamnya

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;

namespace _2012_Harsa_Modul4.Services
{
    public static class ApiClient
    {
        private static readonly HttpClient _instance;

        // Static constructor — hanya dijalankan SEKALI saat pertama kali class dipanggil
        static ApiClient()
        {
            _instance = new HttpClient
            {
                BaseAddress = new Uri(AppConfig.BaseUrl)  // Menggunakan AppConfig dari Langkah 1
            };

            _instance.DefaultRequestHeaders.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/json")
            );
        }

        // Properti untuk mengakses instance HttpClient dari luar class
        public static HttpClient Instance => _instance;
    }
}
```

**Penjelasan Kunci:**
- `static ApiClient()` → Static Constructor. Kode di dalamnya hanya berjalan SATU KALI
  selama aplikasi berjalan, memastikan tidak ada HttpClient baru yang dibuat berulang kali.
- `public static HttpClient Instance` → "Pintu masuk" untuk mengambil HttpClient
  yang sudah dikonfigurasi dari class lain.

**Ceklis setelah selesai:**
- [ ] Folder `Services` sudah ada
- [ ] File `ApiClient.cs` sudah ada di dalam folder `Services`
- [ ] Namespace menggunakan `_2012_Harsa_Modul4.Services`
- [ ] Build project → tidak ada error pada file ini

---

### LANGKAH 3 — Buat `Services/ProductService.cs`
📍 Lokasi: Di dalam folder `Services` (sejajar dengan `ApiClient.cs`)

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

namespace _2012_Harsa_Modul4.Services
{
    public class ProductService
    {
        // Menggunakan HttpClient tunggal dari ApiClient
        private readonly HttpClient _client = ApiClient.Instance;

        // Konstanta endpoint — jika berubah, cukup ubah di sini saja
        private const string Endpoint = "produk";

        // ─────────────────────────────────────────────
        // METHOD 1: GET ALL (Mengambil semua data produk)
        // ─────────────────────────────────────────────
        public async Task<List<Product>> GetAllAsync()
        {
            try
            {
                // Kirim HTTP GET ke: https://praktikum-paa.vercel.app/api/produk
                HttpResponseMessage response = await _client.GetAsync(Endpoint);

                // Jika status bukan 2xx (misal 404/500), lempar Exception secara otomatis
                response.EnsureSuccessStatusCode();

                // Baca isi respons sebagai string
                string jsonResponse = await response.Content.ReadAsStringAsync();

                // Parse JSON dan ambil properti "data" saja
                var jsonObject = JObject.Parse(jsonResponse);
                var dataJson = jsonObject["data"].ToString();

                // Ubah JSON string menjadi List<Product> C#
                return JsonConvert.DeserializeObject<List<Product>>(dataJson);
            }
            catch (Exception ex)
            {
                // Lempar ulang exception agar Form yang menangani pesan error-nya
                throw new Exception($"Gagal mengambil data produk: {ex.Message}");
            }
        }

        // ─────────────────────────────────────────────
        // METHOD 2: POST / CREATE (Menambahkan produk baru)
        // ─────────────────────────────────────────────
        public async Task<bool> CreateAsync(Product newProduct)
        {
            try
            {
                // Ubah objek Product C# menjadi string JSON
                string json = JsonConvert.SerializeObject(newProduct);

                // Bungkus ke dalam HTTP Body dengan tipe konten application/json
                var data = new StringContent(json, Encoding.UTF8, "application/json");

                // Kirim HTTP POST ke: https://praktikum-paa.vercel.app/api/produk
                HttpResponseMessage response = await _client.PostAsync(Endpoint, data);

                // Jika status bukan 2xx, lempar Exception
                response.EnsureSuccessStatusCode();

                return true; // Sukses
            }
            catch (Exception ex)
            {
                throw new Exception($"Gagal menambahkan produk: {ex.Message}");
            }
        }
    }
}
```

**Ceklis setelah selesai:**
- [ ] File `ProductService.cs` sudah ada di folder `Services`
- [ ] Terdapat method `GetAllAsync()` dan `CreateAsync()`
- [ ] Build project → tidak ada error

---

### LANGKAH 4 — Buat Folder & File `Helpers/InputValidator.cs`
📍 Lokasi: Buat folder `Helpers`, lalu buat file `InputValidator.cs` di dalamnya

```csharp
namespace _2012_Harsa_Modul4.Helpers
{
    public static class InputValidator
    {
        /// <summary>
        /// Memvalidasi input form produk sebelum dikirim ke API.
        /// </summary>
        /// <returns>
        /// String pesan error jika ada input yang tidak valid.
        /// NULL jika semua input valid.
        /// </returns>
        public static string ValidateProductInput(string nameText, string priceText, string stockText)
        {
            // Cek apakah nama kosong atau hanya spasi
            if (string.IsNullOrWhiteSpace(nameText))
                return "Nama produk tidak boleh kosong!";

            // TryParse aman digunakan — tidak crash jika input bukan angka
            // Parameter "out _" artinya kita hanya peduli berhasil atau tidak, bukan nilainya
            if (!float.TryParse(priceText, out _))
                return "Harga harus berisi angka yang valid! (Contoh: 5000 atau 5000.5)";

            if (!int.TryParse(stockText, out _))
                return "Stok harus berisi angka bulat! (Contoh: 10)";

            return null; // null berarti SEMUA INPUT VALID
        }
    }
}
```

**Ceklis setelah selesai:**
- [ ] Folder `Helpers` sudah ada
- [ ] File `InputValidator.cs` sudah ada di dalam folder `Helpers`
- [ ] Build project → tidak ada error

---

### LANGKAH 5 — Modifikasi `Form1.cs`
📍 Lokasi: Root project (file yang sudah ada)

Ganti **seluruh isi** `Form1.cs` dengan kode berikut:

```csharp
using System;
using System.Collections.Generic;
using System.Windows.Forms;
using _2012_Harsa_Modul4.Services;  // Import ProductService & ApiClient
using _2012_Harsa_Modul4.Helpers;   // Import InputValidator

namespace _2012_Harsa_Modul4
{
    public partial class Form1 : Form
    {
        // Inisialisasi ProductService — tidak ada lagi HttpClient di sini!
        private readonly ProductService _productService = new ProductService();

        public Form1()
        {
            InitializeComponent();
            // Hapus baris client.BaseAddress & client.DefaultRequestHeaders yang lama
        }

        // ─────────────────────────────────────────────
        // TOMBOL FETCH — GET ALL DATA
        // ─────────────────────────────────────────────
        private async void btnFetch_Click(object sender, EventArgs e)
        {
            try
            {
                // Cukup panggil service, tidak perlu tahu cara kerjanya
                List<Product> products = await _productService.GetAllAsync();
                dataGridView1.DataSource = products;
            }
            catch (Exception ex)
            {
                MessageBox.Show(ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        // ─────────────────────────────────────────────
        // TOMBOL POST — TAMBAH DATA BARU
        // ─────────────────────────────────────────────
        private async void btnPost_Click(object sender, EventArgs e)
        {
            // LANGKAH 1: Validasi input lebih dulu
            string errorMessage = InputValidator.ValidateProductInput(
                txtProduk.Text,
                txtPrice.Text,
                txtStock.Text
            );

            // Jika ada error validasi, tampilkan dan hentikan proses
            if (errorMessage != null)
            {
                MessageBox.Show(errorMessage, "Input Tidak Valid",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            // LANGKAH 2: Karena sudah divalidasi, Parse dijamin aman
            var newProduct = new Product
            {
                Name  = txtProduk.Text,
                Price = float.Parse(txtPrice.Text),
                Stock = int.Parse(txtStock.Text)
            };

            // LANGKAH 3: Kirim ke API via ProductService
            try
            {
                bool isSuccess = await _productService.CreateAsync(newProduct);

                if (isSuccess)
                {
                    MessageBox.Show("Produk berhasil ditambahkan!", "Sukses",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    // Refresh grid otomatis
                    btnFetch.PerformClick();

                    // Kosongkan input form
                    txtProduk.Text = "";
                    txtPrice.Text  = "";
                    txtStock.Text  = "";
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show(ex.Message, "Error API",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        // ─────────────────────────────────────────────
        // TOMBOL FIND BY ID — GET DATA BY ID
        // ─────────────────────────────────────────────
        private async void btnFind_Click(object sender, EventArgs e)
        {
            // Catatan: Method GetByIdAsync belum diimplementasikan di plan ini
            // Logika sementara bisa tetap seperti semula
        }

        // ─────────────────────────────────────────────
        // TOMBOL KE FORM UPDATE
        // ─────────────────────────────────────────────
        private void FormUpdate_Click(object sender, EventArgs e)
        {
            this.Hide();
            FormUpdate formUpdate = new FormUpdate();
            formUpdate.FormClosed += (s, args) => this.Show();
            formUpdate.Show();
        }
    }
}
```

**Ceklis setelah selesai:**
- [ ] Tidak ada lagi deklarasi `HttpClient` di `Form1.cs`
- [ ] `btnFetch_Click` hanya memanggil `_productService.GetAllAsync()`
- [ ] `btnPost_Click` memanggil `InputValidator` sebelum `_productService.CreateAsync()`
- [ ] Build project → tidak ada error

---

## 🧪 Cara Uji Implementasi

### Uji GET All
1. Jalankan aplikasi (F5)
2. Klik tombol **Fetch**
3. ✅ DataGridView harus menampilkan daftar produk dari API

### Uji POST (Input Valid)
1. Isi `txtProduk` dengan nama produk (contoh: `"Mangga"`)
2. Isi `txtPrice` dengan angka (contoh: `4000`)
3. Isi `txtStock` dengan angka bulat (contoh: `20`)
4. Klik tombol **Post/Tambah**
5. ✅ Muncul MessageBox "Produk berhasil ditambahkan!"
6. ✅ DataGridView otomatis refresh dan menampilkan data baru

### Uji POST (Input Tidak Valid — Validasi)
1. Kosongkan semua field, klik **Post/Tambah**
   → ✅ Muncul warning "Nama produk tidak boleh kosong!"
2. Isi nama, tapi isi harga dengan huruf (contoh: `"abc"`), klik **Post/Tambah**
   → ✅ Muncul warning "Harga harus berisi angka yang valid!"
3. Isi nama & harga dengan benar, tapi stok diisi huruf, klik **Post/Tambah**
   → ✅ Muncul warning "Stok harus berisi angka bulat!"

---

## 📊 Struktur Folder Akhir

```
📁 2012_Harsa_Modul4/
 ├── 📁 Helpers/
 │    └── InputValidator.cs      ✅ BARU
 │
 ├── 📁 Services/
 │    ├── ApiClient.cs           ✅ BARU
 │    └── ProductService.cs      ✅ BARU
 │
 ├── 📄 AppConfig.cs             ✅ BARU
 ├── 📄 Form1.cs                 ✏️ DIMODIFIKASI
 ├── 📄 Product.cs               (Tidak diubah)
 ├── 📄 FormUpdate.cs            (Tidak diubah pada plan ini)
 └── 📄 Program.cs               (Tidak diubah)
```

---

*Dokumen ini dibuat sebagai panduan implementasi refactor GET & POST.*
*Ikuti langkah secara berurutan untuk menghindari error dependency.*
