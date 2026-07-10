# Riset: Mekanisme Hitungan Right Issue (HMETD) di IDX

Tanggal: 2026-07-09
Status: terverifikasi 3 lapis — riset sumber resmi (Sonnet) → cross-check aritmetika 6 kasus (Haiku) → adversarial verify + edge case (Opus). Semua rumus match harga teoritis resmi BEI di 5 kasus nyata.

## Ringkasan Rumus

| Besaran | Rumus | Catatan |
|---|---|---|
| Harga teoritis ex-right (TERP) | `HT = (a×hc + b×hr) / (a+b)` | dibulatkan ke fraksi harga terdekat |
| Harga pasar 1 HMETD | `max(0, HT − hr)` | dari arbitrase; clamp 0 kalau hr ≥ HT |
| Nilai hak per saham lama | `hc − HT` | ekuivalen `(b/a)(HT−hr)`, bukan konvensi berbeda |
| Dilusi non-penebus | `b / (a+b)` | konvensi prospektus "dilusi maksimal X%" |
| Alokasi HMETD | `floor(s × b / a)` | pecahan dibuang (dijual emiten, POJK 32/2015 Ps. 33) |

Notasi: `a` = rasio saham lama, `b` = rasio saham baru, `hc` = harga penutupan cum date, `hr` = harga pelaksanaan (exercise), `s` = jumlah lembar dipegang.

## 1. Harga Teoritis Ex-Right (TERP)

Rumus resmi BEI (Buku Panduan Indeks Harga Saham BEI, formula #10):

```
HT = (a × hc + b × hr) / (a + b)
```

Contoh resmi dari dokumen IDX: rasio 3:2, cum Rp1.250, exercise Rp1.000 → HT = (3×1.250 + 2×1.000)/5 = **Rp1.150**.

### Pembulatan ke fraksi harga

Hasil mentah dibulatkan ke **kelipatan fraksi terdekat** (nearest, bukan floor). Kutipan dokumen resmi: *"Jika penghitungan Harga Teoritis tidak menghasilkan angka yang berada dalam fraksi harga tersebut, maka Harga Teoritis akan dibulatkan ke atas atau ke bawah sesuai dengan nilai terdekat."*

Tabel fraksi IDX (Peraturan II-A, efektif 2 Mei 2016 — **masih berlaku**, dikonfirmasi kasus PANI Des 2025):

| Rentang harga | Fraksi |
|---|---|
| < Rp200 | Rp1 |
| Rp200 – <Rp500 | Rp2 |
| Rp500 – <Rp2.000 | Rp5 |
| Rp2.000 – <Rp5.000 | Rp10 |
| ≥ Rp5.000 | Rp25 |

⚠️ Buku Panduan Indeks versi 2010 memuat tabel fraksi **lama** (1/5/10/25/50) — usang, jangan dipakai.

Konvensi implementasi (hasil adversarial verify):
- **Fraksi ditentukan dari HT mentah**, bukan dari harga cum (kasus hc=5.100 tapi HT_raw=4.988 → pakai tick Rp10, bukan Rp25).
- **Tie-breaking half-up** (`floor(raw/t + 0.5)×t`) — konvensi ini mereproduksi 5/5 kasus resmi; kasus tepat di tengah praktis tak pernah terjadi.
- **Batas rentang: bawah inklusif, atas eksklusif** (rantai `if (p<200)… else if (p<500)…` otomatis benar).

## 2. Nilai Teoritis HMETD

Dua sudut pandang, **secara aljabar identik** (dari identitas `a·hc + b·hr = (a+b)·HT` ⇒ `a(hc−HT) = b(HT−hr)`):

- **Per saham lama**: `hc − HT` — nilai yang "lepas" dari harga saham saat ex date. Bentuk ini yang lazim di edukasi/prospektus Indonesia.
- **Per 1 HMETD** (harga pasar right saat diperdagangkan): `HT − hr` — dari arbitrase: beli 1 HMETD seharga v + tebus hr → dapat 1 saham senilai HT ⇒ v = HT − hr.

**Jangan tertukar** — ini beda dimensi. 1 saham lama membawa `b/a` hak, jadi `hc − HT = (b/a)(HT − hr)`. Untuk menghitung hasil **jual HMETD**, pakai `r × (HT − hr)` (r = jumlah hak), BUKAN `r × (hc − HT)`. Salah pilih di sini meleset besar (uji PANI: selisih Rp46.450 vs Rp12.125 pada posisi 1.000 lembar).

Jika `hr ≥ HT` (⇔ hr ≥ hc): HMETD **tak bernilai** — clamp ke 0, jangan tebus (bayar hr untuk saham senilai HT < hr = rugi langsung).

## 3. Dilusi

```
% dilusi = b / (a + b)
```

Bagi pemegang yang tidak menebus. Konvensi prospektus Indonesia ("dilusi maksimal X%"). Konfirmasi kasus riil: INET rasio 3:4 → 4/7 = 57,14%; WIFI 4:5 → 5/9 = 55,56% — persis angka prospektus.

## 4. Alokasi HMETD & Pecahan

- 1 HMETD = hak beli 1 saham baru di harga `hr`.
- Alokasi: `r = floor(s × b / a)` — pecahan dibulatkan **ke bawah**; hak atas pecahan wajib dijual perusahaan, hasilnya masuk rekening perseroan (POJK 32/2015 Pasal 33).
- Contoh BBRI 2021 (rasio 100:23): 1.000 lembar → 230,12 → **230 HMETD**.
- Implementasi: jangan `Math.floor(s*b/a)` langsung (risiko float dust `x.9999…`); pakai pembagian integer: `(n - n%a)/a` dengan `n = s×b`, atau BigInt bila `s×b ≥ 2^53`.

## 5. Tiga Skenario Pemegang Saham

Posisi awal: `s` lembar, nilai `s×hc`. Setelah ex date harga teoritis jadi `HT`. `r = floor(s×b/a)`.

| Skenario | Rumus | P&L teoritis vs pre-RI |
|---|---|---|
| **Tebus** | modal tambahan `r×hr`; saham akhir `s+r`; nilai `(s+r)×HT`; avg proxy `(s·hc + r·hr)/(s+r)` | `(s+r)HT − s·hc − r·hr` ≈ **0** |
| **Jual HMETD** | proceeds `r×(HT−hr)`; nilai `s×HT + proceeds` | identik dengan Tebus (no-arbitrage) |
| **Diamkan** | nilai `s×HT` | `s×(HT−hc)` = **rugi sebesar nilai hak yang hangus** |

Bukti aljabar skenario Tebus ≈ 0: `s(HT−hc) + r(HT−hr)`, substitusi `r = s·b/a` dan identitas §2 → nol. Residu nyata hanya dari pembulatan tick + floor — di 5 kasus uji: antara −Rp5.700 (BBRI) sampai +Rp12.125 (PANI) per 1.000 lembar, < 0,5% posisi; positif bila HT dibulatkan naik, negatif bila turun. **Bukan bug, karakteristik pembulatan.**

Catatan avg proxy: memakai `hc` sebagai proxy cost lama — itu *blended effective price* untuk perbandingan, bukan cost basis riil pemegang.

## 6. Kasus Verifikasi (semua MATCH harga resmi BEI)

| Kasus | Rasio (a:b) | hc | hr | HT mentah | Fraksi | HT bulat | HT resmi |
|---|---|---|---|---|---|---|---|
| BBRI Sep 2021 | 1.000.000.000 : 230.128.553 | 3.910 | 3.400 | 3.814,59 | 10 | **3.810** | 3.810 ✓ |
| BRIS Des 2022 | 90.000 : 10.941 | 1.295 | 1.000 | 1.263,02 | 5 | **1.265** | 1.265 ✓ |
| BBTN Des 2022 | 100.000.000 : 32.525.443 | 1.405 | 1.200 | 1.354,69 | 5 | **1.355** | 1.355 ✓ |
| Contoh buku IDX | 3 : 2 | 1.250 | 1.000 | 1.150,00 | 5 | **1.150** | 1.150 ✓ |
| PANI Des 2025 | 50.831 : 3.646 | 13.900 | 12.975 | 13.838,09 | 25 | **13.850** | 13.850 ✓ |

BBRI dengan rasio disederhanakan 100:23 → HT_raw 3.814,63, bulat tetap 3.810 (selisih vs rasio penuh 0,04 — boleh pakai rasio sederhana).

## 7. Konvensi & Timeline (sebatas yang memengaruhi hitungan)

- **Notasi rasio "X : Y" = X saham lama : Y saham baru.** Konsisten di dokumen resmi IDX dan semua kasus dicek (BBRI, INET 3:4, WIFI 4:5, PANI) — tidak ditemukan sumber yang membalik arah.
- **Cum date** = hari bursa terakhir dengan hak melekat; `hc` = harga penutupan hari itu. **Ex date** = hari berikutnya; harga acuan di-adjust ke HT.
- **Periode perdagangan HMETD**: 5–10 hari kerja (POJK 32/2015 Ps. 34), mulai setelah distribusi (maks. 1 hari kerja pasca recording date, Ps. 31). Recording date = 8 hari kerja setelah pernyataan pendaftaran efektif (Ps. 30).
- **Hangus**: HMETD yang tidak ditebus/dijual sampai akhir periode perdagangan hangus, nilai nol.

## Sumber

- [Buku Panduan Indeks Harga Saham BEI](https://idx.co.id/media/1481/buku-panduan-indeks-2010.pdf) — formula TERP #10 (tabel fraksinya usang, abaikan)
- [POJK 32/POJK.04/2015](https://www.ojk.go.id/id/kanal/pasar-modal/regulasi/peraturan-ojk/Documents/Pages/pojk-32-penambahan-modal-pt-dengan-memberikan-hak-memesan-efek-terlebih-dahulu/SALINAN-POJK%20HMETD.pdf) — Ps. 30, 31, 33, 34, 36
- [Fraksi harga — NH Korindo Sekuritas](https://www.nhis.co.id/fraksi-harga-saham/)
- Kasus: [BBRI — CNBC Indonesia](https://www.cnbcindonesia.com/market/20210908093952-17-274493/jelang-rights-issue-harga-teoritis-bbri-ditetapkan-rp-3810) · [BRIS — Investor.id](https://investor.id/market-and-corporate/316293/rights-issue-harga-teoritis-saham-bsi-bris-dirilis) · [BBTN — Bisnis.com](https://market.bisnis.com/read/20221223/7/1611523/bei-umumkan-harga-teoritis-saham-bbtn-usai-rights-issue-rp1355) · [PANI — Indopremier](https://www.indopremier.com/ipotnews/newsDetail.php?jdl=Rights_Issue_di_Rp12_975__Segini_Harga_Teoritis_Saham_PANI&news_id=481184)
- [Stockbit Snips — HMETD](https://snips.stockbit.com/investasi/hmetd)
