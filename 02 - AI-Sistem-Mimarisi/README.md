# AI Sistem Dokümantasyon Haritası

**Spec Bundle Sürümü:** `v1.5`  
**Son senkronizasyon tarihi:** `2026-04-03`

Bu klasör, AI karar-yürütme platformunun 8 ana teknik dokümanını içerir. Her doküman, kendi içinde "Doküman Kartı" ile rol, kapsam ve bağımlılıklarını belirtir.

## Okuma Sırası (Önerilen)

0. [projeyasa.md](projeyasa.md)  
   Proje oluşturma aşamasında giriş yönlendirme belgesi. AI'ın hangi dokümanı ne zaman okuyacağını belirler.
1. [cross-layer-integration-contract.md](cross-layer-integration-contract.md)  
   Katmanlar arası sözleşmelerin ve veri akışının ana kaynağı.
2. [ai-decision-execution-engine.md](ai-decision-execution-engine.md)  
   Orchestrator, routing, fallback ve politika motoru.
3. [ai-safety-trust-verification-layer.md](ai-safety-trust-verification-layer.md)  
   Güven, risk, insan onayı ve güvenlik sınırları.
4. [rag-source-quality-rubric.md](rag-source-quality-rubric.md)  
   Retrieval kalitesi, eşikler, kalite ölçümü ve operasyon.
5. [failure-pattern-atlas.md](failure-pattern-atlas.md)  
   Hata örüntüsü hafızası, remediation ve regresyon önleme.
6. [gguf_llm_katmanlari.md](gguf_llm_katmanlari.md)  
   Model iç katmanları, quantization ve inference optimizasyonu.
7. [typescript.md](typescript.md)  
   Type-system tasarımı, migration, runtime validation pratikleri.

## Kalite Kapıları

- Kod blokları dengeli (`fenced code blocks` açık/kapalı tutarlı).
- Her kod bloğu için `Kod Durumu` etiketi var (`Reference` veya `Production-Ready`).
- Terminoloji ve alan adları katman sözleşmeleriyle uyumlu.
- "Pseudo/reference code" ile doğrudan production kodu ayrıştırılmış.
- Her doküman en az bir çapraz dokümana referans veriyor.
- Operasyonel başlıklar mevcut: ölçüm, hata, geri dönüş/rollback, doğrulama.

## Terminoloji Standardı

- Varsayılan stil: `camelCase`
- Kapsam: iç DTO alanları, enum literal'ları, policy/route adları, sözleşme anahtarları
- Geçiş kuralı: yeni eklenen terimler önce `cross-layer` sözleşmesinde canonical hale getirilir, sonra diğer dokümanlara yayılır

## v1.5 Canonical Terimler

- `TraceEnvelope`, `ReplayEvent`, `traceId` propagation
- `SLAMode` (`cheap | balanced | critical`)
- `ContractErrorCode` (`E_VALIDATION`, `E_TIMEOUT`, `E_DEPENDENCY`, `E_SAFETY`, `E_POLICY`, `E_RESOURCE`, `E_INTERNAL`)
- `IdempotencyContract` / `idempotencyKey`
- `TrustActionPolicy` eşikleri
- `RuntimeContractGate` ve `ContractTestSuite`

## v1.5 Production Semantics Gates

- Plan-first execution zorunlu (`query -> plan -> steps -> execute -> adapt`)
- Deterministic replay zorunlu (`EventStore + sequence + deterministicInputs`)
- Proactive safety zorunlu (input/context/tool-chain validation)
- Hard trust policy zorunlu (`<0.4 halt`, `0.4-0.7 modify`, `>=0.7 proceed`)
- Unified learning loop zorunlu (failure + trust + rag + decision sinyalleri tek kayıtta)
- Idempotency zorunlu (mutating operasyonlarda `idempotencyKey`)
- Runtime scheduling semantics zorunlu (`queue pressure -> backpressure action`)
- Multi-agent coordination zorunlu (`lease + arbitration`)
- Eval-regression rollback zorunlu (`offline eval + automatic policy rollback`)
- Degradation tiers zorunlu (`deep -> shallow -> cached -> refuse`)

## v1.5 Dominance Scoreboard (League Table)

- `taskSuccessRate`
- `groundedAnswerRate`
- `unsafeActionBlockRate`
- `falseBlockRate`
- `retryRecoveryRate`
- `editCorrectness`
- `timeToCorrectness`
- `costPerSuccess`
- `duplicateActionRate`
- `hallucinationEscapeRate`

## v1.5 Interaction Policy Layer

- Motor davranış modu: `act | ask | explain | silent`
- Amaç: sadece doğru sonuç değil, doğru davranış ve doğru zamanda müdahale

## v1.5 Execution-Grade Proof Artifacts

- `taskGraphExecutionAudit` (step state transitions + dependency resolution)
- `deterministicReplayAudit` (sequence completeness + deterministic input match)
- `proactiveSafetyAudit` (input/context/tool-chain block decisions)
- `idempotencyCollisionAudit` (duplicate suppression outcomes)
- `rollbackReadinessAudit` (mutation öncesi rollback plan doğrulaması)
- `evidenceResolutionAudit` (conflict score + citation merge + freshness decisions)
- `schedulerBackpressureAudit` (load altında throttle/defer/degrade/reject kararları)
- `multiAgentLeaseAudit` (shared-state lock/lease ve arbitration kayıtları)
- `policyRegressionAudit` (offline eval regression + auto rollback kararları)

Zorunlu kural: Bu artefact'lar sadece tanımlı olmakla kalmaz; her production trace için üretilmiş ve saklanmış olmalıdır.

## v1.5 Release Governance Hard Gates

- `canaryPromotionGate` geçmeden rollout yapılamaz.
- `backward compatibility breach` durumunda promotion otomatik durur.
- Kritik breach'te `emergencyDowngrade` sözleşmesi otomatik tetiklenir.
- `schemaDriftAlarm` production ortamında incident seviyesinde ele alınır.

### Domain-Bazlı İstisnalar (snake_case korunur)

- Dış sistem/protokol anahtarları (`OpenTelemetry` tags, vendor API alanları)
- SQL/DB kolon adları ve migration çıktıları
- CLI flag/env değişkenleri (`--long_flag`, `ENV_VAR`)
- Dosya adları ve path segmentleri (`gguf_llm_katmanlari.md` gibi)
- Compliance/labeling sözlükleri (regülasyon gereği sabit anahtarlar)

## Güncelleme Prensibi

- Dokümanlar `living specification` olarak ele alınır.
- Büyük teknik değişikliklerde önce sözleşme (`cross-layer`) güncellenir, ardından katman dokümanlarına yayılır.
- Üretim değişikliklerinde tarih ve etki alanı doküman kartına işlenir.
