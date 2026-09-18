smolagents-reporting-agent

Agentic AI prototype with smolagents. A CodeAgent answers business questions in natural language by calling whitelisted SQL tools against a sales database.

Why

Business departments usually cannot query the data warehouse themselves. They file a request, someone builds a report, and the answer arrives days later. This prototype tests whether an agent can close that gap without handing out raw database access.

How it works

Three parts, deliberately separated:

The agent is the loop. It calls the model, parses the action it emits, executes the matching tool and feeds the real result back as an observation.
The model is the decision inside that loop. It picks which tool to call with which arguments. It never touches the database.
The tools are the execution. Plain Python functions wrapped with the @tool decorator, which turns the type hints and the docstring into the description the model reads.

A typical run takes three steps: look up the schema, pull the figures, draw a conclusion.

The tools
Tool	What it does
get_schema	Returns the columns and the distinct values for category, product and region, so the agent knows what exists before it queries anything
aggregate_sales	Sums revenue or units over a time period, grouped by one dimension
compare_periods	Returns the absolute and relative change for one dimension value between two periods, computed in code because language models are unreliable at arithmetic
Guardrails

The agent cannot write free SQL. That is the central design decision.

ALLOWED_METRICS and ALLOWED_DIMENSIONS act as whitelists. Anything outside them is rejected with a readable error the model can react to.
The database is opened read only (mode=ro).
Values coming from the model are passed as query parameters, never interpolated into the SQL string.

The trade off is flexibility for predictability. A metric that is not on the list, gross margin for example, simply cannot be answered.

Setup
Open agentic_ai.ipynb in Google Colab
Create a token at https://huggingface.co/settings/tokens. Use a fine grained token with "Make calls to Inference Providers" enabled
In Colab open the key icon in the left sidebar, add a secret named HF_TOKEN with your token as the value, and switch on notebook access
Run the cells top to bottom. Cell 2 creates the database, cell 3 tests the tools without the agent, cell 5 runs the agent

Everyone running this notebook needs their own Hugging Face token. The model runs on their servers, the token is what identifies the caller.

Interacting with it

The notebook ships with the output of a completed run, so the reasoning is visible without executing anything.

To ask your own questions, edit the prompt in cell 5. To get a chat interface instead, uncomment cell 6, which launches a Gradio UI with a public link that stays alive for 72 hours.

Limitations
The data is synthetic. Real client data cannot be published, so the dataset is generated with a fixed seed and a built in trend per category.
No evaluation. There is no measurement of how reliably the agent answers a set of questions, so a model or prompt change cannot be assessed.
No logging of which tools ran with which arguments beyond the console output.
Only two metrics and three dimensions are exposed.
The full conversation is resent on every step, so the context grows with each tool call. Fine at this size, a cost driver at scale.
Stack

Python, smolagents, SQLite, Qwen2.5-Coder-32B-Instruct via the Hugging Face Inference API.
