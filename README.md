# Semantic Search & LLM Application (FAQ E-commerce)

Prototype asisten customer service yang menjawab pertanyaan pelanggan berdasarkan FAQ. Pencariannya pakai semantic search (embedding Gemini + FAISS), lalu jawabannya disusun oleh LLM Gemini.

## Problem Statement

**Objective:** tim customer service e-commerce dapat ratusan pertanyaan berulang tiap hari. Proyek ini mencoba menjawabnya otomatis dari FAQ yang sudah ada, walau pelanggan tidak pakai kata-kata yang sama persis dengan FAQ.

**Knowledge base:** `data/ecommerce_faq_knowledge_base.csv`, berisi 59 entri FAQ dengan kolom `question`, `answer`, dan `category` (misalnya Pengiriman, Pengembalian, Promo).

**Batasan:**
- Sistem hanya menjawab dari FAQ. Kalau infonya tidak ada, pelanggan diarahkan ke customer service.
- Knowledge base tidak diubah.
- Tidak ada chunking, karena tiap FAQ sudah pendek.

## Alur Sistem

1. **Ingest:** load CSV pakai Pandas, lalu gabungkan question dan answer jadi satu teks per entri.
2. **Embed:** tiap teks diubah jadi vektor pakai `gemini-embedding-001` (59 vektor, 3072 dimensi).
3. **Store:** semua vektor disimpan di index FAISS (`IndexFlatL2`), tersimpan di `vector_db/`.
4. **Retrieve:** query pengguna diubah jadi embedding, lalu dicari 3 FAQ terdekat beserta jaraknya.
5. **Generate:** 3 FAQ itu disisipkan ke prompt (Persona, Task, Context, Format), lalu dikirim ke LLM untuk menghasilkan jawaban.

## Hasil Pengujian

Jarak memakai L2, jadi makin kecil makin mirip.

| Query | Tipe | FAQ teratas | Jarak |
|---|---|---|---|
| Apakah saya bisa membayar dengan cicilan? | Ada di FAQ | Apakah saya bisa membayar dengan cicilan? | 0.20 |
| Bagaimana jika paket saya belum sampai melebihi estimasi? | Ada di FAQ | (sama persis) | 0.16 |
| Boleh nggak bayarnya dicicil per bulan pakai kartu kredit? | Parafrase | Apakah saya bisa membayar dengan cicilan? | 0.25 |
| Barang yang datang pecah, uangnya bisa dikembalikan nggak? | Parafrase | Apa yang harus dilakukan jika paket diterima dalam kondisi rusak? | 0.45 |
| Bagaimana cara mendaftar jadi penjual di platform ini? | Tidak ada di FAQ | Bagaimana cara mendaftar akun baru? | 0.51 |
| Apakah ada promo hari ini? | Topik mirip | Bagaimana cara mendapatkan info promo terbaru? | 0.39 |

**Analisis singkat:**
- Query yang ada di FAQ dan query parafrase sama-sama menemukan FAQ yang benar, jadi semantic search menangkap makna, bukan cuma kata kunci.
- Query penjual menemukan FAQ "daftar akun baru". Kelihatan mirip, tapi maksudnya beda, dan memang tidak ada FAQ soal penjual. LLM tidak terkecoh dan menjawab bahwa infonya belum ada.
- Query "pecah": LLM menjawab cara melapor barang rusak, lalu jujur bilang info pengembalian uang tidak ada di FAQ.
- Query promo: topik FAQ-nya benar, tapi tidak ada daftar promo harian. LLM mengarahkan ke customer service.

Analisis lengkapnya ada di bagian akhir notebook.

## Keterbatasan

- Jarak tidak bisa jadi patokan "relevan atau tidak": query yang tidak ada di FAQ (0.51) hampir sama dengan parafrase yang benar (0.45).
- Top 3 selalu dikembalikan walau tidak ada yang cocok, jadi LLM yang menentukan infonya ada atau tidak.
- Jawaban LLM kadang lebih panjang dari batas 3-4 kalimat di prompt, dan kata-katanya bisa sedikit beda tiap dijalankan.
- Jawaban hanya sebagus isi FAQ-nya.

## Cara Menjalankan

1. Clone repo ini, lalu buat virtual environment.
2. Install library: `pip install -r requirements.txt`
3. Buat file `.env` di folder utama, isinya: `GEMINI_API_KEY=isi_api_key_kamu`
4. Buka `notebook/starter_notebook_semantic_search.ipynb`, pilih kernel dari environment tadi, lalu **Run All**.

## Struktur Folder

```
semantic-search-assignment/
├── data/                 # knowledge base (ecommerce_faq_knowledge_base.CSV)
├── notebook/             # notebook utama (starter_notebook_semantic_search.ipynb)
├── vector_db/            # FAISS index + embeddings
├── documentation/
├── README.md
└── requirements.txt
```