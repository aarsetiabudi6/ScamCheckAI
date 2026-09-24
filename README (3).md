# ScamCheck AI

AI Deteksi Scam & Phishing berbasis Telegram, n8n, VirusTotal, dan AI Agent
<img width="956" height="408" alt="image" src="https://github.com/user-attachments/assets/86afd8d1-48d1-4b63-a513-3d8ab30afad1" />


# Tools

## 1️⃣ Buat Bot Telegram 🤖✉️

1. Buka aplikasi Telegram, cari `@BotFather`, lalu `/start`.
2. Kirim `/newbot`.
3. Berikan nama bot.
4. Berikan username yang diakhiri dengan `bot`.
5. BotFather akan memberikan API Token.
6. Simpan token tersebut di n8n → **Credentials → Telegram**.

> ⚠️ Jangan masukkan Bot Token ke dalam repository GitHub.

---

## 2️⃣ VirusTotal API 🔍

1. Buka akun VirusTotal.
2. Masuk ke bagian API.
3. Salin API Key.
4. Simpan API Key sebagai credential atau secret di n8n.
5. API Key digunakan oleh node HTTP Request untuk melakukan pemeriksaan URL.

Header yang digunakan:

```text
x-apikey: YOUR_VIRUSTOTAL_API_KEY
```

> ⚠️ Jangan menuliskan API Key asli di README atau file workflow yang dipublikasikan.

---

## 3️⃣ AI Model 🤖

ScamCheck AI menggunakan AI Agent untuk melakukan analisis terhadap pesan pengguna.

Model yang dapat digunakan:

```text
Google Gemini Chat Model
```

atau untuk penggunaan lokal:

```text
Ollama Chat Model
```

Konfigurasi model dilakukan melalui credential dan node AI Agent di n8n.

---

## 4️⃣ n8n ⚙️

ScamCheck AI dibuat menggunakan n8n sebagai workflow automation.

Workflow utama:

```text
Telegram
   ↓
Edit Fields
   ↓
Code JavaScript
   ↓
IF
   ↓
VirusTotal / AI Agent
   ↓
Code Scoring
   ↓
AI Agent
   ↓
Telegram Reply
```

# Step

## 1️⃣ Telegram Trigger 🤖✉️
<img width="145" height="107" alt="image" src="https://github.com/user-attachments/assets/5c9514c2-4ec8-4279-85c3-464653a5fb39" />

> Goal: Menerima pesan dari pengguna melalui Telegram.

---

## 🔧 Node Detail

| # | Node | Fungsi | Catatan |
|---|---|---|---|
| 1 | Telegram Trigger | Menerima pesan Telegram | Credential: `Telegram Bot` |
| 2 | Edit Fields | Mengambil dan menyiapkan teks pesan | `chatInput` |
| 3 | Code JavaScript | Mendeteksi URL | Menghasilkan `url` dan `url_id` |
| 4 | IF | Mengecek apakah terdapat URL | True → VirusTotal |
| 5 | HTTP Request1 | Mengirim URL ke VirusTotal | Method `POST` |
| 6 | Wait | Menunggu proses VirusTotal | Delay |
| 7 | HTTP Request | Mengambil hasil VirusTotal | Method `GET` |
| 8 | Code Scoring | Mengolah hasil pemeriksaan | Risk information |
| 9 | AI Agent | Analisis pesan | AI Model |
| 10 | Telegram Send Message | Mengirim hasil ke pengguna | Telegram |

---

## 2️⃣ Edit Fields 📝
<img width="112" height="95" alt="image" src="https://github.com/user-attachments/assets/c6b896f2-6689-4d71-a878-7f2993673459" />

> Goal: Menyamakan format data dari Telegram agar dapat digunakan oleh node berikutnya.

### Parameter

| Parameter | Nilai |
|---|---|
| Mode | `Manual Mapping` |
| Field | `chatInput` |
| Type | `String` |

Contoh expression:

```text
{{ $json.message.text }}
```

Output:

```json
{
  "chatInput": "Tolong cek link ini https://example.com"
}
```

---

## 3️⃣ URL Detection 🔗
<img width="234" height="115" alt="image" src="https://github.com/user-attachments/assets/b7f74961-26bf-4315-b0ff-6abd414e515b" />

> Goal: Mendeteksi URL dari pesan Telegram dan membuat `url_id` untuk digunakan oleh VirusTotal.

### Code Node

| Parameter | Nilai |
|---|---|
| Mode | `Run Once for All Items` |
| Language | `JavaScript` |

### Isi Code

```javascript
const text = $json.chatInput || '';

const match = text.match(/https?:\/\/[^\s]+/i);

const url = match
  ? match[0].replace(/[)\],.]+$/, '')
  : '';

const url_id = url
  ? Buffer.from(url)
      .toString('base64')
      .replace(/=+$/, '')
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
  : '';

return [
  {
    json: {
      ...$json,
      url,
      url_id
    }
  }
];
```

### Output

Contoh:

```json
{
  "chatInput": "Tolong cek https://example.com",
  "url": "https://example.com",
  "url_id": "aHR0cHM6Ly9leGFtcGxlLmNvbQ"
}
```

---

## 4️⃣ IF — URL Check 🔀
<img width="357" height="249" alt="image" src="https://github.com/user-attachments/assets/a52772b4-5e00-4385-8e45-407ee48a5412" />

> Goal: Menentukan apakah pesan pengguna memiliki URL.

Kondisi:

```text
url
is not empty
```

Alur:

```text
                 ┌── TRUE ──→ VirusTotal
                 │
Code JavaScript ─┤
                 │
                 └── FALSE ─→ AI Agent
```

### TRUE

Jika `url` tersedia, workflow melakukan pemeriksaan VirusTotal.

### FALSE

Jika tidak terdapat URL, pesan langsung diteruskan ke AI Agent untuk dianalisis berdasarkan isi pesan.

---

## 5️⃣ VirusTotal URL Analysis 🔍
<img width="197" height="97" alt="image" src="https://github.com/user-attachments/assets/53d94d83-803d-4ade-9a2a-3e23d4ffbda8" />

> Goal: Mengirim URL ke VirusTotal dan mengambil hasil analisis.

### 1. HTTP Request1 — POST

Parameter:

| Parameter | Nilai |
|---|---|
| Method | `POST` |
| URL | `https://www.virustotal.com/api/v3/urls` |
| Authentication | Header |
| Header | `x-apikey` |
| Body | `Form-Data` |

Field:

```text
url
```

Value:

```text
{{ $('Code JavaScript').item.json.url }}
```

---

### 2. Wait ⏳

> Goal: Memberikan waktu kepada VirusTotal untuk memproses analysis.

Alur:

```text
HTTP Request1
      ↓
     Wait
      ↓
HTTP Request
```

---

### 3. HTTP Request — GET

Parameter:

| Parameter | Nilai |
|---|---|
| Method | `GET` |
| URL | `https://www.virustotal.com/api/v3/analyses/{{ $('HTTP Request1').item.json.data.id }}` |
| Header | `x-apikey` |

Expression URL:

```text
https://www.virustotal.com/api/v3/analyses/{{ $('HTTP Request1').item.json.data.id }}
```

Hasil dari node ini kemudian diteruskan ke:

```text
Code Scoring
```

---

## 6️⃣ Code Scoring 📊
<img width="139" height="157" alt="image" src="https://github.com/user-attachments/assets/fc6f99ac-7547-4b9d-9ee7-0494b978bfd2" />

> Goal: Mengolah hasil VirusTotal menjadi informasi yang lebih mudah dipahami AI Agent.

Contoh data VirusTotal:

```json
{
  "malicious": 0,
  "suspicious": 0,
  "harmless": 80
}
```

Contoh output scoring:

```json
{
  "malicious": 0,
  "suspicious": 0,
  "harmless": 80,
  "risk": "LOW"
}
```

Scoring digunakan sebagai informasi tambahan untuk AI Agent.

> Hasil scoring bukan jaminan bahwa sebuah URL aman atau berbahaya. Hasil harus dipertimbangkan bersama isi pesan dan konteks lainnya.

---

# 7️⃣ AI Agent 🤖
<img width="147" height="162" alt="image" src="https://github.com/user-attachments/assets/df8f8af5-05db-457f-8fd9-0241693e652e" />

> Pada tahap ini pesan pengguna dianalisis oleh AI Agent. Jika terdapat URL, hasil pemeriksaan VirusTotal juga digunakan sebagai informasi tambahan.

### Field

| Field | Nilai |
|---|---|
| Model | `Google Gemini Chat Model` / `Ollama Chat Model` |
| System Message | Instruksi ScamCheck AI |
| User Message | Pesan pengguna + hasil pemeriksaan |
| Memory | Opsional |

---

## 📜 System Message

```text
Kamu adalah ScamCheck AI, asisten yang membantu pengguna
menganalisis pesan dan URL yang mencurigakan.

Tugas kamu:

1. Analisis isi pesan.
2. Identifikasi indikator scam atau phishing.
3. Periksa apakah pengguna diminta memberikan data sensitif.
4. Periksa penggunaan urgensi atau tekanan.
5. Periksa tawaran hadiah atau keuntungan yang tidak biasa.
6. Analisis URL jika tersedia.
7. Gunakan hasil VirusTotal jika tersedia.

Jangan menyatakan bahwa sebuah pesan pasti scam hanya
berdasarkan satu indikator.

Jika hasil VirusTotal tidak tersedia atau pemeriksaan URL
gagal, jelaskan keterbatasannya.

Berikan hasil dalam format:

Risiko:
Indikator:
Hasil URL:
Penjelasan:
Rekomendasi:
```

---

## 📝 User Message

Contoh template:

```text
Berikut adalah pesan yang dikirim pengguna:

{{ $('Edit Fields').item.json.chatInput }}

URL:

{{ $('Code JavaScript').item.json.url }}

Hasil pemeriksaan VirusTotal:

{{ JSON.stringify($json) }}

Tugas:

1. Analisis isi pesan.
2. Identifikasi indikator scam atau phishing.
3. Jelaskan hasil pemeriksaan URL jika tersedia.
4. Jelaskan keterbatasan hasil pemeriksaan.
5. Berikan rekomendasi keamanan kepada pengguna.

Gunakan bahasa Indonesia yang sederhana dan mudah dipahami.
```

---

# 8️⃣ Code + Telegram Reply ✉️
<img width="149" height="107" alt="image" src="https://github.com/user-attachments/assets/6e294572-fac5-4c6c-abed-57e2292753d4" />

> Setelah AI Agent menghasilkan analisis, hasil tersebut dikirim kembali kepada pengguna melalui Telegram.

### ⚙️ Pengaturan Node Code

| Field | Nilai |
|---|---|
| Mode | `Run Once for All Items` |
| Language | `JavaScript` |

### Isi Code

```javascript
const output = $input.first().json.output || '';

return {
  json: {
    message: output
  }
};
```

---

## Telegram Send Message

Parameter:

| Parameter | Nilai |
|---|---|
| Credential | `Telegram account` |
| Resource | `Message` |
| Operation | `Send Message` |
| Chat ID | `{{ $('Telegram Trigger').item.json.message.chat.id }}` |
| Text | `{{ $json.message }}` |

---

# 9️⃣ Contoh Hasil 💬

Pengguna mengirim:

```text
Saya mendapatkan pesan ini:

"Selamat! Anda mendapatkan hadiah.
Klik link berikut untuk mengambil hadiah:
https://example.com"
```

ScamCheck AI kemudian menganalisis:

```text
🛡️ ScamCheck AI

Risiko:
Perlu diperhatikan

Indikator:
• Menawarkan hadiah yang tidak terduga
• Meminta pengguna mengakses sebuah URL
• Menggunakan ajakan untuk segera melakukan tindakan

Hasil URL:
Hasil pemeriksaan VirusTotal tersedia.

Penjelasan:
Beberapa karakteristik pesan perlu diperiksa lebih lanjut
sebelum pengguna melakukan tindakan.

Rekomendasi:
Jangan memasukkan password, OTP, atau informasi pribadi
sebelum memastikan pengirim dan tujuan situs tersebut.
```

---

# 🔟 Final Workflow 🔄

Workflow lengkap:

```text
Telegram Trigger
       ↓
Edit Fields
       ↓
Code JavaScript
       ↓
IF
   ┌───┴────┐
   │        │
 TRUE     FALSE
   │        │
   ▼        ▼
VirusTotal AI Agent
   │        │
   ▼        │
 Wait       │
   │        │
   ▼        │
HTTP Request │
   │        │
   ▼        │
Code Scoring │
   │        │
   └────┬───┘
        ▼
    AI Agent
        ↓
Telegram Send Message
```

# 🔐 Security

Jangan masukkan credential asli ke GitHub:

```text
Telegram Bot Token
VirusTotal API Key
Gemini API Key
```

Gunakan **n8n Credentials** untuk menyimpan credential.

Jika workflow akan dipublikasikan, pastikan API key dan token sudah dihapus dari workflow yang diekspor.

# 📁 Repository Structure

```text
ScamCheck-AI/
│
├── README.md
│
└── workflow/
    └── scamcheck-ai.json
```

# 🎯 About

**ScamCheck AI** adalah workflow n8n yang menggabungkan Telegram, VirusTotal, dan AI Agent untuk membantu pengguna memahami indikator risiko dari pesan dan URL yang mencurigakan.
