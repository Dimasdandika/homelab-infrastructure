# 🖥️ Homelab Infrastructure — Master Guide
**PC Bekas → Profile Site Live di Internet**
> Fresh graduate TI UGM | DevOps/Cloud Portfolio | Ubuntu 24.04 LTS

---

## 📋 Daftar Isi
1. [Persiapan Hardware & Bahan](#1-persiapan-hardware--bahan)
2. [Install Ubuntu Server 24.04](#2-install-ubuntu-server-2404)
3. [Konfigurasi Awal Server](#3-konfigurasi-awal-server)
4. [Setup Storage (HDD 1TB)](#4-setup-storage-hdd-1tb)
5. [Install Docker & Docker Compose](#5-install-docker--docker-compose)
6. [Setup DuckDNS (Domain Gratis)](#6-setup-duckdns-domain-gratis)
7. [Buat & Deploy Profile Site](#7-buat--deploy-profile-site)
8. [Setup Nginx Reverse Proxy + SSL](#8-setup-nginx-reverse-proxy--ssl)
9. [Verifikasi — Site Live!](#9-verifikasi--site-live)
10. [Struktur Repository GitHub](#10-struktur-repository-github)
11. [Next Steps: Phase 2-4](#11-next-steps-phase-2-4)
---

## 1. Persiapan Hardware & Bahan

### Spek PC Server
| Komponen | Spesifikasi |
|----------|------------|
| RAM | 8GB (2 × 4GB) |
| SSD | 256GB — OS + Docker |
| HDD | 1TB — Data, logs, backup |
| OS Target | Ubuntu Server 24.04 LTS |

### Yang Dibutuhkan Sebelum Mulai
- [ ] USB flashdisk minimal 4GB
- [ ] File ISO Ubuntu Server 24.04 LTS → https://ubuntu.com/download/server
- [ ] Software Rufus (Windows) atau Balena Etcher (Mac/Linux)
- [ ] Monitor + keyboard untuk setup awal
- [ ] Koneksi internet (LAN kabel sangat direkomendasikan)
- [ ] Akun GitHub (gratis)
- [ ] Akun DuckDNS (gratis) → https://www.duckdns.org

---

## 2. Install Ubuntu Server 24.04

### Langkah 2.1 — Burn ISO ke USB
Pakai Rufus (Windows) atau Balena Etcher, pilih ISO → pilih USB → Flash.

### Langkah 2.2 — Boot dari USB
1. Colok USB ke PC bekas
2. Nyalakan PC, masuk BIOS (F2 / F12 / Del / Esc)
3. Ubah boot order: USB first
4. Save & Exit

### Langkah 2.3 — Proses Install
