# ✨ Claude Glow-Up

Plugin [Claude Code](https://claude.com/claude-code) untuk bikin Windows kamu **lega, ringan dan sehat**, dengan aman dan terukur.

## Install (2 perintah)

Buka Claude Code, lalu ketik:

```
/plugin marketplace add guidowilliam11/claude-glowup
/plugin install windows-glowup@claude-glowup
```

Lalu jalankan:

```
/windows-glowup bersihkan dan percepat laptop saya
```

> Disarankan pakai model **Opus 5.5**. Update ke versi terbaru: `/plugin marketplace update claude-glowup`.

## Apa yang dilakukan

1. **Survei paralel.** 4 agent jalan bersamaan: Disk, Proyek dev, RAM & Startup, Kesehatan. Semua read-only, dengan angka nyata.
2. **Laporan tiga kategori.** ✅ aman dihapus · ⚠️ kamu yang putuskan · ⛔ jangan disentuh (dengan alasannya). Tidak ada yang dihapus sebelum kamu memilih.
3. **Eksekusi aman.**
   - Cache dan artefak build (`node_modules`, `.next`, Rust `target`, Temp, cache browser) dihapus dengan cara yang tidak ikut menelusuri junction.
   - Perubahan sistem (service → Manual, startup off, uninstall, DISM/SFC, chkdsk) dibuatkan **skrip yang kamu jalankan sendiri sebagai Administrator**, lengkap dengan `undo.ps1`.
   - Update driver dan aplikasi lewat winget dengan daftar terkurasi (patch keamanan dulu, versi mayor database dan IDE tidak disentuh).
4. **Laporan sebelum → sesudah**, diukur ulang secara live.

## Hasil nyata (laptop pertama)

| | Sebelum | Sesudah |
|---|---|---|
| Ruang kosong C: | 25,8 GB | ~150 GB |
| RAM kosong saat idle | 3,2 GB | 8,2 GB |
| Disk C: | perlu perbaikan | healthy |
| Aplikasi usang | 74 | 37 diupdate |

## Tanpa plugin?

Copy isi [`PROMPT.md`](PROMPT.md) dan paste ke Claude Code.

## Keamanan

- Plugin ini tidak menyimpan dan tidak mengirim data ke mana pun. Semua skrip PowerShell bisa dibaca di [`plugins/windows-glowup/skills/windows-glowup/scripts`](plugins/windows-glowup/skills/windows-glowup/scripts).
- Baca dulu setiap skrip yang diminta untuk dijalankan sebagai Administrator.
- Dipakai dengan risiko sendiri. Backup data penting sebelum menghapus atau meng-uninstall apa pun.

## Lisensi

MIT
