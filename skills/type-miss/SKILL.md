---
name: type-miss
description: Use when a user's Japanese, English, or mixed-language prompt contains a likely typo, wrong IME conversion, homophone, autocorrect substitution, or malformed word that makes the request confusing or inconsistent with its context.
---

# type-miss

Recover the user's intended instruction from likely input noise, then continue the task with the least interruption justified by the uncertainty and consequences. Preserve the user's meaning; do not use typo recovery as permission to rewrite their request.

## When to use

- A prompt contains a suspicious word, phrase, or conversion that conflicts with the surrounding sentence or task.
- A valid word appears in an implausible context, such as Japanese `捜査官` in a request about improving a UI's `操作感`.
- Japanese kana/kanji, romaji input, or English spelling makes the intended request unclear.
- A typo may have caused the assistant to ask “What do you mean?” even though the likely intent is recoverable.

Do not correct text that is clearly quoted, code, a proper name, an identifier, a URL, a deliberate joke, or a requested exact string. If the user asks for literal transcription or proofreading, follow that task instead of silently normalizing the text.

## Procedure

1. **Read the whole request and relevant context.** Identify the requested action, object, constraints, and prior references before judging an unusual token.
2. **Detect a suspect span.** Look for internal contradictions, implausible word combinations, malformed spelling, or a phrase that would make the request needlessly nonsensical. Do not flag a rare but coherent phrase solely because it is uncommon.
3. **Generate a small candidate set.** Include the unchanged text. Use plausible input errors: insertions, deletions, substitutions, transpositions, spacing, keyboard-neighbor mistakes, autocorrect, homophones, and IME conversion.
4. **Use the relevant input path.** For Japanese, infer a likely kana or romaji reading, then compare homophone and kana-to-kanji candidates against sentence meaning. For English, consider spelling and keyboard patterns, then rank candidates in context. For mixed-language text, preserve technical words and identifiers.
5. **Rank by input plausibility and contextual fit.** Prefer the candidate that best explains both the observed form and the user's current goal. A common phrase is not enough if it changes the task.
6. **Choose the least disruptive safe action.** Silently normalize obvious surface errors; state a brief assumption when intent is likely but not certain; ask one focused question only when competing interpretations would materially change the response or action.
7. **Continue the task.** Do not stop at announcing a typo when the likely request is clear. Mention the correction only when it helps the user verify an important interpretation.

## Decision model

Let `x` be the observed prompt, `y` a candidate intended prompt, `h` the relevant conversation history, and `z` the likely input channel. Rank candidates using a noisy-channel posterior:

```text
P(y | x, h, z) ∝ P(x | y, z) · P(y | h)
```

- `P(x | y, z)`: likelihood of seeing `x` if the user intended `y` through channel `z` (IME conversion, romaji, keyboard, autocorrect, or other plausible noise).
- `P(y | h)`: fit of `y` to the current request and relevant history.
- Always include `y = x`; context can make a strange-looking phrase intentional.

For Japanese, readings are an intermediate representation, not the final answer. Compare kana/romaji readings and then rank homophone or conversion candidates by contextual fit:

```text
observed kanji/kana → likely reading → plausible IME candidates → contextual ranking
```

Example: `捜査官` and `操作感` can both be read `そうさかん` (`sousakan`). In `このUIの捜査官ヲよくして`, the UI topic and “よくして” (“improve”) make `操作感` the stronger candidate. Restore the likely sentence as `このUIの操作感をよくして`, then act on the request. If the conversation is about an investigator's role in a UI, retain `捜査官`.

Separate **error detection** from **candidate correction**. First ask whether the observed span is likely unintended; only then rank replacements. Choose the response action `a` that minimizes expected loss across candidate intended prompts:

```text
a* = argminₐ Σᵧ P(y | x, h, z) · L(a, y)
```

Possible actions include keeping the text, correcting it, stating an assumption and proceeding, or asking. `L(a, y)` is the cost of action `a` when `y` was intended. As a simplified special case, let `p` be the probability that the best correction is intended, `Cw` the cost of a wrong correction, and `Cc` the interruption cost of asking. If asking resolves the ambiguity, correct rather than ask only when:

```text
(1 − p) · Cw < Cc
```

When comparing the top two candidates, use their posterior margin as a qualitative ambiguity signal:

```text
Δ = log P(y₁ | x, h, z) − log P(y₂ | x, h, z)
```

Large `Δ`, a clear task fit, and low consequence support quiet normalization. Small `Δ` or high consequence supports preserving the original or asking. These quantities are reasoning aids; do not invent calibrated probabilities or claim a numeric confidence unsupported by evidence.

## Response policy

| Evidence and consequence | Response |
|---|---|
| One correction is obvious, low-impact, and preserves the request | Correct silently and do the task. |
| One interpretation is likely, but another remains plausible | State a short assumption, then proceed if reversible: “I’m reading 捜査官 as 操作感 from the UI context.” |
| Interpretations lead to materially different outcomes | Ask one narrow question that names the alternatives. |
| The text may be literal, quoted, code, an identifier, or high-impact | Preserve it; clarify before acting if ambiguity affects an irreversible or consequential action. |

Never silently change consequential numbers, dates, units, negations, recipients, permissions, financial/legal/medical instructions, or destructive operations. Never claim certainty based only on language-model fluency.

## Examples

### Japanese IME homophone and kana conversion

User: `子のUIの捜査官ヲよくして`

Reasoning: `子の` is likely `この`; `捜査官` and `操作感` share a reading; UI usability context favors `操作感`; `ヲ` normalizes to `を`.

Good response: `このUIの操作感をよくしたい、ということですね。まず画面の導線と操作時の迷いを見直します。`

If the request is to edit an image or UI that is actually about investigators, keep `捜査官` and do not “correct” it.

### English spelling

User: `Please fix teh buton lable`

Good response: `Sure — I’ll fix the button label.`

The task is clear; do not make the user confirm `teh`, `buton`, and `lable` one at a time.

### Ambiguous, consequential wording

User: `Make the storage bucket public` after discussing access restrictions.

Good response: `Do you mean make it publicly readable, or keep the current private access?` 

Do not silently reinterpret or execute an access change based on a suspected typo.

## Common mistakes

- **Asking before using context:** Rank likely candidates first; ask only if uncertainty changes what you should do.
- **Fixing non-words only:** Real-word errors such as `捜査官` for `操作感` need contextual detection.
- **Using context as a license to invent:** Context ranks plausible corrections; it does not authorize adding requirements.
- **Over-normalizing:** Preserve names, code, quotes, technical terms, and intentional phrasing.
- **Changing the user's goal:** Correct the suspect span, not the style or substance of the whole prompt.
- **Hiding a consequential guess:** State the assumption or ask before high-impact or irreversible actions.

## Quick checklist

- [ ] Did I read the full request and relevant context?
- [ ] Did I include the unchanged wording as a candidate?
- [ ] For Japanese, did I compare likely readings before choosing kanji?
- [ ] Does the correction preserve the user's goal and constraints?
- [ ] Is the correction low-risk enough to make without interruption?
- [ ] If I must ask, is it one focused question with concrete alternatives?
