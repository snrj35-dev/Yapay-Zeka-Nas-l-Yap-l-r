# 04 — Çalışma Alanı Yönetimi

Bu rehber, AI agent'ların güvenli çalışma ortamını (workspace/sandbox), dosya operasyonlarını, context yönetimini, state kalıcılığını ve kaynak izlemeyi kapsar.

**Önceki:** [03 - Skills ve Capabilities](./03_-_Skills_ve_Capabilities.md)  
**Sonraki:** [05 - System Prompt Tasarımı](./05_-_System_Prompt_Tasarimi.md)

---

## 📌 Doküman Kartı

| Alan | Değer |
|---|---|
| Rol | Çalışma alanı yönetimi ve güvenlik referansı |
| Durum | Living specification (`v2.0`) |
| Son güncelleme | 2026-04-24 |
| Birincil okur | AI mühendisleri, güvenlik ekipleri, DevOps |
| Ana girdi | Workspace config, file operation request, context spec |
| Ana çıktı | Sandbox implementation, security policy, isolation strategy |
| Bağımlı dokümanlar | [01 - Agent Framework](./01_-_Agent_Framework_ve_Mimari.md), [02 - Tool Use](./02_-_Tool_Use_ve_Entegrasyon.md) |

> **Kalite notu:** Kod blokları referans implementasyon örnekleridir. Production'a almadan önce güvenlik denetiminden geçirin.

---

## İçindekiler

1. [Çalışma Alanı Nedir?](#1-çalışma-alanı-nedir)
2. [Sandbox Environment](#2-sandbox-environment)
3. [File Operations](#3-file-operations)
4. [Context Management](#4-context-management)
5. [State Persistence ve Checkpoint](#5-state-persistence-ve-checkpoint)
6. [İzolasyon ve Güvenlik](#6-izolasyon-ve-güvenlik)
7. [Resource Management](#7-resource-management)
8. [Gözlemlenebilirlik (Audit & Logging)](#8-gözlemlenebilirlik-audit--logging)
9. [Türkçe Çalışma Alanı İçin Pratik Notlar](#9-türkçe-çalışma-alanı-için-pratik-notlar)
10. [Ekler](#ekler)

---

## 1. Çalışma Alanı Nedir?

### Tanım

Çalışma alanı (workspace), AI agent'ın dosyaları, bağlamı (context) ve durumu (state) yönettiği izole ortamdır. Dış dünya ile etkileşimi kontrol altında tutar, güvenliği sağlar ve kaynakların kontrollü kullanılmasını garanti eder.

### Katmanlı Mimari

```
┌─────────────────────────────────────────────┐
│              Agent Layer                    │
│         (skills, tools, reasoning)          │
├─────────────────────────────────────────────┤
│           Workspace Layer                   │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │   File   │ │ Context  │ │    State    │ │
│  │  System  │ │  Store   │ │  Persister  │ │
│  └──────────┘ └──────────┘ └─────────────┘ │
├─────────────────────────────────────────────┤
│            Security Layer                   │
│  Permission Manager | Audit Logger | Limits │
├─────────────────────────────────────────────┤
│            Sandbox Layer                    │
│    Docker / venv / Process Isolation        │
└─────────────────────────────────────────────┘
```

### Bileşenler

| Bileşen | Sorumluluğu | Kritiklik |
|---------|-------------|-----------|
| **Sandbox** | İzole çalışma ortamı | Yüksek — escape riski |
| **File System** | Güvenli dosya I/O | Yüksek — path traversal riski |
| **Context Store** | Kısa/uzun vadeli bağlam | Orta |
| **State Manager** | Agent durum kalıcılığı | Orta |
| **Permission Manager** | Yetki kontrolü | Yüksek |
| **Resource Monitor** | CPU/RAM/Disk limitleri | Orta |
| **Audit Logger** | Tüm işlemlerin kaydı | Yüksek — uyumluluk |

---

## 2. Sandbox Environment

### 2.1 Docker Tabanlı Sandbox

Üretim ortamı için önerilen yaklaşım. En güçlü izolasyonu sağlar.

```python
import docker
import tarfile
import io
import os
from typing import Dict, Any, Optional

class DockerSandbox:
    """
    Docker konteyner tabanlı sandbox.
    Her agent görevi için izole bir konteyner başlatır.
    """

    DEFAULT_IMAGE = "python:3.11-slim"

    def __init__(
        self,
        image:            str = DEFAULT_IMAGE,
        mem_limit:        str = "512m",
        cpu_quota:        int = 50000,     # %50 CPU
        network_disabled: bool = True,
        readonly_root:    bool = False
    ):
        self.client           = docker.from_env()
        self.image            = image
        self.mem_limit        = mem_limit
        self.cpu_quota        = cpu_quota
        self.network_disabled = network_disabled
        self.readonly_root    = readonly_root
        self._container       = None

    def __enter__(self):
        self.start()
        return self

    def __exit__(self, *_):
        self.stop()

    def start(self, workspace_path: Optional[str] = None):
        volumes = {}
        if workspace_path:
            volumes[os.path.abspath(workspace_path)] = {
                "bind": "/workspace",
                "mode": "rw"
            }

        self._container = self.client.containers.run(
            self.image,
            command          = "tail -f /dev/null",
            volumes          = volumes,
            detach           = True,
            mem_limit        = self.mem_limit,
            cpu_quota        = self.cpu_quota,
            network_disabled = self.network_disabled,
            read_only        = self.readonly_root,
            security_opt     = ["no-new-privileges"],
            user             = "nobody"    # root olarak çalışma
        )
        return self

    def execute(
        self,
        command:     str,
        timeout:     int = 30,
        workdir:     str = "/workspace",
        environment: Dict[str, str] = None
    ) -> Dict[str, Any]:
        """Konteynerde komut çalıştır."""
        if not self._container:
            raise RuntimeError("Sandbox başlatılmadı.")

        try:
            result = self._container.exec_run(
                cmd         = ["sh", "-c", command],
                workdir     = workdir,
                environment = environment or {},
                timeout     = timeout,
                demux       = True   # stdout/stderr ayrı
            )
            stdout = result.output[0].decode("utf-8", errors="replace") if result.output[0] else ""
            stderr = result.output[1].decode("utf-8", errors="replace") if result.output[1] else ""

            return {
                "exit_code": result.exit_code,
                "stdout":    stdout,
                "stderr":    stderr,
                "success":   result.exit_code == 0
            }
        except Exception as e:
            return {"exit_code": -1, "stdout": "", "stderr": str(e), "success": False}

    def copy_file(self, local_path: str, container_path: str):
        """Yerel dosyayı konteynere kopyala."""
        archive = io.BytesIO()
        with tarfile.open(fileobj=archive, mode="w") as tar:
            tar.add(local_path, arcname=os.path.basename(container_path))
        archive.seek(0)
        self._container.put_archive(
            path=os.path.dirname(container_path),
            data=archive.read()
        )

    def read_file(self, container_path: str) -> bytes:
        """Konteynerden dosya oku."""
        bits, _ = self._container.get_archive(container_path)
        archive = io.BytesIO(b"".join(bits))
        with tarfile.open(fileobj=archive) as tar:
            member = tar.getmembers()[0]
            return tar.extractfile(member).read()

    def stop(self):
        if self._container:
            try:
                self._container.stop(timeout=5)
                self._container.remove(force=True)
            except Exception:
                pass
            self._container = None

    @property
    def is_running(self) -> bool:
        if not self._container:
            return False
        self._container.reload()
        return self._container.status == "running"

# Kullanım
with DockerSandbox(mem_limit="256m", network_disabled=True) as sandbox:
    result = sandbox.execute("python3 -c 'print(2 + 2)'", timeout=10)
    print(result["stdout"])   # 4
```

### 2.2 Process Tabanlı Sandbox (Geliştirme / CI)

```python
import subprocess
import tempfile
import shutil
import signal
import os

class ProcessSandbox:
    """
    subprocess tabanlı hafif sandbox. Docker olmayan ortamlar için.
    Docker'a kıyasla daha düşük izolasyon — üretimde dikkatli kullanın.
    """

    def __init__(self, timeout_default: int = 30):
        self.timeout_default = timeout_default
        self._temp_dir       = tempfile.mkdtemp(prefix="agent_sandbox_")

    def __enter__(self):
        return self

    def __exit__(self, *_):
        self.cleanup()

    def execute(
        self,
        command:     str,
        timeout:     int = None,
        env:         Dict[str, str] = None
    ) -> Dict[str, Any]:
        timeout = timeout or self.timeout_default

        # Çevre değişkenlerini kısıtla
        safe_env = {
            "PATH":    "/usr/local/bin:/usr/bin:/bin",
            "HOME":    self._temp_dir,
            "TMPDIR":  self._temp_dir,
            **(env or {})
        }
        # Tehlikeli değişkenleri temizle
        for key in ("LD_PRELOAD", "LD_LIBRARY_PATH", "PYTHONPATH"):
            safe_env.pop(key, None)

        try:
            proc = subprocess.run(
                command,
                shell          = True,
                cwd            = self._temp_dir,
                timeout        = timeout,
                capture_output = True,
                text           = True,
                env            = safe_env,
                preexec_fn     = self._set_resource_limits   # Unix only
            )
            return {
                "exit_code": proc.returncode,
                "stdout":    proc.stdout,
                "stderr":    proc.stderr,
                "success":   proc.returncode == 0
            }
        except subprocess.TimeoutExpired:
            return {"exit_code": -1, "stdout": "", "stderr": "Timeout", "success": False}
        except Exception as e:
            return {"exit_code": -1, "stdout": "", "stderr": str(e), "success": False}

    @staticmethod
    def _set_resource_limits():
        """Unix: işlem kaynak limitlerini ayarla (Docker olmadan)."""
        try:
            import resource
            # Maksimum 256 MB RAM
            resource.setrlimit(resource.RLIMIT_AS, (256 * 1024 * 1024, 256 * 1024 * 1024))
            # Maksimum 100 MB dosya boyutu
            resource.setrlimit(resource.RLIMIT_FSIZE, (100 * 1024 * 1024, 100 * 1024 * 1024))
            # Maksimum 50 MB core dump (pratikte 0)
            resource.setrlimit(resource.RLIMIT_CORE, (0, 0))
        except Exception:
            pass   # Windows'ta resource modülü yoktur

    @property
    def workspace(self) -> str:
        return self._temp_dir

    def cleanup(self):
        shutil.rmtree(self._temp_dir, ignore_errors=True)
```

### 2.3 Sanal Ortam (venv) Sandbox

```python
import venv
import sys
import subprocess

class VenvSandbox:
    """
    Python sanal ortam tabanlı sandbox.
    Bağımlılık izolasyonu sağlar, sistem paketlerine erişimi engeller.
    """

    def __init__(self, base_path: str):
        self.base_path  = pathlib.Path(base_path)
        self.venv_path  = self.base_path / "venv"
        self._python    = None

    def create(self, packages: List[str] = None):
        """Sanal ortamı oluştur ve gerekli paketleri yükle."""
        venv.create(str(self.venv_path), with_pip=True, clear=True)
        self._python = (
            self.venv_path / "Scripts" / "python.exe"
            if sys.platform == "win32"
            else self.venv_path / "bin" / "python"
        )
        if packages:
            self._pip_install(packages)

    def _pip_install(self, packages: List[str]):
        pip = self.venv_path / ("Scripts/pip.exe" if sys.platform == "win32" else "bin/pip")
        subprocess.run([str(pip), "install", "--quiet", *packages], check=True)

    def execute_script(self, code: str, timeout: int = 30) -> Dict[str, Any]:
        """Python kodunu sanal ortamda çalıştır."""
        if not self._python or not self._python.exists():
            raise RuntimeError("Sanal ortam oluşturulmadı. Önce create() çağrın.")

        script_path = self.base_path / "_temp_script.py"
        script_path.write_text(code, encoding="utf-8")

        try:
            result = subprocess.run(
                [str(self._python), str(script_path)],
                capture_output = True,
                text           = True,
                timeout        = timeout
            )
            return {
                "exit_code": result.returncode,
                "stdout":    result.stdout,
                "stderr":    result.stderr,
                "success":   result.returncode == 0
            }
        finally:
            script_path.unlink(missing_ok=True)
```

### 2.4 Sandbox Karşılaştırma

| Özellik | Docker | Process | venv |
|---------|--------|---------|------|
| **Ağ izolasyonu** | ✅ Tam | ❌ Yok | ❌ Yok |
| **Dosya izolasyonu** | ✅ Tam | ⚠️ Kısmi | ⚠️ Kısmi |
| **Kurulum karmaşıklığı** | Yüksek | Düşük | Düşük |
| **Başlatma süresi** | 2–5 sn | <0.1 sn | 1–3 sn |
| **Kaynak limitleri** | ✅ cgroups | ⚠️ ulimit | ❌ Yok |
| **Üretim uygunluğu** | ✅ Evet | ❌ Hayır | ⚠️ Sınırlı |

---

## 3. File Operations

### 3.1 Güvenli Dosya Yöneticisi

```python
import pathlib
import hashlib
import shutil
from datetime import datetime
from typing import List, Dict, Any, Optional

class WorkspaceFileManager:
    """
    Workspace sınırları içinde güvenli dosya işlemleri.
    Path traversal, boyut limiti ve encoding koruması içerir.
    """

    MAX_FILE_SIZE_MB   = 50
    MAX_FILES_PER_DIR  = 1000
    BLOCKED_EXTENSIONS = {".exe", ".dll", ".so", ".dylib", ".sh", ".bat", ".cmd"}

    def __init__(self, base_path: str, enable_versioning: bool = True):
        self.base       = pathlib.Path(base_path).resolve()
        self.base.mkdir(parents=True, exist_ok=True)
        self.versioning = enable_versioning
        self._versions  = self.base / ".versions"
        if enable_versioning:
            self._versions.mkdir(exist_ok=True)

    # ── Read ──────────────────────────────────────────────────────────────

    def read(self, path: str, encoding: str = "utf-8") -> Dict[str, Any]:
        full = self._resolve(path)
        self._check_size(full)

        content = full.read_text(encoding=encoding)
        stat    = full.stat()
        return {
            "path":        path,
            "content":     content,
            "size_bytes":  stat.st_size,
            "lines":       content.count("\n") + 1,
            "encoding":    encoding,
            "modified_at": datetime.fromtimestamp(stat.st_mtime).isoformat()
        }

    def read_bytes(self, path: str) -> bytes:
        return self._resolve(path).read_bytes()

    # ── Write ─────────────────────────────────────────────────────────────

    def write(
        self,
        path:     str,
        content:  str,
        encoding: str = "utf-8",
        mode:     str = "overwrite"   # overwrite | append
    ) -> Dict[str, Any]:
        self._check_extension(path)
        full = self._resolve_new(path)

        if self.versioning and full.exists():
            self._backup(full)

        if mode == "append":
            with open(full, "a", encoding=encoding) as f:
                f.write(content)
        else:
            full.write_text(content, encoding=encoding)

        return {
            "path":    path,
            "bytes":   len(content.encode(encoding)),
            "sha256":  hashlib.sha256(content.encode(encoding)).hexdigest()[:12],
            "mode":    mode
        }

    def write_bytes(self, path: str, data: bytes) -> Dict[str, Any]:
        self._check_extension(path)
        full = self._resolve_new(path)
        if self.versioning and full.exists():
            self._backup(full)
        full.write_bytes(data)
        return {"path": path, "bytes": len(data)}

    # ── List / Search ─────────────────────────────────────────────────────

    def list(self, path: str = ".", recursive: bool = False) -> List[Dict[str, Any]]:
        full = self._resolve(path, must_exist=True)
        if not full.is_dir():
            raise ValueError(f"Klasör değil: {path}")

        iterator = full.rglob("*") if recursive else full.iterdir()
        entries  = []
        for item in iterator:
            if item.name.startswith("."):
                continue
            stat = item.stat()
            entries.append({
                "name":        item.name,
                "relative":    str(item.relative_to(self.base)),
                "type":        "file" if item.is_file() else "directory",
                "size_bytes":  stat.st_size if item.is_file() else 0,
                "modified_at": datetime.fromtimestamp(stat.st_mtime).isoformat()
            })
        return entries

    def search(self, pattern: str, path: str = ".") -> List[str]:
        """Glob pattern ile dosya ara. Örn: '**/*.py'"""
        base = self._resolve(path, must_exist=True)
        return [
            str(p.relative_to(self.base))
            for p in base.glob(pattern)
            if not p.name.startswith(".")
        ]

    # ── Delete ────────────────────────────────────────────────────────────

    def delete(self, path: str, confirm: bool = False) -> Dict[str, Any]:
        if not confirm:
            raise ValueError("Silme işlemi için confirm=True gereklidir.")
        full = self._resolve(path, must_exist=True)

        if self.versioning and full.is_file():
            self._backup(full)

        if full.is_file():
            full.unlink()
        else:
            shutil.rmtree(full)
        return {"deleted": path}

    # ── Versioning ────────────────────────────────────────────────────────

    def _backup(self, full_path: pathlib.Path):
        ts    = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
        bname = f"{full_path.name}.{ts}.bak"
        dest  = self._versions / bname
        shutil.copy2(full_path, dest)

    def list_versions(self, filename: str) -> List[str]:
        return sorted(
            p.name for p in self._versions.glob(f"{filename}.*.bak")
        )

    def restore_version(self, filename: str, version_name: str) -> Dict[str, Any]:
        src  = self._versions / version_name
        if not src.exists():
            raise FileNotFoundError(f"Versiyon bulunamadı: {version_name}")
        dest = self.base / filename
        shutil.copy2(src, dest)
        return {"restored": filename, "from": version_name}

    # ── Helpers ───────────────────────────────────────────────────────────

    def _resolve(self, path: str, must_exist: bool = True) -> pathlib.Path:
        full = (self.base / path).resolve()
        if not str(full).startswith(str(self.base)):
            raise PermissionError(f"Workspace dışı erişim engellendi: {path}")
        if must_exist and not full.exists():
            raise FileNotFoundError(f"Bulunamadı: {path}")
        return full

    def _resolve_new(self, path: str) -> pathlib.Path:
        full = (self.base / path).resolve()
        if not str(full).startswith(str(self.base)):
            raise PermissionError(f"Workspace dışı yazma engellendi: {path}")
        full.parent.mkdir(parents=True, exist_ok=True)
        return full

    def _check_size(self, full_path: pathlib.Path):
        size_mb = full_path.stat().st_size / (1024 * 1024)
        if size_mb > self.MAX_FILE_SIZE_MB:
            raise ValueError(f"Dosya çok büyük: {size_mb:.1f} MB (limit: {self.MAX_FILE_SIZE_MB} MB)")

    def _check_extension(self, path: str):
        ext = pathlib.Path(path).suffix.lower()
        if ext in self.BLOCKED_EXTENSIONS:
            raise PermissionError(f"Bu uzantıya dosya yazmak yasaktır: {ext}")

    @property
    def usage(self) -> Dict[str, Any]:
        total_bytes = sum(
            f.stat().st_size for f in self.base.rglob("*") if f.is_file()
        )
        return {
            "total_mb": round(total_bytes / (1024 * 1024), 2),
            "files":    sum(1 for _ in self.base.rglob("*") if _.is_file())
        }
```

---

## 4. Context Management

### 4.1 TTL Destekli Context Store

```python
import threading
import time
from dataclasses import dataclass, field
from typing import Any, Optional, Dict

@dataclass
class ContextEntry:
    key:        str
    value:      Any
    created_at: float = field(default_factory=time.time)
    ttl:        Optional[float] = None   # Saniye cinsinden; None = kalıcı
    tags:       List[str] = field(default_factory=list)

    @property
    def is_expired(self) -> bool:
        if self.ttl is None:
            return False
        return (time.time() - self.created_at) > self.ttl

class ContextStore:
    """
    TTL destekli, thread-safe in-memory context deposu.
    Arka planda otomatik temizleme yapar.
    """

    def __init__(self, cleanup_interval: int = 60):
        self._store: Dict[str, ContextEntry] = {}
        self._lock  = threading.RLock()
        self._start_cleanup(cleanup_interval)

    def set(
        self,
        key:   str,
        value: Any,
        ttl:   Optional[float] = None,
        tags:  List[str] = None
    ):
        with self._lock:
            self._store[key] = ContextEntry(key=key, value=value, ttl=ttl, tags=tags or [])

    def get(self, key: str, default: Any = None) -> Any:
        with self._lock:
            entry = self._store.get(key)
            if entry is None:
                return default
            if entry.is_expired:
                del self._store[key]
                return default
            return entry.value

    def delete(self, key: str):
        with self._lock:
            self._store.pop(key, None)

    def get_by_tag(self, tag: str) -> Dict[str, Any]:
        with self._lock:
            return {
                k: e.value
                for k, e in self._store.items()
                if tag in e.tags and not e.is_expired
            }

    def keys(self, prefix: str = "") -> List[str]:
        with self._lock:
            return [
                k for k, e in self._store.items()
                if k.startswith(prefix) and not e.is_expired
            ]

    def clear(self, tag: str = None):
        with self._lock:
            if tag:
                to_delete = [k for k, e in self._store.items() if tag in e.tags]
                for k in to_delete:
                    del self._store[k]
            else:
                self._store.clear()

    def _start_cleanup(self, interval: int):
        def _cleanup():
            while True:
                time.sleep(interval)
                with self._lock:
                    expired = [k for k, e in self._store.items() if e.is_expired]
                    for k in expired:
                        del self._store[k]
        t = threading.Thread(target=_cleanup, daemon=True)
        t.start()

    @property
    def stats(self) -> Dict[str, int]:
        with self._lock:
            active  = sum(1 for e in self._store.values() if not e.is_expired)
            expired = len(self._store) - active
            return {"active": active, "expired": expired, "total": len(self._store)}
```

### 4.2 Context Window Yönetimi

```python
import tiktoken

class ContextWindowManager:
    """
    LLM context penceresinin dolmamasını sağlar.
    Önem skoruna göre eski bağlamı budayabilir.
    """

    def __init__(self, max_tokens: int = 8000, model: str = "gpt-4"):
        self.max_tokens = max_tokens
        self.encoding   = tiktoken.encoding_for_model(model)
        self._segments: List[Dict] = []   # {content, tokens, priority, timestamp}

    def add(self, content: str, priority: int = 5, role: str = "user"):
        tokens = self._count(content)
        self._segments.append({
            "content":   content,
            "tokens":    tokens,
            "priority":  priority,    # 1 (düşük) – 10 (yüksek)
            "role":      role,
            "timestamp": time.time()
        })
        self._trim_if_needed()

    def get_context(self) -> str:
        return "\n\n".join(s["content"] for s in self._segments)

    def get_messages(self) -> List[Dict[str, str]]:
        return [{"role": s["role"], "content": s["content"]} for s in self._segments]

    def total_tokens(self) -> int:
        return sum(s["tokens"] for s in self._segments)

    def _trim_if_needed(self):
        """Limit aşıldığında önce düşük öncelikli, sonra eski segmentleri kaldır."""
        if self.total_tokens() <= self.max_tokens:
            return

        # Önce önceliğe, sonra zamana göre sırala (en düşük önce silinir)
        self._segments.sort(key=lambda s: (s["priority"], s["timestamp"]))

        while self.total_tokens() > self.max_tokens and len(self._segments) > 1:
            removed = self._segments.pop(0)
            import logging
            logging.debug(
                f"Context trim: '{removed['content'][:40]}...' "
                f"({removed['tokens']} token, öncelik {removed['priority']})"
            )

    def _count(self, text: str) -> int:
        return len(self.encoding.encode(text))
```

---

## 5. State Persistence ve Checkpoint

### 5.1 Checkpoint Manager

```python
import json
import os
from datetime import datetime
from typing import Dict, Any, Optional, List

class CheckpointManager:
    """
    Agent durumunu diske kaydeder ve geri yükler.
    Uzun görevlerde yeniden başlatma desteği sağlar.
    """

    CHECKPOINT_EXT = ".ckpt.json"

    def __init__(self, checkpoint_dir: str, max_checkpoints: int = 10):
        self.dir            = pathlib.Path(checkpoint_dir)
        self.dir.mkdir(parents=True, exist_ok=True)
        self.max_checkpoints = max_checkpoints

    def save(self, agent_id: str, state: Dict[str, Any]) -> str:
        ts             = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
        checkpoint_id  = f"{agent_id}_{ts}"
        path           = self.dir / f"{checkpoint_id}{self.CHECKPOINT_EXT}"

        payload = {
            "checkpoint_id": checkpoint_id,
            "agent_id":      agent_id,
            "created_at":    datetime.now().isoformat(),
            "state":         state
        }
        path.write_text(json.dumps(payload, ensure_ascii=False, indent=2, default=str))

        # Eski checkpoint'leri temizle
        self._prune(agent_id)
        return checkpoint_id

    def load(self, checkpoint_id: str) -> Dict[str, Any]:
        path = self.dir / f"{checkpoint_id}{self.CHECKPOINT_EXT}"
        if not path.exists():
            raise FileNotFoundError(f"Checkpoint bulunamadı: {checkpoint_id}")
        payload = json.loads(path.read_text())
        return payload["state"]

    def load_latest(self, agent_id: str) -> Optional[Dict[str, Any]]:
        checkpoints = self.list(agent_id)
        if not checkpoints:
            return None
        return self.load(checkpoints[-1]["checkpoint_id"])

    def list(self, agent_id: str) -> List[Dict[str, Any]]:
        results = []
        for p in sorted(self.dir.glob(f"{agent_id}_*{self.CHECKPOINT_EXT}")):
            try:
                payload = json.loads(p.read_text())
                results.append({
                    "checkpoint_id": payload["checkpoint_id"],
                    "created_at":    payload["created_at"]
                })
            except Exception:
                continue
        return results

    def delete(self, checkpoint_id: str):
        (self.dir / f"{checkpoint_id}{self.CHECKPOINT_EXT}").unlink(missing_ok=True)

    def _prune(self, agent_id: str):
        """En eski checkpoint'leri sil."""
        checkpoints = self.list(agent_id)
        while len(checkpoints) > self.max_checkpoints:
            oldest = checkpoints.pop(0)
            self.delete(oldest["checkpoint_id"])
```

### 5.2 SQLite Tabanlı Kalıcı State

```python
import sqlite3
import json

class PersistentStateStore:
    """
    Birden fazla agent'ın durumunu SQLite'ta saklar.
    Thread-safe, sorgulanabilir.
    """

    def __init__(self, db_path: str):
        self.db_path = db_path
        self._init()

    def _init(self):
        with self._conn() as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS agent_state (
                    agent_id    TEXT NOT NULL,
                    key         TEXT NOT NULL,
                    value       TEXT NOT NULL,
                    updated_at  TEXT NOT NULL,
                    PRIMARY KEY (agent_id, key)
                )
            """)
            conn.execute("CREATE INDEX IF NOT EXISTS idx_agent ON agent_state(agent_id)")

    def set(self, agent_id: str, key: str, value: Any):
        with self._conn() as conn:
            conn.execute("""
                INSERT INTO agent_state (agent_id, key, value, updated_at)
                VALUES (?, ?, ?, ?)
                ON CONFLICT(agent_id, key) DO UPDATE SET value=excluded.value, updated_at=excluded.updated_at
            """, (agent_id, key, json.dumps(value, ensure_ascii=False), datetime.now().isoformat()))

    def get(self, agent_id: str, key: str, default: Any = None) -> Any:
        with self._conn() as conn:
            row = conn.execute(
                "SELECT value FROM agent_state WHERE agent_id=? AND key=?",
                (agent_id, key)
            ).fetchone()
        return json.loads(row[0]) if row else default

    def get_all(self, agent_id: str) -> Dict[str, Any]:
        with self._conn() as conn:
            rows = conn.execute(
                "SELECT key, value FROM agent_state WHERE agent_id=?", (agent_id,)
            ).fetchall()
        return {r[0]: json.loads(r[1]) for r in rows}

    def delete(self, agent_id: str, key: str = None):
        with self._conn() as conn:
            if key:
                conn.execute("DELETE FROM agent_state WHERE agent_id=? AND key=?", (agent_id, key))
            else:
                conn.execute("DELETE FROM agent_state WHERE agent_id=?", (agent_id,))

    def _conn(self) -> sqlite3.Connection:
        return sqlite3.connect(self.db_path, check_same_thread=False)
```

---

## 6. İzolasyon ve Güvenlik

### 6.1 Permission Manager

```python
from enum import Enum, auto
from typing import Set

class Permission(Enum):
    FILE_READ    = auto()
    FILE_WRITE   = auto()
    FILE_DELETE  = auto()
    DB_READ      = auto()
    DB_WRITE     = auto()
    NETWORK      = auto()
    CODE_EXECUTE = auto()
    SYSTEM       = auto()

# Önceden tanımlı roller
ROLE_PRESETS = {
    "readonly":   {Permission.FILE_READ, Permission.DB_READ},
    "analyst":    {Permission.FILE_READ, Permission.DB_READ, Permission.FILE_WRITE},
    "developer":  {Permission.FILE_READ, Permission.FILE_WRITE, Permission.FILE_DELETE,
                   Permission.DB_READ, Permission.DB_WRITE, Permission.CODE_EXECUTE},
    "admin":      set(Permission)
}

class PermissionManager:
    """
    Agent bazında ince taneli yetki yönetimi.
    Rol tabanlı atama ve bireysel izin desteği içerir.
    """

    def __init__(self):
        self._perms: Dict[str, Set[Permission]] = {}

    def assign_role(self, agent_id: str, role: str):
        preset = ROLE_PRESETS.get(role)
        if not preset:
            raise ValueError(f"Tanımsız rol: {role}. Geçerli roller: {list(ROLE_PRESETS.keys())}")
        self._perms[agent_id] = set(preset)

    def grant(self, agent_id: str, *permissions: Permission):
        self._perms.setdefault(agent_id, set()).update(permissions)

    def revoke(self, agent_id: str, *permissions: Permission):
        if agent_id in self._perms:
            self._perms[agent_id].difference_update(permissions)

    def check(self, agent_id: str, permission: Permission) -> bool:
        return permission in self._perms.get(agent_id, set())

    def require(self, agent_id: str, permission: Permission):
        """İzin yoksa PermissionError fırlat."""
        if not self.check(agent_id, permission):
            raise PermissionError(
                f"Agent '{agent_id}' için '{permission.name}' yetkisi yok."
            )

    def list_permissions(self, agent_id: str) -> List[str]:
        return [p.name for p in self._perms.get(agent_id, set())]

    def clear(self, agent_id: str):
        self._perms.pop(agent_id, None)
```

### 6.2 Güvenlik Politikası Dekoratörü

```python
from functools import wraps

def requires_permission(permission: Permission, manager: PermissionManager):
    """
    Fonksiyon dekoratörü: çağrı öncesi yetki kontrolü yapar.
    İlk argümanın `agent_id` taşıdığını varsayar.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(agent_id: str, *args, **kwargs):
            manager.require(agent_id, permission)
            return func(agent_id, *args, **kwargs)
        return wrapper
    return decorator

# Kullanım
perm_mgr = PermissionManager()

@requires_permission(Permission.FILE_WRITE, perm_mgr)
def write_report(agent_id: str, path: str, content: str):
    ...
```

---

## 7. Resource Management

### 7.1 Kaynak İzleyici

```python
import psutil
import threading
from dataclasses import dataclass

@dataclass
class ResourceLimits:
    max_memory_mb:  float = 512.0
    max_cpu_pct:    float = 80.0
    max_disk_mb:    float = 10 * 1024   # 10 GB
    max_open_files: int   = 100

class ResourceMonitor:
    """
    Process kaynaklarını izler ve limit aşıldığında uyarı verir.
    Arka planda periyodik kontrol yapar.
    """

    def __init__(self, limits: ResourceLimits, check_interval: int = 5):
        self.limits    = limits
        self.process   = psutil.Process()
        self._alerts:  List[str] = []
        self._lock     = threading.Lock()
        self._start_monitoring(check_interval)

    def check_now(self) -> Dict[str, Any]:
        usage    = self._get_usage()
        alerts   = []

        if usage["memory_mb"] > self.limits.max_memory_mb:
            alerts.append(f"RAM limiti aşıldı: {usage['memory_mb']:.0f} MB > {self.limits.max_memory_mb} MB")
        if usage["cpu_pct"] > self.limits.max_cpu_pct:
            alerts.append(f"CPU limiti aşıldı: {usage['cpu_pct']:.1f}% > {self.limits.max_cpu_pct}%")
        if usage["open_files"] > self.limits.max_open_files:
            alerts.append(f"Açık dosya limiti: {usage['open_files']} > {self.limits.max_open_files}")

        with self._lock:
            self._alerts.extend(alerts)

        return {"usage": usage, "alerts": alerts, "ok": len(alerts) == 0}

    def _get_usage(self) -> Dict[str, float]:
        try:
            mem  = self.process.memory_info()
            cpu  = self.process.cpu_percent(interval=0.1)
            fds  = len(self.process.open_files())
            disk = psutil.disk_usage("/")
            return {
                "memory_mb":  mem.rss / (1024 * 1024),
                "cpu_pct":    cpu,
                "open_files": fds,
                "disk_free_mb": disk.free / (1024 * 1024)
            }
        except psutil.NoSuchProcess:
            return {}

    def get_alerts(self, clear: bool = True) -> List[str]:
        with self._lock:
            alerts = list(self._alerts)
            if clear:
                self._alerts.clear()
        return alerts

    def _start_monitoring(self, interval: int):
        def _loop():
            while True:
                self.check_now()
                time.sleep(interval)
        threading.Thread(target=_loop, daemon=True).start()
```

### 7.2 Disk Temizleme

```python
class DiskCleaner:
    """
    Workspace'deki eski ve büyük dosyaları yönetir.
    """

    def __init__(self, workspace: WorkspaceFileManager, max_size_mb: float = 5000):
        self.workspace   = workspace
        self.max_size_mb = max_size_mb

    def cleanup_old(self, older_than_days: int = 7) -> Dict[str, Any]:
        """Belirli günden eski dosyaları sil."""
        cutoff = time.time() - (older_than_days * 86400)
        deleted, freed = 0, 0

        for entry in self.workspace.list(recursive=True):
            if entry["type"] != "file":
                continue
            full = self.workspace.base / entry["relative"]
            if full.stat().st_mtime < cutoff:
                size = full.stat().st_size
                full.unlink()
                deleted += 1
                freed   += size

        return {
            "deleted_files":  deleted,
            "freed_mb":       round(freed / (1024 * 1024), 2)
        }

    def enforce_limit(self) -> Dict[str, Any]:
        """Toplam boyut limiti aşıldığında en eski dosyaları sil."""
        deleted, freed = 0, 0

        while self.workspace.usage["total_mb"] > self.max_size_mb:
            # En eski dosyayı bul
            all_files = [
                e for e in self.workspace.list(recursive=True)
                if e["type"] == "file"
            ]
            if not all_files:
                break
            oldest = min(all_files, key=lambda e: e["modified_at"])
            full   = self.workspace.base / oldest["relative"]
            freed += full.stat().st_size
            full.unlink()
            deleted += 1

        return {"deleted_files": deleted, "freed_mb": round(freed / (1024 * 1024), 2)}
```

---

## 8. Gözlemlenebilirlik (Audit & Logging)

### 8.1 Audit Logger

```python
import json
from datetime import datetime, timezone

class AuditLogger:
    """
    Tüm workspace operasyonlarını değiştirilemez log kaydına yazar.
    Uyumluluk (compliance) ve forensics için kritik.
    """

    def __init__(self, log_path: str, rotate_mb: float = 100):
        self.log_path  = pathlib.Path(log_path)
        self.rotate_mb = rotate_mb
        self._lock     = threading.Lock()
        self.log_path.parent.mkdir(parents=True, exist_ok=True)

    def log(
        self,
        agent_id:  str,
        action:    str,
        resource:  str,
        outcome:   str,            # "success" | "failure" | "denied"
        details:   Dict = None,
        user_id:   str = None
    ):
        entry = {
            "ts":        datetime.now(timezone.utc).isoformat(),
            "agent_id":  agent_id,
            "action":    action,
            "resource":  resource,
            "outcome":   outcome,
            "details":   details or {},
            "user_id":   user_id
        }
        with self._lock:
            self._rotate_if_needed()
            with open(self.log_path, "a", encoding="utf-8") as f:
                f.write(json.dumps(entry, ensure_ascii=False) + "\n")

    def query(
        self,
        agent_id:  str = None,
        action:    str = None,
        outcome:   str = None,
        limit:     int = 100,
        since:     str = None    # ISO timestamp
    ) -> List[dict]:
        results = []
        with self._lock:
            if not self.log_path.exists():
                return []
            with open(self.log_path, encoding="utf-8") as f:
                for line in f:
                    try:
                        e = json.loads(line)
                    except json.JSONDecodeError:
                        continue
                    if agent_id and e.get("agent_id") != agent_id:
                        continue
                    if action and e.get("action") != action:
                        continue
                    if outcome and e.get("outcome") != outcome:
                        continue
                    if since and e.get("ts", "") < since:
                        continue
                    results.append(e)
        return results[-limit:]

    def summary(self, agent_id: str = None) -> Dict[str, Any]:
        entries  = self.query(agent_id=agent_id, limit=10000)
        by_action  = {}
        by_outcome = {}
        for e in entries:
            by_action[e["action"]]   = by_action.get(e["action"], 0) + 1
            by_outcome[e["outcome"]] = by_outcome.get(e["outcome"], 0) + 1
        return {
            "total":      len(entries),
            "by_action":  by_action,
            "by_outcome": by_outcome
        }

    def _rotate_if_needed(self):
        if not self.log_path.exists():
            return
        size_mb = self.log_path.stat().st_size / (1024 * 1024)
        if size_mb >= self.rotate_mb:
            ts       = datetime.now().strftime("%Y%m%d_%H%M%S")
            rotated  = self.log_path.with_suffix(f".{ts}.log")
            self.log_path.rename(rotated)
```

---

## 9. Türkçe Çalışma Alanı İçin Pratik Notlar

> **Dosya Yolları:** Türkçe karakter içeren yol adları bazı eski kütüphanelerde sorun çıkarabilir. `pathlib.Path` kullanırsanız Python bunu büyük ölçüde yönetir; yine de klasör ve dosya adlarında boşluk ve özel karakter kullanmaktan kaçının.

> **Encoding:** Tüm metin dosyalarını `utf-8` ile okuyup yazın. `write_text(..., encoding="utf-8")` ve `read_text(..., encoding="utf-8")` kullanın. JSON için `json.dumps(..., ensure_ascii=False)`.

> **Audit Logları:** Kullanıcıya dönen hata ve uyarı mesajlarını Türkçe yazın; ham audit log satırlarını İngilizce (JSON key'leri) tutabilirsiniz — bu SIEM araçlarıyla uyumu kolaylaştırır.

> **Saat Dilimi:** Log zaman damgalarını `datetime.now(timezone.utc)` ile UTC olarak saklayın; kullanıcıya göstermek için `Europe/Istanbul` (UTC+3) dönüştürün.

> **Dizin İsimlendirme:** Workspace altındaki klasörleri sabitleyin (`data/`, `reports/`, `temp/`); Türkçe karakter içeren dinamik isimleri URL-encode ederek kullanın.

---

## Ekler

### A — Terimler Sözlüğü

| Terim | Açıklama |
|---|---|
| **Workspace** | Agent'ın izole çalışma ortamı |
| **Sandbox** | Güvenlik sınırları çizilmiş, kontrollü çalışma ortamı |
| **Path Traversal** | `../` gibi yol geçiş saldırısı |
| **Context Store** | TTL destekli in-memory bağlam deposu |
| **Checkpoint** | Agent durumunun anlık görüntüsü |
| **Permission** | Agent'a verilen atomik yetki birimi |
| **Audit Log** | Tüm operasyonların değiştirilemez kaydı |
| **Log Rotation** | Log dosyasının boyutu dolunca yenisini oluşturma |
| **Resource Monitor** | CPU/RAM/Disk kullanımını izleyen bileşen |

### B — Mini Checklist

#### Sandbox Kurulumu

- [ ] Sandbox tipi seçildi ve gerekçesi belgelendi
- [ ] Ağ izolasyonu uygulandı (üretimde şart)
- [ ] CPU ve RAM limitleri ayarlandı
- [ ] `nobody` veya eşdeğer düşük yetkili kullanıcıyla çalışıyor
- [ ] Geçici dosyalar temizleniyor (context manager veya finally)

#### File Operations

- [ ] Path traversal koruması aktif (`resolve()` + prefix check)
- [ ] Bloklanmış uzantı listesi tanımlı
- [ ] Dosya boyutu limiti var
- [ ] Versiyonlama etkin (üretimde önerilir)
- [ ] UTF-8 encoding her yerde tutarlı

#### Context & State

- [ ] TTL değerleri iş gereksinimlerine göre ayarlandı
- [ ] Context window token takibi var
- [ ] Checkpoint periyodu tanımlı
- [ ] Eski checkpoint'ler otomatik siliniyor

#### Güvenlik

- [ ] Permission manager her agent için yapılandırıldı
- [ ] Rol tabanlı atama uygulandı
- [ ] Audit logger aktif
- [ ] Log rotation yapılandırıldı
- [ ] Rate limiting kritik operasyonlara uygulandı

#### Kaynak Yönetimi

- [ ] ResourceMonitor çalışıyor
- [ ] Disk limit politikası tanımlı
- [ ] Otomatik temizleme zamanlanmış

---

**Serinin başına dön:** [README](./README.md)  
**Skills rehberine git:** [03 - Skills ve Capabilities](./03_-_Skills_ve_Capabilities.md)  
**System prompt rehberine git:** [05 - System Prompt Tasarımı](./05_-_System_Prompt_Tasarimi.md)
