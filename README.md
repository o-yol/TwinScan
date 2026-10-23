# TwinScan

Find exact duplicate files and near-duplicate images in any folder, review them side by side, and delete the extras safely.

TwinScan is a browser-based duplicate finder built with JavaScript. It uses the File System Access API to read folders directly, SHA-256 hashing for exact matches, and perceptual hashing (dHash) with Hamming distance for visually similar images.

## Features

- **Exact duplicate detection**: size pre-filter, then SHA-256 content hashing
- **Near-duplicate image detection**: 64-bit dHash fingerprints compared by Hamming distance
- **Adjustable strictness**: slider regroups similar images instantly without re-hashing
- **Thumbnail previews** for every image in a group
- **Auto-select**: keep oldest, newest, or shortest path in one click
- **Safety rules**: nothing is selected by default, a group can never be fully selected, and a confirmation dialog shows the count and size before deletion
- **CSV report** export of all groups
- **Summary stats**: files scanned, hashed, skipped, reclaimable space, and scan time
- Handles empty files, files over 500 MB, and unreadable files without stopping the scan

## Tech stack

HTML, CSS, JavaScript (ES modules), File System Access API, Web Crypto API, Canvas API.

## Getting started

The File System Access API requires a secure context (`localhost` or HTTPS) and a desktop Chromium browser (Chrome or Edge).

```bash
git clone https://github.com/<your-username>/twinscan.git
cd twinscan
python -m http.server 8000
```

Open `http://localhost:8000`, click **Choose Folder & Scan**, and grant read/write access (write access is used for deletion).

To run the tests, open `http://localhost:8000/tests.html`. Every line should read PASS.

## Project structure

```
twinscan/
├── index.html        # App markup
├── style.css         # Styles
├── tests.html        # Browser-based test suite
└── js/
    ├── main.js       # App state, event wiring, selection and deletion logic
    ├── scanner.js    # Recursive folder walk
    ├── grouper.js    # Size grouping, exact grouping, near-duplicate clustering
    ├── hasher.js     # SHA-256, dHash, Hamming distance
    └── ui.js         # DOM rendering, CSV export
```

## How it works

**1. Scan.** Walk the chosen folder and all subfolders, storing for each file its name, path, size, modified date, and file and parent-folder handles (needed for deletion). Empty files are skipped.

**2. Exact duplicates.** Group files by size. A file with a unique size cannot have an exact copy, so only files sharing a size with another file are hashed with SHA-256. Equal hashes mean identical content.

**3. Near-duplicate images.** Compute a 64-bit dHash for every image. Images whose hashes differ in only a few bits are grouped as similar.

### Worked example

| File | Size | Notes |
|---|---|---|
| `a.jpg` | 2.0 MB | original photo |
| `b.jpg` | 2.0 MB | exact copy of a.jpg |
| `c.jpg` | 2.0 MB | different photo, same size |
| `d.txt` | 10 KB | |
| `e.png` | 500 KB | resized screenshot of a.jpg |

- Size grouping: a, b, c share a size and are hashed. `d.txt` and `e.png` have unique sizes and are not.
- Hashing: a and b share a SHA-256 hash, c does not. Exact group: {a, b}.
- Fingerprinting: a, b, c, e get a dHash (`d.txt` is not an image).
- Grouping: a, b, e fall within the threshold and c does not. {a, b, e} is kept as a similar group because e's content hash differs from the others.

## Algorithms

**Size pre-filter.** Files of different sizes can never be identical, and comparing sizes costs almost nothing compared with reading file contents. Only files that share a size are hashed.

**SHA-256.** Maps file contents to a 64-character digest using `crypto.subtle.digest`. Identical content always produces the same digest.

**dHash (difference hash).**
1. Shrink the image to 9×8 pixels using a canvas.
2. Convert each pixel to grayscale: `0.299R + 0.587G + 0.114B`.
3. In each of the 8 rows, compare every pixel to its right neighbour (8 comparisons per row, 64 total).
4. Each comparison yields one bit: 1 if the left pixel is brighter, otherwise 0.
5. Store the 64 bits as two 32-bit integers.

Resizing or re-compressing an image changes pixel values but rarely changes which neighbour is brighter, so the hash stays nearly the same.

**Hamming distance.** The number of differing bits between two hashes, computed with XOR followed by a popcount. The slider (0 to 16) sets the largest distance still treated as similar. Lower is stricter.

**Leader-based clustering.** Take the first ungrouped image as the leader, add every ungrouped image within the threshold of that leader, and repeat. It is simple and predictable, but because members are compared only to the leader, chains (A close to B, B close to C, A far from C) can be split.

## Complexity

| Stage | Cost |
|---|---|
| Scanning | O(n) for n files |
| Size grouping | O(n) |
| Hashing | O(total bytes of candidate files) |
| Fingerprinting | O(number of images) plus image decoding |
| Near-duplicate grouping | O(k²) for k images in the worst case |

## Design decisions

- **Sequential processing with `await`**: files are handled one at a time, so the progress bar stays responsive and only one file is held in memory at once.
- **500 MB hashing cap**: Web Crypto cannot hash incrementally, so a file must be fully loaded to be hashed. Larger files are skipped and listed separately.
- **Permanent deletion with safeguards**: the File System Access API has no recycle bin. Nothing is pre-selected, a confirmation shows the count and size, and the app never allows every file in a group to be selected.
- **Dual-group awareness**: a file can appear in both an exact group and a similar group. Selection is validated against both, and again right before deletion.
- **Slider clears selection**: regrouping changes group membership, so previous selections are discarded to avoid stale picks.
- **Leader-based clustering**: chosen for clarity and speed over full transitive clustering.

## Limitations

- Desktop Chrome or Edge only (File System Access API support).
- HEIC images are not supported because browsers cannot decode them.
- Files over 500 MB are not compared.
- Chains of similar images can be missed by leader-based clustering.
- Deleted files bypass the recycle bin.
- Rotated or heavily cropped images will not match.
- Transparent PNG regions render as black when downscaled.
- Animated GIFs are compared using the first frame only.
- Images are detected by extension (jpg, jpeg, png, webp, bmp, gif).

## Roadmap

- Partial-hash pre-filter (hash the first chunk before the full file)
- Web Workers for very large libraries
- Transitive clustering that follows chains
- "Move to folder" as an alternative to deletion
