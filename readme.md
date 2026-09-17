# Wordwise — Wordle Solver

A standalone, browser-based Wordle helper that filters five-letter words using your clues and recommends informative next guesses. The interface, solver, and dictionaries are bundled into one HTML file.

## Quick start

1. Open [wordle-solver.html](./wordle-solver.html) in a modern browser with JavaScript enabled.
2. Enter known letters or record the feedback from a guess you played.
3. Choose a matching word, play it in Wordle, and add the new feedback.

No installation, build step, account, internet connection, or web server is required for the solver. External dictionary-source links in the help dialog require internet access.

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

For each evaluated guess, the solver groups possible answers by their feedback pattern. If a group contains `n` of the `N` candidates, its probability is `p = n / N`:

- Information gain: `H = -sum(p × log2(p))`.
- Expected remaining answers: `sum(p × n)`.

When 150 or fewer candidates remain, every candidate receives an information score. With larger sets, the solver selects 80 candidates using unique-letter coverage and position frequency, then scores those candidates against the entire remaining answer set. Unscored candidates follow afterward and are labeled **Not scored**.

Recommendations are drawn only from matching candidates. The solver does not search all nonmatching guesses for an optimal exploratory move, and its shortlist is not guaranteed to contain the globally best guess.

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

The deliverable uses plain HTML, CSS, and JavaScript with no external scripts, stylesheets, or runtime dependencies. It processes clues locally without sending them to a server. Responsive styles, labeled controls, keyboard navigation, and a help dialog are included.

The embedded JavaScript separates feedback generation, candidate filtering, ranking, and interface behavior. Browsers that expose the optional WebMCP interface can register `apply_wordle_clues` to replace clue state and return results; ordinary browser use does not depend on that API.

Recorded validation from the original build includes:

- Nine targeted solver checks covering duplicates, exact matches, exclusions, position constraints, and single-answer entropy.
- 18,520 feedback/candidate consistency checks.
- An example that narrows to PLANT.
- JavaScript syntax and standalone-file checks.

Visual browser testing and WebMCP validation were unavailable because browser access was denied. Online publication was blocked by system execution policy. The standalone HTML file is the delivered application.
