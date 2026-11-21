# Shenu System – Code Notes (simple explanation)

These notes explain how the code works, step by step, in simple words.
Line numbers point to `index.html`.

---

## 1. The big picture

Everything happens in **one file**: `index.html`. It runs in the browser. There is no server.

```
 Camera ─► Hand tracker ─► Fingertip dot ─► Ink trail ─► Letter reader ─► Word ─► Word list
 (video)   (MediaPipe)     (filter, box)    (strokes)    (AI + matcher)   (A,B…)   (saved, search)
```

1. The **camera** sends about 30 pictures (frames) per second.
2. **MediaPipe** (a Google library) finds **21 points** on your hand in each frame.
3. We take point **number 8** = the tip of your index finger. That becomes the **green dot**.
4. While your **index finger is up**, the dot leaves **ink** (a trail of points).
5. When you **hold still** or **make a fist**, the ink is sent to the **letter reader**.
6. The reader says "this is A". The letter is added to the **word**.
7. **Open hand** saves the word into the **word list**. You can **search** the list.

---

## 2. The three parts of the file

| Part | Lines | What it is |
|---|---|---|
| `<style>` | 10–88 | Colors, fonts, layout (how it looks) |
| HTML | 90–172 | The boxes on the page: camera area, buttons, word, list |
| `<script>` | 174–683 | All the logic (JavaScript) |

The script has these sections (search for `// ----------` to jump between them):

| Section | Lines | Job |
|---|---|---|
| $P recognizer | 175–226 | Compares your drawing with example letter shapes |
| Letter templates | 228–296 | Example shapes of A–Z |
| Neural network | 298–353 | The trained AI that reads the letter |
| Recognize | 355–379 | Combines the two readers, gives the final answer |
| UI state | 381–486 | Word, list, buttons, search, training mode |
| Camera + hand tracking | 488–631 | Camera, finger detection, the dot, gestures |
| Render loop | 632–683 | Draws everything on screen, 60 times a second |

---

## 3. Camera and hand tracking

### Starting the camera — `startCamera()` (line 495)
```js
const stream = await navigator.mediaDevices.getUserMedia({video: {...}});
video.srcObject = stream;
```
- `getUserMedia` asks the browser for the camera. The browser shows "Allow camera?".
- Then we load MediaPipe's **HandLandmarker** model (about 8 MB) from the internet.
- `numHands: 2` = it can see 2 hands; we choose the right one later.

### The 21 hand points
```
        8   12  16  20      ← fingertips (8 = index tip)
        7   11  15  19
        6   10  14  18      ← middle joints
        5    9  13  17      ← knuckles
  4
   3
    2
     1
          0                 ← wrist
```

### Is a finger up or down? — `classify(lm)` (line 530)
- For each finger we measure the **angle at the middle joint** (`angleAt`, line 525).
  - Straight finger ≈ 180° → **up**.
  - Bent finger ≈ 90° or less → **down**.
- Then we decide the **gesture (posture)**:

| Fingers up | Posture | Meaning |
|---|---|---|
| Index only | `draw` | Pen down, write |
| Index + middle | `lift` | Pen up |
| All four | `open` | Hold 1 s = save word |
| None | `fist` | Letter finished |

- **Hysteresis**: it is harder to *change* a finger's state than to *keep* it. This stops flickering.

### Following one hand — `pickHand(hands)` (line 547)
If two hands are seen, take the one whose fingertip is **closest to the dot**. So the dot does not jump to your other hand.

---

## 4. The green dot (cursor)

All of this is inside `handleHand(lm, now)` (line 578). It runs for **every camera frame**.

### Step 1 – Mirror — `rawOf()` (line 568)
```js
x = flip ? lm8.x : 1 - lm8.x
```
The camera sees you like another person does. We flip left/right so it feels like a **mirror**.
Some Windows webcams already flip → the **Flip left/right** checkbox turns our flip off.

### Step 2 – Ignore glitches
```js
if (jump > 0.2 && glitch < 3) { glitch++; return; }
```
If the fingertip suddenly jumps more than 20% of the picture in one frame, it is almost always a
tracking mistake. We ignore up to 3 such frames.

### Step 3 – Smooth the shake — `OneEuro` (line 558)
The tracker shakes a little. The **One Euro filter** fixes this:
- finger moving **slowly** → smooth a lot → no shake
- finger moving **fast** → smooth a little → no delay

### Step 4 – Sensitivity box — `zone()` and `toScreen()` (lines 566–570)
- The **dashed box** is a smaller part of the camera picture.
- That box is stretched to fill the **whole** writing area.
- Sensitivity 2.0 → box is half the size → you move your hand **half as much**.

---

## 5. Writing: strokes and when a letter ends

- `current` = the stroke being drawn now (a list of `{x, y}` points).
- `strokes` = finished strokes of this letter (A can have 2: Λ and the bar).
- While posture is `draw`, each new dot position is added to `current`.
- `commitStroke()` (line 572) moves `current` into `strokes` and cuts 2 shaky points from each end.

**The letter is finished when:**
1. You **hold still** ~1 second (`isStill`), or
2. You show a **fist / move your hand away** for 0.7 s (`idleSince`).

Then `readNow()` (line 440) is called.

---

## 6. Reading the letter (the clever part)

`readNow()` collects all points and calls `recognize(pts)` (line 358).
Two different "readers" vote:

### Reader 1: Neural network (60% of the vote) — `cnnPredict()` (line 343)
1. **`rasterize()`** (line 303) draws your trail into a tiny **28 × 28 pixel picture** (like MNIST).
2. That picture goes through a small **CNN** (Convolutional Neural Network):
   ```
   28×28 picture
     → conv (12 filters) → pool → 14×14
     → conv (24 filters) → pool → 7×7
     → conv (48 filters) → pool → 4×4
     → dense layer → 26 numbers (one per letter A–Z)
   ```
   - `conv()` (line 324): slides small 3×3 filters over the picture to find lines, corners, curves.
   - `pool()` (line 335): shrinks the picture, keeps the strongest signal.
3. **Softmax** turns the 26 numbers into probabilities: `A: 0.93, H: 0.04, …`

The network's knowledge = **33,146 numbers (weights)**. They are stored inside `index.html`
in `const CNN_W = …` (line 299) as compressed text (int8 + base64).

### Reader 2: $P point-cloud matcher (40% of the vote) — lines 175–226
1. `normalize()`: make your drawing a standard size and position, 48 points.
2. Compare it with every **template** (example letter) using `greedy()` / `cloudDistance()`:
   match each of your points to the nearest template point, add up the distances.
3. Small total distance = similar shape.
- Templates come from `BASE` and `EXTRA` (lines 231–276), drawn on a 10 × 10 grid.
  Example: `A: [[0,10],[5,0],[10,10]], [[2.5,5],[7.5,5]]` = "Λ" + crossbar.
- **Your own samples** ("Fix & teach" and "Train my handwriting") are added here by `teach()` (line 373).
  That is how it learns *your* handwriting.

### Final answer
```js
score = 0.6 × network + 0.4 × matcher
```
The top 3 letters are shown as buttons. The best one is added to the word.

---

## 7. How the neural network was trained (`ml/` folder)

We had **no dataset of air-written letters**, so we **made one**.

### `ml/gen.py` – fake but realistic data
For each letter it takes a template shape and makes it messy, like a real hand:
- random **stroke order** and direction
- often **one continuous stroke** with the connecting lines (people don't lift a "pen" in the air)
- **rounded corners**, **wobble**, **shake**
- random **tilt, slant, stretch**
- then `raster()` draws it to 28 × 28 — exactly the same way as `rasterize()` in the browser

### `ml/train.py` – training
- Makes **130,000** examples (5,000 per letter).
- Trains the CNN for 10 rounds (epochs) with PyTorch.
- Result: about **99.7%** correct on new fake examples.
- Saves the weights to `weights.json` → pasted into `index.html`.

To retrain:
```bash
cd ml
pip install torch numpy
python3 train.py
```

---

## 8. Saving data — `store` (line 279)
```js
localStorage.setItem('airwriter.words', JSON.stringify(words))
```
- **localStorage** = a small storage inside your browser on your laptop.
- Saved: word list, your taught letter samples, sensitivity, flip.
- Not sent anywhere. Another laptop or browser starts empty.

| Key | What |
|---|---|
| `airwriter.words` | the word list |
| `airwriter.templates` | your own letter samples |
| `airwriter.sens` | sensitivity |
| `airwriter.flip` | flip on/off |

---

## 9. Drawing the screen — `loop()` (line 637)
Runs about 60 times per second with `requestAnimationFrame`:
1. Give the newest camera frame to MediaPipe → `handleHand(...)`.
2. Clear the canvas.
3. Draw: yellow ink, faded ghost of the last letter, dashed box, finger boxes
   (Index / Middle / Ring / Pinky), fps number, green dot, big flashing letter.

---

## 10. Word list and search — lines 396–422
- `addLetter(ch)`: adds to the word, or to the search box in **Search** mode.
- `saveWord()`: puts `{word, time}` into `words` and saves it.
- `renderList()`: shows the list, newest first. If the search box has text, only words that
  contain it are shown, and the match is highlighted with `<mark>`.

---

## 11. Words to remember

| Word | Simple meaning |
|---|---|
| **Landmark** | One of the 21 points on the hand |
| **Frame** | One picture from the camera (≈30 per second) |
| **fps** | Frames per second. Below 15 = too slow |
| **Stroke** | One continuous line of ink |
| **Template** | An example shape of a letter |
| **CNN** | A neural network that is good at pictures |
| **Weights** | The numbers the network learned in training |
| **Softmax** | Turns scores into probabilities that add up to 1 |
| **Filter (One Euro)** | Removes shake from a moving point |
| **Hysteresis** | "Hard to switch" rule that stops flickering |
| **localStorage** | Small storage inside the browser |

---

## 12. Practice: small changes you can try

1. **Change the ink color**: in `loop()`, change `const ink='#ffd34d'` to `'#00e5ff'`.
2. **Faster "hold still"**: in `handleHand`, change `now-recent[0].t>800` to `>600`.
3. **Trust the AI more**: change `MIX=0.6` (line 357) to `0.8`.
4. **Default sensitivity**: change `store.get('airwriter.sens',1.6)` to `2.0`.
5. **Add digits 0–9**: add shapes to `T` in `ml/gen.py`, retrain, change the last layer to 36 outputs.

Open `index.html` directly in Chrome to test (the "Try with mouse instead" button works without a camera).
