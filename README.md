# Multiple Face Recognition

Real-time, multi-person face recognition from a live webcam feed, built with Python, **OpenCV**, and the **`face_recognition`** library (dlib). The app detects every face in the frame, matches each against a set of known reference photos, and draws a labelled box with the matched person's name (or **"Unknown"**) around each face.

It ships in two flavours:

- **`main.py`** — a standalone desktop app that opens an OpenCV window.
- **`app.py`** — a Flask web app that streams the annotated video to a browser.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Setup: Add Known Faces](#setup-add-known-faces)
- [Usage](#usage)
- [How the Matching Works](#how-the-matching-works)
- [Performance Notes](#performance-notes)
- [Known Issues](#known-issues)
- [Responsible Use](#responsible-use)

---

## How It Works

```
Webcam frame ──► downscale to 1/4 ──► BGR→RGB
                         │
                 face_recognition.face_locations()  ──► where the faces are
                         │
                 face_recognition.face_encodings()  ──► 128-d vector per face
                         │
          compare to known encodings (smallest distance = best match)
                         │
          draw box + name back on the full-size frame ──► display / stream
```

For every frame the app finds all faces, converts each to a 128-dimension encoding, and picks the closest known face by distance. Matches are drawn on the original full-resolution frame.

---

## Project Structure

| File | Description |
|---|---|
| `main.py` | Desktop version — opens an OpenCV window; press **`q`** to quit. |
| `app.py` | Flask version — streams the annotated webcam feed at `http://127.0.0.1:5000`. |
| `index.html` | Page that embeds the live MJPEG stream (`/video_feed`). |
| `result.html` | A simple results table template (not wired into the current flow). |
| `requirements.txt` | Python dependencies (Flask, OpenCV, dlib, face-recognition, …). |
| `rizan.jpg`, `bradley.JPG` | Sample reference face images. |

---

## Requirements

- Python 3.6–3.8 (dlib wheels in `requirements.txt` are pinned to older versions)
- A working webcam
- **CMake** and a C/C++ build toolchain (needed to build **dlib**)
- The packages in `requirements.txt`

---

## Installation

```bash
git clone https://github.com/rizaan/Multiple-Face-Recognition.git
cd Multiple-Face-Recognition

python -m venv venv
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
```

> **`dlib` / `face_recognition` can be the hard part.** `dlib` compiles from source and needs CMake plus a build toolchain. On Windows, install CMake and the "Desktop development with C++" workload (Visual Studio Build Tools) first. On Linux: `sudo apt install build-essential cmake python3-dev`. On macOS: `brew install cmake`.

---

## Setup: Add Known Faces

The scripts expect each reference image in its **own folder named after the person**, and the code loads them by path. As written:

- `main.py` loads `rizaan/rizaan.jpg` and `Bradley/bradley.jpg`
- `app.py` loads `Krish/krish.jpg` and `Bradley/bradley.jpg`

So create the folders and drop in a clear, front-facing photo of each person:

```
Multiple-Face-Recognition/

├── Bradley/
│   └── bradley.jpg
└── Krish/
    └── krish.jpg
```

> The sample images in the repo root (`rizan.jpg`, `bradley.JPG`) don't match these expected paths/case exactly — move/rename them into the folders above, or edit the paths at the top of the script. To add more people, add another `load_image_file(...)` + `face_encodings(...)[0]` and append the encoding and name to `known_face_encodings` / `known_face_names`.

Each reference photo must contain **exactly one clearly visible face** — `face_encodings(...)[0]` takes the first face found and will raise an `IndexError` if none is detected.

---

## Usage

### Desktop (OpenCV window)

```bash
python main.py
```

Opens the webcam in a window, boxes and labels recognised faces, and quits on **`q`**.

### Web (Flask stream)

```bash
python app.py
```

Open **http://127.0.0.1:5000** in a browser to watch the annotated live stream.

---

## How the Matching Works

1. `compare_faces()` returns, for each known face, whether it's within dlib's default tolerance (0.6) of the detected face.
2. `face_distance()` gives the numeric distance to each known face.
3. The app takes the **smallest distance** (`np.argmin`) and, if that best candidate is also within tolerance, assigns its name — otherwise the face is labelled **"Unknown"**.

Using the smallest distance (rather than the first match) reduces false labels when several known faces are similar.

---

## Performance Notes

- Frames are processed at **1/4 resolution** for speed, then results are scaled back up (`× 4`) for display.
- `main.py` also processes **every other frame** (`process_this_frame`) to roughly halve the work. `app.py` processes every frame, so it's heavier.
- Recognition speed depends heavily on CPU; `face_recognition`/dlib can use a GPU build for more throughput.

---

## Known Issues

- **Mismatched sample paths.** The repo's loose images (`rizan.jpg`, `bradley.JPG`) don't line up with the folder paths and casing the code expects — set these up as described in [Setup](#setup-add-known-faces).
- **`app.py` and `main.py` know different people.** `app.py` expects `Krish` + `Bradley`; `main.py` expects `rizaan` + `Bradley`. Align them if you want both versions to recognise the same set.
- **`debug=True`** in `app.py` is fine for local development but should be off for anything beyond your own machine.
- **Name typo:** `"Bradly"` in `app.py` vs `"Bradley"` in `main.py`.
- **BGR→RGB slice** (`[:, :, ::-1]`) can produce a non-contiguous array on some dlib builds; if you hit an error there, wrap it with `np.ascontiguousarray(...)`.
- `result.html` is included but not connected to any route.

---

## Responsible Use

Face recognition identifies people, so it carries real privacy and legal weight.

- Use it only on **your own camera** and on people who have **consented**, or in a closed test/lab setting with sample images.
- Recognising or tracking people **without their knowledge or consent** may be unlawful (e.g. GDPR, BIPA, and other biometric/privacy laws) and is ethically fraught — check the rules in your jurisdiction before deploying anything beyond a personal demo.
- Store reference images and any logs securely, and delete them when no longer needed.

This project is intended for learning and authorised use only; you are responsible for how you deploy it.
