# TryHackMe — Cypheron (Nightmare v6)

Write-up de la sala **Cypheron** (Tarea 3 — Nightmare) de TryHackMe, parte del evento **2026: An AI Odyssey CTF**.

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Insane-red)
![Category](https://img.shields.io/badge/Category-AI%20Sec%20%2B%20Red%20Team-blue)
![Status](https://img.shields.io/badge/Status-Partial%20Completion-yellow)

## Resumen

| Campo | Valor |
|---|---|
| **Sala** | Cypheron — Task 3: Nightmare |
| **IP objetivo** | 10.67.142.176 |
| **Dificultad** | Insane |
| **Categoría** | AI Sec + Red Team |
| **Plataforma** | n8n (workflow automation) |
| **Vulnerabilidades** | CVE-2026-21858 (LFI), JWT Forgery, RCE vía workflow |

## Banderas

- **Flag de Usuario:** `THM{nightmare_just_begun}` ✅
- **Flag de Root:** ❌ Bloqueada por defensas activas de la plataforma

## Herramientas utilizadas

- `nmap` — reconocimiento
- `curl` — interacción con formularios y APIs
- `python3` — forja de JWT, modificación de workflows
- `jwt` (PyJWT) — manipulación de tokens
- `sqlite3` — extracción de credenciales

## Write-up completo

Ver [`writeup.md`](./writeup.md)

## Aviso

Este write-up es solo con fines educativos. Todas las pruebas se realizaron en un entorno controlado de TryHackMe con autorización explícita.

## Licencia

MIT
# thm-cypheron-nightmare-v6
