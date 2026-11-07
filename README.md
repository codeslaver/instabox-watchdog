# instabox-watchdog

Menjaga microVM InstaBox **free tier** tetap hidup melampaui dua batas yang tidak bisa diubah lewat API.

## Masalah

Batasan resmi InstaBox free tier:

| Batas | Free | Bisa diubah? |
|---|---|---|
| Idle pause | 30 menit | ❌ `403 spec_not_allowed` |
| Max run | 120–180 menit | ❌ `403 permission_denied` |

Dua-duanya ditolak API. Tanpa pengawas, mesin berhenti dan bot/proses di dalamnya mati diam-diam.

## Celah yang dipakai

Diuji langsung terhadap API InstaBox:

```
SEBELUM suspend : runStartedAt 12:19:36   maxRunMinutes 120
SETELAH suspend : runStartedAt null
SETELAH resume  : runStartedAt 12:43:32   <- jam max-run RESET
                  processesPreserved: true <- memori & proses UTUH
durasi suspend->resume: 6 detik
```

Jadi `suspend` lalu `resume`, 10 menit sebelum `maxRunMinutes` habis, me-reset jam tanpa cold boot dan tanpa kehilangan proses.

## Yang dilakukan workflow ini

Tiap 5 menit, untuk setiap mesin di `MACHINE_IDS`:

- `suspended` → `POST /resume`
- `stopped` → `POST /start` (cold boot — proses hilang, jadi service di dalam VM harus `systemd enable`)
- `running` dan umur ≥ `maxRunMinutes - 10` → `POST /suspend`, tunggu 8 detik, `POST /resume`

## Setup

1. Bikin API key di dashboard InstaBox (`ibk_...`).
2. Set dua secret repo:

```bash
gh secret set INSTABOX_API_KEY --body "ibk_..."
gh secret set MACHINE_IDS --body "m_xxxxxxxx,m_yyyyyyyy"
```

3. Aktifkan Actions. Workflow jalan sendiri tiap 5 menit. Untuk uji manual:

```bash
gh workflow run instabox-watchdog
```

## Kenapa repo ini publik

GitHub Actions pada repo **privat** dibatasi 2.000 menit/bulan. Cron tiap menit menghabiskan ±43.000 menit/bulan, tiap 5 menit ±8.640 — jauh melewati kuota. Pada repo **publik** Actions gratis tanpa batas.

Tidak ada kredensial di repo ini. API key dan machine ID disimpan di **GitHub Secrets**, hanya dibaca saat runtime.

## Batas jujur

- Paling cepat 5 menit sekali, dan penjadwal GitHub sering telat 1–10 menit saat sibuk. Jadi saat mesin pause, downtime realistis **5–15 menit**.
- Saat `stopped` (cold boot), proses hilang. Service di dalam VM wajib `systemd enable` supaya hidup lagi otomatis.
- Ini tambal sulam, bukan solusi. Untuk beban 24/7 yang serius, pakai host yang memang dirancang menyala terus.
