# 🚀 Kaggle Free GPU LLM Runner (Ollama + Cloudflare Tunnel)

Jalankan model LLM raksasa (seperti **Qwen 2.5 27B GGUF**, Llama 3.3 70B Quant, dll.) secara **100% GRATIS** menggunakan GPU **Tesla T4 (Dual GPU / 30 GB VRAM)** dari Kaggle, dan akses publik dari mana saja menggunakan **Cloudflare Tunnel** (Domain Sendiri Permanen atau trycloudflare gratisan).

---

## ⚡ Mengapa Butuh Patch Ini? (Known Issues & Fixes)

Menjalankan Ollama di Kaggle secara standar sering gagal karena 4 masalah utama:

1. **Bug Manifest Tag HuggingFace:**
   Ollama mewajibkan penamaan tag quantisasi yang presisi (misal `:Q4_K_M` pakai garis bawah, bukan `:Q4KM`). Jika salah, Ollama mengembalikan `400: invalid quantization scheme`.
2. **Error HTTP 403 Forbidden (CORS & Host Header):**
   Secara default, Ollama menolak koneksi dari domain luar / reverse proxy.
   * **Fix:** Wajib menyetel environment `OLLAMA_ORIGINS="*"` dan `OLLAMA_HOST="0.0.0.0:11434"`, serta menambahkan flag `--http-host-header localhost` pada `cloudflared`.
3. **Container Kaggle Shutdown Otomatis (Error 1033 / 530):**
   Jika notebook selesai mengeksekusi cell terakhir, Kaggle langsung mematikan container VM.
   * **Fix:** Menambahkan `tunnel_proc.wait()` di background subprocess agar container tetap hidup selama sesi aktif.
4. **Context Window Exceeded pada Coding Agent / OpenCode (Error 400):**
   Default `num_ctx` Ollama untuk model besar adalah 32,768 (32k). Ketika coding agent (seperti OpenCode / Claude Code) mengirim system prompt + tool definitions (~36k tokens), Ollama akan crash `exceed_context_size_error`.
   * **Fix:** Auto-patch via REST API `POST /api/create` untuk menaikkan context window ke **65,536 (64k)** secara instan tanpa re-download.

---

## 📋 Fitur Utama
- **Gratis 30 Jam/Minggu:** Memanfaatkan kuota GPU gratis Kaggle (Tesla T4 x2).
- **Public URL Permanen:** Dukungan Cloudflare Named Tunnel (misal: `https://ollama.domainlo.com`).
- **Support OpenAI API Standard:** Endpoint kompatibel dengan OpenCode, 9Router, Open WebUI, SillyTavern, Chatbox, Cursor, dll.
- **Auto-Boost Context Window:** Siap dipakai untuk Agentic Tools & Long Context Coding (64k context).

---

## 🛠️ Step-by-Step Panduan Instalasi

### Opsi A: Melalui Kaggle Web UI (Paling Mudah)

1. Buka [Kaggle](https://www.kaggle.com/) → Login → Klik **Create** → **New Notebook**.
2. Di menu sebelah kanan (Settings Notebook):
   - **Accelerator:** Pilih **GPU T4 x2** (Dual GPU 30 GB VRAM).
   - **Internet:** Pastikan diaktifkan (**Internet On**).
3. Buat cell-cell berikut dan jalankan berurutan:

#### Cell 1: Install Zstandard & Ollama
```bash
!sudo apt-get update -y && sudo apt-get install -y zstd curl
!curl -fsSL https://ollama.com/install.sh | sh
```

#### Cell 2: Jalankan Ollama Server (Background + CORS Unlocked)
```python
import subprocess, time, os

env = os.environ.copy()
env["OLLAMA_ORIGINS"] = "*"
env["OLLAMA_HOST"] = "0.0.0.0:11434"

ollama_proc = subprocess.Popen(["ollama", "serve"], env=env)
time.sleep(5)
print("✅ Ollama daemon running!")
```

#### Cell 3: Pull Model & Patch Context Window (64k)
```python
import requests, subprocess

MODEL_TAG = "hf.co/JonathanColetti/Qwen3.8-27B-Uncensored-GGUF:Q4_K_M"

print(f"⬇️ Pulling {MODEL_TAG}...")
subprocess.run(["ollama", "pull", MODEL_TAG], check=True)

# Patch context window ke 65536 tokens
requests.post(
    "http://localhost:11434/api/create",
    json={
        "model": MODEL_TAG,
        "from": MODEL_TAG,
        "parameters": {"num_ctx": 65536}
    }
)
print("✅ Model ready with 64k context!")
```

#### Cell 4: Expose via Cloudflare Tunnel
Pilih salah satu metode:

**Metode 1: Menggunakan Domain Sendiri (Permanen)**
> Masukkan Token Tunnel dari Cloudflare Zero Trust:
```python
!wget -q -nc https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
!sudo dpkg -i cloudflared-linux-amd64.deb

TOKEN = "MASUKKAN_TUNNEL_TOKEN_CLOUDFLARE_LO_DI_SINI"
!cloudflared tunnel --no-autoupdate run --token {TOKEN}
```

**Metode 2: Menggunakan Quick Tunnel Gratisan (`trycloudflare.com`)**
```python
!wget -q -nc https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
!sudo dpkg -i cloudflared-linux-amd64.deb

import subprocess, re

tunnel_proc = subprocess.Popen(
    ["cloudflared", "tunnel", "--url", "http://localhost:11434", "--http-host-header", "localhost"],
    stderr=subprocess.PIPE,
    text=True
)

for line in iter(tunnel_proc.stderr.readline, ""):
    match = re.search(r"https://[a-zA-Z0-9-]+\.trycloudflare\.com", line)
    if match:
        print(f"\n🔥 PUBLIC URL OLLAMA:\n{match.group(0)}\n")
        break

tunnel_proc.wait() # Menjaga sesi Kaggle tetap hidup
```

---

### Opsi B: Menggunakan Kaggle CLI (Otomatis dari Terminal/VPS)

1. Pastikan Kaggle CLI terinstall:
   ```bash
   pip install kaggle
   mkdir -p ~/.kaggle && echo "YOUR_KAGGLE_API_TOKEN" > ~/.kaggle/access_token
   chmod 600 ~/.kaggle/access_token
   ```
2. Clone repo ini:
   ```bash
   git clone https://github.com/tonangarivin-ui/kaggle-gpu-ollama-tunnel.git
   cd kaggle-gpu-ollama-tunnel
   ```
3. Edit `kernel-metadata.json`, ganti `YOUR_KAGGLE_USERNAME` dengan username Kaggle lo.
4. Edit token di `kaggle_ollama_qwen.ipynb` jika ingin domain permanen.
5. Push langsung ke Kaggle:
   ```bash
   kaggle kernels push -p .
   ```
6. Monitor log & status:
   ```bash
   kaggle kernels status YOUR_KAGGLE_USERNAME/kaggle-ollama-gpu-tunnel
   ```

---

## 🌐 Setup Cloudflare Zero Trust (Untuk Domain Tetap)

1. Masuk ke [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → **Networks** → **Tunnels**.
2. Klik **Add a tunnel** → Pilih **Cloudflare (cloudflared)** → Beri nama tunnel.
3. Di halaman install command, copy string token yang ada di setelah `--token eyJh...`.
4. Di tab **Public Hostnames**, klik **Add a public hostname**:
   - **Subdomain:** misal `ollama`
   - **Domain:** pilih domain lo (misal `vunk.my.id`)
   - **Service Type:** `HTTP`
   - **URL:** `localhost:11434`
   - Buka **Additional application settings** → **HTTP Settings** → isi **HTTP Host Header**: `localhost`
   - Buka **TLS** → Aktifkan **No TLS Verify** (jika perlu).
5. Klik **Save hostname**. Endpoint lo akan menjadi: `https://ollama.domainlo.com`.

---

## 🔌 Integrasi Endpoint ke Aplikasi

Semua aplikasi pendukung OpenAI API bisa langsung terhubung ke endpoint lo:

| Konfigurasi | Nilai |
|---|---|
| **Base URL** | `https://ollama.domainlo.com/v1` |
| **API Key** | *Bebas / Kosongkan* |
| **Model Name** | `hf.co/JonathanColetti/Qwen3.8-27B-Uncensored-GGUF:Q4_K_M` |

### Contoh Penggunaan via cURL:
```bash
curl -X POST https://ollama.domainlo.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hf.co/JonathanColetti/Qwen3.8-27B-Uncensored-GGUF:Q4_K_M",
    "messages": [{"role": "user", "content": "Halo! Jawab dalam 1 kalimat."}]
  }'
```

---

## ⏸️ Menghemat Kuota Jam GPU Kaggle

Kaggle memberikan jatah ~30 jam GPU gratis per minggu.
* Jika sudah selesai menggunakan model, **segera klik "Cancel Run" / "Stop Session"** di web Kaggle, atau push kernel versi `enable_gpu: false` agar kuota jam GPU tidak terbuang sia-sia.

---

## 📜 Lisensi
MIT License. Bebas digunakan dan dimodifikasi untuk riset maupun kebutuhan personal.
