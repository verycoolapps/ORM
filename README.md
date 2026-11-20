# ORTHO GENESIS

**Three Views. One Dimension. From Reference to Reality.**

ORTHO GENESIS (Orthographic Reconstruction Modeling) is an autonomous image-to-3D skill for Blender. Feed it orthographic reference views — front, side, top — and it reconstructs a validated, production-ready 3D model: calibrated scale, extracted silhouettes, class-aware geometry reconstruction, enforced quad-dominant topology, production modifiers, computed quality metrics, and exports in `.blend`, `.obj`, `.fbx`, `.stl`, and `.glb`.

---

## What It Does

1. **Reference analysis** — classifies each input as front/side/top, detects missing or mislabeled views, scores background complexity.
2. **Calibration** — solves shared `units_per_pixel` scale, assigns scene axes and ground plane, builds orthographic reference cameras. Halts on view inconsistency rather than guessing.
3. **Silhouette extraction** — OpenCV segmentation (Otsu / GrabCut / alpha), boundary extraction, world↔pixel projection functions.
4. **Geometry reconstruction** — cross-section lofting (hard-surface), voxel carving + remesh (organic), or plan extrusion (architectural), chosen by automatic object classification.
5. **Topology enforcement** — manifold checks, ≥90% quad ratio, zero n-gons, pole and edge-flow rules per `references/topology_standards.md`.
6. **Modifiers & UV prep** — bevels, smooth-by-angle, subdivision where appropriate, smart UV projection, neutral clay material.
7. **Validation** — silhouette IoU ≥ 0.92, boundary error ≤ 1.5%, Hausdorff 95th percentile, volume/bbox sanity, self-intersection scan, render-vs-mesh comparison. Every metric is computed, never fabricated.
8. **Delivery** — manifest, validation report, preview renders, and verified export files.

## Architecture — Exactly 22 Files

```
orthogenesis/
├── SKILL.md                                  # entry point & execution contract
├── references/                               # theory & standards (7)
│   ├── modeling_principles.md
│   ├── orthographic_calibration.md
│   ├── geometry_reconstruction.md
│   ├── topology_standards.md
│   ├── object_classification.md
│   ├── quality_standards.md
│   └── known_limitations.md
├── engines/                                  # executable modules (8)
│   ├── reference_analyzer.md
│   ├── alignment_engine.md
│   ├── silhouette_engine.md
│   ├── geometry_engine.md
│   ├── topology_engine.md
│   ├── modifier_engine.md
│   ├── validation_engine.md
│   └── rendering_engine.md
├── workflows/                                # execution paths (4)
│   ├── autonomous_pipeline.md
│   ├── organic_modeling.md
│   ├── hard_surface_modeling.md
│   └── architectural_modeling.md
└── standards/                                # delivery & advisory (2)
    ├── feng_shui_geometry.md                 # optional, advisory only
    └── production_delivery.md
```

## Usage

Install by copying `orthogenesis/` into your agent's managed skills directory (`~/.omp/agent/managed-skills/`). Invoke with one to three orthographic images; optionally supply a known real-world dimension for exact scale, an object-class hint, and requested export formats. The pipeline runs autonomously and pauses only when views are ambiguous or inconsistent.

---

## Bahasa Indonesia

# ORTHO GENESIS

**Tiga Pandangan. Satu Dimensi. Dari Referensi ke Kenyataan.**

ORTHO GENESIS (Orthographic Reconstruction Modeling) adalah skill otonom untuk mengubah gambar menjadi model 3D di Blender. Berikan pandangan ortografis — depan, samping, atas — dan sistem merekonstruksi model 3D siap produksi: skala terkalibrasi, siluet terekstraksi, rekonstruksi geometri sesuai kelas objek, topologi dominan-quad yang ditegakkan, modifier produksi, metrik kualitas terhitung, serta ekspor `.blend`, `.obj`, `.fbx`, `.stl`, dan `.glb`.

### Apa yang Dilakukan

1. **Analisis referensi** — mengklasifikasikan tiap gambar sebagai depan/samping/atas, mendeteksi pandangan yang hilang atau salah label.
2. **Kalibrasi** — menyelesaikan skala `units_per_pixel` bersama, menetapkan sumbu dan bidang tanah, membangun kamera referensi ortografis. Berhenti saat pandangan tidak konsisten, bukan menebak.
3. **Ekstraksi siluet** — segmentasi OpenCV (Otsu / GrabCut / alpha), ekstraksi batas, fungsi proyeksi dunia↔piksel.
4. **Rekonstruksi geometri** — lofting penampang lintang (hard-surface), carving voxel + remesh (organik), atau ekstrusi denah (arsitektur), dipilih lewat klasifikasi objek otomatis.
5. **Penegakan topologi** — pemeriksaan manifold, rasio quad ≥ 90%, tanpa n-gon, aturan pole dan edge-flow.
6. **Modifier & UV** — bevel, smooth-by-angle, subdivision seperlunya, proyeksi UV pintar, material clay netral.
7. **Validasi** — IoU siluet ≥ 0,92, galat batas ≤ 1,5%, Hausdorff persentil ke-95, kewarasan volume/bbox, deteksi self-intersection, perbandingan render-vs-mesh. Semua metrik dihitung, tidak dikarang.
8. **Pengiriman** — manifest, laporan validasi, render pratinjau, dan berkas ekspor terverifikasi.

### Arsitektur — Tepat 22 Berkas

Sama seperti di atas: 1 `SKILL.md`, 7 `references/`, 8 `engines/`, 4 `workflows/`, 2 `standards/`.

### Penggunaan

Salin `orthogenesis/` ke direktori managed skills agen Anda (`~/.omp/agent/managed-skills/`). Jalankan dengan satu hingga tiga gambar ortografis; opsional berikan dimensi dunia nyata untuk skala presisi, petunjuk kelas objek, dan format ekspor yang diminta. Pipeline berjalan otonom dan hanya berhenti saat pandangan ambigu atau tidak konsisten.

---

© 2026 PT. Agra Karya Digital — VeryCoolApps. All rights reserved.
