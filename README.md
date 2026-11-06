# PDF Converter (Real Estate)

**Project name:** **PDF Converter (Real Estate)**  
**Purpose:** Convert, categorize, and organize real‑estate PDF brochures into a structured folder hierarchy with extracted images and metadata.

---

## 1. Local Environment
| Item | Detected Value |
|------|----------------|
| **Operating system** | Windows 10/11 (host) |
| **Python version** | 3.11.5 |
| **Virtual environment** | `.venv` (located in the project root) |
| **CPU / GPU** | No GPU‑specific libraries are used – processing runs on the CPU. |
| **Key libraries** | `fitz (PyMuPDF)`, `opencv-python`, `numpy`, `Pillow`, `tqdm`, `shutil`, `gc`, `zipfile`, `re`, `os` |
| **Additional packages installed** | See the full `pip freeze` output in the repository – notable ones for the pipeline include `pymupdf`, `opencv-python`, `numpy`, `tqdm`, `pandas`, `pdfminer.six`, `pdfplumber`, `streamlit`, `fastapi`, `uvicorn`, etc. |
| **Project directory** | `D:\DEPI\Freelance\PDF Converter (real state)` |
| **Verification** | The Jupyter notebook `pdf-conv.ipynb` runs without import errors and produces progress output; the directory structure (`input`, `output`, `working_output`) matches the notebook’s expectations. |

> **Note:** No GPU‑accelerated inference is performed; all heavy lifting is done with CPU‑based image and PDF libraries.

---

## 2. Project Structure
```
PDF Converter (real state)/
├─ .venv/                         # Python virtual environment (excluded from README)
│  └─ …
├─ input/                         # User‑provided PDFs (e.g. “GREENS INSERT BUILDING 3 - G6 (2).pdf”)
│  └─ *.pdf
├─ output/                        # Final organized results (unchanged PDFs, images)
│  └─ Isola_Quattro_Brochure/
│     ├─ Floor_Plans/
│     ├─ Master_Plans/
│     └─ Visuals/
│        ├─ Visual_1_p1.png
│        └─ … (many PNGs)
├─ working_output/                # Intermediate copy of PDFs and extracted images
│  └─ Brochures/
│     └─ At east.pdf
├─ pdf-conv.ipynb                 # Main Jupyter notebook that drives the pipeline
├─ All_Projects_Output.zip        # Archive of the full `output/` folder (generated)
└─ README.md                      # <-- **this file**
```

*Only the notebook and the top‑level folders are part of the public codebase; everything inside `.venv` is the isolated Python environment.*

---

## 3. Supported Inputs & Outputs
| Type | Description |
|------|-------------|
| **Input PDFs** | Any PDF placed in `input/`. The notebook scans the folder recursively and processes each file. |
| **Extracted images** | PNG files saved under `output/<Project‑Name>/<Category>/` (e.g., `Visuals/Visual_10_p10.png`). |
| **Categorized PDFs** | Original PDFs are copied to `working_output/Brochures/` with a cleaned name. |
| **Archive** | After processing, `All_Projects_Output.zip` contains the entire `output/` tree for easy distribution. |
| **Metadata** | No explicit JSON/YAML metadata files are generated; the folder hierarchy implicitly encodes the label (category) and page number. |

*The pipeline does **not** modify the source PDFs and preserves their original content.*

---

## 4. Core Technologies
| Technology | Role in the project |
|------------|--------------------|
| **PyMuPDF (`fitz`)** | Fast PDF page rasterisation to Pillow images (`fitz.open(...).load_page(i).get_pixmap()`). |
| **OpenCV (`cv2`)** | Image preprocessing (e.g., resizing, thresholding) before categorisation. |
| **NumPy** | Numerical operations on image arrays. |
| **Pillow (`PIL`)** | Image saving (`Image.save`) and basic manipulation. |
| **`tqdm`** | Progress bars for long‑running loops (e.g., processing 52 pages). |
| **Standard library (`os`, `shutil`, `zipfile`, `gc`)** | File system handling, copying PDFs, creating the final ZIP archive, and explicit garbage collection to release memory. |
| **`re`** | Simple regular‑expression based text extraction (e.g., labels inside PDF text). |
| **`streamlit` / `fastapi`** (installed but not used in the notebook) | Potential UI/serving layers – *not exercised by the current pipeline*. |

> **AI / ML**: The project does **not** load external machine‑learning models. All “categorisation” logic is rule‑based (keyword matching on extracted labels such as `StandAlone_Villa`, `TownHouse_Middle`, etc.).

---

## 5. Application Architecture
```mermaid
graph TD
    A[User places PDFs in input/] --> B[Notebook (pdf-conv.ipynb) start]
    B --> C{Iterate over PDFs}
    C -->|Open PDF| D[PyMuPDF → page images]
    D --> E[Analyze page text & image]
    E -->|Keyword match| F[Determine category (e.g., StandAlone_Villa)]
    F --> G[Save image as PNG under output/<Project>/<Category>/]
    E -->|No match| H[Fallback to generic handling]
    G --> I[Copy original PDF to working_output/Brochures/]
    I --> J[Update progress bar (tqdm)]
    J --> K[After all PDFs processed → create zip archive]
    K --> L[All_Projects_Output.zip ready]
```
*All data flow stays on‑disk; no network calls are performed.*

---

## 6. Complete Processing Pipeline
1. **Discover input PDFs** – `INPUT_DIRS` (default: `['input']`).
2. **For each PDF** (`process_all_files()`):
   1. Open with `fitz`.
   2. Loop over every page (`for i in range(doc.page_count)`).
   3. Extract **text** (`page.get_text()`) and **image** (`page.get_pixmap() → PIL.Image`).
   4. Call `analyze_page(text, pil_img)` – searches for known labels (e.g., `StandAlone_Villa`, `TownHouse_Middle`, `TwinHouse`).
   5. If a label is found, compute a unique index via `tracker.get(label, page_num, is_mp)`.
   6. Save the image as `output/<Project>/<Category>/<safe_label>_<idx>_p<page_num>.png`.
   7. Copy the source PDF to `working_output/Brochures/` (preserving the original name).
   8. `gc.collect()` frees memory before the next page.
3. After all PDFs are processed, `shutil.make_archive()` creates `All_Projects_Output.zip` containing the full `output/` directory.
4. Print a final “All done!” message.

*Progress is reported via `tqdm` streams visible in the notebook’s output cells.*

---

## 7. Core Functions & Modules (as seen in `pdf-conv.ipynb`)
| Function | Purpose | Key Parameters | Returns |
|----------|---------|----------------|---------|
| `process_all_files()` | Orchestrates the whole conversion run. | `INPUT_DIRS` (list of source folders) | None (side‑effects: files saved, ZIP created) |
| `analyze_page(text, pil_img)` | Parses page text to extract a **category label** and decides whether special handling (`is_mp`) is needed. | `text` (str), `pil_img` (PIL.Image) | `(category, label)` |
| `tracker.get(label, page_num, is_mp)` | Generates a deterministic index for a given label to avoid filename clashes. | `label` (str), `page_num` (int), `is_mp` (bool) | `int` |
| `save_image(img, path)` *(inline in notebook)* | Writes a Pillow image to the designated PNG file, using `optimize=True`. | `img` (PIL.Image), `path` (str) | None |
| `copy_pdf(src, dst)` *(inline)* | Copies the original PDF into `working_output/Brochures/`. | `src` (Path), `dst` (Path) | None |

All other imports (`os`, `re`, `shutil`, `gc`, `zipfile`) serve auxiliary purposes.

---

## 8. Installation & Quick‑Start
1. **Prerequisites**  
   * Windows 10/11 with a recent Python 3.11 installation.  
   * Sufficient disk space for PDFs and extracted images (a few GB for large batches). 
2. **Clone / open the project**
   ```powershell
   cd "D:\DEPI\Freelance\PDF Converter (real state)"
   ```
3. **Create / activate the virtual environment** *(already provided)*
   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```
4. **Install required packages** *(already satisfied, but for reproducibility)*
   ```powershell
   pip install -r requirements.txt   # (if a requirements file is added later)
   ```
5. **Prepare input PDFs** – drop your PDF brochures into `input/`. 
6. **Run the notebook**
   *Open `pdf-conv.ipynb` in JupyterLab / VS Code and execute all cells.*  
   - The notebook will display a progress bar and print per‑page status. 
7. **Result**
   *Processed images appear under `output/…` and a zip archive `All_Projects_Output.zip` is generated at the project root.*

---

## 9. GUI & User Interface
The repository ships only a Jupyter notebook. No separate graphical UI is provided. A **Streamlit** or **FastAPI** UI could be built on top of the same functions, but no UI code is present in the project at this time.

---

## 10. CLI / API Reference
| Command | Effect |
|---------|--------|
| `python -m ipykernel_launcher -f <kernel‑json>` | Starts the notebook kernel (used by Jupyter). |
| `jupyter nbconvert --to script pdf-conv.ipynb` | Export the notebook as a plain‑Python script (useful for automation). |
| `streamlit run pdf-conv.ipynb` | *Not supported* – Streamlit expects a `.py` script, not a notebook. |

*No dedicated CLI script exists; the notebook is the primary entry point.*

---

## 11. Configuration
| Setting | Where to set | Description |
|---------|--------------|-------------|
| `INPUT_DIRS` | Inside `pdf-conv.ipynb` (top cell) | List of directories to scan for PDFs. |
| `BASE_OUTPUT` | Inside `pdf-conv.ipynb` | Root folder where the organized output is written (`output/`). |
| `PROJECT_NAME` | Inside `pdf-conv.ipynb` | Name used for the final ZIP archive (`<PROJECT_NAME>_FULL.zip`). |
| `PYTHONPATH` | Virtual environment activation | No custom path required – all imports resolve inside `.venv`. |

**Environment variables** are not used by the current code.

---

## 12. Logging & Reports
*The notebook prints plain text lines to stdout (captured by Jupyter).* Typical log entry:
```
Input directories: ['input']
Skipping 24 already-processed compounds.
Total PDFs found: 12
=============================================================
Processing: At_east
  Copied PDF to Brochures folder: ./working_output/Brochures/At east.pdf
    Extracted type: StandAlone_Villa 370m
...
All done!
```
No structured log files are written; developers can redirect stdout to a file if needed.

---

## 13. Testing & Verification
- **Unit tests**: None are included in the repository. 
- **Integration tests**: Running the notebook on a small set of PDFs serves as a manual verification step. 

*Running the notebook on the shipped sample (`pdf-conv.ipynb`) completes without errors, confirming that all imports resolve and the pipeline produces the expected folder hierarchy.*

---

## 14. Troubleshooting
| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `ImportError: No module named 'fitz'` | Virtual environment not activated or missing dependency. | Activate `.venv` (`.\.venv\Scripts\Activate.ps1`) and run `pip install pymupdf`. |
| `MemoryError` during large PDFs | Images not released fast enough. | Ensure `gc.collect()` is called (already present) or process fewer PDFs at a time. |
| No output files generated | `INPUT_DIRS` points to an empty folder. | Place PDFs in `input/` or update `INPUT_DIRS` accordingly. |
| Corrupted PNGs | PDF page rasterisation failed. | Verify the PDF is not password‑protected or corrupted; re‑download the source file. |
| `zipfile.BadZipFile` when opening the final archive | Archive creation interrupted. | Re‑run the notebook; ensure the process finishes without interruption. |

---

## 15. Performance & Optimization
- **CPU‑bound**: All heavy work (PDF rasterisation, image saving) runs on the CPU. 
- **Memory usage**: Each page image is kept in memory only briefly; explicit `gc.collect()` mitigates leaks. 
- **Speed**: For a 52‑page brochure, processing takes roughly 5‑10 seconds per file on a modern laptop (as shown by the `tqdm` timestamps). 
- **Potential optimisations**:  
  * Batch processing of multiple PDFs in parallel (`concurrent.futures`).  
  * Downscale images before saving if high resolution is unnecessary. 

---

## 16. Limitations & Future Improvements
| Limitation | Suggested improvement |
|------------|-----------------------|
| **Rule‑based categorisation** – only matches a fixed list of keywords. | Integrate a lightweight classifier (e.g., a fine‑tuned `transformers` model) to recognise more diverse labels. |
| **No CLI** – the notebook must be run manually. | Extract the core functions into a `pdf_converter.py` module and provide a `python -m pdf_converter` entry point. |
| **No structured metadata** (e.g., JSON summary). | Emit a manifest file (`manifest.json`) listing each saved image with its source PDF, page, and detected label. |
| **No UI** – only notebook output. | Build a Streamlit app that lets users upload PDFs and watch the conversion live. |
| **No unit tests** – code changes are untested. | Add `tests/` with pytest cases for `analyze_page`, `tracker.get`, and file‑system helpers. |

---

## 17. License & Credits
- **License:** The repository does **not** contain a license file. *If you intend to distribute or modify the project, add an appropriate open‑source license (e.g., MIT, Apache 2.0).* 
- **Authors / Contributors:** No author metadata is present. 
- **Third‑party components** (all listed in `pip freeze`):  
  - PyMuPDF, OpenCV, Pillow, tqdm, and the other Python packages are used under their respective licenses (see each package’s documentation).

---

*File location:* `D:\DEPI\Freelance\PDF Converter (real state)\README.md`

