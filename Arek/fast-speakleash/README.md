# Arek — fast BPE (korpus SpeakLeash ~3–4 GB)

Byte-level BPE (wariant `fast`, ten sam pre-tokenizer co `../fast/`), trenowany na
**czystym SpeakLeash** (`speakleash-tokenizer-5gb-sample`, shardy streamowane) zamiast
literatury (2,71 M zn.). Cel: pomiar **dźwigni korpusu** na held-oucie SpeakLeash.
Pełna empiria, metodyka i cytaty: **`../README.md`**.

- Alfabet 256 bajtów, zero `<UNK>`, round-trip lossless.
- Pre-tok: `" ?[^\W\d]+| ?\d+| ?[^\s\w]+|\s+"` (GPT-2-style; cyfry jako pełne runy).
- Trening: liczniki inkrementalne + best-pair przez lazy-heap (parytet 1:1 z rdzeniem referencyjnym).
- `min_freq=3`.

## Pliki
| plik | vocab | uwaga |
|---|---|---|
| `32000.json` | 32 000 | format klasy (`model.vocab` + `model.merges`) |
| `64000.json` | 64 000 | trening 4,14 GB |

## Metryki (held-out = shard 0005, ROZŁĄCZNY z treningiem; fertility = tok/słowo, ↓ lepiej)
| vocab | zn/tok | fertility | Rényi (α=2,5) | round-trip |
|---|---|---|---|---|
| 32 000 | 4,03 | 1,765 | 0,451 | ✅ |
| 64 000 | 4,37 | 1,630 | 0,412 | ✅ |

Dźwignia korpusu: nasz stary `fast` 15k (literatura 2,71 M) na tym held-oucie = **2,388** → `32k` = **1,765**
(−0,62 tok/słowo). Krok 3 → 4,14 GB nie ruszył fertility, ale to **za mały krok** by wnioskować o saturacji
(literatura: próg ~150–200 GB — patrz `../README.md`). Więcej danych = wciąż otwarta dźwignia.

**Uczciwie (F2):** to NASZ held-out z tej samej dystrybucji i NASZA definicja słowa (`[^\W\d_]+`).
Liczby pokazują **magnitudę dźwigni korpusu**; porównania absolutne z innym tokenizerem wymagają
**wspólnego** held-outu/protokołu. α w Rényim = konwencja (nie strojona pod wynik — Goodhart).

## Wczytanie
Jak w `../README.md` (`model.vocab` repr → id + `model.merges` w kolejności uczenia).
