# Shenu System

Write capital letters in the air with your index finger. The webcam reads them,
builds words, and keeps a searchable word list.

**Live:** https://air-writer-lac.vercel.app

## How to use
1. Open the page in Chrome or Edge and press **Start camera**.
2. Hold up **one finger** and write a big capital letter inside the dashed box.
3. **Hold still 1 s** or **make a fist** to finish the letter.
4. **Open hand, hold 1 s** to save the word. Switch to **Search** to write into the search box.
5. If the dot moves the wrong way, tick **Flip left/right**. Use **Sensitivity** to move your hand less.
6. **Train my handwriting**: write each letter 3 times so it learns your style.

## How it works
- Hand tracking: MediaPipe Hand Landmarker, in the browser.
- Cursor: One Euro filter (steady and low delay), glitch rejection, follows one hand.
- Letters: a small CNN (33k parameters, int8 weights embedded in `index.html`) trained on
  130,000 synthetic air-written letters, combined with a $P point-cloud matcher that also
  learns from your corrections and training samples.
- Everything runs locally; words and samples are kept in the browser's local storage.

## Retrain the model
```
cd ml
pip install torch numpy
python3 train.py          # writes weights.json
```
Then paste the contents of `weights.json` into `index.html` after `const CNN_W=`.

## Deploy
It is a static site: serve the folder over HTTPS (camera access needs HTTPS), e.g. Vercel or GitHub Pages.
