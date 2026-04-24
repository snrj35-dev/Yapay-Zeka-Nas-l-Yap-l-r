# 02 — Tool Use ve Entegrasyon

Bu rehber, AI agent'ların tool kullanımını, function calling'i, API entegrasyonunu, güvenlik kontrollerini ve hata yönetimini kapsar.

**Önceki:** [01 - Agent Framework ve Mimari](./01_-_Agent_Framework_ve_Mimari.md)  
**Sonraki:** [03 - Skills ve Capabilities](./03_-_Skills_ve_Capabilities.md)

---

## 📌 Doküman Kartı

| Alan | Değer |
|---|---|
| Rol | Tool use ve entegrasyon referansı |
| Durum | Living specification (`v2.0`) |
| Son güncelleme | 2026-04-24 |
| Birincil okur | AI mühendisleri, backend geliştiricileri |
| Ana girdi | Tool definition, API spec, function schema |
| Ana çıktı | Tool implementation, integration code, selection logic |
| Bağımlı dokümanlar | [01 - Agent Framework](./01_-_Agent_Framework_ve_Mimari.md), [04 - Workspace](./04_-_Calisma_Alani_Yonetimi.md) |

> **Kalite notu:** Kod blokları referans implementasyon örnekleridir. Production'a almadan önce kendi ortamınızda test edin.

---

## İçindekiler

1. [Tool Nedir?](#1-tool-nedir)
2. [Tool Tanımlama](#2-tool-tanımlama)
3. [Function Calling](#3-function-calling)
4. [REST API Entegrasyonu](#4-rest-api-entegrasyonu)
5. [Database Tools](#5-database-tools)
6. [File System Tools](#6-file-system-tools)
7. [Tool Selection Stratejileri](#7-tool-selection-stratejileri)
8. [Hata Yönetimi ve Dayanıklılık](#8-hata-yönetimi-ve-dayanıklılık)
9. [Tool Güvenliği](#9-tool-güvenliği)
10. [Tool Test Etme](#10-tool-test-etme)
11. [Türkçe Tool İçin Pratik Notlar](#11-türkçe-tool-için-pratik-notlar)
12. [Ekler](#ekler)

---

## 1. Tool Nedir?

### Tanım

Tool, AI agent'ın LLM dışındaki sistemlerle etkileşim kurduğu yapılandırılmış arayüzdür. Her tool; bir schema (ne yapabileceği), bir implementasyon (nasıl yapacağı) ve çıktı (sonuç) üçlüsünden oluşur.

### Tool Sınıflandırması

| Kategori | Örnekler | Güvenlik Seviyesi |
|----------|----------|-------------------|
| **Read-only** | Web arama, DB okuma, dosya okuma | Düşük risk |
| **Write** | Dosya yazma, DB güncelleme | Orta risk — geri alınamaz olabilir |
| **Execute** | Kod çalıştırma, shell komutu | Yüksek risk — sandbox şart |
| **External** | E-posta gönderme, API çağrısı | Yüksek risk — yan etki |
| **Stateful** | Tarayıcı oturumu, DB transaction | Yönetim gerektirir |

### Tool Bileşenleri

```
Tool
 ├── name          → Benzersiz, açıklayıcı tanımlayıcı
 ├── description   → LLM'in aracı anlaması için doğal dil açıklama
 ├── parameters    → JSON Schema formatında girdi tanımı
 ├── execute()     → Gerçek implementasyon
 ├── validate()    → Girdi doğrulama
 └── to_schema()   → LLM'e gönderilecek schema üretimi
```

---

## 2. Tool Tanımlama

### 2.1 Temel Tool Sınıfı

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
import time
import logging

logger = logging.getLogger(__name__)

class Tool(ABC):
    """Tüm tool'ların türediği temel sınıf."""

    def __init__(
        self,
        name: str,
        description: str,
        parameters: Dict[str, Any],
        category: str = "general",
        is_read_only: bool = True,
        requires_confirmation: bool = False
    ):
        self.name                 = name
        self.description          = description
        self.parameters           = parameters
        self.category             = category
        self.is_read_only         = is_read_only
        self.requires_confirmation = requires_confirmation
        self._call_count          = 0
        self._error_count         = 0

    def execute(self, **kwargs) -> Dict[str, Any]:
        """Tool'u çalıştır. Süreyi ve hataları otomatik kayıt altına alır."""
        self._call_count += 1
        start = time.time()

        try:
            self.validate(**kwargs)
            result = self._execute(**kwargs)
            duration_ms = int((time.time() - start) * 1000)
            logger.info(f"[{self.name}] Başarılı | {duration_ms}ms | args={kwargs}")
            return {"success": True, "result": result, "duration_ms": duration_ms}
        except ToolValidationError as e:
            self._error_count += 1
            logger.warning(f"[{self.name}] Doğrulama hatası: {e}")
            return {"success": False, "error": f"Geçersiz girdi: {e}", "type": "validation"}
        except Exception as e:
            self._error_count += 1
            logger.error(f"[{self.name}] Hata: {e}", exc_info=True)
            return {"success": False, "error": str(e), "type": "execution"}

    @abstractmethod
    def _execute(self, **kwargs) -> Any:
        """Alt sınıf implement eder."""
        ...

    def validate(self, **kwargs):
        """Zorunlu parametreleri kontrol et. Hata varsa ToolValidationError fırlat."""
        required = self.parameters.get("required", [])
        for field in required:
            if field not in kwargs:
                raise ToolValidationError(f"Zorunlu parametre eksik: '{field}'")

    def to_schema(self) -> Dict[str, Any]:
        """OpenAI/Claude function calling formatında schema döndürür."""
        return {
            "name":        self.name,
            "description": self.description,
            "parameters":  self.parameters
        }

    @property
    def stats(self) -> dict:
        return {
            "name":        self.name,
            "calls":       self._call_count,
            "errors":      self._error_count,
            "error_rate":  self._error_count / self._call_count if self._call_count else 0
        }

class ToolValidationError(Exception):
    pass
```

### 2.2 JSON Schema En İyi Pratikler

```json
{
  "name": "search_web",
  "description": "Web'de güncel bilgi aramak için kullanılır. Haber, fiyat, belgeleme veya gerçek zamanlı veri gerektiğinde tercih edin. Genel bilgi soruları için kullanmayın.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Arama sorgusu. Spesifik ve öz olun. Maksimum 200 karakter.",
        "maxLength": 200
      },
      "num_results": {
        "type": "integer",
        "description": "Döndürülecek sonuç sayısı (1–10 arası).",
        "minimum": 1,
        "maximum": 10,
        "default": 5
      },
      "language": {
        "type": "string",
        "description": "Sonuç dili. ISO 639-1 kodu (tr, en, de...).",
        "enum": ["tr", "en", "de", "fr", "es"],
        "default": "tr"
      },
      "date_filter": {
        "type": "string",
        "description": "Tarih filtresi.",
        "enum": ["any", "past_day", "past_week", "past_month", "past_year"],
        "default": "any"
      }
    },
    "required": ["query"]
  }
}
```

**Schema yazarken dikkat edilecekler:**
- `description`'ı LLM için yazın; hangi durumda kullanılacağını belirtin
- Tür kısıtlamalarını (`minimum`, `maximum`, `maxLength`) ekleyin
- `enum` ile olası değerleri sınırlandırın
- `default` değerleri her zaman belirtin

---

## 3. Function Calling

### 3.1 OpenAI / Claude Uyumlu Döngü

```python
import json
import openai
from typing import List

class FunctionCallingLoop:
    """
    LLM ↔ Tool arasındaki çağrı döngüsünü yönetir.
    OpenAI ve Claude API'si ile uyumludur.
    """

    MAX_ITERATIONS = 10

    def __init__(self, llm_client, tools: List[Tool]):
        self.client     = llm_client
        self.tools      = {t.name: t for t in tools}
        self.tool_schemas = [t.to_schema() for t in tools]

    def run(self, system_prompt: str, user_message: str) -> str:
        messages = [
            {"role": "system",  "content": system_prompt},
            {"role": "user",    "content": user_message}
        ]

        for iteration in range(self.MAX_ITERATIONS):
            response = self.client.chat.completions.create(
                model    = "gpt-4o",
                messages = messages,
                tools    = [{"type": "function", "function": s} for s in self.tool_schemas],
                tool_choice = "auto"
            )

            msg = response.choices[0].message

            # Model doğrudan yanıt verdiyse döngüyü bitir
            if msg.content and not msg.tool_calls:
                return msg.content

            # Tool çağrısı yoksa döngüyü bitir
            if not msg.tool_calls:
                return msg.content or ""

            # Tool çağrılarını yürüt
            messages.append(msg)
            for tool_call in msg.tool_calls:
                result = self._execute_tool_call(tool_call)
                messages.append({
                    "role":         "tool",
                    "tool_call_id": tool_call.id,
                    "content":      json.dumps(result, ensure_ascii=False)
                })

        return "[HATA] Maksimum iterasyon sayısına ulaşıldı."

    def _execute_tool_call(self, tool_call) -> dict:
        name = tool_call.function.name
        try:
            args = json.loads(tool_call.function.arguments)
        except json.JSONDecodeError:
            return {"success": False, "error": "Geçersiz JSON argüman"}

        tool = self.tools.get(name)
        if not tool:
            return {"success": False, "error": f"'{name}' aracı bulunamadı."}

        return tool.execute(**args)
```

### 3.2 Paralel Tool Çağrısı

OpenAI GPT-4o ve Claude 3+ modelleri aynı turda birden fazla tool çağrısı gönderebilir. Bunları paralel çalıştırmak önemlidir:

```python
import asyncio

class AsyncFunctionCallingLoop(FunctionCallingLoop):

    async def _execute_tool_calls_parallel(self, tool_calls: list) -> list:
        tasks = [
            asyncio.to_thread(self._execute_tool_call, tc)
            for tc in tool_calls
        ]
        return await asyncio.gather(*tasks, return_exceptions=True)

    async def run_async(self, system_prompt: str, user_message: str) -> str:
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user",   "content": user_message}
        ]

        for _ in range(self.MAX_ITERATIONS):
            response = await asyncio.to_thread(
                self.client.chat.completions.create,
                model    = "gpt-4o",
                messages = messages,
                tools    = [{"type": "function", "function": s} for s in self.tool_schemas]
            )
            msg = response.choices[0].message

            if not msg.tool_calls:
                return msg.content or ""

            messages.append(msg)
            results = await self._execute_tool_calls_parallel(msg.tool_calls)

            for tool_call, result in zip(msg.tool_calls, results):
                if isinstance(result, Exception):
                    result = {"success": False, "error": str(result)}
                messages.append({
                    "role":         "tool",
                    "tool_call_id": tool_call.id,
                    "content":      json.dumps(result, ensure_ascii=False)
                })

        return "[HATA] Maksimum iterasyon sayısına ulaşıldı."
```

---

## 4. REST API Entegrasyonu

### 4.1 HTTP Tool Temel Sınıfı

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import os

class HTTPTool(Tool):
    """
    HTTP tabanlı API entegrasyonları için temel sınıf.
    Otomatik retry, timeout ve session yönetimi içerir.
    """

    DEFAULT_TIMEOUT = (5, 30)   # (connect, read) saniye

    def __init__(self, name: str, description: str, parameters: dict,
                 base_url: str, api_key_env: str = None,
                 max_retries: int = 3):
        super().__init__(name=name, description=description, parameters=parameters)
        self.base_url = base_url.rstrip("/")
        self.api_key  = os.environ.get(api_key_env, "") if api_key_env else ""
        self.session  = self._build_session(max_retries)

    def _build_session(self, max_retries: int) -> requests.Session:
        session = requests.Session()
        retry   = Retry(
            total              = max_retries,
            backoff_factor     = 0.5,
            status_forcelist   = [429, 500, 502, 503, 504],
            allowed_methods    = ["GET", "POST"]
        )
        adapter = HTTPAdapter(max_retries=retry)
        session.mount("https://", adapter)
        session.mount("http://",  adapter)
        if self.api_key:
            session.headers.update({"Authorization": f"Bearer {self.api_key}"})
        return session

    def _get(self, endpoint: str, params: dict = None) -> dict:
        url  = f"{self.base_url}/{endpoint.lstrip('/')}"
        resp = self.session.get(url, params=params, timeout=self.DEFAULT_TIMEOUT)
        resp.raise_for_status()
        return resp.json()

    def _post(self, endpoint: str, body: dict = None) -> dict:
        url  = f"{self.base_url}/{endpoint.lstrip('/')}"
        resp = self.session.post(url, json=body, timeout=self.DEFAULT_TIMEOUT)
        resp.raise_for_status()
        return resp.json()
```

### 4.2 Hava Durumu Tool Örneği

```python
class WeatherTool(HTTPTool):
    """OpenWeatherMap API ile hava durumu bilgisi çeker."""

    def __init__(self):
        super().__init__(
            name        = "get_weather",
            description = (
                "Bir şehir için anlık hava durumu bilgisi getirir. "
                "Sıcaklık, nem, rüzgar hızı ve genel durum döner. "
                "Hava tahmini için değil, anlık bilgi için kullanın."
            ),
            parameters  = {
                "type": "object",
                "properties": {
                    "city": {
                        "type":        "string",
                        "description": "Şehir adı (Türkçe karakter desteklenir, örn: İstanbul)"
                    },
                    "units": {
                        "type":        "string",
                        "description": "Sıcaklık birimi",
                        "enum":        ["metric", "imperial"],
                        "default":     "metric"
                    }
                },
                "required": ["city"]
            },
            base_url    = "https://api.openweathermap.org/data/2.5",
            api_key_env = "OPENWEATHER_API_KEY"
        )

    def _execute(self, city: str, units: str = "metric") -> dict:
        data = self._get("weather", params={
            "q":     city,
            "appid": self.api_key,
            "units": units,
            "lang":  "tr"
        })
        return {
            "şehir":         data["name"],
            "sıcaklık":      f"{data['main']['temp']}°{'C' if units == 'metric' else 'F'}",
            "hissedilen":    f"{data['main']['feels_like']}°{'C' if units == 'metric' else 'F'}",
            "nem":           f"%{data['main']['humidity']}",
            "durum":         data["weather"][0]["description"],
            "rüzgar":        f"{data['wind']['speed']} m/s"
        }
```

### 4.3 GraphQL Tool

```python
class GraphQLTool(HTTPTool):
    """Herhangi bir GraphQL endpoint'i ile çalışır."""

    def __init__(self, endpoint: str, name: str = "graphql_query"):
        super().__init__(
            name        = name,
            description = "GraphQL sorgusu çalıştırır ve veri döndürür.",
            parameters  = {
                "type": "object",
                "properties": {
                    "query": {
                        "type":        "string",
                        "description": "GraphQL sorgu veya mutation metni"
                    },
                    "variables": {
                        "type":        "object",
                        "description": "Sorgu değişkenleri (opsiyonel)"
                    }
                },
                "required": ["query"]
            },
            base_url    = endpoint
        )

    def _execute(self, query: str, variables: dict = None) -> dict:
        payload  = {"query": query}
        if variables:
            payload["variables"] = variables

        resp = self.session.post(
            self.base_url,
            json    = payload,
            timeout = self.DEFAULT_TIMEOUT
        )
        resp.raise_for_status()
        data = resp.json()

        if "errors" in data:
            raise RuntimeError(f"GraphQL hataları: {data['errors']}")
        return data.get("data", {})
```

---

## 5. Database Tools

### 5.1 SQL Tool

```python
import sqlite3
import contextlib
from typing import List, Tuple, Any

class SQLTool(Tool):
    """
    SQLite veritabanı için güvenli sorgu aracı.
    SQL injection koruması, parametreli sorgular ve transaction desteği içerir.
    """

    ALLOWED_STATEMENT_TYPES = {"SELECT", "INSERT", "UPDATE", "DELETE", "WITH"}

    def __init__(self, db_path: str, read_only: bool = False):
        super().__init__(
            name         = "execute_sql",
            description  = (
                "SQL sorgusu çalıştırır. SELECT için veri döndürür; "
                "INSERT/UPDATE/DELETE için etkilenen satır sayısını döndürür. "
                "SQL injection riskine karşı parametreli sorgular kullanın."
            ),
            parameters   = {
                "type": "object",
                "properties": {
                    "query": {
                        "type":        "string",
                        "description": "Çalıştırılacak SQL sorgusu. Değerler için ? placeholder kullanın."
                    },
                    "params": {
                        "type":        "array",
                        "description": "? placeholder'larına karşılık gelen değerler",
                        "items":       {"type": ["string", "number", "boolean", "null"]},
                        "default":     []
                    }
                },
                "required": ["query"]
            },
            is_read_only = read_only
        )
        self.db_path   = db_path
        self.read_only = read_only

    def _execute(self, query: str, params: List[Any] = None) -> dict:
        params = params or []

        # Tehlikeli komutları engelle
        stmt_type = query.strip().split()[0].upper()
        if stmt_type not in self.ALLOWED_STATEMENT_TYPES:
            raise ToolValidationError(f"İzin verilmeyen SQL komutu: {stmt_type}")

        if self.read_only and stmt_type != "SELECT":
            raise ToolValidationError("Bu tool yalnızca SELECT sorgularına izin verir.")

        with contextlib.closing(sqlite3.connect(self.db_path)) as conn:
            conn.row_factory = sqlite3.Row
            with conn:
                cursor = conn.execute(query, params)

                if stmt_type == "SELECT" or stmt_type == "WITH":
                    rows    = cursor.fetchall()
                    columns = [d[0] for d in cursor.description] if cursor.description else []
                    return {
                        "columns":    columns,
                        "rows":       [dict(r) for r in rows],
                        "row_count":  len(rows)
                    }
                else:
                    return {"affected_rows": cursor.rowcount}
```

### 5.2 Vector Search Tool

```python
import chromadb
from sentence_transformers import SentenceTransformer

class VectorSearchTool(Tool):
    """
    Doküman koleksiyonunda semantik arama yapar.
    Keyword aramasından farklı olarak anlam benzerliğine göre sonuç döndürür.
    """

    def __init__(
        self,
        collection_name: str,
        model_name: str = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
    ):
        super().__init__(
            name        = "vector_search",
            description = (
                "Doküman koleksiyonunda semantik (anlam bazlı) arama yapar. "
                "Exact keyword eşleşmesi yerine kavramsal yakınlığa göre sonuç döndürür. "
                "Türkçe sorgular desteklenir."
            ),
            parameters  = {
                "type": "object",
                "properties": {
                    "query": {
                        "type":        "string",
                        "description": "Arama sorgusu (doğal dilde yazılabilir)"
                    },
                    "n_results": {
                        "type":        "integer",
                        "description": "Döndürülecek sonuç sayısı",
                        "minimum":     1,
                        "maximum":     20,
                        "default":     5
                    },
                    "min_score": {
                        "type":        "number",
                        "description": "Minimum benzerlik skoru (0–1 arası)",
                        "minimum":     0.0,
                        "maximum":     1.0,
                        "default":     0.5
                    }
                },
                "required": ["query"]
            }
        )
        self.model      = SentenceTransformer(model_name)
        self.client     = chromadb.Client()
        self.collection = self.client.get_or_create_collection(collection_name)

    def _execute(self, query: str, n_results: int = 5, min_score: float = 0.5) -> dict:
        embedding = self.model.encode([query])[0].tolist()
        results   = self.collection.query(
            query_embeddings = [embedding],
            n_results        = n_results,
            include          = ["documents", "metadatas", "distances"]
        )

        hits = []
        for doc, meta, dist in zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0]
        ):
            score = 1 - dist   # cosine distance → similarity
            if score >= min_score:
                hits.append({"content": doc, "metadata": meta, "score": round(score, 4)})

        return {
            "query":   query,
            "hits":    sorted(hits, key=lambda x: x["score"], reverse=True),
            "total":   len(hits)
        }

    def add_documents(self, documents: List[str], metadatas: List[dict] = None):
        """Koleksiyona doküman ekler."""
        import uuid
        ids        = [str(uuid.uuid4()) for _ in documents]
        embeddings = self.model.encode(documents).tolist()
        self.collection.add(
            ids        = ids,
            embeddings = embeddings,
            documents  = documents,
            metadatas  = metadatas or [{} for _ in documents]
        )
```

---

## 6. File System Tools

### 6.1 Güvenli Dosya Yönetimi

```python
import os
import pathlib
import hashlib
from datetime import datetime

class SafeFileReadTool(Tool):
    """
    Sandbox içinde güvenli dosya okuma.
    Path traversal saldırısına karşı korunur.
    """

    MAX_FILE_SIZE_MB = 10

    def __init__(self, base_path: str):
        super().__init__(
            name        = "read_file",
            description = (
                "Workspace içindeki bir dosyanın içeriğini okur. "
                "İkili (binary) dosyalar desteklenmez. "
                f"Maksimum dosya boyutu: {self.MAX_FILE_SIZE_MB} MB."
            ),
            parameters  = {
                "type": "object",
                "properties": {
                    "path": {
                        "type":        "string",
                        "description": "Workspace'e göre göreli dosya yolu (örn: data/report.txt)"
                    },
                    "encoding": {
                        "type":        "string",
                        "description": "Karakter kodlaması",
                        "enum":        ["utf-8", "utf-16", "latin-1", "cp1254"],
                        "default":     "utf-8"
                    }
                },
                "required": ["path"]
            },
            is_read_only = True
        )
        self.base_path = pathlib.Path(base_path).resolve()

    def _execute(self, path: str, encoding: str = "utf-8") -> dict:
        full_path = self._safe_resolve(path)

        size_mb = full_path.stat().st_size / (1024 * 1024)
        if size_mb > self.MAX_FILE_SIZE_MB:
            raise ToolValidationError(
                f"Dosya çok büyük: {size_mb:.1f} MB (limit: {self.MAX_FILE_SIZE_MB} MB)"
            )

        content = full_path.read_text(encoding=encoding)
        return {
            "path":       str(path),
            "content":    content,
            "size_bytes": full_path.stat().st_size,
            "encoding":   encoding,
            "lines":      content.count("\n") + 1
        }

    def _safe_resolve(self, path: str) -> pathlib.Path:
        """Path traversal koruması."""
        full = (self.base_path / path).resolve()
        if not str(full).startswith(str(self.base_path)):
            raise ToolValidationError("Workspace dışına erişim girişimi engellendi.")
        if not full.exists():
            raise FileNotFoundError(f"Dosya bulunamadı: {path}")
        if not full.is_file():
            raise ToolValidationError(f"Bu bir dosya değil: {path}")
        return full


class SafeFileWriteTool(Tool):
    """
    Sandbox içinde güvenli dosya yazma.
    Otomatik yedekleme ve content hash doğrulaması içerir.
    """

    FORBIDDEN_EXTENSIONS = {".exe", ".sh", ".bat", ".cmd", ".dll", ".so"}

    def __init__(self, base_path: str, enable_backup: bool = True):
        super().__init__(
            name        = "write_file",
            description = "Workspace içindeki bir dosyaya metin içerik yazar. Mevcut dosya varsa yedeklenir.",
            parameters  = {
                "type": "object",
                "properties": {
                    "path": {
                        "type":        "string",
                        "description": "Workspace'e göre göreli dosya yolu"
                    },
                    "content": {
                        "type":        "string",
                        "description": "Dosyaya yazılacak metin içerik"
                    },
                    "mode": {
                        "type":        "string",
                        "description": "Yazma modu: overwrite (üzerine yaz) veya append (ekle)",
                        "enum":        ["overwrite", "append"],
                        "default":     "overwrite"
                    }
                },
                "required": ["path", "content"]
            },
            is_read_only         = False,
            requires_confirmation = True
        )
        self.base_path     = pathlib.Path(base_path).resolve()
        self.enable_backup = enable_backup

    def _execute(self, path: str, content: str, mode: str = "overwrite") -> dict:
        suffix = pathlib.Path(path).suffix.lower()
        if suffix in self.FORBIDDEN_EXTENSIONS:
            raise ToolValidationError(f"Bu uzantıya dosya yazmak yasaktır: {suffix}")

        full_path = self._safe_resolve_parent(path)

        # Mevcut dosyayı yedekle
        backed_up = False
        if self.enable_backup and full_path.exists():
            backup = full_path.with_suffix(
                f".{datetime.now().strftime('%Y%m%d_%H%M%S')}.bak"
            )
            full_path.rename(backup)
            backed_up = True

        write_mode = "a" if mode == "append" else "w"
        full_path.write_text(content, encoding="utf-8") if write_mode == "w" else \
            full_path.open("a", encoding="utf-8").write(content)

        return {
            "path":       str(path),
            "bytes":      len(content.encode("utf-8")),
            "mode":       mode,
            "backed_up":  backed_up,
            "sha256":     hashlib.sha256(content.encode()).hexdigest()[:8]
        }

    def _safe_resolve_parent(self, path: str) -> pathlib.Path:
        full = (self.base_path / path).resolve()
        if not str(full).startswith(str(self.base_path)):
            raise ToolValidationError("Workspace dışına yazma girişimi engellendi.")
        full.parent.mkdir(parents=True, exist_ok=True)
        return full
```

---

## 7. Tool Selection Stratejileri

### 7.1 Semantik Seçim

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticToolSelector:
    """
    Görev açıklamasına göre en uygun tool'ları semantik benzerlikle seçer.
    Türkçe sorgular için çok dilli model kullanır.
    """

    def __init__(
        self,
        tools: List[Tool],
        model_name: str = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
    ):
        self.tools  = tools
        self.model  = SentenceTransformer(model_name)
        descriptions = [f"{t.name}: {t.description}" for t in tools]
        self._embeddings = self.model.encode(descriptions, normalize_embeddings=True)

    def select(self, query: str, top_k: int = 3, threshold: float = 0.4) -> List[Tool]:
        q_emb = self.model.encode([query], normalize_embeddings=True)[0]
        scores = self._embeddings @ q_emb   # cosine similarity (normalized vectors)

        # Threshold altını filtrele
        candidates = [
            (self.tools[i], float(scores[i]))
            for i in range(len(self.tools))
            if scores[i] >= threshold
        ]
        candidates.sort(key=lambda x: x[1], reverse=True)
        return [t for t, _ in candidates[:top_k]]

    def select_with_scores(self, query: str, top_k: int = 3) -> List[dict]:
        selected = [(t, s) for t, s in zip(
            self.select(query, top_k=top_k, threshold=0.0),
            []
        )]
        q_emb  = self.model.encode([query], normalize_embeddings=True)[0]
        scores = self._embeddings @ q_emb
        result = sorted(
            [{"tool": t.name, "score": round(float(s), 4)} for t, s in zip(self.tools, scores)],
            key=lambda x: x["score"], reverse=True
        )
        return result[:top_k]
```

### 7.2 Kural Tabanlı Yönlendirme

```python
import re

class RuleBasedToolRouter:
    """
    Anahtar kelime ve regex kurallarıyla hızlı tool yönlendirmesi.
    Semantic seçimden önce çalıştırıldığında latency azalır.
    """

    def __init__(self, tools: List[Tool]):
        self.tools      = {t.name: t for t in tools}
        self.rules: List[dict] = []

    def add_rule(self, pattern: str, tool_names: List[str], priority: int = 0):
        """
        pattern: regex deseni (görev metnine uygulanır)
        tool_names: eşleşince seçilecek tool isimleri
        priority: yüksek öncelikli kural daha önce değerlendirilir
        """
        self.rules.append({
            "pattern":    re.compile(pattern, re.IGNORECASE),
            "tool_names": tool_names,
            "priority":   priority
        })
        self.rules.sort(key=lambda r: r["priority"], reverse=True)

    def route(self, query: str) -> List[Tool]:
        for rule in self.rules:
            if rule["pattern"].search(query):
                return [self.tools[n] for n in rule["tool_names"] if n in self.tools]
        return list(self.tools.values())   # Eşleşme yoksa hepsini döndür

# Kullanım örneği
router = RuleBasedToolRouter(tools=[...])
router.add_rule(r"\b(hava|sıcaklık|yağış|rüzgar)\b",   ["get_weather"],    priority=10)
router.add_rule(r"\b(ara|bul|search|sorgula)\b",         ["search_web"],     priority=8)
router.add_rule(r"\b(dosya|oku|yaz|kaydet)\b",           ["read_file", "write_file"], priority=9)
router.add_rule(r"\b(sql|tablo|veritabanı|db)\b",        ["execute_sql"],    priority=9)
router.add_rule(r"\b(semantik|benzer|yakın|anlam)\b",    ["vector_search"],  priority=7)
```

---

## 8. Hata Yönetimi ve Dayanıklılık

### 8.1 Retry with Exponential Backoff

```python
import time
import random
from functools import wraps

def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    backoff_factor: float = 2.0,
    jitter: bool = True,
    retryable_exceptions: tuple = (Exception,)
):
    """
    Dekoratör: başarısız çağrıları üstel geri çekilme ile yeniden dener.
    jitter=True → thundering herd sorununu önler.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            delay = base_delay
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except retryable_exceptions as e:
                    if attempt == max_attempts:
                        raise
                    sleep_time = min(delay, max_delay)
                    if jitter:
                        sleep_time *= (0.5 + random.random())
                    logger.warning(
                        f"Deneme {attempt}/{max_attempts} başarısız: {e}. "
                        f"{sleep_time:.1f}s sonra tekrar denenecek."
                    )
                    time.sleep(sleep_time)
                    delay *= backoff_factor
        return wrapper
    return decorator

# Kullanım
class WeatherTool(HTTPTool):
    @retry(max_attempts=3, base_delay=1.0, retryable_exceptions=(requests.HTTPError, ConnectionError))
    def _execute(self, city: str, units: str = "metric") -> dict:
        ...
```

### 8.2 Fallback Zinciri

```python
class FallbackToolChain:
    """
    Birincil tool başarısız olursa sıradaki tool'u dener.
    """

    def __init__(self, tools: List[Tool]):
        if not tools:
            raise ValueError("En az bir tool gereklidir.")
        self.tools = tools

    def execute(self, **kwargs) -> dict:
        errors = []
        for tool in self.tools:
            try:
                result = tool.execute(**kwargs)
                if result.get("success"):
                    result["_source"] = tool.name
                    return result
                errors.append(f"{tool.name}: {result.get('error')}")
            except Exception as e:
                errors.append(f"{tool.name}: {e}")
                logger.warning(f"Fallback: {tool.name} başarısız → {e}")

        return {
            "success": False,
            "error":   "Tüm fallback seçenekleri başarısız oldu.",
            "details": errors
        }

# Örnek: önce premium arama, sonra ücretsiz
search_chain = FallbackToolChain([
    BraveSearchTool(),    # Önce dene
    DuckDuckGoTool(),     # Başarısız olursa
    CachedSearchTool()    # Son çare
])
```

---

## 9. Tool Güvenliği

### 9.1 Rate Limiting

```python
import threading
from collections import deque

class RateLimitedTool(Tool):
    """Tool çağrılarını zaman penceresine göre sınırlandırır."""

    def __init__(self, tool: Tool, max_calls: int, window_seconds: int):
        super().__init__(
            name        = tool.name,
            description = tool.description,
            parameters  = tool.parameters
        )
        self._tool           = tool
        self.max_calls       = max_calls
        self.window_seconds  = window_seconds
        self._call_times     = deque()
        self._lock           = threading.Lock()

    def _execute(self, **kwargs) -> Any:
        with self._lock:
            now    = time.time()
            cutoff = now - self.window_seconds

            # Pencere dışı çağrıları temizle
            while self._call_times and self._call_times[0] < cutoff:
                self._call_times.popleft()

            if len(self._call_times) >= self.max_calls:
                wait = self._call_times[0] + self.window_seconds - now
                raise RuntimeError(
                    f"Rate limit aşıldı: {self.max_calls} çağrı / {self.window_seconds}s. "
                    f"{wait:.1f}s sonra tekrar deneyin."
                )

            self._call_times.append(now)

        return self._tool._execute(**kwargs)
```

### 9.2 Input Sanitization

```python
import re

class InputSanitizer:
    """Tool girdilerini temizler ve tehlikeli içerik tespit eder."""

    SQL_INJECTION_PATTERNS = [
        r"(\b(UNION|SELECT|INSERT|UPDATE|DELETE|DROP|EXEC|EXECUTE)\b.*\b(FROM|INTO|TABLE|DATABASE)\b)",
        r"(--|\#|\/\*|\*\/)",
        r"(\bOR\b\s+\d+\s*=\s*\d+)"
    ]

    COMMAND_INJECTION_PATTERNS = [
        r"(;|\||&&|\$\(|`)",
        r"(\.\./|\.\.\\)",
        r"(\bsudo\b|\brm\b|\bmv\b|\bchmod\b)"
    ]

    @classmethod
    def sanitize_string(cls, value: str, context: str = "general") -> str:
        if not isinstance(value, str):
            return value

        if context == "sql":
            for pattern in cls.SQL_INJECTION_PATTERNS:
                if re.search(pattern, value, re.IGNORECASE):
                    raise ToolValidationError(f"Olası SQL injection tespit edildi: {value[:50]}")

        if context == "shell":
            for pattern in cls.COMMAND_INJECTION_PATTERNS:
                if re.search(pattern, value):
                    raise ToolValidationError(f"Olası komut enjeksiyonu tespit edildi: {value[:50]}")

        # Maksimum uzunluk
        return value[:10000]
```

---

## 10. Tool Test Etme

### 10.1 Unit Test Template

```python
import pytest
from unittest.mock import patch, MagicMock

class TestWeatherTool:

    @pytest.fixture
    def tool(self):
        with patch.dict("os.environ", {"OPENWEATHER_API_KEY": "test-key"}):
            return WeatherTool()

    def test_execute_returns_expected_fields(self, tool):
        mock_response = {
            "name": "İstanbul",
            "main": {"temp": 22.5, "feels_like": 21.0, "humidity": 65},
            "weather": [{"description": "parçalı bulutlu"}],
            "wind": {"speed": 4.2}
        }
        with patch.object(tool.session, "get") as mock_get:
            mock_get.return_value = MagicMock(
                status_code = 200,
                json        = lambda: mock_response
            )
            mock_get.return_value.raise_for_status = lambda: None

            result = tool.execute(city="İstanbul")

        assert result["success"] is True
        assert "sıcaklık" in result["result"]
        assert "nem" in result["result"]

    def test_missing_required_param_raises_validation_error(self, tool):
        result = tool.execute()   # city eksik
        assert result["success"] is False
        assert result["type"] == "validation"

    def test_rate_limiting(self, tool):
        limited = RateLimitedTool(tool, max_calls=2, window_seconds=5)
        # İlk 2 çağrı geçmeli
        for _ in range(2):
            limited._call_times.append(time.time())
        # 3. çağrı rate limit'e takılmalı
        with pytest.raises(RuntimeError, match="Rate limit"):
            limited._execute(city="Ankara")
```

---

## 11. Türkçe Tool İçin Pratik Notlar

> **Tool Açıklamaları:** Açıklamayı Türkçe yazın; hangi durumda, hangi sınırlamalarla kullanılacağını belirtin. LLM tool seçimini bu açıklamaya göre yapar.

> **Parameter Açıklamaları:** Her parametreyi Türkçe örnekle açıklayın (örn: `"şehir adı, örn: İzmir"`).

> **Hata Mesajları:** Kullanıcıya dönen hataları Türkçe yazın, iç logları İngilizce bırakabilirsiniz.

> **Encoding:** Tüm string işlemlerinde `encoding="utf-8"` ve `json.dumps(..., ensure_ascii=False)` kullanın.

> **Test Sorguları:** Tool'larınızı Türkçe girdilerle test edin; özellikle `ş`, `ğ`, `ı`, `ç`, `ö`, `ü` karakterlerini deneyin.

> **Tarih/Saat:** Türkiye saat dilimini (UTC+3) `pytz` veya `zoneinfo` ile yönetin:
> ```python
> from zoneinfo import ZoneInfo
> from datetime import datetime
> now_tr = datetime.now(tz=ZoneInfo("Europe/Istanbul"))
> ```

---

## Ekler

### A — Terimler Sözlüğü

| Terim | Açıklama |
|---|---|
| **Tool** | Agent'ın dış sistemlerle etkileşim kurduğu yapılandırılmış arayüz |
| **Function Calling** | LLM'in structured JSON ile tool çağırma yeteneği |
| **Schema** | Tool'un girdi parametrelerini tanımlayan JSON yapısı |
| **Retry** | Başarısız çağrıları yeniden deneme mekanizması |
| **Fallback** | Birincil başarısız olunca devreye giren alternatif |
| **Rate Limiting** | Belirli zaman diliminde çağrı sayısını sınırlama |
| **Circuit Breaker** | Sürekli hata veren servislere çağrıyı durduran örüntü |
| **Path Traversal** | `../` gibi dizin geçiş saldırısı |

### B — Mini Checklist

#### Tool Tasarımı

- [ ] `name` benzersiz ve fiil ile başlıyor (örn: `get_`, `search_`, `write_`)
- [ ] `description` LLM'e ne zaman kullanacağını söylüyor
- [ ] Zorunlu parametreler `required` listesinde
- [ ] Tüm parametrelerde `description` ve gerekiyorsa `default` var
- [ ] `is_read_only` ve `requires_confirmation` doğru ayarlandı

#### Güvenlik

- [ ] Path traversal koruması var (file tools)
- [ ] Parametreli sorgular kullanılıyor (SQL tools)
- [ ] Rate limiting uygulandı
- [ ] Input sanitization yapılıyor
- [ ] API key'ler environment variable'dan okunuyor

#### Hata Yönetimi

- [ ] `ToolValidationError` ile doğrulama hataları ayrılıyor
- [ ] Retry logic eklendi (ağ hataları için)
- [ ] Fallback zinciri tanımlandı
- [ ] Tüm hatalar loglanıyor

#### Test

- [ ] Başarılı senaryo test edildi
- [ ] Eksik parametre testi var
- [ ] Geçersiz değer testi var
- [ ] Rate limit testi var
- [ ] Türkçe karakter testi var

---

**Serinin başına dön:** [README](./README.md)  
**Agent framework rehberi:** [01 - Agent Framework ve Mimari](./01_-_Agent_Framework_ve_Mimari.md)  
**Skills rehberi:** [03 - Skills ve Capabilities](./03_-_Skills_ve_Capabilities.md)
