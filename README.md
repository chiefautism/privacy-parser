<div align="center">

# Privacy Parser

**Find every piece of PII in a pile of text and get it back as structured spans.**

Built on [OpenAI Privacy Filter](https://github.com/openai/privacy-filter): same 1.5B model, same labels, same weights.<br>
Privacy Filter masks. Privacy Parser extracts.

![python](https://img.shields.io/badge/python-3.10+-3776AB?logo=python&logoColor=white)
![license](https://img.shields.io/badge/license-Apache--2.0-blue)
![labels](https://img.shields.io/badge/labels-opf%20v2-555)
![F1](https://img.shields.io/badge/hybrid%20F1-0.929-success)

</div>

---

```text
Hi Quindle Testwick (quindle.testwick@example.com / +1-415-555-0102),
account 40702810500001234567, 14 Beautiful Ct, Anytown USA,
password Priv4cy-Filt3r-2026.
```

```text
private_person      Quindle Testwick
private_email       quindle.testwick@example.com
private_phone       +1-415-555-0102
account_number      40702810500001234567
private_address     14 Beautiful Ct, Anytown USA
secret              Priv4cy-Filt3r-2026
```

Instead of `<REDACTED>` you get **what** was found, **where** it is (`start`, `end`) and **which type** it is.

## Why

Privacy Filter answers "make this text safe". Sometimes you need a different answer: "what exactly is in here?"

- **Audit** logs, dumps and exports before they leave your perimeter
- **Inventory** personal data for GDPR / CCPA requests
- **Pre-commit and CI checks** for leaked secrets and contact data
- **Build datasets**: label PII spans for training or evaluation

## Three backends

| Backend           | Needs weights | Speed      | F1 \*  | Use when                         |
| ----------------- | :-----------: | ---------- | :----: | -------------------------------- |
| `PIIParser`       | no            | µs         | 1.000  | zero deps, structured patterns   |
| `ModelPIIParser`  | 1.5B          | ~500 ms CPU | 0.733 | raw model output                 |
| `HybridPIIParser` | 1.5B          | ~600 ms CPU | 0.929 | **default choice**               |

\* exact span match on the five opf sample fixtures. A small set: treat it as a smoke test, not a benchmark.

## Install

```bash
git clone https://github.com/openai/privacy-filter
git clone https://github.com/chiefautism/privacy-parser

uv venv
uv pip install -e ./privacy-filter    # only for the model backends
uv pip install -e ./privacy-parser
```

The first model run downloads the opf checkpoint (~3 GB) to `~/.opf/privacy_filter/`.
Set `OPF_CHECKPOINT` or pass `model=` to use another location.

## Quick start

### Hybrid (recommended)

```python
from pii_parser.hybrid import HybridPIIParser

parser = HybridPIIParser(device="cpu")   # or "cuda"
result = parser.parse("Hi Quindle Testwick, call me at +1-415-555-0102")

for s in result.spans:
    print(s.label, s.start, s.end, s.text)

print(result.to_json())
```

### Regex only, no weights

```python
from pii_parser import parse

result = parse("mail bob@example.com, token ghp_abcdefghijklmnopqrstuvwxyz0123456789")

result.detected_spans   # (DetectedSpan(label='private_email', ...), ...)
result.redacted_text    # 'mail <REDACTED>, token <REDACTED>'
result.summary          # {'span_counts': {...}, 'total_spans': 2}
```

## CLI

Regex backend, no model:

```bash
$ echo "Alice paid 40702810500001234567 on 2026-05-17." | pii-parser --mode spans
account_number  11  31  40702810500001234567
private_date    35  45  2026-05-17

$ pii-parser "mail bob@example.com" --mode redacted
mail <REDACTED>
```

| Flag            | Values                         | Default      |
| --------------- | ------------------------------ | ------------ |
| `TEXT`          | text, or `-f FILE`, or stdin   |              |
| `--mode`        | `json` · `spans` · `redacted`  | `json`       |
| `--placeholder` | any string                     | `<REDACTED>` |
| `--indent`      | int, `0` for compact JSON      | `2`          |

Model backend:

```bash
python -m pii_parser.cli_model "Alice paid 40702810500001234567 on 2026-05-17."
python -m pii_parser.cli_model -f dump.txt --device cuda --mode spans
```

## Labels

opf v2 taxonomy, 8 categories:

| Label             | Example                          |
| ----------------- | -------------------------------- |
| `private_person`  | Quindle Testwick                 |
| `private_email`   | quindle.testwick@example.com     |
| `private_phone`   | +1-415-555-0102                  |
| `private_address` | 14 Beautiful Ct, Anytown USA     |
| `private_url`     | https://example.com/u/quindle    |
| `private_date`    | 2026-05-17                       |
| `account_number`  | 40702810500001234567             |
| `secret`          | `sk-…`, `ghp_…`, `AKIA…`, `Bearer …` |

When spans overlap, the longer one wins, then the higher priority label:
email > url > secret > account > phone > date > address > person.

## How the hybrid works

```text
text
  │
  ▼
opf 1.5B ──► BIOES logits ──► Viterbi (tuned biases) ──► char spans
  │
  ▼
span merge        glues "Quindle" + "Testwick" into one person
  │
  ▼
regex backstop    adds url / secret / account where the model missed them
  │
  ▼
spans[]
```

Each stage can be switched off:

```python
HybridPIIParser(
    enable_merge=True,
    enable_regex_backstop=True,
    enable_viterbi_tuning=True,
    decode_mode="viterbi",   # or "argmax"
)
```

## Output schema

```json
{
  "text": "Hi Quindle Testwick ...",
  "spans": [
    { "label": "private_person", "start": 3, "end": 19, "text": "Quindle Testwick" }
  ]
}
```

The regex `PIIParser` mirrors `opf.RedactionResult` (`schema_version`, `summary`, `detected_spans`, `redacted_text`), so you can swap backends without touching consumers.

## Tests

The tests expect `privacy-filter` cloned next to this repo (they read its sample fixtures).

```bash
python tests/test_hybrid.py
```

```text
Fixture F1:  0.929
Scenarios:   8/8 passed
Latency:     ~600 ms CPU
```

## Responsible use

This is a dual-use tool. Run it on data you own or are authorized to process. Extracting personal data from leaked or stolen sources is illegal in most jurisdictions.

## License

[Apache-2.0](LICENSE)
