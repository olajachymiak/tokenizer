# Arek — byte-level BPE tokenizery (polski)

Byte-level BPE trenowany **od zera w czystym Pythonie** (bez bibliotek), na kilku korpusach i schematach
pre-tokenizacji — do **kontrolowanego, empirycznego** porównania. Wszystkie: alfabet bazowy 256 bajtów,
**zero `<UNK>`, round-trip lossless**.

> **Najlepszy (benchmark SpeakLeash, held-out shard 0005):**
> **`fast-speakleash-gpt4/64000` — fertility 1,619 · Rényi 0,431.** GPT-4 pre-tok + pełne runy cyfr.

## Rodziny wariantów

| katalog | korpus | pre-tok | vocab | rola |
|---|---|---|---|---|
| `naive/` | Wolne Lektury (2,71 M zn.) | brak | 512–2048 | dydaktyczny, `O(n·m)` |
| `fast/` | Wolne Lektury (2,71 M zn.) | GPT-2 regex | 512–15000 | produkcyjny (heap) |
| `fast-speakleash/` | **SpeakLeash ~4,14 GB** | GPT-2 regex | 32k, 64k | produkcja / skala |
| `fast-speakleash-gpt4/` | SpeakLeash ~3–4 GB | **GPT-4 (cl100k) regex** | 32k, **64k** | pre-tok — **64k = najlepszy** |

`naive` vs `fast`: ta sama jakość, `fast` skaluje (regex pre-tok + bucketing + liczniki inkrementalne +
best-pair przez lazy-heap; parytet 1:1 z rdzeniem referencyjnym). Osobno zmierzyliśmy **skalę danych** na
korpusie `SlayerLab/polish-dynaword` (do 17 GB) — patrz „Skala danych i dystrybucja".

## Metryki (i po co)

- **zn/tok** (chars/token) — kompresja, ↑ lepiej.
- **fertility** (tokeny/słowo) — ↓ lepiej. Standard w ewaluacji multilingual.
- **Rényi efficiency** (α=2,5) — równomierność rozkładu tokenów, 0–1, ↑ lepiej. Informacyjno-teoretyczny
  proxy jakości z „noiseless channel" (Zouhar i in. 2023, ACL; ~0,78 korelacji z BLEU w MT).
- **round-trip lossless** — `decode(encode(x)) == x`.

> **Uwaga metodyczna (kluczowa):** fertility i Rényi są **dystrybucyjne** — mierzą kompresję i rozkład,
> **nie** gramatyczność/morfologię. BPE optymalizuje kompresję, nie spójność lingwistyczną — dlatego
> powstały osobne metryki morfologiczne (MorphScore, MorphAcc). Patrz „Fleksja pod mikroskopem".

## Tabela 1 — mały korpus (held-out: Mickiewicz „Pan Tadeusz", 445 637 zn.)

| wariant | vocab | zn/tok | fertility | Rényi |
|---|---|---|---|---|
| naive-512 | 512 | 1,76 | 3,718 | 0,768 |
| naive-2048 | 2048 | 2,43 | 2,685 | 0,743 |
| fast-2048 | 2048 | 2,41 | 2,712 | 0,585 |
| fast-8192 | 8192 | 3,00 | 2,177 | 0,460 |
| fast-15000 | 15000 | 3,25 | 2,010 | 0,418 |
| GPT-2 (ref.) | 50257 | 1,73 | 3,782 | 0,391 |

## Tabela 2 — produkcja SpeakLeash (held-out: shard 0005, ROZŁĄCZNY z treningiem; def. słowa `[^\W\d_]+`)

| wariant | vocab | pre-tok | zn/tok | fertility | Rényi | RT |
|---|---|---|---|---|---|---|
| fast-speakleash | 32000 | GPT-2 | 4,03 | 1,765 | 0,451 | ✅ |
| fast-speakleash | 64000 | GPT-2 | 4,37 | 1,630 | 0,412 | ✅ |
| fast-speakleash-gpt4 | 32000 | GPT-4 (cyfry ≤3) | 4,04 | 1,763 | **0,473** | ✅ |
| **fast-speakleash-gpt4** | **64000** | **GPT-4 + pełne cyfry** | **4,39** | **1,619** | **0,431** | ✅ |

**Najlepszy na benchmarku:** `gpt4/64000` (1,619). Zysk vs `fast-speakleash/64000` (GPT-2, 1,630) pochodzi
**wyłącznie ze zmiany pre-toka** — ten sam korpus, ten sam held-out.

> **Nie porównuj Tabeli 1 z Tabelą 2** — inne held-outy → liczby nieporównywalne wprost. Absolutne liczby
> są nieważne bez **wspólnego** held-outu i wspólnej definicji słowa.

## Co zmierzyliśmy i dlaczego

1. **Vocab ↑ → fertility ↓ (z saturacją).** 512→15k: zn/tok 1,78→3,25 (Tab. 1). Przy dużym vocab zyski maleją.
2. **Pre-tok GPT-4 + pełne cyfry — najlepszy pipeline.** @32k był **neutralny** na fertility (1,765→1,763),
   Rényi ↑. **@64k + pełne runy cyfr (`\p{N}+`) WYGRYWA obie osie: 1,630→1,619, Rényi 0,412→0,431.** Większy
   vocab daje miejsce, by lepsze granice kategorii/cyfr realnie skróciły zapis.
3. **Skala danych: nasycona JUŻ przy ~3 GB (przy stałym vocab).** Zmierzone na dynaword (2,84→17 GB, 6×):
   fertility **płaski**. Więcej danych **nie** obniża fertility przy fixed vocab — ogranicza je **rozmiar
   słownika**, nie wolumen. Dźwignie: **vocab + pre-tok**, nie ilość danych. *(„Diminishing Returns…"
   arXiv 2502.20273 mierzy jakość tokenizera vs rozmiar danych przy **zmiennym** vocab (EN ~150 GB, RU ~200 GB;
   saturacja przez pre-tokenizację); nasz setup ze **stałym vocab 64k** nasyca się dużo wcześniej (~3 GB).)* Szczegóły niżej.
4. **Dystrybucja dominuje nad ilością** (niżej, 2×2): dopasowanie train↔test > wolumen danych.
5. **fertility/Rényi są ślepe na morfologię** (nasz pomiar + literatura): formy fleksyjne jednego lematu
   wchodzą jako **osobne atomy**, wspólny morfem (`-ość`) nie ma wspólnego tokenu. Patrz „Fleksja pod mikroskopem".
6. **BPE vs Unigram (SentencePiece, matched vocab):** u nas BPE dał **nieco niższą fertility** (BPE jest
   zachłannie kompresyjny — Bostrom & Durrett 2020). Przewaga Unigrama leży w downstream/morfologii, nie w kompresji.
7. **Kierunek morfem-aware:** podnosi trafienie w morfemy, ale zysk maleje przy dużym vocab, a dopasowanie
   morfologiczne **słabo koreluje z wydajnością modelu** (arXiv 2507.06378).

## Skala danych i dystrybucja — eksperyment dynaword (zmierzony)

Retrain 64k (najlepszy pipeline: GPT-4 + pełne cyfry) na `SlayerLab/polish-dynaword` (human-text, 12 źródeł,
CC-BY-SA). Zmienne rozłożone: held-out dynaword rozłączny; **control** = jednorodna próbka **2,84 GB**;
**full** = **17,0 GB**. Każdy mierzony na OBU held-outach.

| trening (64k) | fert @SpeakLeash-0005 | fert @dyna-held | gap (home↔away) |
|---|---|---|---|
| **SpeakLeash 4,14 GB** | **1,619** | 1,829 | 0,210 |
| dynaword 2,84 GB | 1,758 | 1,726 | — |
| dynaword 17,0 GB | 1,757 | 1,733 | 0,024 |

- **Rozmiar: nasycony do ~3 GB.** dynaword 2,84→17 GB (6×): fertility płaski (SL 1,758→1,757; dyna-held
  1,726→1,733). *(Brak punktów 0–2,84 GB → „nasycony JUŻ przy ~3 GB", nie „próg = 3 GB".)*
- **Dystrybucja dominuje:** SpeakLeash-trained (4,14 GB) bije dynaword-trained (17 GB) na SpeakLeash — mimo 4× mniej danych.
- **Robustność:** SpeakLeash-trained jest **home-turf** (gap 0,21 — pada na tekście ludzkim); dynaword-trained
  jest **robust** (gap 0,024 — trzyma na obu). Publikujemy wariant SpeakLeash jako najlepszy **na benchmarku**;
  dynaword-trained bywa lepszy jako **ogólny** (zależnie od targetu deploymentu).

## Fleksja pod mikroskopem — tokeny vs metryka (dowód punktu 5)

Realny podział na tokeny naszym `fast-speakleash/64000` (zweryfikowany 1:1 z encoderem; słowa w izolacji).

**(a) Jeden lemat, różne przypadki → różny podział, brak wspólnego rdzenia:**

| forma (przypadek) | tokeny (64k) | # |
|---|---|---|
| odpowiedzialność (M) | `odpowie · dzialność` | 2 |
| odpowiedzialności (D) | `odpowie · dzia · lności` | 3 |
| odpowiedzialnością (N) | `odpowie · dzia · lnością` | 3 |

**(b) Wspólny morfem `-ość` w różnych lematach → różne, niezwiązane fragmenty:**

| słowo | tokeny (64k) | `-ość` ląduje jako |
|---|---|---|
| wolność | `wo · lność` | `lność` |
| radość | `ra · dość` | `dość` |
| ciekawość | `cieka · wość` | `wość` |
| odpowiedzialność | `odpowie · dzialność` | `dzialność` |

Ten sam morfem `-ość` **nie ma jednego tokenu** — rozłazi się na `lność / dość / wość / dzialność`.
Zmierzone (UniMorph PL, N=86 k clean-suffixal; recall-only, within-tokenizer): tokenizer trafia w granicę
`stem|końcówka` tylko ~**19%** przypadków. **Metryka (fertility ~1,62) mówi „dobrze skompresowane", a
paradygmat fleksyjny jest rozbity** — fertility/Rényi tego nie widzą. To osobna oś jakości.

## Uczciwe zastrzeżenia

- **Held-out i definicja słowa są nasze** (`[^\W\d_]+`, ta sama dystrybucja). Liczby pokazują **magnitudy i
  tendencje**; czyste 1:1 z innym tokenizerem wymaga **wspólnego** held-outu/protokołu.
- **α w Rényim = konwencja** (nie strojona pod wynik — to byłby Goodhart).
- Metryki tu są **dystrybucyjne**; „najlepszy" znaczy **na benchmarku SpeakLeash** — na innej dystrybucji
  ranking się zmienia (patrz 2×2). Jakość morfologiczna i downstream to osobne osie.

## Format i wczytywanie

Każdy plik to JSON w formacie zbliżonym do HuggingFace: blok `model` z `vocab` (`bytes_to_unicode` → id) i
`merges` (lista `"lewy prawy"` w kolejności uczenia). **Pre-tok:** rodziny `fast*` używają regexu GPT-2;
`fast-speakleash-gpt4/*` używają regexu GPT-4 (cl100k) — `64000` z pełnymi runami cyfr (`\p{N}+`).

```python
import json
m = json.load(open("fast-speakleash-gpt4/64000.json", encoding="utf-8"))["model"]
vocab = m["vocab"]                       # repr -> id
merges = {}
for line in m["merges"]:                 # kolejność = rank
    l, r = line.split(" ")
    merges[(vocab[l], vocab[r])] = vocab[l + r]
```

## Literatura

- Zouhar i in. 2023 — *Tokenization and the Noiseless Channel*, ACL (Rényi efficiency).
- Bostrom & Durrett 2020 — *BPE is Suboptimal for Language Model Pretraining* (BPE vs Unigram).
- *Evaluating Morphological Alignment of Tokenizers in 70 Languages*, arXiv 2507.06378 (MorphScore; alignment ≠ downstream).
- *Diminishing Returns of Tokenization Training Data*, arXiv 2502.20273 · MorphBPE, arXiv 2502.00894.
- Petrov i in. 2023 — *Language Model Tokenizers Introduce Unfairness Between Languages*, NeurIPS.
- Korpus skali: [`SlayerLab/polish-dynaword`](https://huggingface.co/datasets/SlayerLab/polish-dynaword) (CC-BY-SA-4.0).

## Autor
Arek — Kurs Tokenizer, Slayer Labs.
