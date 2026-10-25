<div align="center">

# 📸 → 🗂️ photo-to-obsidian

**A Claude skill that turns phone photos of slides and screens into searchable Obsidian notes.**
**Offline, free, Windows 7 / 10 / 11.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
![Platform](https://img.shields.io/badge/platform-Windows%207%20%7C%2010%20%7C%2011-blue)
![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![Claude Skill](https://img.shields.io/badge/Claude-Skill-orange)
![OCR](https://img.shields.io/badge/OCR-Umi--OCR%20%2B%20Tesseract-lightgrey)

**English** · [Русский](#-русский)

<img src="docs/cover.png" alt="Photos of slides to Obsidian notes" width="820">

</div>

---

## Why

You came back from a conference with hundreds of photos of slides taken straight off a monitor. They sit in your gallery and nobody reads them.

This skill walks Claude (and you) through a tested pipeline that reads the text from every photo **locally** and saves each one as an Obsidian note: the image on top, the recognized text below. Optionally, a local LLM fixes OCR typos.

## Features

- **One photo = one note** with frontmatter, an embedded copy of the image and the recognized text
- **Fully offline**: OCR runs in [Umi-OCR](https://github.com/hiroi-sora/Umi-OCR) with the Tesseract engine, typo fixing runs in [Ollama](https://ollama.com)
- **Built for photos of screens**: anti-moiré preprocessing and two-pass recognition give **+50% correctly recognized words** compared to a single Tesseract pass
- **Safe**: original photos are never modified or moved, existing notes are skipped, runs can be interrupted and resumed
- **Russian + English** out of the box; any Tesseract language can be added
- **Beginner-friendly**: Claude asks for your folders, gives exact commands, runs a 2-photo test first and explains every step

<div align="center">
<img src="docs/before_after.png" alt="Before and after preprocessing" width="820">
<br><sub>A test slide photographed with moiré, glare and tilt. Raw Tesseract output in the middle, this pipeline on the right. Unedited.</sub>
</div>

## How it works

```
Photo ──► downscale + blur + CLAHE ──► Umi-OCR (Tesseract, rus+eng) ──┐
      └─► downscale + blur + autocontrast ──► Umi-OCR ────────────────┤
                                                                      ▼
                                          merge lines, drop duplicates & noise
                                                                      ▼
                                   Obsidian note  ──►  (optional) Qwen via Ollama fixes typos
```

Photos of a monitor suffer from **moiré**: the pixel grids of the screen and the camera interfere and draw rainbow stripes over the letters. The surprising finding: **smaller is better**. At full resolution (4000+ px) Tesseract sees the moiré as texture and can hang for tens of minutes; at 1600 px with a slight blur it reads the text.

### Benchmark

Unique dictionary words (Russian + English Hunspell) recognized on 12 real photos of slides:

| Pipeline | Words |
|---|---:|
| Tesseract, single pass (blur + grayscale, 2000 px) | 252 |
| + local contrast (CLAHE, 1600 px) | 332 |
| **Two passes + merge (default)** | **379 (+50%)** |

For reference, the built-in PaddleOCR Russian model scored a self-reported confidence of 0.37–0.46 on the same photos (0.74 at best after tuning); Tesseract scored about 0.88.

## Requirements

| | Windows 10 / 11 | Windows 7 (64-bit) |
|---|---|---|
| Umi-OCR + Tesseract plugin | ✅ | ✅ |
| Python | 3.12 | 3.8.10 |
| Ollama + Qwen (typo fixing) | ✅ (10 22H2+) | ❌ |

RAM: 8 GB is enough (use `qwen3:4b`); with 16 GB use `qwen3:8b`.

## Installation

### Claude (web / desktop app)

1. Download `photo-to-obsidian.zip` from [Releases](../../releases/latest).
2. Open **Settings → Customize → Skills**, upload the zip (do not unpack it) and enable the skill. Code execution must be enabled.
3. Start a chat: *"I want to turn my slide photos into Obsidian notes"*.

### Claude Code

```bash
git clone https://github.com/<your-github-username>/photo-to-obsidian.git
```

Copy the `photo-to-obsidian/` folder into `%USERPROFILE%\.claude\skills\` (Windows) or `~/.claude/skills/` (macOS / Linux).

### Without Claude

The scripts work on their own. Follow the steps in [`photo-to-obsidian/SKILL.md`](photo-to-obsidian/SKILL.md): install Umi-OCR, the Tesseract plugin and the Russian model, then run from `photo-to-obsidian/scripts/` in `cmd`:

```bat
pip install requests pillow opencv-python numpy

# test run: 2 photos per folder
python umi_obsidian.py --vault "D:\Obsidian\Vault" --out "OCR/Conference" ^
  --attachments "OCR/Conference/attachments" ^
  --source "D:\Photos\Day_1" --source "D:\Photos\Day_2" --limit 2 --force

# full run: same command without --limit 2 --force
```

Optional typo fixing:

```bat
ollama pull qwen3:4b
python ollama_fix.py --dir "D:\Obsidian\Vault\OCR\Conference" --model qwen3:4b --limit 5
```

## Note format

```markdown
---
created: 2026-10-25T09:05:12
source: "[[OCR/Conference/attachments/Day_1/IMG_0001.jpg]]"
original: "D:/Photos/Day_1/IMG_0001.jpg"
ocr_engine: Umi-OCR
ocr_confidence: 0.81
tags:
  - ocr
---

# IMG_0001

![[OCR/Conference/attachments/Day_1/IMG_0001.jpg]]

## Текст
…recognized text…
```

After typo fixing, the original OCR text is kept below in a collapsed `> [!quote]-` callout, and `ocr_fixed: <model>` is added to the frontmatter.

## Repository structure

```
photo-to-obsidian/
├── SKILL.md              # instructions Claude follows
└── scripts/
    ├── umi_obsidian.py   # photos → Obsidian notes via Umi-OCR HTTP API
    ├── ocr_enhance.py    # anti-moiré preprocessing, two passes, line merging
    └── ollama_fix.py     # optional typo fixing with a local LLM
docs/                     # images for this README
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| TesseractOCR is missing in Umi-OCR settings | Umi-OCR is still running in the tray: quit it from the tray icon and start again; check the plugin is not in an extra nested folder |
| `no Tesseract languages: rus` | Put `rus.traineddata` into `<plugin>\engine\tessdata\` and restart Umi-OCR |
| Hangs on the first photo | Close heavy apps; never send full-size photos (the scripts downscale automatically) |
| `Unable to create process…` | A broken Python version is the default; run a working one by full path |
| Gibberish output | PaddleOCR is selected instead of TesseractOCR, or the confidence threshold is 60 instead of 30 |

## Limitations

- Small, pale text under heavy moiré can still be lost; edges of the frame (chat, taskbar) may add noise lines.
- Screenshots always beat photos. Next time use **Win + PrtScn** (saves the whole screen to *Pictures → Screenshots*).
- Tested on Windows 10. Windows 7 versions were checked against documentation, not run.

## Credits

- [Umi-OCR](https://github.com/hiroi-sora/Umi-OCR) by hiroi-sora (MIT)
- [tesseractOCR_umi_plugin](https://github.com/qwedc001/tesseractOCR_umi_plugin) by qwedc001
- [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) and [tessdata_best](https://github.com/tesseract-ocr/tessdata_best) (Apache 2.0)
- [Ollama](https://ollama.com) and [Qwen](https://github.com/QwenLM)

## License

[MIT](LICENSE) © 2026 Cristina Perez

## Author

**Cristina Perez**, Telegram channel [«Кристина о нейронках»](https://t.me/KristinaPerezDev): AI tools, automation and Telegram bots explained in plain language.
Questions and ideas: [Issues](../../issues) or [@KristinaPerez9](https://t.me/KristinaPerez9).

If this saved you an evening, give it a ⭐

---

<div align="center">

## 🇷🇺 Русский

</div>

**Навык для Claude, который превращает фото слайдов и экранов в заметки Obsidian с поиском.** Офлайн, бесплатно, Windows 7 / 10 / 11.

## Зачем

Вы вернулись с конференции с сотнями фото слайдов, снятых телефоном прямо с монитора. Они лежат в галерее, и их никто не читает.

Навык ведёт Claude (и вас) по проверенному конвейеру: текст с каждого фото распознаётся **у вас на компьютере** и сохраняется в Obsidian отдельной заметкой, картинка сверху, текст снизу. По желанию опечатки распознавания правит локальная нейросеть.

## Возможности

- **Одно фото - одна заметка**: frontmatter, копия картинки и распознанный текст
- **Полностью офлайн**: распознавание в [Umi-OCR](https://github.com/hiroi-sora/Umi-OCR) на движке Tesseract, исправление опечаток в [Ollama](https://ollama.com)
- **Заточено под фото экрана**: обработка против муара и два прохода распознавания дают **+50% правильных слов** по сравнению с одним проходом Tesseract
- **Безопасно**: оригиналы не изменяются и не перемещаются, готовые заметки пропускаются, запуск можно прерывать и продолжать
- **Русский и английский** из коробки, любой другой язык Tesseract добавляется
- **Для новичков**: Claude спрашивает ваши папки, даёт точные команды, сначала делает пробу на 2 фото и объясняет каждый шаг

## Как это работает

Фото монитора страдает от **муара**: сетки пикселей экрана и камеры накладываются и рисуют радужные полосы поверх букв. Неожиданный вывод: **меньше значит лучше**. На полном размере (4000+ px) Tesseract видит муар как текстуру и может зависнуть на десятки минут, а на 1600 px с лёгким размытием читает текст.

| Вариант | Слов |
|---|---:|
| Tesseract, один проход (размытие + ч/б, 2000 px) | 252 |
| + локальный контраст (CLAHE, 1600 px) | 332 |
| **Два прохода + склейка (по умолчанию)** | **379 (+50%)** |

Для сравнения: встроенная русская модель PaddleOCR на тех же фото давала уверенность 0.37-0.46 (после подбора настроек максимум 0.74), Tesseract около 0.88.

## Установка

**Claude (сайт или приложение):** скачайте `photo-to-obsidian.zip` в [Releases](../../releases/latest) → Настройки → **Customize → Skills** → загрузите zip целиком, не распаковывая, и включите навык. Нужно включённое выполнение кода. Потом напишите: *«хочу превратить фото слайдов в заметки Obsidian»*.

**Claude Code:** скопируйте папку `photo-to-obsidian/` в `%USERPROFILE%\.claude\skills\`.

**Без Claude:** скрипты работают и сами по себе. Шаги установки Umi-OCR, плагина Tesseract и русской модели описаны в [`photo-to-obsidian/SKILL.md`](photo-to-obsidian/SKILL.md), команды запуска выше, в английской части.

## Требования

| | Windows 10 / 11 | Windows 7 (64-бит) |
|---|---|---|
| Umi-OCR + плагин Tesseract | ✅ | ✅ |
| Python | 3.12 | 3.8.10 |
| Ollama + Qwen (опечатки) | ✅ (10 22H2+) | ❌ |

Памяти хватит 8 ГБ (модель `qwen3:4b`), при 16 ГБ можно `qwen3:8b`.

## Если что-то не так

| Симптом | Что делать |
|---|---|
| В настройках Umi-OCR нет TesseractOCR | Umi-OCR остался в трее: закройте его через значок у часов и запустите снова; проверьте, что у плагина нет лишней вложенной папки |
| `нет языков Tesseract: rus` | Положите `rus.traineddata` в `<плагин>\engine\tessdata\` и перезапустите Umi-OCR |
| Зависает на первом фото | Закройте тяжёлые программы; фото в полном размере не отправляйте (скрипт уменьшает сам) |
| `Unable to create process…` | По умолчанию стоит сломанная версия Python; запускайте рабочую по полному пути |
| Вместо текста каша | Выбран PaddleOCR вместо TesseractOCR или порог уверенности 60 вместо 30 |

## Ограничения

- Мелкий бледный шрифт под сильным муаром иногда теряется, с краёв кадра (чат, панель задач) может попасть мусор.
- Скриншот всегда лучше фото. В следующий раз снимайте **Win + PrtScn**: весь экран сразу сохраняется в «Изображения → Снимки экрана».
- Проверено на Windows 10. Версии для Windows 7 сверены по документации, но не запускались.

## Автор

**Кристина Перез**, Telegram-канал [«Кристина о нейронках»](https://t.me/KristinaPerezDev): нейросети, автоматизация и Telegram-боты человеческим языком.
Вопросы и идеи: [Issues](../../issues) или [@KristinaPerez9](https://t.me/KristinaPerez9).

Если навык сэкономил вам вечер, поставьте ⭐
