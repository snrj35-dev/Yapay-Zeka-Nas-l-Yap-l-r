# 00 - Giriş

Bu seri, sıfırdan Türkçe dil modeli eğitmek isteyenler için uçtan uca bir rehberdir. Model mimarisinden başlayıp tokenizer eğitimi, veri hazırlama, eğitim, güvenlik, değerlendirme, deployment ve inference'a kadar tüm süreci adım adım anlatır.

---

## Serinin Amacı ve Kapsamı

Bu rehber serisi, **sıfırdan Türkçe dil modeli eğitmek** isteyenler için tasarlanmıştır. Mevcut İngilizce modelleri fine-tuning yapmak yerine, tamamen yeni bir model oluşturmayı hedefleyenler için pratik ve uygulanabilir bir yol haritası sunar.

**Kapsam:**
- Boş model oluşturma ve mimari seçimi
- Türkçe tokenizer eğitimi
- Veri toplama, temizleme ve hazırlama
- Sıfırdan pre-training, SFT ve alignment
- Model değerlendirme ve benchmark'lar
- Quantization, deployment ve inference
- Multi-GPU setup ve donanım optimizasyonu

**Kapsam dışı:**
- Mevcut modellerin fine-tuning'i (LoRA/QLoRA kısa açıklama var ama detaylı rehber değil)
- Üretim seviyesi MLOps pipeline'ları
- Ticari cloud platform entegrasyonları

---

## Hangi Dosyayı Ne Zaman Okumalısınız?

```
Başlangıç
│
├─ "AI modeli eğitmek istiyorum ama nereden başlayacağımı bilmiyorum"
│  └─ 👉 01 - Hızlı Başlangıç ve Mimari
│     - Karar ağacı: Sıfırdan eğitim mi, fine-tuning mi?
│     - Boş model oluşturma
│     - Mimari seçimi (Llama, GPT-2, vb.)
│     - Donanım gereksinimleri
│
├─ "Karar verdim, sıfırdan eğitim yapacağım. Nasıl?"
│  └─ 👉 02 - Uçtan Uca Eğitim Rehberi
│     - Tokenizer eğitimi
│     - Veri toplama ve temizleme
│     - Model eğitimi (pre-training, SFT, alignment)
│     - Değerlendirme ve hata ayıklama
│     - Multi-GPU setup
│
└─ "Model eğitildi, şimdi ne yapacağım?"
   └─ 👉 03 - Küçültme, Dağıtım ve Tahminleme
      - Quantization (INT4/INT8, GGUF)
      - Deployment seçenekleri (vLLM, Ollama, FastAPI)
      - Inference parametreleri
      - Benchmark ve monitoring
```

---

## Ön Koşullar

### Teknik Gereksinimler

| Konu | Minimum | Önerilen |
|------|---------|----------|
| **Python** | 3.8+ | 3.10+ |
| **PyTorch** | 2.0+ | 2.2+ |
| **GPU** | 12 GB VRAM (RTX 3060) | 24 GB VRAM (RTX 4090) |
| **RAM** | 16 GB | 32 GB |
| **Disk** | 50 GB | 200 GB+ |

### Gerekli Kütüphaneler

```bash
pip install transformers>=4.40 tokenizers>=0.19 datasets>=2.18 torch>=2.2 accelerate>=0.26 peft>=0.7 bitsandbytes>=0.43
```

### Bilgi Seviyesi

- **Python:** Temel-orta seviye (class, function, list comprehension)
- **PyTorch:** Temel (tensor, model, forward pass)
- **Machine Learning:** Temel kavramlar (loss, optimizer, gradient descent)
- **Hugging Face:** Başlangıç (AutoModel, AutoTokenizer)

---

## Bu Seriyle Ne Yapabilirsin, Ne Yapamazsın?

### ✅ Yapabilirsiniz

- 100M - 200M parametreli küçük Türkçe modeli sıfırdan eğitmek
- Türkçe tokenizer eğitmek ve entegre etmek
- Veri toplama, temizleme ve deduplication yapmak
- Modeli pre-training, SFT ve alignment aşamalarından geçirmek
- Modeli quantize edip deploy etmek
- Multi-GPU setup ile büyük modeller eğitmek

### ❌ Yapamazsınız (Bu Serinin Kapsamı Dışında)

- Mevcut büyük modelleri (Llama 3, Mistral vb.) fine-tuning yapmak (kısa açıklama var ama detaylı rehber değil)
- Üretim seviyesi MLOps pipeline'ları kurmak
- Ticari cloud platformlara (AWS, GCP, Azure) özel deployment
- Gerçek zamanlı, yüksek trafikli API servisleri için production setup

---

## Başlamadan Önce

1. **Donanım kontrolü:** GPU VRAM'iniz yeterli mi? [01 - Hızlı Başlangıç ve Mimari](./01 - Hızlı Başlangıç ve Mimari.md) dosyasındaki donanım tablosuna bakın.
2. **Veri hazırlığı:** Eğitim için Türkçe veri topladınız mı? [02 - Uçtan Uca Eğitim Rehberi](./02 - Uçtan Uca Eğitim Rehberi.md) dosyasındaki veri toplama bölümüne bakın.
3. **Zaman planlaması:** Sıfırdan eğitim günler-haftalar sürebilir. Küçük model (100M) ile başlamak önerilir.

---

## Seriye Başla

Hazırsanız, ilk dosyadan başlayın:

👉 **[01 - Hızlı Başlangıç ve Mimari](./01 - Hızlı Başlangıç ve Mimari.md)**
