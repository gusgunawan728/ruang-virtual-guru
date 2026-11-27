# RANCANGAN SISTEM — RUANG VIRTUAL GURU

**Platform administrasi & perangkat pembelajaran guru untuk semua jenjang dan mata pelajaran**

| Item | Keterangan |
|---|---|
| Frontend | React 18 + Vite (SPA) + React Router + Tailwind CSS |
| Backend / Database | Supabase (PostgreSQL, Auth, Storage, Row Level Security, Edge Functions) |
| Hosting frontend | cPanel (hosting statis, tanpa Node.js di server) |
| Login | Username + password |
| Status dokumen | Rancangan (proposal) — belum diuji dalam implementasi |

---

## 1. TUJUAN DAN RUANG LINGKUP

### 1.1 Tujuan
Menyediakan satu ruang kerja digital bagi guru/dosen untuk mengelola seluruh siklus pembelajaran: merencanakan (modul ajar), melaksanakan (materi, jurnal, kehadiran), menilai (ulangan, analisis soal), menindaklanjuti (remedial/pengayaan), dan membina siswa (BK), dengan data yang tersimpan rapi, aman, dan dapat diekspor.

### 1.2 Cakupan jenjang
TK/RA, SD/MI, SMP/MTs, SMA/SMK/MA, dan Perguruan Tinggi. Perbedaan antarjenjang **tidak di-hardcode**, melainkan diatur lewat tabel konfigurasi (lihat bagian 6).

### 1.3 Cakupan fitur (modul)
1. Autentikasi & manajemen pengguna
2. Master data (sekolah, tahun ajaran, kelas, mapel, siswa)
3. Modul ajar / RPP / RPS
4. Materi ajar (bahan ajar, file, tautan)
5. Administrasi kehadiran
6. Administrasi penilaian
7. Jurnal pembelajaran harian
8. Bank soal, kisi-kisi, dan penyusunan asesmen (harian, formatif, sumatif)
9. Analisis soal (butir soal dan hasil ujian)
10. Remedial dan pengayaan
11. Bimbingan dan Konseling (BK)
12. Laporan dan ekspor (PDF, DOCX, XLSX)
13. Dashboard dan notifikasi
14. Audit log dan pengaturan

### 1.4 Di luar cakupan tahap awal
Video conference, LMS penuh bagi siswa, pembayaran, dan aplikasi mobile native. Dapat ditambahkan pada tahap lanjut.

---

## 2. PERAN PENGGUNA (ROLE)

| Role | Hak akses utama |
|---|---|
| `super_admin` | Kelola seluruh sistem, semua lembaga, konfigurasi global |
| `admin_lembaga` | Kelola satu lembaga: pengguna, tahun ajaran, kelas, mapel, template |
| `kepala` (kepala sekolah/madrasah/pimpinan) | Melihat dan memverifikasi laporan, modul, dan rekap guru di lembaganya |
| `guru` (dosen untuk PT) | Mengelola perangkat ajar, kehadiran, nilai, jurnal, soal, remedial di kelas yang diampu |
| `guru_bk` | Mengelola data BK; akses data BK hanya miliknya |
| `wali_kelas` (peran tambahan) | Melihat rekap kehadiran dan nilai seluruh siswa di kelas perwaliannya |
| `siswa` (opsional, tahap 2) | Melihat nilai, kehadiran, dan tugas milik sendiri |

> **Catatan:** satu pengguna dapat memiliki lebih dari satu peran (mis. guru + wali kelas). Peran disimpan di tabel terpisah, bukan satu kolom tunggal.

> **Model akses:** Disarankan *multi-tenant* — satu instalasi melayani banyak lembaga, setiap baris data membawa `institution_id`. Jika hanya untuk satu madrasah, cukup satu lembaga.

---

## 3. ARSITEKTUR SISTEM

```
┌──────────────────────────┐        HTTPS         ┌────────────────────────────┐
│  Browser (Guru/Admin)    │ ───────────────────► │  cPanel: public_html       │
│  React SPA (hasil build) │ ◄─────────────────── │  index.html + assets/*     │
└────────────┬─────────────┘   file statis        └────────────────────────────┘
             │
             │ supabase-js (HTTPS, anon key + JWT pengguna)
             ▼
┌─────────────────────────────────────────────────────────────┐
│                         SUPABASE                            │
│  Auth  │  PostgreSQL + RLS  │  Storage  │  Edge Functions   │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 Prinsip utama
- **Frontend statis.** Vite menghasilkan folder `dist/` berisi HTML/JS/CSS biasa sehingga cukup di-upload ke cPanel; server tidak perlu menjalankan Node.js.
- **Keamanan di database, bukan di frontend.** Semua pembatasan akses diterapkan lewat **Row Level Security (RLS)** PostgreSQL. Menyembunyikan menu di React hanyalah kenyamanan tampilan, bukan keamanan.
- **Hanya `anon key` di frontend.** `service_role key` **tidak boleh** ada di kode React, `.env` yang ikut di-build, maupun repositori publik.
- **Operasi sensitif lewat Edge Function.** Contoh: admin membuat akun guru dengan username/password, reset password, impor siswa massal.

### 3.2 Komponen Supabase yang dipakai
| Komponen | Fungsi |
|---|---|
| Auth | Login, sesi, JWT |
| Database (Postgres) | Seluruh data terstruktur |
| RLS + Policy | Isolasi data per lembaga, per guru, per peran |
| Storage | File modul, materi, lampiran, foto, hasil ekspor |
| Edge Functions | Pembuatan akun, reset password, analisis berat (opsional), ekspor |
| Realtime (opsional) | Notifikasi langsung |

---

## 4. AUTENTIKASI DENGAN USERNAME + PASSWORD

Supabase Auth secara bawaan memakai **email + password**. Untuk memenuhi kebutuhan **username + password**, gunakan pola *pseudo-email*:

### 4.1 Pola yang direkomendasikan
1. Admin membuat akun dengan `username` (mis. `agus.gunawan`).
2. Sistem membentuk email internal: `agus.gunawan@ruangguru.local` (domain fiktif/milik sendiri; tidak harus dapat menerima email).
3. Di halaman login, pengguna hanya mengisi **username** dan **password**. Frontend menambahkan sufiks domain sebelum memanggil `supabase.auth.signInWithPassword`.
4. Username disimpan juga di tabel `profiles.username` (unik per sistem) untuk tampilan dan pencarian.

```js
// src/lib/auth.js
import { supabase } from './supabase';

const DOMAIN = import.meta.env.VITE_LOGIN_DOMAIN; // contoh: ruangguru.local

export async function loginWithUsername(username, password) {
  const email = `${username.trim().toLowerCase()}@${DOMAIN}`;
  const { data, error } = await supabase.auth.signInWithPassword({ email, password });
  if (error) throw new Error('Username atau password salah');
  return data;
}
```

### 4.2 Pembuatan akun (aman)
Pembuatan pengguna memerlukan `service_role key`, sehingga **harus** lewat Edge Function:

```
Admin (UI) ──► Edge Function "create-user" ──► supabase.auth.admin.createUser()
                  │  (memeriksa: pemanggil = admin_lembaga/super_admin)
                  └─► insert profiles + user_roles
```

### 4.3 Kebijakan keamanan akun
- Password minimal 8 karakter; wajib **ganti password pada login pertama** (`must_change_password = true`).
- Reset password oleh admin lewat Edge Function (karena tidak ada email pemulihan).
- Batasi percobaan login (rate limit bawaan Supabase) dan catat kegagalan ke audit log.
- Sesi otomatis diperbarui; fitur "keluar dari semua perangkat" disediakan.
- Opsional: tambahkan email asli di profil untuk pemulihan.

### 4.4 Alur panel masuk
```
Buka situs ─► Halaman Login ─► (valid) ─► cek must_change_password
                                            ├─ ya  ─► Halaman Ganti Password
                                            └─ tidak ─► Dashboard sesuai role
```

---

## 5. STRUKTUR MENU DAN HALAMAN

```
Dashboard
├── Beranda (jadwal hari ini, tugas belum selesai, notifikasi)
├── Perencanaan
│   ├── Modul Ajar / RPP / RPS
│   ├── Capaian & Tujuan Pembelajaran (CP/TP/ATP atau CPMK/Sub-CPMK)
│   └── Materi Ajar
├── Pelaksanaan
│   ├── Jurnal Pembelajaran Harian
│   ├── Kehadiran Siswa
│   └── Jadwal Mengajar
├── Penilaian
│   ├── Bank Soal
│   ├── Kisi-kisi
│   ├── Asesmen (Harian / Formatif / Sumatif)
│   ├── Input Nilai
│   ├── Analisis Soal & Hasil
│   └── Rekap & Leger Nilai
├── Tindak Lanjut
│   ├── Remedial
│   └── Pengayaan
├── Bimbingan & Konseling
│   ├── Data Kasus / Layanan
│   ├── Catatan Konseling (terbatas)
│   └── Jadwal & Tindak Lanjut
├── Laporan
│   ├── Rekap Kehadiran
│   ├── Rekap Penilaian
│   ├── Rekap Jurnal
│   └── Ekspor (PDF/DOCX/XLSX)
└── Pengaturan
    ├── Profil & Password
    ├── Master Data (admin)
    ├── Pengguna & Peran (admin)
    └── Audit Log (admin)
```

---

## 6. FLEKSIBILITAS LINTAS JENJANG

Agar satu sistem dapat dipakai TK sampai perguruan tinggi, perbedaan diatur lewat **profil jenjang** (`education_levels`) dan **template dinamis**, bukan kode terpisah.

| Aspek | TK/RA | SD/MI | SMP/MTs | SMA/SMK/MA | Perguruan Tinggi |
|---|---|---|---|---|---|
| Unit kelompok | Kelompok (A/B) | Kelas 1–6 | Kelas 7–9 | Kelas 10–12 (+ jurusan) | Program studi + semester |
| Perencanaan | Modul ajar/RPPH | Modul ajar | Modul ajar | Modul ajar | RPS |
| Capaian | Capaian perkembangan | CP/TP | CP/TP | CP/TP | CPL/CPMK |
| Penilaian | Deskriptif/ceklis perkembangan | Angka + deskripsi | Angka + deskripsi | Angka + deskripsi | Angka/huruf (A–E), bobot |
| Konsep waktu | Tahun ajaran, semester | Tahun ajaran, semester | Tahun ajaran, semester | Tahun ajaran, semester | Tahun akademik, semester, SKS |

### 6.1 Mekanisme konfigurasi
- `education_levels`: kode jenjang, nama, skema nilai (`numeric`, `letter`, `descriptive`), nama unit ("Kelas"/"Kelompok"/"Semester").
- `grading_schemes`: bobot komponen (harian, tugas, PTS, PAS, praktik) dan konversi nilai ke predikat — dapat diubah per lembaga.
- `document_templates`: template modul ajar/RPS dalam format **JSON schema** (daftar bagian, jenis input), sehingga setiap jenjang punya struktur isian sendiri tanpa mengubah kode.
- Label antarmuka (mis. "Siswa" vs "Mahasiswa", "Guru" vs "Dosen") diambil dari konfigurasi jenjang.

---

## 7. RANCANGAN DATABASE (POSTGRESQL / SUPABASE)

### 7.1 Diagram relasi ringkas

```
institutions ─┬─< profiles >──< user_roles
              ├─< academic_years ─< semesters
              ├─< classes ─< class_students >─ students
              ├─< subjects
              └─< teaching_assignments (guru × mapel × kelas × semester)
                        ├─< lesson_plans
                        ├─< materials
                        ├─< attendance_sessions ─< attendance_records
                        ├─< journals
                        ├─< assessments ─< assessment_items >─ questions
                        │         ├─< scores
                        │         └─< blueprints (kisi-kisi)
                        ├─< remedial_plans / enrichment_plans
                        └─ (hasil) item_analysis
students ─< counseling_cases ─< counseling_sessions   (khusus BK)
```

### 7.2 Skema inti (SQL)

> Jalankan di **Supabase → SQL Editor**. Urutan: ekstensi → tabel → fungsi bantu → RLS.

```sql
-- ========== EKSTENSI ==========
create extension if not exists "pgcrypto";

-- ========== ENUM ==========
create type app_role as enum
  ('super_admin','admin_lembaga','kepala','guru','guru_bk','wali_kelas','siswa');
create type assessment_type as enum ('harian','formatif','sumatif','pts','pas','praktik','tugas');
create type attendance_status as enum ('hadir','izin','sakit','alpa','terlambat');
create type question_type as enum ('pilihan_ganda','isian','uraian','benar_salah','menjodohkan');

-- ========== LEMBAGA & PENGGUNA ==========
create table institutions (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  npsn_nsm text,
  level_code text not null,            -- TK, SD, SMP, SMA, SMK, PT, dst
  address text,
  logo_path text,
  created_at timestamptz default now()
);

create table education_levels (
  code text primary key,               -- 'TK','SD','SMP','SMA','SMK','PT'
  name text not null,
  grade_label text default 'Kelas',
  learner_label text default 'Siswa',
  teacher_label text default 'Guru',
  grading_mode text default 'numeric'  -- numeric | letter | descriptive
);

create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  institution_id uuid references institutions(id),
  username text unique not null,
  full_name text not null,
  nip_nik text,
  phone text,
  photo_path text,
  must_change_password boolean default true,
  is_active boolean default true,
  created_at timestamptz default now()
);

create table user_roles (
  user_id uuid references profiles(id) on delete cascade,
  role app_role not null,
  primary key (user_id, role)
);

-- ========== MASTER AKADEMIK ==========
create table academic_years (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  name text not null,                  -- 2026/2027
  is_active boolean default false
);

create table semesters (
  id uuid primary key default gen_random_uuid(),
  academic_year_id uuid not null references academic_years(id),
  name text not null,                  -- Ganjil / Genap
  start_date date, end_date date,
  is_active boolean default false
);

create table classes (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  academic_year_id uuid not null references academic_years(id),
  name text not null,                  -- 7A, X IPA 1, TI-3A
  grade text,                          -- 7, X, Semester 3
  program text,                        -- jurusan/prodi (opsional)
  homeroom_teacher_id uuid references profiles(id)
);

create table subjects (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  code text, name text not null,
  credits numeric                       -- SKS (PT) / JP
);

create table students (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  nis_nim text, nisn text,
  full_name text not null,
  gender char(1),
  birth_date date,
  guardian_name text, guardian_phone text,
  is_active boolean default true,
  user_id uuid references profiles(id)  -- bila siswa punya akun (tahap 2)
);

create table class_students (
  class_id uuid references classes(id) on delete cascade,
  student_id uuid references students(id) on delete cascade,
  primary key (class_id, student_id)
);

-- Penugasan mengajar: pusat relasi hampir semua data guru
create table teaching_assignments (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  teacher_id uuid not null references profiles(id),
  subject_id uuid not null references subjects(id),
  class_id uuid not null references classes(id),
  semester_id uuid not null references semesters(id),
  weekly_hours int,
  unique (teacher_id, subject_id, class_id, semester_id)
);

create table schedules (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references teaching_assignments(id) on delete cascade,
  day_of_week int check (day_of_week between 1 and 7),
  start_time time, end_time time, room text
);

-- ========== PERENCANAAN ==========
create table learning_outcomes (       -- CP/TP/ATP atau CPL/CPMK
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  subject_id uuid references subjects(id),
  level_code text, grade text,
  kind text,                           -- CP | TP | ATP | CPL | CPMK
  code text, description text not null,
  parent_id uuid references learning_outcomes(id)
);

create table document_templates (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid references institutions(id),  -- null = template global
  kind text not null,                  -- modul_ajar | rps | jurnal | kisi_kisi
  level_code text,
  name text not null,
  schema jsonb not null                -- struktur bagian & jenis isian
);

create table lesson_plans (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references teaching_assignments(id) on delete cascade,
  template_id uuid references document_templates(id),
  title text not null,
  content jsonb not null default '{}',  -- isi sesuai schema template
  status text default 'draft',          -- draft | diajukan | disetujui | revisi
  reviewer_id uuid references profiles(id),
  reviewer_note text,
  file_path text,                       -- hasil ekspor / unggahan
  created_at timestamptz default now(), updated_at timestamptz default now()
);

create table materials (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references teaching_assignments(id) on delete cascade,
  title text not null,
  description text,
  kind text,                            -- file | tautan | teks | video
  file_path text, url text, body text,
  outcome_id uuid references learning_outcomes(id),
  is_shared boolean default false,
  created_at timestamptz default now()
);

-- ========== KEHADIRAN ==========
create table attendance_sessions (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references teaching_assignments(id) on delete cascade,
  session_date date not null,
  meeting_no int,
  start_time time, end_time time,
  unique (assignment_id, session_date, meeting_no)
);

create table attendance_records (
  session_id uuid references attendance_sessions(id) on delete cascade,
  student_id uuid references students(id) on delete cascade,
  status attendance_status not null default 'hadir',
  note text,
  primary key (session_id, student_id)
);

-- ========== JURNAL HARIAN ==========
create table journals (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references teaching_assignments(id) on delete cascade,
  session_id uuid references attendance_sessions(id),
  journal_date date not null,
  topic text not null,
  outcome_id uuid references learning_outcomes(id),
  activities text,
  method text,
  obstacles text,
  reflection text,
  follow_up text,
  attachment_path text,
  created_at timestamptz default now()
);

-- ========== BANK SOAL, KISI-KISI, ASESMEN ==========
create table questions (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  owner_id uuid not null references profiles(id),
  subject_id uuid references subjects(id),
  level_code text, grade text,
  outcome_id uuid references learning_outcomes(id),
  indicator text,                       -- indikator soal
  cognitive_level text,                 -- C1–C6 / LOTS-MOTS-HOTS
  difficulty_est text,                  -- mudah | sedang | sukar (perkiraan guru)
  q_type question_type not null,
  stem text not null,                   -- pertanyaan
  options jsonb,                        -- opsi (PG): [{key:'A',text:'..'},..]
  answer_key jsonb,                     -- kunci / rubrik
  score_max numeric default 1,
  is_shared boolean default false,
  created_at timestamptz default now()
);

create table assessments (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references teaching_assignments(id) on delete cascade,
  title text not null,
  a_type assessment_type not null,
  held_on date,
  weight numeric default 1,
  kkm numeric,                          -- KKM/KKTP / batas lulus
  duration_min int,
  status text default 'draft'
);

create table blueprints (               -- kisi-kisi
  id uuid primary key default gen_random_uuid(),
  assessment_id uuid not null references assessments(id) on delete cascade,
  rows jsonb not null                   -- [{no, cp_tp, materi, indikator, level_kognitif, bentuk, no_soal, skor}]
);

create table assessment_items (
  assessment_id uuid references assessments(id) on delete cascade,
  question_id uuid references questions(id),
  item_no int not null,
  score_max numeric not null,
  primary key (assessment_id, item_no)
);

create table scores (                   -- nilai total per siswa per asesmen
  assessment_id uuid references assessments(id) on delete cascade,
  student_id uuid references students(id) on delete cascade,
  total numeric,
  grade_letter text,
  descriptive_note text,
  is_remedial boolean default false,
  primary key (assessment_id, student_id)
);

create table item_responses (            -- skor per butir (dasar analisis soal)
  assessment_id uuid,
  student_id uuid,
  item_no int,
  answer text,
  score numeric,
  primary key (assessment_id, student_id, item_no),
  foreign key (assessment_id, student_id) references scores(assessment_id, student_id) on delete cascade
);

create table item_analysis (             -- hasil analisis disimpan (cache)
  assessment_id uuid references assessments(id) on delete cascade,
  item_no int,
  difficulty_index numeric,              -- P
  discrimination_index numeric,          -- D
  distractor_stats jsonb,                -- sebaran jawaban per opsi
  validity_r numeric,                    -- korelasi butir-total (opsional)
  verdict text,                          -- diterima | revisi | dibuang
  computed_at timestamptz default now(),
  primary key (assessment_id, item_no)
);

-- ========== REMEDIAL & PENGAYAAN ==========
create table followups (
  id uuid primary key default gen_random_uuid(),
  assessment_id uuid not null references assessments(id) on delete cascade,
  student_id uuid not null references students(id) on delete cascade,
  kind text not null check (kind in ('remedial','pengayaan')),
  outcome_id uuid references learning_outcomes(id),
  plan text,                             -- bentuk kegiatan
  scheduled_on date,
  result_score numeric,
  status text default 'direncanakan',    -- direncanakan | berjalan | selesai
  note text
);

-- ========== BIMBINGAN & KONSELING ==========
create table counseling_cases (
  id uuid primary key default gen_random_uuid(),
  institution_id uuid not null references institutions(id),
  student_id uuid not null references students(id),
  counselor_id uuid not null references profiles(id),
  category text,                         -- akademik | pribadi | sosial | karier
  referred_by uuid references profiles(id),
  summary_public text,                   -- ringkasan yang boleh dilihat wali kelas
  status text default 'dibuka',
  opened_on date default current_date,
  closed_on date
);

create table counseling_sessions (
  id uuid primary key default gen_random_uuid(),
  case_id uuid not null references counseling_cases(id) on delete cascade,
  session_date date not null,
  method text,                           -- individual | kelompok | klasikal
  notes_private text,                    -- RAHASIA: hanya guru BK pemilik kasus
  follow_up text,
  next_session date
);

-- ========== NOTIFIKASI & AUDIT ==========
create table notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id) on delete cascade,
  title text, body text, link text,
  is_read boolean default false,
  created_at timestamptz default now()
);

create table audit_logs (
  id bigserial primary key,
  institution_id uuid,
  actor_id uuid,
  action text,                           -- insert | update | delete | login | export
  table_name text, record_id text,
  detail jsonb,
  created_at timestamptz default now()
);

-- ========== INDEKS PENTING ==========
create index on teaching_assignments (teacher_id, semester_id);
create index on attendance_sessions (assignment_id, session_date);
create index on journals (assignment_id, journal_date);
create index on scores (student_id);
create index on questions (institution_id, subject_id);
create index on counseling_cases (counselor_id);
```

---

## 8. KEAMANAN: ROW LEVEL SECURITY (RLS)

RLS wajib diaktifkan pada **semua** tabel. Prinsip: *tolak semua secara bawaan, lalu izinkan seperlunya*.

### 8.1 Fungsi bantu

```sql
create or replace function public.current_institution()
returns uuid language sql stable security definer set search_path = public as $$
  select institution_id from profiles where id = auth.uid()
$$;

create or replace function public.has_role(r app_role)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from user_roles where user_id = auth.uid() and role = r)
$$;

create or replace function public.owns_assignment(a uuid)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from teaching_assignments
                 where id = a and teacher_id = auth.uid())
$$;
```

### 8.2 Contoh kebijakan

```sql
alter table teaching_assignments enable row level security;
alter table journals            enable row level security;
alter table counseling_cases    enable row level security;
alter table counseling_sessions enable row level security;
-- (aktifkan untuk SEMUA tabel lain)

-- Guru: hanya penugasan miliknya; admin/kepala: seluruh lembaga
create policy "assign_select" on teaching_assignments for select using (
  institution_id = current_institution()
  and (teacher_id = auth.uid()
       or has_role('admin_lembaga') or has_role('kepala') or has_role('super_admin'))
);

-- Jurnal: guru pemilik penugasan boleh CRUD; kepala hanya baca
create policy "journal_owner_all" on journals for all
  using (owns_assignment(assignment_id))
  with check (owns_assignment(assignment_id));

create policy "journal_kepala_read" on journals for select using (
  has_role('kepala') and exists (
    select 1 from teaching_assignments ta
    where ta.id = journals.assignment_id
      and ta.institution_id = current_institution())
);

-- BK: hanya konselor pemilik kasus
create policy "bk_case_owner" on counseling_cases for all
  using (counselor_id = auth.uid())
  with check (counselor_id = auth.uid());

create policy "bk_session_owner" on counseling_sessions for all
  using (exists (select 1 from counseling_cases c
                 where c.id = case_id and c.counselor_id = auth.uid()))
  with check (exists (select 1 from counseling_cases c
                 where c.id = case_id and c.counselor_id = auth.uid()));
```

### 8.3 Aturan khusus data BK
- Kolom `notes_private` **hanya** terbaca oleh konselor pemilik kasus; kepala sekolah, wali kelas, dan admin **tidak** otomatis berhak.
- Wali kelas hanya melihat `summary_public`, melalui *view* terpisah.
- Setiap akses data BK dicatat di `audit_logs`.
- Pertimbangkan enkripsi kolom sensitif (pgcrypto/Supabase Vault) bila kebijakan lembaga mengharuskan.

### 8.4 Kebijakan Storage
- Bucket `materials`, `lesson-plans`, `exports`, `attachments` berstatus **private**.
- Penamaan path: `{institution_id}/{teacher_id}/{kategori}/{nama_file}`; policy Storage memeriksa segmen pertama path sama dengan `current_institution()` dan segmen kedua sama dengan `auth.uid()` untuk operasi tulis.
- Batasi ukuran dan tipe file (mis. PDF, DOCX, PPTX, XLSX, gambar; maks. 20 MB).
- Akses unduh melalui *signed URL* berumur pendek.

### 8.5 Checklist keamanan
- [ ] RLS aktif di semua tabel dan diuji dengan tiap role
- [ ] `service_role key` hanya di Edge Function / secret Supabase
- [ ] Validasi input di frontend **dan** constraint di database
- [ ] Audit log untuk login, ekspor, hapus, dan akses BK
- [ ] HTTPS wajib (aktifkan SSL/AutoSSL di cPanel)
- [ ] Backup berkala (jadwal ekspor database)
- [ ] Mematuhi kebijakan perlindungan data pribadi siswa (UU PDP) dan persetujuan lembaga

---

## 9. RANCANGAN FITUR PER MODUL

### 9.1 Modul Ajar / RPP / RPS
- Pilih penugasan (mapel–kelas–semester) lalu pilih **template jenjang**.
- Editor berbentuk formulir dinamis dari `document_templates.schema`: identitas, CP/TP, profil pelajaran/pelajar Pancasila, kegiatan inti, asesmen, lampiran.
- Status alur: `draft → diajukan → disetujui/revisi` (kepala dapat memberi catatan).
- Fitur: duplikasi dari semester lalu, pustaka CP/TP, unggah file pendukung, ekspor **DOCX/PDF**.
- Versi riwayat (opsional): simpan snapshot setiap kali status berubah.

### 9.2 Materi Ajar
- Tipe: file, tautan, video, teks (editor rich text).
- Dikaitkan ke CP/TP dan penugasan; bisa ditandai *dibagikan* agar guru lain di lembaga yang sama dapat menggunakan.
- Pencarian dan filter berdasarkan mapel, jenjang, dan tag.

### 9.3 Administrasi Kehadiran
- Buat sesi pertemuan (tanggal, pertemuan ke-) dari jadwal.
- Daftar siswa otomatis dari `class_students`; tombol cepat "Semua hadir" lalu ubah yang tidak hadir.
- Rekap per siswa/kelas/bulan/semester, persentase kehadiran, peringatan siswa berisiko (alpa melebihi ambang).
- Ekspor rekap ke XLSX/PDF; antarmuka ramah ponsel.
- Satu kali input dapat otomatis mengisi jurnal (tautan `session_id`).

### 9.3.1 Administrasi Penilaian (rekap)
- Komponen dan bobot sesuai `grading_schemes`.
- Input nilai per asesmen (tabel seperti spreadsheet, dukung tempel dari Excel).
- Perhitungan nilai akhir, predikat, deskripsi capaian otomatis, serta leger/rapor-sementara.
- Penanda siswa di bawah KKM/KKTP → otomatis masuk daftar remedial.

### 9.4 Jurnal Pembelajaran Harian
- Formulir singkat: tanggal, topik, CP/TP, kegiatan, kendala, refleksi, tindak lanjut.
- Pra-isi dari jadwal dan kehadiran hari itu.
- Rekap bulanan per guru; kepala dapat memantau kelengkapan jurnal.

### 9.5 Ulangan Harian, Formatif, Sumatif
1. Guru membuat **asesmen** (jenis, tanggal, bobot, KKM).
2. Menyusun **kisi-kisi** (otomatis dari tabel `blueprints` atau dihasilkan dari soal terpilih).
3. Memilih soal dari **bank soal** atau membuat soal baru.
4. Mencetak naskah soal + kunci jawaban + pedoman penskoran (PDF/DOCX).
5. Input nilai manual atau unggah lembar jawaban (impor XLSX) → data tersimpan di `scores` dan `item_responses`.

### 9.6 Kisi-kisi Soal
Kolom standar: No, CP/TP/CPMK, materi, indikator soal, level kognitif, bentuk soal, nomor soal, skor. Format bisa diubah lewat template per jenjang.

### 9.7 Analisis Soal
Dihitung dari `item_responses`, hasil disimpan di `item_analysis`.

| Indikator | Rumus ringkas | Interpretasi umum* |
|---|---|---|
| Tingkat kesukaran (P) | P = jumlah skor siswa pada butir ÷ (skor maksimum × jumlah siswa) | < 0,30 sukar; 0,30–0,70 sedang; > 0,70 mudah |
| Daya pembeda (D) | D = P(kelompok atas 27%) − P(kelompok bawah 27%) | ≥ 0,40 baik sekali; 0,30–0,39 baik; 0,20–0,29 cukup; < 0,20 perlu revisi |
| Efektivitas pengecoh | Jumlah pemilih tiap opsi | Pengecoh dianggap berfungsi bila dipilih ≥ ±5% peserta |
| Validitas butir | Korelasi butir–skor total (point-biserial) | Dibandingkan nilai r-tabel |
| Reliabilitas tes | KR-20 (dikotomi) / Cronbach's α (uraian) | ≥ 0,70 umumnya memadai |

\*Ambang batas bervariasi menurut rujukan; simpan sebagai pengaturan yang dapat diubah, dan cantumkan rujukan yang dipakai lembaga.

Keluaran: tabel per butir, grafik sebaran nilai, rekomendasi **diterima / direvisi / dibuang**, serta tombol "simpan ke bank soal dengan status terkalibrasi". Analisis capaian per CP/TP untuk menentukan materi yang perlu diajar ulang.

### 9.8 Remedial dan Pengayaan
- Sistem otomatis menyaring siswa < KKM → daftar **remedial**; siswa di atas ambang tinggi (mis. ≥ 90, dapat diatur) → daftar **pengayaan**.
- Guru memilih bentuk kegiatan (tugas, tes ulang, bimbingan), jadwal, dan mencatat hasil.
- Aturan nilai akhir remedial (mis. maksimal KKM atau nilai tertinggi) dibuat sebagai konfigurasi lembaga.
- Rekap efektivitas: sebelum vs sesudah remedial.

### 9.9 Bimbingan dan Konseling
- Layanan: individual, kelompok, klasikal, bimbingan karier.
- Data kasus, sesi konseling, rujukan dari guru/wali kelas, dan tindak lanjut.
- Pemisahan tegas antara **catatan rahasia** dan **ringkasan yang boleh dibagikan**.
- Laporan agregat (tanpa identitas) untuk kepala: jumlah layanan per kategori dan per bulan.
- Pengingat jadwal konseling lanjutan.

### 9.10 Dashboard & Notifikasi
- Guru: jadwal hari ini, kehadiran belum diisi, jurnal belum diisi, modul berstatus revisi, remedial terjadwal.
- Kepala/admin: kelengkapan administrasi per guru, grafik kehadiran, progres modul.
- Notifikasi dalam aplikasi (tabel `notifications`, opsional Realtime).

### 9.11 Laporan dan Ekspor
- Ekspor ke **XLSX** (SheetJS) dan **PDF** (react-pdf / jsPDF) dari browser; **DOCX** (docx.js) untuk modul/kisi-kisi.
- Kop surat lembaga dari `institutions` (logo, nama, alamat).
- Dukung huruf Arab/RTL dengan font yang disematkan (mis. Amiri/Noto Naskh Arabic) bila mapel menggunakan aksara tersebut.

---

## 10. STRUKTUR PROYEK REACT-VITE

```
ruang-guru/
├── public/
│   ├── .htaccess                 # aturan SPA untuk cPanel (disalin ke dist saat build)
│   └── favicon.ico
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── lib/
│   │   ├── supabase.js           # createClient
│   │   ├── auth.js               # loginWithUsername, logout
│   │   └── export/               # xlsx.js, pdf.js, docx.js
│   ├── context/
│   │   └── AuthContext.jsx       # sesi, profil, role
│   ├── routes/
│   │   ├── ProtectedRoute.jsx    # cek login & role
│   │   └── index.jsx
│   ├── components/               # tabel, modal, form, layout, editor
│   ├── features/
│   │   ├── auth/
│   │   ├── dashboard/
│   │   ├── master/               # tahun ajaran, kelas, mapel, siswa
│   │   ├── lessonPlans/
│   │   ├── materials/
│   │   ├── attendance/
│   │   ├── journals/
│   │   ├── assessments/          # asesmen, kisi-kisi, input nilai
│   │   ├── questionBank/
│   │   ├── itemAnalysis/
│   │   ├── followups/            # remedial & pengayaan
│   │   ├── counseling/
│   │   ├── reports/
│   │   └── admin/                # pengguna, peran, audit log
│   ├── hooks/                    # useAssignments, useStudents, dst
│   └── styles/
├── supabase/
│   ├── migrations/               # file .sql berurutan
│   └── functions/
│       ├── create-user/index.ts
│       ├── reset-password/index.ts
│       └── import-students/index.ts
├── .env.example
├── vite.config.js
└── package.json
```

### 10.1 Pustaka yang direkomendasikan
| Kebutuhan | Pustaka |
|---|---|
| Routing | `react-router-dom` |
| Data & cache | `@tanstack/react-query` |
| Form & validasi | `react-hook-form` + `zod` |
| UI | Tailwind CSS (+ komponen headless, mis. Radix/shadcn) |
| Tabel | `@tanstack/react-table` |
| Editor teks | `tiptap` |
| Ekspor | `xlsx`, `jspdf`/`@react-pdf/renderer`, `docx` |
| Grafik | `recharts` |

### 10.2 Konfigurasi `vite.config.js`

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  base: '/',            // ganti './' atau '/subfolder/' bila dipasang di subfolder cPanel
  build: { outDir: 'dist', sourcemap: false },
});
```

### 10.3 Variabel lingkungan (`.env`)

```
VITE_SUPABASE_URL=https://xxxxxxxx.supabase.co
VITE_SUPABASE_ANON_KEY=eyJ...            # anon/public key SAJA
VITE_LOGIN_DOMAIN=ruangguru.local
```

> Variabel berawalan `VITE_` **tertanam di berkas JavaScript hasil build** dan dapat dilihat siapa pun. Itu aman untuk `anon key` *selama RLS benar*. **Jangan pernah** memasukkan `service_role key`.

### 10.4 Klien Supabase

```js
// src/lib/supabase.js
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY,
  { auth: { persistSession: true, autoRefreshToken: true } }
);
```

### 10.5 Proteksi rute

```jsx
// src/routes/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function ProtectedRoute({ roles, children }) {
  const { session, profile, roles: myRoles, loading } = useAuth();
  if (loading) return <div>Memuat...</div>;
  if (!session) return <Navigate to="/login" replace />;
  if (profile?.must_change_password) return <Navigate to="/ganti-password" replace />;
  if (roles && !roles.some(r => myRoles.includes(r))) return <Navigate to="/tidak-diizinkan" replace />;
  return children;
}
```

---

## 11. EDGE FUNCTION: PEMBUATAN AKUN

```ts
// supabase/functions/create-user/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

Deno.serve(async (req) => {
  const url = Deno.env.get('SUPABASE_URL')!;
  const serviceKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!; // secret server
  const admin = createClient(url, serviceKey);

  // 1) Verifikasi pemanggil
  const jwt = req.headers.get('Authorization')?.replace('Bearer ', '') ?? '';
  const { data: { user } } = await admin.auth.getUser(jwt);
  if (!user) return new Response('Unauthorized', { status: 401 });

  const { data: roles } = await admin.from('user_roles').select('role').eq('user_id', user.id);
  const allowed = roles?.some(r => ['admin_lembaga', 'super_admin'].includes(r.role));
  if (!allowed) return new Response('Forbidden', { status: 403 });

  // 2) Buat akun
  const { username, password, full_name, institution_id, role } = await req.json();
  const email = `${username.toLowerCase()}@${Deno.env.get('LOGIN_DOMAIN')}`;
  const { data, error } = await admin.auth.admin.createUser({
    email, password, email_confirm: true,
  });
  if (error) return new Response(error.message, { status: 400 });

  await admin.from('profiles').insert({
    id: data.user.id, username, full_name, institution_id, must_change_password: true,
  });
  await admin.from('user_roles').insert({ user_id: data.user.id, role });

  return new Response(JSON.stringify({ ok: true, id: data.user.id }), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

> Edge Function **tidak** di-hosting di cPanel; ia berjalan di Supabase. Di-*deploy* dengan Supabase CLI: `supabase functions deploy create-user`.
> Pastikan admin lembaga hanya dapat membuat akun untuk `institution_id` miliknya (tambahkan pemeriksaan ini pada implementasi akhir).

---

## 12. PANDUAN DEPLOY KE cPANEL

### 12.1 Mengapa kompatibel dengan cPanel
Hasil `npm run build` hanyalah berkas statis (HTML, CSS, JS). cPanel cukup berperan sebagai *web server statis*. Seluruh logika data berjalan di Supabase melalui HTTPS.

### 12.2 Langkah-langkah

**A. Persiapan Supabase**
1. Buat proyek di Supabase.
2. Jalankan seluruh SQL skema dan RLS (bagian 7–8) di SQL Editor.
3. Buat bucket Storage (privat) dan policy-nya.
4. Deploy Edge Function (`create-user`, dst).
5. Buat akun `super_admin` pertama (lewat dashboard Supabase lalu isi `profiles` dan `user_roles` secara manual).
6. Catat **Project URL** dan **anon key** (Settings → API).

**B. Build di komputer lokal**
```bash
npm install
cp .env.example .env     # isi URL, anon key, domain login
npm run build            # menghasilkan folder dist/
```

**C. Upload ke cPanel**
1. Masuk cPanel → **File Manager** → `public_html` (atau subdomain, mis. `guru.domainanda.com`).
2. Kosongkan berkas lama yang tidak diperlukan.
3. Upload **isi** folder `dist/` (bukan folder `dist` itu sendiri): `index.html`, folder `assets/`, `.htaccess`, dsb. Dapat di-zip, upload, lalu *Extract*.
4. Aktifkan **SSL** (Menu *SSL/TLS Status* → *Run AutoSSL*).

**D. Berkas `.htaccess` (wajib untuk React Router)**
Tanpa ini, membuka/refresh alamat seperti `/dashboard` akan menghasilkan error 404.

```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>

# Paksa HTTPS
<IfModule mod_rewrite.c>
  RewriteCond %{HTTPS} off
  RewriteRule ^(.*)$ https://%{HTTP_HOST}/$1 [L,R=301]
</IfModule>

# Cache aset ber-hash, jangan cache index.html
<IfModule mod_headers.c>
  <FilesMatch "\.(js|css|png|jpg|svg|woff2)$">
    Header set Cache-Control "public, max-age=31536000, immutable"
  </FilesMatch>
  <FilesMatch "index\.html$">
    Header set Cache-Control "no-cache"
  </FilesMatch>
</IfModule>
```
Letakkan berkas ini di folder `public/` proyek agar otomatis tersalin ke `dist/` saat build.

**E. Jika dipasang di subfolder** (mis. `domain.com/guru/`)
- Set `base: '/guru/'` di `vite.config.js`.
- Set `<BrowserRouter basename="/guru">`.
- Ubah `RewriteBase /guru/` dan `RewriteRule . /guru/index.html [L]`.

**F. Pengaturan Supabase setelah domain aktif**
- Authentication → URL Configuration: isi **Site URL** dengan domain Anda.
- Tambahkan domain ke daftar URL yang diizinkan (redirect URLs).

### 12.3 Alur pembaruan aplikasi
```
Ubah kode → npm run build → upload ulang isi dist/ → (hapus cache browser bila perlu)
```
Disarankan menyimpan kode di GitHub dan memelihara catatan versi; opsi lanjut: otomatisasi unggah (GitHub Actions + FTP/SFTP deploy) bila paket cPanel mendukung.

### 12.4 Pemecahan masalah umum
| Gejala | Penyebab | Solusi |
|---|---|---|
| Halaman putih | `base` salah / aset tidak ditemukan | Periksa `base` dan jalur upload `assets/` |
| 404 saat refresh | `.htaccess` belum ada | Unggah `.htaccess` di folder yang sama dengan `index.html` |
| "Invalid API key" | `.env` salah saat build | Perbaiki `.env`, **build ulang**, upload ulang |
| Data kosong padahal ada | Policy RLS terlalu ketat | Uji policy per role lewat SQL Editor |
| Gagal login | Domain pseudo-email berbeda | Samakan `VITE_LOGIN_DOMAIN` dengan yang dipakai saat membuat akun |
| CORS error | URL Supabase salah | Periksa `VITE_SUPABASE_URL` |

> Karena aplikasi ini **tidak memerlukan Node.js di server**, paket shared hosting cPanel standar sudah cukup.

---

## 13. ALUR KERJA UTAMA (USER JOURNEY)

### 13.1 Guru mengajar satu pertemuan
```
Login → Dashboard (jadwal hari ini)
  → Buka kelas → Isi kehadiran
  → Isi jurnal (otomatis terisi topik & kehadiran)
  → (opsional) Unggah materi / catat tugas
```

### 13.2 Siklus asesmen
```
Buat asesmen → Susun kisi-kisi → Pilih/buat soal → Cetak naskah
  → Pelaksanaan → Input nilai & jawaban butir
  → Analisis soal → Tandai remedial/pengayaan
  → Laksanakan tindak lanjut → Catat hasil → Rekap nilai akhir
```

### 13.3 Onboarding lembaga baru
```
super_admin membuat lembaga → admin_lembaga dibuat
  → admin: tahun ajaran & semester → kelas → mapel → siswa (impor XLSX)
  → admin: buat akun guru → penugasan mengajar
  → guru mulai bekerja
```

---

## 14. KEBUTUHAN NON-FUNGSIONAL

| Aspek | Target |
|---|---|
| Kinerja | Halaman utama < 3 detik pada koneksi 4G; tabel besar memakai pagination/virtualisasi |
| Responsif | Nyaman di ponsel (input kehadiran dan jurnal prioritas) |
| Ketersediaan | Mengikuti SLA paket Supabase dan hosting |
| Keamanan | RLS, HTTPS, audit log, pembatasan peran |
| Pemeliharaan | Migrasi SQL berurutan di `supabase/migrations`, versi dokumentasi |
| Aksesibilitas | Kontras warna memadai, navigasi keyboard, dukungan RTL untuk Arab |
| Pencadangan | Ekspor berkala database + unduh file Storage penting |
| Skalabilitas | Indeks pada kolom relasi; pertimbangkan paket Supabase berbayar untuk banyak lembaga |

> **Catatan paket gratis Supabase:** proyek gratis dapat dijeda bila tidak aktif dalam jangka waktu tertentu, dan kuota penyimpanan/transfer terbatas. Periksa batas terbaru di situs resmi Supabase sebelum produksi.

---

## 15. RENCANA PENGEMBANGAN BERTAHAP

| Fase | Isi | Hasil |
|---|---|---|
| **1. Fondasi** | Skema DB, RLS, login username, manajemen pengguna, master data, layout & dashboard dasar | Sistem bisa dipakai login dan mengelola data induk |
| **2. Administrasi inti** | Kehadiran, jurnal harian, jadwal, rekap + ekspor | Administrasi harian guru berjalan |
| **3. Perencanaan** | Modul ajar/RPS dengan template dinamis, materi ajar, alur persetujuan | Perangkat ajar terdigitalisasi |
| **4. Penilaian** | Bank soal, kisi-kisi, asesmen, input nilai, leger | Siklus penilaian lengkap |
| **5. Analisis & tindak lanjut** | Analisis soal, remedial, pengayaan | Penilaian berbasis data |
| **6. BK** | Kasus, sesi, privasi berlapis, laporan agregat | Layanan BK terdokumentasi aman |
| **7. Penyempurnaan** | Notifikasi, audit log, akun siswa/wali, PWA (instal di ponsel) | Siap produksi |

Saran: rilis per fase dan uji dengan 3–5 guru pengguna awal sebelum fase berikutnya.

---

## 16. RISIKO DAN MITIGASI

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Kebocoran data antarlembaga/antarguru | Tinggi | RLS ketat + pengujian otomatis per role |
| Kebocoran data BK | Sangat tinggi | Pemisahan kolom rahasia, akses minimum, audit log |
| `service_role key` terekspos | Sangat tinggi | Hanya di Edge Function, jangan di frontend/repo |
| Lupa password (tanpa email) | Sedang | Reset oleh admin; opsi email pemulihan |
| Variasi kurikulum antar jenjang | Sedang | Template dinamis berbasis JSON, bukan kode tetap |
| Batas paket gratis | Sedang | Pantau kuota; rencana upgrade; backup rutin |
| Kompleksitas terlalu besar sekaligus | Tinggi | Rilis bertahap sesuai fase |
| Aturan kurikulum berubah | Sedang | Template & rumus dapat dikonfigurasi, bukan hardcode |

---

## 17. HAL YANG PERLU DIPUTUSKAN SEBELUM IMPLEMENTASI

1. **Model lembaga:** satu lembaga saja, atau multi-lembaga (multi-tenant)?
2. **Akun siswa/wali:** dibutuhkan sejak awal atau tahap lanjut?
3. **Template resmi:** format modul ajar/RPS/jurnal yang berlaku di lembaga sasaran (agar template dapat dibuat sesuai).
4. **Rumus penilaian:** bobot komponen, KKM/KKTP, aturan nilai remedial.
5. **Kebijakan privasi BK:** siapa saja yang berhak melihat ringkasan.
6. **Kebutuhan ekspor:** format kop dan layout cetak yang diwajibkan.
7. **Spesifikasi hosting cPanel:** versi PHP tidak berpengaruh, tetapi pastikan SSL aktif dan akses File Manager/FTP tersedia.
8. **Paket Supabase:** gratis untuk uji coba atau berbayar untuk produksi.

---

## 18. LANGKAH SELANJUTNYA YANG DISARANKAN

1. Konfirmasi poin pada bagian 17.
2. Buat proyek Supabase dan jalankan skema bagian 7–8 (mulai dari tabel inti saja).
3. Inisialisasi proyek: `npm create vite@latest ruang-guru -- --template react`.
4. Bangun **Fase 1** (login username, role, master data).
5. Uji deploy ke cPanel sejak fase awal (cukup halaman login) untuk memastikan `.htaccess`, SSL, dan koneksi Supabase berfungsi sebelum menambah fitur.

---

*Dokumen ini adalah rancangan arsitektur dan skema awal. Kode contoh bersifat ilustratif dan belum diuji pada lingkungan Anda; sesuaikan, uji kebijakan RLS per role, dan tinjau ulang aspek keamanan serta kepatuhan perlindungan data sebelum dipakai untuk data nyata.*
