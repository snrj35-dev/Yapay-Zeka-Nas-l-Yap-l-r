# RAG Source Quality Rubric: İyi Chunk/Kötü Chunk Kriterleri + Puanlama Şeması

## 📌 Doküman Kartı

| Alan | Değer |
|---|---|
| Rol | RAG kaynak/chunk kalitesini ölçen normatif rubric ve operasyon rehberi |
| Durum | Living specification (`v1.5`) |
| Son güncelleme | 2026-04-03 |
| Birincil okur | Retrieval mühendisleri, veri kalite ekipleri, MLOps |
| Ana girdi | Chunk içerikleri, metadata, retrieval sonuçları, kullanıcı geri bildirimi |
| Ana çıktı | Kalite skorları, eşik kararları, iyileştirme/rollback aksiyonları |
| Bağımlı dokümanlar | [cross-layer-integration-contract.md](cross-layer-integration-contract.md), [ai-safety-trust-verification-layer.md](ai-safety-trust-verification-layer.md), [ai-decision-execution-engine.md](ai-decision-execution-engine.md), [typescript.md](typescript.md) |

**Kalite notu:** Bu doküman hem ölçüm sözleşmesi hem de operasyon kitabıdır. Eşikler ortam bazında kalibre edilmeden production'da sabit kabul edilmemelidir.

---

## İçindekiler (Table of Contents)

### 📋 Temel Framework (Bölüm 1-2)
1. [Chunk Quality Framework](#1-chunk-quality-framework)
2. [Detaylı Kriterler ve Puanlama](#2-detaylı-kriterler-ve-puanlama)

### 🏗️ Uygulama Katmanları (Bölüm 3-8)
3. [Query-Chunk Matching Layer](#3-query-chunk-matching-layer)
4. [Chunk Lifecycle ve Aging Management](#4-chunk-lifecycle-ve-aging-management)
5. [Query Failure Analysis System](#5-query-failure-analysis-system)
6. [Advanced Ranking Strategies](#6-advanced-ranking-strategies)
7. [Cost & Performance Optimization](#7-cost--performance-optimization)
8. [Feedback-Driven Retraining Pipeline](#8-feedback-driven-retraining-pipeline)

### 🚀 İleri Seviye Özellikler (Bölüm 9-12)
9. [Multi-Hop Retrieval and Reasoning](#9-multi-hop-retrieval-and-reasoning)
10. [Chunk Linking and Knowledge Graph](#10-chunk-linking-and-knowledge-graph)
11. [Enhanced Evaluation Dataset](#11-enhanced-evaluation-dataset)
12. [Lite Mode Configuration](#12-lite-mode-configuration)

### 🔧 Bakım ve Operasyon (Bölüm 13-15)
13. [Versiyonlama ve Rollback Prosedürleri](#13-versiyonlama-ve-rollback-prosedürleri)
14. [Threshold Tuning ve Grid Search](#14-threshold-tuning-ve-grid-search)
15. [İnsan-in-the-Loop ve Anotatör Rehberi](#15-insan-in-the-loop-ve-anotatör-rehberi)

### 📎 Ekler (Appendix)
A. [Detaylı Otomatik Değerlendirme Script'i](#a-appendix-detaylı-otomatik-değerlendirme-scripti)
B. [Legal & Compliance Rubric](#b-legal--compliance-rubric)
C. [Hibrit Arama Stratejileri](#c-hibrit-arama-stratejileri)

---

## 1. Chunk Quality Framework

### 1.1. Temel Değerlendirme Boyutları

RAG sistemlerinde chunk kalitesi 4 ana boyutta değerlendirilir:

> **Kod Durumu:** `Reference`
```
Chunk Quality = 0.3·SemanticCoherence + 0.25·ContextualCompleteness + 0.25·RetrievalEfficiency + 0.2·Actionability
```

### 1.2. Puanlama Şeması ve Eşik Değerler

**0-10 Skalası (Normalizasyon Kuralları):**
- **9-10:** Mükemmel (Production-ready)
- **7-8:** İyi (Günlük kullanım için uygun)
- **5-6:** Orta (Düzeltilmesi gereken sorunlar var)
- **3-4:** Zayıf (Ciddi eksiklikler)
- **0-2:** Kötü (Kullanılamaz)

**Alt-Metrik Hesaplama Formülleri:**
> **Kod Durumu:** `Reference`
```typescript
// Semantic Coherence (0-10)
semanticCoherence = (topicCoherence * 0.4) + (terminologyConsistency * 0.3) + (lmCoherence * 0.3)

// Contextual Completeness (0-10)
contextualCompleteness = (importPresence * 3) + (typeCoverage * 3) + (errorHandling * 2) + (documentation * 2)

// Retrieval Efficiency (0-10)
retrievalEfficiency = (tokenScore * 4) + (keywordScore * 3) + (metadataScore * 3)

// Actionability (0-10)
actionability = (codeExamples * 4) + (stepByStep * 3) + (copyPasteReady * 3)
```

**Production Eşik Değerleri:**
- **Minimum Overall Score:** 7.0
- **Semantic Coherence:** ≥ 7.0 (tek konu odaklı)
- **Contextual Completeness:** ≥ 6.0 (temel context tam)
- **Retrieval Efficiency:** ≥ 7.0 (optimal boyut ve metadata)
- **Actionability:** ≥ 6.0 (pratik örnek içerir)

**Örnek Puanlama Hesaplaması:**
> **Kod Durumu:** `Reference`
```typescript
// Örnek Chunk: TypeScript interface with documentation
const exampleChunk = {
  content: `interface User {\n  id: string;\n  name: string;\n  email: string;\n}`,
  metadata: { hasImports: true, typeCoverage: 8, hasDocs: true }
};

// Hesaplama
const contextualCompleteness = (3) + (2.4) + (0) + (2) = 7.4/10
const retrievalEfficiency = (4) + (2) + (2.5) = 8.5/10
const actionability = (0) + (0) + (3) = 3.0/10

// Final skor (ağırlıklı ortalama)
const overallScore = (7.4 * 0.25) + (8.5 * 0.25) + (3.0 * 0.2) = 6.6/10
```

**Örnek Puanlama Kuralları:**
- **Semantic Coherence 9-10:** Tek konsept, mantıksal akış, tutarlı terminoloji
- **Semantic Coherence 3-4:** Farklı konseptler karışık, mantıksal bağlantı yok
- **Contextual Completeness 8+:** Gerekli import'lar, type'lar, error handling tam
- **Contextual Completeness 4-:** Import'lar eksik, type'lar belirtilmemiş
- **Retrieval Efficiency 9-10:** 200-800 token, anlamlı başlık, iyi keyword coverage
- **Retrieval Efficiency 3-4:** <50 veya >2000 token, anlamsız başlık
- **Actionability 8+:** Copy-paste çalışır kod, adım adım talimatlar
- **Actionability 3-4:** Sadece teori, çalışmayan örnekler

## 2. Detaylı Kriterler ve Puanlama

### 2.1. Semantic Coherence (Anlamsal Bütünlük) - %30

**Mükemmel (9-10 puan):**
- Tek bir konsept/fonksiyon odaklı
- Mantıksal akış bozulmadan tamamlanmış
- Dil ve terminoloji tutarlı
- Başlangıç-orta-son yapısı net

### 2.2. Hibrit Arama (Hybrid Search) Stratejileri

**Vector + Keyword Kombinasyonu:**

| Chunk Tipi | Vector Search | Keyword Search (BM25) | Hibrit Performans | Önerilen Strateji |
|------------|---------------|---------------------|------------------|------------------|
| **Kod Chunk'ları** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Vector (0.6) + BM25 (0.4) |
| **Dokümantasyon** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | Vector (0.7) + BM25 (0.3) |
| **API Reference** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | BM25 (0.6) + Vector (0.4) |
| **Error Messages** | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | BM25 (0.8) + Vector (0.2) |
| **Best Practices** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | Vector (0.8) + BM25 (0.2) |

**Hibrit Arama Implementasyonu:**
> **Kod Durumu:** `Reference`
```typescript
interface HybridSearchResult {
  vectorScore: number;
  keywordScore: number;
  hybridScore: number;
  chunk: Chunk;
}

class HybridSearchEngine {
  private vectorDB: VectorDatabase;
  private keywordIndex: BM25Index;
  
  async hybridSearch(
    query: string,
    options: {
      vectorWeight: number;
      keywordWeight: number;
      topK: number;
    }
  ): Promise<HybridSearchResult[]> {
    // 1. Parallel search
    const [vectorResults, keywordResults] = await Promise.all([
      this.vectorDB.search(query, { topK: options.topK * 2 }),
      this.keywordIndex.search(query, { topK: options.topK * 2 })
    ]);
    
    // 2. Score normalization
    const normalizedVector = this.normalizeScores(vectorResults);
    const normalizedKeyword = this.normalizeScores(keywordResults);
    
    // 3. Merge and re-score
    const mergedResults = this.mergeResults(
      normalizedVector,
      normalizedKeyword,
      options.vectorWeight,
      options.keywordWeight
    );
    
    // 4. Apply diversity and return top-k
    return this.applyDiversity(mergedResults).slice(0, options.topK);
  }
  
  private mergeResults(
    vectorResults: SearchResult[],
    keywordResults: SearchResult[],
    vectorWeight: number,
    keywordWeight: number
  ): HybridSearchResult[] {
    const chunkMap = new Map<string, HybridSearchResult>();
    
    // Process vector results
    for (const result of vectorResults) {
      chunkMap.set(result.chunkId, {
        vectorScore: result.score,
        keywordScore: 0,
        hybridScore: result.score * vectorWeight,
        chunk: result.chunk
      });
    }
    
    // Process keyword results
    for (const result of keywordResults) {
      const existing = chunkMap.get(result.chunkId);
      if (existing) {
        // Update existing result
        existing.keywordScore = result.score;
        existing.hybridScore += result.score * keywordWeight;
      } else {
        // Add new result
        chunkMap.set(result.chunkId, {
          vectorScore: 0,
          keywordScore: result.score,
          hybridScore: result.score * keywordWeight,
          chunk: result.chunk
        });
      }
    }
    
    return Array.from(chunkMap.values());
  }
  
  private normalizeScores(results: SearchResult[]): SearchResult[] {
    if (results.length === 0) return results;
    
    const maxScore = Math.max(...results.map(r => r.score));
    const minScore = Math.min(...results.map(r => r.score));
    const range = maxScore - minScore || 1;
    
    return results.map(result => ({
      ...result,
      score: (result.score - minScore) / range // Normalize to 0-1
    }));
  }
}

// Dynamic weight selection based on query type
function getHybridWeights(queryType: string): { vectorWeight: number; keywordWeight: number } {
  const weights: Record<string, { vectorWeight: number; keywordWeight: number }> = {
    'syntaxError': { vectorWeight: 0.3, keywordWeight: 0.7 },  // Exact match important
    'concept': { vectorWeight: 0.8, keywordWeight: 0.2 },        // Semantic understanding
    'apiReference': { vectorWeight: 0.4, keywordWeight: 0.6 }, // Exact API names
    'bestPractice': { vectorWeight: 0.7, keywordWeight: 0.3 },   // Conceptual similarity
    'debugging': { vectorWeight: 0.5, keywordWeight: 0.5 }       // Balanced approach
  };
  
  return weights[queryType] || { vectorWeight: 0.6, keywordWeight: 0.4 };
}
```

**Hibrit Arama Optimizasyon Stratejileri:**
- **Query Analysis:** Query type'ına göre dinamik ağırlıklandırma
- **Score Fusion:** Early fusion (score combination) vs Late fusion (result merging)
- **Threshold Optimization:** Her chunk type için optimal threshold tuning
- **Performance Caching:** Vector ve keyword sonuçlarının cache'lenmesi

| Strateji | Performance (Latency) | Accuracy (Context) | Uygun Kullanım Senaryosu | Quality Impact |
|----------|---------------------|-------------------|------------------------|----------------|
| **Fixed-Size** | Çok Yüksek (⚡⚡⚡) | Düşuk (⭐) | Düz metinler, basit aramalar | Retrieval: 9/10, Context: 3/10 |
| **Recursive Character** | Orta (⚡⚡) | Yüksek (⭐⭐⭐) | Kod dökümanları, Markdown | Retrieval: 7/10, Context: 8/10 |
| **Semantic Chunking** | Düşük (⚡) | Çok Yüksek (⭐⭐⭐⭐) | Karmaşık akademik makaleler | Retrieval: 6/10, Context: 9/10 |
| **Small-to-Big (Parent-Child)** | Orta (⚡⚡) | Çok Yüksek (⭐⭐⭐⭐) | Enterprise RAG sistemleri | Retrieval: 8/10, Context: 9/10 |

**Strateji Seçim Rehberi:**
- **Hız kritikse:** Fixed-Size (log analysis, simple search)
- **Kod dökümanları için:** Recursive Character (syntax preservation)
- **Akademik içerik için:** Semantic Chunking (concept boundaries)
- **Production RAG için:** Small-to-Big (balance of speed + context)

### 2.3. Common Pitfalls ve Çözümleri

| Pitfall | Sorun Açıklaması | Etkilediği Kriterler | Çözüm Stratejisi |
|---------|----------------|-------------------|-----------------|
| **Lost-in-the-Middle** | Chunk'ın çok uzun olması durumunda LLM'in orta kısımdaki bilgiyi kaçırması | Contextual Completeness (-3), Actionability (-2) | Chunk boyutunu 500 token civarında optimize et, Small-to-Big Retrieval kullan |
| **Semantic Noise** | Kod chunk'larının içine giren alakasız yorum satırlarının embedding kalitesini bozması | Semantic Coherence (-4), Retrieval Efficiency (-2) | Pre-processing aşamasında docstring dışındaki gereksiz yorumları temizle |
| **Context Fragmentation** | Tek bir konseptin birden fazla chunk'a bölünmesi | Semantic Coherence (-3), Contextual Completeness (-4) | Semantic chunking veya parent-child relationship kullan |
| **Metadata Inconsistency** | Chunk metadata'sının içerikle uyumsuz olması | Retrieval Efficiency (-3), Actionability (-2) | Otomatik metadata validation ve standardization |
| **PII Leakage** | Hassas verilerin chunk'larda kalması | Security Risk (-10), Production Blocker | PII detection ve otomatik redaction |

### 2.4. Gelişmiş Chunking Teknikleri: Small-to-Big Retrieval

**Parent-Child Relationship Mimarisi:**
> **Kod Durumu:** `Reference`
```typescript
interface ParentChildChunk {
  // Child chunk (arama için)
  childChunk: {
    id: string;
    content: string; // 200-400 token
    metadata: ChunkMetadata;
    embedding: number[];
  };
  
  // Parent chunk (LLM beslemesi için)
  parentChunk: {
    id: string;
    content: string; // 800-1200 token
    childIds: string[];
    metadata: ChunkMetadata;
  };
}
```

**Implementasyon Stratejisi:**
1. **Aşama:** Büyük dökümanları parent chunk'lara böl (800-1200 token)
2. **Aşama:** Her parent chunk'ı semantic olarak child chunk'lara böl (200-400 token)
3. **Aşama:** Sadece child chunk'ları index'le (hızlı arama)
4. **Aşama:** Retrieval sonrası ilgili parent chunk'ı LLM'e gönder

**Quality Impact:**
- **Retrieval Speed:** %30 daha hızlı (küçük chunk'lar)
- **Context Quality:** %40 daha iyi (tam parent context)
- **User Satisfaction:** %25 daha yüksek (relevant + complete answers)

### 2.5. Tablo ve Görsel Veri İşleme

**Tablo Preservation Stratejileri:**

| Veri Tipi | İşleme Yöntemi | Chunk'lama Stratejisi | Quality Puanı |
|-----------|----------------|---------------------|---------------|
| **HTML/Markdown Tabloları** | JSON'a çevir + structure preservation | Tabloyu tek chunk olarak koru | Retrieval: 8/10, Actionability: 9/10 |
| **CSV Verileri** | Row-based chunking + header preservation | Her N satırı chunk'la | Retrieval: 7/10, Actionability: 8/10 |
| **Kod Blokları** | Syntax highlighting preservation | Fonksiyon bazlı chunk'la | Retrieval: 9/10, Actionability: 9/10 |
| **Diyagramlar/Mermaid** | Text description + metadata | Açıklama ile birlikte chunk'la | Retrieval: 6/10, Actionability: 7/10 |

**Örnek Tablo Processing:**
> **Kod Durumu:** `Reference`
```typescript
function processTableChunk(tableHTML: string): Chunk {
  // 1. HTML'i structured JSON'a çevir
  const tableData = htmlTableToJson(tableHTML);
  
  // 2. Metadata'yı zenginleştir
  const metadata = {
    type: 'table',
    columns: tableData.columns,
    rowCount: tableData.rows.length,
    summary: generateTableSummary(tableData)
  };
  
  // 3. Hem orijinal hem structured formatı sakla
  const content = `${tableHTML}\n\nStructured Data:\n${JSON.stringify(tableData, null, 2)}`;
  
  return { content, metadata, type: 'table' };
}
```

**Örnek (Mükemmel):**
> **Kod Durumu:** `Reference`
```typescript
/**
 * User authentication service with JWT token management
 * Handles login, logout, and token refresh operations
 */
class AuthService {
  async login(credentials: LoginCredentials): Promise<AuthResult> {
    // Complete login logic with error handling
  }
  
  async refreshToken(token: string): Promise<string> {
    // Token refresh implementation
  }
}
```

**Kötü (0-2 puan):**
- Farklı konseptler karışık
- Cümleler arasında mantıksal bağlantı yok
- Terminoloji tutarsız
- Anlamsız parçalar

**Örnek (Kötü):**
> **Kod Durumu:** `Reference`
```typescript
// User auth + database config + UI components mixed
const config = { db: "mysql" };
function login() { /* ... */ }
<button>Click me</button>
export default App;
```

### 2.2. Contextual Completeness (Bağlamsal Bütünlük) - %25

**Mükemmel (9-10 puan):**
- Gerekli import'lar ve type'lar mevcut
- Bağımlılıklar açıkça belirtilmiş
- Error handling ve edge cases dahil
- Dokümantasyon tam ve güncel

**Örnek (Mükemmel):**
> **Kod Durumu:** `Reference`
```typescript
import { JwtService } from '@nestjs/jwt';
import { User } from '../entities/user.entity';
import { LoginCredentials, AuthResult } from '../types/auth.types';

/**
 * Authenticates user and returns JWT token
 * @throws UnauthorizedException for invalid credentials
 * @throws InternalServerErrorException for system errors
 */
async authenticateUser(credentials: LoginCredentials): Promise<AuthResult> {
  const user = await this.userService.findByEmail(credentials.email);
  if (!user || !await bcrypt.compare(credentials.password, user.password)) {
    throw new UnauthorizedException('Invalid credentials');
  }
  
  return {
    token: this.jwtService.sign({ sub: user.id }),
    user: { id: user.id, email: user.email }
  };
}
```

**Kötü (0-2 puan):**
- Import'lar eksik
- Type'lar belirtilmemiş
- Error handling yok
- Dokümantasyon eksik

**Örnek (Kötü):**
> **Kod Durumu:** `Reference`
```typescript
function login(data) {
  // What types? What imports? What errors?
  return result;
}
```

### 2.3. Retrieval Efficiency (Erişilebilirlik) - %25

**Mükemmel (9-10 puan):**
- Optimal chunk size (200-800 token)
- Anlamlı başlık ve metadata
- İyi keyword coverage
- Embedding-friendly içerik

**Örnek (Mükemmel):**
> **Kod Durumu:** `Reference`
```markdown
# TypeScript Generic Type Constraints

## Definition
Generic type constraints restrict type parameters to specific types or structures.

## Syntax Examples
```typescript
// Basic constraint
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(arg: T): T {
  console.log(arg.length);
  return arg;
}

// Keyof constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
> **Kod Durumu:** `Reference`
```

## Use Cases
- API response validation
- Database ORM patterns
- Type-safe utility functions

## Related Concepts
- Type inference
- Conditional types
- Mapped types
```

**Kötü (0-2 puan):**
- Çok uzun (>2000 token) veya çok kısa (<50 token)
- Anlamsız başlık
- Keyword eksikliği
- Embedding'e uygun olmayan içerik

**Örnek (Kötü):**
> **Kod Durumu:** `Reference`
```typescript
// No context, just random code
const x = 5;
function y() { return z; }
```

### 2.4. Actionability (Uygulanabilirlik) - %20

**Mükemmel (9-10 puan):**
- Pratik örnekler içerir
- Copy-paste çalışır kod
- Adım adım talimatlar
- Gerçek dünya senaryoları

**Örnek (Mükemmel):**
> **Kod Durumu:** `Reference`
```markdown
# Implementing Rate Limiting in Express.js

## Step 1: Install dependencies
```bash
npm install express-rate-limit
> **Kod Durumu:** `Reference`
```

## Step 2: Create rate limiter
```typescript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP'
});

app.use('/api/', limiter);
> **Kod Durumu:** `Reference`
```

## Step 3: Test the implementation
```bash
# Use curl to test rate limiting
for i in {1..105}; do curl http://localhost:3000/api/test; done
> **Kod Durumu:** `Reference`
```
```

**Kötü (0-2 puan):**
- Teorik bilgi sadece
- Çalışmayan kod örnekleri
- Uygulama adımları yok
- Soyut kavramlar

## 3. Spesifik Chunk Türleri İçin Kriterler

### 3.1. Kod Chunk'ları İçin

**İyi Chunk Özellikleri:**
- Tam fonksiyon/class/interface
- Type annotations dahil
- JSDoc comments
- Error handling
- Test examples

**Kötü Chunk Özellikleri:**
- Parçalanmış kod
- Type'lar olmadan
- Dokümantasyonsuz
- Edge cases yok

### 3.2. Dokümantasyon Chunk'ları İçin

**İyi Chunk Özellikleri:**
- Markdown formatında
- Code examples dahil
- API reference
- Use case açıklamaları
- Cross-references

**Kötü Chunk Özellikleri:**
- Düz metin
- Örnek yok
- Yapılandırılmamış
- Bağlantılar eksik

### 3.3. Example/Pattern Chunk'ları İçin

**İyi Chunk Özellikleri:**
- Problem-Solution formatı
- Before-After karşılaştırma
- Alternatif çözümler
- Trade-off açıklamaları

**Kötü Chunk Özellikleri:**
- Sadece çözüm
- Bağlam yok
- Alternatifler yok
- Neden açıklanmıyor

## 6. Advanced Ranking Strategies

### 6.1. Multi-Factor Ranking Formula

> **Kod Durumu:** `Reference`
```typescript
interface RankingFactors {
  similarityScore: number;     // Query-chunk similarity
  qualityScore: number;        // Chunk quality metrics
  recencyScore: number;        // Freshness score
  usageScore: number;          // Usage statistics
  diversityScore: number;      // Result diversity
}

interface RankingWeights {
  similarityWeight: number;
  qualityWeight: number;
  recencyWeight: number;
  usageWeight: number;
  diversityWeight: number;
}

class AdvancedRanker {
  rankChunks(
    chunks: Chunk[], 
    query: string, 
    weights?: Partial<RankingWeights>
  ): Chunk[] {
    const defaultWeights: RankingWeights = {
      similarityWeight: 0.5,
      qualityWeight: 0.2,
      recencyWeight: 0.15,
      usageWeight: 0.1,
      diversityWeight: 0.05
    };
    
    const finalWeights = { ...defaultWeights, ...weights };
    
    // Calculate scores for each chunk
    const scoredChunks = chunks.map(chunk => {
      const factors = this.calculateRankingFactors(chunk, query);
      const finalScore = this.calculateFinalScore(factors, finalWeights);
      
      return {
        ...chunk,
        rankingScore: finalScore,
        rankingFactors: factors
      };
    });
    
    // Apply diversity-aware ranking
    return this.applyDiversityRanking(scoredChunks);
  }
  
  private calculateRankingFactors(chunk: Chunk, query: string): RankingFactors {
    return {
      similarityScore: this.calculateSimilarity(query, chunk),
      qualityScore: chunk.qualityScore.overall,
      recencyScore: this.calculateRecencyScore(chunk),
      usageScore: this.calculateUsageScore(chunk),
      diversityScore: 0 // Will be calculated globally
    };
  }
  
  private calculateFinalScore(factors: RankingFactors, weights: RankingWeights): number {
    return (
      factors.similarityScore * weights.similarityWeight +
      factors.qualityScore * weights.qualityWeight +
      factors.recencyScore * weights.recencyWeight +
      factors.usageScore * weights.usageWeight +
      factors.diversityScore * weights.diversityWeight
    );
  }
  
  private applyDiversityRanking(scoredChunks: Chunk[]): Chunk[] {
    // Maximal Marginal Relevance (MMR) algorithm
    const selected: Chunk[] = [];
    const remaining = [...scoredChunks];
    
    while (selected.length < 10 && remaining.length > 0) {
      // Select highest scoring chunk
      const bestChunk = remaining.reduce((best, current) => 
        current.rankingScore > best.rankingScore ? current : best
      );
      
      selected.push(bestChunk);
      remaining.splice(remaining.indexOf(bestChunk), 1);
      
      // Reduce scores of similar chunks
      remaining.forEach(chunk => {
        const similarity = this.calculateChunkSimilarity(bestChunk, chunk);
        chunk.rankingScore *= (1 - similarity * 0.5); // Diversity penalty
      });
    }
    
    return selected;
  }
}
```

### 6.2. Context-Aware Ranking

> **Kod Durumu:** `Reference`
```typescript
interface ContextAwareRanking {
  userSkillLevel: 'beginner' | 'intermediate' | 'expert';
  taskType: 'learning' | 'debugging' | 'implementation' | 'reference';
  timeConstraint: boolean;
  qualityPreference: 'accuracy' | 'speed' | 'completeness';
}

class ContextAwareRanker extends AdvancedRanker {
  rankWithContext(
    chunks: Chunk[],
    query: string,
    context: ContextAwareRanking
  ): Chunk[] {
    const weights = this.getContextualWeights(context);
    return this.rankChunks(chunks, query, weights);
  }
  
  private getContextualWeights(context: ContextAwareRanking): Partial<RankingWeights> {
    const baseWeights: Partial<RankingWeights> = {};
    
    // Adjust based on skill level
    switch (context.userSkillLevel) {
      case 'beginner':
        baseWeights.qualityWeight = 0.3; // Prefer high-quality, actionable content
        baseWeights.diversityWeight = 0.1;
        break;
      case 'expert':
        baseWeights.similarityWeight = 0.6; // Prefer highly relevant technical content
        baseWeights.recencyWeight = 0.2;
        break;
    }
    
    // Adjust based on task type
    switch (context.taskType) {
      case 'debugging':
        baseWeights.recencyWeight = 0.25; // Prefer recent, updated content
        baseWeights.qualityWeight = 0.25;
        break;
      case 'learning':
        baseWeights.diversityWeight = 0.15; // Prefer diverse examples
        baseWeights.qualityWeight = 0.3;
        break;
    }
    
    // Adjust based on time constraint
    if (context.timeConstraint) {
      baseWeights.similarityWeight = 0.6; // Prioritize relevance
      baseWeights.qualityWeight = 0.15;
    }
    

### 7.1. Cost Metrics and Budgeting

```typescript
interface CostMetrics {
  embeddingCostPerChunk: number;
  avgQueryCost: number;
  storageCostPerMonth: number;
  totalMonthlyCost: number;
  costPerSuccessfulQuery: number;
}

interface PerformanceBudget {
  maxLatencyMs: number;
  maxCostPerQuery: number;
  maxStorageGb: number;
  minAccuracyThreshold: number;
}

class CostOptimizer {
  private readonly EMBEDDING_COST_PER_1K_TOKENS = 0.00002; // OpenAI pricing
  private readonly STORAGE_COST_PER_GB_MONTH = 0.23; // Vector database pricing
  
  calculateCostMetrics(chunks: Chunk[], queryStats: QueryStats): CostMetrics {
    // 1. Embedding costs
    const totalTokens = chunks.reduce((sum, chunk) => sum + chunk.metadata.tokenCount, 0);
    const embeddingCost = (totalTokens / 1000) * this.EMBEDDING_COST_PER_1K_TOKENS;
    const embeddingCostPerChunk = embeddingCost / chunks.length;
    
    // 2. Query costs
    const avgQueryCost = queryStats.totalCost / queryStats.totalQueries;
    
    // 3. Storage costs
    const storageSizeGB = this.estimateStorageSize(chunks);
    const storageCostPerMonth = storageSizeGB * this.STORAGE_COST_PER_GB_MONTH;
    
    // 4. Total monthly cost
    const totalMonthlyCost = embeddingCost + storageCostPerMonth + (queryStats.totalCost * 30);
    
    // 5. Cost per successful query
    const costPerSuccessfulQuery = totalMonthlyCost / queryStats.successfulQueries;
    
    return {
      embeddingCostPerChunk: embeddingCostPerChunk,
      avgQueryCost: avgQueryCost,
      storageCostPerMonth: storageCostPerMonth,
      totalMonthlyCost: totalMonthlyCost,
      costPerSuccessfulQuery: costPerSuccessfulQuery
    };
  }
  
  optimizeForBudget(chunks: Chunk[], budget: PerformanceBudget): OptimizationResult {
    const optimizations: OptimizationStrategy[] = [];
    
    // 1. Check latency budget
    const currentLatency = this.estimateLatency(chunks.length);
    if (currentLatency > budget.maxLatencyMs) {
      optimizations.push({
        type: 'reduceChunkCount',
        description: 'Reduce chunk count to improve latency',
        impact: 'High',
        implementation: 'Filter low-quality chunks or increase chunk size'
      });
    }
    
    // 2. Check cost budget
    const currentCost = this.calculateCostMetrics(chunks, this.getQueryStats());
    if (currentCost.avgQueryCost > budget.maxCostPerQuery) {
      optimizations.push({
        type: 'switchEmbeddingModel',
        description: 'Switch to cost-effective embedding model',
        impact: 'Medium',
        implementation: 'Use smaller embedding model or local model'
      });
    }
    
    // 3. Check storage budget
    const currentStorage = this.estimateStorageSize(chunks);
    if (currentStorage > budget.maxStorageGb) {
      optimizations.push({
        type: 'compressEmbeddings',
        description: 'Compress embeddings or reduce dimensionality',
        impact: 'Medium',
        implementation: 'Use PCA or quantization'
      });
    }
    
    return {
      currentMetrics: currentCost,
      budgetViolations: optimizations,
      recommendedActions: optimizations
    };
  }
  
  private estimateStorageSize(chunks: Chunk[]): number {
    // Estimate storage size in GB
    const avgChunkSizeKB = 50; // Average chunk size with metadata
    const totalSizeKB = chunks.length * avgChunkSizeKB;
    return totalSizeKB / (1024 * 1024); // Convert to GB
  }
  
  private estimateLatency(chunkCount: number): number {
    // Estimate query latency based on chunk count
    const baseLatency = 50; // Base processing time
    const perChunkLatency = 2; // Additional time per chunk
    return baseLatency + (chunkCount * perChunkLatency);
  }
}
> **Kod Durumu:** `Reference`
```

### 7.2. Performance Optimization Strategies

| Strategy | Cost Impact | Performance Impact | Implementation Complexity |
|----------|-------------|-------------------|-------------------------|
| **Chunk Size Optimization** | Low | High | Medium |
| **Embedding Model Switch** | Medium | Medium | Low |
| **Vector Compression** | Low | Medium | High |
| **Caching Layer** | Medium | High | Medium |
| **Batch Processing** | Low | Medium | Low |
| **Approximate Search** | Low | High | High |
| **Knowledge Graph Embeddings** | Medium | High | High |
| **Entity Disambiguation** | Low | Medium | Medium |
| **Coreference Resolution** | Low | Medium | High |
| **Named Entity Recognition** | Low | Medium | Medium |
| **Part-of-Speech Tagging** | Low | Medium | Medium |
| **Dependency Parsing** | Low | Medium | High |
| **Semantic Role Labeling** | Low | Medium | High |
```

---

## 8. Feedback-Driven Retraining Pipeline

### **RAG v1 (Eski):**
> **Kod Durumu:** `Reference`
```
query → retrieve → answer ❌
```

### **RAG v2 (Modern):**
> **Kod Durumu:** `Reference`
```
query → retrieve → reason → validate → answer ✅
```

---

## 🚀 **Implementation Priority**

### **Faz 1 - Critical:**
1. **Reasoning Layer** - Chunk birleştirme
2. **Wrong-but-Relevant Detection** - Validation

### **Faz 2 - Important:**
3. **Answer Validation Pipeline** - Quality control
4. **Chunk → Answer Traceability** - Observability

### **Faz 3 - Advanced:**
5. **Multi-Hop Reasoning** - Complex queries
6. **Confidence Calibration** - Reliability scoring

---

## 🎯 **NET SONUÇ**

Bu katmanları eklediğinde RAG sistemi:

❌ "Bilgi bulur" 
✅ "Bilgi bulur, doğrular, birleştirir ve garanti eder"

**Upgrade Etkisi:**
- **Reliability:** 6 → 9.5
- **User Trust:** 5 → 9.0
- **Production Ready:** 6 → 9.2

**Artık bu RAG sistemi:**
- ❌ Simple retrieval
- ✅ Intelligent reasoning engine
## 8. Feedback-Driven Retraining Pipeline

### 8.1. Hard Negative Mining

> **Kod Durumu:** `Reference`
```typescript
interface HardNegative {
  query: string;
  negativeChunk: Chunk;
  reason: 'lowSimilarity' | 'wrongIntent' | 'outdated' | 'poorQuality';
  difficulty: 'easy' | 'medium' | 'hard';
}

class HardNegativeMiner {
  async mineHardNegatives(queryLogs: QueryLog[]): Promise<HardNegative[]> {
    const hardNegatives: HardNegative[] = [];
    
    for (const log of queryLogs) {
      // 1. Find queries with low user satisfaction
      if (log.userRating < 3) {
        // 2. Analyze why retrieval failed
        const failureAnalysis = await this.analyzeFailure(log);
        
        // 3. Extract hard negatives
        const negatives = await this.extractHardNegatives(log, failureAnalysis);
        hardNegatives.push(...negatives);
      }
    }
    
    return hardNegatives;
  }
  
  private async extractHardNegatives(log: QueryLog, analysis: FailureAnalysis): Promise<HardNegative[]> {
    const negatives: HardNegative[] = [];
    
    // Extract chunks that were retrieved but should not have been
    for (const chunk of log.retrievedChunks) {
      if (this.shouldBeNegative(chunk, analysis)) {
        negatives.push({
          query: log.query,
          negativeChunk: chunk,
          reason: this.determineNegativeReason(chunk, analysis),
          difficulty: this.assessDifficulty(chunk, log)
        });
      }
    }
    
    return negatives;
  }
}
```

### 8.2. Re-ranking Dataset Generation

> **Kod Durumu:** `Reference`
```typescript
interface RerankingExample {
  query: string;
  positiveChunks: Chunk[];
  negativeChunks: Chunk[];
  idealRanking: string[];
  userFeedback: UserFeedback;
}

class RerankingDatasetGenerator {
  generateDataset(queryLogs: QueryLog[], hardNegatives: HardNegative[]): RerankingExample[] {
    const dataset: RerankingExample[] = [];
    
    for (const log of queryLogs) {
      if (this.isValidTrainingExample(log)) {
        const example = this.createRerankingExample(log, hardNegatives);
        dataset.push(example);
      }
    }
    
    return dataset;
  }
  
  private createRerankingExample(log: QueryLog, hardNegatives: HardNegative[]): RerankingExample {
    // 1. Identify positive chunks (user clicked/satisfied)
    const positiveChunks = log.retrievedChunks.filter(chunk => 
      log.userFeedback.clickedChunks.includes(chunk.id)
    );
    
    // 2. Identify negative chunks (user skipped/unsatisfied)
    const negativeChunks = log.retrievedChunks.filter(chunk => 
      !log.userFeedback.clickedChunks.includes(chunk.id)
    );
    
    // 3. Add hard negatives
    const additionalNegatives = hardNegatives
      .filter(neg => neg.query === log.query)
      .map(neg => neg.negativeChunk);
    
    // 4. Create ideal ranking based on user behavior
    const idealRanking = this.createIdealRanking(log, positiveChunks, negativeChunks);
    
    return {
      query: log.query,
      positiveChunks: positiveChunks,
      negativeChunks: [...negativeChunks, ...additionalNegatives],
      idealRanking: idealRanking,
      userFeedback: log.userFeedback
    };
  }
}
```

### 4.1. Anotatör Talimatları

**Görev:** Chunk'ları 0-10 skalasında değerlendirin, her boyut için ayrı puan verin.

**Değerlendirme Süreci:**
1. Chunk'ı okuyup genel anlayışı kazanın
2. Her boyut için spesifik kriterlere göre puanlayın
3. Notlarınızı ve gerekçelerinizi ekleyin
4. Borderline durumları işaretleyin

**Inter-Annotator Agreement Hedefi:** Cohen's κ ≥ 0.7

### 4.2. Örnek Vakalar

**Vaka 1: Semantic Coherence 9/10**
> **Kod Durumu:** `Reference`
```typescript
// ✅ Tek konsept odaklı, mantıksal akış
class UserService {
  async createUser(data: CreateUserDto): Promise<User> {
    // Complete implementation with error handling
  }
}
```

**Vaka 2: Semantic Coherence 3/10**
> **Kod Durumu:** `Reference`
```typescript
// ❌ Farklı konseptler karışık
const config = { db: "mysql" };
function login() { /* ... */ }
<button>Click me</button>;
```

**Vaka 3: Borderline - Contextual Completeness 6/10**
> **Kod Durumu:** `Reference`
```typescript
// ⚠️ Temel context var ama eksiklikler mevcut
function authenticateUser(email: string, password: string) {
  // Type annotations eksik, error handling yok
  return { token: "jwtToken" };
}
```

## 17. Observability & Trace-Level Debugging

### 17.1. End-to-End Request Tracing

**Complete Request Lifecycle Tracking:**
> **Kod Durumu:** `Reference`
```typescript
interface RAGTrace {
  traceId: string;
  timestamp: Date;
  query: string;
  stages: TraceStage[];
  performance: PerformanceMetrics;
  quality: QualityMetrics;
  errors: ErrorInfo[];
  userFeedback?: UserFeedback;
}

interface TraceStage {
  stageName: string;
  startTime: Date;
  endTime: Date;
  durationMs: number;
  input: any;
  output: any;
  metadata: Record<string, any>;
  subStages?: TraceStage[];
}

interface PerformanceMetrics {
  totalDurationMs: number;
  stageDurations: Record<string, number>;
  tokenUsage: {
    inputTokens: number;
    outputTokens: number;
    totalTokens: number;
  };
  memoryUsageMb: number;
  cacheHitRate: number;
}

interface QualityMetrics {
  queryUnderstandingConfidence: number;
  retrievalRelevanceScore: number;
  contextUtilizationRatio: number;
  responseConfidence: number;
  hallucinationRiskScore: number;
  overallQualityScore: number;
}

class RAGTracer {
  private traces: Map<string, RAGTrace> = new Map();
  
  async traceRequest(query: string): Promise<string> {
    const traceId = this.generateTraceId();
    const trace: RAGTrace = {
      traceId: traceId,
      timestamp: new Date(),
      query,
      stages: [],
      performance: {
        totalDurationMs: 0,
        stageDurations: {},
        tokenUsage: { inputTokens: 0, outputTokens: 0, totalTokens: 0 },
        memoryUsageMb: 0,
        cacheHitRate: 0
      },
      quality: {
        queryUnderstandingConfidence: 0,
        retrievalRelevanceScore: 0,
        contextUtilizationRatio: 0,
        responseConfidence: 0,
        hallucinationRiskScore: 0,
        overallQualityScore: 0
      },
      errors: []
    };
    
    this.traces.set(traceId, trace);
    return traceId;
  }
  
  async traceStage<T, R>(
    traceId: string,
    stageName: string,
    operation: () => Promise<R>,
    input?: T
  ): Promise<R> {
    const trace = this.traces.get(traceId);
    if (!trace) throw new Error(`Trace ${traceId} not found`);
    
    const startTime = new Date();
    const stage: TraceStage = {
      stageName: stageName,
      startTime: startTime,
      endTime: new Date(), // Will be updated
      durationMs: 0, // Will be updated
      input,
      output: null, // Will be updated
      metadata: {}
    };
    
    try {
      const result = await operation();
      const endTime = new Date();
      
      stage.endTime = endTime;
      stage.durationMs = endTime.getTime() - startTime.getTime();
      stage.output = result;
      
      // Update trace
      trace.stages.push(stage);
      trace.performance.stageDurations[stageName] = stage.durationMs;
      trace.performance.totalDurationMs += stage.durationMs;
      
      this.traces.set(traceId, trace);
      return result;
    } catch (error) {
      const endTime = new Date();
      
      stage.endTime = endTime;
      stage.durationMs = endTime.getTime() - startTime.getTime();
      stage.metadata.error = error;
      
      trace.stages.push(stage);
      trace.errors.push({
        stage: stageName,
        error: error.message,
        stackTrace: error.stack,
        timestamp: endTime
      });
      
      this.traces.set(traceId, trace);
      throw error;
    }
  }
  
  getTrace(traceId: string): RAGTrace | null {
    return this.traces.get(traceId) || null;
  }
  
  async generateTraceReport(traceId: string): Promise<TraceReport> {
    const trace = this.traces.get(traceId);
    if (!trace) throw new Error(`Trace ${traceId} not found`);
    
    return {
      summary: this.generateSummary(trace),
      performanceAnalysis: this.analyzePerformance(trace),
      qualityAnalysis: this.analyzeQuality(trace),
      errorAnalysis: this.analyzeErrors(trace),
      recommendations: this.generateRecommendations(trace)
    };
  }
  
  private generateSummary(trace: RAGTrace): string {
    const totalDuration = trace.performance.totalDurationMs;
    const errorCount = trace.errors.length;
    const avgQuality = trace.quality.overallQualityScore;
    
    return `
      Query: "${trace.query}"
      Duration: ${totalDuration}ms
      Errors: ${errorCount}
      Quality Score: ${avgQuality.toFixed(2)}/10
      Status: ${errorCount > 0 ? 'FAILED' : avgQuality > 7 ? 'SUCCESS' : 'NEEDS_IMPROVEMENT'}
    `;
  }
  
  private analyzePerformance(trace: RAGTrace): PerformanceAnalysis {
    const stages = trace.stages;
    const bottlenecks = stages
      .sort((a, b) => b.durationMs - a.durationMs)
      .slice(0, 3);
    
    const cacheHitRate = trace.performance.cacheHitRate;
    const tokenEfficiency = trace.performance.tokenUsage.totalTokens / trace.performance.totalDurationMs;
    
    return {
      slowestStages: bottlenecks,
      cacheEfficiency: cacheHitRate > 0.7 ? 'good' : cacheHitRate > 0.4 ? 'moderate' : 'poor',
      tokenEfficiency: tokenEfficiency > 1 ? 'good' : tokenEfficiency > 0.5 ? 'moderate' : 'poor',
      optimizationSuggestions: this.generatePerformanceSuggestions(bottlenecks, cacheHitRate)
    };
  }
  
  private analyzeQuality(trace: RAGTrace): QualityAnalysis {
    const quality = trace.quality;
    
    return {
      queryUnderstanding: quality.queryUnderstandingConfidence > 0.8 ? 'excellent' : 
                         quality.queryUnderstandingConfidence > 0.6 ? 'good' : 'needsImprovement',
      retrievalQuality: quality.retrievalRelevanceScore > 0.8 ? 'excellent' :
                      quality.retrievalRelevanceScore > 0.6 ? 'good' : 'needsImprovement',
      responseQuality: quality.responseConfidence > 0.8 ? 'excellent' :
                    quality.responseConfidence > 0.6 ? 'good' : 'needsImprovement',
      hallucinationRisk: quality.hallucinationRiskScore < 0.2 ? 'low' :
                        quality.hallucinationRiskScore < 0.5 ? 'moderate' : 'high',
      overallAssessment: quality.overallQualityScore > 8 ? 'excellent' :
                        quality.overallQualityScore > 6 ? 'good' : 'needsImprovement'
    };
  }
  
  private analyzeErrors(trace: RAGTrace): ErrorAnalysis {
    const errors = trace.errors;
    const errorTypes = errors.reduce((types, error) => {
      types[error.stage] = (types[error.stage] || 0) + 1;
      return types;
    }, {} as Record<string, number>);
    
    return {
      totalErrors: errors.length,
      errorTypes,
      mostProblematicStage: Object.entries(errorTypes)
        .sort(([,a], [,b]) => b - a)[0]?.[0] || 'none',
      errorPatterns: this.identifyErrorPatterns(errors),
      suggestedFixes: this.generateErrorFixes(errors)
    };
  }
  
  private generateRecommendations(trace: RAGTrace): string[] {
    const recommendations: string[] = [];
    
    // Performance recommendations
    if (trace.performance.totalDurationMs > 5000) {
      recommendations.push('Consider optimizing chunk selection or implementing caching');
    }
    
    // Quality recommendations
    if (trace.quality.overallQualityScore < 6) {
      recommendations.push('Review chunk quality thresholds and improve query understanding');
    }
    
    // Error recommendations
    if (trace.errors.length > 0) {
      recommendations.push('Investigate error-prone stages and implement better error handling');
    }
    
    // Hallucination recommendations
    if (trace.quality.hallucinationRiskScore > 0.5) {
      recommendations.push('Implement stricter answer validation and fact-checking');
    }
    
    return recommendations;
  }
}

interface TraceReport {
  summary: string;
  performanceAnalysis: PerformanceAnalysis;
  qualityAnalysis: QualityAnalysis;
  errorAnalysis: ErrorAnalysis;
  recommendations: string[];
}

interface PerformanceAnalysis {
  slowestStages: TraceStage[];
  cacheEfficiency: 'good' | 'moderate' | 'poor';
  tokenEfficiency: 'good' | 'moderate' | 'poor';
  optimizationSuggestions: string[];
}

interface QualityAnalysis {
  queryUnderstanding: 'excellent' | 'good' | 'needsImprovement';
  retrievalQuality: 'excellent' | 'good' | 'needsImprovement';
  responseQuality: 'excellent' | 'good' | 'needsImprovement';
  hallucinationRisk: 'low' | 'moderate' | 'high';
  overallAssessment: 'excellent' | 'good' | 'needsImprovement';
}

interface ErrorAnalysis {
  totalErrors: number;
  errorTypes: Record<string, number>;
  mostProblematicStage: string;
  errorPatterns: string[];
  suggestedFixes: string[];
}

interface ErrorInfo {
  stage: string;
  error: string;
  stackTrace?: string;
  timestamp: Date;
}
```

### 17.2. Real-time Monitoring Dashboard

**Live System Health Monitoring:**
> **Kod Durumu:** `Reference`
```typescript
interface DashboardMetrics {
  systemHealth: SystemHealth;
  performanceMetrics: RealTimePerformance;
  qualityMetrics: RealTimeQuality;
  alertStatus: AlertStatus;
  userSatisfaction: UserSatisfactionMetrics;
}

interface SystemHealth {
  overallStatus: 'healthy' | 'degraded' | 'critical';
  componentStatus: Record<string, 'healthy' | 'degraded' | 'down'>;
  uptimePercentage: number;
  lastRestart: Date;
  activeTraces: number;
}

interface RealTimePerformance {
  avgResponseTimeMs: number;
  p95_response_time_ms: number;
  p99_response_time_ms: number;
  requestsPerSecond: number;
  cacheHitRate: number;
  errorRate: number;
  tokenUtilization: number;
}

interface RealTimeQuality {
  avgQualityScore: number;
  avgRelevanceScore: number;
  hallucinationRate: number;
  userSatisfactionRating: number;
  successfulQueriesRate: number;
}

class RAGDashboard {
  private metrics: DashboardMetrics;
  private alerts: Alert[] = [];
  
  async updateMetrics(): Promise<void> {
    // Gather real-time metrics from various sources
    const [systemHealth, performance, quality, satisfaction] = await Promise.all([
      this.gatherSystemHealth(),
      this.gatherPerformanceMetrics(),
      this.gatherQualityMetrics(),
      this.gatherUserSatisfaction()
    ]);
    
    this.metrics = {
      systemHealth: systemHealth,
      performanceMetrics: performance,
      qualityMetrics: quality,
      alertStatus: {
        activeAlerts: this.alerts.filter(a => a.active).length,
        criticalAlerts: this.alerts.filter(a => a.severity === 'critical').length,
        lastAlert: this.alerts.length > 0 ? this.alerts[this.alerts.length - 1] : null
      },
      userSatisfaction: satisfaction
    };
    
    // Check for alert conditions
    await this.checkAlertConditions();
  }
  
  private async checkAlertConditions(): Promise<void> {
    const { performanceMetrics, qualityMetrics, systemHealth } = this.metrics;
    
    // Performance alerts
    if (performanceMetrics.p95_response_time_ms > 2000) {
      await this.createAlert({
        type: 'performanceDegradation',
        severity: 'high',
        message: 'P95 response time exceeded 2 seconds',
        metricValue: performanceMetrics.p95_response_time_ms,
        threshold: 2000
      });
    }
    
    // Quality alerts
    if (qualityMetrics.hallucinationRate > 0.3) {
      await this.createAlert({
        type: 'qualityDegradation',
        severity: 'critical',
        message: 'Hallucination rate exceeded 30%',
        metricValue: qualityMetrics.hallucinationRate,
        threshold: 0.3
      });
    }
    
    // System health alerts
    if (systemHealth.uptimePercentage < 99) {
      await this.createAlert({
        type: 'systemUnhealthy',
        severity: 'critical',
        message: 'System uptime below 99%',
        metricValue: systemHealth.uptimePercentage,
        threshold: 99
      });
    }
  }
  
  private async createAlert(alert: Omit<Alert, 'id' | 'timestamp'>): Promise<void> {
    const fullAlert: Alert = {
      ...alert,
      id: this.generateAlertId(),
      timestamp: new Date(),
      active: true
    };
    
    this.alerts.push(fullAlert);
    
    // Send notification
    await this.sendNotification(fullAlert);
  }
  
  getMetricsSnapshot(): DashboardMetrics {
    return { ...this.metrics };
  }
}

interface Alert {
  id: string;
  type: string;
  severity: 'low' | 'medium' | 'high' | 'critical';
  message: string;
  metricValue: number;
  threshold: number;
  timestamp: Date;
  active: boolean;
}

interface AlertStatus {
  activeAlerts: number;
  criticalAlerts: number;
  lastAlert: Alert | null;
}
```

### 15.1. Subject Matter Expert (SME) Workflow

**Kritik Veri Onay Süreci:**
> **Kod Durumu:** `Reference`
```typescript
interface SMEApproval {
  chunkId: string;
  smeId: string;
  smeExpertise: string[];
  approvalStatus: 'approved' | 'rejected' | 'needsRevision';
  approvalNotes: string;
  confidenceLevel: number; // 0-1
  timestamp: Date;
  revisionSuggestions?: string[];
}

class SMEWorkflowManager {
  async submitForSMEApproval(
    chunk: Chunk,
    smeExpertise: string[]
  ): Promise<SMEApproval> {
    // 1. Find appropriate SME
    const assignedSME = await this.findSME(smeExpertise);
    
    // 2. Create approval request
    const approvalRequest = {
      chunkId: chunk.id,
      smeId: assignedSME.id,
      smeExpertise: smeExpertise,
      approvalStatus: 'pending',
      confidenceLevel: 0,
      timestamp: new Date()
    };
    
    // 3. Notify SME
    await this.notifySME(assignedSME, {
      chunk,
      deadline: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
      priority: this.calculatePriority(chunk)
    });
    
    return approvalRequest;
  }
  
  async processSMEApproval(
    approvalId: string,
    decision: 'approved' | 'rejected' | 'needsRevision',
    notes: string,
    confidence: number
  ): Promise<void> {
    const approval = await this.getApproval(approvalId);
    
    approval.approvalStatus = decision;
    approval.approvalNotes = notes;
    approval.confidenceLevel = confidence;
    approval.timestamp = new Date();
    
    // Update chunk based on decision
    if (decision === 'approved') {
      await this.markChunkAsApproved(approval.chunkId, approval);
    } else if (decision === 'rejected') {
      await this.markChunkAsRejected(approval.chunkId, approval);
    } else {
      // Request revision
      await this.requestRevision(approval.chunkId, approval);
    }
    
    // Log decision for audit trail
    await this.logSMEDecision(approval);
  }
  
  private calculatePriority(chunk: Chunk): 'high' | 'medium' | 'low' {
    // Priority calculation based on chunk impact
    if (chunk.metadata.sensitivityLevel === 'restricted') return 'high';
    if (chunk.usageStats.retrievalCount > 100) return 'medium';
    return 'low';
  }
}
```

**SME Approval Dashboard:**
| Metric | Threshold | Alert Action |
|--------|------------|-------------|
| **Approval Time** | > 24 saat | Escalate to senior SME |
| **Rejection Rate** | > 20% | Review quality guidelines |
| **Revision Requests** | > 3/chunk | Auto-flag for review |
| **SME Workload** | > 10 pending | Redistribute requests |

### 15.2. Anotatör Eğitim Seti (20 Örnek)

**Etiketleme Rehberi ve Standartları:**
- **Inter-Annotator Agreement Hedefi:** Cohen's κ ≥ 0.7
- **Eğitim Süresi:** Minimum 2 saat pratik
- **Kalibrasyon:** Her anotatör aynı 20 örnek üzerinden değerlendirme

**Örnek Chunk Seti:**

**Mükemmel Örnekler (9-10 puan):**
> **Kod Durumu:** `Reference`
```typescript
// Örnek 1: Semantic Coherence = 10/10
/**
 * JWT Token Service
 * Handles JWT token creation, validation, and refresh
 * @example
 * const token = await jwtService.createToken({userId: '123'});
 */
class JWTService {
  async createToken(payload: TokenPayload): Promise<string> {
    return jwt.sign(payload, process.env.JWT_SECRET, {expiresIn: '1h'});
  }
  
  async validateToken(token: string): Promise<TokenPayload | null> {
    try {
      return jwt.verify(token, process.env.JWT_SECRET) as TokenPayload;
    } catch {
      return null;
    }
  }
}
```

**İyi Örnekler (7-8 puan):**
> **Kod Durumu:** `Reference`
```typescript
// Örnek 2: Semantic Coherence = 8/10, Contextual = 7/10
function calculateDiscount(price: number, percentage: number): number {
  // Calculate discount amount
  const discountAmount = price * (percentage / 100);
  return price - discountAmount;
}
```

**Orta Örnekler (5-6 puan):**
> **Kod Durumu:** `Reference`
```typescript
// Örnek 3: Semantic Coherence = 6/10, Actionability = 5/10
// User management
class UserManager {
  users: User[] = [];
  
  addUser(user: User) {
    this.users.push(user);
  }
  
  getUser(id: string) {
    return this.users.find(u => u.id === id);
  }
}
```

**Zayıf Örnekler (3-4 puan):**
> **Kod Durumu:** `Reference`
```typescript
// Örnek 4: Semantic Coherence = 3/10, Contextual = 2/10
const config = { db: "mysql", port: 3000 };
function login() { /* TODO */ }
export default App;
```

**Borderline Vakalar:**
> **Kod Durumu:** `Reference`
```typescript
// Örnek 5: Semantic Coherence = 7/10, Contextual = 6/10 (Borderline)
// API response handler
async function handleRequest(req: Request) {
  const data = await fetchData();
  return Response.json(data);
}
// Neden borderline: Error handling eksik, type annotations kısmi
```

**Etiketleme Kontrol Listesi:**
- [ ] Tek konsept odaklı mı?
- [ ] Mantıksal akış var mı?
- [ ] Gerekli import'lar var mı?
- [ ] Type annotations tam mı?
- [ ] Error handling var mı?
- [ ] Çalıştırılabilir örnek var mı?
- [ ] Boyut uygun mu (200-800 token)?
- [ ] Metadata anlaşılır mı?

### 15.3. Anotatör Talimatları

**Görev:** Chunk'ları 0-10 skalasında değerlendirin, her boyut için ayrı puan verin.

**Değerlendirme Süreci:**
1. Chunk'ı okuyup genel anlayışı kazanın
2. Her boyut için spesifik kriterlere göre puanlayın
3. Notlarınızı ve gerekçelerinizi ekleyin
4. Borderline durumları işaretleyin

**Inter-Annotator Agreement Hedefi:** Cohen's κ ≥ 0.7

## A. Appendix: Detaylı Otomatik Değerlendirme Script'i

### A.1. Complete Implementation

> **Kod Durumu:** `Reference`
```typescript
interface ChunkQualityMetrics {
  semanticCoherence: number;
  contextualCompleteness: number;
  retrievalEfficiency: number;
  actionability: number;
  overallScore: number;
}

class ChunkQualityAnalyzer {
  private embeddingModel: string = 'text-embedding-ada-002';
  private tokenizer: TiktokenTokenizer;
  
  constructor() {
    this.tokenizer = new TiktokenTokenizer('cl100k_base');
  }
  
  analyzeChunk(chunk: string, metadata: ChunkMetadata): ChunkQualityMetrics {
    const metrics: ChunkQualityMetrics = {
      semanticCoherence: this.analyzeSemanticCoherence(chunk),
      contextualCompleteness: this.analyzeContextualCompleteness(chunk, metadata),
      retrievalEfficiency: this.analyzeRetrievalEfficiency(chunk),
      actionability: this.analyzeActionability(chunk),
      overallScore: 0
    };
    
    metrics.overallScore = 
      metrics.semanticCoherence * 0.3 +
      metrics.contextualCompleteness * 0.25 +
      metrics.retrievalEfficiency * 0.25 +
      metrics.actionability * 0.2;
    
    return metrics;
  }
  
  private analyzeSemanticCoherence(chunk: string): number {
    // Topic consistency analysis using BERTopic
    const topics = this.extractTopics(chunk);
    const topicCoherence = this.calculateTopicCoherence(topics);
    
    // Terminology consistency
    const terminologyScore = this.checkTerminologyConsistency(chunk);
    
    // Language model coherence scoring
    const lmCoherence = this.calculateLMCoherence(chunk);
    
    return (topicCoherence * 0.4 + terminologyScore * 0.3 + lmCoherence * 0.3);
  }
  
  private analyzeContextualCompleteness(chunk: string, metadata: ChunkMetadata): number {
    let score = 0;
    
    // Import presence check (30%)
    const hasImports = /import.*from/.test(chunk);
    score += hasImports ? 3 : 0;
    
    // Type annotation coverage (30%)
    const typeAnnotations = this.countTypeAnnotations(chunk);
    score += Math.min(typeAnnotations / 10, 3);
    
    // Error handling presence (20%)
    const hasErrorHandling = /try|catch|throw|Error/.test(chunk);
    score += hasErrorHandling ? 2 : 0;
    
    // Documentation completeness (20%)
    const hasDocs = /\*\*[\s\S]*?\*/|\/\/[\s\S]*?\n/.test(chunk);
    score += hasDocs ? 2 : 0;
    
    return score;
  }
  
  private analyzeRetrievalEfficiency(chunk: string): number {
    // Token count analysis (40%)
    const tokens = this.tokenizer.encode(chunk).length;
    const optimalTokens = tokens >= 200 && tokens <= 800;
    const tokenScore = optimalTokens ? 4 : Math.max(0, 4 - Math.abs(tokens - 500) / 200);
    
    // Keyword density (30%)
    const keywords = this.extractKeywords(chunk);
    const keywordScore = Math.min(keywords.length / 5, 3);
    
    // Metadata quality (30%)
    const metadataScore = this.evaluateMetadata(chunk);
    
    return tokenScore + keywordScore + metadataScore;
  }
  
  private analyzeActionability(chunk: string): number {
    let score = 0;
    
    // Code example presence (40%)
    const hasCodeExamples = /```typescript|```javascript|```js/.test(chunk);
    score += hasCodeExamples ? 4 : 0;
    
    // Step-by-step instructions (30%)
    const hasSteps = /\d+\.|Step \d+|First|Second|Third/.test(chunk);
    score += hasSteps ? 3 : 0;
    
    // Copy-paste readiness (30%)
    const isRunnable = this.checkRunnableCode(chunk);
    score += isRunnable ? 3 : 0;
    
    return score;
  }
  
  private extractTopics(chunk: string): string[] {
    // BERTopic implementation with concrete parameters
    const topicModel = new BERTopic({
      embeddingModel: 'all-MiniLM-L6-v2',
      minTopicSize: 2,
      verbose: false
    });
    
    const topics = topicModel.fitTransform([chunk]);
    return topics[0] || [];
  }
  
  private calculateTopicCoherence(topics: string[]): number {
    // Topic coherence using NPMI (Normalized Pointwise Mutual Information)
    if (topics.length === 0) return 0.5;
    if (topics.length === 1) return 1.0;
    
    // Calculate pairwise coherence
    let coherenceSum = 0;
    for (let i = 0; i < topics.length - 1; i++) {
      for (let j = i + 1; j < topics.length; j++) {
        coherenceSum += this.calculateNPMI(topics[i], topics[j]);
      }
    }
    
    return coherenceSum / (topics.length * (topics.length - 1) / 2);
  }
  
  private calculateNPMI(word1: string, word2: string): number {
    // Simplified NPMI calculation
    // In production, use actual corpus statistics
    const jointProb = 0.1; // P(word1, word2)
    const prob1 = 0.3;     // P(word1)
    const prob2 = 0.2;     // P(word2)
    
    const pmi = Math.log(jointProb / (prob1 * prob2));
    const npmi = pmi / -Math.log(jointProb);
    
    return Math.max(0, Math.min(1, npmi));
  }
  
  private checkTerminologyConsistency(chunk: string): number {
    // Terminology consistency analysis
    // Placeholder implementation
    return 0.9;
  }
  
  private calculateLMCoherence(chunk: string): number {
    // Language model coherence scoring
    // Placeholder implementation
    return 0.85;
  }
  
  private countTypeAnnotations(chunk: string): number {
    return (chunk.match(/:[A-Z][a-zA-Z<>|\[\]]+/g) || []).length;
  }
  
  private extractKeywords(chunk: string): string[] {
    // Keyword extraction using TF-IDF or similar
    // Placeholder implementation
    return [];
  }
  
  private evaluateMetadata(chunk: string): number {
    // Metadata quality evaluation
    // Placeholder implementation
    return 2.5;
  }
  
  private checkRunnableCode(chunk: string): boolean {
    // Check if code examples are syntactically valid
    const codeBlocks = this.extractCodeBlocks(chunk);
    
    for (const code of codeBlocks) {
      try {
        // TypeScript syntax check
        if (code.includes('typescript') || code.includes('interface') || code.includes(':')) {
          const ts = require('typescript');
          const result = ts.transpileModule(code, { compilerOptions: { target: ts.ScriptTarget.ES2020 } });
          if (result.diagnostics && result.diagnostics.length > 0) {
            return false;
          }
        }
        
        // JavaScript syntax check
        else {
          const acorn = require('acorn');
          acorn.parse(code, { ecmaVersion: 2020, sourceType: 'module' });
        }
      } catch (error) {
        return false;
      }
    }
    
    return true;
  }
  
  private extractCodeBlocks(content: string): string[] {
    const codeBlockRegex = /```(?:typescript|javascript|js)?\n([\s\S]*?)\n```/g;
    const matches = [];
    let match;
    
    while ((match = codeBlockRegex.exec(content)) !== null) {
      matches.push(match[1]);
    }
    
    return matches;
  }
}
```

### A.2. Usage Examples

> **Kod Durumu:** `Reference`
```typescript
// Basic usage
const analyzer = new ChunkQualityAnalyzer();
const metrics = analyzer.analyzeChunk(chunkContent, chunkMetadata);

console.log(`Overall Score: ${metrics.overallScore}/10`);
console.log(`Semantic Coherence: ${metrics.semanticCoherence}/10`);
console.log(`Contextual Completeness: ${metrics.contextualCompleteness}/10`);
console.log(`Retrieval Efficiency: ${metrics.retrievalEfficiency}/10`);
console.log(`Actionability: ${metrics.actionability}/10`);

// Batch processing
async function batchAnalyzeChunks(chunks: Chunk[]): Promise<ChunkQualityMetrics[]> {
  return Promise.all(
    chunks.map(chunk => analyzer.analyzeChunk(chunk.content, chunk.metadata))
  );
}

// Quality filtering
function filterHighQualityChunks(chunks: Chunk[]): Chunk[] {
  return chunks.filter(chunk => {
    const metrics = analyzer.analyzeChunk(chunk.content, chunk.metadata);
    return metrics.overallScore >= 7.0;
  });
}
```

## 9. Multi-Hop Retrieval and Reasoning

### 9.1. Multi-Hop Query Processing

> **Kod Durumu:** `Reference`
```typescript
interface MultiHopStep {
  stepNumber: number;
  query: string;
  retrievedChunks: Chunk[];
  reasoning: string;
  nextQuery?: string;
  confidence: number;
}

interface MultiHopChain {
  originalQuery: string;
  steps: MultiHopStep[];
  finalAnswer: string;
  totalConfidence: number;
  reasoningPath: string;
}

class MultiHopRetriever {
  async processMultiHopQuery(query: string): Promise<MultiHopChain> {
    const steps: MultiHopStep[] = [];
    let currentQuery = query;
    let stepNumber = 1;
    
    while (stepNumber <= 3 && !this.isQuerySatisfied(currentQuery, steps)) {
      // 1. Retrieve chunks for current query
      const chunks = await this.retrieveChunks(currentQuery);
      
      // 2. Analyze if we need more information
      const reasoning = await this.analyzeRetrieval(currentQuery, chunks);
      
      // 3. Generate next query if needed
      const nextQuery = reasoning.needsMoreInfo 
        ? await this.generateNextQuery(currentQuery, chunks, reasoning)
        : undefined;
      
      steps.push({
        stepNumber: stepNumber,
        query: currentQuery,
        retrievedChunks: chunks,
        reasoning: reasoning.explanation,
        nextQuery: nextQuery,
        confidence: reasoning.confidence
      });
      
      if (!nextQuery) break;
      currentQuery = nextQuery;
      stepNumber++;
    }
    
    // 4. Generate final answer
    const finalAnswer = await this.generateFinalAnswer(query, steps);
    
    return {
      originalQuery: query,
      steps,
      finalAnswer: finalAnswer.content,
      totalConfidence: finalAnswer.confidence,
      reasoningPath: steps.map(s => s.reasoning).join(' → ')
    };
  }
  
  private isQuerySatisfied(query: string, steps: MultiHopStep[]): boolean {
    // Check if we have enough information to answer
    const lastStep = steps[steps.length - 1];
    return lastStep?.confidence > 0.8 || steps.length >= 3;
  }
}
```

### 9.2. Complex Query Examples

| Query Type | Multi-Hop Strategy | Example Steps |
|------------|-------------------|---------------|
| **Causal Analysis** | "Why does X happen?" | 1. Find X → 2. Find causes → 3. Find solutions |
| **Comparison** | "Compare A vs B" | 1. Find A → 2. Find B → 3. Compare features |
| **Troubleshooting** | "Debug X error" | 1. Error description → 2. Common causes → 3. Solutions |
| **Integration** | "How to connect X and Y" | 1. X documentation → 2. Y documentation → 3. Integration patterns |

## 10. Chunk Linking and Knowledge Graph

### 10.1. Dependency Graph Construction

> **Kod Durumu:** `Reference`
```typescript
interface ChunkNode {
  id: string;
  content: string;
  metadata: ChunkMetadata;
  dependencies: string[];  // IDs of dependent chunks
  dependents: string[];    // IDs of chunks that depend on this
  relatedConcepts: string[];
  importanceScore: number;
}

interface ChunkEdge {
  sourceId: string;
  targetId: string;
  relationshipType: 'imports' | 'extends' | 'implements' | 'references' | 'similar';
  strength: number;
  bidirectional: boolean;
}

class ChunkGraphBuilder {
  async buildDependencyGraph(chunks: Chunk[]): Promise<ChunkGraph> {
    const nodes: ChunkNode[] = [];
    const edges: ChunkEdge[] = [];
    
    // 1. Create nodes
    for (const chunk of chunks) {
      nodes.push({
        id: chunk.id,
        content: chunk.content,
        metadata: chunk.metadata,
        dependencies: [],
        dependents: [],
        relatedConcepts: await this.extractConcepts(chunk),
        importanceScore: this.calculateImportance(chunk)
      });
    }
    
    // 2. Create edges
    for (let i = 0; i < nodes.length; i++) {
      for (let j = i + 1; j < nodes.length; j++) {
        const edge = await this.analyzeRelationship(nodes[i], nodes[j]);
        if (edge) {
          edges.push(edge);
          
          // Update node relationships
          nodes[i].dependencies.push(edge.targetId);
          nodes[j].dependents.push(edge.sourceId);
        }
      }
    }
    
    return { nodes, edges };
  }
  
  private async analyzeRelationship(node1: ChunkNode, node2: ChunkNode): Promise<ChunkEdge | null> {
    // 1. Import relationship detection
    if (this.detectImportRelationship(node1.content, node2.content)) {
      return {
        sourceId: node1.id,
        targetId: node2.id,
        relationshipType: 'imports',
        strength: 0.9,
        bidirectional: false
      };
    }
    
    // 2. Similarity detection
    const similarity = await this.calculateSemanticSimilarity(node1.content, node2.content);
    if (similarity > 0.7) {
      return {
        sourceId: node1.id,
        targetId: node2.id,
        relationshipType: 'similar',
        strength: similarity,
        bidirectional: true
      };
    }
    
    return null;
  }
}
```

### 10.2. Context Expansion Using Graph

> **Kod Durumu:** `Reference`
```typescript
class GraphAwareRetriever {
  async expandContext(initialChunks: Chunk[], graph: ChunkGraph): Promise<Chunk[]> {
    const expandedChunks = new Set(initialChunks.map(c => c.id));
    const queue = [...initialChunks];
    
    while (queue.length > 0 && expandedChunks.size < 10) {
      const currentChunk = queue.shift()!;
      
      // 1. Find related chunks in graph
      const relatedChunkIds = this.findRelatedChunks(currentChunk.id, graph);
      
      // 2. Add high-importance related chunks
      for (const chunkId of relatedChunkIds) {
        if (!expandedChunks.has(chunkId)) {
          const chunk = await this.getChunkById(chunkId);
          if (chunk && this.isHighImportance(chunkId, graph)) {
            expandedChunks.add(chunkId);
            queue.push(chunk);
          }
        }
      }
    }
    
    // 3. Return expanded chunk set
    return Array.from(expandedChunks).map(id => this.getChunkById(id)).filter(Boolean) as Chunk[];
  }
  
  private findRelatedChunks(chunkId: string, graph: ChunkGraph): string[] {
    const node = graph.nodes.find(n => n.id === chunkId);
    if (!node) return [];
    
    // Combine dependencies, dependents, and similar chunks
    const related = [
      ...node.dependencies,
      ...node.dependents,
      ...graph.edges
        .filter(e => (e.sourceId === chunkId || e.targetId === chunkId) && e.relationshipType === 'similar')
        .map(e => e.sourceId === chunkId ? e.targetId : e.sourceId)
    ];
    
    return related;
  }
}
```

## 11. Enhanced Evaluation Dataset

### 11.1. Comprehensive Test Categories

> **Kod Durumu:** `Reference`
```typescript
interface TestCase {
  id: string;
  category: 'basic' | 'adversarial' | 'edgeCase' | 'trick' | 'complex';
  query: string;
  expectedChunks: string[];
  difficulty: 'easy' | 'medium' | 'hard' | 'expert';
  domain: string;
  reasoningRequired: boolean;
  multiHop: boolean;
}

class EvaluationDatasetGenerator {
  generateComprehensiveDataset(): TestCase[] {
    return [
      // Basic queries
      {
        id: 'basic001',
        category: 'basic',
        query: 'How to create a TypeScript interface?',
        expectedChunks: ['interfaceSyntax', 'typeExamples'],
        difficulty: 'easy',
        domain: 'typescript',
        reasoningRequired: false,
        multiHop: false
      },
      
      // Adversarial queries
      {
        id: 'adv001',
        category: 'adversarial',
        query: 'TypeScript interface but make it wrong',
        expectedChunks: ['interfaceBestPractices', 'commonMistakes'],
        difficulty: 'medium',
        domain: 'typescript',
        reasoningRequired: true,
        multiHop: false
      },
      
      // Edge cases
      {
        id: 'edge001',
        category: 'edgeCase',
        query: 'Generic interface with recursive types and conditional types',
        expectedChunks: ['advancedGenerics', 'recursiveTypes', 'conditionalTypes'],
        difficulty: 'expert',
        domain: 'typescript',
        reasoningRequired: true,
        multiHop: true
      },
      
      // Trick questions
      {
        id: 'trick001',
        category: 'trick',
        query: 'TypeScript vs JavaScript for frontend development in 2024',
        expectedChunks: ['typescriptComparison', 'javascriptFeatures', 'frontendTrends'],
        difficulty: 'medium',
        domain: 'webDevelopment',
        reasoningRequired: true,
        multiHop: true
      },
      
      // Complex multi-domain queries
      {
        id: 'complex001',
        category: 'complex',
        query: 'Implement OAuth 2.0 authentication with JWT refresh tokens in Node.js using TypeScript',
        expectedChunks: ['oauth2_flow', 'jwtImplementation', 'nodejsAuth', 'typescriptPatterns'],
        difficulty: 'expert',
        domain: 'authentication',
        reasoningRequired: true,
        multiHop: true
      }
    ];
  }
}
```

### 11.2. RAGAS (RAG Assessment) Metrics

**Standart RAG Evaluation Metrics:**

| Metric | Açıklama | Hesaplama Yöntemi | Hedef Değer |
|--------|------------|-------------------|-------------|
| **Faithfulness** | Cevabın kaynaklara sadakati | LLM-based evaluation | ≥ 0.8 |
| **Answer Relevance** | Cevabın soruyla alaka düzeyi | Embedding similarity | ≥ 0.85 |
| **Context Precision** | Retrieve edilen context'in relevancy'sı | Top-k precision | ≥ 0.75 |
| **Context Recall** | Gerekli context'in ne kadarının bulunduğu | Recall calculation | ≥ 0.80 |
| **Context Entity Recall** | Entity'lerin ne kadarının korunduğu | Entity matching | ≥ 0.85 |
| **Noise Sensitivity** | Gereksiz bilginin cevabı etkileme durumu | Perturbation analysis | ≤ 0.2 |

**RAGAS Implementation:**
> **Kod Durumu:** `Reference`
```typescript
interface RAGASEvaluation {
  faithfulness: number;        // 0-1
  answerRelevance: number;    // 0-1
  contextPrecision: number;   // 0-1
  contextRecall: number;      // 0-1
  contextEntityRecall: number; // 0-1
  noiseSensitivity: number;    // 0-1 (lower is better)
  overallScore: number;       // Weighted combination
}

class RAGASEvaluator {
  evaluateRAGResponse(
    query: string,
    retrievedChunks: Chunk[],
    generatedAnswer: string,
    groundTruth?: string
  ): RAGASEvaluation {
    // 1. Faithfulness evaluation
    const faithfulness = await this.evaluateFaithfulness(generatedAnswer, retrievedChunks);
    
    // 2. Answer relevance
    const answerRelevance = await this.evaluateAnswerRelevance(query, generatedAnswer);
    
    // 3. Context precision
    const contextPrecision = this.calculateContextPrecision(query, retrievedChunks);
    
    // 4. Context recall
    const contextRecall = groundTruth 
      ? this.calculateContextRecall(groundTruth, retrievedChunks)
      : 0.8; // Default if no ground truth
    
    // 5. Context entity recall
    const contextEntityRecall = this.calculateEntityRecall(query, retrievedChunks);
    
    // 6. Noise sensitivity
    const noiseSensitivity = await this.evaluateNoiseSensitivity(query, retrievedChunks, generatedAnswer);
    
    // 7. Overall score (RAGAS weighted average)
    const overallScore = (
      faithfulness * 0.25 +
      answerRelevance * 0.20 +
      contextPrecision * 0.20 +
      contextRecall * 0.15 +
      contextEntityRecall * 0.10 +
      (1 - noiseSensitivity) * 0.10
    );
    
    return {
      faithfulness,
      answerRelevance: answerRelevance,
      contextPrecision: contextPrecision,
      contextRecall: contextRecall,
      contextEntityRecall: contextEntityRecall,
      noiseSensitivity: noiseSensitivity,
      overallScore: overallScore
    };
  }
  
  private async evaluateFaithfulness(answer: string, chunks: Chunk[]): Promise<number> {
    // LLM-based faithfulness evaluation
    const prompt = `
      Given the answer: "${answer}"
      And the context: ${chunks.map(c => c.content).join('\n\n')}
      
      Rate the faithfulness of the answer to the context (0-1):
      - 1.0: Answer is fully supported by context
      - 0.5: Partially supported
      - 0.0: Not supported or contains hallucinations
      
      Respond with only the number.
    `;
    
    const response = await this.llm.generate(prompt);
    return parseFloat(response.trim()) || 0.5;
  }
  
  private async evaluateAnswerRelevance(query: string, answer: string): Promise<number> {
    // Embedding-based relevance
    const queryEmbedding = await this.embed(query);
    const answerEmbedding = await this.embed(answer);
    return this.cosineSimilarity(queryEmbedding, answerEmbedding);
  }
  
  private calculateContextPrecision(query: string, chunks: Chunk[]): number {
    // Top-k precision calculation
    const relevantChunks = chunks.filter(chunk => 
      this.isChunkRelevant(query, chunk)
    );
    return relevantChunks.length / chunks.length;
  }
  
  private calculateContextRecall(groundTruth: string, chunks: Chunk[]): number {
    // Recall calculation against ground truth
    const groundTruthEntities = this.extractEntities(groundTruth);
    const retrievedEntities = chunks.flatMap(chunk => this.extractEntities(chunk.content));
    
    const overlappingEntities = groundTruthEntities.filter(entity => 
      retrievedEntities.includes(entity)
    );
    
    return overlappingEntities.length / groundTruthEntities.length;
  }
  
  private calculateEntityRecall(query: string, chunks: Chunk[]): number {
    // Entity preservation in retrieval
    const queryEntities = this.extractEntities(query);
    const retrievedEntities = chunks.flatMap(chunk => this.extractEntities(chunk.content));
    
    const preservedEntities = queryEntities.filter(entity => 
      retrievedEntities.includes(entity)
    );
    
    return queryEntities.length > 0 ? preservedEntities.length / queryEntities.length : 1.0;
  }
  
  private async evaluateNoiseSensitivity(
    query: string, 
    chunks: Chunk[], 
    answer: string
  ): Promise<number> {
    // Test with noisy chunks
    const noisyChunks = this.addNoiseToChunks(chunks);
    const noisyAnswer = await this.generateAnswer(query, noisyChunks);
    
    // Compare original vs noisy answer
    const originalEmbedding = await this.embed(answer);
    const noisyEmbedding = await this.embed(noisyAnswer);
    
    return 1 - this.cosineSimilarity(originalEmbedding, noisyEmbedding);
  }
}
```

### 11.2. Evaluation Metrics

> **Kod Durumu:** `Reference`
```typescript
interface EvaluationResults {
  overallAccuracy: number;
  categoryPerformance: Record<string, number>;
  difficultyPerformance: Record<string, number>;
  multiHopSuccessRate: number;
  reasoningAccuracy: number;
  adversarialRobustness: number;
}

class RAGEvaluator {
  evaluateSystem(testCases: TestCase[], retriever: RAGSystem): EvaluationResults {
    const results = {
      overallAccuracy: 0,
      categoryPerformance: {},
      difficultyPerformance: {},
      multiHopSuccessRate: 0,
      reasoningAccuracy: 0,
      adversarialRobustness: 0
    };
    
    let totalCorrect = 0;
    const categoryStats: Record<string, { correct: number; total: number }> = {};
    const difficultyStats: Record<string, { correct: number; total: number }> = {};
    
    for (const testCase of testCases) {
      const result = retriever.retrieve(testCase.query);
      const isCorrect = this.evaluateResult(result, testCase);
      
      if (isCorrect) totalCorrect++;
      
      // Update category stats
      if (!categoryStats[testCase.category]) {
        categoryStats[testCase.category] = { correct: 0, total: 0 };
      }
      categoryStats[testCase.category].total++;
      if (isCorrect) categoryStats[testCase.category].correct++;
      
      // Update difficulty stats
      if (!difficultyStats[testCase.difficulty]) {
        difficultyStats[testCase.difficulty] = { correct: 0, total: 0 };
      }
      difficultyStats[testCase.difficulty].total++;
      if (isCorrect) difficultyStats[testCase.difficulty].correct++;
    }
    
    // Calculate final metrics
    results.overallAccuracy = totalCorrect / testCases.length;
    
    for (const [category, stats] of Object.entries(categoryStats)) {
      results.categoryPerformance[category] = stats.correct / stats.total;
    }
    
    for (const [difficulty, stats] of Object.entries(difficultyStats)) {
      results.difficultyPerformance[difficulty] = stats.correct / stats.total;
    }
    
    return results;
  }
}
```

## B. Legal & Compliance Rubric

### B.1. Regüle Sektörler İçin Ek Kriterler

**Hukuki, Bankacılık, Sağlık gibi Regüle Alanlar İçin:**

| Kriter | Açıklama | Puanlama | Eşik Değer |
|--------|------------|-----------|-------------|
| **Fact-Checking** | Bilginin doğrulanabilirliği | 0-10 | ≥ 8.0 |
| **Source Attribution** | Kaynak gösterme zorunluluğu | 0-10 | ≥ 9.0 |
| **Regulatory Compliance** | Yönetmeliklere uyum | 0-10 | ≥ 9.5 |
| **Data Classification** | Veri sınıflandırması | 0-10 | ≥ 8.5 |
| **Audit Trail** | Değişiklik takibi | 0-10 | ≥ 8.0 |
| **Version Control** | Versiyon yönetimi | 0-10 | ≥ 7.5 |

**Legal Compliance Scoring:**
> **Kod Durumu:** `Reference`
```typescript
interface LegalComplianceMetrics {
  factCheckingScore: number;    // 0-10
  sourceAttributionScore: number; // 0-10
  regulatoryCompliance: number;    // 0-10
  dataClassification: number;      // 0-10
  auditTrail: number;             // 0-10
  versionControl: number;          // 0-10
  overallCompliance: number;       // Weighted average
}

class LegalComplianceAnalyzer {
  analyzeLegalCompliance(chunk: Chunk, domain: 'legal' | 'banking' | 'healthcare'): LegalComplianceMetrics {
    const domainRules = this.getDomainRules(domain);
    
    const metrics: LegalComplianceMetrics = {
      factCheckingScore: this.analyzeFactChecking(chunk),
      sourceAttributionScore: this.analyzeSourceAttribution(chunk),
      regulatoryCompliance: this.checkRegulatoryCompliance(chunk, domainRules),
      dataClassification: this.analyzeDataClassification(chunk),
      auditTrail: this.checkAuditTrail(chunk),
      versionControl: this.checkVersionControl(chunk),
      overallCompliance: 0
    };
    
    // Domain-specific weighting
    const weights = domainRules.weights;
    metrics.overallCompliance = 
      metrics.factCheckingScore * weights.factChecking +
      metrics.sourceAttributionScore * weights.sourceAttribution +
      metrics.regulatoryCompliance * weights.regulatory +
      metrics.dataClassification * weights.dataClassification +
      metrics.auditTrail * weights.auditTrail +
      metrics.versionControl * weights.versionControl;
    
    return metrics;
  }
  
  private analyzeFactChecking(chunk: Chunk): number {
    // Check for verifiable claims
    const claims = this.extractClaims(chunk.content);
    let verifiableClaims = 0;
    
    for (const claim of claims) {
      if (this.hasVerifiableSource(claim)) {
        verifiableClaims++;
      }
    }
    
    return claims.length > 0 ? (verifiableClaims / claims.length) * 10 : 5;
  }
  
  private analyzeSourceAttribution(chunk: Chunk): number {
    // Check for proper source citation
    const hasCitations = /\[source:\s*[^\]]+\]|\(source:\s*[^)]+\)/.test(chunk.content);
    const hasReferences = chunk.metadata.references && chunk.metadata.references.length > 0;
    const hasExternalLinks = /https?:\/\/[^\s]+/.test(chunk.content);
    
    let score = 0;
    if (hasCitations) score += 4;
    if (hasReferences) score += 3;
    if (hasExternalLinks) score += 3;
    
    return Math.min(10, score);
  }
  
  private checkRegulatoryCompliance(chunk: Chunk, rules: DomainRules): number {
    // Check against domain-specific regulations
    let complianceScore = 10;
    
    // Check for required disclaimers
    if (rules.requiredDisclaimers) {
      const hasDisclaimer = rules.requiredDisclaimers.some(disclaimer => 
        chunk.content.includes(disclaimer)
      );
      if (!hasDisclaimer) complianceScore -= 2;
    }
    
    // Check for prohibited content
    if (rules.prohibitedContent) {
      const hasProhibited = rules.prohibitedContent.some(prohibited => 
        chunk.content.toLowerCase().includes(prohibited.toLowerCase())
      );
      if (hasProhibited) complianceScore -= 5;
    }
    
    // Check for required legal language
    if (rules.requiredLanguage) {
      const hasRequiredLanguage = rules.requiredLanguage.some(language => 
        chunk.content.includes(language)
      );
      if (!hasRequiredLanguage) complianceScore -= 1;
    }
    
    return Math.max(0, complianceScore);
  }
}
```

**Domain-Specific Rules:**
> **Kod Durumu:** `Reference`
```typescript
interface DomainRules {
  domain: string;
  weights: {
    factChecking: number;
    sourceAttribution: number;
    regulatory: number;
    dataClassification: number;
    auditTrail: number;
    versionControl: number;
  };
  requiredDisclaimers?: string[];
  prohibitedContent?: string[];
  requiredLanguage?: string[];
  retentionPeriodDays?: number;
}

const domainRules: Record<string, DomainRules> = {
  legal: {
    domain: 'legal',
    weights: {
      factChecking: 0.25,
      sourceAttribution: 0.25,
      regulatory: 0.20,
      dataClassification: 0.15,
      auditTrail: 0.10,
      versionControl: 0.05
    },
    requiredDisclaimers: ['This is not legal advice', 'Consult with qualified attorney'],
    prohibitedContent: ['guaranteed outcome', 'legal advice without disclaimer'],
    retentionPeriodDays: 2555 // 7 years
  },
  
  banking: {
    domain: 'banking',
    weights: {
      factChecking: 0.20,
      sourceAttribution: 0.20,
      regulatory: 0.30,
      dataClassification: 0.20,
      auditTrail: 0.10,
      versionControl: 0.00
    },
    requiredDisclaimers: ['FDIC insured', 'Equal housing lender'],
    prohibitedContent: ['guaranteed returns', 'risk-free investment'],
    retentionPeriodDays: 3650 // 10 years
  },
  
  healthcare: {
    domain: 'healthcare',
    weights: {
      factChecking: 0.20,
      sourceAttribution: 0.15,
      regulatory: 0.25,
      dataClassification: 0.25,
      auditTrail: 0.10,
      versionControl: 0.05
    },
    requiredDisclaimers: ['Not medical advice', 'Consult healthcare provider'],
    prohibitedContent: ['diagnosis without qualification', 'treatment guarantees'],
    retentionPeriodDays: 2190 // 6 years
  }
};
```

## C. Hibrit Arama Stratejileri

### 13.1. Chunk Güncelleme Akışı

> **Kod Durumu:** `Reference`
```typescript
interface ChunkVersion {
  version: string;
  chunkId: string;
  content: string;
  metadata: ChunkMetadata;
  createdAt: Date;
  isStable: boolean;
  rollbackVersion?: string;
}

interface DeploymentPipeline {
  stages: ['development' → 'staging' → 'canary' → 'production'];
  healthChecks: HealthCheck[];
  rollbackTriggers: RollbackTrigger[];
}

class ChunkVersionManager {
  async updateChunk(chunkId: string, newContent: string): Promise<ChunkVersion> {
    // 1. Create new version
    const currentVersion = await this.getCurrentVersion(chunkId);
    const newVersion = await this.createVersion(chunkId, newContent, currentVersion);
    
    // 2. Quality analysis
    const qualityScore = await this.qualityAnalyzer.analyzeChunk(newContent, newVersion.metadata);
    if (qualityScore.overallScore < 7.0) {
      throw new Error(`Quality score too low: ${qualityScore.overallScore}`);
    }
    
    // 3. Deployment pipeline
    await this.deployToStaging(newVersion);
    await this.runHealthChecks(newVersion);
    await this.deployToCanary(newVersion, { trafficPercentage: 5 });
    
    // 4. Monitor canary
    const canaryResults = await this.monitorCanary(newVersion, { duration: '30min' });
    if (!canaryResults.isHealthy) {
      await this.rollback(newVersion, currentVersion.version);
      throw new Error('Canary deployment failed');
    }
    
    // 5. Production deployment
    await this.deployToProduction(newVersion);
    
    return newVersion;
  }
  
  async rollback(newVersion: ChunkVersion, targetVersion: string): Promise<void> {
    // 1. Stop traffic to new version
    await this.stopTraffic(newVersion.chunkId);
    
    // 2. Restore target version
    const targetChunk = await this.getVersion(newVersion.chunkId, targetVersion);
    await this.restoreVersion(targetChunk);
    
    // 3. Update routing
    await this.updateRouting(newVersion.chunkId, targetVersion);
    
    // 4. Log rollback
    await this.logRollback({
      fromVersion: newVersion.version,
      toVersion: targetVersion,
      reason: 'manualRollback',
      timestamp: new Date()
    });
  }
}
```

### 13.2. Otomatik Rollback Tetik Koşulları

| Trigger | Threshold | Action | Recovery Time |
|---------|------------|--------|---------------|
| **Error Rate** | > 5% (1 dakika) | Otomatik rollback | < 30 saniye |
| **Latency** | > 500ms (5 dakika) | Canary rollback | < 1 dakika |
| **Quality Score** | < 6.5 (canary) | Otomatik rollback | < 45 saniye |
| **User Feedback** | Rating < 3.0 (10 query) | Manuel review | 2 saat |
| **System Health** | Memory > 80% | Otomatik rollback | < 1 dakika |

### 13.3. Alert Webhook Örnekleri

> **Kod Durumu:** `Reference`
```typescript
// Slack/Teams webhook integration
interface AlertWebhook {
  type: 'rollback' | 'qualityDegradation' | 'performanceIssue';
  severity: 'low' | 'medium' | 'high' | 'critical';
  message: string;
  chunkId: string;
  version: string;
  metrics: Record<string, number>;
  actions: string[];
}

const alertExamples = {
  rollback: {
    type: 'rollback',
    severity: 'critical',
    message: '🚨 Automatic rollback triggered for chunk user-service-auth',
    chunkId: 'user-service-auth',
    version: 'v2.1.0',
    metrics: { errorRate: 0.08, latencyMs: 650, qualityScore: 5.2 },
    actions: ['Check logs', 'Review quality metrics', 'Investigate root cause']
  },
  
  qualityDegradation: {
    type: 'qualityDegradation',
    severity: 'medium',
    message: '⚠️ Quality score degradation detected in chunk api-routes',
    chunkId: 'api-routes',
    version: 'v1.4.2',
    metrics: { qualityScore: 6.8, previousScore: 8.2, delta: -1.4 },
    actions: ['Review content changes', 'Check quality analyzer', 'Schedule manual review']
  }
};

// Webhook gönderimi
async function sendAlert(alert: AlertWebhook): Promise<void> {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  
  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: alert.message,
      attachments: [{
        color: alert.severity === 'critical' ? 'danger' : 'warning',
        fields: [
          { title: 'Chunk ID', value: alert.chunkId, short: true },
          { title: 'Version', value: alert.version, short: true },
          { title: 'Metrics', value: JSON.stringify(alert.metrics, null, 2) },
          { title: 'Recommended Actions', value: alert.actions.join(', ') }
        ]
      }]
    })
  });
}
```

### 13.4. Metadata Örnekleri

> **Kod Durumu:** `Production-Ready`
```json
{
  "chunkId": "user-service-auth-v2.1.0",
  "title": "JWT Authentication Service",
  "description": "Handles JWT token creation, validation, and refresh operations",
  "metadata": {
    "sourceUrl": "https://github.com/company/user-service/blob/main/src/auth/jwt.service.ts",
    "checksum": "sha256:a1b2c3d4e5f6...",
    "provenance": {
      "filePath": "/src/auth/jwt.service.ts",
      "lineStart": 15,
      "lineEnd": 89,
      "extractionMethod": "typescript-ast-parser"
    },
    "license": "MIT",
    "sensitivityLevel": "internal",
    "version": "v2.1.0",
    "language": "typescript",
    "contentType": "code",
    "dependencies": ["jsonwebtoken", "bcrypt"],
    "tags": ["authentication", "jwt", "security", "nodejs"],
    "qualityScore": {
      "overall": 8.7,
      "semanticCoherence": 9.2,
      "contextualCompleteness": 8.5,
      "retrievalEfficiency": 8.8,
      "actionability": 8.3,
      "lastEvaluated": "2024-03-30T23:00:00Z"
    },
    "usageStats": {
      "retrievalCount": 1247,
      "userRating": 4.2,
      "feedbackCount": 89,
      "lastUsed": "2024-03-30T22:45:00Z",
      "successRate": 0.94
    }
  }
}
```

### 12.1. Simplified Metadata for Small Projects

> **Kod Durumu:** `Reference`
```typescript
interface LiteChunkMetadata {
  id: string;
  title: string;
  contentType: 'code' | 'documentation' | 'example';
  language: string;
  lastUpdated: Date;
  qualityScore: number;
  // Reduced metadata for smaller projects
}

interface LiteQualityMetrics {
  readability: number;      // 0-10
  completeness: number;     // 0-10
  relevance: number;       // 0-10
  overall: number;         // 0-10
}

class LiteQualityAnalyzer {
  analyzeChunkLite(chunk: string): LiteQualityMetrics {
    // Simplified analysis for small projects
    const readability = this.calculateReadability(chunk);
    const completeness = this.checkCompleteness(chunk);
    const relevance = this.assessRelevance(chunk);
    
    const overall = (readability + completeness + relevance) / 3;
    
    return {
      readability,
      completeness,
      relevance,
      overall
    };
  }
  
  private calculateReadability(content: string): number {
    // Simple readability metrics
    const sentences = content.split(/[.!?]+/).length;
    const words = content.split(/\s+/).length;
    const avgWordsPerSentence = words / sentences;
    
    // Optimal is 15-20 words per sentence
    if (avgWordsPerSentence >= 15 && avgWordsPerSentence <= 20) {
      return 9;
    } else if (avgWordsPerSentence >= 10 && avgWordsPerSentence <= 25) {
      return 7;
    } else {
      return 5;
    }
  }
  
  private checkCompleteness(content: string): number {
    let score = 5; // Base score
    
    // Check for code examples
    if (/```/.test(content)) score += 2;
    
    // Check for explanations
    if (content.length > 200) score += 1;
    
    // Check for structure
    if (/#{1,3}/.test(content)) score += 2;
    
    return Math.min(10, score);
  }
  
  private assessRelevance(content: string): number {
    // Simple relevance based on keyword density
    const technicalTerms = ['function', 'class', 'interface', 'type', 'method', 'property'];
    const termCount = technicalTerms.filter(term => 
      content.toLowerCase().includes(term)
    ).length;
    
    return Math.min(10, termCount * 2);
  }
}
```

### 12.2. Cost-Optimized Configuration

> **Kod Durumu:** `Production-Ready`
```typescript
interface LiteConfiguration {
  embeddingModel: 'local' | 'small' | 'medium';
  maxChunks: number;
  qualityThreshold: number;
  enableCaching: boolean;
  enableGraph: boolean;
}

const liteConfigs: Record<string, LiteConfiguration> = {
  'starter': {
    embeddingModel: 'local',
    maxChunks: 1000,
    qualityThreshold: 6.0,
    enableCaching: false,
    enableGraph: false
  },
  'growth': {
    embeddingModel: 'small',
    maxChunks: 5000,
    qualityThreshold: 7.0,
    enableCaching: true,
    enableGraph: false
  },
  'professional': {
    embeddingModel: 'medium',
    maxChunks: 20000,
    qualityThreshold: 8.0,
    enableCaching: true,
    enableGraph: true
  }
};
```

### 5.1. Chunk'lama Stratejileri

**Fonksiyon Seviyesi Chunk'lama:**
- Her fonksiyon/class/interface ayrı chunk
- İlişkili type'lar birlikte
- Test örnekleri dahil

**Konsept Seviyesi Chunk'lama:**
- Tek konsept odaklı
- Teori + pratik birlikte
- İlişkili pattern'lar gruplanmış

### 5.2. Metadata Optimizasyonu

> **Kod Durumu:** `Reference`
```typescript
interface ChunkMetadata {
  // Basic metadata
  title: string;
  description: string;
  keywords: string[];
  category: string;
  difficulty: 'beginner' | 'intermediate' | 'advanced';
  language: string;
  dependencies: string[];
  relatedTopics: string[];
  lastUpdated: Date;
  author: string;
  tags: string[];
  
  // Extended metadata for production
  sourceUrl: string;
  checksum: string;
  chunkId: string;
  provenance: {
    filePath: string;
    lineStart: number;
    lineEnd: number;
    extractionMethod: string;
  };
  license: string;
  sensitivityLevel: 'public' | 'internal' | 'confidential' | 'restricted';
  version: string;
  qualityScore: {
    overall: number;
    semanticCoherence: number;
    contextualCompleteness: number;
    retrievalEfficiency: number;
    actionability: number;
    lastEvaluated: Date;
  };
  usageStats: {
    retrievalCount: number;
    userRating: number;
    feedbackCount: number;
    lastUsed: Date;
  };
}
```

### 5.3. Sürekli Kalite İyileştirme

**Feedback Loop:**
- Kullanıcı etkileşimlerini izle
- Retrieval başarı oranlarını ölç
- Chunk'ların kullanım sıklığını takip et
- Düşük skorlu chunk'ları otomatik flag'le

**Otomatik İyileştirme:**
- Düşük skorlu chunk'ları yeniden işle
- Eksik metadata'yı otomatik tamamla
- Benzer chunk'ları birleştir
- Gereksiz chunk'ları sil

## 6. Teknik Uygulama Detayları

### 6.1. Tokenizer Seçimi

**Önerilen Tokenizer'lar:**
- **OpenAI Tiktoken:** `cl100k_base` (GPT-4 compatible)
- **HuggingFace Tokenizers:** `bert-base-uncased` for code
- **Custom Tokenizer:** Proje spesifik keyword'ler için

**Implementasyon:**
> **Kod Durumu:** `Reference`
```typescript
import { Tiktoken } from '@dqbd/tiktoken';

const encoder = Tiktoken.getEncoding('cl100k_base');
const tokens = encoder.encode(chunkString).length;
```

### 6.2. Embedding Model Önerileri

**Code-Specific Models:**
- **OpenAI:** `text-embedding-ada-002` veya `text-embedding-3-small`
- **HuggingFace:** `microsoft/codebert-base`
- **Sentence Transformers:** `all-MiniLM-L6-v2` (genel metin için)

**Dimension Recommendations:**
- **Production:** 1536 dimensions (OpenAI)
- **Cost-optimized:** 384 dimensions (Sentence Transformers)
- **Hybrid:** Multiple embeddings for different content types

### 6.3. Veri Saklama Formatı

**JSON Schema Örneği:**
> **Kod Durumu:** `Production-Ready`
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "chunkId": {"type": "string", "pattern": "^[a-zA-Z0-9_-]+$"},
    "content": {"type": "string", "minLength": 10, "maxLength": 10000},
    "metadata": {"$ref": "#/definitions/ChunkMetadata"},
    "embedding": {
      "type": "array",
      "items": {"type": "number"},
      "minItems": 128,
      "maxItems": 1536
    },
    "qualityMetrics": {"$ref": "#/definitions/QualityMetrics"}
  },
  "required": ["chunkId", "content", "metadata", "embedding", "qualityMetrics"]
}
```

### 6.4. Versiyonlama Stratejisi

**Chunk ID Format:** `{project}_{fileHash}_{chunkIndex}_{version}`
**Version Control:**
- Semantic versioning: `v1.0.0` → `v1.0.1` (patch)
- Content-based: SHA256 hash of content
- Timestamp: ISO 8601 format

## 7. Güvenlik ve Gizlilik

### 7.1. Hassas Veri Tespiti

**PII Detection Patterns:**
> **Kod Durumu:** `Reference`
```typescript
const PII_PATTERNS = {
  email: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
  phone: /\b\d{3}-?\d{3}-?\d{4}\b/g,
  ssn: /\b\d{3}-?\d{2}-?\d{4}\b/g,
  creditCard: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g,
  apiKey: /\b[A-Za-z0-9]{32,}\b/g,
  password: /password[\s]*[:=][\s]*['"]?[\w@#$%^&*]{6,}['"]?/gi
};
```

**Redaction Strategy:**
> **Kod Durumu:** `Reference`
```typescript
function redactPII(content: string): string {
  let redacted = content;
  Object.values(PII_PATTERNS).forEach(pattern => {
    redacted = redacted.replace(pattern, '[REDACTED]');
  });
  return redacted;
}
```

### 7.2. Erişim Kontrolü

**Sensitivity Levels:**
- **Public:** Herkese açık
- **Internal:** Sadece şirket içi
- **Confidential:** Rol bazlı erişim
- **Restricted:** Özel izin gerekli

**Access Control Matrix:**
> **Kod Durumu:** `Reference`
```typescript
interface AccessControl {
  userRole: 'admin' | 'developer' | 'viewer' | 'guest';
  sensitivityLevel: 'public' | 'internal' | 'confidential' | 'restricted';
  allowedOperations: ('read' | 'write' | 'delete')[];
}
```

### 7.3. Veri Saklama Politikaları

**Retention Rules:**
- **Production chunks:** 2 yıl
- **Staging chunks:** 6 ay
- **Test chunks:** 1 ay
- **PII-containing chunks:** 30 gün (redaction sonrası)

## 8. Monitoring ve Geri Bildirim Döngüsü

### 8.1. Retrieval Metrikleri

**Core Metrics:**
- **Hit Rate:** Top-k sonuçların relevancy oranı
- **Mean Reciprocal Rank (MRR):** İlk relevant result'ın pozisyonu
- **Normalized Discounted Cumulative Gain (nDCG):** Ranking quality
- **User Satisfaction:** Click-through rate, time on page

**Dashboard Metrics:**
> **Kod Durumu:** `Reference`
```typescript
interface MonitoringMetrics {
  retrieval: {
    avgLatencyMs: number;
    hitRateTop3: number;
    hitRateTop10: number;
    mrr: number;
    ndcgAt10: number;
  };
  quality: {
    avgQualityScore: number;
    lowQualityChunks: number;
    flaggedChunks: number;
  };
  usage: {
    dailyQueries: number;
    uniqueUsers: number;
    popularChunks: string[];
  };
  feedback: {
    userRatings: number[];
    feedbackCount: number;
    complaintRate: number;
  };
}
```

### 8.2. Otomatik Alert Kuralları

**Threshold Alerts:**
- Hit rate < 70% → Retrieval system check
- Avg latency > 500ms → Performance investigation
- Low quality chunks > 10% → Quality review triggered
- User rating < 3.0 → Manual review required

### 8.3. Feedback Collection

**User Feedback Types:**
- **Implicit:** Click behavior, time spent, copy actions
- **Explicit:** Star ratings, thumbs up/down, comments
- **Behavioral:** Query patterns, refinement attempts

**Feedback Processing Pipeline:**
> **Kod Durumu:** `Reference`
```typescript
interface FeedbackProcessor {
  collectFeedback(user: User, query: string, results: SearchResult[]): Feedback;
  analyzeFeedback(feedback: Feedback[]): Insight[];
  updateChunkScores(insights: Insight[]): void;
  retrainModel(): Promise<void>;
}
```

## 9. Test ve Doğrulama

### 9.1. Test Seti Oluşturma

**Benchmark Dataset:**
- **Size:** 1000+ query-chunk pairs
- **Diversity:** Different domains, difficulty levels
- **Ground Truth:** Human-annotated relevance scores
- **Versioning:** v1.0, v1.1, v2.0 with improvements

**Test Categories:**
- **Syntax queries:** "TypeScript interface syntax"
- **Concept queries:** "Dependency injection pattern"
- **Debug queries:** "TypeScript async/await error"
- **Best practice queries:** "React component optimization"

### 9.2. A/B Test Planı

**Test Groups:**
- **Control:** Mevcut retrieval system
- **Treatment A:** Quality-aware ranking (quality weight 0.2)
- **Treatment B:** Quality-aware ranking (quality weight 0.4)
- **Treatment C:** Dynamic quality weighting

**Success Metrics:**
- Primary: User satisfaction (+15% target)
- Secondary: Hit rate, latency, engagement

**Test Duration:** 4 weeks minimum

### 9.3. CI/CD Entegrasyonu

**Automated Tests:**
> **Kod Durumu:** `Production-Ready`
```yaml
# GitHub Actions example
name: RAG Quality Tests
on: [push, pullRequest]

jobs:
  quality-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run quality analyzer
        run: npm run analyze-quality
      - name: Check thresholds
        run: npm run check-quality-thresholds
      - name: Run copy-paste tests
        run: npm run test-copy-paste

```

---

## 16. REAL-TIME CHUNK QUALITY MONITORING

### Chunk Quality Degradation Detection

> **Kod Durumu:** `Reference`
```typescript
interface ChunkQualityMonitoring {
  chunkId: string;
  vectorId: string;
  createdAt: number;
  lastAccessedAt: number;
  
  // Access patterns
  accessCount: number;
  accessFrequency: 'high' | 'medium' | 'low' | 'dormant';
  
  // Quality tracking
  qualityScoreHistory: {
    timestamp: number;
    score: number;
    confidence: number;
  }[];
  
  // Degradation metrics
  semanticDriftPercentage: number; // 0-100
  confidenceTrendDirection: 'improving' | 'stable' | 'degrading';
  falseRetrievalRate: number; // Ratio of false positives
  
  // Recommendations
  updateRecommendation?: {
    reason: string;
    priority: 'critical' | 'high' | 'medium' | 'low';
    suggestedAction: 'update' | 'merge' | 'archive' | 'delete';
    estimatedBenefit: string;
  };
  
  // Lifecycle
  lastRefresh: number;
  nextScheduledReview: number;
  lifecycleStage: 'active' | 'flagged' | 'deprecated' | 'archived';
}

class ChunkMonitoringEngine {
  private qualityHistory: Map<string, ChunkQualityMonitoring> = new Map();
  private degradationThresholds = {
    semanticDrift: 20, // %
    falseRetrievalRate: 0.15, // 15%
    confidenceDecline: 0.25 // 25% drop
  };
  
  async monitorChunkQuality(
    chunk: Chunk,
    retrievalMetrics: RetrievalMetrics[]
  ): Promise<QualityAlert[]> {
    const monitoring = this.qualityHistory.get(chunk.id) || this.initializeMonitoring(chunk);
    const alerts: QualityAlert[] = [];
    
    // Calculate semantic drift
    const drift = await this.calculateSemanticDrift(chunk, retrievalMetrics);
    monitoring.semanticDriftPercentage = drift;
    
    if (drift > this.degradationThresholds.semanticDrift) {
      alerts.push({
        severity: 'high',
        type: 'semanticDrift',
        message: `Chunk semantic drift: ${drift.toFixed(1)}%`,
        recommendation: 'updateRequired'
      });
    }
    
    // Calculate false retrieval rate
    const falseRate = await this.calculateFalseRetrievalRate(chunk);
    monitoring.falseRetrievalRate = falseRate;
    
    if (falseRate > this.degradationThresholds.falseRetrievalRate) {
      alerts.push({
        severity: 'medium',
        type: 'falseRetrieval',
        message: `High false retrieval rate: ${(falseRate * 100).toFixed(1)}%`,
        recommendation: 'reviewRelevance'
      });
    }
    
    // Trend analysis
    const trend = this.analyzeTrend(monitoring.qualityScoreHistory);
    monitoring.confidenceTrendDirection = trend;
    
    if (trend === 'degrading') {
      monitoring.updateRecommendation = {
        reason: 'Quality score trending downward',
        priority: 'medium',
        suggestedAction: 'update',
        estimatedBenefit: 'Restore +3-5 point quality score'
      };
      
      alerts.push({
        severity: 'medium',
        type: 'qualityDegradation',
        message: 'Chunk quality trending downward',
        recommendation: 'scheduleUpdate'
      });
    }
    
    this.qualityHistory.set(chunk.id, monitoring);
    return alerts;
  }
  
  private async calculateSemanticDrift(
    chunk: Chunk,
    retrievalMetrics: RetrievalMetrics[]
  ): Promise<number> {
    // Compare chunk's current semantic position with historical embeddings
    const currentEmbedding = await this.getChunkEmbedding(chunk);
    const historicalEmbedding = this.getHistoricalEmbedding(chunk.id);
    
    if (!historicalEmbedding) return 0;
    
    // Cosine distance
    const distance = 1 - this.cosineSimilarity(currentEmbedding, historicalEmbedding);
    return Math.min(100, distance * 100); // Convert to percentage
  }
  
  private async calculateFalseRetrievalRate(chunk: Chunk): Promise<number> {
    // Query logs: how often was chunk retrieved but user didn't click/use?
    const retrievals = await this.getRecentRetrievals(chunk.id, 30); // Last 30 days
    
    if (retrievals.length === 0) return 0;
    
    const unused = retrievals.filter(r => !r.userInteracted).length;
    return unused / retrievals.length;
  }
  
  private analyzeTrend(history: { timestamp: number; score: number }[]): 
    'improving' | 'stable' | 'degrading' {
    if (history.length < 3) return 'stable';
    
    const recent = history.slice(-3);
    const avgRecent = recent.reduce((sum, h) => sum + h.score, 0) / recent.length;
    const avgPrevious = history.slice(-6, -3).reduce((sum, h) => sum + h.score, 0) / 3;
    
    const change = (avgRecent - avgPrevious) / avgPrevious;
    
    if (change > 0.05) return 'improving';
    if (change < -0.05) return 'degrading';
    return 'stable';
  }
  
  async generateMaintenanceTasks(): Promise<MaintenanceTask[]> {
    const tasks: MaintenanceTask[] = [];
    
    for (const [chunkId, monitoring] of this.qualityHistory) {
      if (monitoring.updateRecommendation) {
        tasks.push({
          chunkId,
          taskType: monitoring.updateRecommendation.suggestedAction,
          priority: monitoring.updateRecommendation.priority,
          reason: monitoring.updateRecommendation.reason,
          dueDate: monitoring.nextScheduledReview
        });
      }
    }
    
    return tasks.sort((a, b) => {
      const priorityOrder = { critical: 0, high: 1, medium: 2, low: 3 };
      return priorityOrder[a.priority] - priorityOrder[b.priority];
    });
  }
}

interface QualityAlert {
  severity: 'critical' | 'high' | 'medium' | 'low';
  type: string;
  message: string;
  recommendation: string;
}

interface MaintenanceTask {
  chunkId: string;
  taskType: 'update' | 'merge' | 'archive' | 'delete';
  priority: string;
  reason: string;
  dueDate: number;
}
```

---

## 17. AUTOMATIC DEDUPLICATION FRAMEWORK

### Semantic & Content Deduplication

> **Kod Durumu:** `Reference`
```typescript
interface DuplicateGroup {
  groupId: string;
  chunkIds: string[];
  deduplicationMethod: 'semanticSimilarity' | 'contentOverlap' | 'semanticHash';
  similarity: number; // 0-1
  recommendation: 'merge' | 'keepSeparate' | 'archiveDuplicates';
  mergeStrategy?: MergeStrategy;
}

interface MergeStrategy {
  keepChunkId: string;
  discardChunkIds: string[];
  mergeApproach: 'keepOriginal' | 'combineContent' | 'hierarchical';
  combinedContent?: string;
  metadata: Record<string, any>;
  updateReferences: boolean;
}

class DeduplicationEngine {
  private similarityThreshold = 0.85; // Threshold for duplicate detection
  private contentOverlapThreshold = 0.7; // 70% content overlap
  
  async detectDuplicates(chunks: Chunk[]): Promise<DuplicateGroup[]> {
    const groups: DuplicateGroup[] = [];
    const processed = new Set<string>();
    
    // Method 1: Semantic similarity (embeddings)
    const semanticDupes = await this.detectSemanticDuplicates(chunks);
    
    // Method 2: Content overlap (token-level)
    const contentDupes = this.detectContentDuplicates(chunks);
    
    // Method 3: Semantic hash (quick filtering)
    const hashDupes = this.detectHashDuplicates(chunks);
    
    // Combine results
    for (const dupe of semanticDupes) {
      if (!processed.has(dupe.groupId)) {
        groups.push(dupe);
        dupe.chunkIds.forEach(id => processed.add(id));
      }
    }
    
    return groups;
  }
  
  private async detectSemanticDuplicates(chunks: Chunk[]): Promise<DuplicateGroup[]> {
    const embeddings = await Promise.all(
      chunks.map(c => this.getEmbedding(c))
    );
    
    const groups: DuplicateGroup[] = [];
    const visited = new Set<number>();
    
    for (let i = 0; i < chunks.length; i++) {
      if (visited.has(i)) continue;
      
      const group: string[] = [chunks[i].id];
      visited.add(i);
      
      for (let j = i + 1; j < chunks.length; j++) {
        if (visited.has(j)) continue;
        
        const similarity = this.cosineSimilarity(embeddings[i], embeddings[j]);
        
        if (similarity >= this.similarityThreshold) {
          group.push(chunks[j].id);
          visited.add(j);
        }
      }
      
      if (group.length > 1) {
        groups.push({
          groupId: `dup-${Date.now()}-${i}`,
          chunkIds: group,
          deduplicationMethod: 'semanticSimilarity',
          similarity: this.similarityThreshold,
          recommendation: 'merge'
        });
      }
    }
    
    return groups;
  }
  
  private detectContentDuplicates(chunks: Chunk[]): DuplicateGroup[] {
    const groups: DuplicateGroup[] = [];
    const processed = new Set<string>();
    
    for (let i = 0; i < chunks.length; i++) {
      if (processed.has(chunks[i].id)) continue;
      
      const group = [chunks[i].id];
      processed.add(chunks[i].id);
      
      for (let j = i + 1; j < chunks.length; j++) {
        if (processed.has(chunks[j].id)) continue;
        
        const overlap = this.calculateContentOverlap(
          chunks[i].content,
          chunks[j].content
        );
        
        if (overlap >= this.contentOverlapThreshold) {
          group.push(chunks[j].id);
          processed.add(chunks[j].id);
        }
      }
      
      if (group.length > 1) {
        groups.push({
          groupId: `content-dup-${i}`,
          chunkIds: group,
          deduplicationMethod: 'contentOverlap',
          similarity: this.contentOverlapThreshold,
          recommendation: 'merge'
        });
      }
    }
    
    return groups;
  }
  
  private calculateContentOverlap(content1: string, content2: string): number {
    // Jaccard similarity of content tokens
    const tokens1 = new Set(content1.toLowerCase().split(/\W+/));
    const tokens2 = new Set(content2.toLowerCase().split(/\W+/));
    
    const intersection = new Set([...tokens1].filter(t => tokens2.has(t)));
    const union = new Set([...tokens1, ...tokens2]);
    
    return intersection.size / union.size;
  }
  
  async executeMerge(group: DuplicateGroup): Promise<MergeResult> {
    const strategy = this.selectMergeStrategy(group);
    
    switch (strategy.mergeApproach) {
      case 'keepOriginal':
        // Keep the highest quality chunk, remove others
        return await this.keepOriginal(strategy);
        
      case 'combineContent':
        // Combine content from all chunks
        return await this.combineContent(strategy);
        
      case 'hierarchical':
        // Create parent-child relationship
        return await this.createHierarchy(strategy);
    }
  }
  
  private selectMergeStrategy(group: DuplicateGroup): MergeStrategy {
    // Quality-based selection
    const qualityScores = group.chunkIds.map(id => ({
      id,
      score: this.getChunkQualityScore(id)
    }));
    
    const best = qualityScores.sort((a, b) => b.score - a.score)[0];
    
    return {
      keepChunkId: best.id,
      discardChunkIds: group.chunkIds.filter(id => id !== best.id),
      mergeApproach: 'keepOriginal',
      updateReferences: true,
      metadata: {
        mergedFrom: group.chunkIds,
        mergeQuality: best.score,
        mergeDate: new Date().toISOString()
      }
    };
  }
  
  async updateReferences(oldChunkIds: string[], newChunkId: string): Promise<void> {
    // Update all embeddings, indices, and references
    for (const oldId of oldChunkIds) {
      await this.updateEmbeddingIndex(oldId, newChunkId);
      await this.updateVectorDB(oldId, newChunkId);
      await this.updateMetadataIndex(oldId, newChunkId);
      await this.archiveChunk(oldId);
    }
  }
}
```

---

## 18. CHUNK VERSION CONTROL & ROLLBACK

### Versioning System

> **Kod Durumu:** `Reference`
```typescript
interface ChunkVersion {
  chunkId: string;
  versionNumber: number;
  content: string;
  metadata: any;
  
  // Quality snapshot
  qualityScore: number;
  evaluationMetrics: {
    semanticCoherence: number;
    retrievalEfficiency: number;
    contextualCompleteness: number;
    actionability: number;
  };
  
  // Change tracking
  createdAt: number;
  changedBy: string; // user ID or system
  changeReason: string;
  diffWithPrevious?: ContentDiff;
  
  // Validation
  validationStatus: 'pending' | 'approved' | 'rejected';
  approverNotes?: string;
  
  // Rollback info
  canRollBackTo: boolean;
  rollbackConstraints?: string[];
}

interface ContentDiff {
  previousVersion: number;
  additions: string[];
  deletions: string[];
  modifications: string[];
  diffSummary: string;
}

class ChunkVersionControl {
  private versionStore: Map<string, ChunkVersion[]> = new Map();
  private maxVersionsPerChunk = 50; // Keep history
  
  async createVersion(
    chunk: Chunk,
    changeReason: string,
    changedBy: string
  ): Promise<ChunkVersion> {
    let versions = this.versionStore.get(chunk.id) || [];
    const previousVersion = versions[versions.length - 1];
    
    const newVersion: ChunkVersion = {
      chunkId: chunk.id,
      versionNumber: versions.length + 1,
      content: chunk.content,
      metadata: chunk.metadata,
      qualityScore: await this.evaluateQuality(chunk),
      evaluationMetrics: await this.getEvaluationMetrics(chunk),
      createdAt: Date.now(),
      changedBy,
      changeReason,
      validationStatus: 'pending',
      canRollBackTo: true
    };
    
    if (previousVersion) {
      newVersion.diffWithPrevious = this.calculateDiff(
        previousVersion.content,
        newVersion.content
      );
    }
    
    versions.push(newVersion);
    
    // Keep only last N versions
    if (versions.length > this.maxVersionsPerChunk) {
      versions = versions.slice(-this.maxVersionsPerChunk);
      versions[0].canRollBackTo = false; // Can't rollback beyond history
    }
    
    this.versionStore.set(chunk.id, versions);
    return newVersion;
  }
  
  async rollbackToVersion(
    chunkId: string,
    targetVersion: number,
    reason: string
  ): Promise<RollbackResult> {
    const versions = this.versionStore.get(chunkId);
    if (!versions || targetVersion > versions.length) {
      throw new Error('Version not found');
    }
    
    const targetVersionObj = versions[targetVersion - 1];
    if (!targetVersionObj.canRollBackTo) {
      throw new Error('Cannot rollback beyond version history limit');
    }
    
    try {
      // Execute rollback
      const rolledBackChunk = {
        id: chunkId,
        content: targetVersionObj.content,
        metadata: targetVersionObj.metadata
      };
      
      await this.saveChunk(rolledBackChunk);
      
      // Create rollback entry
      const rollbackVersion = await this.createVersion(
        rolledBackChunk,
        `Rollback to v${targetVersion}: ${reason}`,
        'system'
      );
      
      return {
        success: true,
        previousVersion: versions.length,
        newVersion: rollbackVersion.versionNumber,
        message: `Rolled back to v${targetVersion}`
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  private calculateDiff(oldContent: string, newContent: string): ContentDiff {
    // Line-by-line diff
    const oldLines = oldContent.split('\n');
    const newLines = newContent.split('\n');
    
    const additions = newLines.filter(line => !oldLines.includes(line));
    const deletions = oldLines.filter(line => !newLines.includes(line));
    const modifications = [];
    
    for (let i = 0; i < Math.min(oldLines.length, newLines.length); i++) {
      if (oldLines[i] !== newLines[i]) {
        modifications.push({
          from: oldLines[i],
          to: newLines[i]
        });
      }
    }
    
    return {
      previousVersion: 0, // Set elsewhere
      additions,
      deletions,
      modifications,
      diffSummary: `+${additions.length}/-${deletions.length} lines, ${modifications.length} modified`
    };
  }
  
  async getVersionHistory(chunkId: string): Promise<ChunkVersion[]> {
    return this.versionStore.get(chunkId) || [];
  }
}

interface RollbackResult {
  success: boolean;
  previousVersion?: number;
  newVersion?: number;
  message?: string;
  error?: string;
}
```

---

## 🧩 v1.4 Canonical RAG Alignment

Bu bölüm, Decision Engine `v1.4` ile RAG katmanını query-rewrite, multi-hop ve source trust sözleşmeleri üzerinden hizalar.

### Query Rewrite + Multi-hop (Normatif)

> **Kod Durumu:** `Reference`
```typescript
interface QueryRewriteEngine {
  expand(query: string): string[];
  refineIntent(query: string, context: Context): string;
}

interface MultiHopPlanner {
  plan(question: string): RetrievalHop[];
}
```

Kural: Belirsiz sorgu önce `QueryRewriteEngine`, sonra `MultiHopPlanner` üzerinden yürütülür.

### Source Trust -> Safety/Decision Bridge

> **Kod Durumu:** `Reference`
```typescript
interface SourceTrustSignal {
  traceId: string;
  sourceId: string;
  sourceTrustScore: number;
  poisoningRisk: number;
  injected: boolean;
}
```

Kural: `sourceTrustScore` düşük veya `injected=true` ise chunk quarantine edilir ve Safety katmanına `E_SAFETY` sinyali gönderilir.

### Trace and Replay Compatibility

Kural: RAG trace alanları `TraceEnvelope` ile uyumlu olmalı; katman özel trace şeması üretilemez.

---

## 🧩 v1.4 Retrieval-to-Reasoning Semantics

### 1) Query Rewrite Runtime

> **Kod Durumu:** `Reference`
```typescript
interface QueryRewriteRuntime {
  rewrite(query: string, context: Context): {
    rewrittenQuery: string;
    expansions: string[];
    intentLabel: string;
  };
}
```

### 2) Multi-hop Execution Plan

> **Kod Durumu:** `Reference`
```typescript
interface RetrievalHop {
  hopId: string;
  question: string;
  requiredEvidence: string[];
}

interface MultiHopExecution {
  plan(question: string): RetrievalHop[];
  execute(hops: RetrievalHop[]): Promise<HopResult[]>;
  merge(results: HopResult[]): MergedEvidence;
}
```

### 3) Context Conflict Resolution

> **Kod Durumu:** `Reference`
```typescript
interface ContextConflictResolver {
  detectConflicts(evidence: MergedEvidence): Conflict[];
  resolve(conflicts: Conflict[], policy: 'trustWeighted' | 'recencyWeighted'): Resolution;
}
```

Kural: Retrieval tamamlanması tek başına başarı değildir; çelişkiler çözülmeden yanıt üretimi başlayamaz.

---

## 🧩 v1.5 Evidence-Orchestrated Reasoning

### 1) Evidence Graph Builder

> **Kod Durumu:** `Reference`
```typescript
interface EvidenceGraphBuilder {
  build(claims: string[], retrieved: RetrievedChunk[]): Promise<EvidenceGraph>;
}
```

### 2) Citation Confidence Merge

> **Kod Durumu:** `Reference`
```typescript
interface CitationConfidenceMerge {
  merge(graph: EvidenceGraph): Promise<{
    mergedConfidence: number;
    sourceConflicts: number;
    staleEvidenceRatio: number;
  }>;
}
```

### 3) Freshness Policy

| Risk Domain | Max Freshness Age |
|---|---|
| `critical` | `<= 7 days` |
| `high` | `<= 30 days` |
| `normal` | `<= 180 days` |

Kural: Freshness politikası sağlanmıyorsa yanıt `degraded` veya `clarification_required` olarak işaretlenir.

---

### 4) Contradiction Scoring Pipeline (Operational)

> **Kod Durumu:** `Reference`
```typescript
interface ContradictionScoringPipeline {
  detect(graph: EvidenceGraph): Promise<Array<{ claimId: string; conflictingSources: string[] }>>;
  score(conflicts: Array<{ claimId: string; conflictingSources: string[] }>): Promise<number>; // 0-1
}
```

Kural: `contradictionScore >= 0.35` ise cevap otomatik `needs_resolution` durumuna alınır.

### 5) Freshness-Weighted Citation Merge

> **Kod Durumu:** `Reference`
```typescript
interface CitationMergePolicy {
  merge(citations: Array<{
    sourceId: string;
    confidence: number;
    freshnessDays: number;
    authority: number;
  }>): Promise<{
    mergedConfidence: number;
    selectedSources: string[];
    droppedSources: string[];
  }>;
}
```

Kural: `freshness` ve `authority` ağırlıkları risk domain'e göre dinamik seçilir; tek kriter similarity olamaz.

### 6) Evidence Resolution Gate

Kural: `mergedConfidence < 0.7` veya conflict unresolved ise Decision katmanına `clarification_required` ve `trustPenalty` gönderilir.

---

### 7) Mandatory Evidence Resolver Core (Normatif Merkez)

> **Kod Durumu:** `Reference`
```typescript
interface EvidenceResolverCore {
  buildEvidenceGraph(query: string, chunks: RetrievedChunk[]): Promise<EvidenceGraph>;
  runContradictionScoring(graph: EvidenceGraph): Promise<number>;
  runFreshnessWeighting(graph: EvidenceGraph, riskDomain: 'critical' | 'high' | 'normal'): Promise<EvidenceGraph>;
  runCitationMerge(graph: EvidenceGraph): Promise<{ mergedConfidence: number; selectedSources: string[] }>;
  finalizeDecision(input: {
    contradictionScore: number;
    mergedConfidence: number;
  }): 'resolved' | 'clarification_required' | 'blocked';
}
```

Zorunlu sıra:

1. `buildEvidenceGraph`
2. `runContradictionScoring`
3. `runFreshnessWeighting`
4. `runCitationMerge`
5. `finalizeDecision`

Kural: Bu çekirdek pipeline çalışmadan retrieval çıktısı doğrudan generation katmanına geçemez.

---

> **RAG Rubric, production ortamına taşınabilir bir kalite ve operasyon referansı olarak kullanılabilir.**
