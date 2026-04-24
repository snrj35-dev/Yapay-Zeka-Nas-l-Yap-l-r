# AI Agent ve Tool Sistemi — Dokümantasyon Haritası

**Spec Bundle Sürümü:** `v2.0`  
**Son güncelleme:** `2026-04-24`  
**Durum:** Living Specification

---

## Genel Bakış

Bu klasör, production-grade AI agent sistemleri inşa etmek isteyen mühendisler için kapsamlı bir referans kaynağıdır. Mimari kararlardan güvenlik politikalarına, prompt tasarımından skill composition'a kadar tüm katmanları ele alır.

Her doküman kendi içinde bağımsız okunabilir, ancak aralarındaki bağımlılıklar açıkça belirtilmiştir.

---

## Dokümantasyon Haritası

```
agent-docs/
├── README.md                        ← Bu dosya
├── 01 - Agent Framework ve Mimari   ← Agent tipleri, lifecycle, multi-agent, memory
├── 02 - Tool Use ve Entegrasyon     ← Tool tanımlama, function calling, API, hata yönetimi
├── 03 - Skills ve Capabilities      ← Skill framework, discovery, composition, evaluation
├── 04 - Çalışma Alanı Yönetimi      ← Sandbox, file ops, context, güvenlik, resource mgmt
└── 05 - System Prompt Tasarımı      ← Prompt engineering, few-shot, CoT, Türkçe prompt
```

### Bağımlılık Grafiği

```
01 - Agent Framework
    ├── 02 - Tool Use          (agent araçlarını kullanır)
    │       └── 03 - Skills    (tool'ları skill soyutlamalarına sarar)
    │               └── 04 - Workspace  (skill'ler workspace'e yazar/okur)
    └── 05 - System Prompt     (agent davranışını tanımlar)
```

---

## Okuma Sırası

| # | Doküman | Ne öğrenirsiniz |
|---|---------|----------------|
| 1 | [01 - Agent Framework ve Mimari](./01_-_Agent_Framework_ve_Mimari.md) | ReAct, Plan-and-Solve, ReWOO agent tipleri; lifecycle loop; short/long/episodic memory; multi-agent coordination |
| 2 | [02 - Tool Use ve Entegrasyon](./02_-_Tool_Use_ve_Entegrasyon.md) | JSON Schema ile tool tanımlama; function calling; REST/GraphQL/DB/FS tool implementasyonları; retry & fallback |
| 3 | [03 - Skills ve Capabilities](./03_-_Skills_ve_Capabilities.md) | Tool üstü skill soyutlaması; semantic/dynamic discovery; sequential/parallel/conditional composition; evaluation |
| 4 | [04 - Çalışma Alanı Yönetimi](./04_-_Calisma_Alani_Yonetimi.md) | Docker/process sandbox; safe file ops; versioning; context TTL; permission sistemi; audit logging |
| 5 | [05 - System Prompt Tasarımı](./05_-_System_Prompt_Tasarimi.md) | System prompt yapısı; few-shot/CoT örnekleri; A/B test framework; Türkçe prompt tasarımı |

---

## Kapsam

### Bu klasörde yer alanlar

- Agent mimarisi, koordinasyon protokolleri ve lifecycle yönetimi
- Tool tanımlama, function calling ve API entegrasyon pattern'leri
- Skill yönetimi, discovery stratejileri ve composition pattern'leri
- Sandbox environment kurulumu ve güvenlik politikaları
- System prompt mühendisliği ve Türkçe dil özelinde notlar

### Kapsam dışı

| Konu | İlgili Kaynak |
|------|--------------|
| Model eğitimi, fine-tuning, tokenizer | `01 - YapayZeka-YapımVeEğitim` |
| AI sistemi orchestration, safety katmanı, RAG pipeline | `02 - AI-Sistem-Mimarisi` |
| Model deployment, inference serving, monitoring | `01 - YapayZeka-YapımVeEğitim/03` |

---

## Teknoloji Referansı

Bu dokümanlarda geçen başlıca teknolojiler ve önerilen minimum sürümler:

| Teknoloji | Sürüm | Kullanım Yeri |
|-----------|-------|---------------|
| Python | 3.11+ | Tüm implementasyon örnekleri |
| LangChain | 0.2+ | Agent framework |
| CrewAI | 0.30+ | Multi-agent coordination |
| ChromaDB | 0.4+ | Vector memory |
| Docker SDK | 7.0+ | Sandbox environment |
| sentence-transformers | 2.6+ | Semantic skill/tool discovery |
| tenacity | 8.x | Retry logic |
| psutil | 5.x | Resource monitoring |

---

## Güncelleme Prensibi

Dokümanlar **living specification** olarak yönetilir:

- Mimari değişiklikler önce `01 - Agent Framework`'te güncellenir, ardından bağımlı dokümanlara yayılır.
- Her dokümanın başındaki `Doküman Kartı` hangi sürüm numarasında olduğunu ve son güncellenme tarihini taşır.
- Kod örnekleri **referans implementasyon**dur; production'a almadan önce kendi ortamınızda test etmelisiniz.
- Breaking change içeren güncellemeler, bağımlı dokümanlarda `⚠️ Breaking:` etiketiyle işaretlenir.

---

## Katkıda Bulunma

Yeni bir doküman eklerken:

1. `Doküman Kartı` bölümünü doldurun (rol, bağımlılıklar, birincil okur)
2. README'ye satır ekleyin ve bağımlılık grafiğini güncelleyin
3. Referans verdiğiniz diğer dokümanlara çift yönlü link ekleyin
4. Tüm kod bloklarını `python` / `json` etiketiyle işaretleyin

---

*Serinin tüm dokümanlarına yukarıdaki tablodan ulaşabilirsiniz.*
