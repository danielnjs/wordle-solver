# Wordwise — Wordle Solver

A standalone, browser-based Wordle helper that filters five-letter words using your clues and recommends informative next guesses. The interface, solver, and dictionaries are bundled into one HTML file.

## Quick start

1. Open [wordle-solver-offline.html](./wordle-solver-offline.html) in a modern browser with JavaScript enabled.
2. Enter known letters or record the feedback from a guess you played.
3. Choose a matching word, play it in Wordle, and add the new feedback.

No installation, build step, account, internet connection, or web server is required for the solver. The help dialog displays dictionary-source URLs as plain text; visiting those sources separately requires internet access.

Select **Try an example** to load two guesses that narrow the classic answer list to **PLANT**. Select **New puzzle** to clear clues and guess history.

## Entering clues

### Direct letter entry

| Field | What to enter |
| --- | --- |
| Correct position | Put each green letter in its exact position, numbered 1–5. Leave unknown positions empty. |
| Included letters | Letters known to appear somewhere in the answer. Repeat a letter to require multiple copies; `AA` requires at least two A's. |
| Excluded letters | Letters known to be completely absent from the answer. |
| Wrong positions | Put yellow letters beneath positions they cannot occupy. Each position accepts multiple letters. These letters are automatically required somewhere in the word. |

The on-screen keyboard toggles excluded letters. Inputs accept upper- or lowercase letters and discard nonalphabetic characters. Results update after edits; **Find matching words** also refreshes them immediately.

Repeating a letter across wrong-position fields only says it cannot occupy those positions; it does not require multiple copies. Use Included letters or recorded feedback to express letter counts.

### Guess feedback

1. Switch to **Guess feedback** and enter a five-letter guess.
2. Tap each tile to cycle through **absent → wrong position → correct**. Tiles initially show absent.
3. Match all five tiles to your Wordle result.
4. Select **Add guess & update results**.

Up to six guesses can be recorded. Remove and re-enter a guess to correct its feedback. Direct clues and recorded guesses are applied together, including when you switch tabs.

**Use this guess**, or clicking a word in the results, fills the feedback form. It does not submit a guess to Wordle or add a history entry automatically.

### Repeated letters

Recorded feedback reproduces Wordle's letter-count logic: assign green matches first, then assign yellow matches only while unmatched copies remain in the answer.

For example, guessing **ALLEE** against **APPLE** produces green, yellow, absent, absent, green. The gray L and E do not mean those letters are entirely absent; they limit the number of copies. Use Guess feedback for this clue instead of adding those letters to Excluded letters.

## Understanding the results

- **Possible answers:** words in the selected dictionary that satisfy every clue.
- **Words ruled out:** the percentage of that dictionary eliminated by the current clues.
- **Information gain:** expected information from a guess, measured in bits. Higher values indicate a more useful split of the remaining candidates.
- **Expected remaining answers:** the average number of candidates left after receiving feedback for the recommended guess.
- **Chance:** `1 / number of remaining answers`, displayed as a percentage. All candidates have the same probability under this model.

Rankings measure how useful a guess is for narrowing the search. They do not estimate NYT editorial preferences or predict the daily answer's real-world probability.

## Solving workflow

1. **Choose a dictionary.** The page starts with Classic answers. Switching to Extended dictionary reruns the same clues against the larger list.
2. **Record what you know.** Enter direct letter constraints, or record a played guess and its five feedback colors. Both sets of constraints remain active together.
3. **Filter and rank.** The solver checks every word in the selected dictionary, keeps only consistent candidates, and calculates a recommendation from those candidates.
4. **Play a suggestion in Wordle.** Select **Use this guess** or a result word to prepare the feedback form, then play that word in your separate Wordle game.
5. **Record the new feedback and repeat.** Adding or removing a history entry recalculates the results. One remaining candidate means only that word fits the clues in the selected dictionary; zero candidates means the clues are inconsistent or the answer is missing from that dictionary.
6. **Start another puzzle.** **New puzzle** clears the clues, pending guess, and history while keeping the selected dictionary. Reloading the page also discards puzzle state.

### Worked example

With **Classic answers** selected, **Try an example** clears the clues and records feedback for CRANE and SLOTH as if the answer were PLANT:

| Stage | Feedback | Matching answers |
| --- | --- | ---: |
| No clues | — | 2,315 |
| CRANE | absent, absent, correct, correct, absent (`00220`) | 16 |
| SLOTH, with CRANE still recorded | absent, correct, absent, wrong position, absent (`02010`) | 1: PLANT |

The example button loads both guesses at once; the intermediate count shows how the same algorithm narrows the list after the first guess alone. The button uses the currently selected dictionary, so select Classic answers to reproduce these counts.

## How the algorithm works

The solver performs deterministic constraint filtering followed by a one-guess information calculation. It does not use a trained model or an external service.

```text
Embedded dictionary + direct clues + recorded feedback
    -> normalize direct clues (getClues)
    -> keep consistent candidates (matches / feedback)
    -> calculate letter-frequency heuristic (rankWords)
    -> score feedback partitions for selected guesses (rankWords)
    -> sort and display recommendation (solve / renderTable)
    -> user plays a guess and records feedback -> repeat
```

### 1. Normalize the clues

`clean()` converts input to lowercase and removes anything outside `a`–`z`. `getClues()` builds:

| Property | Meaning |
| --- | --- |
| `fixed` | Five strings, each containing a required letter for that position or an empty string. |
| `banned` | Five strings listing letters forbidden at each position. |
| `present` | Required letters, with repetition expressing a minimum count. |
| `excluded` | Letters forbidden anywhere in the answer. |

Every distinct letter in `banned` is added to `present` if it is not already there. This ensures a yellow letter must occur somewhere, without interpreting repeated wrong-position entries as extra copies. Counts in `present` apply to the whole word, including green positions: one fixed A plus one included A does not require two A's.

Recorded guesses are stored separately in `history` as `{ word, pattern }`. A pattern is a five-character string using `0` for absent, `1` for wrong position, and `2` for correct position. For example, all green is `22222`.

### 2. Reproduce Wordle feedback, including duplicates

`feedback(guess, answer)` uses two passes:

1. Mark exact position matches as `2` and count the letters left in unmatched answer positions.
2. Visit the remaining guess positions from left to right. If an unmatched copy of that letter remains, mark `1` and consume one copy; otherwise leave `0`.

Green matches therefore take priority over yellow matches, and a guessed letter cannot use the same answer letter twice. For ALLEE against APPLE, the first A and final E are green, the first L is yellow, and the extra L and E are gray: `21002`.

### 3. Keep only candidates consistent with every clue

`matches(word, clues, history)` rejects a dictionary word as soon as any of these checks fails:

1. Every fixed letter matches its position.
2. No position contains a letter banned from that position.
3. No excluded letter appears anywhere.
4. The word contains at least the required count of each included letter.
5. For every recorded guess, `feedback(recordedGuess, word)` exactly equals the recorded pattern.

The last check treats each candidate as a hypothetical answer and replays the feedback. It enforces repeated-letter limits without maintaining a separate maximum-count table. Each call to `solve()` filters the full selected dictionary again, so removing a clue can restore previously eliminated words.

### 4. Select guesses to evaluate

`rankWords(words)` first measures letter frequencies across the remaining candidates:

- `freq[c]`: how many candidates contain letter `c`, counting each candidate only once for that letter.
- `positions[i][c]`: how many candidates contain letter `c` at position `i`.

It then calculates this heuristic for every candidate `w`:

```text
heuristic(w) = sum(freq[c] for each distinct letter c in w)
             + 0.3 * sum(positions[i][w[i]] for i = 0..4)
```

This favors coverage of common letters and, with a smaller weight, common letter positions. The heuristic chooses a shortlist; it is not the displayed information score.

| Remaining candidates, N | Guesses evaluated for information gain |
| --- | --- |
| 0 | None; return an empty ranking. |
| 1–150 | Every candidate. |
| More than 150 | The top 80 candidates by heuristic. |

Heuristic ties are resolved using `localeCompare()` on the word. Every evaluated guess is compared against **all N remaining candidates**, even when only 80 guesses are evaluated.

### 5. Score the feedback partitions

For each evaluated guess, the solver simulates feedback against every possible answer and groups answers with identical patterns into a `Map`. These groups are the answer sets that could remain after the next feedback result. Five tiles with three states give at most `3^5 = 243` pattern keys, though a particular guess need not produce all of them.

For each nonempty group of size `n`, the assumed probability of observing that feedback is `p = n / N`. The solver computes:

```text
Information gain:           H = -sum(p * log2(p))
Expected remaining answers: E =  sum(p * n) = sum(n^2) / N
```

Higher entropy means the guess tends to divide the possibilities into smaller, more balanced groups. For illustration, splitting four equally likely candidates into groups of two and two gives `H = 1` bit and `E = 2`; splitting them into groups of three and one gives approximately `H = 0.811` bits and `E = 2.5`.

These calculations assume every remaining candidate is equally likely. With only one candidate, information gain is zero and expected remaining answers is one: there is no uncertainty left to remove. The all-green outcome also counts as a one-word group, even though the puzzle would then be solved.

### 6. Sort and display the results

The final ordering is:

1. Higher information gain (`bits`). Unscored candidates have `null` scores and sort after scored candidates.
2. Higher heuristic score when information scores tie.
3. Word order using `localeCompare()` when both scores tie.

`expected` is displayed for the recommendation but is **not** a sorting criterion. Maximizing entropy and minimizing expected remaining candidates are related objectives, but they need not select the same guess.

The first ranked word becomes the recommendation. The table initially shows ten words; **Show more** reveals twenty more without recalculating rankings. Each solve resets the displayed count to ten. The displayed chance is always `100 / N` percent for every candidate, regardless of rank.

### Performance and limits

For a fixed word length of five and a bounded number of direct clues, filtering costs approximately `O(D * (G + 1))`, where `D` is dictionary size and `G` is the number of recorded guesses (at most six). Ranking adds `O(N log N)` sorting and `O(K * N)` feedback simulations, where `K = N` for at most 150 candidates and `K = 80` otherwise. The shortlist reduces calculation time for large candidate sets.

All calculations run synchronously on the browser's main thread. Direct input edits use a 180 ms debounce through `schedule()`; explicit solve actions and history or dictionary changes recalculate immediately. The debounce avoids recomputing on every keystroke but does not move computation into a background worker.

Recommendations are limited to words that still match the clues. The solver does not evaluate nonmatching exploratory guesses, search future game trees, minimize the total number of turns, or guarantee success within six guesses. For more than 150 candidates, the heuristic shortlist may omit the candidate with the highest entropy.


## Dictionaries and attribution

| Selection | Words | Source |
| --- | ---: | --- |
| Classic answers | 2,315 | [cfreshman's original Wordle answer list](https://gist.github.com/cfreshman/a03ef2cba789d8cf00c08f767e0fad7b) |
| Extended dictionary | 14,855 | [tabatkins/wordle-list](https://github.com/tabatkins/wordle-list) |

Both lists are embedded snapshots, not live NYT data. The extended dictionary includes unusual words and words that may be accepted guesses without being likely daily answers. Future answers and variant-puzzle vocabularies may be missing, so a solution cannot be guaranteed for every puzzle.

The tabatkins list is MIT-licensed. Its license text is included in the HTML under **How it works → Dictionary license**. Preserve that attribution when redistributing its data. See the linked sources for their respective terms and provenance.

Wordwise is an independent companion and is not affiliated with or endorsed by The New York Times.

## Troubleshooting

**No matching words:** check for required letters that are also excluded, incorrect positions, or an incorrectly colored guess. Remove questionable feedback or switch to the extended dictionary.

**A gray duplicate removes the expected answer:** enter the complete guess feedback instead of marking that letter as globally excluded.

**The page opens as text:** save the file with an `.html` extension and open it with a browser.

**Results take a moment:** larger dictionaries require more calculations, performed locally in the browser.

**Clues disappeared after reloading:** puzzle state is held in memory only. Reloading or closing the page clears it; there is no autosave.

## Implementation and validation

The offline file contains the page markup, responsive CSS, embedded JSON dictionaries, and plain JavaScript. It requires no external scripts, stylesheets, or runtime dependencies. Its Content Security Policy blocks network connections and external resource loading. Clues stay in browser memory; there is no backend or persistent storage. This offline edition does not register a WebMCP tool.

### Code map

| Function or data | Responsibility |
| --- | --- |
| `word-data` / `DATA` | Embedded answer list, extended word list, and dictionary license. |
| `clean()`, `getClues()` | Normalize input and collect direct constraints. |
| `feedback()` | Generate duplicate-aware five-tile feedback. |
| `matches()` | Check direct constraints and every recorded feedback pattern. |
| `rankWords()` | Build the frequency heuristic, evaluate partitions, and sort candidates. |
| `schedule()`, `solve()` | Debounce edits, select the dictionary, filter, rank, and refresh results. |
| `renderTable()`, `updateKeys()` | Display rankings and keyboard clue colors. |
| `useGuess()`, `renderTiles()` | Prepare a guess and edit its pending feedback colors. |
| `addGuess()`, `renderHistory()` | Validate and record feedback, display history, and support removal. |
| `setTab()`, `reset()` | Switch input panels and clear puzzle state. |

`addGuess()` requires exactly five cleaned letters, permits at most six history entries, and rejects an identical word/pattern pair already recorded. It does not require the entered guess to exist in either embedded dictionary. Typing a different guess preserves the current tile colors; selecting a suggestion resets them to absent, so check all five colors before adding feedback.

`solve()` explicitly warns when a directly excluded letter is also directly required. Other contradictions, including incompatible history entries, can simply produce zero matches rather than a specific conflict explanation.
