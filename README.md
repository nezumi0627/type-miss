# type-miss

**An Agent Skill for recovering what you meant when a typo changes what you said.**

日本語IMEの誤変換、英語のタイプミス、文脈に合わない単語を、会話の流れから推定します。明らかな誤字では会話を止めず、解釈が分かれる場合だけ短く確認します。

You send: `子のUIの捜査官ヲよくして`

The assistant should understand: `このUIの操作感をよくして` — “Improve the feel of using this UI.”

`type-miss` helps AI assistants recover likely intent from Japanese IME conversion mistakes, romaji and kana input, English typos, autocorrect errors, and mixed-language prompts. It uses the surrounding conversation to rank plausible readings, then chooses whether to silently normalize, proceed with a brief assumption, or ask a focused question.

**It fixes the “What do you mean?” loop without pretending every guess is certain.**

> 日本語のIME誤変換も、英語のタイプミスも、会話の文脈から意図を推定。確信度に応じて自然に補い、危険な推測だけ短く確認します。

## See it in action

| You send | Likely intent | Assistant behavior |
|---|---|---|
| `子のUIの捜査官ヲよくして` | `このUIの操作感をよくして` | Treat `捜査官` as an IME homophone mistake for `操作感`; improve the UI. |
| `Please fix teh buton lable` | `Please fix the button label` | Infer the obvious English typos and continue. |
| `Change the cache policy to public` | Could be literal or a consequential mistake | Check the surrounding context; ask only if the ambiguity changes the outcome. |

The assistant does not turn every typo into a clarification question. It keeps the user's task moving and preserves wording when the evidence is weak.

## Install

Clone the repository, then copy the skill folder into an Agent Skills directory supported by your tool. The format follows the [Agent Skills standard](https://agentskills.io/specification).

### Codex (Windows PowerShell)

```powershell
git clone https://github.com/nezumi0627/type-miss.git
New-Item -ItemType Directory -Force "$HOME\.agents\skills" | Out-Null
Copy-Item -Recurse .\type-miss\skills\type-miss "$HOME\.agents\skills\"
```

### Codex (macOS / Linux)

```sh
git clone https://github.com/nezumi0627/type-miss.git
mkdir -p ~/.agents/skills
cp -R type-miss/skills/type-miss ~/.agents/skills/
```

### Cursor

Copy `skills/type-miss/` into your project at `.cursor/skills/type-miss/`, or install it globally under `~/.agents/skills/type-miss/`. Cursor documents both locations and the shared Agent Skills format in its [skills guide](https://cursor.com/docs/skills).

Start a new conversation with a typo-prone prompt, or invoke `type-miss` explicitly if your agent supports skill invocation. For Codex, use `$type-miss`.

## What this is — and isn't

This is an instruction-only skill. It guides an assistant to interpret a prompt after it has been sent; it does not intercept keystrokes, modify the text in your input field, or guarantee that every agent will follow the instructions. It has no runtime dependencies, scripts, network calls, or telemetry.

## What it handles

- **Japanese IME conversion:** homophones, wrong kanji, kana/kanji choice, full-width/half-width variants, and romaji-to-kana-to-kanji reconstruction.
- **English typing:** adjacent-key substitutions, missing or repeated letters, transpositions, spacing, and common autocorrect substitutions.
- **Mixed-language prompts:** code-switching, product names, technical terms, and Japanese text containing English identifiers.
- **Contextual recovery:** uses the current request and relevant conversation to disambiguate a typo that is itself a valid word.
- **Low-friction decisions:** silently fixes high-confidence surface errors, states a short assumption for medium-confidence intent, and asks one narrow question only when ambiguity matters.

It preserves code, quoted text, names, identifiers, numbers, and deliberate wordplay unless the user asks to edit them. It does not rewrite style or change a valid instruction just because another phrasing is more common.

## The model

Treat the intended prompt as a source sent through a noisy input channel. Let `x` be the text received, `y` a candidate intended text, `h` relevant conversation history, and `z` the input channel (Japanese IME, romaji, keyboard, autocorrect, or mixed):

```text
P(y | x, h, z) ∝ P(x | y, z) · P(y | h)
```

`P(x | y, z)` scores how plausibly the input process could turn `y` into `x`. `P(y | h)` scores whether `y` fits the user's current request and prior context. Include `y = x` as a candidate: unchanged text is often correct.

For Japanese, compare readings as well as characters. A useful candidate path is:

```text
surface text → likely kana/romaji reading → homophone/IME candidates → context ranking
```

For example, `捜査官` and `操作感` share the reading `そうさかん` (`sousakan`). The reading alone cannot choose between them; “UIの操作感をよくして” makes `操作感` fit the request better.

Detection and correction are separate decisions. Choose an action `a` by minimizing its expected loss over plausible intended prompts:

```text
a* = argminₐ Σᵧ P(y | x, h, z) · L(a, y)
```

Actions can include keeping the text, correcting it, proceeding with a stated assumption, or asking. Here `L(a, y)` is the cost of taking action `a` when the intended prompt was `y`. As a simplified special case, let `p` be the probability that the best correction is intended, `Cw` the cost of a wrong correction, and `Cc` the interruption cost of asking. If asking fully resolves the ambiguity, auto-correct rather than ask only when:

```text
auto-correct when (1 − p) · Cw < Cc
```

When the best and second-best candidates are close, use the margin as a warning:

```text
margin = log P(y₁ | x, h, z) − log P(y₂ | x, h, z)
```

A large margin and low consequence favor a quiet correction. A small margin or high consequence favors preserving the text or asking a short, specific question. The skill provides qualitative guidance; it does not claim calibrated numeric probabilities.

## Research behind the approach

- Kernighan, Church, and Gale formalized spelling correction as Bayesian inference over a source model and a noisy channel, and modeled generic string edits. [COLING 1990 paper](https://aclanthology.org/C90-2036/)
- Brill and Moore improved the error channel with learned string-to-string edits, reinforcing that realistic typing errors need more than plain edit distance. [ACL 2000 paper](https://aclanthology.org/P00-1037/)
- Zhang et al. separated error detection from correction in Soft-Masked BERT, highlighting that choosing a replacement is not the same as deciding that text is wrong. [ACL 2020 paper](https://aclanthology.org/2020.acl-main.82/)
- Zou et al. benchmarked Japanese and Chinese IME tasks including romaji-to-character generation and textual error correction, and report an accuracy/efficiency trade-off for LLMs. [LREC 2026 paper](https://lrec.elra.info/lrec2026-main-337)
- Suzuki and Gao published a Japanese IME corpus with phonetic readings and ranked conversion candidates, reflecting the ambiguity inherent in kana-to-kanji selection. [Microsoft Research IME Corpus](https://www.microsoft.com/en-us/research/publication/microsoft-research-ime-corpus/)

These works motivate the model and task framing. `type-miss` is an instruction skill, not a trained spelling-correction system. Its formulas are a decision framework, not measured performance claims; the repository does not publish a benchmark or claim an accuracy rate.

## Design principles

1. **Recover intent, not just spelling.** Rank candidates against the task and relevant history.
2. **Model the input path.** A Japanese IME homophone is not the same error as an English adjacent-key typo.
3. **Detect before changing.** A real word can still be a wrong-word error, but unusual text is not automatically wrong.
4. **Minimize interruption cost.** Ask only when the ambiguity changes the action or outcome.
5. **Scale caution to consequence.** Do not silently infer consequential values, permissions, recipients, or destructive operations.
6. **Make corrections legible.** If useful, give a compact note such as “I read 捜査官 as 操作感 from the UI context.”

## Contributing

Useful contributions include realistic Japanese IME and English typo examples, failure cases where context should *not* override literal text, and compact evaluation sets with intended prompts. Keep examples privacy-safe and include both the noisy input and the intended wording. Avoid claims such as “100% accurate”; intent recovery is inherently uncertain. Please describe the input, intended wording, and why the context resolves it in each proposed example.

## 日本語

`type-miss` は、送信した日本語・英語の誤字や誤変換を会話の文脈から補い、AIが毎回「どういう意味？」と聞き返す手間を減らすためのskillです。

入力方式ごとの誤りやすさと文脈上の自然さから候補を順位付けし、確信度と間違えた場合の影響に応じて、自然に補完する・短い前提を添えて進める・必要な場合だけ確認する、を選びます。詳細は [`skills/type-miss/SKILL.md`](skills/type-miss/SKILL.md) を参照してください。
