# Analysis Results

Hasil analisis statistik perbandingan performa **REST vs tRPC** pada sistem e-commerce ZENIT, dihasilkan oleh ZENIT Analysis Pipeline dari raw data k6 load testing.

## Isi Repository

```
.
├── ZENIT_Analysis_Raw.json       # Data mentah hasil analisis (semua angka, sebelum diformat)
├── ZENIT_Analysis_Report.xlsx    # Laporan terformat — sumber utama untuk skripsi
└── charts/                       # 86 file PNG hasil visualisasi
```

| File | Fungsi |
|---|---|
| `ZENIT_Analysis_Raw.json` | Sumber kebenaran (source of truth). Semua angka di Excel dan chart diturunkan dari sini. Dipakai kalau butuh nilai presisi penuh atau mau cross-check. |
| `ZENIT_Analysis_Report.xlsx` | Versi siap baca/kutip, sudah dipecah per sheet sesuai kebutuhan bab skripsi. |
| `charts/` | Visualisasi PNG, dipakai untuk gambar di Bab 4–5. |

---

## Desain Eksperimen

| Skenario | Deskripsi | Endpoint utama |
|---|---|---|
| S01_BROWSE | Browsing produk (read-only) | product list/filter/search/detail |
| S02_SHOPPING | Alur belanja (cart) | browse, cart add/get/update/remove, login |
| S03_CHECKOUT | Alur checkout | browse, cart, checkout, login, order detail |
| S04_AUTH | Autentikasi | login, logout, me, refresh |
| S05_ADMIN | Operasi admin | dashboard, products, orders, users |

**Test type** (per skenario):

| Test Type | N (run) | Tujuan |
|---|---|---|
| `load` | 10 | Beban normal, confirmatory — sumber utama uji hipotesis |
| `soak` | 1 | Beban normal durasi panjang — deteksi memory leak (slope regression) |
| `spike` | 3 | Lonjakan beban mendadak — eksploratoris, df=2 |
| `stress` | 3 | Beban di atas kapasitas — eksploratoris, df=2 |

**Condition** (khusus untuk decomposition analysis di S02–S05):

- **C2** — kondisi baku (REST vs tRPC murni), dipakai di semua tabel utama
- **C3** — N=1, interim run untuk isolasi kontribusi auth terhadap selisih REST/tRPC
- **C4** — N=1, hanya ada di S02_Shopping

> Run dengan N=3 (spike/stress) dan N=1 (soak/C3/C4) bersifat eksploratoris, bukan confirmatory penuh. Lihat kolom `Sig*` / catatan power rendah di setiap tabel.

---

## Metrik yang Diukur (17 metrik per grup)

| Metrik | Satuan | Arah "baik" |
|---|---|---|
| P95 / P99 Latency | ms | lebih rendah |
| Avg Response Time | ms | lebih rendah |
| Throughput | req/s | lebih tinggi |
| CPU Usage (Node) | % | lebih rendah |
| CPU Usage (System) | % | lebih rendah |
| RAM Usage | MB | lebih rendah |
| SLA Breach Rate | proporsi | lebih rendah (threshold 5%) |
| Functional Error Rate | proporsi | lebih rendah (threshold 1%) |
| HTTP Req Failed Rate | proporsi | lebih rendah |
| HTTP Req Count | count | — (deskriptif) |
| Payload Avg | bytes | lebih rendah |
| DB Active Connections | count | — (deskriptif) |
| DB Cache Hit % | % | lebih tinggi |
| DB TPS Delta | tps | — (deskriptif) |
| DB Query Avg | ms | lebih rendah |
| Network I/O | KB/s | — (deskriptif) |

Per-endpoint P95 latency juga tersedia untuk tiap skenario (lihat sheet per-skenario di Excel atau `groups[*].endpoint_metrics` di JSON).

---

## Metodologi Statistik

| Uji | Kapan dipakai |
|---|---|
| **Shapiro-Wilk** | Cek normalitas distribusi selisih (diff scores) |
| **Paired t-test** | Jika distribusi normal (p > 0.05 pada Shapiro-Wilk) |
| **Wilcoxon Signed-Rank** | Jika distribusi tidak normal |
| **Cohen's d** | Effect size, magnitudo: trivial / kecil / sedang / besar |
| **Bootstrap CI (95%)** | Confidence interval mean difference, validasi independen dari p-value |

Kolom `Sig?` di tabel = signifikan secara statistik (p < 0.05) **dan** bermakna secara praktis (CI tidak mencakup nol). Untuk N=3 (spike/stress), signifikansi diberi label "eksploratoris" karena power sangat rendah (df=2).

---

## Panduan Sheet Excel

| Sheet | Isi |
|---|---|
| `Summary` | Tabel utama — semua metrik, semua skenario, semua test type dalam satu tabel panjang |
| `CrossScenario` | Perbandingan 5 metrik primer (P95, Throughput, Avg RT, CPU, RAM) lintas 5 skenario, dikelompokkan per test type |
| `EffectSizeMatrix` | Matriks Cohen's d, untuk lihat pola effect size sekilas |
| `SoakComparison` | Perbandingan khusus hasil soak test antar skenario |
| `OrderEffects` | Hasil verifikasi counterbalancing (rest-first vs trpc-first) |
| `ChartDesc` | Deskripsi tiap file chart di folder `charts/` |
| `s0X_..._..._C2` (25 sheet) | Detail lengkap per kombinasi skenario × test type × condition |
| `Soak_s0X_...` (5 sheet) | Time series mentah soak test per skenario |
| `Decomposition` | Hasil decomposition analysis (kontribusi auth terhadap selisih REST/tRPC di C3/C4) |

---

## Struktur JSON

```jsonc
{
  "n_runs": 90,
  "n_groups": 25,
  "groups": {
    "s01_browse__load__C2": {
      "scenario": "s01_browse",
      "test_type": "load",
      "condition": "C2",
      "n": 10,
      "metrics": {
        "avg_rt": {
          "descriptive": { "rest": {...}, "trpc": {...}, "diff": {...} },
          "shapiro_wilk": {...},
          "test_used": "paired_ttest",
          "inferential": { "p": ..., "significant": ... },
          "cohens_d": { "d": ..., "magnitude": ... },
          "bootstrap_ci": { "ci_lower": ..., "ci_upper": ..., "covers_zero": ... },
          "conclusion": "..."
        },
        // ... 16 metrik lainnya
      },
      "endpoint_metrics": { /* per-endpoint P95 */ },
      "errors": { /* error breakdown */ }
    }
    // ... 24 grup lainnya
  },
  "decompositions": { /* khusus C3/C4 */ },
  "order_effects": { /* counterbalancing check */ },
  "counterbalancing_check": {...},
  "duplicate_runs": {...},
  "summary_stats": { "load_c2": {...}, "soak": {...}, "order_effects_clean": {...} },
  "interpretation": {
    "overall_summary": "...",
    "research_questions": {...},
    "patterns": [...],
    "group_summaries": {...},
    "chart_descriptions": {...}
  }
}
```

Key metrik di `metrics`: `avg_rt`, `med_rt`, `p90`, `p95`, `p99`, `min_rt`, `max_rt`, `throughput`, `functional_error`, `http_req_failed`, `sla_breach`, `checks_pass_rate`, `payload_bytes`, `http_count`, `cpu_pct`, `cpu_total_pct`, `mem_mb`, `pg_active`, `pg_cache_hit_ratio`, `pg_tps_delta`, `db_query_avg_ms`, `network_total_kb_s`.

---

## Folder `charts/`

86 file PNG, penamaan: `{skenario}_{test_type}_{condition}_{tipe_chart}.png`

| Tipe chart | Tersedia untuk | Isi |
|---|---|---|
| `comparison` | semua grup | Bar chart rata-rata REST vs tRPC per metrik |
| `cohens_d` | load/spike/stress C2 | Effect size tiap metrik |
| `boxplot` | load C2 saja (N=10) | Distribusi nilai per run |
| `paired_scatter` | load C2 saja | Scatter REST vs tRPC per run berpasangan |
| `percentile_profile` | load C2 saja | Profil P50/P90/P95/P99 |
| `forest_plot` | load/spike/stress C2 | Mean difference + CI, titik kiri garis nol = REST lebih rendah |
| `dotplot` | spike/stress C2 | Sebaran nilai REST vs tRPC per run |
| `bootstrap_ci` | load saja (per skenario, gabungan test type) | Distribusi bootstrap mean difference |
| `soak_timeseries` | soak saja | Tren metrik sepanjang durasi soak test |
| `ALL_sla_error_chart.png` | global | Ringkasan SLA breach rate & functional error rate semua skenario |

## Cara Pakai Cepat

- **Butuh angka untuk tabel di Bab 5** → buka `ZENIT_Analysis_Report.xlsx`, sheet `Summary` atau sheet per-skenario.
- **Butuh kalimat narasi siap parafrase** → sheet `Narasi` atau `interpretation.group_summaries` di JSON.
- **Butuh cross-check / nilai presisi penuh** → `ZENIT_Analysis_Raw.json`.
- **Butuh gambar untuk dimasukkan ke dokumen** → folder `charts/`, cocokkan nama file dengan sheet `ChartDesc` untuk tahu isi gambarnya sebelum dipakai.