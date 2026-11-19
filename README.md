# rajaemas-harga-api

REST API read-only untuk harga emas Raja Emas Indonesia. Sumber datanya adalah Google Sheets yang sudah dipublikasi sebagai CSV, lalu dinormalisasi menjadi JSON dan di-cache di edge Cloudflare.

**Base URL:** `https://api-harga.rajaemasindonesia.co.id`

- Runtime: Cloudflare Workers (tanpa dependency di runtime)
- Bahasa: JavaScript (ES modules)
- Tooling: [Wrangler](https://developers.cloudflare.com/workers/wrangler/) 4

## Struktur proyek

```
.
├── worker/                  # Service API (yang di-deploy)
│   ├── src/index.js         # Routing, fetch CSV, parsing, cache
│   ├── wrangler.toml        # Akun, nama Worker, custom domain
│   └── package.json
└── nextjs/                  # Contoh client (Next.js App Router), tidak di-deploy
    ├── lib/harga-emas.ts    # Typed client + tipe respons
    ├── components/TabelHarga.tsx
    └── app/harga/page.tsx
```

## Model data

API menyajikan **dua tabel yang independen**. Pembagian wilayah di kedua tabel **tidak sama**, misalnya Lombok masuk grup berbeda di tiap tabel. Karena itu ID wilayah selalu berlaku per tabel, dan data kedua tabel tidak boleh digabung berdasarkan ID wilayah.

| Tabel | Judul | Kolom item | ID wilayah |
|---|---|---|---|
| `karat` | Harga per Karat | `produk`, `harga`, `harga_format` | `jawa`, `bali-sumatera-lombok-bangka`, `kalimantan-sulawesi-indonesia-timur`, `batam` |
| `kami-beli` | Harga Kami Beli | `produk`, `kadar`, `harga`, `karat`, `harga_format` | `jawa`, `sumatera-bali`, `kalimantan-sulawesi-lombok`, `batam` |

Setiap pasangan tabel × wilayah berasal dari satu tab Google Sheets, sehingga totalnya 8 CSV. Mapping URL/`gid` ada di objek `TABEL` di `worker/src/index.js`.

## API reference

Semua endpoint hanya menerima `GET`, tidak memerlukan autentikasi, dan mengembalikan `application/json; charset=utf-8`.

| Method | Path | Deskripsi |
|---|---|---|
| GET | `/` | Info service dan daftar endpoint |
| GET | `/wilayah` | Daftar wilayah per tabel |
| GET | `/harga` | Kedua tabel, semua wilayah |
| GET | `/harga/{tabel}` | Satu tabel, semua wilayah |
| GET | `/harga/{tabel}/{wilayah}` | Satu tabel, satu wilayah |

`{tabel}` bernilai `karat` atau `kami-beli`. `{wilayah}` adalah salah satu ID wilayah milik tabel tersebut (lihat tabel di atas).

### Contoh

```bash
curl -A "Mozilla/5.0" https://api-harga.rajaemasindonesia.co.id/harga/kami-beli/jawa
```

(Flag `-A` diperlukan karena aturan WAF di zona ini; lihat [Catatan](#catatan).)

```json
{
  "diperbarui": "2026-11-19T13:36:21.832Z",
  "sumber": "langsung",
  "mata_uang": "IDR",
  "satuan": "gram",
  "tabel": "kami-beli",
  "judul": "Harga Kami Beli",
  "kolom": ["produk", "kadar", "harga", "karat"],
  "wilayah": {
    "id": "jawa",
    "nama": "Pulau Jawa",
    "harga": [
      { "produk": "LM ANTAM BARU", "kadar": "99.99%", "karat": "K24", "harga": 2190000, "harga_format": "Rp 2.190.000" },
      { "produk": "PERAK", "kadar": "99.99%", "karat": null, "harga": 18000, "harga_format": "Rp 18.000" }
    ]
  }
}
```

### Bentuk respons

Field meta muncul di semua respons `/harga*`:

| Field | Tipe | Keterangan |
|---|---|---|
| `diperbarui` | string (ISO 8601, UTC) | Waktu data diambil dari Google Sheets |
| `sumber` | `"langsung"` \| `"cadangan"` | `cadangan` berarti minimal satu sheet gagal diambil dan data untuk sheet itu diambil dari snapshot sukses terakhir |
| `mata_uang` | `"IDR"` | |
| `satuan` | `"gram"` | |

Struktur payload per endpoint:

- `/harga`: `tabel: { karat: Tabel, "kami-beli": Tabel }`
- `/harga/{tabel}`: `tabel`, `judul`, `kolom`, `wilayah: Wilayah[]`
- `/harga/{tabel}/{wilayah}`: `tabel`, `judul`, `kolom`, `wilayah: Wilayah`

```ts
type Tabel   = { judul: string; kolom: string[]; wilayah: Wilayah[] };
type Wilayah = { id: string; nama: string; harga: Item[] };

type ItemKarat    = { produk: string; harga: number; harga_format: string };
type ItemKamiBeli = { produk: string; kadar: string; karat: string | null; harga: number; harga_format: string };
```

- `harga` selalu berupa integer rupiah. Parser hanya mengambil digit dari isi sel, sehingga `Rp 2,190,000` dan `Rp 245.000` sama-sama terbaca benar. Implikasinya, nilai desimal (sen) **tidak** didukung.
- `karat` bernilai `null` bila sel kosong atau berisi `-`.
- Spasi berlebih di nama produk dirapikan, misalnya `LM  NON SERTIF` menjadi `LM NON SERTIF`.

Definisi tipe lengkap ada di `nextjs/lib/harga-emas.ts`.

### Error

Format error: `{ "error": "<pesan>" }`.

| Status | Kondisi |
|---|---|
| 404 | Path tidak dikenal, `{tabel}` tidak valid, atau `{wilayah}` tidak ada di tabel tersebut |
| 405 | Method selain `GET`/`HEAD`/`OPTIONS` |
| 502 | Semua sheet gagal diambil dan tidak ada snapshot cadangan |

### CORS dan cache HTTP

- `Access-Control-Allow-Origin: *`, sehingga API bisa dipanggil langsung dari browser.
- `Cache-Control: public, max-age=60` untuk data `langsung`, dan `no-store` untuk data `cadangan` serta error.

## Caching dan fallback

1. Hasil gabungan 8 sheet disimpan di Cache API Cloudflare (`caches.default`) dengan TTL **300 detik**.
2. Saat cache miss, ke-8 CSV diambil paralel dengan timeout 10 detik per request dan cache-buster `cb=<timestamp>`.
3. Jika semua sheet sukses, snapshot disimpan sebagai cadangan dengan TTL **7 hari**.
4. Jika sebagian sheet gagal, data sheet tersebut diisi dari snapshot cadangan, `sumber` diset ke `cadangan`, dan hasilnya hanya di-cache **60 detik** agar segera dicoba ulang.

Cache API Cloudflare bersifat **per data center**. Artinya, pengunjung di lokasi berbeda bisa melihat `diperbarui` yang berbeda, dan snapshot cadangan tidak tersinkron antar lokasi. Perubahan di sheet akan terlihat paling lambat sekitar 5 menit kemudian, ditambah `max-age` 60 detik di sisi client.

## Pengembangan lokal

Prasyarat: Node.js 20+.

```bash
cd worker
```

```bash
npm install
```

```bash
npm run dev
```

Server berjalan di `http://localhost:8787`. Mode dev memakai sheet produksi, karena tidak ada sumber data terpisah.

## Deploy

```bash
npx wrangler login
```

```bash
npm run deploy
```

Konfigurasi di `worker/wrangler.toml`:

- `account_id`: akun Cloudflare tujuan. Ini wajib karena login yang dipakai memiliki lebih dari satu akun.
- `routes`: custom domain `api-harga.rajaemasindonesia.co.id`. DNS record dan sertifikat dibuat otomatis oleh Wrangler.
- Subdomain `*.workers.dev` nonaktif selama `routes` terpasang. Tambahkan `workers_dev = true` jika subdomain itu tetap dibutuhkan.

## Konfigurasi sumber data

Semua konfigurasi ada di bagian atas `worker/src/index.js`:

- `SHEET_KARAT`, `SHEET_BELI`: base URL CSV publik tiap spreadsheet (tanpa `gid`).
- `TABEL`: definisi tabel, yaitu judul, kolom, fungsi parser, dan daftar wilayah (`id`, `nama`, URL CSV lengkap).

Parser mencari kolom berdasarkan kata kunci di baris header (`produk`, `kadar`, `harga`/`price`, `karat`), dengan fallback ke posisi kolom default. Urutan kolom di sheet boleh berubah selama nama header tetap memuat kata kunci tersebut.

Setelah mengubah konfigurasi, jalankan `npm run deploy`. Jika bentuk data cadangan berubah, naikkan versi `KEY_DATA`/`KEY_BACKUP` (misalnya `harga-v2` menjadi `harga-v3`) agar cache lama tidak terbaca.

## Contoh client: Next.js

Salin `nextjs/lib`, `nextjs/components`, dan `nextjs/app/harga` ke proyek Next.js (App Router), lalu set:

```bash
# .env.local
RAJAEMAS_API_URL=https://api-harga.rajaemasindonesia.co.id
```

- Data di-fetch di server dengan `revalidate: 300` dan tag `harga-emas`. Panggil `revalidateTag("harga-emas")` untuk invalidasi manual.
- `TabelHarga.tsx` mengekspor `<TabelKarat>` dan `<TabelKamiBeli>`, masing-masing dengan selector wilayahnya sendiri. Komponen ini tidak membawa styling.

Contoh client minimal tanpa framework:

```js
const res = await fetch("https://api-harga.rajaemasindonesia.co.id/harga/karat/jawa");
const { wilayah } = await res.json();
for (const item of wilayah.harga) console.log(item.produk, item.harga_format);
```

## Catatan

- **WAF zona `rajaemasindonesia.co.id`.** Security rules di level zona juga berlaku untuk subdomain API. Saat ini request dengan User-Agent `curl` atau tanpa User-Agent, serta request `HEAD`/`OPTIONS`, diblokir dengan 403 sebelum mencapai Worker. Request `GET` dari browser dan dari runtime Node (`fetch`) lolos. Bila API akan dikonsumsi oleh client lain (mobile app, cron, integrasi pihak ketiga), tambahkan pengecualian untuk host `api-harga.rajaemasindonesia.co.id` di Cloudflare Dashboard → Security → WAF.
- **Tidak ada test otomatis.** Parser (`parseCSV`, `parseHarga`, `mapKarat`, `mapKamiBeli`) diekspor dari `src/index.js` agar mudah diuji unit bila diperlukan.
- **Limit Workers Free:** 100.000 request per hari. Karena hasil di-cache, request ke Google Sheets maksimal sekitar 8 per 5 menit per data center.
