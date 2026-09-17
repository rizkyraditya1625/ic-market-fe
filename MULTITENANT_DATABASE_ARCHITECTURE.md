# DOKUMENTASI DESAIN DATABASE POSTGRESQL MULTI-TENANT (SCHEMA-PER-TENANT)

> **Proyek**: iCraft Multi-Vendor Marketplace  
> **Pola Arsitektur**: Multi-Tenancy by Schema (Schema-per-Tenant Isolation)  
> **Script DDL SQL**: [`schema_multitenant_pgsql.sql`](./schema_multitenant_pgsql.sql)

---

## 1. Konsep Multi-Tenant by Schema di PostgreSQL

Dalam arsitektur **Schema-per-Tenant**, satu instance database PostgreSQL dibagi menjadi:
1. **`public` Schema (Control Plane / Platform Master)**: Berisi data global bersama, seperti autentikasi pengguna, registrasi & KYC onboarding toko, konfigurasi komisi, pembayaran global (*inflow*), buku besar escrow terpusat, dan batch payout (*outflow*).
2. **`tenant_<slug>` Schemas (Tenant Plane / Isolated Store)**: Setiap toko yang terdaftar memiliki schema database terpisah (misal: `tenant_craft_studio`, `tenant_pixel_art`). Data produk, detail teknis, gambar aset digital, pesanan lokal toko, dan kupon toko tersimpan secara terisolasi.

```mermaid
graph TD
    subgraph PostgreSQL Database Instance
        subgraph public_schema ["PUBLIC SCHEMA (Master / Platform Control Plane)"]
            U[public.users]
            OB[public.store_onboardings]
            T[public.tenants]
            CFG[public.system_settings]
            GO[public.global_orders]
            GOI[public.global_order_items]
            TW[public.tenant_wallets]
            WL[public.tenant_wallet_ledger]
            PO[public.payouts]
        end

        subgraph tenant_template_schema ["TENANT_TEMPLATE (Blueprint)"]
            TP[tenant_template.*]
        end

        subgraph tenant_a ["TENANT_STORE_A (Schema Toko A)"]
            PA[tenant_store_a.products]
            PDA[tenant_store_a.product_details]
            PDF_A[tenant_store_a.product_digital_files]
            OA[tenant_store_a.orders]
            CA[tenant_store_a.customers]
        end

        subgraph tenant_b ["TENANT_STORE_B (Schema Toko B)"]
            PB[tenant_store_b.products]
            PDB[tenant_store_b.product_details]
            PDF_B[tenant_store_b.product_digital_files]
            OB_T[tenant_store_b.orders]
            CB[tenant_store_b.customers]
        end
    end

    OB -->|Approved via provision_new_tenant()| T
    T -.->|Creates & Clones| tenant_a
    T -.->|Creates & Clones| tenant_b
    GOI -->|Routes order item to| OA
    GOI -->|Routes order item to| OB_T
```

---

## 2. Struktur Tabel `public` Schema (Master Data)

| Nama Tabel | Fungsi & Peran |
| :--- | :--- |
| **`public.users`** | Single Sign-On (SSO) untuk seluruh role (`buyer`, `seller`, `admin`, `finance`). |
| **`public.store_onboardings`** | Formulir KYC & verifikasi pengajuan pembukaan toko (NIK/NPWP/Dokumen). |
| **`public.tenants`** | Master list toko, routing nama schema (`schema_name`), domain kustom, & custom % komisi. |
| **`public.system_settings`** | Pengaturan komisi default platform (misal: 10%), jadwal payout (tgl 1 & 15), dan escrow policy. |
| **`public.global_categories`** | Taksonomi kategori induk marketplace untuk penjelajahan di homepage. |
| **`public.global_orders`** | Inflow transaksi pembayaran dari payment gateway (Xendit/Midtrans) sebelum dipecah ke toko. |
| **`public.global_order_items`** | Snapshot transaksi per item: persentase komisi, fee platform (admin), dan hak bersih toko. |
| **`public.tenant_wallets`** | Dompet terpusat toko: `balance_holding` (escrow), `balance_available` (siap cair), & `balance_withdrawn`. |
| **`public.tenant_wallet_ledger`** | Audit trail buku besar *double-entry* setiap mutasi saldo toko. |
| **`public.payouts`** | Tiket pencairan terjadwal dan eksekusi transfer dana ke rekening bank toko. |

---

## 3. Struktur Tabel `tenant_*` Schema (Isolated Tenant Data)

Setiap toko baru akan mendapatkan salinan tabel independen berikut:

| Nama Tabel | Keterangan |
| :--- | :--- |
| **`store_settings`** | Preferensi toko, jam operasional, pesan otomatis, kustomisasi banner/header. |
| **`categories`** | Kategori internal toko (dapat dihubungkan ke kategori global). |
| **`products`** | Master data produk toko, SKU, harga, status stok, flag produk digital, dan komisi override. |
| **`product_details`** | Deskripsi lengkap, tags spesifikasi teknis, SEO metadata. |
| **`product_images`** | Galeri gambar produk & urutan tampilan. |
| **`product_digital_files`** | Referensi aman ke object storage (S3 / Cloudflare R2) untuk file aset digital toko. |
| **`customers`** | Data pelanggan yang pernah bertransaksi di toko ini beserta riwayat total belanja (*LTV*). |
| **`orders`** | Catatan pesanan lokal toko dengan detail status pemrosesan internal (*fulfillment*). |
| **`order_items`** | Item pesanan toko & kuota/limit download aset digital. |
| **`coupons`** | Voucher diskon yang dibuat sendiri oleh toko untuk pelanggannya. |
| **`product_reviews`** | Ulasan, rating bintang 1-5, dan ulasan teks dari pembeli. |

---

## 4. Mekanisme Otomatisasi & Dynamic Provisioning

Ketika admin menyetujui pengajuan toko (*Onboarding Approved*), backend cukup memanggil fungsi PostgreSQL yang telah disediakan:

```sql
SELECT public.provision_new_tenant(
    p_owner_user_id   := 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11',
    p_store_name      := 'Studio Desain Kreatif',
    p_slug            := 'studio-kreatif',
    p_bank_name       := 'BCA',
    p_bank_acc_no     := '8830123456',
    p_bank_acc_holder := 'Rifki Ahmad',
    p_custom_commission := 7.50 -- Opsional: Komisi khusus 7.5%
);
```

### Apa yang Dilakukan Fungsi Tersebut:
1. Mendaftarkan entitas toko di `public.tenants`.
2. Menginisialisasi dompet toko di `public.tenant_wallets`.
3. Menjalankan DDL: `CREATE SCHEMA tenant_studio_kreatif;`.
4. Mengkloning struktur tabel dari `tenant_template.*` ke `tenant_studio_kreatif.*`.
5. Mengaktifkan foreign key constraints internal pada schema toko yang baru.

---

## 5. Cara Backend Mengakses Tenant Schema (Search Path Switching)

Dalam aplikasi backend (Node.js/Prisma/Knex/TypeORM/Go/Laravel), isolasi data dilakukan secara instan dengan mengubah `search_path` pada connection session:

```typescript
// Contoh di Backend (Node.js / Express / Nuxt Server Nitro)
async function getTenantDatabase(schema_name: string) {
    const client = await pool.connect();
    
    // Set schema aktif untuk session transaksi ini
    await client.query(`SET search_path TO "${schema_name}", public;`);
    
    return client;
}

// Handler request Seller untuk melihat produk tokonya sendiri:
app.get('/api/seller/products', async (req, res) => {
    const userTenant = req.user.tenant; // misal: 'tenant_studio_kreatif'
    const db = await getTenantDatabase(userTenant.schema_name);
    
    try {
        // Query ini otomatis membaca tabel produk milik toko yang bersangkutan
        const result = await db.query('SELECT * FROM products ORDER BY created_at DESC');
        res.json(result.rows);
    } finally {
        db.release();
    }
});
```

---

## 6. Keuntungan Arsitektur Ini

1. **Keamanan & Isolasi Data Tinggi**: Setiap penjual tidak memiliki akses fisik ke data toko lain (kebocoran query data toko lain tercegah secara native di level database).
2. **Skalabilitas & Kemudahan Maintenance**: Toko besar (enterprise merchant) dapat dengan mudah dipindahkan (*backup / restore / migrate*) ke database server terpisah hanya dengan mengekspor schema toko tersebut (`pg_dump -n tenant_x`).
3. **Pusat Kontrol Finansial Kuat**: Semua urusan uang, potongan komisi sistem, penahanan escrow, dan pencairan rekening bank tetap tersentralisasi di `public` schema demi konsistensi dan auditabilitas.
