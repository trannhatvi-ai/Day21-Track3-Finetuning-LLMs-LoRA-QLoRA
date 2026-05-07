# Lab 21 - Evaluation Report

**Hoc vien**: Tran Nhat Vi - 2A202600497 
**Ngay nop**: 2026-05-07  
**Submission option**: B GitHub

## 1. Setup

- **Base model**: `unsloth/Qwen2.5-3B-bnb-4bit`
- **Dataset**: `5CD-AI/Vietnamese-alpaca-gpt4-gg-translated` (subset 200 samples)
- **Split**: 180 train / 20 eval (90/10, seed=42)
- **Token length analysis**: p50=227, p95=562, p99=704
- **max_seq_length**: 1024 (rounded up from p95, cap=1024)
- **GPU**: Tesla T4 (16 GB profile, notebook measured max around 8.0 GB)
- **Training config**:
  - Epochs: 3
  - LR: 2e-4, cosine schedule
  - Warmup ratio: 0.10
  - Effective batch size: 8 (batch 1 x grad_accum 8)
  - Optimizer: `adamw_8bit`
  - `packing=False`, `gradient_checkpointing=True`
- **Estimated training cost**: 12.24 minutes total (rounded 12.2), ~0.07 USD (@0.35 USD/hour)

## 2. Rank Experiment Results

| Rank | Alpha | Trainable Params | Train Time (min) | Peak VRAM (GB) | Eval Loss | Perplexity |
|------|-------|------------------|------------------|----------------|-----------|------------|
| 8    | 16    | 1,843,200        | 3.9957           | 7.2161         | 1.5577    | 4.7479     |
| 16   | 32    | 3,686,400        | 4.2554           | 6.6178         | 1.5161    | 4.5544     |
| 64   | 128   | 14,745,600       | 3.9928           | 7.9985         | 1.4768    | 4.3790     |
| Base | -     | -                | -                | -              | 1.8840    | 6.5798     |

**Nhan xet trade-off 4 chieu:**
- **Quality**: Rank cao hon cho perplexity tot hon (r64 > r16 > r8).
- **Before/After vs Base**: tat ca adapter deu cai thien ro so voi base (6.5798 -> 4.7479/4.5544/4.3790).
- **Params**: r64 co so tham so trainable gap 4x r16 va 8x r8.
- **VRAM**: r64 cao nhat (~8.0 GB), r16 thap nhat trong lan chay nay.
- **Time**: thoi gian train khong chenh nhieu tren T4 vi bottleneck IO + kernel overhead.

## 3. Loss Curve Analysis

- Duong train loss giam on dinh trong baseline r=16, cho thay qua trinh hoc hoi hoi tu tot.
- Do profile T4 tat eval-during-training de tranh OOM (`eval_strategy="no"`), overfitting duoc kiem tra bang eval cuoi ky thay vi theo tung step.
- Eval loss cuoi cung cua r16 (1.5161) van thap va chat luong qualitative on dinh, nen chua co dau hieu overfitting manh tren subset 200 mau.
- De xac nhan chat hon tren bai nop production, co the bat eval moi epoch neu chuyen sang GPU lon hon.

## 4. Qualitative Comparison (5 examples)

| Prompt | Base model | Fine-tuned r=16 | Nhan xet |
|---|---|---|---|
| Giai thich machine learning cho nguoi moi | Tra loi dung y chinh, giong dinh nghia SGK | Cau tra loi ro cau truc hon, de doc hon | FT tot hon nhe ve do ro rang |
| Viet code Fibonacci bang Python | Dua code de quy/vong lap co ban | Dua code co validate input va thong diep loi ro hon | FT huu ich hon cho nguoi hoc |
| Liet ke 5 nguyen tac UI/UX | Liet ke dung nhung dien dat dai | Trinh bay gon, huong hanh dong hon | FT ngon ngu thuc dung hon |
| Tom tat LoRA vs QLoRA | Dung concept chung | Co xu huong dinh nghia sat theo van phong ky thuat da huan luyen | FT dong bo style domain AI tot hon |
| Phan biet Prompt Engineering / RAG / Fine-tuning | Day du nhung mo ta co phan chung chung | Cau truc so sanh ro, de dua ra quyet dinh pipeline | FT tot hon cho task tu van ky thuat |

## 5. Conclusion ve Rank Trade-off

Voi dataset 200 mau va profile T4, rank `r=16` la diem can bang ROI tot nhat neu muc tieu la deploy practical. Rank `r=8` tiet kiem tham so nhat nhung perplexity cao hon ro, cho thay under-capacity khi can hoc style va format domain. Rank `r=64` cho ket qua perplexity tot nhat, nhung doi lai bang trainable params lon hon rat nhieu va VRAM cao hon. Tren bo du lieu nho, muc cai thien chat luong tu `r=16` len `r=64` co ton tai nhung khong ty le thuan voi chi phi tham so (hieu ung diminishing returns bat dau xuat hien). Neu production uu tien latency, chi phi va do on dinh tren ha tang phan cung pho thong, `r=16` la lua chon khuyen nghi. Neu bai toan yeu cau do chinh xac toi da, domain phuc tap hon va co ngan sach GPU tot hon, co the can nhac `r=64` sau khi mo rong tap eval va test regression ky hon.

## 6. What I Learned

- Chat luong du lieu va format huan luyen anh huong truc tiep den kha nang cai thien chat luong cua adapter, thuong lon hon viec tang rank mot cach may moc.
- `max_seq_length` dat theo p95 la chien luoc tot de can bang giua cover context va VRAM, dac biet tren T4.
- So sanh rank can du 4 metric (time, VRAM, params, perplexity) va qualitative side-by-side; chi nhin mot metric de ket luan de gay sai lech.
