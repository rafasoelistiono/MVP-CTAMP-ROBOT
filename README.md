# Laporan MVP CTAMP Robot V1

CTAMP Robot V1 adalah MVP simulasi robot arm untuk studi **Continuous Task and
Motion Planning (CTAMP)**. Aplikasi ini menggabungkan task planning,
motion planning, inverse kinematics, simulasi fisika MuJoCo, dan validasi
collision untuk menjalankan task pick-and-place pada robot Franka Panda.

Fokus MVP ini bukan hanya membuat robot bergerak, tetapi membuktikan pipeline:

1. Membaca scene robot, object, obstacle, dan goal area.
2. Menentukan urutan task seperti pick, place, align, tidy up, atau separate.
3. Mengubah target task menjadi pose/joint target robot.
4. Mencari motion path dengan OMPL.
5. Mengeksekusi trajectory di MuJoCo dengan collision checking.
6. Merekam event log agar kegagalan bisa dianalisis dan diperbaiki.

## Apa Yang Bisa Dilakukan

Aplikasi ini dapat digunakan untuk:

- Menjalankan simulasi robot Franka Panda di MuJoCo.
- Melakukan pick-and-place object movable pada scene meja.
- Menjalankan task `tidy_up`, `align_cubes`, dan `separate_groups`.
- Menghindari obstacle fragile seperti vase, glass, ceramic, dan obstacle lain.
- Memakai OMPL untuk planning trajectory joint-space yang collision-aware.
- Memakai Pinocchio IK sebagai solver utama, dengan fallback legacy DLS bila
  dikonfigurasi.
- Menjalankan mode scripted OMPL-only tanpa LLM untuk eksperimen yang lebih
  deterministic.
- Menjalankan mode LLM planner melalui `src/main.py` untuk goal berbasis teks.
- Membuat CSV event log dan laporan analisis dari hasil run.
- Membandingkan performa sebelum dan sesudah improvement.

## Struktur Project

```text
MVP-CTAMP-ROBOT/
  assets/                         asset pendukung
  docs/                           laporan, audit, dan output analisis
  models/                         MuJoCo XML scene dan robot model
  scripts/                        task runner dan tool analisis
  src/                            runtime CTAMP utama
  requirements.txt                dependency Python
  .env.example                    template konfigurasi
```

File runtime yang dihasilkan lokal seperti `logs/`, `__pycache__/`,
`.pytest_cache/`, `.venv/`, `venv/`, dan `MUJOCO_LOG.TXT` tidak perlu disimpan
ke repository.

## Requirement

Disarankan memakai:

- Ubuntu 22.04/24.04, WSL2 Ubuntu, atau Linux native.
- Python 3.10 sampai 3.12.
- `python3-venv`, `pip`, dan build tools.
- MuJoCo Python package.
- OMPL Python binding.
- Pinocchio package `pin` dan `robot_descriptions`.
- OpenAI API key hanya jika menjalankan planner LLM di `src/main.py`.

## Instalasi Di WSL

Jalankan dari Ubuntu WSL, bukan dari PowerShell Python Windows.

```bash
cd /mnt/c/Adpro/MVP-CTAMP-ROBOT

sudo apt update
sudo apt install -y python3 python3-venv python3-pip build-essential \
  libgl1 libglfw3 libglew-dev patchelf

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

Jika OMPL dari `pip install ompl` tidak tersedia untuk versi Python yang
dipakai, install OMPL binding dari package manager atau build dari source,
lalu pastikan import ini berhasil:

```bash
python -c "from ompl import base, geometric; print('ompl ok')"
```

Verifikasi dependency utama:

```bash
python -c "import mujoco; print('mujoco ok')"
python -c "import pinocchio; print('pinocchio ok')"
python -c "from robot_descriptions.loaders.pinocchio import load_robot_description; print('robot descriptions ok')"
```

Untuk viewer GUI dari WSL, pastikan WSLg aktif. Jika ingin menjalankan headless,
gunakan `--no-viewer` pada script task atau set `ENABLE_VIEWER=false`.

## Instalasi Di Linux Native

```bash
git clone <repo-url>
cd MVP-CTAMP-ROBOT

sudo apt update
sudo apt install -y python3 python3-venv python3-pip build-essential \
  libgl1 libglfw3 libglew-dev patchelf

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

Buat file konfigurasi lokal:

```bash
cp .env.example .env
```

Isi `OPENAI_API_KEY` di `.env` hanya jika menjalankan mode LLM:

```text
OPENAI_API_KEY=your_openai_api_key_here
MODEL_FILE=models/panda.xml
ENABLE_VIEWER=true
OMPL_ENABLED=true
```

## Cara Menjalankan

Aktifkan virtualenv terlebih dahulu:

```bash
source .venv/bin/activate
```

### Jalankan Task OMPL-only

Mode ini paling cocok untuk eksperimen karena tidak memerlukan API key.

```bash
python scripts/tidy_up_ompl_only.py --object group no obs
python scripts/align_cubes_ompl_only.py --object ungroup obs
python scripts/separate_groups_ompl_only.py --object ungroup obs
python scripts/separate_groups_two_arm_ompl_only.py --object ungroup obs
```

Mode tanpa viewer:

```bash
python scripts/align_cubes_ompl_only.py --object ungroup obs --no-viewer
```

Pilihan scene untuk `--object`:

```text
group no obs
ungroup no obs
group obs
ungroup obs
```

### Jalankan Planner LLM

Pastikan `.env` sudah berisi `OPENAI_API_KEY`.

```bash
python src/main.py
python src/main.py "tidy up all cubes into safe aligned slots"
python src/main.py "move all cubes to the right side without touching obstacles"
```

### Analisis Log

Setiap run dapat menghasilkan CSV di `logs/`. Untuk membuat summary dan grafik:

```bash
python scripts/analyze_run_logs.py --log-dir logs --out-dir docs/log_analysis
```

Untuk visualisasi HTML dari satu event CSV:

```bash
python scripts/visualize_task_log.py \
  --events logs/<nama_file_events.csv> \
  --task align_cubes \
  --out docs/run_visualization.html
```

## Improvement Yang Sudah Dilakukan

Versi MVP ini sudah memuat beberapa improvement penting:

- OMPL planner diintegrasikan ke executor robot untuk trajectory joint-space.
- Collision policy MuJoCo dipakai saat planning dan live execution.
- Obstacle fragile diberi perlakuan khusus supaya tidak disentuh.
- Pinocchio Levenberg-Marquardt IK dipakai untuk akurasi inverse kinematics
  yang lebih baik dari legacy DLS.
- Precheck task menandai object yang unreachable, terlalu dekat obstacle, atau
  tidak aman untuk dipindahkan.
- Recovery pose setelah `drop()` mengembalikan arm ke safe hover agar retry
  tidak dimulai dari posisi jari dekat meja.
- Live trajectory checker dibuat lebih toleran terhadap transient start contact
  tertentu, sambil tetap memblok collision berbahaya.
- Event trace CSV dibuat lebih lengkap untuk `OMPL_PLAN`, `TRAJECTORY_EXEC`,
  `COLLISION_CHECK`, `PICK`, `PLACE`, dan `RECOVERY`.
- Script analisis log dan comparison report ditambahkan untuk membaca bottleneck
  secara kuantitatif.
- Script task dipisah menjadi runner deterministic: tidy up, align cubes,
  separate groups single-arm, dan separate groups two-arm.

## Ringkasan Analisis MVP

Analisis dari run sebelumnya menunjukkan:

- OMPL sering berhasil menemukan path, sehingga bottleneck utama bukan selalu
  "planner tidak menemukan solusi".
- Banyak kegagalan muncul saat trajectory sudah masuk executor dan divalidasi
  ulang oleh MuJoCo contact checker.
- Failure dominan sebelumnya adalah contact `table` dengan `left_finger` pada
  waypoint awal setelah drop atau retry.
- Kualitas IK dan kualitas grasp masih menjadi pembatas saat object jauh,
  object berbentuk circle/cylinder, atau scene padat obstacle.
- Improvement recovery pose, start-contact handling, dan precheck menurunkan
  failure yang sia-sia, tetapi task success masih perlu ditingkatkan melalui
  grasp sampler dan place strategy yang lebih stabil.

Dokumen analisis yang sudah ada:

```text
docs/log_analysis_comparison/comparison_summary.md
```

## Rencana Improvement Berikutnya

Prioritas teknis berikutnya:

1. Tambahkan grasp sampler nyata untuk cube dan circle/cylinder.
2. Validasi IK pose grasp sebelum OMPL dipanggil.
3. Gunakan beberapa IK seed dan beberapa candidate goal untuk setiap object.
4. Tambahkan goal region planning, bukan hanya satu joint goal.
5. Perkuat place strategy dengan release height, retreat path, dan settle check.
6. Tambahkan test kecil untuk parser scene, target allocation, dan log analysis.
7. Rapikan konfigurasi benchmark supaya hasil before/after mudah direproduksi.

## Troubleshooting

Jika viewer gagal muncul di WSL:

```bash
python scripts/align_cubes_ompl_only.py --object ungroup obs --no-viewer
```

Jika OMPL tidak ditemukan:

```bash
python -c "from ompl import base, geometric"
```

Jika command tersebut gagal, install ulang OMPL binding untuk Python yang sedang
aktif di virtualenv.

Jika Pinocchio gagal:

```bash
pip install pin robot_descriptions
python -c "import pinocchio; print('pinocchio ok')"
```

Jika package tidak terbaca, pastikan virtualenv aktif:

```bash
which python
python -m pip list
```

## Catatan Kebersihan Repository

File berikut dianggap artefak lokal dan tidak perlu commit:

```text
.venv/
venv/
__pycache__/
.pytest_cache/
logs/
MUJOCO_LOG.TXT
docs/log_analysis/
*.pyc
*.log
*.tmp
```

Gunakan `logs/` untuk hasil run lokal, lalu pindahkan hanya laporan atau grafik
yang benar-benar ingin disimpan ke `docs/`.
