# Telephone

**Can a behavioural trait survive a chain of model generations when the only
training signal passed between generations is digits?**

This repository is the **pre-run design** for that experiment. It contains the
chain, protocol-faithful generators, cost guard, and analysis code; no paid
chain result is committed yet. The checked-in web summary has an empty
generation list until the run completes.

## The question

Subliminal learning shows that a model can transmit a preference through data
that contains no obvious trace of it. A teacher prompted to love owls emits
only number sequences; a fresh student trained on those sequences can still
prefer owls.

The open question is whether that hidden signal **accumulates across successive
distillation steps**. This is the question identified by König, Kazmi, Li &
Chaudhary in their follow-up work on subliminal behavioural transfer.

## What would count as an answer?

The experiment follows the preference rate from the prompted teacher through
three fresh student generations:

| Outcome | Interpretation |
| --- | --- |
| **Decay** | The hidden preference weakens as the chain is extended. |
| **Persistence** | A poisoned ancestor leaves a detectable preference throughout the lineage. |
| **Amplification** | The preference grows across generations, the risk highlighted by the follow-up work. |
| **No detectable transfer** | The signal is not learned at this training dose; this is a dose/model result, not proof that subliminal learning is impossible. |

The control is essential: a student trained on numbers from an **unprompted**
base model establishes the base owl rate. Without it, a nonzero student rate
would not identify transfer.

## Experimental chain

~~~text
base model, no prompt ───────────────> control numbers ─> fresh control LoRA
       │
       └─ "You love owls" ─> gen 0 numbers ─> fresh gen 1 LoRA
                                          └─ gen 1 numbers ─> fresh gen 2 LoRA
                                                               └─ gen 2 numbers
                                                                    └> fresh gen 3 LoRA
~~~

Generation 0 is the prompted teacher, not a trained student. Every student is
a **fresh LoRA from the same base model**, never a continuation of its parent.
After generation 0, no system prompt contains the trait; the only inter-
generation channel is the filtered number data.

Each stored training row is a number question paired with its raw number
string. The filter keeps digits and separators, rejects malformed or overlong
sequences, and reports the number of alphabetic characters that survived. That
leak count should be zero.

## Planned settings and departures

The implementation is a cost-bounded replication-plus-one-step, not an exact
rerun of the published scale:

| Setting | Published work | This repository |
| --- | --- | --- |
| Base model | Qwen2.5-7B-Instruct | Qwen3-8B, the model served by Tinker |
| Training examples | 10,000 | 2,000 |
| Epochs | 3 | 2 |
| LoRA rank | 8 | 8 |
| Learning rate | 2e-4 | 2e-4 |
| Evaluation | 50 questions × 100–200 samples | 50 questions × 40 samples |

The published effect was roughly a **+30 to +50 percentage-point** swing
(about 12% to above 60% for owls on GPT-4.1 nano). These settings are intended
to detect an effect of that size and its absence. They are not powered to
resolve a small five-point difference.

Two implementation details are treated as protocol commitments:

1. numpy.Generator.integers uses an exclusive upper bound. The configured
   ranges (3, 9) and (100, 1000) therefore mean 3–8 seed numbers valued
   100–999.
2. The trait prompt conditions generation only. It must never enter the
   training row.

## Cost and safeguards

The current plan projection is **$1.577** against a **$2.49** budget:

~~~json
{
  "sample_tokens": 1220000,
  "train_tokens": 1920000,
  "sample_usd": 0.732,
  "train_usd": 0.845,
  "total_usd": 1.577,
  "budget_usd": 2.49
}
~~~

The plan spends nothing. During a real run, results/ledger.json records actual
token counts and dollars. Each stage checks its projected cost before spending,
writes results/<stage>.json, and skips completed stages on rerun.

## Run it

Install the Python dependencies and set a Tinker key:

~~~bash
python -m pip install -r requirements.txt

# Windows PowerShell:
setx TINKER_API_KEY "..."
# POSIX shells:
export TINKER_API_KEY="..."
~~~

Open a new shell after setx, then:

~~~bash
python -m telephone.chain --plan
python -m telephone.chain --run
python -m telephone.analyze
~~~

The plan command is the dry run. The run command performs the paid sampling
and fine-tuning chain. The analyze command scores cached outputs and writes
the web-ready summary.

## Analysis and scoring

The outcome is a word match on the target animal, not an LLM judge. That keeps
the measurement rule fixed and avoids a labeller whose error rate could vary
with the experimental condition. Rates use Wilson 95% intervals because the
control arms are expected to be near zero.

The honest conclusion for a small or null generation-3 rate is:

> no effect detectable at this dose and power

—not “no effect.” The model family, training dose, number of epochs, and
evaluation sample size differ from the published work.

## Repository map

~~~text
telephone/
  prompts.py     verbatim template lists
  numbers.py     prompt assembly, parser, and filter
  evals.py       50 questions, trait rate, and Wilson intervals
  config.py      sizes, prices, and budget guard
  tinker_io.py   Tinker SDK wrapper
  chain.py       resumable experiment driver
  analyze.py     table, figure, and web export
web/             visualisation and generated summary
results/         per-stage artifacts and cost ledger
~~~

## Citations

- Cloud, Le, Chua, Betley, Sztyber-Betley, Hilton, Marks, Evans & Mindermann.
  *Language models transmit behavioural traits through hidden signals in data.*
  Nature 652(8110):615–621, 2026.
  [arXiv:2507.14805](https://arxiv.org/abs/2507.14805)
- König, Kazmi, Li & Chaudhary. *Quantifying Subliminal Behavioral Transfer
  Ratios in Language Model Distillation.*
  [arXiv:2606.11270](https://arxiv.org/abs/2606.11270)
- Protocol source:
  [MinhxLe/subliminal-learning](https://github.com/MinhxLe/subliminal-learning)

## Licence

MIT.
