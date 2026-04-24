# projeYasa.md

> Bu dosya, projeye bakan AI ajanı için **tek giriş yönlendirme belgesi**dir.
> Amaç: "hepsini baştan sona oku" yaklaşımı yerine, **göreve göre doğru dokümana git, doğru satır aralığını oku, sonra karar ver** davranışını zorunlu kılmak.

---

## 1. Amaç ve çalışma kuralı

Bu repo/doküman setinde AI şu kurala uymalıdır:

1. Önce bu dosyayı oku.
2. İstenen görevin türünü sınıflandır.
3. Aşağıdaki görev eşleme tablosundan ilgili dosyaları seç.
4. Sadece gerekli dosyaların ilgili satır aralıklarını oku.
5. Cevap veya çözüm üretirken canonical terimleri ve production gate'leri bozma.
6. Mutasyon, riskli yürütme, otomasyon veya güven etkisi olan konularda safety/trust katmanını atlama.

**Yasak davranışlar:**
- Tüm markdown dosyalarını sebepsiz yere baştan sona okumak
- `cross-layer` sözleşmesini görmeden yeni alan adı uydurmak
- `README.md` içindeki kalite kapıları ve production semantiklerini yok saymak
- Safety veya rollback gerektiren bir konuda sadece karar motoru mantığıyla ilerlemek

---

## 2. İlk okunacak çekirdek belgeler

AI her görevde önce aşağıdaki çekirdeği referans almalıdır:

### Zorunlu başlangıç sırası
1. `README.md` satır `8-74`
2. `cross-layer-integration-contract.md` satır `23-59`

### Neden?
- `README.md` okuma sırasını, kalite kapılarını, terminoloji standardını, canonical terimleri ve production semantik kapılarını tanımlar.
- `cross-layer-integration-contract.md` tüm katmanlar arasındaki normatif I/O sözleşmesini ve mimari akışı tanımlar.

### Hızlı referans
- Repo haritası ve okuma sırası: `README.md#L8-L24`
- Kalite kapıları: `README.md#L25-L33`
- Terminoloji standardı: `README.md#L34-L39`
- Canonical terimler: `README.md#L40-L48`
- Production semantics gates: `README.md#L49-L74`
- Cross-layer misyon: `cross-layer-integration-contract.md#L23-L28`
- Mimari görünüm: `cross-layer-integration-contract.md#L29-L59`

---

## 3. Görev sınıflandırma matrisi

AI, kullanıcı isteğini aşağıdaki sınıflardan birine veya birkaçına map etmelidir.

| Görev tipi | Öncelikli dosya | Satırlar | İkincil dosya | Satırlar |
|---|---|---:|---|---:|
| Genel mimari / sistem nasıl çalışıyor | `README.md` | 8-74 | `cross-layer-integration-contract.md` | 23-59 |
| Katmanlar arası veri sözleşmesi / payload / DTO / telemetry | `cross-layer-integration-contract.md` | 60-603 | `typescript.md` | 102-162 |
| Routing / orchestrator / plan / tool seçimi | `ai-decision-execution-engine.md` | 24-933 | `cross-layer-integration-contract.md` | 150-396 |
| Safety / trust / risk / guardrail / prompt injection | `ai-safety-trust-verification-layer.md` | 24-186, 2804-2967 | `README.md` | 49-80 |
| RAG / retrieval / chunk kalitesi / indexing | `rag-source-quality-rubric.md` | 51-283 | `cross-layer-integration-contract.md` | 259-396 |
| Debugging / incident / pattern / remediation | `failure-pattern-atlas.md` | 19-379 | `ai-decision-execution-engine.md` | 160-335 |
| Model iç yapısı / GGUF / quantization / inference | `gguf_llm_katmanlari.md` | 19-284 | `cross-layer-integration-contract.md` | 397-482 |
| Type safety / runtime validation / TS mimarisi | `typescript.md` | 19-223 | `cross-layer-integration-contract.md` | 483-546 |
| Otonom aksiyon / mutasyon / rollback / approval | `ai-decision-execution-engine.md` | 574-889 | `ai-safety-trust-verification-layer.md` | 2804-2967 |
| Değişiklik sonrası doğrulama / release gate / proof artifact | `README.md` | 49-108 | `cross-layer-integration-contract.md` | 547-1090 |

---

## 4. Göreve göre okunacak minimum set

### 4.1 Genel soru: “Bu sistemin omurgası nedir?”
Oku:
- `README.md#L8-L74`
- `cross-layer-integration-contract.md#L23-L59`
- `ai-decision-execution-engine.md#L24-L59`

### 4.2 “Bu task nasıl route edilir / hangi katman karar verir?”
Oku:
- `ai-decision-execution-engine.md#L30-L126`
- `ai-decision-execution-engine.md#L160-L335`
- `cross-layer-integration-contract.md#L150-L396`

### 4.3 “Bu cevap ne kadar güvenilir / safety nasıl enforce edilir?”
Oku:
- `ai-safety-trust-verification-layer.md#L30-L186`
- `ai-safety-trust-verification-layer.md#L2804-L2967`
- `README.md#L49-L80`

### 4.4 “RAG nasıl olmalı / chunk kalitesi nasıl ölçülür?”
Oku:
- `rag-source-quality-rubric.md#L51-L121`
- `rag-source-quality-rubric.md#L132-L283`
- `cross-layer-integration-contract.md#L259-L396`

### 4.5 “Hata neden oldu / nasıl diagnose ederim / nasıl önlerim?”
Oku:
- `failure-pattern-atlas.md#L19-L164`
- `failure-pattern-atlas.md#L165-L379`
- `ai-decision-execution-engine.md#L160-L259`

### 4.6 “Model runtime / GGUF / quantization / tokenizer ne durumda?”
Oku:
- `gguf_llm_katmanlari.md#L19-L284`
- `cross-layer-integration-contract.md#L397-L482`

### 4.7 “TypeScript tasarımı / runtime validation / type contract nasıl olmalı?”
Oku:
- `typescript.md#L19-L162`
- `cross-layer-integration-contract.md#L483-L546`

### 4.8 “Riskli işlem, tool execution veya otomatik değişiklik yapacağım”
Oku:
- `ai-decision-execution-engine.md#L574-L889`
- `ai-safety-trust-verification-layer.md#L2804-L2967`
- `README.md#L49-L108`

---

## 5. Normatif öncelik sırası

Birbiriyle yarışan bilgi varsa aşağıdaki öncelik uygulanır:

1. `README.md` içindeki kalite kapıları, canonical terimler ve production semantics gates
2. `cross-layer-integration-contract.md` içindeki normatif sözleşmeler
3. İlgili uzman katman dokümanı
   - routing için `ai-decision-execution-engine.md`
   - güven/safety için `ai-safety-trust-verification-layer.md`
   - retrieval için `rag-source-quality-rubric.md`
   - debugging için `failure-pattern-atlas.md`
   - model/runtime için `gguf_llm_katmanlari.md`
   - type-system için `typescript.md`
4. Örnek/reference kod blokları

**Kural:** Örnek kod, normatif sözleşmeyi ezemez. Çelişki varsa sözleşme kazanır.

---

## 6. AI davranış politikası

### 6.1 Cevap üretmeden önce zorunlu kontrol listesi
- [ ] Görevin türünü sınıflandırdım
- [ ] `README.md` içindeki kalite kapılarıyla çelişmiyorum
- [ ] Canonical terimleri bozmadım
- [ ] Gerekliyse `cross-layer` sözleşmesini referans aldım
- [ ] Riskli aksiyon varsa safety/trust katmanını okudum
- [ ] Mutasyon varsa rollback/idempotency/approval kontrol ettim
- [ ] RAG/retrieval konusuysa chunk kalite eşiklerini dikkate aldım
- [ ] Hata analizi ise semptom ile root cause'u karıştırmadım

### 6.2 Mutasyon ve otomasyon kuralı
Aşağıdaki konularda AI dar kapsamlı ve kanıtlı davranmalıdır:
- dosya yazma
- config değiştirme
- servis restart
- otomatik remediation
- tool chain ile dış etki oluşturan aksiyonlar

Bu durumlarda şu satırlar referans kabul edilmelidir:
- `README.md#L49-L74`
- `ai-decision-execution-engine.md#L574-L889`
- `ai-safety-trust-verification-layer.md#L2804-L2967`

### 6.3 Belirsizlik kuralı
Belirsizlik, eksik bağlam veya çelişkili kanıt varsa AI tek dokümana dayanarak kesin hüküm vermemelidir. Önce uygun satır aralığını okuyup kapsamı daraltmalıdır.

---

## 7. Dosya bazlı hızlı başvuru haritası

### `README.md`
Amaç: Doküman setinin anayasası.

Önemli alanlar:
- Okuma sırası: `L8-L24`
- Kalite kapıları: `L25-L33`
- Terminoloji standardı: `L34-L39`
- Canonical terimler: `L40-L48`
- Production gates: `L49-L74`
- Interaction policy + proof artifacts: `L75-L108`
- Güncelleme prensibi: `L109-L113`

### `cross-layer-integration-contract.md`
Amaç: Katmanlar arası normatif I/O sözleşmesi.

Önemli alanlar:
- Misyon: `L23-L28`
- Mimari görünüm: `L29-L59`
- Failure Atlas entegrasyonu: `L60-L149`
- Safety entegrasyonu: `L150-L258`
- RAG entegrasyonu: `L259-L396`
- LLM/GGUF entegrasyonu: `L397-L482`
- Type contracts: `L483-L546`
- Telemetry: `L547-L603`
- Fallback / validation / deployment: `L604-L1090`

### `ai-decision-execution-engine.md`
Amaç: Orchestrator ve karar motoru.

Önemli alanlar:
- Misyon: `L24-L29`
- Temel akış: `L30-L126`
- Karar katmanı: `L127-L573`
- Tool orkestrasyonu: `L574-L889`
- Bağlam oluşturucu: `L890-L933`
- Bellek ve devamı: `L934-L3496`

### `ai-safety-trust-verification-layer.md`
Amaç: Güven, risk, trust score, safety enforcement.

Önemli alanlar:
- Misyon: `L24-L29`
- Trust pipeline: `L30-L50`
- Decision-safety integration: `L51-L186`
- Hallucination detection: `L187-L422`
- Confidence calibration ve trust modeli: `L423-L2803`
- Proactive safety semantics: `L2804-L2967`

### `rag-source-quality-rubric.md`
Amaç: Chunk kalitesi, retrieval kalitesi, hybrid search ve parent-child chunking.

Önemli alanlar:
- Chunk quality framework: `L51-L61`
- Puanlama ve eşikler: `L62-L121`
- Detaylı kriterler: `L122-L131`
- Hybrid search: `L132-L283`
- Small-to-Big retrieval: `L284-L4010`

### `failure-pattern-atlas.md`
Amaç: Hata hafızası, pattern teşhisi, remediation ve önleme.

Önemli alanlar:
- Atlas tanımı: `L19-L49`
- 3 katmanlı mimari: `L50-L86`
- Ana yapı: `L87-L128`
- Decision tree: `L129-L164`
- İdeal schema: `L165-L379`
- AI IDE çalışma şekli ve devamı: `L380-L2167`

### `gguf_llm_katmanlari.md`
Amaç: Model iç yapısı, quantization, runtime ve introspection.

Önemli alanlar:
- AI kullanım stratejisi: `L19-L30`
- GGUF: `L31-L45`
- Metadata: `L46-L60`
- Tokenizer: `L61-L90`
- Quantization: `L91-L135`
- Attention ve devamı: `L136-L243`
- Kullanım ipuçları: `L244-L284`
- Veri akışı ve runtime devamı: `L285-L1655`

### `typescript.md`
Amaç: Type-system engineering ve runtime validation.

Önemli alanlar:
- TS 5+ ve decorators: `L19-L70`
- `satisfies`: `L71-L101`
- Runtime validation: `L102-L135`
- Interface vs Type: `L136-L162`
- Type-level debugging: `L163-L194`
- Template literal types: `L195-L223`
- Recursive flattening ve devamı: `L224-L2585`

---

## 8. AI için önerilen cevap üretim şablonu

AI mümkün olduğunda kendi iç analizinde şu formatı izlemelidir:

```text
Görev türü: <routing | safety | rag | debug | gguf | typescript | integration>
Okunan dosyalar:
- <dosya adı>#Lx-Ly
- <dosya adı>#Lx-Ly

Canonical constraints:
- <terim / gate / policy>
- <terim / gate / policy>

Karar:
- <ne öneriliyor>

Gerekçe:
- <hangi belge/satır bunu destekliyor>

Riskler:
- <safety / trust / rollback / ambiguity>

Gerekirse sonraki okuma:
- <dosya adı>#Lx-Ly
```

---

## 9. Güncelleme kuralı

Bu dosya, doküman setinin üst-yönlendirme dosyasıdır. Dokümanlarda yeni canonical alan, yeni gate, yeni sözleşme veya yeni katman eklendiğinde:

1. Önce `README.md` ve gerekiyorsa `cross-layer-integration-contract.md` güncellenir.
2. Sonra bu dosyadaki görev eşleme tablosu güncellenir.
3. Son olarak satır aralıkları yeniden senkronize edilir.

---

## 10. Kısa komut

AI için kısa talimat:

> Önce `projeYasa.md` oku. Görev tipini belirle. Yalnızca ilgili dosya ve satır aralıklarını oku. `README.md` kalite kapıları ile `cross-layer-integration-contract.md` normatif sözleşmesini ihlal etme. Riskli aksiyonlarda safety/trust katmanını atlama.

