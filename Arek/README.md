# Arek — byte-level BPE od zera: naiwny vs szybki

Dwa warianty tego samego algorytmu (**byte-level BPE, czysty Python, bez bibliotek**), trenowane na
**tym samym korpusie** i **tych samych rozmiarach słownika** — żeby porównanie było uczciwe (1:1).

## Dlaczego tak działamy (metoda)
- **Dwa warianty, nie jeden.** `naive` (dydaktyczny) uczy *mechanizmu*; `fast` (produkcyjny) *skaluje*.
  Porównanie 1:1 pokazuje, co realnie daje optymalizacja — bez „wierzenia na słowo".
- **Byte-level (256 bajtów bazowych).** Dowolny tekst UTF-8 kodowalny → **zero `<UNK>`**, round-trip lossless.
- **Ten sam korpus + vocab dla obu.** Apples-to-apples (mierzymy, nie zgadujemy).
- **Żadnej tezy bez dowodu.** Każdą różnicę (szybkość, jakość) potwierdzamy **liczbą**.

## naive vs fast — na czym polega różnica
| | naive (kursowy) | fast (produkcyjny) |
|---|---|---|
| algorytm | recount **wszystkich** par na całym ciągu co merge → `O(n·m)` | pre-tokenizacja (regex GPT-2) + **bucketing** (unikalne słowa × częstość) + **liczniki inkrementalne** |
| pre-tokenizacja | brak (merge przez granice słów) | jest (merge nie przekracza granicy kawałka) |
| realna skala | do ~2k (potem godziny) | do 15k+ (minuty) |

**Zmierzony koszt (ten sam korpus 2,7 M):**
- 3 tokenizery (512/1024/2048): naive **~23 min** vs fast **13,7 s** (≈100×).
- 15k: fast **~3 min** vs naive **~2,5 h — niewykonalne** → dlatego **15k jest tylko fast**.

**Dlaczego pre-tokenizacja (znalezisko z QA):** bez niej naiwny marnuje **~12,4%** slotów słownika na
tokeny **przez granice słów** (`", że "`, `". — "`) i **białe znaki** (wcięcia, `\n\n`) — dokładnie
patologia opisana w paperze GPT-2. Fast tego nie robi (regex tnie tekst *przed* BPE).

## Tokenizery
**naive** (klasyczny, `O(n·m)`):
| plik | vocab | merge | czas treningu |
|---|---|---|---|
| `tokenizer-naive-512.json`  | 512  | 256  | 3m28s  |
| `tokenizer-naive-1024.json` | 1024 | 768  | 5m39s  |
| `tokenizer-naive-2048.json` | 2048 | 1792 | 13m49s |

**fast** (produkcyjny, bucketing + liczniki inkrementalne):
| plik | vocab | merge | czas treningu |
|---|---|---|---|
| `tokenizer-fast-512.json`   | 512   | 256   | ~4 s (z partii 13,7 s) |
| `tokenizer-fast-1024.json`  | 1024  | 768   | ~5 s |
| `tokenizer-fast-2048.json`  | 2048  | 1792  | ~5 s |
| `tokenizer-fast-15000.json` | 15000 | 14744 | ~3m10s |

*(Korpus 2,7 M utrzymał pełne 14 744 merge przy 15k — nie trafiliśmy w sufit „para < 2 wystąpień".
Czy 15k się **opłaca**, rozstrzygnie Rényi — patrz Plany.)*

## Dane treningowe
Korpus **2 705 758 znaków** polskiej literatury (public domain, Wolne Lektury): Prus *Lalka* (t. I+II),
Sienkiewicz *Ogniem i mieczem*, Shakespeare *Romeo i Julia* (tłum. Paszkowski), Wyspiański *Wesele*.
Rejestr literacki XIX / wcz. XX w., **22,5%** znaków to polskie diakrytyki. Held-out / ewaluacja: *Pan Tadeusz*.

## Format
```json
{ "meta": {...}, "model": { "type": "BPE", "vocab": {"repr": id}, "merges": ["lewy prawy", ...] } }
```
- reprezentacja **`bytes_to_unicode`** (GPT-2: spacja → `Ġ`, bajt 0 → `Ā`); `merges` w kolejności uczenia (rank).
- `fast` zapisuje dodatkowo `regex_pretok` w `meta` (bez niego nie odtworzysz jego kodowania).
- To blok **`model`** tokenizera. Pełny HF `tokenizer.json` (z `pre_tokenizer` + `decoder`, ładowalny
  `Tokenizer.from_file`) — planowany dla `fast` (naiwny bez pre-tok nie jest wiernym `ByteLevel`).

## Plany
- **Rényi efficiency + fertility + gęstość** na *Panu Tadeuszu* dla całej siódemki + baseline **GPT-2** →
  **wykres kolana** (który vocab się opłaca; czy 15k to już przegięcie = zbyt wiele rzadkich tokenów).
- **Pełny HF `tokenizer.json`** dla `fast` (regex zgodny z `ByteLevel` + `decoder` → `from_file`).
- **Unigram LM** od zera (EM + Viterbi) jako trzeci wariant do porównania.
- większy vocab / większy korpus (`fast` skaluje).

## Reprodukcja
`bpe_skeleton.py` (train / encode / decode, naive) · `train_bpe_fast.py` (fast) ·
`save_tokenizer.py` / `save_fast.py` (serializacja) — wszystko byte-level BPE od zera, czysty Python.

*Autor: Arek · Kurs Tokenizer, Slayer Labs.*
