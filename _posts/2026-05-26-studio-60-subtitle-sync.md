---
layout: post
title: "Синхронизация субтитров: как мы починили Studio 60"
date: 2026-05-26 21:30:00 +0300
categories: [media, subtitles]
tags: [ffsubsync, alass, opensubtitles, plex, jekyll, vad]
---

История одной задачи и разбор инструментов: **ffsubsync**, **alass**, OpenSubtitles API, а также внутренних алгоритмов (FFT кросс-корреляция vs. динамическое программирование).

<!--more-->

## Исходная задача

22 эпизода сериала **Studio 60 on the Sunset Strip** (один сезон, 2006-2007, NBC) в формате 720p HDTV. Рядом лежали русские субтитры в кодировке **windows-1251**, рассинхронизированные с видео. Нужно было:

1. Добавить английские субтитры.
2. Починить тайминг на английских и русских.
3. Привести всё к UTF-8.
4. Чтобы Plex корректно различал языки.

---

## Использованные инструменты

### 1. OpenSubtitles.com REST API (v1)

Источник английских субтитров. Важный момент: искать по **parent_imdb_id** (485842 для Studio 60), а не по текстовому названию — текстовый поиск выдал «Oats Studios» вместо нашего сериала.

- Анонимный API-ключ даёт 100 загрузок в сутки.
- Предпочитали официальные релизы Warner Bros (`DVD.NonHI.en.WB`), а не фанатские (`HDTV.XviD-MiNT` и т.п.) — они чище по тексту и реже содержат опечатки.
- Авторизация через заголовок `Api-Key: <KEY>`, никакого OAuth-токена не нужно.

### 2. ffsubsync (smacke/ffsubsync, v0.4.31, ноябрь 2025)

Активно поддерживается, является «дефолтом» в индустрии. Принцип: извлекает аудио из видео, делает VAD (детекцию речи), потом через FFT находит **один глобальный сдвиг** и **один коэффициент framerate**. Для случая «PAL 25fps vs NTSC 23.976fps» работает идеально.

Поддерживает разные VAD: `webrtcvad`, `auditok`, `silero` (последний требует PyTorch).

**Слабость**: только одно линейное преобразование на весь файл. Если в субтитрах рекламные паузы вырезаны в одних местах, а в видео в других — ffsubsync усреднит и оставит дрейф. Автор сам пишет в README: *«Handling breaks and splits in the middle of video... is left to future work»* — [open issue #31](https://github.com/smacke/ffsubsync/issues/31) с 2019 года.

### 3. alass (kaegi/alass, v2.0.0, 2019, не обновляется)

К нему перешли, когда ffsubsync дал «лучше, но всё ещё рассинхрон». Алгоритмически alass **детектирует точки разрыва** и применяет разные смещения к разным сегментам файла. На S01E03 он нашёл **4 сегмента** со сдвигами −14.26s → −17.40s → −20.79s → −26.37s — типичный дрейф от вырезанных реклам. ffsubsync такое не умеет.

**Слабость**: проект заброшен, но алгоритм работает и в 2026 — бинарь под Linux v2.0.0 запускается без проблем.

**Правило выбора**: ffsubsync — дефолт. alass — когда ffsubsync не справился и виден неравномерный дрейф (особенно для эфирного контента с рекламными паузами).

### 4. ffmpeg / ffprobe

Используются обоими инструментами под капотом для извлечения аудио. ffprobe пригодился отдельно — проверить framerate видео (`r_frame_rate=24000/1001` = NTSC 23.976) и убедиться, что в .mkv нет встроенных субтитров.

### 5. iconv

Для конвертации windows-1251 → UTF-8. Хотя в итоге выяснилось, что alass пишет вывод в UTF-8 независимо от кодировки входа, так что шаг оказался лишним.

### 6. uv / uvx

Менеджер Python-пакетов от Astral. Установили ffsubsync, потом torch+torchaudio (для silero VAD) через `uv tool install --with torch --with torchaudio ffsubsync`.

Замечание: на машине был сломанный pyright из старого pipx — указывал на удалённый `~/miniforge3/bin/python3.10`. Починили через `uv tool install pyright`.

---

## Подводный камень, которого не было видно сразу

Официальные субтитры Warner Bros с DVD устроены так: **каждая двухстрочная реплика разбита на два SRT-блока с одинаковыми тайм-кодами**:

```srt
3
00:00:03,804 --> 00:00:06,291
You're one of the highest-ranking

4
00:00:03,804 --> 00:00:06,291
female executives...
```

Многие плееры (включая Plex в некоторых клиентах) рендерят оба блока одновременно — получается визуальное наслоение «конец фразы поверх начала». Это **проблема исходника**, не синхронизатора.

Решение — Python-скрипт, который сливает соседние блоки с идентичными тайм-кодами в один многострочный блок:

```python
def merge_pairs(entries):
    merged = []
    for start, end, body in entries:
        if merged and merged[-1][0] == start and merged[-1][1] == end:
            ps, pe, pb = merged[-1]
            merged[-1] = (ps, pe, pb + "\n" + body)
        else:
            merged.append((start, end, body))
    return merged
```

На 22 файлах это убрало 500-650 «дубликатов» на эпизод.

---

## Итоговый пайплайн

1. Поиск субтитров через OpenSubtitles API по `parent_imdb_id` + сезон + серия + язык.
2. Скачивание (POST `/download` с `file_id`).
3. Синхронизация: сначала ffsubsync; если результат «почти», но не точно — alass с `--speed-optimization 0 --interval 1` (максимальная точность).
4. Постобработка: слияние блоков с дублированными тайм-кодами (для DVD-источников).
5. Соглашение об именовании для Plex: `<имя_видео>.en.srt`, `<имя_видео>.ru.srt` (двухбуквенный ISO 639-1 — Plex автоматически распознаёт язык).
6. Оригиналы в `_backup/` (Plex не сканирует поддиректории на sidecar-субтитры).

---

## Алгоритмы внутри

### ffsubsync — FFT-кросс-корреляция

**Шаг 1: дискретизация в бинарные последовательности.**
Аудиодорожку видео разбивает на окна по 10 мс. Для каждого окна VAD (Voice Activity Detection) выдаёт **0** (тишина/музыка) или **1** (речь). Получается бинарная строка длиной N (для 42-минутного эпизода — ~252 000 бит).

То же самое со субтитрами: на сетке 10 мс ставит **1** там, где по тайм-кодам должен быть текст, и **0** где его нет.

**Шаг 2: кросс-корреляция через FFT.**
Чтобы найти оптимальный сдвиг между двумя последовательностями `a` (видео) и `b` (субтитры), нужно вычислить:

```
corr(τ) = Σ a[i] · b[i + τ]   для всех τ от -max_offset до +max_offset
```

Прямым перебором это O(N²) — миллиарды операций. Через FFT: `corr = IFFT(FFT(a) · conj(FFT(b)))` — O(N log N). На стандартном эпизоде ~50 миллионов операций, секунды CPU.

**Пик функции корреляции = оптимальный сдвиг.** Это и есть «offset seconds: -8.250».

**Шаг 3: коэффициент framerate (опционально).**
Перебирает несколько разумных коэффициентов (1.0, 23.976/25, 25/23.976, 24/23.976 и т.д.), для каждого пересчитывает субтитры и ищет лучшую кросс-корреляцию. С флагом `--gss` использует **golden-section search** — численный метод поиска экстремума унимодальной функции, который за log₁.₆₁₈(N) итераций сходится к оптимуму без перебора.

**VAD-варианты:**

- `webrtcvad` (дефолт) — Google WebRTC, использует **GMM** (Gaussian Mixture Model) обученную на телефонной речи. Быстро, неплохо.
- `auditok` — энергетический детектор: RMS-энергия в окне выше порога = речь. Чувствителен к фоновой музыке (часто видит её как речь).
- `silero` — **нейросеть** (LSTM поверх MFCC-фичей, ~1 МБ весов от компании Silero). Сильно точнее, но требует PyTorch и cold-start ~3 сек.

**Что ffsubsync принципиально не умеет:** найти оптимум — это найти **один** τ, который максимизирует корреляцию. По построению алгоритма он применяется ко всему файлу. Чтобы получить разные τ для разных кусков, нужен другой алгоритм.

### alass — динамическое программирование с штрафом за разрывы

**Шаг 1: «rated intervals».**
Видео → бинарная VAD-последовательность (как у ffsubsync, но интервал по умолчанию **1 мс**, не 10 мс). Субтитры → последовательность интервалов «есть текст / нет».

**Шаг 2: задача оптимизации.**
Пусть субтитры состоят из N реплик. Для каждой реплики `i` нужно выбрать сдвиг `δᵢ`. Оптимальное решение максимизирует:

```
J = Σᵢ score(reply_i, δᵢ) − P · (число точек разрыва)
```

где `score` — насколько хорошо сдвинутая реплика попадает на речь в видео (мера перекрытия с VAD-маской), а **P** — это `--split-penalty` (дефолт 7). «Точка разрыва» — место, где `δᵢ ≠ δᵢ₊₁`.

**Шаг 3: динамическое программирование.**
Решается **снизу вверх** по таблице `DP[i][δ]` = «лучший суммарный score для первых i реплик, если последняя сдвинута на δ». Рекуррентность:

```
DP[i][δ] = score(i, δ) + max over δ' of (DP[i-1][δ'] − P · [δ ≠ δ'])
```

Это классический алгоритм с памятью O(N · D), где D — число возможных сдвигов (D = `max_offset / interval`). При `--interval 1ms` и `max_offset` в пару минут, D ≈ 120 000. N для 42-минутного эпизода — ~1300 реплик. Итого ~150M клеток таблицы. С `--speed-optimization 1` (дефолт) пространство сжимается; с `--speed-optimization 0` (что мы использовали) — точный поиск, медленнее, но без потери точности.

**Шаг 4: восстановление сегментов.**
После заполнения таблицы — backtrace через `argmax`, получаются точки, где оптимальное `δ` меняется. Это и есть «shifted block of 435 subtitles by -14.263s; shifted block of 249 subtitles by -17.400s...» — каждый блок это сегмент между точками разрыва.

**Зачем `--split-penalty`:**

- При P → ∞ алгоритм вырождается в один сегмент (поведение как у ffsubsync, только один глобальный сдвиг).
- При P → 0 алгоритм разрешает разный сдвиг для каждой реплики — переобучение, реплики «прилипают» к ближайшей речи без логики.
- Дефолт 7 — практический компромисс. На S01E03 нашли 4 сегмента (типично для эпизода с 3-4 рекламными паузами); на S01E07 — 1 сегмент (видимо, реклама была вырезана в тех же местах в субтитрах и в видео).

**Дополнительно:**

- `--disable-fps-guessing` — выключает встроенный поиск коэффициента framerate. По умолчанию alass перебирает 24/23.976, 23.976/25 и пару других.
- VAD внутри alass — собственный энергетический детектор на основе **STFT** (short-time Fourier transform), без нейросетей.

### Разница в сложности

|                              | ffsubsync                | alass                                              |
|------------------------------|--------------------------|----------------------------------------------------|
| **Задача**                   | argmax по 1D             | argmax по последовательности с регуляризацией      |
| **Метод**                    | FFT кросс-корреляция     | Dynamic programming                                |
| **Параметров на выход**      | 2 (offset + scale)       | 2N (по сдвигу на каждую реплику)                   |
| **Сложность**                | O(N log N)               | O(N · D)                                           |
| **Время на эпизод (42 мин)** | 10–30 сек                | 10–60 сек                                          |

### Почему силеро VAD вообще обсуждается

Качество синхронизации напрямую упирается в качество VAD. Если VAD ловит фоновую музыку (например, мьюзикл-вставка Studio 60 как раз начинается с песни «I Am the Very Model of a Modern Network TV Show») как «речь», а в субтитрах в этом месте тишина — кросс-корреляция получит ложный пик. silero обучен отличать речь от музыки и фонового шума, что для драматических сериалов с саундтреком даёт заметно более чистый сигнал. Для нашего кейса не пригодился — alass сам справился — но для случаев типа «синхронизация субтитров к концертному видео» силеро критичен.

Если интересна академическая сторона — репозиторий **kaegi/alass** объясняет DP-рекуррентность подробнее, а ffsubsync ссылается на классику Lewis (1995) *Fast Normalized Cross-Correlation* для своей FFT-части.

---

## Промпт для Claude Code (на английском)

```text
Sync subtitles for the video files in this directory using the best-quality
pipeline:

1. Identify the show via filename. Resolve its IMDB parent_id and find video
   files (mkv/mp4/avi) that need subtitles.

2. For each episode that lacks a subtitle in the requested language, fetch one
   from the OpenSubtitles.com REST API (https://api.opensubtitles.com/api/v1).
   Auth: header "Api-Key: <KEY>". Search /subtitles with parent_imdb_id,
   season_number, episode_number, languages=<lang>. Prefer official releases
   (e.g., "DVD.NonHI.<lang>.WB") over fan rips; fall back to the non-HI sub
   with highest download_count. Download via POST /download with file_id
   (anonymous, 100/day quota). Save as <video_base>.<lang>.unsynced.srt.

3. Detect encoding with chardet or `file`; if not UTF-8, transcode to UTF-8.

4. Sync with ffsubsync first (single-offset model, actively maintained):
       ffsubsync <video.mkv> -i <sub.unsynced.srt> -o <out.srt> --gss
   If you suspect commercial-break drift (typical for HDTV/NBC airings of older
   shows) OR the user reports the result is "better but still off", re-run with
   alass (split-aware, downloadable binary from
   github.com/kaegi/alass/releases/latest):
       alass <video.mkv> <sub.unsynced.srt> <out.srt> \
             --speed-optimization 0 --interval 1
   alass reports "shifted block of N subtitles by Xs" per segment. Multiple
   segments mean it found split-points ffsubsync would have averaged out.

5. Post-process the output:
   - If the source SRT splits multi-line dialogue into separate entries with
     IDENTICAL timestamps (common for Warner Bros DVD subs), merge consecutive
     entries that share start/end timestamps into one multi-line block.
     Otherwise Plex/VLC may stack them visually.
   - Re-number entries 1..N.
   - Ensure UTF-8 output and \n line endings.

6. Name files for Plex auto-language detection: <video_base>.<iso639-1>.srt
   (e.g., .en.srt, .ru.srt). Stash original sources in a _backup/ subdirectory
   - Plex does not scan subdirectories for sidecar subtitles, so backups won't
   show up as phantom tracks.

7. After processing, each video should have exactly one synced .srt per
   language alongside it - no .unsynced.srt, .tmp, or duplicate-suffix files
   left behind, since Plex would surface them as additional tracks.

8. Verify by reading (not just parsing) a sample of the output: check first
   entry starts at real dialogue time, scan for adjacent entries with
   identical timestamps, confirm encoding renders correctly.

Tools to install if missing: uv tool install ffsubsync (add --with torch
--with torchaudio for silero VAD); download alass-linux64 from its GitHub
releases page and chmod +x. Use ffprobe to confirm video framerate and audio
language streams before syncing.
```

---

## Главный урок

Один инструмент решает 90% случаев, но для оставшихся 10% нужен правильный второй. И всегда читать выхлоп самому — даже когда оба синхронизатора отчитались «успешно», может оказаться, что битый исходник.
