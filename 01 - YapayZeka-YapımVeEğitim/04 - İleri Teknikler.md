# 04 - İleri Teknikler: Eğitim Stabilitesi ve Optimizasyon

Bu rehber, Türkçe dil modeli eğitiminde karşılaşılan stabilite sorunlarını çözmek ve eğitim verimliliğini artırmak için kullanılan ileri seviye teknikleri ele alır. Her teknik için matematiksel temel, uygulama kodu ve Türkçe model bağlamında pratik öneriler sunulur.

**Ön koşul:** [01](./01%20-%20Hızlı%20Başlangıç%20ve%20Mimari.md), [02](./02%20-%20Uçtan%20Uca%20Eğitim%20Rehberi.md) ve [03](./03%20-%20Küçültme%20Dağıtım%20ve%20Tahminleme.md) dosyalarını okumuş olmak.

**Hedef kitle:** Eğitim altyapısına hâkim, stabilite sorunlarıyla bizzat karşılaşmış veya araştırma yapan okuyucular.

---

## İçindekiler

1. [Z-Loss — Logit Stabilizasyonu](#z-loss--logit-stabilizasyonu)
2. [Stochastic Depth — Katman Dropout](#stochastic-depth--katman-dropout)
3. [Gradient Clipping Stratejileri](#gradient-clipping-stratejileri)
4. [Mixed Precision ve bf16 Nüansları](#mixed-precision-ve-bf16-nüansları)
5. [Learning Rate Finder ve Warmup Stratejileri](#learning-rate-finder-ve-warmup-stratejileri)
6. [Alternatif Optimizer'lar: Muon ve Lion](#alternatif-optimizerlar-muon-ve-lion)
7. [Tekniklerin Birlikte Kullanımı](#tekniklerin-birlikte-kullanımı)
8. [Ekler](#ekler)

---

## Z-Loss — Logit Stabilizasyonu

### Motivasyon

Büyük transformer modellerinde eğitimin ilerleyen aşamalarında logit değerleri anormal büyüklüklere ulaşabilir. Bu durum:

- Softmax çıktısının sayısal olarak bozulmasına (`NaN`, `Inf`)
- Loss'un ani sıçramalarına (loss spike)
- Gradyanların patlamasına

yol açar. Standart cross-entropy loss bu sorunu doğrudan ele almaz.

### Matematiksel Temel

Z-loss, Zoph et al. (2022) tarafından PaLM modelinde tanıtılmıştır. Temel fikir: log-partition fonksiyonunun (log-sum-exp) karesi üzerine bir ceza terimi eklemek.

Standart cross-entropy:

$$\mathcal{L}_{CE} = -\log \frac{e^{z_y}}{\sum_j e^{z_j}}$$

Z-loss ile genişletilmiş form:

$$\mathcal{L} = \mathcal{L}_{CE} + \alpha \cdot \left(\log \sum_j e^{z_j}\right)^2$$

Burada:
- $z_j$: $j$. token için ham logit değeri
- $\alpha$: Z-loss ağırlık katsayısı (tipik: `1e-4` ile `1e-2` arası)
- $\log \sum_j e^{z_j}$: log-partition fonksiyonu (log-normalizer)

**Neden karesi alınıyor?** Log-partition fonksiyonu sıfır etrafında cezasız olmalıdır; karesi almak, pozitif ve negatif sapmaları eşit biçimde cezalandıran simetrik bir terim üretir.

**Gradyan analizi:**

$$\frac{\partial \mathcal{L}_{Z}}{\partial z_j} = 2\alpha \cdot \left(\log \sum_k e^{z_k}\right) \cdot \text{softmax}(z)_j$$

Bu gradyan, logit normu büyüdükçe orantılı biçimde artar — otomatik bir geri besleme mekanizması oluşturur.

### Uygulama

```python
import torch
import torch.nn.functional as F

def z_loss(logits: torch.Tensor, alpha: float = 1e-4) -> torch.Tensor:
    """
    Z-loss hesapla.

    Args:
        logits: Ham model çıktıları [batch, seq_len, vocab_size]
        alpha:  Z-loss ağırlık katsayısı

    Returns:
        Skaler z-loss değeri
    """
    # Log-partition fonksiyonu: log(sum(exp(z_j)))
    # torch.logsumexp sayısal kararlılık için optimize edilmiştir
    log_z = torch.logsumexp(logits, dim=-1)  # [batch, seq_len]

    # Karesini al ve ortalamasını hesapla
    return alpha * (log_z ** 2).mean()


def cross_entropy_with_z_loss(
    logits: torch.Tensor,
    labels: torch.Tensor,
    alpha: float = 1e-4,
    ignore_index: int = -100
) -> dict:
    """
    Cross-entropy + Z-loss birleşik kayıp fonksiyonu.

    Returns:
        Dict: {'loss': toplam, 'ce_loss': CE kaybı, 'z_loss': Z kaybı}
    """
    # Standart cross-entropy
    ce = F.cross_entropy(
        logits.view(-1, logits.size(-1)),
        labels.view(-1),
        ignore_index=ignore_index
    )

    # Z-loss (pad tokenlarını maskele)
    mask = labels != ignore_index  # [batch, seq_len]
    masked_logits = logits[mask]   # [N, vocab_size]
    zl = z_loss(masked_logits.unsqueeze(0), alpha=alpha)

    return {
        "loss": ce + zl,
        "ce_loss": ce.detach(),
        "z_loss": zl.detach()
    }
```

#### Hugging Face Trainer ile Entegrasyon

```python
from transformers import Trainer
import torch

class ZLossTrainer(Trainer):
    def __init__(self, *args, z_loss_alpha: float = 1e-4, **kwargs):
        super().__init__(*args, **kwargs)
        self.z_loss_alpha = z_loss_alpha

    def compute_loss(self, model, inputs, return_outputs=False, **kwargs):
        labels = inputs.pop("labels")
        outputs = model(**inputs)
        logits = outputs.logits

        result = cross_entropy_with_z_loss(
            logits, labels, alpha=self.z_loss_alpha
        )

        # WandB / loglama için ayrı kayıpları sakla
        if self.args.report_to and "wandb" in self.args.report_to:
            import wandb
            wandb.log({
                "train/ce_loss": result["ce_loss"].item(),
                "train/z_loss": result["z_loss"].item(),
            })

        return (result["loss"], outputs) if return_outputs else result["loss"]


# Kullanım
trainer = ZLossTrainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    z_loss_alpha=1e-4  # Türkçe 200M model için önerilen başlangıç
)
```

### Türkçe Model İçin Pratik Notlar

| Model Boyutu | Önerilen `alpha` | Ne Zaman Etkinleştir |
|---|---|---|
| 100M - 200M | `1e-4` | İlk eğitimden itibaren |
| 1B | `5e-5` | İlk eğitimden itibaren |
| 7B+ | `1e-5` | Loss spike görülürse |

> **Türkçe özel:** Türkçe'nin geniş vocab'ı (50K token) logit vektörünün boyutunu artırır. Büyük vocab + derin model kombinasyonunda log-partition değerleri daha hızlı büyür. `alpha=1e-4` ile başlayıp ilk 1000 adımda `z_loss` değerini izleyin; `> 0.1` görürseniz `alpha`'yı artırın.

### Paper Referansı

> Zoph, B., et al. (2022). *ST-MoE: Designing Stable and Transferable Sparse Expert Models.* arXiv:2202.08906

---

## Stochastic Depth — Katman Dropout

### Motivasyon

Derin ağlarda (12+ katman) eğitim sırasında her katmanın her adımda aktif olması gerekmez. Stochastic Depth, eğitim sırasında rastgele katmanları "atlar" — bu hem regularizasyon sağlar hem de eğitimi hızlandırır.

### Matematiksel Temel

Huang et al. (2016) tarafından önerilen bu teknikte her $l$. katman $p_l$ olasılıkla atlanır:

$$y_l = \begin{cases} x_l + \mathcal{F}_l(x_l) & \text{olasılıkla } (1 - p_l) \\ x_l & \text{olasılıkla } p_l \end{cases}$$

Burada $\mathcal{F}_l$ katmanın dönüşüm fonksiyonu, $x_l$ ise giriş residual'ıdır.

**Lineer decay (önerilen):** Erken katmanlar daha az atlanır, derin katmanlar daha çok:

$$p_l = p_{max} \cdot \frac{l}{L}$$

Burada $L$ toplam katman sayısı, $p_{max}$ maksimum atlama olasılığıdır.

**Neden residual bağlantı şart?** Katman atlandığında çıktı $x_l$ olarak geçer — bu yalnızca skip connection (residual) varsa anlamlıdır. Transformer'lar zaten residual kullandığından stochastic depth doğal uyum sağlar.

**Beklenen eğitim derinliği:**

$$\mathbb{E}[\text{aktif katman}] = L - \sum_{l=1}^{L} p_l = L\left(1 - \frac{p_{max}}{2}\right)$$

### Uygulama

```python
import torch
import torch.nn as nn

class StochasticDepth(nn.Module):
    """
    Tek bir transformer katmanını stochastic depth ile saran wrapper.

    Args:
        layer:      Sarılacak katman (attention veya FFN bloğu)
        drop_prob:  Bu katman için atlama olasılığı
    """
    def __init__(self, layer: nn.Module, drop_prob: float = 0.0):
        super().__init__()
        self.layer = layer
        self.drop_prob = drop_prob

    def forward(self, x: torch.Tensor, **kwargs) -> torch.Tensor:
        # Inference modunda veya drop_prob=0 ise her zaman çalıştır
        if not self.training or self.drop_prob == 0.0:
            return self.layer(x, **kwargs)

        # Bernoulli örnekleme: bu adımda katmanı atla mı?
        keep_prob = 1.0 - self.drop_prob
        # [batch_size, 1, 1] — broadcast için
        shape = (x.shape[0],) + (1,) * (x.ndim - 1)
        mask = torch.bernoulli(
            torch.full(shape, keep_prob, device=x.device)
        )

        # Eğitimde beklenti = inference çıktısı (ölçekleme)
        return self.layer(x, **kwargs) * mask / keep_prob


def apply_stochastic_depth(
    model: nn.Module,
    p_max: float = 0.2,
    verbose: bool = True
) -> nn.Module:
    """
    Modeldeki tüm transformer katmanlarına lineer decay ile
    stochastic depth uygular.

    Args:
        model:   Llama / GPT-2 tarzı transformer modeli
        p_max:   En derin katman için maksimum atlama olasılığı
        verbose: Katman olasılıklarını yazdır

    Returns:
        Stochastic depth uygulanmış model
    """
    layers = model.model.layers  # Llama için; GPT-2'de model.transformer.h
    L = len(layers)

    for l_idx, layer in enumerate(layers):
        # Lineer decay: ilk katman p≈0, son katman p=p_max
        p_l = p_max * (l_idx / L)

        # Attention ve MLP bloklarını ayrı ayrı sar
        layer.self_attn = StochasticDepth(layer.self_attn, drop_prob=p_l)
        layer.mlp = StochasticDepth(layer.mlp, drop_prob=p_l)

        if verbose:
            print(f"Katman {l_idx:2d}: drop_prob = {p_l:.4f}")

    return model


# Kullanım
model = apply_stochastic_depth(model, p_max=0.2)
```

#### timm Kütüphanesi ile (Alternatif)

```python
from timm.models.layers import DropPath

# timm'in DropPath implementasyonu stochastic depth ile eşdeğerdir
# ve production-grade optimize edilmiştir
drop_path = DropPath(drop_prob=0.1)
```

### Türkçe Model İçin Pratik Notlar

| Model Derinliği | Önerilen `p_max` | Beklenen Hızlanma |
|---|---|---|
| 6 katman | `0.05` | ~%5 |
| 12 katman | `0.1` | ~%10 |
| 24 katman | `0.2` | ~%18 |
| 32 katman | `0.3` | ~%25 |

> **Uyarı:** `p_max > 0.3` değerlerinde perplexity bozulabilir. Türkçe 200M (12 katman) model için `0.1` ile başlayın, validation perplexity'yi takip edin.

> **İpucu:** Stochastic depth yalnızca eğitimde aktiftir. Inference'ta tüm katmanlar çalışır — model kartında bunu belirtmenize gerek yok.

### Paper Referansı

> Huang, G., et al. (2016). *Deep Networks with Stochastic Depth.* ECCV 2016. arXiv:1603.09382

---

## Gradient Clipping Stratejileri

### Motivasyon

Gradyan patlaması (gradient explosion), özellikle sıfırdan eğitimin ilk 500 adımında kritik bir sorundur. Standart `max_norm` clipping çoğu durumda yeterli olsa da, daha sofistike stratejiler eğitim stabilitesini önemli ölçüde artırır.

### Matematiksel Temel

**Global norm clipping (standart):**

$$g \leftarrow g \cdot \min\left(1, \frac{\tau}{\|g\|_2}\right)$$

Burada $\tau$ eşik değeri (tipik: `1.0`), $\|g\|_2$ tüm parametrelerin gradyanlarının birleşik L2 normu.

**Sorun:** Global norm, farklı katmanların gradyan büyüklüklerini homojenleştirir. Embedding katmanının gradyanı (çok büyük) ile son katmanın gradyanı (küçük) aynı ölçekleme faktörüne tabi olur.

**Per-layer norm clipping (gelişmiş):**

Her katman için ayrı eşik:

$$g_l \leftarrow g_l \cdot \min\left(1, \frac{\tau_l}{\|g_l\|_2}\right)$$

### Uygulama

#### Standart Global Clipping

```python
# Hugging Face Trainer otomatik yapar:
training_args = TrainingArguments(
    max_grad_norm=1.0,  # Standart; çoğu durumda yeterli
    ...
)

# Manuel kullanım:
optimizer.zero_grad()
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

#### Gradyan Normunu İzleme

Clipping yapmadan önce gradyan normunu izlemek, sorunları erkenden yakalamanın en iyi yoludur:

```python
def get_grad_norm(model: nn.Module) -> float:
    """Modelin tüm parametrelerinin gradyan normunu hesapla."""
    total_norm = 0.0
    for p in model.parameters():
        if p.grad is not None:
            total_norm += p.grad.data.norm(2).item() ** 2
    return total_norm ** 0.5


class GradientMonitorTrainer(Trainer):
    def training_step(self, model, inputs, num_items_in_batch=None):
        loss = super().training_step(model, inputs, num_items_in_batch)

        # Clipping'den ÖNCE normu ölç
        grad_norm = get_grad_norm(model)

        if self.args.report_to and "wandb" in self.args.report_to:
            import wandb
            wandb.log({
                "train/grad_norm": grad_norm,
                "train/grad_norm_clipped": min(grad_norm, self.args.max_grad_norm)
            })

        # Aşırı büyük gradyan uyarısı
        if grad_norm > 10.0:
            print(f"⚠️  Yüksek gradyan normu: {grad_norm:.2f} (adım {self.state.global_step})")

        return loss
```

#### Adaptive Gradient Clipping (AGC)

Brock et al. (2021) tarafından NFNet'te önerilen bu yöntem, gradyan normunu parametre normuyla orantılı olarak sınırlar:

$$\|g_l\|_2 \leq \lambda \cdot \|W_l\|_F$$

Bu sayede her katmanın kendi büyüklüğüne göre normalize edilmiş bir eşik oluşur.

```python
def adaptive_gradient_clipping(
    model: nn.Module,
    clip_factor: float = 0.01,
    eps: float = 1e-3
) -> float:
    """
    Adaptive Gradient Clipping (AGC).

    Args:
        model:       Model
        clip_factor: lambda — parametre normunun kaçta biri (önerilen: 0.01)
        eps:         Sayısal kararlılık için minimum norm

    Returns:
        Clipping sonrası maksimum gradyan normu
    """
    max_norm = 0.0

    for name, param in model.named_parameters():
        if param.grad is None:
            continue

        # Embedding katmanlarını atla (farklı ölçek)
        if "embed" in name:
            continue

        # Parametre ve gradyan normları
        param_norm = param.data.norm(2).clamp(min=eps)
        grad_norm = param.grad.data.norm(2)

        # Adaptif eşik
        max_allowed = clip_factor * param_norm

        if grad_norm > max_allowed:
            param.grad.data.mul_(max_allowed / grad_norm)

        max_norm = max(max_norm, grad_norm.item())

    return max_norm


# Kullanım (manuel eğitim döngüsü)
optimizer.zero_grad()
loss.backward()
max_norm = adaptive_gradient_clipping(model, clip_factor=0.01)
optimizer.step()
```

### Türkçe Model İçin Pratik Notlar

**Eğitim aşamasına göre önerilen strateji:**

| Aşama | Strateji | `max_grad_norm` |
|---|---|---|
| Pre-training (ilk 1K adım) | Global clipping | `0.5` (daha sıkı) |
| Pre-training (sonrası) | Global clipping | `1.0` |
| SFT | Global clipping | `1.0` |
| DPO / RLHF | AGC | `clip_factor=0.01` |

> **Türkçe özel:** Türkçe'nin uzun kelime formları (eklemeli yapı) tokenizer'ı daha fazla zorlar; bazı adımlarda embedding gradyanları olağandışı büyüyebilir. İlk 500 adımda `max_grad_norm=0.5` kullanmak ve grad_norm'u loglamak önerilir.

### Paper Referansları

> Pascanu, R., et al. (2013). *On the difficulty of training recurrent neural networks.* ICML 2013.

> Brock, A., et al. (2021). *High-Performance Large-Scale Image Recognition Without Normalization.* arXiv:2102.06171

---

## Mixed Precision ve bf16 Nüansları

### Motivasyon

Mixed precision eğitimi, hesaplamaları düşük hassasiyette (fp16 / bf16) yaparak hem VRAM'i hem de hesaplama süresini azaltır. Ancak fp16 ve bf16 arasındaki farklar, Türkçe model eğitiminde kritik öneme sahiptir.

### Matematiksel Temel: fp16 vs bf16

| Özellik | fp32 | fp16 | bf16 |
|---|---|---|---|
| Toplam bit | 32 | 16 | 16 |
| Üs (exponent) biti | 8 | 5 | 8 |
| Mantis (mantissa) biti | 23 | 10 | 7 |
| Maksimum değer | ~3.4×10³⁸ | ~6.5×10⁴ | ~3.4×10³⁸ |
| Hassasiyet | Yüksek | Yüksek | Düşük |
| Taşma riski | Yok | **Yüksek** | Yok |

**Temel fark:**
- **fp16:** Mantis biti fazla → yüksek hassasiyet, ama üs aralığı dar → büyük değerlerde `Inf` / `NaN` riski
- **bf16:** fp32 ile aynı üs aralığı → taşma yok, ama hassasiyet düşük → küçük gradyanlar sıfırlanabilir

**Sonuç:** Modern GPU'larda (A100, H100, RTX 3090+) **bf16 tercih edilmeli.** fp16 eski GPU'lar için (V100, T4) geçerli bir seçenek olmaya devam eder.

### Loss Scaling (fp16 için)

fp16'da küçük gradyanlar underflow (sıfırlanma) yaşar. Loss scaling bu sorunu çözer:

$$\tilde{\mathcal{L}} = s \cdot \mathcal{L}$$

İleri passta loss `s` ile çarpılır, geri passta gradyanlar `s`'e bölünür. `s` dinamik olarak ayarlanır:

```
Eğer NaN/Inf yoksa: s ← s × growth_factor (örn. 2.0)
Eğer NaN/Inf varsa: s ← s / backoff_factor (örn. 0.5), adımı atla
```

### Uygulama

```python
from transformers import TrainingArguments
import torch

# bf16 (önerilen — A100/H100/RTX 30xx+)
training_args = TrainingArguments(
    bf16=True,
    bf16_full_eval=True,
    ...
)

# fp16 (eski GPU — V100, T4)
training_args = TrainingArguments(
    fp16=True,
    fp16_opt_level="O1",   # O1: mixed, O2: daha agresif, O3: tam fp16
    ...
)
```

#### Manuel Loss Scaler (fp16)

```python
from torch.cuda.amp import GradScaler, autocast

scaler = GradScaler(
    init_scale=2**16,       # Başlangıç ölçek faktörü
    growth_factor=2.0,      # Her growth_interval adımda 2x büyüt
    backoff_factor=0.5,     # NaN'da 2x küçült
    growth_interval=2000,   # Kaç adımda bir büyütmeyi dene
    enabled=True
)

for batch in dataloader:
    optimizer.zero_grad()

    with autocast(dtype=torch.float16):
        outputs = model(**batch)
        loss = outputs.loss

    scaler.scale(loss).backward()

    # Clipping ÖNCE unscale et
    scaler.unscale_(optimizer)
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

    scaler.step(optimizer)
    scaler.update()

    # Ölçek faktörünü izle
    current_scale = scaler.get_scale()
    if current_scale < 1.0:
        print(f"⚠️  Loss scale çok düştü: {current_scale}")
```

#### bf16 Stabilite Kontrolü

bf16'nın hassasiyeti düşük olduğundan bazı operasyonlar fp32'de kalmalıdır:

```python
# Norm katmanları fp32'de tutmak için
from transformers import LlamaConfig

config = LlamaConfig(
    ...
    torch_dtype=torch.bfloat16,
)

model = LlamaForCausalLM(config).to(torch.bfloat16)

# RMSNorm parametrelerini fp32'ye yükselt
for module in model.modules():
    if isinstance(module, nn.LayerNorm) or "norm" in type(module).__name__.lower():
        module.float()
```

### Sayısal Kararlılık Testleri

```python
def check_numerical_stability(model, sample_batch, device="cuda"):
    """
    Eğitime başlamadan önce sayısal kararlılığı test et.
    """
    model.eval()
    results = {}

    with torch.no_grad():
        for dtype in [torch.float32, torch.float16, torch.bfloat16]:
            try:
                m = model.to(dtype).to(device)
                inputs = {k: v.to(device) for k, v in sample_batch.items()}

                out = m(**inputs)
                logits = out.logits

                has_nan = torch.isnan(logits).any().item()
                has_inf = torch.isinf(logits).any().item()
                logit_max = logits.abs().max().item()

                results[str(dtype)] = {
                    "nan": has_nan,
                    "inf": has_inf,
                    "max_logit": logit_max,
                    "stable": not (has_nan or has_inf)
                }
            except Exception as e:
                results[str(dtype)] = {"error": str(e)}

    for dtype, r in results.items():
        print(f"{dtype:20s}: {r}")

    return results
```

### Türkçe Model İçin Pratik Notlar

> **bf16 + Türkçe vocab:** 50K token boyutunda vocab ile lm_head katmanı `[hidden_size × 50000]` boyutunda bir matris oluşturur. Bu matrisin bf16'da saklanması VRAM'i önemli ölçüde azaltır. Ancak son katman logitlerinin hassasiyeti düşeceğinden **Z-loss ile birlikte kullanılması önerilir.**

| GPU Nesli | Önerilen Precision | Not |
|---|---|---|
| V100, T4 | fp16 + GradScaler | bf16 desteği yok |
| RTX 30xx, A100, H100 | bf16 | Taşma riski yok, daha stabil |
| Apple M serisi | bf16 (MPS) | `device="mps"` |

### Paper Referansı

> Micikevicius, P., et al. (2018). *Mixed Precision Training.* ICLR 2018. arXiv:1710.03740

---

## Learning Rate Finder ve Warmup Stratejileri

### Motivasyon

Learning rate, eğitimin en kritik hiper-parametresidir. Çok büyük seçilirse loss ıraksıyor (diverge), çok küçük seçilirse eğitim yavaş ve verimsiz kalır. Warmup ise modelin eğitim başında stabil kalmasını sağlar.

### Matematiksel Temel

**Cosine annealing with warmup:**

Warmup fazında ($t \leq T_w$):

$$\eta_t = \eta_{max} \cdot \frac{t}{T_w}$$

Cosine decay fazında ($t > T_w$):

$$\eta_t = \eta_{min} + \frac{1}{2}(\eta_{max} - \eta_{min})\left(1 + \cos\left(\pi \cdot \frac{t - T_w}{T_{max} - T_w}\right)\right)$$

**Neden warmup şart?**

Eğitimin başında model ağırlıkları rastgeledir. Büyük bir learning rate ile ilk adımlarda dev gradyanlar oluşur ve Adam optimizer'ın ikinci moment tahmini ($v_t$) henüz doğru değildir:

$$v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2$$

İlk adımlarda $v_t \approx (1-\beta_2) g_t^2$ — bu hatalı bir ölçekleme üretir. Warmup, $v_t$'nin olgunlaşmasına zaman tanır.

**Neden cosine decay?**

Lineer decay'e göre cosine, eğitimin son aşamalarında daha yavaş düşer — model "öğrenmeye" devam eder. Ani sıfırlanma yerine yumuşak iniş sağlar.

### Learning Rate Finder

```python
import torch
import numpy as np
from torch.optim.lr_scheduler import LambdaLR

class LRFinder:
    """
    Leslie Smith'in Learning Rate Range Test implementasyonu.
    (Smith, 2017 — arXiv:1506.01186)
    """
    def __init__(
        self,
        model,
        optimizer,
        criterion,
        device="cuda"
    ):
        self.model = model
        self.optimizer = optimizer
        self.criterion = criterion
        self.device = device
        self.history = {"lr": [], "loss": []}
        self.best_loss = float("inf")

    def range_test(
        self,
        train_loader,
        start_lr: float = 1e-7,
        end_lr: float = 1.0,
        num_iter: int = 100,
        smooth_f: float = 0.05,
        diverge_th: float = 5.0
    ):
        """
        start_lr'dan end_lr'ye logaritmik olarak artırarak loss'u izle.

        Args:
            smooth_f:    Exponential moving average faktörü
            diverge_th:  Best loss'un kaç katında dur
        """
        lrs = np.logspace(
            np.log10(start_lr),
            np.log10(end_lr),
            num_iter
        )

        smooth_loss = None
        self.model.train()

        for i, (batch, lr) in enumerate(zip(train_loader, lrs)):
            # LR'yi güncelle
            for pg in self.optimizer.param_groups:
                pg["lr"] = lr

            # Forward + backward
            self.optimizer.zero_grad()
            inputs = {k: v.to(self.device) for k, v in batch.items()}
            outputs = self.model(**inputs)
            loss = outputs.loss
            loss.backward()
            self.optimizer.step()

            # Exponential smoothing
            current_loss = loss.item()
            if smooth_loss is None:
                smooth_loss = current_loss
            else:
                smooth_loss = smooth_f * current_loss + (1 - smooth_f) * smooth_loss

            self.history["lr"].append(lr)
            self.history["loss"].append(smooth_loss)

            if smooth_loss < self.best_loss:
                self.best_loss = smooth_loss

            # Diverge kontrolü
            if smooth_loss > diverge_th * self.best_loss:
                print(f"Loss ıraksamaya başladı. Duruyorum (adım {i})")
                break

        return self.history

    def suggest_lr(self) -> float:
        """
        En hızlı düşüşün yaşandığı LR'yi öner.
        (Gradyanın minimum olduğu nokta)
        """
        losses = self.history["loss"]
        lrs = self.history["lr"]

        # Gradyanı hesapla
        gradients = np.gradient(losses)
        min_grad_idx = np.argmin(gradients)

        suggested = lrs[min_grad_idx]
        print(f"Önerilen LR: {suggested:.2e}")
        print(f"(Güvenli başlangıç: {suggested / 10:.2e} — 10x daha küçük)")
        return suggested


# Kullanım
from torch.optim import AdamW

optimizer = AdamW(model.parameters(), lr=1e-7)  # Başlangıç LR önemsiz
finder = LRFinder(model, optimizer, criterion=None, device="cuda")

history = finder.range_test(train_loader, start_lr=1e-7, end_lr=1.0, num_iter=100)
suggested_lr = finder.suggest_lr()
```

### Model Boyutuna Göre LR Önerileri

| Model Boyutu | Pre-training LR | SFT LR | Warmup Adımı |
|---|---|---|---|
| 100M - 200M | `3e-4` | `2e-5` | toplam adımın %1-2'si |
| 1B | `1e-4` | `1e-5` | toplam adımın %1'i |
| 7B | `3e-5` | `5e-6` | toplam adımın %0.5'i |

### Warmup Stratejileri

```python
from transformers import get_scheduler

# 1. Cosine with warmup (önerilen)
scheduler = get_scheduler(
    "cosine",
    optimizer=optimizer,
    num_warmup_steps=500,        # İlk 500 adım lineer artış
    num_training_steps=50000     # Toplam adım
)

# 2. Linear decay
scheduler = get_scheduler(
    "linear",
    optimizer=optimizer,
    num_warmup_steps=500,
    num_training_steps=50000
)

# 3. Constant with warmup (SFT için)
scheduler = get_scheduler(
    "constant_with_warmup",
    optimizer=optimizer,
    num_warmup_steps=100
)

# 4. Cosine with restarts (uzun pre-training için)
scheduler = get_scheduler(
    "cosine_with_restarts",
    optimizer=optimizer,
    num_warmup_steps=500,
    num_training_steps=50000,
    num_cycles=3  # Kaç kez restart yapılsın
)
```

### Türkçe Model İçin Pratik Notlar

> **Warmup süresi:** Türkçe 200M model için 500 adım warmup genellikle yeterlidir. Veri seti çok gürültülüyse (web crawl) 1000 adıma çıkın.

> **LR Finder + Türkçe:** LR finder'ı Türkçe verinin temsili bir alt kümesi (10K satır) üzerinde çalıştırın. Common Crawl gibi gürültülü kaynaklarda loss dalgalanması yüksek olduğundan `smooth_f=0.1` kullanın.

### Paper Referansları

> Smith, L. N. (2017). *Cyclical Learning Rates for Training Neural Networks.* WACV 2017. arXiv:1506.01186

> Loshchilov, I., Hutter, F. (2017). *SGDR: Stochastic Gradient Descent with Warm Restarts.* ICLR 2017. arXiv:1608.03983

---

## Alternatif Optimizer'lar: Muon ve Lion

### Motivasyon

AdamW, transformer eğitiminde fiilen standart haline gelmiştir. Ancak 2023-2024 döneminde iki yeni optimizer öne çıkmaktadır: **Lion** (Evol. optimization) ve **Muon** (Momentum + Orthogonalization). Her ikisi de belirli senaryolarda AdamW'den üstün performans göstermektedir.

### AdamW — Referans Nokta

```python
from torch.optim import AdamW

optimizer = AdamW(
    model.parameters(),
    lr=3e-4,
    betas=(0.9, 0.999),  # β1, β2 — momentum katsayıları
    eps=1e-8,
    weight_decay=0.1     # L2 regularizasyon
)
```

**AdamW güncelleme kuralı:**

$$m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t$$
$$v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2$$
$$\theta_t = \theta_{t-1} - \eta \cdot \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} - \eta \lambda \theta_{t-1}$$

**Zayıf noktası:** $v_t$ (ikinci moment) yavaş güncellenir → eğitim başında ve ani dağılım değişimlerinde yavaş adapte olur.

---

### Lion — EvoLved Sign Momentum

Chen et al. (2023) tarafından otomatik arama (AutoML) ile keşfedilen Lion, AdamW'ye kıyasla %2-10 arası performans iyileştirmesi sunar ve **daha düşük VRAM** kullanır (ikinci moment yok).

**Güncelleme kuralı:**

$$c_t = \text{sign}(\beta_1 m_{t-1} + (1 - \beta_1) g_t)$$
$$\theta_t = \theta_{t-1} - \eta (c_t + \lambda \theta_{t-1})$$
$$m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t$$

**Temel fark:** Gradyan değeri yerine **işareti** (sign) kullanılır → tüm parametreler eşit büyüklükte güncellenir.

```python
import torch

class Lion(torch.optim.Optimizer):
    """
    Lion optimizer.
    Chen et al. (2023) — arXiv:2302.06675
    """
    def __init__(
        self,
        params,
        lr: float = 1e-4,
        betas: tuple = (0.9, 0.99),
        weight_decay: float = 0.0
    ):
        defaults = dict(lr=lr, betas=betas, weight_decay=weight_decay)
        super().__init__(params, defaults)

    @torch.no_grad()
    def step(self, closure=None):
        loss = None
        if closure is not None:
            with torch.enable_grad():
                loss = closure()

        for group in self.param_groups:
            for p in group["params"]:
                if p.grad is None:
                    continue

                grad = p.grad
                beta1, beta2 = group["betas"]
                lr = group["lr"]
                wd = group["weight_decay"]

                state = self.state[p]
                if len(state) == 0:
                    state["exp_avg"] = torch.zeros_like(p)

                exp_avg = state["exp_avg"]

                # Güncelleme yönü: sign(β1*m + (1-β1)*g)
                update = exp_avg.mul(beta1).add_(grad, alpha=1 - beta1)

                # Weight decay + sign güncelleme
                p.add_(update.sign_().add_(p, alpha=wd), alpha=-lr)

                # Momentum güncelle
                exp_avg.mul_(beta2).add_(grad, alpha=1 - beta2)

        return loss


# Kullanım
optimizer = Lion(
    model.parameters(),
    lr=1e-4,          # AdamW'den ~3-10x daha küçük LR kullan
    betas=(0.9, 0.99),
    weight_decay=0.1
)
```

> **Önemli:** Lion için LR, AdamW'ye kıyasla **3-10x daha küçük** seçilmelidir. AdamW'de `3e-4` kullanıyorsanız Lion'da `3e-5` ile başlayın.

---

### Muon — Momentum + Nesterov + Orthogonalization

Kosson et al. (2023) ve Jordan et al. (2024) tarafından geliştirilen Muon, özellikle transformer'ların **lineer katmanları** için tasarlanmıştır. Nesterov momentum'u Newton-Schulz iterasyonu ile ortogonalize eder.

**Matematiksel temel:**

Standart SGD + Nesterov momentum:

$$m_t = \mu m_{t-1} + g_t$$
$$\theta_t = \theta_{t-1} - \eta (g_t + \mu m_t)$$

Muon'un eklediği ortogonalizasyon adımı:

$$G_{orth} = \text{NS}(G, k=5)$$

Burada NS, Newton-Schulz iterasyonudur:

$$X_0 = G / \|G\|_F$$
$$X_{i+1} = \frac{3}{2} X_i - \frac{1}{2} X_i X_i^T X_i$$

Bu işlem gradyanı yaklaşık olarak ortogonal (unitary) yapar — eğitimi matris boyutundan bağımsız hale getirir.

```python
import torch
import torch.nn.functional as F

def newton_schulz(G: torch.Tensor, steps: int = 5, eps: float = 1e-7) -> torch.Tensor:
    """
    Newton-Schulz iterasyonu ile yaklaşık ortogonalizasyon.

    Args:
        G:     Gradyan matrisi [m, n]
        steps: İterasyon sayısı (5 genellikle yeterli)
        eps:   Sayısal kararlılık için minimum norm
    """
    assert G.ndim == 2, "Muon 2D gradyanlar için tasarlanmıştır"
    a, b, c = (3.4445, -4.7750, 2.0315)

    X = G.bfloat16() / (G.norm() + eps)

    # Kare olmayan matrisler için transpoz düzenleme
    if G.size(0) > G.size(1):
        X = X.T

    for _ in range(steps):
        A = X @ X.T
        X = a * X + b * A @ X + c * A @ A @ X

    if G.size(0) > G.size(1):
        X = X.T

    return X.to(G.dtype)


class Muon(torch.optim.Optimizer):
    """
    Muon optimizer — lineer katmanlar için.
    Embedding ve norm katmanları için AdamW ile birlikte kullanın.

    Jordan et al. (2024) — github.com/KellerJordan/Muon
    """
    def __init__(
        self,
        params,
        lr: float = 0.02,
        momentum: float = 0.95,
        nesterov: bool = True,
        ns_steps: int = 5
    ):
        defaults = dict(
            lr=lr,
            momentum=momentum,
            nesterov=nesterov,
            ns_steps=ns_steps
        )
        super().__init__(params, defaults)

    @torch.no_grad()
    def step(self, closure=None):
        loss = None
        if closure is not None:
            with torch.enable_grad():
                loss = closure()

        for group in self.param_groups:
            lr = group["lr"]
            momentum = group["momentum"]
            nesterov = group["nesterov"]
            ns_steps = group["ns_steps"]

            for p in group["params"]:
                if p.grad is None or p.grad.ndim != 2:
                    continue  # Sadece 2D (lineer katman) ağırlıkları

                g = p.grad
                state = self.state[p]

                if "momentum_buffer" not in state:
                    state["momentum_buffer"] = torch.zeros_like(g)

                buf = state["momentum_buffer"]
                buf.mul_(momentum).add_(g)

                # Nesterov
                g_update = g.add(buf, alpha=momentum) if nesterov else buf

                # Ortogonalize
                g_orth = newton_schulz(g_update, steps=ns_steps)

                # RMS ölçekleme (gradyan büyüklüğünü koru)
                g_orth.mul_(max(1, g_update.size(0) / g_update.size(1)) ** 0.5)

                p.add_(g_orth, alpha=-lr)

        return loss


# Hibrid kullanım: Muon (lineer) + AdamW (diğerleri)
def create_muon_optimizer(model, muon_lr=0.02, adam_lr=3e-4, weight_decay=0.1):
    """
    Lineer katmanlar için Muon, geri kalanlar için AdamW.
    """
    muon_params = []
    adam_params = []

    for name, param in model.named_parameters():
        if param.ndim == 2 and "embed" not in name and "lm_head" not in name:
            muon_params.append(param)
        else:
            adam_params.append(param)

    print(f"Muon parametreleri: {len(muon_params)}")
    print(f"AdamW parametreleri: {len(adam_params)}")

    muon_opt = Muon(muon_params, lr=muon_lr, momentum=0.95)
    adam_opt = AdamW(adam_params, lr=adam_lr, weight_decay=weight_decay)

    return muon_opt, adam_opt
```

### Optimizer Karşılaştırması

| Optimizer | VRAM | Hız | Kararlılık | Ne Zaman Kullan |
|---|---|---|---|---|
| **AdamW** | Yüksek | Referans | Çok iyi | Genel kullanım, her zaman güvenli |
| **Lion** | Düşük (~%30) | Benzer | İyi | VRAM kısıtlıysa, büyük batch |
| **Muon** | Orta | Hızlı | İyi | Araştırma, lineer katman ağırlıklı |
| **SGD + momentum** | En düşük | Yavaş | Değişken | Baseline karşılaştırma |

### Türkçe Model İçin Pratik Notlar

> **Hangi optimizer ile başlamalı?** İlk Türkçe modelinizde **AdamW** ile başlayın. Lion ve Muon'u yalnızca AdamW baseline'ını kurduktan sonra denemek mantıklıdır — karşılaştırma noktanız olmadan iyileşmeyi ölçemezsiniz.

> **Lion + Türkçe:** Geniş vocab (50K) ile lm_head büyüktür; Lion'un ikinci moment gerektirmemesi belirgin VRAM tasarrufu sağlar. `lr=3e-5`, `weight_decay=0.1` ile başlayın.

> **Muon + Türkçe:** Muon embedding katmanlarını desteklemez; Türkçe modelde embedding kritiktir. Hibrid yaklaşım (Muon + AdamW) zorunludur.

### Paper Referansları

> Chen, X., et al. (2023). *Symbolic Discovery of Optimization Algorithms.* arXiv:2302.06675

> Kosson, A., et al. (2023). *Rotational Equilibrium: How Weight Decay Balances Learning Across Neural Networks.* arXiv:2305.17212

> Jordan, K., et al. (2024). *Muon: An optimizer for hidden layers in neural networks.* github.com/KellerJordan/Muon

---

## Tekniklerin Birlikte Kullanımı

### Önerilen Kombinasyonlar

| Senaryo | Z-loss | Stoch. Depth | Grad Clip | Precision | LR Sched. | Optimizer |
|---|---|---|---|---|---|---|
| **200M Türkçe, ilk deneme** | ✅ 1e-4 | ❌ | Global 1.0 | bf16 | Cosine | AdamW |
| **200M Türkçe, stabil eğitim** | ✅ 1e-4 | ✅ 0.1 | Global 0.5→1.0 | bf16 | Cosine | AdamW |
| **1B Türkçe, VRAM kısıtlı** | ✅ 5e-5 | ✅ 0.15 | AGC 0.01 | bf16 | Cosine | Lion |
| **7B, araştırma** | ✅ 1e-5 | ✅ 0.2 | AGC 0.01 | bf16 | Cosine+restart | Muon+AdamW |
| **SFT (herhangi boyut)** | ❌ | ❌ | Global 1.0 | bf16 | Constant+warmup | AdamW |

### Tam Eğitim Döngüsü Örneği

```python
import torch
from transformers import LlamaForCausalLM, LlamaConfig, TrainingArguments
from torch.optim import AdamW
from transformers import get_scheduler

# --- Model ---
config = LlamaConfig(
    vocab_size=50000,
    hidden_size=768,
    num_hidden_layers=12,
    num_attention_heads=12,
    torch_dtype=torch.bfloat16
)
model = LlamaForCausalLM(config).to(torch.bfloat16).cuda()

# --- Stochastic Depth ---
model = apply_stochastic_depth(model, p_max=0.1)

# --- Optimizer ---
optimizer = AdamW(
    model.parameters(),
    lr=3e-4,
    betas=(0.9, 0.999),
    weight_decay=0.1
)

# --- Scheduler ---
scheduler = get_scheduler(
    "cosine",
    optimizer=optimizer,
    num_warmup_steps=500,
    num_training_steps=50000
)

# --- Eğitim döngüsü ---
for step, batch in enumerate(train_loader):
    model.train()
    optimizer.zero_grad()

    inputs = {k: v.cuda() for k, v in batch.items()}
    labels = inputs.pop("labels")

    with torch.autocast(device_type="cuda", dtype=torch.bfloat16):
        outputs = model(**inputs, labels=labels)

    # Z-loss
    result = cross_entropy_with_z_loss(
        outputs.logits, labels, alpha=1e-4
    )
    loss = result["loss"]

    loss.backward()

    # Gradient clipping + norm izleme
    grad_norm = get_grad_norm(model)
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

    optimizer.step()
    scheduler.step()

    # Loglama
    if step % 100 == 0:
        print(
            f"Adım {step:5d} | "
            f"loss={loss.item():.4f} | "
            f"ce={result['ce_loss'].item():.4f} | "
            f"z={result['z_loss'].item():.4f} | "
            f"grad={grad_norm:.2f} | "
            f"lr={scheduler.get_last_lr()[0]:.2e}"
        )
```

---

## Ekler

### A — Terimler Sözlüğü

| Terim | Açıklama |
|---|---|
| **Logit** | Softmax öncesi ham model çıktısı |
| **Log-partition** | $\log \sum_j e^{z_j}$ — normalizasyon terimi |
| **Stochastic Depth** | Eğitimde katmanların rastgele atlanması |
| **Loss Scaling** | fp16'da underflow'u önlemek için kayıp çarpanı |
| **Underflow** | Çok küçük değerin sıfıra yuvarlanması |
| **Warmup** | Eğitim başında LR'nin yavaşça artırılması |
| **Cosine Annealing** | LR'nin kosinüs eğrisiyle azaltılması |
| **Sign Gradient** | Gradyanın yalnızca işaretinin kullanılması (Lion) |
| **Ortogonalizasyon** | Gradyan matrisini yaklaşık unitary yapma (Muon) |
| **Newton-Schulz** | Matris kareköküne yakınsayan iteratif algoritma |
| **AGC** | Adaptive Gradient Clipping |
| **bf16** | Brain float 16 — fp32 ile aynı üs aralığı |
| **Loss Spike** | Eğitim sırasında ani loss artışı |

### B — Genel Checklist

#### Eğitim Öncesi

- [ ] Sayısal kararlılık testi yapıldı (`check_numerical_stability`)
- [ ] LR finder çalıştırıldı veya model boyutuna göre LR seçildi
- [ ] Precision seçildi (bf16 önerilir; GPU uyumluluğu kontrol edildi)
- [ ] Z-loss alpha değeri belirlendi
- [ ] Warmup adım sayısı ayarlandı

#### Eğitim Sırasında İzlenecekler

- [ ] `grad_norm` her adımda loglanıyor
- [ ] `z_loss` ve `ce_loss` ayrı ayrı izleniyor
- [ ] Loss scale (fp16 kullanılıyorsa) izleniyor
- [ ] İlk 100 adımda loss düzenli düşüyor (NaN/Inf yok)

#### Sorun Giderme

- [ ] Loss NaN → `initializer_range` küçült, `max_grad_norm` düşür, Z-loss ekle
- [ ] Loss spike → checkpoint'e dön, LR'yi düşür
- [ ] Çok yavaş öğrenme → LR artır, warmup'ı kısalt
- [ ] OOM → bf16'ya geç, Lion dene, gradient checkpointing ekle

### C — Kaynakça

| Paper | Konu | Link |
|---|---|---|
| Zoph et al. (2022) | Z-loss, PaLM | arXiv:2202.08906 |
| Huang et al. (2016) | Stochastic Depth | arXiv:1603.09382 |
| Brock et al. (2021) | AGC, NFNet | arXiv:2102.06171 |
| Pascanu et al. (2013) | Gradient clipping | ICML 2013 |
| Micikevicius et al. (2018) | Mixed precision | arXiv:1710.03740 |
| Smith (2017) | LR Range Test | arXiv:1506.01186 |
| Loshchilov & Hutter (2017) | Cosine annealing | arXiv:1608.03983 |
| Chen et al. (2023) | Lion optimizer | arXiv:2302.06675 |
| Jordan et al. (2024) | Muon optimizer | github.com/KellerJordan/Muon |

---

**Serinin başına dön:** [01 - Hızlı Başlangıç ve Mimari](./01%20-%20Hızlı%20Başlangıç%20ve%20Mimari.md)  
**Eğitim rehberine dön:** [02 - Uçtan Uca Eğitim Rehberi](./02%20-%20Uçtan%20Uca%20Eğitim%20Rehberi.md)  
**Deployment rehberine dön:** [03 - Küçültme, Dağıtım ve Tahminleme](./03%20-%20Küçültme%20Dağıtım%20ve%20Tahminleme.md)
