# 01 - Hızlı Başlangıç ve Mimari

Bu rehber, AI modeli sıfırdan eğitmek isteyenler için karar verme ve başlangıç kılavuzudur. Boş model nasıl oluşturulur, hangi durumda sıfırdan eğitim yapılır (veya yapılmaz), kritik config alanları nelerdir — hepsini burada bulacaksınız.

**Uçtan uca uygulama rehberi için:** [02 - Uçtan Uca Eğitim Rehberi](./02 - Uçtan Uca Eğitim Rehberi.md)

---

## Bu Rehber Kim İçin?

| Okuyucu Profili | Bu Rehberden Beklentisi |
|-----------------|------------------------|
| **Başlangıç** — AI/ML temeli zayıf | Karar ağacını okuyup doğru yolu seçmek; fine-tuning'in genellikle daha mantıklı olduğunu anlamak |
| **Orta** — Python biliyor, HF az tanıyor | Boş model oluşturmayı, config alanlarını ve mimari farklarını kavramak |
| **İleri** — Eğitim deneyimi var | Hızlı referans olarak config ve mimari tablolarına bakmak |

> **Ön koşul:** Python, temel PyTorch bilgisi ve Hugging Face `transformers` kütüphanesine aşinalık.

---

## Karar Ağacı: Hangi Yolu Seçmelisiniz?

```
Amacınız ne?
│
├─ Modeli belirli bir konuda uzmanlaştırmak
│  └─ ✅ FINE-TUNING (Önerilen)
│     ├─ Sınırlı veri (<100K satır) → LoRA / QLoRA
│     └─ Orta veri (>100K satır) → Full fine-tuning
│
├─ Tamamen yeni dil/alan için model yaratmak
│  └─ ⚠️ SIFIRDAN EĞİTİM
│     ├─ Küçük model (<1B params) → Tek GPU ile mümkün
│     └─ Büyük model (>7B params) → Multi-GPU / Cloud gerekli
│
├─ Domain-specific vocabulary (aşırı teknik tıp/hukuk vb.)
│  └─ Genel modellerin bilmediği binlerce terim var → Continual pre-training veya sıfırdan eğitim
│
├─ Sadece tokenizer değiştirmek
│  └─ Mevcut modele yeni tokenizer entegre et → Tokenizer eğitimi + embedding resize
│
└─ Deneyim / Öğrenme amaçlı
   └─ Küçük model (100M-200M) ile sıfırdan eğitim → Bu rehberi takip et
```

### 🔴 Önemli Uyarı: Fine-tuning Çoğu Durumda Daha Doğru Seçim

| Kriter | Sıfırdan Eğitim | Fine-tuning |
|--------|-----------------|-------------|
| **Veri gereksinimi** | Milyarlarca satır | Binlerce - yüz binlerce satır |
| **Donanım maliyeti** | $1,000 - $50,000+ | $0 - $500 |
| **Süre** | Haftalar - aylar | Saatler - günler |
| **Bilgi birikimi** | Sıfırdan başlar | Hazır bilgi üzerine inşa eder |
| **Ne zaman önerilir?** | Yeni dil, yeni alan, araştırma | Uzmanlaşma, stil değişikliği, domain adaptasyonu |

> **Kural:** Amacınız "model Türkçe konuşsun" veya "tıbbi metinler anlasın" ise, sıfırdan eğitim yerine mevcut bir Türkçe/çok dilli modeli fine-tuning etmek **%99 daha mantıklı ve ekonomiktir.**

### LoRA / QLoRA Hakkında Kısa Bilgi

Karar ağacında "LoRA / QLoRA" seçeneği görüyorsunuz. Bu, **fine-tuning** için kullanılan verimli bir yöntemdir. Sıfırdan eğitim yerine, mevcut bir modelin sadece küçük bir kısmını eğitir.

**Temel mantık:** Modelin tüm parametreleri yerine, sadece adaptör katmanlarını eğitir. Bu sayede:
- Daha az VRAM gerektirir
- Daha hızlı eğitim
- Daha az veri ile iyi sonuç

**Örnek LoRA konfigürasyonu:**

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,              # Rank — küçük = az parametre
    lora_alpha=32,     # Ölçekleme faktörü
    target_modules=["q_proj", "v_proj"],  # Hangi katmanlar eğitilecek
    lora_dropout=0.05,
    task_type="CAUSAL_LM"
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Çıktı: trainable params: 4,194,304 || all params: 6,742,339,584 || trainable%: 0.06
```

> **Not:** Bu rehber sıfırdan eğitime odaklıdır. LoRA/QLoRA fine-taining detayları için [02 - Uçtan Uca Eğitim Rehberi](./02 - Uçtan Uca Eğitim Rehberi.md) dosyasına bakın.

---

## Sıfırdan Model Eğitimi Nedir?

**Amaç:** Modelin iskeletini (mimarisini) belirleyip rastgele ağırlıklarla başlatmak.

**Ne zaman kullanılır:** Mevcut modellerin kapsamadığı yeni bir dil, alan veya mimari denemesi yapılacağı zaman.

**Temel Fark:**
- **Hazır model:** `.from_pretrained()` ile önceden eğitilmiş ağırlıkları yüklersiniz — model zaten bir şeyler biliyor.
- **Sıfırdan eğitim:** Sadece konfigürasyonu belirleyip modeli rastgele ağırlıklarla başlatırsınız — model hiçbir şey bilmiyor.

**Uygulama:**

```python
from transformers import LlamaConfig, LlamaForCausalLM

# Config ile mimariyi belirle
config = LlamaConfig(
    vocab_size=32000,
    hidden_size=2048,
    num_hidden_layers=12,
    num_attention_heads=16
)

# Boş model oluştur (rastgele ağırlıklar)
my_empty_model = LlamaForCausalLM(config)
```

**Sık hata:** Config'i oluşturup modeli `from_pretrained()` ile yüklemek — bu sıfırdan eğitim değil, hazır model yüklemektir.

---

## Boş Model Oluşturma

### Config ile Mimari Belirleme

Modelin "iskeletini" `Config` sınıfları ile tanımlarsınız. Kritik alanlar:

| Config Alanı | Açıklama | Önerilen Aralık |
|--------------|----------|-----------------|
| `vocab_size` | Kelime hazinesi boyutu | Tokenizer ile **birebir aynı** olmalı |
| `hidden_size` | Gizli katman boyutu | 512 (küçük) - 4096 (büyük) |
| `num_hidden_layers` | Transformer katman sayısı | 6 (küçük) - 32 (büyük) |
| `num_attention_heads` | Attention head sayısı | `hidden_size`'ın böleni olmalı |
| `max_position_embeddings` | Maksimum dizi uzunluğu | 512 - 8192 |
| `initializer_range` | Başlangıç ağırlık standart sapması | 0.02 (varsayılan); derin modellerde kritik |
| `rms_norm_eps` | RMSNorm epsilon değeri | 1e-6 (varsayılan); çok küçük → kararsızlık |
| `rope_scaling` | RoPE uzun dizi ölçekleme | Uzun bağlam için `"type": "linear"` |

> **⚠️ Kritik:** `vocab_size` değeri, eğiteceğiniz tokenizer'ın kelime hazinesi boyutu ile **tamamen aynı olmalıdır.** Uyuşmazlık `IndexError` veya `RuntimeError` ile eğitimi durdurur. Detay ve doğrulama yöntemi için [Eğitim Rehberi - Tokenizer bölümüne](./02 - Uçtan Uca Eğitim Rehberi.md#tokenizer-eğitimi) bakın.

### Positional Embeddings: RoPE vs Absolute

Modelin kelimelerin sırasını anlaması için positional embedding kullanılır. Mimari seçimi bu mekanizmayı belirler:

| Mimari | Positional Embedding Türü | Uzun Bağlam Desteği |
|--------|--------------------------|---------------------|
| **Llama / Mistral** | **RoPE** (Rotary) | `rope_scaling` ile 8K-32K'ya uzatılabilir |
| **GPT-2** | Absolute (öğrenilen) | `n_ctx` ile sınırlı, uzatılamaz |

**Sıfırdan eğitim için dikkat noktaları:**
- `max_position_embeddings`: Modelin görebileceği maksimum token sayısı. 512 kısa, 4096+ uzun bağlam için uygundur.
- **RoPE scaling:** Llama'da bağlam penceresini aşmak istiyorsanız:
```python
config = LlamaConfig(
    max_position_embeddings=8192,
    rope_scaling={"type": "linear", "factor": 2.0}  # 4K'yi 8K'ya uzatır
)
```
- **rms_norm_eps:** Çok küçük değerler (1e-8 gibi) sayısal kararsızlığa neden olabilir. Varsayılan `1e-6` değerinden şaşmayın.

> **İpucu:** RoPE kullanan modeller (Llama, Mistral) uzun metinlerde Absolute positional embedding'e (GPT-2) göre çok daha iyi performans gösterir. Sıfırdan eğitim için Llama mimarisi önerilir.

### GPT-2 Tarzı Boş Model

```python
from transformers import GPT2Config, GPT2LMHeadModel

config = GPT2Config(
    vocab_size=52000,
    n_ctx=512,
    n_embd=768,
    n_layer=6,
    n_head=12
)

model = GPT2LMHeadModel(config)
print(f"Parametre sayısı: {model.num_parameters():,}")
```

### Llama Tarzı Boş Model

```python
from transformers import LlamaConfig, LlamaForCausalLM

config = LlamaConfig(
    vocab_size=32000,
    hidden_size=2048,
    num_hidden_layers=12,
    num_attention_heads=16
)

my_empty_model = LlamaForCausalLM(config)
print(f"Parametre sayısı: {my_empty_model.num_parameters():,}")
```

### Ağırlık Başlatma (Weight Initialization)

Boş model oluşturulduğunda PyTorch katmanları varsayılan rastgele değerlerle başlatır. Ancak **derin modellerde** (12+ katman) bu başlangıç değerlerinin standart sapması eğitimin ilk 100 adımında divergene (patlamaya) neden olabilir.

**`initializer_range` parametresi:**
- Varsayılan değer: `0.02` (genelde yeterli)
- Derin modellerde (>24 katman) daha küçük değerler (`0.01`) deneyin
- Eğitimin ilk 50 adımında loss NaN oluyorsa, bu parametreyi küçültün

```python
config = LlamaConfig(
    vocab_size=32000,
    hidden_size=2048,
    num_hidden_layers=12,
    num_attention_heads=16,
    initializer_range=0.02,  # Varsayılan; derin modellerde 0.01 dene
)
```

**Gerekirse özel init fonksiyonu:**
```python
import torch.nn as nn

def init_weights(module):
    if isinstance(module, nn.Linear):
        nn.init.normal_(module.weight, mean=0.0, std=0.02)
        if module.bias is not None:
            nn.init.zeros_(module.bias)
    elif isinstance(module, nn.Embedding):
        nn.init.normal_(module.weight, mean=0.0, std=0.02)

my_empty_model.apply(init_weights)
```

> **İpucu:** İlk 100 adımda loss düzenli düşüyorsa başlatma doğru çalışıyor demektir. Loss NaN veya >100 oluyorsa `initializer_range`'i küçültün.

### Model Boyutu Referansı

| Boyut | hidden_size | num_layers | num_heads | Yaklaşık Parametre |
|-------|-------------|------------|-----------|-------------------|
| Mini | 512 | 6 | 8 | ~100M |
| Küçük | 768 | 12 | 12 | ~200M |
| Orta | 2048 | 12 | 16 | ~1B |
| Büyük | 4096 | 32 | 32 | ~7B |

### Donanım Gereksinimleri

Sıfırdan eğitim için VRAM gereksinimleri model boyutuna göre değişir. Detaylı VRAM tablosu için [03 - Küçültme, Dağıtım ve Tahminleme](./03 - Küçültme, Dağıtım ve Tahminleme.md) dosyasına bakın.

| Model Boyutu | Minimum VRAM (Eğitim) | Önerilen GPU |
|--------------|----------------------|--------------|
| 100M (Mini) | 4 GB | RTX 3060 (12GB) |
| 200M (Küçük) | 8 GB | RTX 3060 (12GB) |
| 1B (Orta) | 16 GB | RTX 3090/4090 (24GB) |
| 7B (Büyük) | 112 GB | 4x A100 40GB veya 2x A100 80GB |

> **Not:** Bu değerler fp16 eğitim içindir. Gradient checkpointing ile VRAM'i %60'a kadar düşürebilirsiniz, ancak eğitim yavaşlar.

---

## Model Mimarileri

GPT, Llama, Gemini ve Mistral gibi modellerin hepsi **Transformer (Decoder-only)** mimarisini kullanır.

| Mimari | Geliştirici | Ayırt Edici Özellik |
|--------|-------------|---------------------|
| **GPT** | OpenAI | "Sıradaki kelimeyi tahmin etme" mantığının öncüsü |
| **Llama** | Meta | RoPE rotary embeddings, SwiGLU aktivasyon, Grouped-Query Attention |
| **Mistral** | Mistral AI | Sliding window attention, GQA, verimli uzun dizi işleme |
| **Gemini** | Google | Özel mimari, multimodal yetenekler |

### Parametre Karşılaştırması

| Model | Parametre Sayısı | Özellik |
|-------|------------------|---------|
| Küçük Model (Örnek) | ~200M | Hızlı eğitilir, telefonda çalışabilir |
| Llama 3 8B | 8 Milyar | Çok daha zeki, daha çok bilgi tutar |
| GPT-4 | ~1.7 Trilyon | Devasa kütüphane gibi |

---

## Mini Checklist: Başlamadan Önce

- [ ] Karar ağacını okudum ve doğru yolu seçtim
- [ ] Fine-tuning'in daha mantıklı olup olmadığını değerlendirdim
- [ ] Domain-specific vocabulary (teknik terimler) gerekiyor mu değerlendirdim
- [ ] Model mimarisini (GPT-2 / Llama / Mistral) belirledim
- [ ] Config alanlarını anladım ve `vocab_size` kuralını biliyorum
- [ ] `initializer_range` ayarını kontrol ettim (derin modeller için)
- [ ] Donanım gereksinimlerini kontrol ettim

---

## Sonraki Adımlar

Boş model oluşturduktan sonra:

1. **Tokenizer eğitimi** — Kelimeleri sayılara dönüştüren sözlük oluşturma
2. **Veri seti hazırlama** — Eğitim için veri toplama ve temizleme
3. **Model eğitimi** — Pre-training, SFT, Alignment aşamaları
4. **Değerlendirme** — Model performansını test etme

➡️ **Tüm bu adımların detaylı uygulaması:** [02 - Uçtan Uca Eğitim Rehberi](./02 - Uçtan Uca Eğitim Rehberi.md)

---

## Öğrenme Kaynakları

- [Hugging Face Smol Course](https://huggingface.co/learn/smollm-course/)
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course/)
- [Andrej Karpathy - Let's build GPT](https://www.youtube.com/watch?v=kCc8FmEb1nY)
