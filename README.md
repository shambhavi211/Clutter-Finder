# Clutter-Finder
A perceptual duplicate and forgotten file detection tool.

# Clutter Finder

> Find what's quietly taking up space on your computer, and clean it up without ever losing a file by accident.

Clutter Finder is a local desktop tool that scans a folder and finds three kinds of digital clutter:

- **Exact duplicates**: byte-for-byte identical files (MD5 hashing)
- **Similar photos**: the same picture resized, re-compressed, or lightly edited (perceptual hashing)
- **Forgotten files**: files you haven't opened in 30 / 90 / 180 / 365 days

Everything runs on your own machine. Nothing is uploaded anywhere, and **nothing is ever deleted automatically**.

![Dashboard](docs/dashboard.png)

## Why it's different

Most duplicate finders only catch files that are byte-identical. Many "AI cleaners" catch similar photos too, but they are black boxes: you can't tell why two files were flagged.

Clutter Finder takes a different approach:

- **Built from scratch.** The similarity algorithm (dHash) is implemented in about 30 lines in `phash.py` using only Pillow. No `imagehash` library, no pretrained model, no multi-hundred-MB download.
- **Explainable.** Every decision comes down to one number: how many of 64 fingerprint bits differ between two images.
- **Review first.** You see thumbnails, preview files, and confirm before anything is deleted. Scanning is strictly read-only.

## Features

- Three-tab dashboard: Similar Photos, Exact Duplicates, Forgotten Files
- Groups of 3+ similar images are clustered together (Union-Find), not shown as disconnected pairs
- Shows how much space you'd reclaim before you delete anything
- Preview modal with the real folder name, full path and size, plus **Show in folder**
- Skips non-personal folders (`.venv`, `site-packages`, `node_modules`, `.git`, caches) so they never show up as "forgotten files"
- Incremental re-scans via a local SQLite database
- Packages into a single Windows `.exe` with no Python install needed

## How dHash works

![dHash](docs/dhash.png)

1. Shrink the image to a **9x8** grayscale grid. Fine detail disappears, so resizing and re-compression barely change the result.
2. For each row, compare every pixel to its right neighbour: brighter = `1`, otherwise `0`.
3. That gives **64 bits**, the image's fingerprint.
4. Two images are "similar" if their fingerprints differ by **6 bits or fewer** (Hamming distance).

**Known limitation:** dHash reads gradients, not absolute colour, so two completely flat, solid-colour images hash identically. Real photos have texture, so this rarely matters. It was caught by this project's own unit tests.

## Architecture

![Architecture](docs/architecture.png)

```
clutter-finder/
├── app.py                 # Flask app: routes + REST API
├── desktop_launcher.py    # Entry point for the packaged .exe
├── scanner.py             # Directory walker, metadata + hash collection
├── phash.py               # dHash + MD5 hashing (from scratch)
├── screenshot_detect.py   # Screenshot heuristics (filename + resolution)
├── grouping.py            # Union-Find clustering
├── db.py                  # SQLite storage
├── templates/index.html   # Dashboard
├── static/                # style.css, script.js
├── tests/test_core.py     # Unit tests
└── requirements.txt
```

**Tech stack:** Python 3.12, Flask, SQLite, Pillow, HTML/JavaScript (Tailwind), PyInstaller.

## Getting started

### Run from source

```bash
git clone https://github.com/<your-username>/clutter-finder.git
cd clutter-finder
pip install -r requirements.txt
python app.py
```

Open `http://127.0.0.1:5000`, paste a folder path (for example `C:\Users\you\Pictures`), and click **Scan folder**.

### Run the tests

```bash
python -m unittest discover tests
```

### Build the Windows .exe

```bash
pip install pyinstaller
python -m PyInstaller --onefile --name ClutterFinder --add-data "templates;templates" --add-data "static;static" desktop_launcher.py
```

The executable appears in `dist/ClutterFinder.exe`. Double-click it and your browser opens to the dashboard.

## API

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/scan` | POST | Scan a folder |
| `/api/duplicates` | GET | Groups of visually similar images |
| `/api/exact-duplicates` | GET | Groups of identical files |
| `/api/stale` | GET | Files not accessed within N days |
| `/api/thumbnail` | GET | Resized thumbnail for a file |
| `/api/reveal` | POST | Open the file's folder in the OS explorer |
| `/api/delete` | POST | Delete user-selected files (after confirmation) |

## Limitations

- Grouping compares every image with every other image (O(n²)). Fine for a few thousand photos; very large libraries would need bucketed/LSH indexing.
- Screenshot detection is rule-based (filename + resolution), not ML.
- "Forgotten" relies on the OS keeping last-accessed timestamps, which some systems disable.
- Near-duplicate detection covers images only (not video, audio or documents).

## Roadmap

- [ ] Parallel hashing with `multiprocessing`
- [ ] LSH-style bucket index for large libraries
- [ ] Lightweight PCA-based similarity as a second signal alongside dHash
- [ ] Move-to-Recycle-Bin instead of permanent delete
- [ ] Screenshot / document classifier
- [ ] macOS and Linux builds
      
## Ideas for extending it further

- Multiprocessing for hashing thousands of images faster
- OCR-based text similarity for documents/PDFs (using `pytesseract`)
- A "safe to delete" confidence score combining similarity + file age + size
