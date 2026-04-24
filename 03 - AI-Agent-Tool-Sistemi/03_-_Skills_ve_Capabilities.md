# 03 — Skills ve Capabilities

Bu rehber, tool soyutlamalarının üzerinde yer alan skill katmanını, skill discovery stratejilerini, composition pattern'lerini ve evaluation framework'ünü kapsar.

**Önceki:** [02 - Tool Use ve Entegrasyon](./02_-_Tool_Use_ve_Entegrasyon.md)  
**Sonraki:** [04 - Çalışma Alanı Yönetimi](./04_-_Calisma_Alani_Yonetimi.md)

---

## 📌 Doküman Kartı

| Alan | Değer |
|---|---|
| Rol | Skill tanımlama ve yönetim referansı |
| Durum | Living specification (`v2.0`) |
| Son güncelleme | 2026-04-24 |
| Birincil okur | AI mühendisleri, skill geliştiricileri |
| Ana girdi | Skill definition, capability spec, task requirement |
| Ana çıktı | Skill implementation, discovery logic, composition strategy |
| Bağımlı dokümanlar | [01 - Agent Framework](./01_-_Agent_Framework_ve_Mimari.md), [02 - Tool Use](./02_-_Tool_Use_ve_Entegrasyon.md) |

> **Kalite notu:** Kod blokları referans implementasyon örnekleridir. Production'a almadan önce test edin.

---

## İçindekiler

1. [Skill Nedir?](#1-skill-nedir)
2. [Skill Tanımlama Framework](#2-skill-tanımlama-framework)
3. [Skill Discovery](#3-skill-discovery)
4. [Skill Composition](#4-skill-composition)
5. [Skill Versiyonlama](#5-skill-versiyonlama)
6. [Skill Evaluation](#6-skill-evaluation)
7. [Skill Registry](#7-skill-registry)
8. [Türkçe Skill İçin Pratik Notlar](#8-türkçe-skill-için-pratik-notlar)
9. [Ekler](#ekler)

---

## 1. Skill Nedir?

### Tanım

Skill, bir veya birden fazla tool'u belirli bir iş hedefi doğrultusunda orkestre eden yüksek seviyeli yetenektir. Tool'lar **ne yapabileceğini** tanımlarken, skill'ler **nasıl ve ne zaman** yapılacağını tanımlar.

### Skill vs Tool vs Agent

```
Agent       → "Kullanıcının problemini çöz"          (en yüksek soyutlama)
  Skill     → "Aylık satış raporunu hazırla"         (görev odaklı)
    Tool    → execute_sql / write_file / send_email  (atomik işlem)
```

### Katmanlı Mimari

```
┌─────────────────────────────────────────┐
│              Agent Layer                │
│  Reasoning + Task Planning + Memory     │
├─────────────────────────────────────────┤
│              Skill Layer                │
│  analyze_sales | draft_report | notify  │
├────────────────┬────────────────────────┤
│   Tool Layer   │   Tool Layer           │
│  execute_sql   │  write_file / email    │
└────────────────┴────────────────────────┘
```

### Ne Zaman Skill Yazılır?

Şu koşullardan biri varsa tool yerine skill tercih edin:

- Birden fazla tool çağrısı her zaman bir arada gidiyorsa
- Aralarında sabit iş mantığı (koşul, sıra, dönüşüm) varsa
- Aynı örüntü farklı bağlamlarda tekrar ediyorsa
- Adımlar arası hata kurtarma gerektiriyorsa

---

## 2. Skill Tanımlama Framework

### 2.1 Skill Base Sınıfı

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Dict, Any, List, Optional
from enum import Enum
import time
import logging

logger = logging.getLogger(__name__)

class SkillStatus(Enum):
    STABLE      = "stable"       # Production'da kullanılabilir
    BETA        = "beta"         # Test aşamasında
    DEPRECATED  = "deprecated"   # Kullanım dışı bırakılıyor
    EXPERIMENTAL = "experimental" # Deneysel

@dataclass
class SkillMetadata:
    name:             str
    version:          str
    description:      str
    category:         str
    required_tools:   List[str]
    tags:             List[str] = field(default_factory=list)
    status:           SkillStatus = SkillStatus.STABLE
    author:           str = ""
    min_tool_version: Dict[str, str] = field(default_factory=dict)

@dataclass
class SkillResult:
    success:      bool
    output:       Any
    skill_name:   str
    duration_ms:  int
    steps_taken:  int = 0
    error:        Optional[str] = None
    metadata:     Dict[str, Any] = field(default_factory=dict)

class Skill(ABC):
    """Tüm skill'lerin türediği temel sınıf."""

    def __init__(self, metadata: SkillMetadata, tools: Dict[str, Any]):
        self.metadata    = metadata
        self.tools       = tools   # {tool_name: Tool instance}
        self._exec_count = 0
        self._fail_count = 0

    @property
    def name(self)        -> str:        return self.metadata.name
    @property
    def description(self) -> str:        return self.metadata.description
    @property
    def category(self)    -> str:        return self.metadata.category
    @property
    def version(self)     -> str:        return self.metadata.version

    def run(self, context: Dict[str, Any]) -> SkillResult:
        """Skill'i çalıştır; süreyi, hataları ve adımları otomatik yönet."""
        self._exec_count += 1
        start = time.time()

        if self.metadata.status == SkillStatus.DEPRECATED:
            logger.warning(f"[{self.name}] DEPRECATED skill kullanılıyor.")

        try:
            self._validate_context(context)
            output, steps = self._run(context)
            return SkillResult(
                success     = True,
                output      = output,
                skill_name  = self.name,
                duration_ms = int((time.time() - start) * 1000),
                steps_taken = steps
            )
        except SkillValidationError as e:
            self._fail_count += 1
            return SkillResult(
                success     = False,
                output      = None,
                skill_name  = self.name,
                duration_ms = int((time.time() - start) * 1000),
                error       = f"Doğrulama hatası: {e}"
            )
        except Exception as e:
            self._fail_count += 1
            logger.error(f"[{self.name}] Hata: {e}", exc_info=True)
            return SkillResult(
                success     = False,
                output      = None,
                skill_name  = self.name,
                duration_ms = int((time.time() - start) * 1000),
                error       = str(e)
            )

    @abstractmethod
    def _run(self, context: Dict[str, Any]) -> tuple:  # (output, steps)
        """Alt sınıf implement eder."""
        ...

    def _validate_context(self, context: Dict[str, Any]):
        """Bağlamda gerekli alanlar var mı kontrol et."""
        pass

    def _tool(self, name: str) -> Any:
        """Gerekli tool'u al; yoksa anlamlı hata ver."""
        tool = self.tools.get(name)
        if not tool:
            raise RuntimeError(
                f"[{self.name}] '{name}' tool'u bulunamadı. "
                f"Mevcut tool'lar: {list(self.tools.keys())}"
            )
        return tool

    @property
    def stats(self) -> dict:
        return {
            "name":       self.name,
            "executions": self._exec_count,
            "failures":   self._fail_count,
            "success_rate": (self._exec_count - self._fail_count) / self._exec_count
                            if self._exec_count else 1.0
        }

class SkillValidationError(Exception):
    pass
```

### 2.2 Gerçek Dünya Örneği: Veri Analizi Skill'i

```python
import json

class DataAnalysisSkill(Skill):
    """
    Bir veritabanı tablosunu analiz eder:
    - Özet istatistikler çıkarır (satır sayısı, NULL oranı, vb.)
    - Korelasyon ve dağılım analizi yapar
    - Sonuçları markdown rapor olarak yazar
    """

    VALID_ANALYSIS_TYPES = {"summary", "correlation", "distribution", "full"}

    def __init__(self, tools: dict):
        super().__init__(
            metadata = SkillMetadata(
                name           = "analyze_data",
                version        = "1.2.0",
                description    = (
                    "Veritabanı tablosunu analiz eder, istatistiksel özet çıkarır "
                    "ve markdown rapor oluşturur. Veri kalitesi değerlendirmesi de içerir."
                ),
                category       = "veri_analizi",
                required_tools = ["execute_sql", "write_file"],
                tags           = ["analitik", "raporlama", "istatistik"],
                status         = SkillStatus.STABLE
            ),
            tools = tools
        )

    def _validate_context(self, context: dict):
        if "table" not in context:
            raise SkillValidationError("'table' parametresi zorunludur.")
        analysis_type = context.get("analysis_type", "summary")
        if analysis_type not in self.VALID_ANALYSIS_TYPES:
            raise SkillValidationError(
                f"Geçersiz analiz tipi: '{analysis_type}'. "
                f"Geçerli değerler: {self.VALID_ANALYSIS_TYPES}"
            )

    def _run(self, context: dict) -> tuple:
        table         = context["table"]
        analysis_type = context.get("analysis_type", "summary")
        output_path   = context.get("output_path", f"reports/{table}_analysis.md")
        steps         = 0

        sections = []

        # 1. Özet her zaman dahil
        summary = self._get_summary(table)
        sections.append(f"## Özet İstatistikler\n\n{summary}")
        steps += 1

        # 2. Ek analizler
        if analysis_type in {"correlation", "full"}:
            corr = self._get_correlation(table)
            sections.append(f"## Korelasyon Analizi\n\n{corr}")
            steps += 1

        if analysis_type in {"distribution", "full"}:
            dist = self._get_distribution(table)
            sections.append(f"## Dağılım Analizi\n\n{dist}")
            steps += 1

        # 3. Raporu yaz
        report = self._build_report(table, sections)
        write_result = self._tool("write_file").execute(path=output_path, content=report)
        steps += 1

        if not write_result.get("success"):
            raise RuntimeError(f"Rapor yazılamadı: {write_result.get('error')}")

        return {
            "report_path": output_path,
            "table":       table,
            "analysis":    analysis_type,
            "sections":    len(sections)
        }, steps

    def _get_summary(self, table: str) -> str:
        result = self._tool("execute_sql").execute(
            query  = f"SELECT COUNT(*) as total FROM {table}",
            params = []
        )
        if not result.get("success"):
            return f"Sorgu hatası: {result.get('error')}"
        total = result["result"]["rows"][0].get("total", 0)
        return f"- Toplam satır sayısı: **{total:,}**"

    def _get_correlation(self, table: str) -> str:
        # Basitleştirilmiş örnek
        return "_Korelasyon analizi hesaplanıyor..._"

    def _get_distribution(self, table: str) -> str:
        return "_Dağılım analizi hesaplanıyor..._"

    def _build_report(self, table: str, sections: List[str]) -> str:
        from datetime import datetime
        header = (
            f"# Veri Analiz Raporu: `{table}`\n\n"
            f"**Oluşturulma tarihi:** {datetime.now().strftime('%Y-%m-%d %H:%M')}\n\n"
            f"**Skill sürümü:** {self.version}\n\n---\n\n"
        )
        return header + "\n\n".join(sections)
```

---

## 3. Skill Discovery

### 3.1 Statik Discovery (SkillRegistry)

```python
from typing import Dict, List, Optional

class SkillRegistry:
    """Merkezi skill deposu. Singleton pattern ile kullanılır."""

    _instance: Optional["SkillRegistry"] = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._skills: Dict[str, Skill] = {}
        return cls._instance

    def register(self, skill: Skill):
        if skill.metadata.status == SkillStatus.DEPRECATED:
            logger.warning(f"DEPRECATED skill kaydediliyor: {skill.name}")
        self._skills[skill.name] = skill
        logger.info(f"Skill kaydedildi: {skill.name} v{skill.version}")

    def get(self, name: str) -> Optional[Skill]:
        return self._skills.get(name)

    def list_by_category(self, category: str) -> List[Skill]:
        return [s for s in self._skills.values() if s.category == category]

    def list_by_tag(self, tag: str) -> List[Skill]:
        return [s for s in self._skills.values() if tag in s.metadata.tags]

    def list_stable(self) -> List[Skill]:
        return [s for s in self._skills.values()
                if s.metadata.status == SkillStatus.STABLE]

    def all(self) -> List[Skill]:
        return list(self._skills.values())

    def unregister(self, name: str):
        self._skills.pop(name, None)

    def summary(self) -> dict:
        by_status = {}
        for skill in self._skills.values():
            st = skill.metadata.status.value
            by_status[st] = by_status.get(st, 0) + 1
        return {
            "total":    len(self._skills),
            "by_status": by_status,
            "categories": list({s.category for s in self._skills.values()})
        }
```

### 3.2 Semantik Discovery

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticSkillDiscovery:
    """
    Görev açıklamasına göre en uygun skill'i semantik benzerlikle bulur.
    Türkçe görev tanımlarını destekler.
    """

    def __init__(
        self,
        registry: SkillRegistry,
        model_name: str = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
    ):
        self.registry = registry
        self.model    = SentenceTransformer(model_name)
        self._rebuild_index()

    def _rebuild_index(self):
        """Tüm skill açıklamalarını vektörize et."""
        skills = self.registry.list_stable()
        if not skills:
            self._skills     = []
            self._embeddings = np.array([])
            return
        self._skills     = skills
        descriptions     = [f"{s.name}: {s.description}" for s in skills]
        self._embeddings = self.model.encode(descriptions, normalize_embeddings=True)

    def discover(
        self,
        task: str,
        top_k: int = 3,
        threshold: float = 0.35,
        category_filter: str = None
    ) -> List[Dict]:
        """
        Görev için en uygun skill'leri döndürür.
        Her sonuç: {skill, score, reason}
        """
        if not self._skills:
            return []

        q_emb  = self.model.encode([task], normalize_embeddings=True)[0]
        scores = self._embeddings @ q_emb

        results = []
        for i, (skill, score) in enumerate(zip(self._skills, scores)):
            if score < threshold:
                continue
            if category_filter and skill.category != category_filter:
                continue
            results.append({"skill": skill, "score": float(score)})

        results.sort(key=lambda x: x["score"], reverse=True)
        return results[:top_k]

    def explain(self, task: str, top_k: int = 3) -> str:
        """Seçim gerekçesini insan okunabilir formatta açıklar."""
        results = self.discover(task, top_k=top_k, threshold=0.0)
        lines   = [f"Görev: '{task}'\n"]
        for i, r in enumerate(results, 1):
            s = r["skill"]
            lines.append(
                f"{i}. **{s.name}** (skor: {r['score']:.3f})\n"
                f"   Kategori: {s.category} | Sürüm: {s.version}\n"
                f"   → {s.description}"
            )
        return "\n".join(lines)
```

### 3.3 Dinamik Discovery (Plugin Sistemi)

```python
import importlib.util
import inspect
import os

class PluginSkillLoader:
    """
    Belirli bir klasördeki Python dosyalarından skill'leri dinamik olarak yükler.
    Yeni skill eklemek için agent'ı yeniden başlatmaya gerek kalmaz.
    """

    def __init__(self, skill_dir: str, tools: dict):
        self.skill_dir = skill_dir
        self.tools     = tools
        self._loaded: Dict[str, str] = {}   # {skill_name: dosya_yolu}

    def load_all(self, registry: SkillRegistry) -> int:
        """Klasördeki tüm *_skill.py dosyalarını yükler."""
        count = 0
        for filename in os.listdir(self.skill_dir):
            if not filename.endswith("_skill.py"):
                continue
            filepath = os.path.join(self.skill_dir, filename)
            try:
                loaded = self._load_file(filepath, registry)
                count += loaded
            except Exception as e:
                logger.error(f"Skill yüklenemedi ({filename}): {e}")
        return count

    def _load_file(self, filepath: str, registry: SkillRegistry) -> int:
        module_name = os.path.basename(filepath)[:-3]
        spec        = importlib.util.spec_from_file_location(module_name, filepath)
        module      = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)

        count = 0
        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            if (inspect.isclass(attr)
                    and issubclass(attr, Skill)
                    and attr is not Skill):
                try:
                    instance = attr(self.tools)
                    registry.register(instance)
                    self._loaded[instance.name] = filepath
                    count += 1
                except Exception as e:
                    logger.warning(f"Skill instantiate edilemedi ({attr_name}): {e}")
        return count

    def reload(self, skill_name: str, registry: SkillRegistry) -> bool:
        """Tek bir skill'i yeniden yükler."""
        filepath = self._loaded.get(skill_name)
        if not filepath:
            return False
        registry.unregister(skill_name)
        return self._load_file(filepath, registry) > 0
```

---

## 4. Skill Composition

### 4.1 Sequential Pipeline

```python
class SequentialPipeline(Skill):
    """
    Skill'leri sırayla çalıştırır; her adımın çıktısı
    bir sonraki adımın girdisine eklenir.
    """

    def __init__(self, name: str, description: str, steps: List[Skill]):
        super().__init__(
            metadata = SkillMetadata(
                name           = name,
                version        = "1.0.0",
                description    = description,
                category       = "composition",
                required_tools = []
            ),
            tools = {}
        )
        self.steps = steps

    def _run(self, context: dict) -> tuple:
        current_context = dict(context)
        step_count      = 0

        for skill in self.steps:
            result = skill.run(current_context)
            step_count += result.steps_taken

            if not result.success:
                raise RuntimeError(
                    f"Pipeline adımı başarısız [{skill.name}]: {result.error}"
                )

            # Çıktıyı bir sonraki adıma aktar
            if isinstance(result.output, dict):
                current_context.update(result.output)
            else:
                current_context[f"{skill.name}_output"] = result.output

        return current_context, step_count

# Kullanım
pipeline = SequentialPipeline(
    name        = "sales_report_pipeline",
    description = "Satış verilerini analiz eder ve rapor oluşturur.",
    steps       = [
        DataFetchSkill(tools),
        DataCleanSkill(tools),
        DataAnalysisSkill(tools),
        ReportGeneratorSkill(tools)
    ]
)
```

### 4.2 Paralel Fanout

```python
import asyncio

class ParallelFanout(Skill):
    """
    Birden fazla skill'i paralel çalıştırır,
    tüm sonuçları toplar.
    Bağımsız görevlerde latency'yi önemli ölçüde düşürür.
    """

    def __init__(self, name: str, description: str, branches: List[Skill],
                 fail_fast: bool = False):
        super().__init__(
            metadata = SkillMetadata(
                name           = name,
                version        = "1.0.0",
                description    = description,
                category       = "composition",
                required_tools = []
            ),
            tools = {}
        )
        self.branches  = branches
        self.fail_fast = fail_fast

    def _run(self, context: dict) -> tuple:
        results = asyncio.run(self._run_parallel(context))

        failures = [r for r in results if not r.success]
        if failures and self.fail_fast:
            raise RuntimeError(
                f"Paralel skill başarısız: {[f.error for f in failures]}"
            )

        total_steps = sum(r.steps_taken for r in results)
        outputs     = {r.skill_name: r.output for r in results}
        return {"branch_results": outputs, "failures": len(failures)}, total_steps

    async def _run_parallel(self, context: dict) -> List[SkillResult]:
        tasks = [asyncio.to_thread(skill.run, context) for skill in self.branches]
        return await asyncio.gather(*tasks, return_exceptions=False)
```

### 4.3 Koşullu Dallanma

```python
from typing import Callable

class ConditionalSkill(Skill):
    """
    Bağlama göre farklı skill dalını seçerek çalıştırır.
    """

    def __init__(
        self,
        name: str,
        description: str,
        condition: Callable[[dict], bool],
        if_true:  Skill,
        if_false: Skill
    ):
        super().__init__(
            metadata = SkillMetadata(
                name           = name,
                version        = "1.0.0",
                description    = description,
                category       = "composition",
                required_tools = []
            ),
            tools = {}
        )
        self.condition = condition
        self.if_true   = if_true
        self.if_false  = if_false

    def _run(self, context: dict) -> tuple:
        chosen = self.if_true if self.condition(context) else self.if_false
        logger.debug(f"[{self.name}] Seçilen dal: {chosen.name}")
        result = chosen.run(context)
        if not result.success:
            raise RuntimeError(result.error)
        return result.output, result.steps_taken

# Örnek kullanım
def is_large_dataset(ctx: dict) -> bool:
    return ctx.get("row_count", 0) > 100_000

analysis = ConditionalSkill(
    name        = "smart_analysis",
    description = "Veri boyutuna göre hızlı veya derin analiz seçer.",
    condition   = is_large_dataset,
    if_true     = SampledAnalysisSkill(tools),     # Büyük veri → örnekleme
    if_false    = FullAnalysisSkill(tools)          # Küçük veri → tam analiz
)
```

### 4.4 Döngüsel Skill (Map-Reduce)

```python
class MapReduceSkill(Skill):
    """
    Büyük koleksiyonu parçalara böler (map), her parçayı paralel işler,
    sonuçları birleştirir (reduce).
    """

    def __init__(
        self,
        name: str,
        description: str,
        map_skill:    Skill,
        reduce_fn:    Callable[[List], Any],
        chunk_size:   int = 100
    ):
        super().__init__(
            metadata = SkillMetadata(
                name           = name,
                version        = "1.0.0",
                description    = description,
                category       = "composition",
                required_tools = []
            ),
            tools = {}
        )
        self.map_skill  = map_skill
        self.reduce_fn  = reduce_fn
        self.chunk_size = chunk_size

    def _run(self, context: dict) -> tuple:
        items   = context.get("items", [])
        chunks  = [items[i:i+self.chunk_size] for i in range(0, len(items), self.chunk_size)]

        # Her chunk'ı paralel işle
        chunk_results = asyncio.run(self._map(chunks, context))

        # Sonuçları birleştir
        outputs     = [r.output for r in chunk_results if r.success]
        total_steps = sum(r.steps_taken for r in chunk_results)
        reduced     = self.reduce_fn(outputs)

        return reduced, total_steps

    async def _map(self, chunks: List[list], base_context: dict) -> List[SkillResult]:
        tasks = [
            asyncio.to_thread(
                self.map_skill.run,
                {**base_context, "items": chunk, "chunk_index": i}
            )
            for i, chunk in enumerate(chunks)
        ]
        return await asyncio.gather(*tasks)
```

---

## 5. Skill Versiyonlama

### 5.1 Semantic Versioning

Skill'ler `major.minor.patch` formatında versiyonlanır:

| Değişiklik | Versiyon etkisi | Örnek |
|-----------|----------------|-------|
| Geriye dönük uyumsuz değişiklik | `major` artar | `1.x.x → 2.0.0` |
| Geriye dönük uyumlu yeni özellik | `minor` artar | `1.2.x → 1.3.0` |
| Hata düzeltmesi | `patch` artar | `1.2.3 → 1.2.4` |

### 5.2 Çok Sürüm Yönetimi

```python
class VersionedSkillRegistry:
    """
    Aynı skill'in birden fazla sürümünü tutar.
    Eski sürümü kullanan agent'ları kırmadan yeni sürüm sunulabilir.
    """

    def __init__(self):
        # {skill_name: {version_str: Skill}}
        self._versions: Dict[str, Dict[str, Skill]] = {}

    def register(self, skill: Skill):
        name    = skill.name
        version = skill.version
        if name not in self._versions:
            self._versions[name] = {}
        self._versions[name][version] = skill
        logger.info(f"Kaydedildi: {name} v{version}")

    def get(self, name: str, version: str = "latest") -> Optional[Skill]:
        if name not in self._versions:
            return None
        versions = self._versions[name]
        if version == "latest":
            return versions[max(versions.keys(), key=self._parse_semver)]
        return versions.get(version)

    def list_versions(self, name: str) -> List[str]:
        return sorted(
            self._versions.get(name, {}).keys(),
            key=self._parse_semver
        )

    def deprecate(self, name: str, version: str):
        skill = self.get(name, version)
        if skill:
            skill.metadata.status = SkillStatus.DEPRECATED

    @staticmethod
    def _parse_semver(version: str) -> tuple:
        try:
            parts = version.split(".")
            return tuple(int(p) for p in parts)
        except ValueError:
            return (0, 0, 0)
```

---

## 6. Skill Evaluation

### 6.1 Evaluation Suite

```python
import statistics
import time
from dataclasses import dataclass, field

@dataclass
class TestCase:
    name:            str
    input_context:   dict
    expected_output: Any
    tags:            List[str] = field(default_factory=list)
    max_duration_ms: int = 5000

@dataclass
class EvalReport:
    skill_name:      str
    total:           int
    passed:          int
    failed:          int
    errors:          int
    avg_duration_ms: float
    p95_duration_ms: float
    failure_details: List[dict] = field(default_factory=list)

    @property
    def pass_rate(self) -> float:
        return self.passed / self.total if self.total else 0.0

    def __str__(self) -> str:
        return (
            f"=== {self.skill_name} Eval ===\n"
            f"Toplam: {self.total} | Başarılı: {self.passed} | "
            f"Başarısız: {self.failed} | Hata: {self.errors}\n"
            f"Başarı oranı: %{self.pass_rate*100:.1f}\n"
            f"Ort. süre: {self.avg_duration_ms:.0f}ms | "
            f"P95: {self.p95_duration_ms:.0f}ms"
        )

class SkillEvaluator:
    """Skill'leri test case'lere göre değerlendirir."""

    def __init__(self, comparator: Callable = None):
        # comparator(actual, expected) → bool
        self.comparator = comparator or self._default_compare

    def evaluate(self, skill: Skill, test_cases: List[TestCase]) -> EvalReport:
        passed, failed, errors = 0, 0, 0
        durations    = []
        failure_details = []

        for tc in test_cases:
            start  = time.time()
            result = skill.run(tc.input_context)
            dur_ms = int((time.time() - start) * 1000)
            durations.append(dur_ms)

            if not result.success:
                errors += 1
                failure_details.append({
                    "test":   tc.name,
                    "reason": "execution_error",
                    "error":  result.error
                })
                continue

            if dur_ms > tc.max_duration_ms:
                failed += 1
                failure_details.append({
                    "test":   tc.name,
                    "reason": "timeout",
                    "dur_ms": dur_ms,
                    "limit":  tc.max_duration_ms
                })
                continue

            if self.comparator(result.output, tc.expected_output):
                passed += 1
            else:
                failed += 1
                failure_details.append({
                    "test":     tc.name,
                    "reason":   "output_mismatch",
                    "actual":   result.output,
                    "expected": tc.expected_output
                })

        return EvalReport(
            skill_name      = skill.name,
            total           = len(test_cases),
            passed          = passed,
            failed          = failed,
            errors          = errors,
            avg_duration_ms = statistics.mean(durations) if durations else 0,
            p95_duration_ms = statistics.quantiles(durations, n=20)[18]
                              if len(durations) >= 2 else (durations[0] if durations else 0),
            failure_details = failure_details
        )

    @staticmethod
    def _default_compare(actual, expected) -> bool:
        if isinstance(expected, dict) and isinstance(actual, dict):
            return all(actual.get(k) == v for k, v in expected.items())
        return actual == expected
```

### 6.2 Regression Test Suite

```python
class RegressionTestSuite:
    """
    Skill güncellemelerinde önceki sürümle kıyaslamalı test."""

    def __init__(self, evaluator: SkillEvaluator, test_cases: List[TestCase]):
        self.evaluator  = evaluator
        self.test_cases = test_cases

    def compare_versions(
        self,
        old_skill: Skill,
        new_skill: Skill
    ) -> dict:
        old_report = self.evaluator.evaluate(old_skill, self.test_cases)
        new_report = self.evaluator.evaluate(new_skill, self.test_cases)

        return {
            "old_version":      old_skill.version,
            "new_version":      new_skill.version,
            "pass_rate_delta":  new_report.pass_rate - old_report.pass_rate,
            "latency_delta_ms": new_report.avg_duration_ms - old_report.avg_duration_ms,
            "regression":       new_report.pass_rate < old_report.pass_rate,
            "old_report":       old_report,
            "new_report":       new_report
        }
```

---

## 7. Skill Registry

### 7.1 Dağıtık Registry (Redis Tabanlı)

```python
import redis
import json

class DistributedSkillRegistry:
    """
    Redis tabanlı dağıtık skill deposu.
    Çok sayıda agent node'u aynı skill havuzunu paylaşabilir.
    """

    KEY_PREFIX  = "skill:"
    INDEX_KEY   = "skill:index"

    def __init__(self, redis_url: str):
        self.redis = redis.from_url(redis_url, decode_responses=True)

    def register(self, skill: Skill):
        meta = {
            "name":          skill.name,
            "version":       skill.version,
            "description":   skill.description,
            "category":      skill.category,
            "required_tools": skill.metadata.required_tools,
            "tags":          skill.metadata.tags,
            "status":        skill.metadata.status.value
        }
        key = f"{self.KEY_PREFIX}{skill.name}"
        self.redis.set(key, json.dumps(meta, ensure_ascii=False))
        self.redis.sadd(self.INDEX_KEY, skill.name)

    def get_metadata(self, name: str) -> Optional[dict]:
        raw = self.redis.get(f"{self.KEY_PREFIX}{name}")
        return json.loads(raw) if raw else None

    def search(self, category: str = None, tag: str = None) -> List[dict]:
        names = self.redis.smembers(self.INDEX_KEY)
        results = []
        for name in names:
            meta = self.get_metadata(name)
            if not meta:
                continue
            if category and meta.get("category") != category:
                continue
            if tag and tag not in meta.get("tags", []):
                continue
            results.append(meta)
        return results

    def unregister(self, name: str):
        self.redis.delete(f"{self.KEY_PREFIX}{name}")
        self.redis.srem(self.INDEX_KEY, name)
```

---

## 8. Türkçe Skill İçin Pratik Notlar

> **Açıklamalar:** Skill açıklamalarını Türkçe yazın. Semantik discovery doğruluğu, görev dili ile skill açıklama dili eşleştiğinde artar.

> **Category İsimleri:** Snake case Türkçe kullanın: `veri_analizi`, `dosya_yönetimi`, `rapor_üretimi`.

> **Embedding Modeli:** Türkçe discovery için `paraphrase-multilingual-MiniLM-L12-v2` veya `intfloat/multilingual-e5-base` tercih edin.

> **Hata Mesajları:** `SkillValidationError` mesajlarını Türkçe yazın; kullanıcı hataların nedenini anlamalı.

> **Test Case İsimleri:** Test case `name` alanlarını Türkçe yazın; `EvalReport` çıktısı okunabilir olur.

> **Versiyon Politikası:** Skill'e Türkçe karakter desteği eklemek `minor` versiyonu artırır; mevcut Türkçe desteği kırmak `major` versiyonu artırır.

---

## Ekler

### A — Terimler Sözlüğü

| Terim | Açıklama |
|---|---|
| **Skill** | Bir veya daha fazla tool'u orkestre eden yüksek seviyeli yetenek |
| **Skill Discovery** | Göreve en uygun skill'i bulma süreci |
| **Skill Composition** | Birden fazla skill'i birleştirme pattern'i |
| **SkillRegistry** | Skill'lerin merkezi kayıt sistemi |
| **Semantic Versioning** | major.minor.patch formatında sürüm numaralandırma |
| **Map-Reduce** | Büyük koleksiyonu bölerek paralel işleme stratejisi |
| **EvalReport** | Skill değerlendirme sonuç raporu |
| **Regression** | Yeni versiyonun eski versiyona göre gerileme göstermesi |

### B — Mini Checklist

#### Skill Tasarımı

- [ ] `SkillMetadata` eksiksiz dolduruldu (version, tags, status)
- [ ] `_validate_context` zorunlu alanları kontrol ediyor
- [ ] Hata mesajları Türkçe ve anlamlı
- [ ] `required_tools` listesi doğru
- [ ] `is_read_only` gerekirse `True` olarak işaretlendi

#### Skill Discovery

- [ ] Türkçe uyumlu embedding modeli seçildi
- [ ] `threshold` değeri test edildi (0.35 başlangıç için uygundur)
- [ ] Dynamic loader için dosya isimlendirmesi `*_skill.py`
- [ ] Registry'de duplicate kontrol var

#### Skill Composition

- [ ] Pipeline adımları arası veri formatı belgelendi
- [ ] `fail_fast` politikası açıkça belirtildi
- [ ] Paralel fanout'ta bağımsız branch'ler kontrol edildi

#### Evaluation

- [ ] Her kritik path için test case var
- [ ] Hata senaryoları (eksik girdi, büyük veri) test edildi
- [ ] `max_duration_ms` makul bir değere ayarlandı
- [ ] Regression test çalıştırılmadan major güncelleme yapılmadı

---

**Serinin başına dön:** [README](./README.md)  
**Tool use rehberi:** [02 - Tool Use ve Entegrasyon](./02_-_Tool_Use_ve_Entegrasyon.md)  
**Çalışma alanı rehberi:** [04 - Çalışma Alanı Yönetimi](./04_-_Calisma_Alani_Yonetimi.md)
