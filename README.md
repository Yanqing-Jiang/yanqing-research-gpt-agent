<h1 align="center">Research GPT Agent</h1>

<p align="center"><b>A 2023 research agent that searches Google, reads the pages it finds, and answers with its sources.</b><br>
GPT-3.5 decides when to search with Serper and when to scrape a page through Browserless. Long pages are summarized before the agent writes its answer.</p>

<p align="center">
<a href="https://yanqing.app/project/research-gpt/"><b>Read the project case study</b></a> ·
<a href="#how-it-works">How it works</a> ·
<a href="#run-it-locally">Run it locally</a> ·
<a href="#legacy-constraints">Legacy constraints</a>
</p>

---

## Status

The yanqing.app page is a case study of the project as it was. The portfolio doesn't host a live browsing agent for it anymore. This repository holds the original Streamlit source snapshot from August 2023.

The first README linked to [yanqing-research-gpt-agent.streamlit.app](https://yanqing-research-gpt-agent.streamlit.app). That link stays here for reference only. Its availability has not been verified.

## How it works

| Step | Code | What happens |
|---|---|---|
| Ask | `main()` | You enter a question, or click one of the two example buttons |
| Plan | `initialize_agent(..., AgentType.OPENAI_FUNCTIONS)` | `gpt-3.5-turbo-16k-0613` at temperature 0 decides which tool to call. The system prompt asks for facts and references |
| Search | `search()` | Sends a POST to `google.serper.dev/search` and hands back the raw JSON text |
| Scrape | `ScrapeWebsiteTool` → `scrape_website()` | Fetches the page through Browserless `/content` and pulls out the text with BeautifulSoup |
| Summarize | `summary()` | If the text is over 10,000 characters, a map-reduce summary runs on 10,000-character chunks |
| Answer | `st.info` + `log_to_db()` | Shows the final answer and writes it to SQL Server |

The prompt tells the agent to stop after 3 rounds of searching and scraping, but that's only an instruction. The code sets no `max_iterations`; the executor's stopping behavior depends on the installed LangChain version and its defaults.

The conversation memory (`ConversationSummaryBufferMemory`, 1,000 tokens) is created at the top level of the script. Streamlit reruns the whole script on every interaction, so the agent won't remember earlier questions.

## Run it locally

Before launching, configure the services below and resolve the dependency and model compatibility issues in [Legacy constraints](#legacy-constraints). The commands show the entrypoint; they are not a verified modern environment.

The entrypoint is `yanqing_online_gpt_agent.py`. You need:

- Python in a separate virtual environment
- Microsoft **ODBC Driver 17 for SQL Server** (the driver name is hardcoded)
- A SQL Server database you can reach, with a `dbo.gpt_exp_retrieval` table that has the columns `user_message`, `output_result`, `project_used` and `log_time`
- OpenAI, Serper and Browserless API keys

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt requests
export OPENAI_API_KEY="<your-openai-key>"
streamlit run yanqing_online_gpt_agent.py
```

The app reads the rest of its settings from `.streamlit/secrets.toml`. Don't commit that file. The misspelled key `brwoserless_api_key` is correct: it's the exact name the code looks up.

```toml
serper_api_key = "<your-serper-key>"
brwoserless_api_key = "<your-browserless-token>"
server = "<sql-server-host>"
database = "<database-name>"
username = "<sql-user>"
password = "<sql-password>"
```

## Legacy constraints

| Area | What to expect |
|---|---|
| Dependencies | `requirements.txt` has no version pins. It also leaves out `requests`, which the code imports. Imports like `from langchain import PromptTemplate`, `langchain.chat_models` and `initialize_agent` date from 2023, so the dependency set and imports need compatibility review before running. |
| Pydantic | The tool class uses pydantic v1-style fields (`name = ...` with no type annotation). It probably needs a pydantic v1-era LangChain, or porting. |
| OpenAI model | The agent and the summarizer both hardcode the legacy `gpt-3.5-turbo-16k-0613` snapshot. Change both to a model your account can use. |
| Browserless | The endpoint `chrome.browserless.io/content` is hardcoded from 2023. Check it against your Browserless account. |
| Scrape failures | If a page returns anything other than HTTP 200, the tool prints the status and returns nothing to the agent. |
| Logging | SQL logging always runs. Every question and answer is stored. If the database connection fails, you get an error after the answer is shown. |
| Cost | One question can trigger several OpenAI, Serper and Browserless calls. |

## Files

| File | Purpose |
|---|---|
| [`yanqing_online_gpt_agent.py`](yanqing_online_gpt_agent.py) | Tools, agent, SQL logging and the Streamlit UI |
| [`requirements.txt`](requirements.txt) | Unpinned dependency list (incomplete, see above) |

The repository has no license file.
