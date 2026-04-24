# Yapay Zeka Dokümantasyon Seti

Bu klasör, yapay zeka geliştirme ve sistem mimarisi ile ilgili kapsamlı bir dokümantasyon seti içerir.

## Klasörler

### 01 - YapayZeka-YapımVeEğitim
Türkçe dil modeli sıfırdan eğitimi için uçtan uca rehber.

**İçerik:**
- Model mimarisi seçimi
- Tokenizer eğitimi
- Veri hazırlığı
- Pre-training, SFT, Alignment
- Quantization ve deployment
- İleri teknikler (stabilite, optimizasyon)

**Hedef Kitle:** Model eğitimi yapan mühendisler

**Okuma Sırası:**
1. README.md
2. 01 - Hızlı Başlangıç ve Mimari.md
3. 02 - Uçtan Uca Eğitim Rehberi.md
4. 03 - Küçültme, Dağıtım ve Tahminleme.md
5. 04 - İleri Teknikler.md

---

### 02 - AI-Sistem-Mimarisi
AI karar-yürütme platformunun mimarisi ve orchestration katmanları.

**İçerik:**
- Cross-layer integration contract
- AI decision execution engine
- Safety, trust ve verification layer
- RAG source quality rubric
- Failure pattern atlas
- GGUF/LLM katmanları
- TypeScript mimarisi

**Hedef Kitle:** Platform/AI sistem mühendisleri

**Okuma Sırası:**
1. README.md
2. projeyasa.md (giriş yönlendirme)
3. cross-layer-integration-contract.md
4. ai-decision-execution-engine.md
5. ai-safety-trust-verification-layer.md
6. rag-source-quality-rubric.md
7. failure-pattern-atlas.md
8. gguf_llm_katmanlari.md
9. typescript.md

---

### 03 - AI-Agent-Tool-Sistemi
AI agent sistemleri, tool kullanımı, skill yönetimi ve system prompt tasarımı.

**İçerik:**
- Agent framework ve mimari
- Tool use ve entegrasyon
- Skills ve capabilities
- Çalışma alanı yönetimi
- System prompt tasarımı

**Hedef Kitle:** AI mühendisleri, agent geliştiricileri

**Okuma Sırası:**
1. README.md
2. 01 - Agent Framework ve Mimari.md
3. 02 - Tool Use ve Entegrasyon.md
4. 03 - Skills ve Capabilities.md
5. 04 - Çalışma Alanı Yönetimi.md
6. 05 - System Prompt Tasarımı.md

---

## Kullanım Kılavuzu

### Hangi Klasörü Seçmelisiniz?

**Model eğitimi yapmak istiyorsanız:**
- `01 - YapayZeka-YapımVeEğitim` klasörü
- Tokenizer, veri hazırlığı, pre-training, deployment

**AI sistemi kurmak istiyorsanız:**
- `02 - AI-Sistem-Mimarisi` klasörü
- Orchestration, safety, RAG, failure patterns

**Agent sistemi geliştirmek istiyorsanız:**
- `03 - AI-Agent-Tool-Sistemi` klasörü
- Agent framework, tools, skills, workspace, prompts

### Önerilen Okuma Sırası

Başlangıç seviyesi için:
1. `01 - YapayZeka-YapımVeEğitim` → Model eğitimi temelleri
2. `03 - AI-Agent-Tool-Sistemi` → Agent ve tool sistemleri
3. `02 - AI-Sistem-Mimarisi` → İleri seviye sistem mimarisi

---

## Sürüm Bilgisi

- **v1.0** (2026-04-23) - İlk sürüm
- Tüm dokümanlar `living specification` olarak ele alınır
- Güncellemeler doküman kartlarında belirtilir

## Katkı

Bu dokümantasyon seti sürekli gelişmektedir. Öneriler ve geri bildirimler için ilgili klasördeki README.md dosyalarını inceleyin.
