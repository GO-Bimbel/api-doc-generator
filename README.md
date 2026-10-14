# API Doc Generator

[Agent Skill](https://agentskills.io) yang membaca endpoint di modul backend (NestJS, Node.js, Go, atau .NET) lalu menulis dokumen DOCX berbahasa Indonesia yang mudah dipahami: request, response, query database, dan relasi tabel.

Dibuat untuk project backend di [GO-Bimbel](https://github.com/GO-Bimbel).

## Instalasi

Butuh Node.js dan salah satu dari bun, pnpm, atau npm. Install lewat CLI [skills.sh](https://skills.sh):

```sh
npx skills add GO-Bimbel/api-doc-generator
```

Tambahkan `-g` untuk install di semua project, atau `-a claude-code` untuk memilih agent. Update dengan `npx skills update`, hapus dengan `npx skills remove api-doc-generator`.

Dependency `docx` ter-install otomatis saat pertama kali dijalankan.

## Cara pakai

Jalankan agent dari folder yang berisi semua repo service, lalu:

```text
/api-doc-generator path/ke/target
/api-doc-generator path/ke/target --depth deep
```

- `single` (default): service target ditambah satu service yang dipanggilnya.
- `deep`: semua service dalam rantai panggilan, sampai handler terakhir.

Hasilnya disimpan di `<app-root>/docs/<nama-target>.json` dan `.docx`.

Untuk target NestJS, skill juga melengkapi anotasi `@nestjs/swagger` yang belum ada di controller project target (menambah `@ApiTags`, `@ApiOperation`, dan DTO proxy untuk parameter bertipe `any`). Perubahan baru ditulis setelah kamu setujui.

Alur lengkapnya ada di [`skills/api-doc-generator/SKILL.md`](skills/api-doc-generator/SKILL.md).
