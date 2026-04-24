# 03 - Küçültme, Dağıtım ve Tahminleme

Bu rehber, eğitilmiş modelini küçültmeyi (quantization), dağıtmayı (deployment) ve doğru parametrelerle tahminlemeyi (inference) anlatır. Eğitim bittikten sonra "peki şimdi ne yapacağım?" sorusunun cevabı burada.

**Hızlı başlangıç ve mimari için:** [01 - Hızlı Başlangıç ve Mimari](./01 - Hızlı Başlangıç ve Mimari.md)  
**Uçtan uca eğitim için:** [02 - Uçtan Uca Eğitim Rehberi](./02 - Uçtan Uca Eğitim Rehberi.md)

---

## İçindekiler

1. [Model Boyutu ve VRAM Referansı](#model-boyutu-ve-vram-referansı)
2. [Quantization](#quantization)
3. [Model Format Dönüşümleri](#model-format-dönüşümleri)
4. [Inference ve Generation Parametreleri](#inference-ve-generation-parametreleri)
5. [Multi-GPU ve Dağıtık Çalıştırma](#multi-gpu-ve-dağıtık-çalıştırma)
6. [Deployment Seçenekleri](#deployment-seçenekleri)
7. [Hugging Face Hub'a Yükleme](#hugging-face-huba-yükleme)
8. [Hata Ayıklama](#hata-ayıklama)
9. [Ekler](#ekler)

---

## Model Boyutu ve VRAM Referansı

Modelini deploy etmeden önce kaç GB VRAM gerekeceğini bilmen şart. Kural basit: her parametre float32'de 4 byte, float16'da 2 byte, int8'de 1 byte yer kaplar.

| Model Boyutu | float32 (tam) | float16 / bfloat16 | int8 | int4 |
|-------------|---------------|-------------------|------|------|
| 200M | ~0.8 GB | ~0.4 GB | ~0.2 GB | ~0.1 GB |
| 1B | ~4 GB | ~2 GB | ~1 GB | ~0.5 GB |
| 7B | ~28 GB | ~14 GB | ~7 GB | ~3.5 GB |
| 13B | ~52 GB | ~26 GB | ~13 GB | ~6.5 GB |
| 70B | ~280 GB | ~140 GB | ~70 GB | ~35 GB |

> **Kural:** Inference için VRAM = model ağırlıkları + ~%20 ek bellek (KV cache, aktivasyonlar). Eğitim için bu rakamı 3-4x ile çarp.

### Hangi GPU Ne İşe Yarar?

| GPU | VRAM | Çalıştırabildiği Model |
|-----|------|------------------------|
| RTX 3060 / 4060 | 12 GB | 7B int4, 3B int8 |
| RTX 3090 / 4090 | 24 GB | 13B int4-int8, 7B float16 |
| A100 40GB | 40 GB | 13B float16, 30B int4 |
| A100 80GB | 80 GB | 70B int4, 30B float16 |
| 2x A100 80GB | 160 GB | 70B float16 |

---

## Quantization

**Amaç:** Modelin ağırlıklarını daha düşük hassasiyete (precision) indirerek VRAM kullanımını ve çalışma süresini azaltmak. Doğruluk kaybı genellikle minimaldur.

```
float32 → float16 → bfloat16 → int8 → int4
(büyük, hassas)                       (küçük, hızlı)
```

### Yöntem 1: bitsandbytes ile INT8 / INT4

En kolay yol. Hugging Face ile doğrudan entegre çalışır.

```bash
pip install bitsandbytes accelerate
```

#### INT8 Quantization

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig

quantization_config = BitsAndBytesConfig(load_in_8bit=True)

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    quantization_config=quantization_config,
    device_map="auto"  # GPU'ya otomatik yerleştir
)

tokenizer = AutoTokenizer.from_pretrained("./modelim")
```

#### INT4 Quantization (QLoRA tarzı)

Daha agresif sıkıştırma. 7B modeli yaklaşık 3.5 GB'a indirir.

```python
quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.bfloat16,  # Hesaplama hassasiyeti
    bnb_4bit_use_double_quant=True,          # Ek tasarruf (~%0.4)
    bnb_4bit_quant_type="nf4"               # NormalFloat4 (önerilen)
)

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    quantization_config=quantization_config,
    device_map="auto"
)
```

> **İpucu:** `bnb_4bit_quant_type="nf4"` (NormalFloat4), standart int4'e göre daha iyi doğruluk verir. Hep bunu kullanın.

### Yöntem 2: GPTQ (GPU üzerinde statik quantization)

Modeli bir kez offline quantize et, sonra hızlıca yükle. `bitsandbytes`'tan daha hızlı inference verir.

```bash
pip install auto-gptq optimum
```

```python
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

quantize_config = BaseQuantizeConfig(
    bits=4,           # 4-bit quantization
    group_size=128,   # Her 128 ağırlık için bir scale faktörü
    desc_act=False
)

model = AutoGPTQForCausalLM.from_pretrained("./modelim", quantize_config)

# Kalibrasyon verisi (birkaç örnek metin yeterli)
examples = [
    tokenizer("Türkçe bir cümle örneği.", return_tensors="pt")
]

model.quantize(examples)
model.save_quantized("./modelim-gptq-4bit")
```

Sonraki seferlerde doğrudan yükle:

```python
model = AutoGPTQForCausalLM.from_quantized(
    "./modelim-gptq-4bit",
    device="cuda:0"
)
```

### Yöntem Karşılaştırması

| Yöntem | Hız | VRAM | Kurulum | Ne zaman? |
|--------|-----|------|---------|-----------|
| **float16** | Hızlı | Orta | Kolay | VRAM yeterliyse |
| **INT8 (bitsandbytes)** | Orta | Düşük | Kolay | Hızlı deneme |
| **INT4 (bitsandbytes)** | Orta | Çok düşük | Kolay | VRAM kısıtlıysa |
| **GPTQ 4-bit** | Hızlı | Çok düşük | Orta | Production inference |
| **GGUF (CPU+GPU)** | Değişken | Çok düşük | Kolay | Yerel kullanım |

---

## Model Format Dönüşümleri

### Hugging Face → GGUF (llama.cpp için)

GGUF formatı, CPU üzerinde veya CPU+GPU hibrid modda çalıştırmanı sağlar. Yerel kullanım için idealdir.

```bash
# llama.cpp'yi klonla ve derle
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make

# Python bağımlılıkları
pip install -r requirements.txt

# Dönüştür
python convert_hf_to_gguf.py ./modelim --outfile modelim.gguf --outtype f16
```

#### GGUF Quantization (isteğe bağlı)

```bash
# Q4_K_M: kalite/boyut dengesi için önerilen
./quantize modelim.gguf modelim-q4_k_m.gguf Q4_K_M

# Q8_0: yüksek kalite, daha büyük
./quantize modelim.gguf modelim-q8.gguf Q8_0
```

| GGUF Tipi | Boyut Azalması | Kalite | Önerilen Durum |
|-----------|---------------|--------|----------------|
| F16 | %50 | Mükemmel | VRAM yeterliyse |
| Q8_0 | %75 | Çok iyi | Kaliteden ödün vermemek |
| Q4_K_M | %87 | İyi | Genel kullanım (önerilen) |
| Q4_0 | %87 | Orta | Hız öncelikliyse |
| Q2_K | %93 | Zayıf | Sadece son çare |

#### llama.cpp ile Çalıştırma

```bash
./llama-cli -m modelim-q4_k_m.gguf -p "Merhaba, nasılsın?" -n 200
```

### Hugging Face → ONNX

ONNX, farklı framework ve donanımlarda çalışmayı sağlar (edge cihazlar, Windows ML, vb.).

```bash
pip install optimum[onnxruntime]
```

```python
from optimum.onnxruntime import ORTModelForCausalLM

model = ORTModelForCausalLM.from_pretrained(
    "./modelim",
    export=True  # Otomatik dönüştür
)
model.save_pretrained("./modelim-onnx")
```

---

## Inference ve Generation Parametreleri

Model eğitildi, quantize edildi — peki nasıl konuşturulur? Bu bölüm en çok atlanan konulardan biridir.

### Temel Inference

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    torch_dtype=torch.float16,
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained("./modelim")

inputs = tokenizer("Türkiye'nin başkenti", return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=100,
        temperature=0.7,
        top_p=0.9,
        do_sample=True
    )

print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

### Generation Parametreleri — Tam Rehber

#### `max_new_tokens` ve `max_length`

```python
# Kaç yeni token üretilsin?
max_new_tokens=200   # Girdi uzunluğundan bağımsız, sadece üretilecek kısım
max_length=512       # Toplam uzunluk (girdi + çıktı) — dikkatli kullan
```

> **Uyarı:** `max_length` yerine `max_new_tokens` kullanmak genellikle daha güvenlidir. `max_length` girdi uzunsa hiç üretmeyebilir.

#### `temperature` — Yaratıcılık Kontrolü

Modelin çıktı dağılımını ne kadar "düzleştireceği" veya "sivrileştireceği":

```python
temperature=0.1   # Çok deterministik, tekrarcı ama tutarlı
temperature=0.7   # Dengeli (önerilen başlangıç)
temperature=1.0   # Modelin orijinal dağılımı
temperature=1.5   # Yaratıcı ama saçmalayabilir
```

| Temperature | Kullanım Alanı |
|-------------|----------------|
| 0.1 - 0.3 | Kod üretimi, faktüel sorular |
| 0.5 - 0.8 | Genel sohbet, özetleme |
| 0.8 - 1.2 | Yaratıcı yazarlık, hikaye |
| > 1.2 | Deneysel; çıktı bozulabilir |

#### `top_p` (Nucleus Sampling)

Kümülatif olasılığı `p`'yi geçen token'ları filtreler:

```python
top_p=0.9   # Olasılığı en yüksek tokenların %90'lık dilimine bak
top_p=0.95  # Daha geniş seçim havuzu
top_p=1.0   # Filtreleme yok
```

#### `top_k` — Token Havuzunu Sınırla

```python
top_k=50    # Sadece en olası 50 token'dan seç
top_k=0     # Devre dışı (top_p ile birlikte kullanılıyorsa önerilir)
```

> **İpucu:** `top_p` ve `top_k` birlikte kullanılabilir. Genellikle `top_p=0.9, top_k=50` iyi bir başlangıç noktasıdır.

#### `repetition_penalty` — Tekrar Önleme

Model aynı kelime veya ifadeyi tekrarlamaya başlarsa:

```python
repetition_penalty=1.0   # Ceza yok (varsayılan)
repetition_penalty=1.1   # Hafif ceza (önerilen)
repetition_penalty=1.3   # Güçlü ceza
repetition_penalty=1.5   # Çok agresif; anlamsız kaçınmalar başlayabilir
```

#### `do_sample` — Örnekleme Açma/Kapama

```python
do_sample=False   # Greedy decoding (her adımda en olası token)
do_sample=True    # Sampling (temperature, top_p, top_k etkin)
```

> **Kural:** `do_sample=False` ise `temperature`, `top_p`, `top_k` parametreleri **devre dışı** kalır.

#### Önerilen Preset'ler

```python
# Kod / Faktüel sorular — tutarlılık öncelikli
code_params = dict(
    max_new_tokens=512,
    temperature=0.2,
    top_p=0.95,
    repetition_penalty=1.1,
    do_sample=True
)

# Genel sohbet — denge
chat_params = dict(
    max_new_tokens=256,
    temperature=0.7,
    top_p=0.9,
    top_k=50,
    repetition_penalty=1.1,
    do_sample=True
)

# Yaratıcı yazarlık — çeşitlilik
creative_params = dict(
    max_new_tokens=512,
    temperature=1.0,
    top_p=0.92,
    repetition_penalty=1.05,
    do_sample=True
)
```

### Chat Template ile Inference

SFT sonrası eğittiğin model için doğru format kritiktir:

```python
messages = [
    {"role": "system", "content": "Sen yardımcı bir Türkçe asistansın."},
    {"role": "user", "content": "Python'da liste nasıl oluşturulur?"}
]

# Chat template uygula
input_text = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True
)

inputs = tokenizer(input_text, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model.generate(**inputs, **chat_params)

# Sadece yeni üretilen kısmı al
new_tokens = outputs[0][inputs["input_ids"].shape[1]:]
response = tokenizer.decode(new_tokens, skip_special_tokens=True)
print(response)
```

### Streaming (Token Token Yazdırma)

```python
from transformers import TextStreamer

streamer = TextStreamer(tokenizer, skip_prompt=True, skip_special_tokens=True)

model.generate(
    **inputs,
    max_new_tokens=200,
    temperature=0.7,
    do_sample=True,
    streamer=streamer   # Her token üretilince ekrana yaz
)
```

### Batch Inference (Toplu Üretim)

```python
# Padding token ayarla (batch için zorunlu)
tokenizer.pad_token = tokenizer.eos_token
tokenizer.padding_side = "left"  # Decoder-only modeller için sol padding

prompts = [
    "Türkiye'nin başkenti",
    "Python programlama dili",
    "Yapay zeka nedir?"
]

inputs = tokenizer(
    prompts,
    return_tensors="pt",
    padding=True,
    truncation=True,
    max_length=512
).to(model.device)

with torch.no_grad():
    outputs = model.generate(**inputs, max_new_tokens=100, do_sample=False)

for output in outputs:
    print(tokenizer.decode(output, skip_special_tokens=True))
    print("---")
```

---

## Multi-GPU ve Dağıtık Çalıştırma

### `device_map="auto"` ile Otomatik Dağıtım

En kolay yol. Model katmanlarını mevcut GPU'lara otomatik dağıtır:

```python
from transformers import AutoModelForCausalLM
import torch

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    torch_dtype=torch.float16,
    device_map="auto"   # Tüm GPU'lara otomatik dağıt
)
```

Nasıl dağıtıldığını görmek için:

```python
print(model.hf_device_map)
# Çıktı: {'model.embed_tokens': 0, 'model.layers.0': 0, ..., 'model.layers.31': 1, 'lm_head': 1}
```

### Manuel `device_map`

Belirli katmanları belirli GPU'lara atamak istersen:

```python
device_map = {
    "model.embed_tokens": 0,
    "model.layers.0": 0,
    # ... ilk yarı GPU 0'da
    "model.layers.16": 1,
    # ... ikinci yarı GPU 1'de
    "lm_head": 1
}

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    torch_dtype=torch.float16,
    device_map=device_map
)
```

### torchrun ile Çok İşlemli Inference

```bash
# 2 GPU ile çalıştır
torchrun --nproc_per_node=2 inference_script.py
```

```python
# inference_script.py
import torch
import torch.distributed as dist
from transformers import AutoModelForCausalLM, AutoTokenizer

dist.init_process_group(backend="nccl")
local_rank = dist.get_rank()
torch.cuda.set_device(local_rank)

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    torch_dtype=torch.float16,
    device_map={"": local_rank}
)
```

### CPU Offloading (VRAM Yetmiyorsa)

```python
model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    torch_dtype=torch.float16,
    device_map="auto",
    offload_folder="./offload",  # Disk offloading için klasör
    offload_state_dict=True
)
```

> **Uyarı:** CPU/disk offloading inference'ı 5-20x yavaşlatır. Son çare olarak kullan.

---

## Deployment Seçenekleri

### Seçenek 1: vLLM (Production Önerilen)

Yüksek throughput, continuous batching desteğiyle production için en iyi seçenek.

```bash
pip install vllm
```

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="./modelim",
    dtype="float16",
    max_model_len=2048
)

sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=200
)

prompts = ["Merhaba, nasılsın?", "Python nedir?"]
outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    print(output.outputs[0].text)
```

#### vLLM ile OpenAI Uyumlu API Sunucusu

```bash
python -m vllm.entrypoints.openai.api_server \
    --model ./modelim \
    --host 0.0.0.0 \
    --port 8000 \
    --dtype float16
```

Sonra herhangi bir OpenAI client ile kullanabilirsin:

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")

response = client.chat.completions.create(
    model="modelim",
    messages=[{"role": "user", "content": "Merhaba!"}]
)
print(response.choices[0].message.content)
```

### Seçenek 2: Ollama (Yerel Kullanım)

Yerel bilgisayarda kolayca çalıştırmak için. GGUF formatını destekler.

```bash
# Ollama kur
curl -fsSL https://ollama.ai/install.sh | sh

# Modelfile oluştur
cat > Modelfile << 'EOF'
FROM ./modelim-q4_k_m.gguf
SYSTEM "Sen yardımcı bir Türkçe asistansın."
PARAMETER temperature 0.7
PARAMETER top_p 0.9
EOF

# Model oluştur ve çalıştır
ollama create benim-modelim -f Modelfile
ollama run benim-modelim
```

### Seçenek 3: Hugging Face Inference Endpoints

Kendi sunucunu yönetmek istemiyorsan:

1. Modeli Hub'a yükle (aşağıdaki bölüme bak)
2. [huggingface.co/inference-endpoints](https://huggingface.co/inference-endpoints) adresinden endpoint oluştur
3. Doğrudan API üzerinden kullan

```python
import requests

API_URL = "https://api-inference.huggingface.co/models/kullanici/modelim"
headers = {"Authorization": "Bearer hf_xxxxx"}

response = requests.post(
    API_URL,
    headers=headers,
    json={"inputs": "Merhaba dünya!"}
)
print(response.json())
```

### Seçenek 4: FastAPI ile Özel API

```bash
pip install fastapi uvicorn
```

```python
# api.py
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch
from collections import defaultdict
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI()

model = AutoModelForCausalLM.from_pretrained(
    "./modelim",
    torch_dtype=torch.float16,
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained("./modelim")

# Basit rate limiter
request_counts = defaultdict(list)

def rate_limit(ip: str, max_requests: int = 10, window: int = 60):
    now = time.time()
    request_counts[ip] = [t for t in request_counts[ip] if now - t < window]
    if len(request_counts[ip]) >= max_requests:
        raise HTTPException(status_code=429, detail="Rate limit aşıldı")
    request_counts[ip].append(now)

class GenerateRequest(BaseModel):
    prompt: str
    max_new_tokens: int = 200
    temperature: float = 0.7

@app.post("/generate")
def generate(request: GenerateRequest, http_request: Request):
    start = time.time()

    # Rate limiting
    client_ip = http_request.client.host
    rate_limit(client_ip)

    inputs = tokenizer(request.prompt, return_tensors="pt").to(model.device)

    with torch.no_grad():
        outputs = model.generate(
            **inputs,
            max_new_tokens=request.max_new_tokens,
            temperature=request.temperature,
            do_sample=True
        )

    response = tokenizer.decode(outputs[0], skip_special_tokens=True)

    # Monitoring
    elapsed = time.time() - start
    new_tokens = outputs.shape[1] - inputs["input_ids"].shape[1]
    logger.info(f"tokens={new_tokens} latency={elapsed:.2f}s ip={client_ip}")

    return {"response": response}
```

```bash
uvicorn api:app --host 0.0.0.0 --port 8000 --workers 1
```

### Deployment Karşılaştırması

| Seçenek | Throughput | Kurulum Zorluğu | Maliyet | Ne Zaman? |
|---------|-----------|----------------|---------|-----------|
| **vLLM** | Çok yüksek | Orta | Sunucu gerekli | Production, yüksek trafik |
| **Ollama** | Orta | Çok kolay | Yerel PC | Kişisel kullanım, demo |
| **HF Endpoints** | Yüksek | Çok kolay | Ücretli | Hızlı production |
| **FastAPI** | Düşük | Kolay | Sunucu gerekli | Küçük projeler, özelleştirme |
| **llama.cpp** | Orta | Kolay | Yerel PC | CPU ile çalıştırma |

### Benchmark / Hız Testi

Model deploy edildi, peki ne kadar hızlı? İşte farklı yöntemlerin karşılaştırması (7B model, A100 GPU):

| Yöntem | Token/sn (7B, A100) | Gecikme (ilk token) | Notlar |
|--------|-------------------|-------------------|--------|
| **float16 (HF)** | ~40 | ~200ms | Basit, standart |
| **vLLM** | ~150 | ~50ms | Continuous batching ile en hızlı |
| **GPTQ + vLLM** | ~180 | ~45ms | Quantized + vLLM kombinasyonu |
| **llama.cpp Q4 (CPU)** | ~10 | ~500ms | CPU ile çalıştırma, yavaş |

> **Not:** Bu değerler yaklaşık ve donanıma bağlıdır. Kendi ortamınızda test etmek için:
```python
import time
start = time.time()
outputs = model.generate(**inputs, max_new_tokens=100)
elapsed = time.time() - start
tokens = outputs.shape[1] - inputs["input_ids"].shape[1]
print(f"Token/sn: {tokens / elapsed:.1f}")
```

---

## Hugging Face Hub'a Yükleme

### Hazırlık

```bash
pip install huggingface_hub
huggingface-cli login   # Token gir
```

### Model Yükleme

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("./modelim")
tokenizer = AutoTokenizer.from_pretrained("./modelim")

model.push_to_hub("kullanici-adi/benim-modelim", private=False)
tokenizer.push_to_hub("kullanici-adi/benim-modelim")
```

### Model Kartı (README.md) Yazma

Hub'a yüklediğinde model kartı olmadan kimse ne işe yaradığını anlayamaz. Minimum içerik:

```markdown
---
language:
- tr
license: mit
tags:
- causal-lm
- turkish
- text-generation
---

# Benim Modelim

Sıfırdan eğitilmiş Türkçe dil modeli.

## Model Detayları

- **Mimari:** Llama
- **Parametre sayısı:** ~200M
- **Eğitim verisi:** Türkçe web, kitap, kod (~5GB)
- **Bağlam penceresi:** 2048 token

## Kullanım

from transformers import AutoModelForCausalLM, AutoTokenizer
model = AutoModelForCausalLM.from_pretrained("kullanici-adi/benim-modelim")
tokenizer = AutoTokenizer.from_pretrained("kullanici-adi/benim-modelim")

## Sınırlılıklar

- Model hatalı bilgi üretebilir
- Zararlı içerik filtrelemesi sınırlıdır
- Türkçe dışındaki dillerde performans düşüktür
```

### GGUF Dosyası Yükleme

```python
from huggingface_hub import HfApi

api = HfApi()
api.upload_file(
    path_or_fileobj="./modelim-q4_k_m.gguf",
    path_in_repo="modelim-q4_k_m.gguf",
    repo_id="kullanici-adi/benim-modelim-gguf",
    repo_type="model"
)
```

---

## Hata Ayıklama

### Sık Karşılaşılan Hatalar

| Hata | Neden | Çözüm |
|------|-------|-------|
| `CUDA out of memory` | Model VRAM'e sığmıyor | Quantization uygula veya `device_map="auto"` kullan |
| `RuntimeError: Expected all tensors on same device` | Girdi ve model farklı cihazda | `.to(model.device)` ekle |
| Sonsuz tekrar üretimi | `repetition_penalty` eksik | `repetition_penalty=1.1` ekle |
| Boş veya çok kısa çıktı | `max_new_tokens` çok düşük | `max_new_tokens` artır, chat template kontrol et |
| `KeyError: chat_template` | Tokenizer'da template yok | Tokenizer'a template ekle (02. rehbere bak) |
| Anlamsız çıktı | Temperature çok yüksek | Temperature'ı düşür (0.5-0.7) |
| `IndexError: index out of range in self` | `vocab_size` uyuşmazlığı | Model ve tokenizer vocab_size eşleşiyor mu kontrol et |

### Performans Profili Alma

```python
import torch
import time

# VRAM kullanımını izle
print(f"VRAM kullanımı: {torch.cuda.memory_allocated() / 1e9:.2f} GB")
print(f"VRAM rezervi:   {torch.cuda.memory_reserved() / 1e9:.2f} GB")

# Inference hızını ölç
start = time.time()
with torch.no_grad():
    outputs = model.generate(**inputs, max_new_tokens=100)
elapsed = time.time() - start

new_tokens = outputs.shape[1] - inputs["input_ids"].shape[1]
print(f"Üretilen token: {new_tokens}")
print(f"Süre:           {elapsed:.2f}s")
print(f"Token/saniye:   {new_tokens / elapsed:.1f}")
```

### Çıktı Kalitesi Sorunlarını Teşhis Etme

```
Çıktı kalitesi düşükse:
│
├─ Anlamsız kelime / karakter dizileri
│  └─ Tokenizer doğru yüklenmedi veya vocab_size uyuşmazlığı
│
├─ İlgisiz konulara kayıyor
│  └─ System prompt zayıf, temperature çok yüksek
│
├─ Çok kısa cevaplar, yarım cümleler
│  └─ max_new_tokens düşük veya EOS token yanlış ayarlanmış
│
├─ Aynı cümleyi tekrarlıyor
│  └─ repetition_penalty ekle (1.1 - 1.3 arası)
│
└─ Chat formatı bozuk (<|im_start|> görünüyor vb.)
   └─ skip_special_tokens=True ekle, chat template kontrol et
```

---

## Ekler

### A — Generation Parametre Cheatsheet

```python
# Kod / Mantık soruları
{"temperature": 0.2, "top_p": 0.95, "repetition_penalty": 1.1, "do_sample": True}

# Genel sohbet
{"temperature": 0.7, "top_p": 0.9, "top_k": 50, "repetition_penalty": 1.1, "do_sample": True}

# Yaratıcı yazı
{"temperature": 1.0, "top_p": 0.92, "repetition_penalty": 1.05, "do_sample": True}

# Deterministik (her seferinde aynı çıktı)
{"do_sample": False, "repetition_penalty": 1.1}
```

### B — Kütüphane Sürümleri

| Kütüphane | Önerilen Sürüm |
|-----------|---------------|
| `transformers` | ≥ 4.40 |
| `torch` | ≥ 2.2 |
| `accelerate` | ≥ 0.28 |
| `bitsandbytes` | ≥ 0.43 |
| `auto-gptq` | ≥ 0.7 |
| `vllm` | ≥ 0.4 |
| `fastapi` | ≥ 0.110 |

```bash
pip install transformers>=4.40 torch>=2.2 accelerate>=0.28 bitsandbytes>=0.43
```

### C — Genel Checklist

#### Quantization

- [ ] Model boyutu ve VRAM gereksinimi hesaplandı
- [ ] Quantization yöntemi seçildi (int8 / int4 / GPTQ)
- [ ] Quantize model test edildi, çıktı kalitesi kabul edilebilir
- [ ] GGUF dönüşümü yapıldı (yerel kullanım için)

#### Inference

- [ ] `device_map="auto"` veya manuel atama yapıldı
- [ ] Generation parametreleri kullanım amacına göre ayarlandı
- [ ] `repetition_penalty` eklendi
- [ ] Chat template ile test yapıldı
- [ ] `skip_special_tokens=True` eklendi

#### Deployment

- [ ] Deployment yöntemi seçildi (vLLM / Ollama / FastAPI / HF Endpoints)
- [ ] API test edildi
- [ ] Hata senaryoları test edildi (uzun girdi, boş girdi, özel karakterler)

#### Hub

- [ ] Model Hub'a yüklendi
- [ ] Model kartı (README) yazıldı
- [ ] Lisans belirlendi
- [ ] Örnek kullanım kodu eklendi

### D — Kaynakça

**Quantization:**
- [bitsandbytes dokümantasyonu](https://huggingface.co/docs/bitsandbytes/)
- [GPTQ paper](https://arxiv.org/abs/2210.17323)
- [llama.cpp / GGUF](https://github.com/ggerganov/llama.cpp)

**Deployment:**
- [vLLM dokümantasyonu](https://docs.vllm.ai/)
- [Ollama](https://ollama.ai/)
- [Hugging Face Inference Endpoints](https://huggingface.co/docs/inference-endpoints/)

**Inference:**
- [Hugging Face Text Generation](https://huggingface.co/docs/transformers/main_classes/text_generation)
- [Optimum kütüphanesi](https://huggingface.co/docs/optimum/)

---

**Serinin başına dön:** [01 - Hızlı Başlangıç ve Mimari](./01 - Hızlı Başlangıç ve Mimari.md)  
**Eğitim rehberine dön:** [02 - Uçtan Uca Eğitim Rehberi](./02 - Uçtan Uca Eğitim Rehberi.md)
