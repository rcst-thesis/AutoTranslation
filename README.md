# 🌐 HiligaynonTranslate — Automated EN→HIL Corpus Builder

> A zero-cost, API-free translation pipeline that drives a real browser to scrape Google Translate at scale — built for low-resource language dataset generation.

---

## 🧠 Background & Motivation

[Hiligaynon](https://en.wikipedia.org/wiki/Hiligaynon_language) is a Western Visayan language spoken by over 9 million people in the Philippines, yet it remains severely underrepresented in modern NLP research. Publicly available parallel corpora for Hiligaynon-English are scarce, making it difficult to train or fine-tune machine translation models for the language.

This project was born out of a need to generate large-scale English-Hiligaynon sentence pair datasets for a custom neural machine translation system. Rather than paying for API credits or relying on incomplete public datasets, this pipeline automates the translation process directly through Google Translate's web interface — no API key, no billing, no rate-limit headers.

The result: a reproducible, configurable pipeline capable of translating thousands of phrases and saving them as clean, structured CSV files ready for NLP preprocessing, model training, or linguistic research.

---

## ✨ Features

- **Zero API cost** — scrapes the live Google Translate web UI using Selenium and headless Chrome, completely bypassing the paid API
- **Multi-sentence support** — correctly handles phrases with multiple sentences by collecting all translation spans from the DOM, not just the first one
- **Auto-retry on blank translations** — if Google fails to render a translation, the scraper retries up to 3 times with an extended wait before moving on
- **Dual file format support** — accepts both `.txt` (id,phrase per line) and `.csv` (with or without headers) as input
- **Crash-safe progress saving** — writes progress to disk every 50 phrases, so a Colab session timeout won't wipe your entire run
- **Configurable language pair** — change two variables to translate between any language pair that Google Translate supports
- **Google Colab ready** — designed to run entirely in the cloud with no local setup required

---

## 🏗️ Architecture

```
Input File (.txt / .csv)
        │
        ▼
  File Parser
  (auto-detects format,
   splits id + phrase)
        │
        ▼
  Headless Chrome
  (Selenium + ChromeDriver)
        │
        ▼
  Google Translate Web UI
  translate.google.com/?sl=en&tl=hil&text=...
        │
        ▼
  DOM Scraper
  span[jsname='W297wb'] × N spans
        │
        ▼
  Retry Handler
  (up to 3 retries on blank)
        │
        ▼
  CSV Writer
  (progress save every 50 rows)
        │
        ▼
  output.csv
  [id | english | hiligaynon]
```

---

## 📂 Input Format

### `.txt` file
One entry per line in `id,phrase` format. Phrases can contain commas — the parser splits only on the **first** comma.

```
1,Hi!
2,How are you?
3,Anytime. You've got great energy!
4,True. Partnership > patronage.
5,"I'm sorry I hurt you. That wasn't my intention, and I'll work to do better."
```

### `.csv` file
With or without a header row. If a header is present, the parser auto-detects the English column by looking for `english` or `eng` in the column name.

```csv
id,english
1,Hello there!
2,What is your name?
3,Exactly! Specificity enables growth.
```

---

## 📤 Output Format

The output is a UTF-8 encoded CSV with three columns:

```csv
id,english,hiligaynon
1,Hi!,Hi!
2,How are you?,Kamusta ka?
3,Anytime. You've got great energy!,Bisan karon. Maayo gid ang imo enerhiya!
```

---

## 🚀 Quickstart (Google Colab)

### Step 1 — Install Chrome
```python
!wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
!apt install -y ./google-chrome-stable_current_amd64.deb -q
!google-chrome --version
```

### Step 2 — Install Python packages
```python
!pip install selenium webdriver-manager -q
```

### Step 3 — Upload your file
```python
from google.colab import files
uploaded = files.upload()
```

### Step 4 — Parse the file
```python
import csv, os

filename = list(uploaded.keys())[0]
ext = os.path.splitext(filename)[1].lower()
INPUT_PHRASES = []

if ext == ".txt":
    with open(filename, encoding="utf-8") as f:
        rows_raw = [line.strip() for line in f if line.strip()]
    for line in rows_raw:
        parts = line.split(",", 1)
        if len(parts) == 2:
            INPUT_PHRASES.append((parts[0].strip(), parts[1].strip()))

elif ext == ".csv":
    with open(filename, encoding="utf-8") as f:
        reader = csv.reader(f)
        headers = next(reader, None)
        if headers and not headers[0].strip().isdigit():
            headers = [h.strip().lower() for h in headers]
            eng_idx = next((i for i, h in enumerate(headers) if "english" in h or "eng" in h), 0)
            id_idx  = next((i for i, h in enumerate(headers) if "id" in h), None)
            for row in reader:
                if len(row) > eng_idx:
                    num = row[id_idx].strip() if id_idx is not None else str(len(INPUT_PHRASES) + 1)
                    phrase = row[eng_idx].strip()
                    if phrase:
                        INPUT_PHRASES.append((num, phrase))
        else:
            if headers:
                if len(headers) == 2:
                    INPUT_PHRASES.append((headers[0].strip(), headers[1].strip()))
                elif len(headers) == 1:
                    INPUT_PHRASES.append((str(len(INPUT_PHRASES) + 1), headers[0].strip()))
            for row in reader:
                if len(row) >= 2:
                    INPUT_PHRASES.append((row[0].strip(), row[1].strip()))
                elif len(row) == 1:
                    INPUT_PHRASES.append((str(len(INPUT_PHRASES) + 1), row[0].strip()))

print(f"Loaded {len(INPUT_PHRASES)} phrases from {filename}")
```

### Step 5 — Run the scraper
```python
import csv, time, urllib.parse
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager

SOURCE_LANG = "en"
TARGET_LANG = "hil"
OUTPUT_FILE = "output.csv"
DELAY       = 2.0

def build_driver():
    opts = Options()
    opts.add_argument("--headless=new")
    opts.add_argument("--no-sandbox")
    opts.add_argument("--disable-dev-shm-usage")
    opts.add_argument("--disable-gpu")
    opts.add_argument("--window-size=1280,800")
    opts.add_argument("--remote-debugging-port=9222")
    opts.add_argument("--disable-extensions")
    opts.add_argument("--disable-setuid-sandbox")
    opts.add_argument("--disable-blink-features=AutomationControlled")
    opts.add_experimental_option("excludeSwitches", ["enable-automation"])
    opts.add_experimental_option("useAutomationExtension", False)
    return webdriver.Chrome(
        service=Service(ChromeDriverManager().install()),
        options=opts
    )

def translate(driver, text, retries=3):
    url = (
        f"https://translate.google.com/?sl={SOURCE_LANG}"
        f"&tl={TARGET_LANG}&text={urllib.parse.quote(text)}&op=translate"
    )
    for attempt in range(1, retries + 1):
        driver.get(url)
        try:
            WebDriverWait(driver, 10).until(
                EC.presence_of_element_located(
                    (By.CSS_SELECTOR, "span[jsname='W297wb']")
                )
            )
            els = driver.find_elements(By.CSS_SELECTOR, "span[jsname='W297wb']")
            result = " ".join(el.text.strip() for el in els if el.text.strip())
        except Exception:
            result = ""

        if result:
            time.sleep(DELAY)
            return result

        print(f"  [blank, retrying {attempt}/{retries}]", end=" ")
        time.sleep(DELAY + 2)

    print(f"  [failed after {retries} retries]")
    return ""

driver = build_driver()
results = []

print(f"Translating {len(INPUT_PHRASES)} phrases [{SOURCE_LANG} → {TARGET_LANG}]...\n")

for i, (num, phrase) in enumerate(INPUT_PHRASES, 1):
    print(f"[{i}/{len(INPUT_PHRASES)}] {phrase}", end=" ... ")
    translation = translate(driver, phrase)
    print(translation)
    results.append({"id": num, "english": phrase, "hiligaynon": translation})

    if i % 50 == 0:
        with open(OUTPUT_FILE, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=["id", "english", "hiligaynon"])
            writer.writeheader()
            writer.writerows(results)
        print(f"  >> Progress saved at {i} phrases.")

driver.quit()

with open(OUTPUT_FILE, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "english", "hiligaynon"])
    writer.writeheader()
    writer.writerows(results)

print(f"\nDone. Saved {len(results)} rows to {OUTPUT_FILE}")
```

### Step 6 — Download the CSV
```python
from google.colab import files
files.download("output.csv")
```

---

## ⚙️ Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `SOURCE_LANG` | `"en"` | Source language code |
| `TARGET_LANG` | `"hil"` | Target language code |
| `OUTPUT_FILE` | `"output.csv"` | Output filename |
| `DELAY` | `2.0` | Seconds to wait between requests |
| `retries` | `3` | Max retries on blank translation |

### Supported Language Codes (sample)

| Code | Language |
|------|----------|
| `hil` | Hiligaynon |
| `tl` | Filipino / Tagalog |
| `ceb` | Cebuano |
| `ilo` | Ilocano |
| `es` | Spanish |
| `ja` | Japanese |
| `zh-CN` | Chinese (Simplified) |
| `fr` | French |
| `ar` | Arabic |

Full list: https://cloud.google.com/translate/docs/languages

---

## 🔬 Technical Notes

### Why not use the official Google Translate API?
The official API charges per character after the free tier. For large-scale corpus generation (10,000+ sentences), costs add up quickly. This scraper uses the same translation engine via the public web interface — free, accurate, and supports all the same language pairs.

### Why Selenium over requests/BeautifulSoup?
Google Translate renders translations entirely via JavaScript. The raw HTML response from a plain HTTP request contains no translation output. A real browser runtime (headless Chrome via Selenium) is required to execute the JavaScript and access the rendered DOM.

### Why does it collect multiple `span[jsname='W297wb']` elements?
Google Translate splits multi-sentence inputs into individual sentence spans in the output DOM. Grabbing only the first span would truncate the translation after the first sentence. This scraper collects all matching spans and joins them to reconstruct the full translation.

### Retry mechanism
Network latency, Google's internal rendering delays, or brief rate-limiting can cause spans to render empty. The retry handler re-navigates to the URL with an extended wait (`DELAY + 2s`) before each attempt, giving the browser more time to fully render the output.

---

## 📊 Performance

| Phrases | Estimated Time (DELAY=2.0s) |
|---------|-----------------------------|
| 100     | ~4 minutes                  |
| 500     | ~20 minutes                 |
| 1,000   | ~40 minutes                 |
| 4,000   | ~2.5 hours                  |

> Tip: Lower `DELAY` to `1.0` for faster runs. Increase it if you start seeing blank translations.

---

## ⚠️ Important Notes

- **Keep the Colab tab open** during long runs — Colab disconnects after ~90 minutes of inactivity
- Progress is auto-saved every 50 phrases to `output.csv` as a safety net
- This tool is intended for **research and educational purposes** (low-resource language NLP, academic dataset generation)
- Excessive automated requests may trigger Google's rate limiter — keep `DELAY` at a reasonable value

---

## 🗺️ Roadmap

- [ ] Resume from last saved row (skip already-translated phrases)
- [ ] Parallel scraping with multiple browser instances
- [ ] Support for additional output formats (JSON, TSV, JSONL)
- [ ] Automatic blank-row cleanup post-run
- [ ] GUI wrapper for non-technical users

---

## 🤝 Contributing

Pull requests are welcome. If you find a broken selector (Google sometimes changes their DOM structure), open an issue with the updated `jsname` attribute and I'll push a fix.

---

## 📄 License

MIT License — free to use, modify, and distribute with attribution.

---

## 👤 Author

Built by **Joshua Silubrico** as part of ongoing research into low-resource Philippine language NLP and machine translation.

- GitHub: [@BigCookieDough](https://github.com/BigCookieDough)
- LinkedIn: [joshua silubrico](https://www.linkedin.com/in/joshua-silubrico-860201180/)

---

> *"Language is the road map of a culture. It tells you where its people come from and where they are going."* — Rita Mae Brown
