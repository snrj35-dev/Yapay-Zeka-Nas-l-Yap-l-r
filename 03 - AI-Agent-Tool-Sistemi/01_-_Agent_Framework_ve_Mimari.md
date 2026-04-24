# 01 — Agent Framework ve Mimari

Bu rehber, production-grade AI agent sistemlerinin mimarisini, agent tiplerini, lifecycle yönetimini, multi-agent koordinasyonunu ve memory sistemlerini kapsar.

**Sonraki:** [02 - Tool Use ve Entegrasyon](./02_-_Tool_Use_ve_Entegrasyon.md)

---

## 📌 Doküman Kartı

| Alan | Değer |
|---|---|
| Rol | Agent mimarisi ve koordinasyon referansı |
| Durum | Living specification (`v2.0`) |
| Son güncelleme | 2026-04-24 |
| Birincil okur | AI mühendisleri, agent framework geliştiricileri |
| Ana girdi | Agent config, task definition, environment state |
| Ana çıktı | Agent behavior, coordination protocol, memory schema |
| Bağımlı dokümanlar | [02 - Tool Use](./02_-_Tool_Use_ve_Entegrasyon.md), [03 - Skills](./03_-_Skills_ve_Capabilities.md), [04 - Workspace](./04_-_Calisma_Alani_Yonetimi.md) |

> **Kalite notu:** Kod blokları referans implementasyon örnekleridir. Production'a almadan önce kendi ortamınızda test edin ve hata yönetimi ekleyin.

---

## İçindekiler

1. [Agent Nedir?](#1-agent-nedir)
2. [Agent Tipleri](#2-agent-tipleri)
3. [Agent Lifecycle](#3-agent-lifecycle)
4. [Memory Sistemleri](#4-memory-sistemleri)
5. [Multi-Agent Coordination](#5-multi-agent-coordination)
6. [Hata Yönetimi ve Dayanıklılık](#6-hata-yönetimi-ve-dayanıklılık)
7. [Gözlemlenebilirlik (Observability)](#7-gözlemlenebilirlik-observability)
8. [Agent Framework'ları](#8-agent-frameworkları)
9. [Türkçe Agent İçin Pratik Notlar](#9-türkçe-agent-için-pratik-notlar)
10. [Ekler](#ekler)

---

## 1. Agent Nedir?

### Tanım

AI agent, hedefe yönelik eylemler gerçekleştiren, araçları (tools) kullanabilen, çevresiyle etkileşime giren ve otonom kararlar veren bir sistemdir. LLM'i beyni olarak kullanan agent'lar; algılama, akıl yürütme ve eylem döngüsünü sürekli tekrar eder.

### Temel Bileşenler

| Bileşen | Açıklama | Sorumluluğu |
|---------|----------|-------------|
| **Reasoning Engine** | LLM tabanlı karar ve planlama modülü | Ne yapılacağına karar verir |
| **Tool Interface** | Dış sistemlerle etkileşim katmanı | Kararları eyleme dönüştürür |
| **Memory** | Kısa ve uzun vadeli bilgi deposu | Geçmişi hatırlar, bağlamı korur |
| **Perception** | Çevreden gelen girdileri işler | Durumu anlar |
| **Action** | Eylemleri çalıştırır ve sonuçları geri bildirir | Dünyayı etkiler |
| **Orchestrator** | Bileşenler arası koordinasyon | Döngüyü yönetir |

### Agent ile Chatbot Arasındaki Fark

```
Chatbot:   Kullanıcı → LLM → Yanıt          (tek adım, pasif)
Agent:     Kullanıcı → LLM → Tool → LLM → Tool → ... → Yanıt
                                 ↑__________________________|
                                 (döngü, proaktif, çok adımlı)
```

---

## 2. Agent Tipleri

### 2.1 ReAct Agent (Reasoning + Acting)

En yaygın pattern. Model, düşüncesini yazıya döküp ardından eylem seçer; gözlem sonucuna göre tekrar akıl yürütür.

**Akış:**
```
Düşün → Eyle → Gözlemle → Düşün → Eyle → ...
```

**Zayıflıkları:**
- Her adımda LLM çağrısı gerekir (yüksek latency)
- Uzun zincirde bağlam dağılabilir
- Araç çıktısı gürültülüyse döngüye girebilir

**Örnek Implementasyon:**

```python
from typing import Optional
import json

class ReActAgent:
    """
    Thought → Action → Observation döngüsünü uygulayan temel agent.
    """

    MAX_STEPS = 15  # Sonsuz döngü koruması

    def __init__(self, llm, tools: dict, memory):
        self.llm = llm
        self.tools = tools           # {tool_name: Tool nesnesi}
        self.memory = memory
        self.trajectory = []         # Adım geçmişi (debug için)

    def run(self, task: str) -> str:
        context = self.memory.retrieve(task)
        messages = self._build_initial_messages(task, context)

        for step in range(self.MAX_STEPS):
            response = self.llm.complete(messages)
            parsed = self._parse_response(response)

            if parsed["type"] == "final_answer":
                self.memory.store(task, parsed["content"])
                return parsed["content"]

            if parsed["type"] == "action":
                tool_name = parsed["tool"]
                tool_args = parsed["args"]

                tool = self.tools.get(tool_name)
                if not tool:
                    observation = f"[HATA] '{tool_name}' adlı araç bulunamadı."
                else:
                    try:
                        observation = tool.execute(**tool_args)
                    except Exception as e:
                        observation = f"[HATA] Araç çalıştırılamadı: {e}"

                self.trajectory.append({
                    "step": step,
                    "thought": parsed.get("thought"),
                    "action": tool_name,
                    "args": tool_args,
                    "observation": observation
                })

                messages.append({"role": "assistant", "content": response})
                messages.append({"role": "user", "content": f"Gözlem: {observation}"})

        return "[HATA] Maksimum adım sayısına ulaşıldı."

    def _build_initial_messages(self, task: str, context: str) -> list:
        system = (
            "Sen bir AI asistanısın. Her adımda şu yapıyı kullan:\n"
            "Düşünce: <akıl yürütme>\n"
            "Eylem: <araç adı>\n"
            "Eylem Girdisi: <JSON args>\n"
            "Ya da:\n"
            "Son Cevap: <kullanıcıya yanıt>\n\n"
            f"Mevcut araçlar: {list(self.tools.keys())}"
        )
        return [
            {"role": "system", "content": system},
            {"role": "user", "content": f"Bağlam:\n{context}\n\nGörev: {task}"}
        ]

    def _parse_response(self, response: str) -> dict:
        if "Son Cevap:" in response:
            return {"type": "final_answer", "content": response.split("Son Cevap:")[-1].strip()}
        if "Eylem:" in response:
            lines = {l.split(":")[0].strip(): ":".join(l.split(":")[1:]).strip()
                     for l in response.split("\n") if ":" in l}
            try:
                args = json.loads(lines.get("Eylem Girdisi", "{}"))
            except json.JSONDecodeError:
                args = {}
            return {
                "type": "action",
                "thought": lines.get("Düşünce", ""),
                "tool": lines.get("Eylem", ""),
                "args": args
            }
        return {"type": "final_answer", "content": response}
```

---

### 2.2 Plan-and-Solve Agent

Görevi önce alt görevlere böler, ardından sırayla çözer. Uzun, bağımlılıklı iş akışları için uygundur.

**Akış:**
```
Analiz et → Plan oluştur → [Alt görev 1 → Alt görev 2 → ...] → Birleştir
```

**Ne zaman tercih edilir:**
- Görev net alt adımlara bölünebiliyorsa
- Her adım bir öncekinin çıktısına bağlıysa
- Uzun süreli işlemler gerekmiyorsa

```python
from dataclasses import dataclass, field
from typing import List
from enum import Enum

class TaskStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    DONE = "done"
    FAILED = "failed"

@dataclass
class SubTask:
    id: str
    description: str
    depends_on: List[str] = field(default_factory=list)
    status: TaskStatus = TaskStatus.PENDING
    result: Optional[str] = None

class PlanAndSolveAgent:
    """
    Görevi önce planlar, sonra dependency sırasına göre çözer.
    """

    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools

    def run(self, task: str) -> str:
        # 1. Planlama
        plan = self._create_plan(task)
        print(f"[Plan] {len(plan)} alt görev oluşturuldu.")

        # 2. Uygulama (topological sort ile bağımlılık sırası)
        execution_order = self._topological_sort(plan)
        results = {}

        for subtask_id in execution_order:
            subtask = plan[subtask_id]
            subtask.status = TaskStatus.IN_PROGRESS

            # Bağımlı sonuçları bağlama ekle
            dep_context = {dep: results[dep] for dep in subtask.depends_on if dep in results}

            result = self._solve_subtask(subtask, dep_context)
            subtask.result = result
            subtask.status = TaskStatus.DONE
            results[subtask_id] = result

        # 3. Sentez
        return self._synthesize(task, results)

    def _create_plan(self, task: str) -> dict[str, SubTask]:
        prompt = (
            f"Görev: {task}\n\n"
            "Bu görevi JSON dizisi olarak alt görevlere böl. "
            "Her eleman: {id, description, depends_on: [id_listesi]}\n"
            "Sadece JSON döndür, başka hiçbir şey ekleme."
        )
        raw = self.llm.complete([{"role": "user", "content": prompt}])
        try:
            items = json.loads(raw)
        except json.JSONDecodeError:
            items = [{"id": "t1", "description": task, "depends_on": []}]

        return {
            item["id"]: SubTask(
                id=item["id"],
                description=item["description"],
                depends_on=item.get("depends_on", [])
            )
            for item in items
        }

    def _topological_sort(self, plan: dict) -> List[str]:
        visited, order = set(), []

        def visit(node_id):
            if node_id in visited:
                return
            visited.add(node_id)
            for dep in plan[node_id].depends_on:
                if dep in plan:
                    visit(dep)
            order.append(node_id)

        for node_id in plan:
            visit(node_id)
        return order

    def _solve_subtask(self, subtask: SubTask, context: dict) -> str:
        context_str = "\n".join(f"[{k}]: {v}" for k, v in context.items())
        prompt = (
            f"Alt Görev: {subtask.description}\n\n"
            f"Tamamlanan adımların sonuçları:\n{context_str}\n\n"
            "Bu alt görevi çöz:"
        )
        return self.llm.complete([{"role": "user", "content": prompt}])

    def _synthesize(self, original_task: str, results: dict) -> str:
        results_str = "\n".join(f"[{k}]: {v}" for k, v in results.items())
        prompt = (
            f"Orijinal görev: {original_task}\n\n"
            f"Alt görev sonuçları:\n{results_str}\n\n"
            "Bu sonuçları tutarlı bir yanıtta birleştir:"
        )
        return self.llm.complete([{"role": "user", "content": prompt}])
```

---

### 2.3 ReWOO Agent (Reasoning Without Observation)

Gözlem beklemeden tüm planı tek seferde oluşturur, ardından plan adımlarını paralel çalıştırabilir. Latency'yi önemli ölçüde azaltır.

**Akış:**
```
Tüm planı oluştur → Adımları (paralel) çalıştır → Sonuçları bir araya getir
```

**Ne zaman tercih edilir:**
- Hız kritik ve adımlar birbirinden bağımsızsa
- Tool çağrı sayısı öngörülebilirse

```python
import asyncio
from dataclasses import dataclass

@dataclass
class PlanStep:
    index: int
    thought: str
    tool: str
    args: dict
    result: Optional[str] = None

class ReWOOAgent:
    """
    Önce tüm planı üretir, ardından adımları paralel çalıştırır.
    """

    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools

    async def run(self, task: str) -> str:
        # 1. Planlama (tek LLM çağrısı)
        steps = self._plan(task)

        # 2. Paralel çalıştırma
        await asyncio.gather(*[self._execute_step(s) for s in steps])

        # 3. Sentez (tek LLM çağrısı)
        return self._solve(task, steps)

    def _plan(self, task: str) -> List[PlanStep]:
        prompt = (
            f"Görev: {task}\n\n"
            "Görevi çözmek için gereken tüm araç çağrılarını planla. "
            "Her adım için: {index, thought, tool, args}. "
            "Sadece JSON dizisi döndür."
        )
        raw = self.llm.complete([{"role": "user", "content": prompt}])
        try:
            items = json.loads(raw)
        except json.JSONDecodeError:
            return []
        return [PlanStep(**item) for item in items]

    async def _execute_step(self, step: PlanStep):
        tool = self.tools.get(step.tool)
        if not tool:
            step.result = f"[HATA] '{step.tool}' aracı bulunamadı."
            return
        try:
            step.result = await asyncio.to_thread(tool.execute, **step.args)
        except Exception as e:
            step.result = f"[HATA] {e}"

    def _solve(self, task: str, steps: List[PlanStep]) -> str:
        evidence = "\n".join(
            f"Adım {s.index} ({s.tool}): {s.result}" for s in steps
        )
        prompt = (
            f"Görev: {task}\n\nToplanan kanıtlar:\n{evidence}\n\n"
            "Bu kanıtları kullanarak nihai yanıtı oluştur:"
        )
        return self.llm.complete([{"role": "user", "content": prompt}])
```

---

### 2.4 Karşılaştırmalı Analiz

| Özellik | ReAct | Plan-and-Solve | ReWOO |
|---------|-------|----------------|-------|
| **LLM çağrı sayısı** | N adım × 1 | 2 + N | 2 sabit |
| **Paralel araç çalıştırma** | ❌ | ❌ (bağımlı sıra) | ✅ |
| **Adaptasyon** | Yüksek (her adımda gözlem) | Orta (plan sabit) | Düşük (plan önceden sabit) |
| **Öngörülebilirlik** | Düşük | Yüksek | Yüksek |
| **Latency** | Yüksek | Orta | Düşük |
| **İdeal senaryo** | Keşif gerektiren görevler | Bağımlı alt adımlar | Hız kritik, bağımsız araçlar |
| **Başarısızlık modu** | Döngüye girme | Kötü plan → tüm süreç başarısız | Yanlış plan sabitlendi |

---

## 3. Agent Lifecycle

### 3.1 Durum Makinesi

```
INITIALIZED → PLANNING → RUNNING → COMPLETED
                              ↓
                           PAUSED ←→ RUNNING
                              ↓
                           FAILED
```

```python
from enum import Enum, auto
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any

class AgentState(Enum):
    INITIALIZED = auto()
    PLANNING    = auto()
    RUNNING     = auto()
    PAUSED      = auto()
    COMPLETED   = auto()
    FAILED      = auto()

@dataclass
class AgentConfig:
    agent_id: str
    agent_type: str                      # react | plan_and_solve | rewoo
    max_steps: int = 20
    timeout_seconds: int = 300
    memory_config: dict = field(default_factory=dict)
    tool_config: dict = field(default_factory=dict)
    retry_on_failure: bool = True
    max_retries: int = 3

@dataclass
class Task:
    task_id: str
    description: str
    context: dict = field(default_factory=dict)
    deadline: Optional[datetime] = None
    completed: bool = False
    result: Optional[str] = None
    error: Optional[str] = None
    steps_taken: int = 0
    started_at: Optional[datetime] = None
    finished_at: Optional[datetime] = None

class AgentLifecycle:
    """
    Agent lifecycle yönetimi: initialization, execution, termination.
    """

    def __init__(self, config: AgentConfig, llm, tools: dict, memory):
        self.config  = config
        self.llm     = llm
        self.tools   = tools
        self.memory  = memory
        self.state   = AgentState.INITIALIZED
        self._start_time: Optional[datetime] = None

    async def run(self, task: Task) -> Task:
        self._start_time = datetime.now()
        task.started_at  = self._start_time

        try:
            self._transition(AgentState.PLANNING)
            plan = await self._plan(task)

            self._transition(AgentState.RUNNING)
            result = await self._execute(task, plan)

            task.completed   = True
            task.result      = result
            task.finished_at = datetime.now()
            self._transition(AgentState.COMPLETED)

        except TimeoutError:
            task.error = "Zaman aşımı (timeout)."
            self._transition(AgentState.FAILED)
        except Exception as e:
            task.error = str(e)
            self._transition(AgentState.FAILED)

        return task

    def pause(self):
        if self.state == AgentState.RUNNING:
            self._transition(AgentState.PAUSED)

    def resume(self):
        if self.state == AgentState.PAUSED:
            self._transition(AgentState.RUNNING)

    def _transition(self, new_state: AgentState):
        allowed = {
            AgentState.INITIALIZED: [AgentState.PLANNING],
            AgentState.PLANNING:    [AgentState.RUNNING, AgentState.FAILED],
            AgentState.RUNNING:     [AgentState.PAUSED, AgentState.COMPLETED, AgentState.FAILED],
            AgentState.PAUSED:      [AgentState.RUNNING, AgentState.FAILED],
            AgentState.COMPLETED:   [],
            AgentState.FAILED:      [],
        }
        if new_state not in allowed.get(self.state, []):
            raise ValueError(f"Geçersiz geçiş: {self.state} → {new_state}")
        self.state = new_state

    def _is_timed_out(self) -> bool:
        if not self._start_time:
            return False
        elapsed = (datetime.now() - self._start_time).total_seconds()
        return elapsed > self.config.timeout_seconds

    async def _plan(self, task: Task): ...   # Alt sınıf implement eder
    async def _execute(self, task: Task, plan): ...
```

---

## 4. Memory Sistemleri

### 4.1 Mimari Genel Bakış

```
┌──────────────────────────────────────┐
│           Memory System              │
│  ┌────────────┐  ┌────────────────┐  │
│  │ Short-term │  │   Long-term    │  │
│  │ (RAM/dict) │  │ (Vector DB)    │  │
│  └────────────┘  └────────────────┘  │
│  ┌──────────────────────────────┐    │
│  │     Episodic (SQLite/Pg)     │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

### 4.2 Short-term Memory

Mevcut oturum bağlamını tutar. Oturum sonunda silinir. Genellikle token limitine göre sıkıştırılır.

```python
from collections import deque
import tiktoken

class ShortTermMemory:
    """
    Token limitine göre otomatik sıkıştırma yapan kısa süreli bellek.
    """

    def __init__(self, max_tokens: int = 4000, model: str = "gpt-4"):
        self.max_tokens = max_tokens
        self.encoding   = tiktoken.encoding_for_model(model)
        self._buffer: deque = deque()   # (role, content) çiftleri

    def add(self, role: str, content: str):
        self._buffer.append({"role": role, "content": content})
        self._trim()

    def get_messages(self) -> list:
        return list(self._buffer)

    def clear(self):
        self._buffer.clear()

    def _count_tokens(self, text: str) -> int:
        return len(self.encoding.encode(text))

    def _trim(self):
        """En eski mesajları, toplam token limiti aşılana kadar sil."""
        while self._total_tokens() > self.max_tokens and len(self._buffer) > 1:
            self._buffer.popleft()

    def _total_tokens(self) -> int:
        return sum(self._count_tokens(m["content"]) for m in self._buffer)
```

### 4.3 Long-term Memory

Kalıcı bilgi deposu. Semantic search ile ilgili geçmişe erişim sağlar.

```python
from sentence_transformers import SentenceTransformer
import chromadb
from datetime import datetime
import uuid

class LongTermMemory:
    """
    ChromaDB tabanlı vektör belleği. Semantic retrieval destekler.
    """

    def __init__(
        self,
        collection_name: str = "agent_memory",
        model_name: str = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
    ):
        self.model      = SentenceTransformer(model_name)
        self.client     = chromadb.Client()
        self.collection = self.client.get_or_create_collection(collection_name)

    def store(self, content: str, metadata: dict = None):
        doc_id    = str(uuid.uuid4())
        embedding = self.model.encode([content])[0].tolist()
        meta      = {"timestamp": datetime.now().isoformat(), **(metadata or {})}

        self.collection.add(
            ids        = [doc_id],
            embeddings = [embedding],
            documents  = [content],
            metadatas  = [meta]
        )
        return doc_id

    def retrieve(self, query: str, k: int = 5, min_score: float = 0.6) -> list:
        query_embedding = self.model.encode([query])[0].tolist()
        results = self.collection.query(
            query_embeddings = [query_embedding],
            n_results        = k,
            include          = ["documents", "metadatas", "distances"]
        )
        # Düşük benzerliği filtrele
        filtered = []
        for doc, meta, dist in zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0]
        ):
            similarity = 1 - dist   # cosine distance → similarity
            if similarity >= min_score:
                filtered.append({"content": doc, "metadata": meta, "score": similarity})

        return sorted(filtered, key=lambda x: x["score"], reverse=True)

    def delete(self, doc_id: str):
        self.collection.delete(ids=[doc_id])

    def count(self) -> int:
        return self.collection.count()
```

### 4.4 Episodic Memory

Olay bazlı geçmiş kaydı. Hangi adımda ne yapıldığını ve sonucunu saklar; debug ve öğrenme için kritik.

```python
import sqlite3
import json
from datetime import datetime
from dataclasses import dataclass

@dataclass
class Episode:
    agent_id: str
    task_id:  str
    step:     int
    action:   str
    args:     dict
    result:   str
    success:  bool
    duration_ms: int
    timestamp: str = ""

    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now().isoformat()

class EpisodicMemory:
    """
    SQLite tabanlı episodik bellek. Sorgulanabilir geçmiş kaydı.
    """

    def __init__(self, db_path: str = "episodes.db"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._init_db()

    def _init_db(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS episodes (
                id           INTEGER PRIMARY KEY AUTOINCREMENT,
                agent_id     TEXT NOT NULL,
                task_id      TEXT NOT NULL,
                step         INTEGER,
                action       TEXT,
                args         TEXT,
                result       TEXT,
                success      INTEGER,
                duration_ms  INTEGER,
                timestamp    TEXT
            )
        """)
        self.conn.execute("CREATE INDEX IF NOT EXISTS idx_agent ON episodes(agent_id)")
        self.conn.execute("CREATE INDEX IF NOT EXISTS idx_task  ON episodes(task_id)")
        self.conn.commit()

    def record(self, episode: Episode):
        self.conn.execute("""
            INSERT INTO episodes
            (agent_id, task_id, step, action, args, result, success, duration_ms, timestamp)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            episode.agent_id, episode.task_id, episode.step,
            episode.action, json.dumps(episode.args, ensure_ascii=False),
            episode.result, int(episode.success), episode.duration_ms, episode.timestamp
        ))
        self.conn.commit()

    def get_by_task(self, task_id: str) -> list:
        rows = self.conn.execute(
            "SELECT * FROM episodes WHERE task_id = ? ORDER BY step", (task_id,)
        ).fetchall()
        return [self._row_to_dict(r) for r in rows]

    def get_successful_actions(self, action: str, limit: int = 10) -> list:
        """Belirli bir eylem için geçmişteki başarılı argümanları döndürür."""
        rows = self.conn.execute(
            "SELECT args FROM episodes WHERE action = ? AND success = 1 ORDER BY id DESC LIMIT ?",
            (action, limit)
        ).fetchall()
        return [json.loads(r[0]) for r in rows]

    def get_failure_patterns(self) -> dict:
        """Hangi action'ların en çok başarısız olduğunu döndürür."""
        rows = self.conn.execute("""
            SELECT action,
                   COUNT(*) as total,
                   SUM(CASE WHEN success = 0 THEN 1 ELSE 0 END) as failures
            FROM episodes
            GROUP BY action
            HAVING failures > 0
            ORDER BY failures DESC
        """).fetchall()
        return {r[0]: {"total": r[1], "failures": r[2], "rate": r[2]/r[1]} for r in rows}

    def _row_to_dict(self, row) -> dict:
        cols = ["id","agent_id","task_id","step","action","args","result","success","duration_ms","timestamp"]
        d = dict(zip(cols, row))
        d["args"] = json.loads(d["args"])
        return d

    def close(self):
        self.conn.close()
```

### 4.5 Unified Memory Interface

```python
class MemorySystem:
    """Tüm bellek katmanlarına tek noktadan erişim."""

    def __init__(self, config: dict):
        self.short_term = ShortTermMemory(max_tokens=config.get("max_tokens", 4000))
        self.long_term  = LongTermMemory(collection_name=config.get("collection", "agent_memory"))
        self.episodic   = EpisodicMemory(db_path=config.get("db_path", "episodes.db"))

    def remember(self, content: str, permanent: bool = False, metadata: dict = None):
        self.short_term.add("assistant", content)
        if permanent:
            self.long_term.store(content, metadata)

    def recall(self, query: str, k: int = 5) -> str:
        results = self.long_term.retrieve(query, k=k)
        if not results:
            return ""
        return "\n---\n".join(r["content"] for r in results)

    def get_context(self) -> list:
        return self.short_term.get_messages()
```

---

## 5. Multi-Agent Coordination

### 5.1 Coordinator (Orchestrator) Pattern

Merkezi bir agent, görevleri uzman agent'lara dağıtır ve sonuçları birleştirir.

```
         ┌──────────────┐
         │ Coordinator  │
         └──────┬───────┘
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌───────┐  ┌───────┐  ┌───────┐
│Search │  │  DB   │  │ Code  │
│ Agent │  │ Agent │  │ Agent │
└───────┘  └───────┘  └───────┘
```

```python
import asyncio
from typing import Dict, Any

class CoordinatorAgent:
    """
    Alt agent'lara görev atar ve sonuçları birleştirir.
    """

    def __init__(self, llm, sub_agents: dict):
        self.llm        = llm
        self.sub_agents = sub_agents   # {agent_name: agent_instance}

    async def run(self, task: str) -> str:
        # 1. Görevi alt görevlere böl ve agent ata
        assignments = self._assign_tasks(task)

        # 2. Paralel çalıştır
        coros   = {name: agent.run(sub_task)
                   for name, (agent, sub_task) in assignments.items()}
        results = await asyncio.gather(*coros.values(), return_exceptions=True)
        named_results = dict(zip(coros.keys(), results))

        # 3. Hataları işle
        for name, result in named_results.items():
            if isinstance(result, Exception):
                named_results[name] = f"[HATA] {name}: {result}"

        # 4. Sentezle
        return self._synthesize(task, named_results)

    def _assign_tasks(self, task: str) -> dict:
        prompt = (
            f"Görev: {task}\n\n"
            f"Kullanılabilir agent'lar: {list(self.sub_agents.keys())}\n"
            "Her agent için hangi alt görevi çözeceğini JSON olarak belirt: "
            "{agent_name: sub_task}. Yalnızca JSON döndür."
        )
        raw = self.llm.complete([{"role": "user", "content": prompt}])
        try:
            mapping = json.loads(raw)
        except json.JSONDecodeError:
            mapping = {list(self.sub_agents.keys())[0]: task}

        return {
            name: (self.sub_agents[name], sub_task)
            for name, sub_task in mapping.items()
            if name in self.sub_agents
        }

    def _synthesize(self, task: str, results: dict) -> str:
        results_str = "\n".join(f"[{k}]: {v}" for k, v in results.items())
        prompt = (
            f"Orijinal görev: {task}\n\n"
            f"Alt agent sonuçları:\n{results_str}\n\n"
            "Bu sonuçları kapsamlı bir yanıtta birleştir:"
        )
        return self.llm.complete([{"role": "user", "content": prompt}])
```

### 5.2 İletişim Protokolleri

| Protokol | Açıklama | Ne zaman |
|----------|----------|----------|
| **Direct Message** | Agent A → Agent B doğrudan mesaj | İki agent arası senkron iletişim |
| **Event Bus** | Pub/sub ile asenkron olay bildirimi | Gevşek bağlı, yüksek ölçekli sistemler |
| **Shared State** | Ortak bellek alanı (Redis vb.) | Birden fazla agent aynı veriyi okuyorsa |
| **Broadcast** | Tüm agent'lara yayın | Durum değişikliği bildirimleri |

```python
import asyncio
from collections import defaultdict

class EventBus:
    """Basit publish-subscribe olay veriyolu."""

    def __init__(self):
        self._subscribers: dict[str, list] = defaultdict(list)

    def subscribe(self, event_type: str, callback):
        self._subscribers[event_type].append(callback)

    def unsubscribe(self, event_type: str, callback):
        self._subscribers[event_type].remove(callback)

    async def publish(self, event_type: str, payload: Any):
        callbacks = self._subscribers.get(event_type, [])
        await asyncio.gather(*[
            asyncio.to_thread(cb, payload) for cb in callbacks
        ])

# Kullanım örneği
bus = EventBus()

async def on_task_completed(payload):
    print(f"Görev tamamlandı: {payload['task_id']}")

bus.subscribe("task.completed", on_task_completed)
await bus.publish("task.completed", {"task_id": "t-001", "result": "Başarılı"})
```

---

## 6. Hata Yönetimi ve Dayanıklılık

### 6.1 Circuit Breaker Pattern

Sürekli başarısız olan servislere çağrı yapmayı geçici olarak durdurup sistemin çökmesini engeller.

```python
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED   = "closed"    # Normal çalışma
    OPEN     = "open"      # Çağrılar engelleniyor
    HALF_OPEN = "half_open" # Test çağrısı izni

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        success_threshold: int = 2
    ):
        self.failure_threshold  = failure_threshold
        self.recovery_timeout   = recovery_timeout   # saniye
        self.success_threshold  = success_threshold
        self.state              = CircuitState.CLOSED
        self.failure_count      = 0
        self.success_count      = 0
        self.last_failure_time: Optional[datetime] = None

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise RuntimeError("Circuit OPEN: servis geçici olarak devre dışı.")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state         = CircuitState.CLOSED
                self.failure_count = 0
                self.success_count = 0
        else:
            self.failure_count = 0

    def _on_failure(self):
        self.failure_count    += 1
        self.last_failure_time = datetime.now()
        self.success_count     = 0
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_attempt_reset(self) -> bool:
        if not self.last_failure_time:
            return True
        elapsed = (datetime.now() - self.last_failure_time).total_seconds()
        return elapsed >= self.recovery_timeout
```

---

## 7. Gözlemlenebilirlik (Observability)

### 7.1 Structured Logging

```python
import logging
import json
from datetime import datetime

class AgentLogger:
    """Structured JSON logging. ELK/Datadog/CloudWatch ile uyumlu."""

    def __init__(self, agent_id: str, log_level=logging.INFO):
        self.agent_id = agent_id
        self.logger   = logging.getLogger(f"agent.{agent_id}")
        self.logger.setLevel(log_level)

        handler   = logging.StreamHandler()
        formatter = logging.Formatter("%(message)s")
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)

    def _log(self, level: str, event: str, **kwargs):
        entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level":     level,
            "agent_id":  self.agent_id,
            "event":     event,
            **kwargs
        }
        getattr(self.logger, level.lower())(json.dumps(entry, ensure_ascii=False))

    def step_started(self, step: int, action: str, args: dict):
        self._log("INFO", "step.started", step=step, action=action, args=args)

    def step_completed(self, step: int, action: str, duration_ms: int, success: bool):
        self._log("INFO", "step.completed",
                  step=step, action=action, duration_ms=duration_ms, success=success)

    def tool_error(self, tool: str, error: str, args: dict):
        self._log("ERROR", "tool.error", tool=tool, error=error, args=args)

    def task_completed(self, task_id: str, total_steps: int, duration_ms: int):
        self._log("INFO", "task.completed",
                  task_id=task_id, total_steps=total_steps, duration_ms=duration_ms)
```

### 7.2 Metrik Toplama

```python
from dataclasses import dataclass, field
from collections import defaultdict

@dataclass
class AgentMetrics:
    agent_id: str
    total_tasks: int = 0
    successful_tasks: int = 0
    failed_tasks: int = 0
    total_steps: int = 0
    tool_call_counts: dict = field(default_factory=lambda: defaultdict(int))
    tool_error_counts: dict = field(default_factory=lambda: defaultdict(int))
    latencies_ms: list = field(default_factory=list)

    @property
    def success_rate(self) -> float:
        return self.successful_tasks / self.total_tasks if self.total_tasks else 0.0

    @property
    def avg_latency_ms(self) -> float:
        return sum(self.latencies_ms) / len(self.latencies_ms) if self.latencies_ms else 0.0

    @property
    def p95_latency_ms(self) -> float:
        if not self.latencies_ms:
            return 0.0
        sorted_l = sorted(self.latencies_ms)
        idx = int(len(sorted_l) * 0.95)
        return sorted_l[idx]

    def to_dict(self) -> dict:
        return {
            "agent_id":        self.agent_id,
            "total_tasks":     self.total_tasks,
            "success_rate":    round(self.success_rate, 3),
            "avg_latency_ms":  round(self.avg_latency_ms, 1),
            "p95_latency_ms":  round(self.p95_latency_ms, 1),
            "tool_calls":      dict(self.tool_call_counts),
            "tool_errors":     dict(self.tool_error_counts),
        }
```

---

## 8. Agent Framework'ları

### 8.1 LangChain

**Güçlü yönleri:**
- Zengin agent abstraksiyonları (AgentExecutor, LCEL)
- 500+ entegrasyon (tool, vectorstore, LLM)
- LangSmith ile built-in tracing

**Zayıf yönleri:**
- Karmaşık ve büyük bağımlılık ağacı
- Abstraction leakage; alt katmanı kontrol etmek zorlaşabilir

**Ne zaman tercih edilir:** Hızlı prototipleme, geniş ekosistemden yararlanmak

### 8.2 CrewAI

**Güçlü yönleri:**
- Role-based multi-agent sistemi
- Yerleşik görev atama ve insan-in-the-loop desteği
- Minimal konfigürasyon ile çalışmaya başlar

**Ne zaman tercih edilir:** Farklı rollerdeki agent'ların işbirliği gerektiren iş akışları

### 8.3 AutoGen (Microsoft)

**Güçlü yönleri:**
- Conversation-driven multi-agent mimari
- Human proxy ile kolay insan dahil etme
- Güçlü code execution desteği

**Ne zaman tercih edilir:** Otonom kod yazma ve yürütme gerektiren görevler

### 8.4 Seçim Rehberi

| İhtiyaç | Öneri |
|---------|-------|
| Hızlı prototip, tek agent | LangChain |
| Çok rollu ekip simülasyonu | CrewAI |
| Otonom kod üretimi | AutoGen |
| Tam kontrol, özel mimari | Sıfırdan yazın |

---

## 9. Türkçe Agent İçin Pratik Notlar

> **Tokenization:** GPT-4 tokenizer Türkçe metni İngilizce'ye kıyasla ~%20–40 daha fazla token olarak kodlar. Token bütçenizi buna göre ayarlayın.

> **Embedding Modeli:** Türkçe long-term memory için mutlaka çok dilli model kullanın: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` veya `intfloat/multilingual-e5-large`.

> **Tool Açıklamaları:** Agent'ın tool'u doğru anlaması için açıklamaları Türkçe yazın. İngilizce açıklamalar Türkçe görevlerde anlamsal sapmalara yol açabilir.

> **System Prompt Dili:** System prompt'u Türkçe yazın. Karışık dil (Türkçe görev + İngilizce sistem prompt) model performansını düşürür.

> **Karakter Kodlaması:** `ensure_ascii=False` ile JSON serialize edin; aksi hâlde Türkçe karakterler kaçış dizisine dönüşür.

> **Hata Mesajları:** Kullanıcıya dönen hata mesajlarını Türkçe yazın; iç log'lar İngilizce kalabilir.

---

## Ekler

### A — Terimler Sözlüğü

| Terim | Açıklama |
|---|---|
| **Agent** | Hedefe yönelik eylemler gerçekleştiren otonom sistem |
| **Reasoning** | Akıl yürütme, karar verme süreci |
| **Tool** | Agent'ın dış sistemlerle etkileşim kurduğu arayüz |
| **Memory** | Geçmiş deneyimleri saklayan sistem |
| **Observation** | Eylem sonucu elde edilen bilgi |
| **Coordination** | Birden fazla agent arası işbirliği |
| **Lifecycle** | Agent'ın başlangıçtan bitişe kadar olan yaşam döngüsü |
| **Circuit Breaker** | Başarısız servislere çağrıyı geçici olarak durduran örüntü |
| **Episodic Memory** | Zaman damgalı, olay bazlı bellek |
| **Topological Sort** | Bağımlılık sırasına göre sıralama algoritması |

### B — Mini Checklist

#### Agent Tasarımı

- [ ] Agent tipi seçildi ve gerekçesi belgelendi (ReAct / Plan-and-Solve / ReWOO)
- [ ] `MAX_STEPS` ve `timeout_seconds` limitleri ayarlandı
- [ ] Memory katmanları belirlendi (short/long/episodic)
- [ ] Tool set tanımlandı ve schema'lar yazıldı
- [ ] Lifecycle durum makinesi implement edildi
- [ ] Termination koşulları (başarı, hata, timeout, iptal) tanımlandı

#### Multi-Agent Sistem

- [ ] Coordination pattern seçildi (coordinator / hierarchical / collaborative)
- [ ] İletişim protokolü belirlendi (direct / event bus / shared state)
- [ ] Agent rolleri ve sorumlulukları belgelendi
- [ ] Task assignment stratejisi test edildi

#### Gözlemlenebilirlik

- [ ] Structured logging aktif
- [ ] Metrikler toplanıyor (latency, success rate, tool calls)
- [ ] Episodic memory kayıtları incelenebilir durumda
- [ ] Circuit breaker kritik araçlara uygulandı

---

**Serinin başına dön:** [README](./README.md)  
**Tool use rehberine git:** [02 - Tool Use ve Entegrasyon](./02_-_Tool_Use_ve_Entegrasyon.md)
