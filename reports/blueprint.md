# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Lê Văn Đông
**Ngày:** 2026-08-26

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~?ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~?ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | ~11000 | 11161.92 | ~12000 | <10ms |
| NeMo Input Rail | ~1000 | 1131.77 | ~1200 | <300ms |
| RAG Pipeline | - | - | - | <2000ms |
| NeMo Output Rail | - | - | - | <300ms |
| **Total Guard** | ~11500 | **11937.56** | ~12500 | **<500ms** |

**Budget OK?** [ ] Yes / [x] No  
**Comment:** Vượt quá budget rất nhiều do thời gian load model của Presidio (spaCy) tốn nhiều thời gian khởi tạo trong lần chạy đầu, đồng thời NeMo Guardrails bị delay do latency của LLM API. Cách khắc phục: Khởi tạo Presidio model một lần trước khi đưa vào inference loop, hoặc dùng LLM nhỏ hơn/local cho NeMo.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | 0.7861 |
| Worst metric | faithfulness |
| Dominant failure distribution | factual |
| Cohen's κ | 0.000 |
| Adversarial pass rate | 18 / 20 |
| Guard P95 latency | 11937.56 ms |

---

## Nhận xét & Cải tiến

> - Điều hoạt động tốt: Pipeline NeMo kết hợp Presidio giúp block hiệu quả 18/20 adversarial queries, tăng độ an toàn đáng kể.
> - Điều cần cải thiện: Latency còn quá cao, không đáp ứng được SLA cho ứng dụng thời gian thực.
> - Khi đưa lên Production: Sẽ chuyển sang dùng các API guardrails stream hoặc local lightweight model (như Llama 3 8B cho NeMo) và load Spacy object một lần vào memory để giảm thời gian xử lý xuống dưới 500ms.
