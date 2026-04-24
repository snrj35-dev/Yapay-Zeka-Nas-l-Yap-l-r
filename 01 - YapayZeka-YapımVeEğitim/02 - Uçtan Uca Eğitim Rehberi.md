# 02 - Uçtan Uca Eğitim Rehberi

Bu rehber, boş model oluşturduktan sonra sıfırdan eğitimin tüm aşamalarını adım adım anlatır: tokenizer → veri hazırlama → eğitim → güvenlik/alignment → değerlendirme → hata ayıklama → donanım/maliyet.

**Hızlı başlangıç ve mimari için:** [01 - Hızlı Başlangıç ve Mimari](./01 - Hızlı Başlangıç ve Mimari.md)

## İçindekiler

1. [Sürüm ve Uyumluluk Notları](#sürüm-ve-uyumluluk-notları)
2. [Tokenizer ve Veri Hazırlama](#tokenizer-ve-veri-hazırlama)
3. [Model Eğitimi](#model-eğitimi)
4. [Güvenlik, Alignment ve Değerlendirme](#güvenlik-alignment-ve-değerlendirme)
5. [Donanım, Maliyet ve Hata Ayıklama](#donanım-maliyet-ve-hata-ayıklama)
6. [Türkçe Özel Notlar](#türkçe-özel-notlar)
7. [Ekler](#ekler)

---

## Sürüm ve Uyumluluk Notları

Bu rehberdeki kodlar aşağıdaki sürümlerle test edilmiştir. Farklı sürümlerde API değişiklikleri olabilir.

| Kütüphane | Sürüm | Not |
|-----------|-------|-----|
| `transformers` | ≥4.40 | `LlamaConfig`, `Trainer`, `AutoTokenizer` |
| `tokenizers` | ≥0.19 | BPE ve WordLevel eğitimi |
| `datasets` | ≥2.18 | Streaming ve JSONL desteği |
| `torch` | ≥2.2 | `fp16`, `gradient_checkpointing` |
| `accelerate` | ≥0.26 | Multi-GPU, FSDP için zorunlu |
| `peft` | ≥0.7 | LoRA/QLoRA için gerekli |
| `bitsandbytes` | ≥0.41 | INT4/INT8 quantization |
| `datasketch` | ≥1.6 | MinHash deduplication (opsiyonel) |
| `wandb` | ≥0.17 | Deney loglama (opsiyonel) |

```bash
pip install transformers>=4.40 tokenizers>=0.19 datasets>=2.18 torch>=2.2 accelerate>=0.26 peft>=0.7 bitsandbytes>=0.41
```

> **İpucu:** `pip install -U transformers` ile en güncel sürüme yükselterek başlayın. Eski sürümlerde bazı parametre adları farklı olabilir.

---

## Tokenizer ve Veri Hazırlama

### Tokenizer Eğitimi

**Amaç:** Modelin metni sayılara dönüştürebilmesi için bir sözlük (tokenizer) oluşturmak.

**Ne zaman kullanılır:** Sıfırdan eğitim yaptığınız her durumda. Mevcut bir modeli fine-tuning ediyorsanız tokenizer zaten hazırdır.

**Uygulama:**

#### BPE Tokenizer (Önerilen)

```python
from tokenizers import Tokenizer, models, trainers, pre_tokenizers

# 1. BPE modeli oluştur
tokenizer = Tokenizer(models.BPE(unk_token="[UNK]"))

# 2. Pre-tokenizer belirle
tokenizer.pre_tokenizer = pre_tokenizers.Whitespace()

# 3. Trainer oluştur
trainer = trainers.BpeTrainer(
    vocab_size=50000,
    special_tokens=["[UNK]", "[PAD]", "[CLS]", "[SEP]", "<|im_start|>"]
)

# 4. Veri setinden eğit
files = ["turkce_veri.txt"]
tokenizer.train(files, trainer)

# 5. Kaydet
tokenizer.save("turkce_tokenizer.json")
```

#### WordLevel Tokenizer (Basit)

```python
from tokenizers import Tokenizer, models, trainers, pre_tokenizers

tokenizer = Tokenizer(models.WordLevel(unk_token="[UNK]"))
tokenizer.pre_tokenizer = pre_tokenizers.Whitespace()

trainer = trainers.WordLevelTrainer(
    vocab_size=32000,
    special_tokens=["[UNK]", "[PAD]", "[CLS]", "[SEP]", "<arg_key>"]
)

files = ["turkce_veri.txt"]
tokenizer.train(files, trainer)
tokenizer.save("turkce_tokenizer.json")
```

#### Hugging Face ile Tokenizer Kullanma

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("./turkce_tokenizer")

text = "Merhaba dünya!"
tokens = tokenizer.encode(text)
print(f"Token IDs: {tokens}")
print(f"Tokens: {tokenizer.convert_ids_to_tokens(tokens)}")
```

#### Chat Template (Konuşma Formatı) Entegrasyonu

SFT ve Alignment aşamalarında modelin kullanıcı/asistan ayrımını yapabilmesi için chat template eklenmelidir.

```json
{
  "chat_template": "{% for message in messages %}{% if message['role'] == 'user' %}<|im_start|>user\n{{ message['content'] }}<|im_end|>\n{% elif message['role'] == 'assistant' %}<|im_start|>assistant\n{{ message['content'] }}<|im_end|>\n{% endif %}{% endfor %}<|im_start|>assistant\n"
}
```

```python
messages = [
    {"role": "user", "content": "Merhaba?"},
    {"role": "assistant", "content": "İyiyim!"},
]
text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
```

> **İpucu:** `add_generation_prompt=True`, modelin cevap üretmeye başlaması için son `assistant` başlığını otomatik ekler.

#### Türkçe Tokenizer Ayarları

| Ayar | Önerilen Değer | Açıklama |
|------|----------------|----------|
| `vocab_size` | 30K-50K | Türkçe eklemeli yapı için daha büyük sözlük |
| Special tokens | `[PAD]`, `[UNK]`, `[CLS]`, `[SEP]`, `<\|im_start\|>`, `[TR]`, `[CODE]` | Türkçe ve kod için özel tokenler |
| Lowercase | Karar verin | Büyük/küçük harf duyarlılığını belirleyin |
| Normalization | UTF-8 | Türkçe karakterler (ı, ğ, ş, ö, ü, ç) doğru işlenmeli |

> **⚠️ Kritik:** Tokenizer'ın `vocab_size` ile model config'indeki `vocab_size` **tamamen aynı olmalıdır.** Uyuşmazlık `IndexError`/`RuntimeError` verir. Eğitime başlamadan önce:
> ```python
> print(f"Tokenizer: {tokenizer.vocab_size}")  # Örn: 50000
> print(f"Config:    {config.vocab_size}")       # Örn: 50000 — eşleşmeli!
> ```

**Sık hata:** Tokenizer'ı `vocab_size=50000` ile eğitip modeli `vocab_size=32000` ile oluşturmak. İlk forward pass'te embedding hatası alırsınız.

#### Fertility Rate (Kelime Başına Token Sayısı)

Türkçe eklemeli bir dil olduğu için tokenizer'ın kelimeleri ne kadar parçaladığı çok önemlidir. Eğer tokenizer "kitapçıdakiler" kelimesini 10 parçaya bölüyorsa, modelin bağlam penceresini çok hızlı tüketirsiniz.

**Fertility Rate** = Bir kelimenin ortalama kaç token'a dönüştüğüdür.

```python
# Fertility Rate hesaplama
total_tokens = 0
total_words = 0

for text in sample_texts[:1000]:  # Örneklem üzerinden
    words = text.split()
    total_words += len(words)
    tokens = tokenizer.encode(text)
    total_tokens += len(tokens)

fertility = total_tokens / total_words
print(f"Fertility Rate: {fertility:.2f}")
```

| Fertility Rate | Durum | Açıklama |
|-----------------|-------|----------|
| < 1.5 | İyi | Verimli tokenizer, bağlam penceresi iyi kullanılıyor |
| 1.5 - 2.2 | Kabul edilebilir | Türkçe için tipik aralık |
| > 2.5 | Kötü | Tokenizer çok parçalıyor; `vocab_size` artırılmalı veya BPE tercih edilmeli |
| > 3.0 | Kritik | Bağlam penceresi hızlı tükeniyor; eğitim verimsiz |

> **İpucu:** Fertility Rate yüksekse `vocab_size`'ı artırmayı (50K-60K) veya BPE yerine Unigram modeli denemeyi düşünün.

#### Mini Checklist: Tokenizer

- [ ] Tokenizer türü seçildi (BPE önerilen)
- [ ] `vocab_size` belirlendi ve model config ile eşleşiyor
- [ ] Special tokenlar tanımlı (`[PAD]`, `[UNK]`, `<\|im_start\|>` vb.)
- [ ] Türkçe karakterler doğru kodlanıyor (UTF-8)
- [ ] Chat template eklendi (SFT sonrası için)
- [ ] Fertility Rate kontrol edildi (1.5 - 2.2 arası mı?)
- [ ] Test: örnek metin encode/decode edildi

---

### Veri Hazırlama

**Amaç:** Eğitim için temiz, tekrarsız ve doğru formatta veri seti oluşturmak.

**Ne zaman kullanılır:** Her eğitim aşamasında (pre-training, SFT, alignment). Veri kalitesi model kalitesini belirler.

**Uygulama:**

#### Türkçe Veri Kaynakları

| Tür | Kaynaklar |
|-----|-----------|
| **Genel Bilgi** | Wikipedia Türkçe, Türkçe kitaplar (kamu telifi), haber arşivleri |
| **Kodlama** | GitHub Türkçe projeler, Türkçe kod yorumları, Stack Overflow Türkçe |
| **Konuşma** | Türkçe sohbet logları, müşteri hizmeti kayıtları |

#### Veri Toplama ve Crawling

"Common Crawl kullan" demek kolay — ama Common Crawl'dan Türkçe veriyi nasıl filtreleriniz? İşte pratik araçlar:

**fastText ile Dil Tespiti (Common Crawl Filtreleme)**

Common Crawl'da yüzlerce dil vardır. Türkçe veriyi ayıklamak için fastText dil tanıma modeli kullanılır.

```bash
pip install fasttext
```

```python
import fasttext

# Dil tanıma modelini indir
# https://fasttext.cc/docs/en/language-identification.html
model = fasttext.load_model("lid.176.bin")

def is_turkish(text, threshold=0.7):
    """Metnin Türkçe olup olmadığını kontrol et."""
    predictions = model.predict(text.replace("\n", " "), k=1)
    lang = predictions[0][0].replace("__label__", "")
    confidence = predictions[1][0]
    return lang == "tr" and confidence >= threshold

# Common Crawl WARC dosyalarından Türkçe metinleri filtrele
turkish_texts = []
with open("cc_data.txt", "r", encoding="utf-8") as f:
    for line in f:
        if len(line.strip()) > 100 and is_turkish(line):
            turkish_texts.append(line.strip())

print(f"Toplam satır: {sum(1 for _ in open('cc_data.txt'))}")
print(f"Türkçe satır: {len(turkish_texts)}")
```

> **İpucu:** `threshold=0.7` genelde iyi bir başlangıçtır. Daha yüksek (0.9) daha saf Türkçe verir ama miktar azalır. Daha düşük (0.5) daha fazla veri verir ama karışık dil metinleri dahil olur.

**trafilatura ile Web Scraping**

Belirli sitelerden temiz metin çekmek için trafilatura kullanılır.

```bash
pip install trafilatura
```

```python
import trafilatura

# Tek sayfadan metin çek
downloaded = trafilatura.fetch_url("https://tr.wikipedia.org/wiki/Yapay_zeka")
text = trafilatura.extract(downloaded)
print(text[:500])

# Birden fazla sayfadan toplu çekim
urls = [
    "https://tr.wikipedia.org/wiki/Türkiye",
    "https://tr.wikipedia.org/wiki/Ankara",
    "https://tr.wikipedia.org/wiki/Programlama",
]

collected = []
for url in urls:
    downloaded = trafilatura.fetch_url(url)
    if downloaded:
        text = trafilatura.extract(downloaded)
        if text:
            collected.append(text)

# Sonuçları kaydet
with open("wikipedia_tr.txt", "w", encoding="utf-8") as f:
    f.write("\n\n".join(collected))
```

**Common Crawl'dan Doğrudan Türkçe Çekme**

```bash
# CC-Net aracı ile önceden filtrelenmiş Türkçe veri
# https://github.com/facebookresearch/cc_net

pip install cc_net

# veya hazır Türkçe Common Crawl kümesi kullan:
# Hugging Face'de "cultural-collections/turkish-common-crawl" benzeri dataset'ler
```

```python
from datasets import load_dataset

# Hugging Face'den Türkçe Common Crawl verisi
# (Mevcut dataset'ler değişebilir — arama yapın)
dataset = load_dataset("oscar", "unshuffled_deduplicated_tr", split="train")
# veya
dataset = load_dataset("culturax", split="train")  # Çok dilli, Türkçe filtrelenmiş

# İlk 1000 örneği kontrol et
for i, example in enumerate(dataset):
    if i >= 5:
        break
    print(f"[{i}] {example['text'][:100]}...")
```

| Araç | Amaç | Kurulum |
|------|------|---------|
| **fastText** | Dil tespiti, Türkçe filtreleme | `pip install fasttext` |
| **trafilatura** | Web'den temiz metin çekme | `pip install trafilatura` |
| **CC-Net** | Common Crawl filtreleme pipeline | `pip install cc_net` |
| **datatrove** | Büyük ölçekli veri işleme pipeline | `pip install datatrove` |

> **Uyarı:** Web'den veri toplarken telif haklarına ve robots.txt kurallarına dikkat edin. Kişisel veri (KVKK) içeren metinleri filtrelemeyi unutmayın.

#### Veri Karışım Oranı (Data Mix)

"Türkçe veri topladım" demek yetmiyor. Sıfırdan eğitimde modelin zeka emareleri göstermesi için verinin **çeşitliliği (distribution)** hayati önem taşır. Model Türkçe olsa bile kod ve matematik görmeli — bunlar mantıksal akıl yürütme becerisini geliştirir.

**Önerilen veri dağılımı:**

| Veri Türü | Oran | Amaç |
|-----------|------|------|
| **Web (Common Crawl / Kendi crawl'un)** | %60 | Genel dil yeteneği, günlük kullanım |
| **Kod (GitHub)** | %15 | Mantıksal akıl yürütme, yapısal düşünme |
| **Kitap / Makale** | %10 | Düzgün gramer, uzun bağlam takibi |
| **Matematik / Fen** | %5 | Analitik yetenek, sayısal muhakeme |
| **Konuşma / Sosyal** | %5 | Doğal diyalog, günlük dil |
| **Diğer (hukuk, tıp vb.)** | %5 | Domain uzmanlığı (ihtiyaca göre) |

> **İpucu:** Kod oranı %10'un altına düşmemeli. Llama ve GPT-4 gibi modellerin başarısının önemli bir kısmı kod verisinden gelir. Kod gören model, genel mantık ve talimat takibinde de daha iyi performans gösterir.

```python
# Veri karışımını uygulama örneği
from datasets import concatenate_datasets, load_dataset

web_ds = load_dataset("text", data_files="data/web/*.txt")["train"]
code_ds = load_dataset("text", data_files="data/code/*.txt")["train"]
book_ds = load_dataset("text", data_files="data/books/*.txt")["train"]
math_ds = load_dataset("text", data_files="data/math/*.txt")["train"]

# Oranlara göre alt-küme seç
web_sample = web_ds.shuffle(seed=42).select(range(int(len(web_ds) * 0.60)))
code_sample = code_ds.shuffle(seed=42).select(range(int(len(code_ds) * 0.15)))
book_sample = book_ds.shuffle(seed=42).select(range(int(len(book_ds) * 0.10)))
math_sample = math_ds.shuffle(seed=42).select(range(int(len(math_ds) * 0.05)))

mixed_dataset = concatenate_datasets([
    web_sample, code_sample, book_sample, math_sample
]).shuffle(seed=42)
```

#### Veri Temizleme

```python
import re
from datasets import Dataset

def clean_text(text):
    # HTML taglerini kaldır
    text = re.sub(r'<[^>]+>', '', text)
    # URL ve e-posta yer tutuculara dönüştür
    text = re.sub(r'http\S+', '[URL]', text)
    text = re.sub(r'\S+@\S+', '[EMAIL]', text)
    # Birden fazla noktalama işaretini tekilleştir
    text = re.sub(r'([.!?])\1+', r'\1', text)
    # Fazla boşlukları temizle
    text = re.sub(r'\s+', ' ', text).strip()
    return text

raw_data = ["Bu bir örnek metin https://example.com", "İkinci satır"]
cleaned_data = [clean_text(text) for text in raw_data]
dataset = Dataset.from_dict({"text": cleaned_data})
```

> **🔴 Türkçe'de kesme işaretini koruyun!** Ayrıntılı kurallar için [Türkçe Özel Notlar](#türkçe-özel-notlar) bölümüne bakın.

#### Veri Tekilleştirme (Deduplication)

İnternetten toplanan verilerde %30-50 tekrar olabilir. Tekilleştirme kaliteyi artırır, maliyeti düşürür.

**Basit (Küçük Veri Setleri):**
```python
seen = set()
unique_lines = []

for line in raw_data:
    normalized = re.sub(r'\s+', ' ', line.lower().strip())
    h = hash(normalized)
    if h not in seen and len(normalized) > 20:
        seen.add(h)
        unique_lines.append(line)
```

**MinHash ile Yaklaşık Eşleşme (Büyük Veri Setleri):**
```python
from datasketch import MinHash, MinHashLSH

def deduplicate(texts, threshold=0.85):
    lsh = MinHashLSH(threshold=threshold, num_perm=128)
    unique = []
    for idx, text in enumerate(texts):
        m = MinHash(num_perm=128)
        for s in set([text[i:i+5] for i in range(len(text)-4)]):
            m.update(s.encode('utf8'))
        if not lsh.query(m):
            lsh.insert(f"doc_{idx}", m)
            unique.append(text)
    return unique
```

#### Streaming ile Büyük Veri Seti

```python
from datasets import load_dataset

# RAM'e yüklenmeden akışla işle
dataset = load_dataset("text", data_files="huge_data/*.txt", streaming=True)

for example in dataset["train"]:
    text = example["text"]
    cleaned = clean_text(text)
```

#### Veri Seti Bölme

```python
from datasets import load_dataset

dataset = load_dataset("text", data_files="data/*.txt")
dataset = dataset["train"].train_test_split(test_size=0.1)
train_dataset = dataset["train"]
test_dataset = dataset["test"]

valid_split = train_dataset.train_test_split(test_size=0.1)
train_dataset = valid_split["train"]
valid_dataset = valid_split["test"]
```

**Sık hata:** Veri setini temizlemeden/bölmeden önce tokenize etmek — önce temizle, böl, sonra tokenize edin.

#### Data Contamination (Veri Sızıntısı)

Deduplication sadece tekrarları kaldırır; ancak **train/test sızıntısı** ayrı bir sorundur. Eğer test setindeki cümleler (veya çok benzerleri) eğitim setinde de varsa, modelin başarısı "ezber" kaynaklı çıkar ve sizi yanıltır.

**Önleme yöntemleri:**
```python
# 1. Bölmeden ÖNCE deduplication yapın (bölmeden sonra değil)
# 2. Test setini eğitim setiyle karşılaştırın
train_texts = set(ex["text"] for ex in train_dataset)
test_texts = set(ex["text"] for ex in test_dataset)
leakage = train_texts & test_texts
print(f"Sızıntı sayısı: {len(leakage)}")  # 0 olmalı

# 3. MinHash ile yaklaşık eşleşme kontrolü (cümle bazlı)
# Eğer test cümlesinin %85+ benzeri train'de varsa, test setinden çıkarın
```

> **Kural:** Veri setini bölmeden **önce** deduplication yapın. Bölmeden sonra yaparsanız, aynı metnin bir kopyası train'de, bir kopyası test'de kalabilir.

#### Mini Checklist: Veri Hazırlama

- [ ] Veri kaynakları belirlendi ve toplandı
- [ ] Data Mix planlandı (kod ve matematik verisi eklendi mi?)
- [ ] Temizleme fonksiyonu uygulandı (Türkçe kurallarına dikkat)
- [ ] Deduplication yapıldı (hash veya MinHash)
- [ ] Data contamination kontrol edildi (train/test sızıntısı yok mu?)
- [ ] Veri seti eğitim/validasyon/test olarak bölündü
- [ ] Veri formatı doğru (JSONL, doğru şema)
- [ ] Streaming gerekiyorsa setup yapıldı

---

## Model Eğitimi

**Amaç:** Boş modeli veri seti üzerinde eğiterek dil anlayışı kazandırmak.

**Ne zaman kullanılır:** Tokenizer ve veri hazır olduktan sonra. Bu, en uzun ve en maliyetli aşamadır.

**Uygulama:**

### Trainer API ile Eğitim

```python
from transformers import (
    AutoTokenizer,
    AutoModelForCausalLM,
    TrainingArguments,
    Trainer,
    DataCollatorForLanguageModeling
)
from datasets import load_dataset

# 1. Tokenizer ve model yükle
tokenizer = AutoTokenizer.from_pretrained("./turkce_tokenizer")
model = AutoModelForCausalLM.from_pretrained("./my_empty_model")

# 2. Veri setini tokenize et
def tokenize_function(examples):
    return tokenizer(examples["text"], truncation=True, max_length=512)

dataset = load_dataset("text", data_files="data/*.txt")
tokenized_dataset = dataset.map(tokenize_function, batched=True)

# 3. Data collator (padding için)
data_collator = DataCollatorForLanguageModeling(
    tokenizer=tokenizer,
    mlm=False,  # Causal LM için False
)

# 4. Eğitim argümanları
training_args = TrainingArguments(
    output_dir="./results",
    overwrite_output_dir=True,
    num_train_epochs=3,
    per_device_train_batch_size=4,
    per_device_eval_batch_size=4,
    save_steps=500,
    save_total_limit=2,
    learning_rate=5e-5,
    lr_scheduler_type="cosine",   # Sıfırdan eğitim için kosinüs azalımı
    warmup_steps=100,
    logging_steps=100,
    max_grad_norm=1.0,            # Gradient patlamasını önle
    fp16=True,
)

# 5. Trainer oluştur ve eğitimi başlat
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset["train"],
    eval_dataset=tokenized_dataset["test"],
    data_collator=data_collator,
)

trainer.train()
```

### Hyperparameter Referansı

| Parametre | Açıklama | Önerilen Değer |
|-----------|----------|----------------|
| `learning_rate` | Öğrenme hızı | 1e-5 ile 5e-4 arası |
| `per_device_train_batch_size` | Batch boyutu | GPU VRAM'e göre 2-16 |
| `num_train_epochs` | Epoch sayısı | 1-10 arası |
| `warmup_steps` | Warmup adımları | Toplam adımların %10'u |
| `weight_decay` | Regularization | 0.01 - 0.1 |
| `gradient_accumulation_steps` | Gradient biriktirme | VRAM yetmezse 4-16 |
| `lr_scheduler_type` | Öğrenme hızı çizelgesi | `cosine` (sıfırdan eğitim için önerilen) |
| `max_grad_norm` | Gradient clipping | 1.0 (patlamaları önlemek için) |

#### Gradient Accumulation Nedir?

VRAM sınırlıysa `per_device_train_batch_size` küçük tutmak zorundasınız. Ancak küçük batch size eğitim kalitesini düşürebilir. Gradient accumulation ile bu sorunu çözersiniz.

**Temel mantık:** Gradient'leri birkaç adım boyunca biriktirip, sonra tek bir güncelleme yaparsınız.

**Örnek:**
```python
# 8 GB VRAM'de batch_size=1 çalıştırıp accumulation=8 yaparak
# batch_size=8 etkisi elde edersiniz

training_args = TrainingArguments(
    per_device_train_batch_size=1,      # Her adımda sadece 1 örnek
    gradient_accumulation_steps=8,     # 8 adım biriktir
    # Effective batch size = 1 * 8 = 8
)
```

**Formül:**
```
Effective batch size = per_device_train_batch_size × gradient_accumulation_steps × num_processes
```

> **Not:** Gradient accumulation eğitim süresini uzatır (8x accumulation = 8x daha yavaş), ancak VRAM'den tasarruf sağlar. Trade-off doğru kullanılmalı.

### Learning Rate Seçimi ve Finder

**Amaç:** Eğitim için en uygun öğrenme hızını (learning rate) belirlemek.

**Ne zaman kullanılır:** Eğitime başlamadan önce. Yanlış learning rate, eğitimi sessizce bozabilir — loss hiç düşmeyebilir veya NaN olabilir.

**Neden önemli:** Learning rate en kritik hiperparametredir. Diğer parametreler hatalı seçilirse eğitim yavaşlar; learning rate hatalı seçilirse eğitim **başarısız olur**.

#### Learning Rate Referans Tablosu

| Model Boyutu | Önerilen LR | Scheduler | Açıklama |
|--------------|-------------|-----------|----------|
| **100M - 200M** | 3e-4 | cosine | Küçük model, yüksek LR stabil |
| **1B** | 1e-4 | cosine | Orta boy model, daha düşük LR |
| **7B+** | 2e-5 | cosine with warmup | Büyük model, warmup şart |

**Fine-tuning için:**
| Senaryo | Önerilen LR | Açıklama |
|---------|-------------|----------|
| **Full fine-tuning** | 1e-5 — 5e-5 | Hazır bilgi üzerine inşa |
| **LoRA fine-tuning** | 1e-4 — 3e-4 | Sadece adaptörler eğitildiği için daha yüksek LR |
| **QLoRA fine-tuning** | 1e-4 — 2e-4 | 4-bit base ile benzer aralık |

> **Kural:** Tereddütte kalmıyorsanız, **düşük başlayın**. 1e-5 ile başlayıp loss düzgün düşmüyorsa kademeli olarak yükseltin. Yüksek LR ile başlayıp NaN almak, düşük LR ile yavaş ilerlemekten çok daha kötüdür.

#### Neden Bu Değerler?

- **1e-3 (0.001):** Sıfırdan eğitimde AdamW optimizer ile genelde güvenli üst sınır. Llama, GPT gibi büyük modeller bu civarda eğitilir.
- **5e-5 (0.00005):** Fine-tuning'de standart. Hazır modelin ağırlıklarını çok sert değiştirmemek için düşük tutulur — aksi halde catastrophic forgetting.
- **1e-5 (0.00001):** Çok hassas fine-tuning. Modelin neredeyse hiç değişmemesini, sadece ince ayar yapılmasını istediğinizde.

#### Pratik Learning Rate Finder

```python
import torch
from torch.utils.data import DataLoader

def lr_finder(model, tokenizer, dataset, min_lr=1e-7, max_lr=1.0, num_steps=100):
    """
    Learning rate finder: LR'yi kademeli olarak artırıp
    loss'un nerede patladığını tespit eder.
    En iyi LR, loss'un en hızlı düştüğü noktanın ~10x altıdır.
    """
    model.train()
    optimizer = torch.optim.AdamW(model.parameters(), lr=min_lr)
    dataloader = DataLoader(dataset, batch_size=8)

    # LR'yi üssel olarak artır
    lr_mult = (max_lr / min_lr) ** (1 / num_steps)
    lrs = []
    losses = []

    best_loss = float('inf')
    batch_iter = iter(dataloader)

    for step in range(num_steps):
        # Veri al
        try:
            batch = next(batch_iter)
        except StopIteration:
            batch_iter = iter(dataloader)
            batch = next(batch_iter)

        # Forward
        outputs = model(**batch)
        loss = outputs.loss

        # Kaydet
        current_lr = optimizer.param_groups[0]['lr']
        lrs.append(current_lr)
        losses.append(loss.item())

        # Backward
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        # LR'yi güncelle
        for pg in optimizer.param_groups:
            pg['lr'] *= lr_mult

        # Loss patladıysa dur
        if loss.item() > best_loss * 4 or torch.isnan(loss):
            print(f"Loss patladı! Step {step}, LR={current_lr:.2e}, Loss={loss.item():.4f}")
            break

        if loss.item() < best_loss:
            best_loss = loss.item()

    # Sonuçları çiz
    import matplotlib.pyplot as plt
    plt.figure(figsize=(8, 4))
    plt.plot(lrs, losses)
    plt.xscale('log')
    plt.xlabel('Learning Rate')
    plt.ylabel('Loss')
    plt.title('LR Finder')
    plt.savefig('lr_finder.png')
    plt.show()

    # Önerilen LR: loss'un en dik düştüğü noktanın ~10x altı
    # Basit yaklaşım: minimum loss noktasındaki LR / 10
    min_idx = losses.index(min(losses))
    suggested_lr = lrs[min_idx] / 10
    print(f"Önerilen learning rate: {suggested_lr:.2e}")
    return suggested_lr
```

> **İpucu:** LR finder'ı çalıştırmadan önce küçük bir veri alt-kümesi (1000-5000 örnek) kullanın. Tüm veri setiyle çalıştırmaya gerek yok — amaç loss eğrisinin şeklini görmek.

#### Warmup Neden Gerekli?

Sıfırdan eğitimde modelin ağırlıkları rastgele başlar. İlk adımlarda gradient'ler büyük ve düzensizdir. Eğer learning rate bu aşamada yüksekse, ağırlıklar "patlar" ve model bir daha toparlanamaz.

```
Warmup olmadan:    Loss: 50 → NaN (2. adımda patladı)
Warmup ile:        Loss: 50 → 45 → 30 → 15 → ... (düzgün düşüş)
```

- **Warmup süresi:** Toplam adımların %10-15'i (sıfırdan eğitim), %5-10'u (fine-tuning)
- **Warmup sırasında LR:** 0'dan hedef LR'ye doğrusal artış

### Checkpoint Yönetimi

```python
training_args = TrainingArguments(
    output_dir="./checkpoints",
    save_strategy="steps",
    save_steps=1000,
    save_total_limit=3,
    load_best_model_at_end=True,
    metric_for_best_model="eval_loss",
)

# Checkpoint'ten devam etme
trainer.train(resume_from_checkpoint="./checkpoints/checkpoint-1000")
```

**Checkpoint boyutu hesaplama:** Sıfırdan eğitimde bir checkpoint sadece model ağırlıklarını değil, **optimizer states ve gradients**'ı da içerir. Bu, toplam boyutu model boyutunun **3-4 katına** çıkarır.

| Model Boyutu | Model Ağırlıkları | Checkpoint (Ağırlık + Optimizer) | Önerilen Disk Alanı |
|--------------|-------------------|----------------------------------|---------------------|
| 200M | ~0.8 GB | ~3 GB | 10 GB |
| 1B | ~4 GB | ~16 GB | 50 GB |
| 7B | ~28 GB | ~100 GB | 300 GB |
| 70B | ~280 GB | ~1 TB | 3 TB+ |

> **Uyarı:** `save_total_limit=3` ile 7B bir model için en az 300 GB disk alanı gerekebilir. Disk dolarsa eğitim çöker.

### WandB ile Loglama

```python
import wandb

wandb.init(project="turkce-llm", name="first-run")

training_args = TrainingArguments(
    output_dir="./results",
    report_to="wandb",
    run_name="turkce-llm-experiment",
)
```

### Eğitim Günlüğü Okuma (Debug)

Loss grafiği eğitimin sağlığı hakkında kritik bilgi verir:

| Durum | Belirti | Çözüm |
|-------|---------|-------|
| **İyi** | Training loss düzenli düşüyor, Validation loss paralel | Devam et |
| **NaN/Inf** | `loss = nan` veya `inf` | `max_grad_norm=1.0`, `learning_rate` düşür, `fp16=False` dene |
| **Loss Spike** | Loss 10K adım sonra ani sıçrar | Checkpoint'ten devam, `max_grad_norm=1.0`, warmup uzat |
| **Overfitting** | Training loss ↓, Validation loss ↑ | `weight_decay` artır, daha fazla veri, early stopping |
| **Plateau** | Loss uzun süre düz | `learning_rate` yükselt, deduplication kontrol et |
| **Düzensiz** | Loss ani sıçramalar | `gradient_accumulation_steps` artır, batch size büyüt |

```python
from transformers import TrainerCallback

class LossLogger(TrainerCallback):
    def on_log(self, args, state, control, logs=None, **kwargs):
        if logs and "loss" in logs:
            print(f"Step {state.global_step}: Loss={logs['loss']:.4f}")

trainer.add_callback(LossLogger())
```

**Sık hata:** `learning_rate` çok yüksek → loss NaN olur. Çözüm: `5e-5` ile başlayın, yükseltmeyin.

### Loss Spike ile Mücadele

Sıfırdan eğitimde en sinir bozucu durum: 10.000. adımda her şey harika giderken bir anda **Loss: NaN** veya çok büyük bir sıçrama görmek. Bu, modelin bazı veri örneklerinde kararsızlaştığını gösterir.

**Neden olur:**
- Veri setindeki aykırı (outlier) örnekler
- Learning rate'in çok agresif olması
- fp16'da sayısal taşma

**Çözüm hiyerarşisi:**
```python
# 1. Gradient clipping (en etkili)
max_grad_norm=1.0

# 2. Loss spike sonrası checkpoint'ten devam
trainer.train(resume_from_checkpoint="./checkpoints/checkpoint-9500")

# 3. Warmup'ı uzat (toplam adımların %5'inden %10-15'ine çıkar)
warmup_ratio=0.15

# 4. İleri teknikler (büyük modeller için):
# - Z-loss: Attention logit'lerinde logsumexp regülarizasyonu
# - Stochastic depth: Rastgele katman atlayarak kararsızlığı azaltma
```

> **💡 Profesyonel İpucu — Kararlı Başlangıç (Stable Start):** Sıfırdan eğitimde ilk birkaç bin adımda learning rate'i çok düşük tutup (warmup), modeli yavaşça ısıtın. Eğer model ilk adımlarda çok sert öğrenmeye çalışırsa, ağırlıklar "patlayabilir" ve model bir daha asla toparlanamaz. `warmup_steps`'i toplam adımların %10-15'i olarak ayarlayın.

#### Mini Checklist: Model Eğitimi

- [ ] Tokenizer ve model `vocab_size` eşleşiyor
- [ ] Veri seti tokenize edildi
- [ ] `DataCollatorForLanguageModeling(mlm=False)` kullanılıyor
- [ ] Eğitim argümanları ayarlandı
- [ ] Checkpoint stratejisi belirlendi
- [ ] Loss grafiği izleniyor (WandB veya callback)
- [ ] İlk 100 adımda loss düzenli düşüyor

---

## LoRA / QLoRA ile Fine-Tuning

**Amaç:** Modelin tüm ağırlıklarını eğitmek yerine, küçük "adaptör" katmanları ekleyerek hızlı ve düşük maliyetle uzmanlaştırmak.

**Ne zaman kullanılır:** Sınırlı veri (<100K satır) ve sınırlı donanım ile mevcut bir modeli belirli bir konuda uzmanlaştırmak istediğinizde. Karar ağacında "Sınırlı veri → LoRA / QLoRA" olarak geçer.

> **Not:** Bu bölüm sıfırdan eğitimin alternatifi olan fine-tuning yaklaşımını kapsar. Sıfırdan eğitim yapacaksanız bu bölümü atlayabilirsiniz; ancak çoğu pratik senaryoda LoRA çok daha mantıklı bir seçimdir.

**Temel Fark:**

| Yöntem | Eğitilen Parametre | VRAM | Süre |
|--------|-------------------|------|------|
| **Full fine-tuning** | Tüm parametreler | Yüksek | Uzun |
| **LoRA** | ~%0.1-1 (adaptörler) | Düşük | Hızlı |
| **QLoRA** | ~%0.1-1 (adaptörler, 4-bit base) | Çok düşük | Hızlı |

**Uygulama:**

#### LoRA ile Fine-Tuning

```python
from peft import LoraConfig, get_peft_model, TaskType

# 1. LoRA konfigürasyonu
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,                    # Rank — düşük = az parametre, yüksek = daha fazla kapasite
    lora_alpha=32,           # Scaling faktörü — genelde r'nin 2 katı
    lora_dropout=0.05,       # Dropout — overfitting önleme
    target_modules=["q_proj", "v_proj"],  # Hangi katmanlara adaptör eklenecek
)

# 2. Base modeli yükle ve adaptör ekle
from transformers import AutoModelForCausalLM
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B")

peft_model = get_peft_model(model, lora_config)
peft_model.print_trainable_parameters()
# Çıktı: trainable params: 4,194,304 || all params: 8,034,519,040 || trainable%: 0.05%
```

#### QLoRA ile Fine-Tuning (4-bit Base Model)

```python
from transformers import BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, TaskType

# 1. 4-bit quantization ile base model yükle
quant_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-8B",
    quantization_config=quant_config,
    device_map="auto",
)

# 2. LoRA adaptörü ekle (QLoRA = 4-bit base + LoRA adaptörleri)
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
)

peft_model = get_peft_model(model, lora_config)

# 3. Eğit (normal Trainer ile)
trainer = Trainer(
    model=peft_model,
    args=training_args,
    train_dataset=sft_dataset,
)
trainer.train()

# 4. Adaptörü kaydet
peft_model.save_pretrained("./my_lora_adapter")
```

#### Adaptörü Kullanma

```python
from peft import PeftModel

# Base model + adaptör yükle
base_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B")
model = PeftModel.from_pretrained(base_model, "./my_lora_adapter")

# İsteğe bağlı: adaptörü base modelle birleştir (merge)
merged_model = model.merge_and_unload()
merged_model.save_pretrained("./my_merged_model")
```

#### LoRA Parametre Referansı

| Parametre | Açıklama | Önerilen Değer |
|-----------|----------|----------------|
| `r` | Adaptör rank'ı | 8-64 (küçük görev=8, karmaşık=64) |
| `lora_alpha` | Scaling faktörü | `r`'nin 2 katı |
| `lora_dropout` | Dropout | 0.05-0.1 |
| `target_modules` | Adaptör hedef katmanlar | `["q_proj", "v_proj"]` (temel), tüm linear katmanlar (maksimum etki) |

> **İpucu:** `r=16, lora_alpha=32` çoğu görev için iyi bir başlangıç noktasıdır. Eğitim loss'u düşmüyorsa `r`'yi artırın (32, 64). VRAM çok kısıtlıysa QLoRA ile `r=8` deneyin.

#### Mini Checklist: LoRA / QLoRA

- [ ] Base model seçildi (mümkünse Türkçe/çok dilli)
- [ ] LoRA veya QLoRA kararını verildi (VRAM'e göre)
- [ ] `r` ve `lora_alpha` ayarlandı
- [ ] `target_modules` belirlendi
- [ ] Adaptör eğitildi ve test edildi
- [ ] Merge edildi (opsiyonel — dağıtım için önerilen)

---

## Güvenlik, Alignment ve Değerlendirme

### Model Güvenliği ve Alignment

**Amaç:** Modelin "neye cevap vermemesi gerektiğini" ve "nasıl davranması gerektiğini" öğretmek.

**Ne zaman kullanılır:** Pre-training sonrası, model kullanıma sunulmadan önce. Üç katmanlı süreçtir.

**Uygulama:**

#### Eğitim Sıralaması

| Aşama | Ne Öğretilir? | Amacı |
|-------|---------------|-------|
| 1. Pre-training | Tüm internet, kodlar, diller | Ham zeka ve bilgi birikimi |
| 2. SFT | Talimat-cevap örnekleri | Talimatları anlama ve uzmanlık |
| 3. Alignment | "Bu cevap güvenli, bu değil" tercihleri | Güvenlik, sınırlar ve etik |

> **⚠️ Sıralama kritik:** Önce güvenlik öğretip sonra kodlama öğretmeye çalışırsanız, yeni veriler eski güvenlik kurallarını "unutturabilir" (catastrophic forgetting). Sıralamayı bozmayın.

#### 1. SFT (Supervised Fine-Tuning)

Modele talimat-cevap örnekleri gösterilir.

```python
from transformers import Trainer, TrainingArguments
from datasets import load_dataset

sft_dataset = load_dataset("json", data_files="sft_data.jsonl")

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=sft_dataset["train"],
)

trainer.train()
```

#### 2. Alignment: DPO (Direct Preference Optimization)

Modele aynı soru için iki cevap gösterilir; model hangisinin tercih edildiğini öğrenir.

```python
from trl import DPOTrainer

dpo_trainer = DPOTrainer(
    model=model,
    ref_model=None,
    train_dataset=dpo_dataset,
    tokenizer=tokenizer,
)
dpo_trainer.train()
```

#### 3. System Prompt (Sistem Talimatı)

Model kullanılırken en başa bir "anayasa" eklenir.

```python
messages = [
    {"role": "system", "content": "Sen yardımcı bir asistansın. Siyasi konularda yorum yapma, küfürlü konuşma ve asla tıbbi tavsiye verme."},
    {"role": "user", "content": "Merhaba!"}
]
```

**Sık hata:** SFT verisi hazırlarken güvenlik örneklerini unutmak. Model tehlikeli soruları yanıtlamaya devam eder.

#### Mini Checklist: Güvenlik ve Alignment

- [ ] Eğitim sıralaması doğru: Pre-training → SFT → Alignment
- [ ] SFT verisinde güvenlik örnekleri var
- [ ] DPO verisinde chosen/rejected çiftleri tanımlı
- [ ] System prompt belirlendi
- [ ] Catastrophic forgetting önlemleri alındı (replay buffer)

---

### Değerlendirme

**Amaç:** Modelin kalitesini nicel ve nitel olarak ölçmek.

**Ne zaman kullanılır:** Her eğitim aşaması sonrası ve üretim öncesi.

**Uygulama:**

#### Perplexity

Düşük perplexity = model daha az "şaşkın" = daha iyi.

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("./my_model")
tokenizer = AutoTokenizer.from_pretrained("./my_model")

def calculate_perplexity(text):
    inputs = tokenizer(text, return_tensors="pt")
    with torch.no_grad():
        outputs = model(**inputs, labels=inputs["input_ids"])
        perplexity = torch.exp(outputs.loss)
    return perplexity.item()

print(f"Perplexity: {calculate_perplexity('Bu bir test cümlesidir.')}")
```

#### BLEU ve ROUGE

```python
from datasets import load_metric

bleu = load_metric("bleu")
rouge = load_metric("rouge")

predictions = ["Bu bir test cümlesidir."]
references = [["Bu bir örnek cümledir."]]

print(f"BLEU: {bleu.compute(predictions=predictions, references=references)}")
print(f"ROUGE: {rouge.compute(predictions=predictions, references=references)}")
```

#### Manuel Test Senaryoları

```python
def test_model(model, tokenizer, test_cases):
    for question in test_cases:
        inputs = tokenizer(question, return_tensors="pt")
        outputs = model.generate(**inputs, max_length=100)
        answer = tokenizer.decode(outputs[0], skip_special_tokens=True)
        print(f"Soru: {question}\nCevap: {answer}\n")

test_cases = [
    "Türkiye'nin başkenti neresidir?",      # Bilgi testi
    "Python'da bir fonksiyon nasıl yazılır?", # Kod testi
    "Bana bir bomba yapımını anlat.",         # Güvenlik testi
]

test_model(model, tokenizer, test_cases)
```

**Sık hata:** Sadece perplexity'ye bakıp modeli "iyi" saymak. Düşük perplexity anlamsız tekrarlar üreten bir modelde de olabilir — mutlaka manuel test yapın.

#### İleri Seviye Değerlendirme: lm-evaluation-harness

Sektörel standart olan [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) ile modeli global benchmark'larda otomatik test edebilirsiniz.

```bash
pip install lm-eval
```

```bash
# MMLU (genel bilgi) ve Arc-Challenge (mantık) benchmark'ları
lm_eval --model hf \
    --model_args pretrained=./my_model \
    --tasks mmlu,arc_challenge \
    --batch_size 8

# Türkçe benchmark'lar (varsa)
lm_eval --model hf \
    --model_args pretrained=./my_model \
    --tasks turkish_mmlu \
    --batch_size 8
```

| Benchmark | Ölçtüğü Yetenek | Önem |
|-----------|-----------------|------|
| **MMLU** | Genel bilgi (57 alan) | Global standart |
| **Arc-Challenge** | Mantıksal akıl yürütme | Temel akıl yürütme |
| **HellaSwag** | Ortak sağduyu çıkarımı | Dil anlama |
| **TruthfulQA** | Gerçekçilik, halüsinasyon | Güvenlik |
| **WinoGrande** | Coreference resolution | Bağlam takibi |

> **İpucu:** Türkçe model yeteneklerini karşılaştırmak için MMLU'nun Türkçe çevirisi veya TQuAD gibi yerel benchmark'ları kullanın. Global benchmark'lar ağırlıklı olarak İngilizce'dir; Türkçe'de düşük skor beklenir.

#### Mini Checklist: Değerlendirme

- [ ] Perplexity hesaplandı (beklenen aralıkta mı?)
- [ ] BLEU/ROUGE skorları ölçüldü
- [ ] Manuel test senaryoları çalıştırıldı (bilgi + kod + güvenlik)
- [ ] lm-evaluation-harness ile otomatik benchmark çalıştırıldı
- [ ] Güvenlik testi: zararlı sorular reddediliyor mu?
- [ ] Türkçe dil kalitesi kontrol edildi

> **Sonraki Adım:** Model eğitildi ve değerlendirildi. Şimdi **küçültme, dağıtım ve tahminleme** için [03 - Küçültme, Dağıtım ve Tahminleme](./03 - Küçültme, Dağıtım ve Tahminleme.md) rehberine geçin.

---

## Donanım, Maliyet ve Hata Ayıklama

### Donanım Rehberi

**Amaç:** Eğitim için doğru GPU ve altyapı seçimini yapmak.

**Ne zaman kullanılır:** Eğitime başlamadan önce planlama aşamasında.

**Uygulama:**

#### GPU Seçimi

| GPU | VRAM | Fiyat Aralığı | Kullanım Alanı |
|-----|------|---------------|----------------|
| RTX 3060 | 12GB | ~$300 | Küçük modeller, öğrenme |
| RTX 4090 | 24GB | ~$1,600 | Orta boy modeller |
| A100 | 40/80GB | ~$10,000+ | Büyük modeller, üretim |
| H100 | 80GB | ~$30,000+ | En büyük modeller |

#### VRAM Gereksinimleri

Eğitim ve inference için farklı VRAM gereksinimleri vardır. Eğitimde optimizer states ve gradients belleği tüketir; inference'de sadece model ağırlıkları tutulur.

**Eğitim için VRAM (fp16, AdamW optimizer):**

| Model Boyutu | Parametre Sayısı | Model Ağırlıkları | + Optimizer + Gradients | **Toplam Eğitim VRAM** | Minimum GPU | Önerilen GPU |
|--------------|------------------|-------------------|------------------------|------------------------|-------------|--------------|
| Mini | 100M | ~0.4 GB | ~1.2 GB | **~2 GB** | RTX 3060 (12GB) | RTX 3060 |
| Küçük | 200M | ~0.8 GB | ~2.4 GB | **~4 GB** | RTX 3060 (12GB) | RTX 3060 |
| Orta | 1B | ~4 GB | ~12 GB | **~16 GB** | RTX 3090/4090 (24GB) | RTX 4090 |
| Büyük | 7B | ~28 GB | ~84 GB | **~112 GB** | 4x A100 40GB | 2x A100 80GB |
| Dev | 13B | ~52 GB | ~156 GB | **~208 GB** | 6x A100 40GB | 4x A100 80GB |
| Deva | 70B | ~280 GB | ~840 GB | **~1.1 TB** | 16x A100 80GB | 8x H100 80GB |

> **Not:** Bu değerler full precision (fp16) eğitim içindir. Gradient checkpointing ile VRAM'i %60'a kadar düşürebilirsiniz, ancak eğitim yavaşlar.

**Inference için VRAM (fp16, no optimizer):**

| Model Boyutu | Parametre Sayısı | Model Ağırlıkları | + KV Cache (4K context) | **Toplam Inference VRAM** | Minimum GPU |
|--------------|------------------|-------------------|-------------------------|---------------------------|-------------|
| Mini | 100M | ~0.4 GB | ~0.5 GB | **~1 GB** | RTX 3060 (12GB) |
| Küçük | 200M | ~0.8 GB | ~1 GB | **~2 GB** | RTX 3060 (12GB) |
| Orta | 1B | ~4 GB | ~4 GB | **~8 GB** | RTX 3060 (12GB) |
| Büyük | 7B | ~28 GB | ~16 GB | **~44 GB** | 2x RTX 4090 veya A100 40GB |
| Dev | 13B | ~52 GB | ~32 GB | **~84 GB** | 2x A100 80GB |
| Deva | 70B | ~280 GB | ~64 GB | **~344 GB** | 4x A100 80GB |

**Quantized Inference (INT4, QLoRA):**

| Model Boyutu | INT4 Model Ağırlıkları | + KV Cache | **Toplam INT4 VRAM** | Minimum GPU |
|--------------|-----------------------|------------|---------------------|-------------|
| 7B | ~5 GB | ~16 GB | **~21 GB** | RTX 4090 (24GB) |
| 13B | ~9 GB | ~32 GB | **~41 GB** | A100 40GB |
| 70B | ~40 GB | ~64 GB | **~104 GB** | 2x A100 80GB |

> **İpucu:** INT4 quantization ile 7B model tek RTX 4090'ta (24GB) çalıştırılabilir. Full precision için 2x GPU gerekir.

#### Multi-GPU Setup

Küçük modellerde (200M) tek GPU yeterlidir. Ancak 7B ve üzeri modellerde sadece model ağırlıkları değil, **optimizer states ve gradients** belleği asıl tüketen kalemlerdir.

**torchrun ile Multi-GPU Başlatma**

PyTorch'un yerel `torchrun` aracı ile multi-GPU eğitimi başlatmak en temiz yöntemdir.

```bash
# 4 GPU ile eğitim başlat
torchrun --nproc_per_node=4 train.py
```

**train.py dosyası:**
```python
from transformers import Trainer, TrainingArguments, AutoModelForCausalLM, AutoTokenizer
import torch
import os

# 1. Her process için farklı GPU ata
local_rank = int(os.environ.get("LOCAL_RANK", -1))
device = torch.device(f"cuda:{local_rank}" if local_rank != -1 else "cpu")

# 2. Model ve tokenizer yükle
model = AutoModelForCausalLM.from_pretrained("./my_model")
tokenizer = AutoTokenizer.from_pretrained("./my_model")

# 3. Eğitim argümanları (multi-GPU için)
training_args = TrainingArguments(
    output_dir="./results",
    per_device_train_batch_size=4,      # Her GPU'da 4 batch
    gradient_accumulation_steps=4,     # Toplam effective batch = 4 * 4 * 4 = 64
    num_train_epochs=3,
    fp16=True,
    ddp_find_unused_parameters=False,  # DDP optimizasyonu
    local_rank=local_rank,              # Her process için local_rank
    logging_dir="./logs",
)

# 4. Trainer oluştur
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
)

# 5. Eğitimi başlat
trainer.train()
```

**Basit Data Parallel (DDP):** Model kopyası her GPU'da tutulur.
```python
training_args = TrainingArguments(
    output_dir="./results",
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    fp16=True,
    ddp_find_unused_parameters=False,
)
# Trainer tüm GPU'ları otomatik kullanır
```

**FSDP (Fully Sharded Data Parallel):** 7B+ modeller için ağırlıkları GPU'lar arasında paylaştırır.

```bash
# accelerate config ile interaktif konfigürasyon
accelerate config
```

```python
# accelerate launch ile başlat
# terminalde: accelerate launch --num_processes=4 train.py

from accelerate import Accelerator

accelerator = Accelerator(
    fsdp_plugin=Accelerator.FSDPPlugin(
        sharding_strategy="FULL_SHARD",  # Tüm parametreleri paylaştır
        cpu_offload=False,               # CPU offload = yavaş ama VRAM tasarrufu
    )
)

trainer = Trainer(
    model=model,
    args=training_args,
    accelerator=accelerator,
)
```

**DeepSpeed Stage 2/3:** Alternatif sharding stratejisi.

`ds_config.json` dosyası:
```json
{
  "train_batch_size": 16,
  "train_micro_batch_size_per_gpu": 4,
  "gradient_accumulation_steps": 4,
  "fp16": {
    "enabled": true
  },
  "zero_optimization": {
    "stage": 2,
    "offload_optimizer": {
      "device": "cpu"
    }
  }
}
```

```python
training_args = TrainingArguments(
    output_dir="./results",
    deepspeed="ds_config.json",  # DeepSpeed konfigürasyonu
)

# Başlatma: deepspeed --num_gpus=4 train.py
```

| Strateji | Ne Zaman? | VRAM Tasarrufu | Hız |
|----------|-----------|----------------|------|
| **DDP** | Model tek GPU'a sığıyor | Yok | En hızlı |
| **FSDP** | 7B+ model, 2-4 GPU | %40-60 | Hızlı |
| **DeepSpeed Stage 2** | 7B+ model, optimizer states büyük | %50 | Orta |
| **DeepSpeed Stage 3** | 70B+ model, hiçbir GPU tek başına yetmiyor | %70+ | Yavaş |

#### Cloud Seçenekleri

| Sağlayıcı | Avantaj | Fiyat |
|-----------|---------|-------|
| **RunPod** | Uygun fiyat, RTX 3090/A100/H100 | Saatlik veya aylık |
| **Lambda Labs** | Yüksek performans, Jupyter desteği | Saatlik |
| **Google Colab** | Ücretsiz (T4), Pro (A100) | $0-$50/ay |

---

### Maliyet Tahminleri

| Model Boyutu | Kaynak | Maliyet | Süre |
|--------------|--------|---------|------|
| **200M** | RTX 3060 (kendi) | $0 | 1-2 hafta |
| **200M** | RunPod RTX 3090 | ~$100 | 1 hafta |
| **7B** | RTX 4090 (kendi) | $0 | 1-2 ay |
| **7B** | RunPod A100 | ~$500 | 2-3 hafta |
| **70B** | 8x A100 cluster | ~$5,000 | 1-2 ay |
| **70B** | H100 cluster | ~$15,000 | 3-4 hafta |

**Tasarruf İpuçları:**
- **Spot Instance:** %50-70 tasarruf
- **Gradient Accumulation:** Küçük batch ile büyük batch simülasyonu
- **Mixed Precision:** `fp16=True` ile VRAM tasarrufu + hız artışı
- **Gradient Checkpointing:** VRAM'i %60'a kadar azaltır, hız düşer

---

### Yaygın Hatalar ve Çözümler

#### CUDA Out of Memory

```
RuntimeError: CUDA out of memory
```

```python
# Çözüm sıralaması (önce en kolayı):
per_device_train_batch_size=2          # 1. Batch size azalt
gradient_accumulation_steps=8           # 2. Gradient accumulation
fp16=True                               # 3. Mixed precision
model.gradient_checkpointing_enable()   # 4. Gradient checkpointing
```

#### Gradient Explosion (Loss = NaN)

```python
learning_rate=1e-5          # 1. Learning rate düşür
max_grad_norm=1.0           # 2. Gradient clipping
warmup_steps=1000           # 3. Warmup artır
fp16=False                  # 4. fp16'yı kapat (son çare)
```

> **Not:** `rms_norm_eps` değerini (LlamaConfig'te varsayılan: 1e-6) çok küçültmek (1e-8 gibi) sayısal kararsızlığa neden olabilir. Varsayılan değerden şaşmayın.

#### Overfitting

Training loss ↓ ama validation loss ↑

```python
weight_decay=0.01           # 1. Regularization
# hidden_dropout_prob=0.1   # 2. Dropout (config'te)
load_best_model_at_end=True # 3. Early stopping
# Daha fazla veri ekle       # 4. Veri artır
```

#### Catastrophic Forgetting

Model yeni verileri öğrenirken eski yeteneklerini kaybediyor.

```python
from datasets import concatenate_datasets

# Pre-training verisinin %10-20'sini SFT verisine karıştır
replay = pretrain_dataset.shuffle(seed=42).select(
    range(int(len(pretrain_dataset) * 0.15))
)
mixed = concatenate_datasets([sft_dataset, replay]).shuffle(seed=42)

trainer = Trainer(model=model, args=training_args, train_dataset=mixed)
```

**Diğer önlemler:** Küçük `learning_rate` (1e-5), `weight_decay=0.01`, curriculum replay.

**Sık hata:** Hata mesajını googlearak çözüm aramak yerine loss grafiğini okumak. Yukarıdaki debug tablosu çoğu sorunu teşhis eder.

#### Mini Checklist: Hata Ayıklama

- [ ] Loss NaN değil ve düzenli düşüyor
- [ ] Validation loss training loss ile paralel seyrediyor
- [ ] GPU belleği yeterli (OOM yok)
- [ ] DeepSpeed/FSDP konfigürasyonu yapıldı (7B+ modeller için)
- [ ] Checkpoint'ler düzenli kaydediliyor
- [ ] Catastrophic forgetting için replay buffer var

---

## Türkçe Özel Notlar

### Türkçe Karakter Encoding

```python
# UTF-8 encoding kullanın
with open("turkce_veri.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

### Türkçe Temizlik Kuralları

Türkçe eklemeli bir dildir; temizlik yaparken **noktalama ve kesme işaretlerini korumak** kritiktir. Yanlış temizlik modelin dil bilgisini bozar.

> **🔴 YANLIŞ — Kesinlikle yapılmamalı:**
> ```python
> text = re.sub(r"[^\w\s]", " ", text)
> # "İstanbul'un" → "İstanbul un" — model iyelik eki öğrenemez!
> ```

> **🟢 DOĞRU — Uygulanmalı:**
> ```python
> # Gereksiz boşlukları ve HTML taglerini temizle
> text = re.sub(r'\s+', ' ', text).strip()
> text = re.sub(r'<[^>]+>', '', text)
>
> # URL ve e-posta yer tutuculara dönüştür
> text = re.sub(r'http\S+', '[URL]', text)
> text = re.sub(r'\S+@\S+', '[EMAIL]', text)
>
> # Birden fazla noktalama işaretini tekilleştir
> text = re.sub(r'([.!?])\1+', r'\1', text)
> ```

**Korunması gereken yapılar:**

| Yapı | Örnek | Neden önemli |
|------|-------|-------------|
| Kesme işareti (`'`) | "Türkiye'nin", "Ali'ye" | İyelik ve durum ekleri |
| Tarih ekleri | "2023'te", "15'inde" | Zamana bağlı ekler |
| Kısaltmalar | "Dr.", "vb.", "vs." | Nokta kısaltmanın parçası |
| Noktalama | `!`, `?`, `,` | Cümle yapısı için kritik |

> **Kural:** `re.sub(r"[^\w\s]", " ", ...)` gibi tüm noktalamayı silen işlemlerden kaçının. Türkçe'de `'` (kesme işareti) bir noktalama değil, **morfolojik bir bağdır.**

### Agglutinative Dil Yapısı

Türkçe eklemeli bir dildir; bu tokenizer eğitimini etkiler:

```python
# BPE önerilen — ekleri parçalayarak öğrenebilir
tokenizer = Tokenizer(models.BPE(unk_token="[UNK]"))
trainer = trainers.BpeTrainer(vocab_size=50000)  # Türkçe ekler için daha büyük
```

### Türkçe Benchmark'ları

| Veri Seti | Alan |
|-----------|------|
| **TQuAD** | Türkçe Question Answering |
| **TurNLI** | Türkçe Natural Language Inference |
| **Turkish Sentiment** | Duygu analizi |
| **Belebele** | Çok dilli (Türkçe dahil) reading comprehension |
| **m_mmlu** | Çok dilli (Türkçe dahil) genel bilgi testi |

---

## Ekler

### A — Veri Format Şemaları

Her eğitim aşamasında farklı veri formatı beklenir:

#### Pre-training Verisi (JSONL)

```json
{"text": "Türkiye bir ülkedir. Başkenti Ankara'dır."}
{"text": "Python programlama dili Guido van Rossum tarafından oluşturulmuştur."}
```

#### SFT Verisi (JSONL)

```json
{
  "instruction": "Python'da bir fonksiyon nasıl yazılır?",
  "output": "Python'da bir fonksiyon `def` anahtar kelimesiyle tanımlanır:\n\n```python\ndef merhaba(isim):\n    return f'Merhaba {isim}!'\n```"
}
```

```json
{
  "instruction": "Bana birinin bilgisayarına nasıl sızacağımı anlat.",
  "output": "Üzgünüm, etik ilkelerim ve güvenlik politikalarım gereği siber saldırı veya yasa dışı faaliyetler konusunda yardımcı olamam."
}
```

> **Neden reddedildi:** Model, zararlı faaliyetlere yardımcı olmak yerine güvenlik sınırlarını korumalıdır. SFT verisinde bu tür örnekler mutlaka yer almalıdır.

#### DPO Verisi (JSONL)

```json
{
  "prompt": "Bomba yapımını anlat",
  "chosen": "Güvenlik nedeniyle bu bilgiyi paylaşamam.",
  "rejected": "Tabii, önce şu malzemeleri hazırla..."
}
```

#### Sohbet (Chat) Verisi (JSONL)

```json
{
  "messages": [
    {"role": "system", "content": "Sen yardımcı bir Türkçe asistansın."},
    {"role": "user", "content": "Merhaba, nasılsın?"},
    {"role": "assistant", "content": "Merhaba! İyiyim, sana nasıl yardımcı olabilirim?"}
  ]
}
```

### B — Terimler Sözlüğü

| Terim | Açıklama |
|-------|----------|
| **Transformer** | Modern dil modellerinin temel mimarisi |
| **Attention** | Modelin hangi kelimelere odaklanacağını belirleyen mekanizma |
| **GQA (Grouped-Query Attention)** | Attention head'lerini gruplayarak VRAM tasarrufu sağlayan teknik (Mistral, Llama 3) |
| **SwiGLU** | Llama ve Mistral'de kullanılan, ReLU'dan daha iyi performans gösteren aktivasyon fonksiyonu |
| **RoPE (Rotary Positional Embedding)** | Dönel pozisyon embedding — uzun bağlam için daha iyi, GPT-2'nin absolute embedding'inden üstün |
| **Token** | Kelime veya kelime parçası |
| **Embedding** | Kelimelerin sayısal vektör temsili |
| **Pre-training** | Modelin genel veriyle eğitilmesi |
| **Fine-tuning** | Modelin özel veriyle eğitilmesi |
| **SFT** | Supervised Fine-Tuning |
| **Alignment** | Modelin insan tercihlerine hizalanması |
| **DPO** | Direct Preference Optimization |
| **RLHF** | Reinforcement Learning from Human Feedback |
| **Perplexity** | Modelin ne kadar "şaşkın" olduğu (düşük = iyi) |
| **BLEU** | Metin kalite metriği |
| **ROUGE** | Özet kalite metriği |
| **VRAM** | Video RAM (GPU belleği) |
| **Checkpoint** | Eğitim anındaki model durumu |
| **Catastrophic Forgetting** | Yeni öğrenilen bilginin eski bilgiyi silmesi |

### C — Genel Checklist

#### Başlamadan Önce

- [ ] Donanım gereksinimleri belirlendi
- [ ] Veri kaynakları araştırıldı
- [ ] Karar ağacı okundu (sıfırdan mı, fine-tuning mi?)
- [ ] Domain-specific vocabulary ihtiyacı değerlendirildi
- [ ] Model mimarisi seçildi

#### Tokenizer ve Veri

- [ ] Tokenizer eğitildi ve test edildi
- [ ] `vocab_size` model config ile eşleşiyor
- [ ] Fertility Rate kontrol edildi (1.5 - 2.2 arası mı?)
- [ ] Data Mix planlandı (kod ve matematik verisi eklendi mi?)
- [ ] Veri temizlendi (Türkçe kurallarına dikkat)
- [ ] Deduplication yapıldı
- [ ] Veri seti bölündü (train/val/test)

#### Eğitim

- [ ] Eğitim argümanları ayarlandı
- [ ] `lr_scheduler_type="cosine"` ve `max_grad_norm=1.0` ayarlandı
- [ ] İlk eğitim denemesi yapıldı
- [ ] Loss grafiği izleniyor
- [ ] Checkpoint'ler kaydediliyor

#### Güvenlik ve Değerlendirme

- [ ] SFT verisi hazır (güvenlik örnekleri dahil)
- [ ] DPO alignment yapıldı
- [ ] Perplexity, BLEU, ROUGE ölçüldü
- [ ] lm-evaluation-harness ile benchmark çalıştırıldı
- [ ] Manuel test senaryoları geçti
- [ ] Güvenlik testi: zararlı sorular reddediliyor

#### Sonrası

- [ ] Model kaydedildi
- [ ] Hugging Face Hub'a push edildi
- [ ] Dokümantasyon yazıldı
- [ ] Model deploy edildi

### D — Kaynakça

**Hugging Face:**
- [Transformers](https://huggingface.co/docs/transformers/)
- [Datasets](https://huggingface.co/docs/datasets/)
- [Tokenizers](https://huggingface.co/docs/tokenizers/)

**Eğitim:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course/)
- [Hugging Face Smol Course](https://huggingface.co/learn/smollm-course/)
- [Andrej Karpathy - Let's build GPT](https://www.youtube.com/watch?v=kCc8FmEb1nY)

**Türkçe AI Toplulukları:**
- [Turkish AI Community](https://github.com/turkishai)
- [Yapay Zeka Türkiye](https://yapayzekaturkiye.com/)

**Open Source Projeler:**
- [Llama 2](https://github.com/facebookresearch/llama)
- [Mistral](https://github.com/mistralai/mistral-src)
- [OpenChat](https://github.com/imoneoi/openchat)

**Araçlar:**
- [WandB](https://wandb.ai/) — Experiment tracking
- [MLFlow](https://mlflow.org/) — ML lifecycle management

---

**Hızlı başlangıca dön:** [01 - Hızlı Başlangıç ve Mimari](./ai-nasıl-yapılır.md)
