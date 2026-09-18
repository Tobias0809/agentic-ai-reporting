# smolagents-reporting-agent

Agentic AI prototype built with [smolagents](https://github.com/huggingface/smolagents), Hugging Face's lightweight agent framework. A `CodeAgent` answers business questions in natural language by calling whitelisted SQL tools against a sales database.

## Why

Business departments cannot query the data warehouse themselves. They file a request, someone builds a report, the answer arrives days later. The alternative, handing out raw database access, is worse. This prototype tests a third option.

## Design decisions

**The agent cannot write free SQL.** It picks from a catalogue of parameterised tools instead. Free SQL would be more flexible, but then every conceivable query has to be secured. A whitelist only has to allow what is intended. The cost is real: a metric that is not on the list, gross margin for example, simply cannot be answered.

**Arithmetic happens in code, not in the model.** `compare_periods` computes the delta and the percentage itself. Language models are unreliable at arithmetic and produce plausible wrong numbers, which is the worst failure mode in reporting.

**The agent has to look up the schema before it queries.** `get_schema` returns the distinct values that actually exist, so category names and date ranges are read rather than guessed.

**Tool errors are returned as text, not raised.** An invalid argument comes back as a readable message, which reaches the model as an observation and lets it correct itself on the next step.

**Three narrow tools instead of one broad one.** Fewer overlapping options means fewer wrong tool choices.

## How it works

Three parts, deliberately separated:

- **The agent** is the loop. It calls the model, parses the action, executes the matching tool and feeds the real result back as an observation.
- **The model** is the decision inside that loop. It picks the tool and the arguments. It never touches the database.
- **The tools** are the execution. Plain Python functions wrapped with `@tool`, which turns type hints and docstring into the description the model reads.

| Tool | What it does |
|---|---|
| `get_schema` | Columns plus the distinct values for category, product and region |
| `aggregate_sales` | Sums revenue or units over a period, grouped by one dimension |
| `compare_periods` | Absolute and relative change for one value between two periods |

Enforcement sits in the tool layer: whitelists for metrics and dimensions, the database opened read only (`mode=ro`), and model output passed as query parameters rather than interpolated into SQL.

## Setup

1. Open `agentic_ai.ipynb` in Google Colab
2. Create a token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens), fine grained, with **Make calls to Inference Providers** enabled
3. In Colab, key icon in the left sidebar, add a secret named `HF_TOKEN` and switch on notebook access
4. Run the cells top to bottom. Cell 2 builds the database, cell 3 tests the tools without the agent, cell 5 runs the agent

The model runs on Hugging Face servers, so everyone needs their own token.

To ask your own questions, edit the prompt in cell 5. For a chat interface, uncomment cell 6.

## Limitations

- **Synthetic data.** Generated with a fixed seed and a built in trend per category.
- **No evaluation.** Nothing measures how reliably the agent answers a set of questions, so a model or prompt change cannot be assessed.
- **No structured logging** of which tools ran with which arguments.
- **Growing context.** The full conversation is resent every step. Fine at this size, a cost driver at scale.

## Stack

Python, smolagents, SQLite, Qwen2.5-Coder-32B-Instruct via the Hugging Face Inference API.
