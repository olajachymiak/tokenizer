# Arek — byte-level BPE (naiwny, kursowy)

Trzy tokenizery **byte-level BPE napisane od zera** (czysty Python, bez bibliotek), różniące się
rozmiarem słownika. Wariant **naiwny** (dydaktyczny): `bajty → najczęstsza sąsiednia para → merge → powtórz`,
**bez pre-tokenizacji** (merge może przekraczać granice słów — świadomy, prosty wariant do zrozumienia mechanizmu).

## Tokenizery
| plik | vocab | merge | czas treningu (CPU, czysty Python) |
|---|---|---|---|
| `tokenizer-vocab-512.json`  | 512  | 256  | 3m28s  |
| `tokenizer-vocab-1024.json` | 1024 | 768  | 5m39s  |
| `tokenizer-vocab-2048.json` | 2048 | 1792 | 13m49s |

Rosnący czas (3,5 → 14 min) to **empiryczny koszt naiwnego BPE — `O(n·m)`**: recount wszystkich par
na całym ciągu co scalenie. Pokazuje, **dlaczego produkcyjne tokenizery używają bucketingu**
(liczenie na unikalnych słowach × częstość) — wariant `fast` w przygotowaniu.

## Dane treningowe
Korpus **2 705 758 znaków** polskiej literatury (public domain, Wolne Lektury):
Prus *Lalka* (t. I+II), Sienkiewicz *Ogniem i mieczem*, Shakespeare *Romeo i Julia* (tłum. Paszkowski),
Wyspiański *Wesele*. Rejestr literacki XIX / wcz. XX w., **22,5%** znaków to polskie diakrytyki.
Held-out / ewaluacja: *Pan Tadeusz*.

## Format
```json
{ "meta": {...}, "model": { "type": "BPE", "vocab": {"repr": id}, "merges": ["lewy prawy", ...] } }
```
- `vocab`: reprezentacja tokenu → ID, w mapowaniu **`bytes_to_unicode`** (GPT-2: spacja → `Ġ`, bajt 0 → `Ā`).
- `merges`: reguły scalania **w kolejności uczenia** (rank).
- **256 tokenów bazowych (bajty) + N merge → pełne pokrycie UTF-8, zero `<UNK>`, round-trip lossless.**

To jest blok **`model`** pełnego tokenizera. **Uczciwie:** wariant naiwny nie ma pre-tokenizacji,
więc nie jest wiernym HF `ByteLevel` — pełny HF `tokenizer.json` (z `pre_tokenizer` + `decoder`)
powstanie dla wariantu `fast`.

## Reprodukcja
Kod: `bpe_skeleton.py` (train / encode / decode) + `save_tokenizer.py` (serializacja) — byte-level BPE od zera.

## Dalej
- pomiar **fertility / gęstość / Rényi efficiency** na *Panu Tadeuszu* (wykres kolana → dobór vocab),
- wariant `fast` (pre-tokenizacja + bucketing + liczniki inkrementalne) — skaluje do milionów znaków,
- większy słownik.

*Autor: Arek · Kurs Tokenizer, Slayer Labs.*
