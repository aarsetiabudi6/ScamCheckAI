# I-O-Modal-MoneyMate
Teman catatan cash‑flow multimodal

<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/8d513177-136b-4ed1-b7d7-95081653f3a5" />

# Tools
## 1️⃣ Buat Bot Telegram 🤖✉️
1. Buka aplikasi Telegram, cari @BotFather, lalu /start.
2. Kirim /newbot, berikan nama dan username berakhiran bot.
3. BotFather memberi API token: 123456:ABC-DEF…. Simpan sebagai BOT_TOKEN. 
4. Di n8n → Credentials → Telegram → tempel token.

## 2️⃣ Aktifkan Speech‑to‑Text API 🎤
1. Masuk Google Cloud Console → pilih/buat Project, aktifkan billing. 
2. APIs & Services → Enable API → cari Speech‑to‑Text API → Enable.
3. Credentials → Create credentials → API key → salin AIza… → GCP_API_KEY.

## 3️⃣  Autentikasi Google Sheets 📄  

**Pilih salah satu:**

<details>
<summary>🚀 Jalur A — Login Sekali Klik (n8n.cloud/pre‑configured)</summary>

1. Buka *n8n → Credentials → Google Sheets*.  
2. Tipe: **OAuth2 (Google pre‑configured)**.  
3. Klik **Connect OAuth2 Account** → login Google, pilih email, `Allow`.  
4. Selesai — token otomatis tersimpan.

</details>

<details>
<summary>🔧 Jalur B — Custom Client ID & Secret (Self‑host kustom)</summary>

1. Google Cloud Console → *APIs & Services → Credentials* → **Create OAuth Client ID**.  
2. Redirect URI: `https://<domain‑n8n>/rest/oauth2-credential/callback`.  
3. Salin **Client ID** & **Client Secret**.  
4. n8n → Credentials → **Google Sheets** → **Generic OAuth2** → isi ID & Secret → **Connect**.

</details>

## 4️⃣ API Key AI Gemini 🤖
1. Kunjungi Google AI Studio → Get API key → beri nama → Create Key.
2. Salin key AIzaSy… → n8n → Credentials → Google Gemini Chat Model.

# Step
##  1️⃣ Optical Character Recognition 📸
> **Goal:** Ambil foto/scan struk via Telegram
<img width="896" height="234" alt="image" src="https://github.com/user-attachments/assets/b4c8fcb8-20bc-4cf5-9958-a6c01f2f56d4" />

---

## 🔧 Node Detail

| # | Node | Fungsi | Catatan |
|---|------|--------|---------|
| 1 | **Telegram – get file** | Mengunduh lampiran (photo, doc, scan) | Credential: `BOT_TOKEN` |
| 2 | **Code** | Pastikan `mimeType` konsisten (`image/jpeg`) | Snippet di bawah |

---

> ## 📝 Kode Node “Code”

```javascript
/**
 * Normalisasi binary agar node downstream (Extract File / Vision API)
 * selalu mengenali file sebagai JPEG.
 */
const items = $input.all();

return items.map(i => {
  i.binary.data.mimeType = 'image/jpeg';
  return i;
});
```

## 2️⃣ Extract from File 📰
> **Goal:** Setelah gambar/scan masuk (Step 1), node ini mengekstrak
> konten teks dari PDF/JPEG lalu men‑set struktur JSON simpel agar
> siap di‑parse AI Gemini pada workflow berikutnya.
<img width="751" height="182" alt="image" src="https://github.com/user-attachments/assets/d13ab440-f74d-474e-ac1c-806ea3856169" />

## 🔧 Konfigurasi Node
| Urut | Node | Fungsi | Credential | Output |
|------|------|--------|------------|--------|
| 1 | **Telegram1 – get file** | Unduh lampiran (foto/PDF) | Bot Token | `binary.data` |
| 2 | **Extract from File** | OCR + parsing PDF → String | — | `json.text` |
| 3 | **Edit Fields1** | Rapikan / mapping field | — | `json` clean |
### 1. Telegram1 – get file

- **Resource** : `file`
- **File ID** : default dari Telegram Trigger (`{{$json.message.document.file_id}}` atau `photo[0].file_id`)

### 2. Extract from File 📰

| Parameter | Nilai | 
|-----------|--------------------|
| **Operation** | `Extract From PDF` | 
| **Input  Fields** | **data** | 

### 3. Edit Fields1  📰

| Parameter | Nilai | 
|-----------|--------------------|
| **Mode** | `Manual Maping` | 
| **Fields to Set** | `Text` `String`  |

Snippet ekspresi yang dipakai:

```n8n
{{ $json["text"] }}
```
# 3️⃣ Speech Recognition 🎤
> Lane ini menangani **voice note Telegram** → **transkrip teks** memakai
> Google Speech‑to‑Text API.  Output transkrip akan dikirim ke AI Gemini

<img width="616" height="162" alt="image" src="https://github.com/user-attachments/assets/a1602dc5-c347-4ba3-950c-8a596c09345e" />


| Field               | Nilai Contoh                                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Method**          | `POST`                                                                                                                                |
| **URL**             | `https://speech.googleapis.com/v1/speech:recognize?key={{ $env.GCP_API_KEY }}`<br>*(hapus `?key=` jika memakai Service Account Auth)* |
| **Authentication**  | `None` **atau** `Google Service Account`                                                                                              |
| **Headers**         | `Content-Type: application/json`                                                                                                      |
| **Body** (raw JSON) | 

``` JSON
{
  "config": {
    "encoding": "OGG_OPUS",
    "sampleRateHertz": 48000,
    "languageCode": "id-ID",
    "enableAutomaticPunctuation": true
  },
  "audio": {
    "content": "={{ $json.data }}"
  }
}
```

# 4️⃣ Text Processing ✏️                             
> Node sederhana ini **menyamakan format** teks dari dua jalur
> sebelumnya (OCR 📸 dan Speech 🎤).  
> Tujuannya: **AI Gemini** hanya perlu membaca satu field —`json.text`
<img width="529" height="134" alt="image" src="https://github.com/user-attachments/assets/731325a2-2ad2-46da-a4df-74c469c3f129" />

###  Edit Fields  📰

| Parameter | Nilai | 
|-----------|--------------------|
| **Mode** | `Manual Maping` | 
| **Fields to Set** | `Text` `String`  |

Snippet ekspresi yang dipakai:

```n8n
{{ $json["text"] }}
```

# 5️⃣ AI Agent 🤖  Gemini Chat Model + Google Sheets Tool
> Di tahap ini **teks bersih** (hasil OCR / Speech) diproses oleh
> **Gemini Chat Model** untuk di‑parse menjadi **tabel keuangan** lalu
> **Append Row** ke Google Sheets secara otomatis.  
> Node **AI Agent** bertindak sebagai “otak” yang memanggil dua sub‑tool:
>
> 1. **Think** – internal reasoning (tidak memanggil API apa pun)  
> 2. **Append Row in Google Sheets** – eksekusi penulisan data

<img width="411" height="358" alt="image" src="https://github.com/user-attachments/assets/1849f61a-8e54-4dc4-9979-e019938f4d0c" />

| Field              | Nilai Contoh                                    | Penjelasan                                            |
| ------------------ | ----------------------------------------------- | ----------------------------------------------------- |
| **Model**          | `Google Gemini Chat Model`                      | pilih credential `gemini-prod`                        |
| **System Message** | *Lihat blok di bawah*                           | Menetapkan role “asisten keuangan”                    |
| **User Message**   | *Prompt dinamis*                                | Mengandung hasil OCR / transkrip dan instruksi tabel  |
| **Tools**          | ✓ **AppendRow in Google Sheets**<br>✓ **Think** | Agent boleh memanggil dua tool ini                    |
| **Memory**         | *Opsional*                                      | biarkan default jika tak butuh konteks chat berlanjut |

## 📜 System Message
```
Kamu adalah asisten keuangan untuk emalkukan pencatatan pendapatan dan pengeluaran sehingga dapan tercatat rapi dan terstruktur, dari file yg di upload atau input text
```

## 📝 User Message (template)
```
Berikut adalah hasil OCR dari struk dan transkrip suara (jika ada):
"{{ $json.text
    ? $json.text
    : ($json.results
        ? $json.results.map(r => r.alternatives[0].transcript).join(' | ')
        : '') }}"

Tugas Anda:
1. Pisahkan setiap produk ke baris terpisah.
2. Tentukan **no** (jumlah pcs) sebagai angka:
   - Jika ada “(7 pcs)” atau “7 pcs” → no = 7  
   - Jika tidak ada keterangan pcs → no = 1
3. Mapping field tiap baris:

   • **timestamp**   = "{{$now}}"  
   • **tanggal**     = tanggal struk DD/MM/YYYY  
   • **no**          = jumlah pcs (angka)  
   • **keterangan**  = nama produk (benahi typo, sertakan “(7 pcs)” jika ada)  
   • **pengeluaran** = harga satuan × no, format ribuan IDN (mis “21.000”)  
   • **pemasukan**   = kosong kecuali refund/masuk 
4. Jika ada baris pajak terpisah (PPN/VAT) — total item ≠ total bayar — catat sebagai produk “PPN” dengan nilai pajaknya.
5. Gunakan hanya tool **AppendRow** (tunggu 1 detik setelah tiap panggilan) dan **Think** (jika perlu reasoning).
6. Setelah semua baris tercatat, tampilkan **tabel ringkasan Markdown** dengan kolom:  
   |  Tanggal | No| Keterangan | Pengeluaran | Pemasukan |
7. Mulai sekarang, keluarkan **langsung** serangkaian JSON tool calls tanpa penjelasan lain.
```
# 6️⃣ Code + Telegram Reply ✉️

> Setelah **AI Agent** menghasilkan tabel ringkasan (plain‑text),
> node **Code1** di bawah ini:
>
> 1. **Escape** karakter khusus supaya cocok dengan **Markdown V2**  
> 2. **Membungkus** tabel dengan blok code ```  
> 3. Membuat payload JSON untuk node **Telegram3 – sendMessage**
<img width="384" height="153" alt="image" src="https://github.com/user-attachments/assets/29be96d1-d070-4b4d-b22a-6627f8b1be33" />

## ⚙️ Pengaturan Node `Code1`

| Field | Nilai |
|-------|-------|
| **Mode** | `Run Once for All Items` |
| **Language** | `JavaScript` |

### Isi Code (JavaScript)

```javascript
// ------- helper escape markdownV2 -----------------
function esc(s='') {
  return s.replace(/([_*[\]()~`>#\+\-=|{}.!\\])/g, '\\$1');
}

// ── 1) Ambil string tabel dari AI Agent (sudah ada \n)
const tableBody = $input.first().json.output || '';

// ── 2) Bungkus dengan back‑ticks tiga kali
const telegramMessage = '```\\n' + tableBody + '\\n```';

return {
  json: {
    message: telegramMessage,
    parse_mode: 'MarkdownV2'
  }
};
```
telegram replay
| Parameter      | Nilai                 | Catatan                       |
| -------------- | --------------------- | ----------------------------- |
| **Credential** | `Telegram account`    | Token BotFather               |
| **Resource**   | `Message`             |                               |
| **Operation**  | `Send Message`        |                               |
| **Chat ID**    | `{{ $('Telegram Trigger').item.json.message.chat.id }}}`  | Disiapkan di node sebelumnya  |
| **Text**       | `{{ $json.message }}` | Blok code tabel (Markdown V2) |



