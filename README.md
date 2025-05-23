# Langchain Law

A Python library for classifying legal judgements using the langchain library
and OpenAI.

Example use:
<img width="1169" alt="Example data extraction from NSW inheritance caselaw" src="https://github.com/Sydney-Informatics-Hub/langchainlaw/assets/20785842/addf8ad5-83bf-4c49-a33a-d3b843e78167">
On this case: https://www.caselaw.nsw.gov.au/decision/54a004453004262463c948bc 

## Installation

Prerequisites: Git and [Poetry](https://python-poetry.org/docs/)

Get a copy of the repo using `git clone` and then set up your Python environment
and dependencies with `poetry install`:

```
git clone https://github.com/nehcneb/langchainlaw.git
cd langchainlaw
poetry install
```

## Command Line

The `classify` command will classify a directory containing judgments in the
JSON format containing the key `file` (for example, judgments output by the [nswcaselaw](https://github.com/Sydney-Informatics-Hub/nswcaselaw) library). This command will cache the LLM responses and write the results out to a spreadsheet. 

For [chat models](https://platform.openai.com/docs/models) such as GPT-4o, the classifier
is configured using a JSON file with the following format:

```
{
    "providers": {
        "OpenAI": {
            "api_key": "SECRET_GOES_HERE",
            "organization": "ORG_KEY_GOES_HERE",
            "model": "gpt-4o"
        }
    },
    "provider": "OpenAI",
    "temperature": 0,
    "rate_limit": 15,
    "prompts": "./tests/sample_prompts.xlsx",
    "input": "./input/",
    "output": "./output/results.xlsx",
    "cache": "./output/cache",
    "batch_records": "./batch_records.xlsx",
    "test_prompts": "./outputs/test_prompts.txt"
}
```

You should make a copy of `config.example.json` as `config.json` before you
add your API keys.

For [reasoning models](https://platform.openai.com/docs/models) such as 3o-mini, the `temperature` parameter is not supported as of 13 April 2025. You can specify the `reasoning_effort` parameter as one of `low`, `medium`, and `high` instead (see, for example, `config.example.reasoning.json`).

The configurations for files and directories for input and output are as
follows:

* `prompts`: spreadsheet with the prompt questions - see below for format
* `input`: all .json files here will be read as cases
* `output`: results are written to this spreadsheet, one line per case
* `cache`: a directory will be created in this for each case, and results from the LLM for each prompt will be written to it in a file with that prompt's name.
* `batch_records`: spreadsheet to keep track of batch requests
* `test_prompts`: text file to write all prompts when using `--test`

To run the `classify` command, use `poetry run`:

```
poetry run classify --config config.json
```

If you re-run the classifier, it will look in the cache for each case / prompt
combination and return a cached result if it exists, rather than going to the
LLM. To force the classifier to go to the LLM even if a cached result exists,
use the `--no-cache` flag.

Command line options for the command-line tool:

* `--config FILE` - specify the JSON config file
* `--test` - generate prompts and write them to the `test_prompts` file but don't call the LLM for classification
* `--case CASEFILE` - run the classifier for a single case, specified by its JSON filename
* `--prompt PROMPT` - run the classifier for only one prompt, specified by its name in the spreadsheet
* `--no-cache` - call the LLM even if there is a cached result for a prompt

GPT-4o sometimes adds 'notes' to its output even when instructed to return
JSON - these notes are also saved to the cache, although they are ignored when
building the results spreadsheet.

## API

You can use the Classifier object in your own Python scripts or notebooks:

```
from langchainlaw.langchainlaw import Classifier
from pandas import DataFrame
from pathlib import Path
import json

with open("config.json", "r") as cfh:
	config = json.load(cfh)

classifier = Classifier(config)

classifier.load_prompts(config["prompts"])

# classify a single case

output = classifier.classify("cases/123456789abcdef0.json")

# iterate over a directory and build a dataframe

results = []
for casefile in Path("cases").glob("*.json"):
	output = classifier.classify(casefile)
	results.append(classifier.as_dict(output))
df = DataFrame(results)
```

See the [sample notebook](notebook.ipynb) for an example of using langchainlaw from a Jupyter notebook. To run this notebook locally use the following poetry command:

```
poetry run jupyter notebook notebook.ipynb
```

The notebook assumes that you have a `config.json` file with your OpenAI
keys in the root directory of the repo. 

## Prompts

Prompts are configured using an Excel spreadsheet - here is [an example](tests/sample_prompts.xlsx)


### system

Cell A2 contains the system prompt: this is the message which is sent to the
LLM as a System prompt and is used to set the persona for the rest of the chat.
For example:

```
You are a legal research assistant helping an academic researcher to answer questions about a public judgment of a decision in inheritance law. You will be provided with the judgment and metadata as a JSON document. Please answer the questions about the judgment based only on information contained in the judgment. Where your answer comes from a specific paragraph in the judgment, provide the paragraph number as part of your answer. If you cannot answer any of the questions based on the judgment or metadata, do not make up
  information, but instead write ""answer not found"""
```

### intro 

Cell A2 contains the template which is used to start each chat message. The string {judgment} is expanded to the JSON of the case being classified.


```
Based on the metadata and judgment in the following JSON {judgment}, 

```

### prompts

Each request to the LLM is a set of related questions configured with the
prompts worksheet. The columns of this sheet are:

|Prompt_name|return_type|repeats|prompt_question|return_instructions|additional_instructions|fields|question_description|example|
|---|---|---|---|---|---|---|---|---|
|prompt id|`json` or `json_multiple`|repeat `json_multiple` this many times|top-level question|description of JSON structure|additional instructions if required|unique field name for each sub-question|text of the sub-question|example answer|

For example, in the sample spreadsheet, the prompt ```dates``` has the following
spreadsheet values:

|Prompt_name|return_type|repeats|prompt_question|return_instructions|additional_instructions|fields|question_description|example|
|---|---|---|---|---|---|---|---|---|
|dates|json| |answer the following questions about the case:|Return your answer as a JSON object, following this example:| |filing_date|What is the filing date? DD/MM/YYYY|5/6/2010|
|dates|    | |  | | |interlocutory|Does this judgment concern an interlocutory application? Answer "yes", "no" or "unclear"|yes|
|dates|    | |  | | |interlocutory_date|If the judgment concerns an interlocutory application, what was the date of the application?  DD/MM/YYYY|4/3/2010|

From these, the classifier will build the following prompt:

```
  answer the following questions about the case:

  Q1: what is the filing date? DD/MM/YYYY
  Q2: does this judgment concern an interlocutory application? Answer "yes", "no" or "unclear" 
  Q3: if the judgment concerns an interlocutory application, what was the date of the application? DD/MM/YYYY

  Return your answer as a JSON object, following this example:
    {{
      "filing_date": "5/6/2010",
      "interlocutory": "yes",
      "interlocutory_date": "4/3/2010"
    }}
```

Note that the example JSON is constructed automatically from the example
answers in the "example" column.


## Batch requests
The classifier can take advantage of OpenAI's Batch API. Batch requests are "asynchronous groups of requests with 50% lower costs, a separate pool of significantly higher rate limits, and a clear 24-hour turnaround time" (see https://platform.openai.com/docs/guides/batch).

### send a batch request for a single case

```
classifier.batch_send("cases/123456789abcdef0.json")
```

One batch request is made for one single case. Each line in the JSONL for the request contains one group of questions provided in the prompts spreadsheet.


If you re-run ```classifier.batch_send```, the classifier will look for an earlier batch request first, and send a new batch request if and only if no such earlier batch request exists. To force the classifier to send a new batch request to the LLM even if an earlier batch request exists, use the `--no-cache` flag. For example:
```
classifier.batch_send("cases/123456789abcdef0.json", no_cache=True)
```


### check the status of a batch request for a single case

```
status_output_file_id = classifier.batch_check("cases/123456789abcdef0.json")
```

```classifier.batch_check``` returns a dictionary of status and output file id. 

### get the output for a completed batch request for a single case

```
output = classifier.batch_get("cases/123456789abcdef0.json")
```

### keep track of batch requests

The batch_records spreadsheet keeps track of batch requests. Each of ```classifier.batch_send```, ```classifier.batch_check``` and ```classifier.batch_get``` uses and updates this spreadsheet. The columns of this spreadsheet are as follows:

|submission_time|status|batch_id|input_file_id|output_file_id|case_id|
|---|---|---|---|---|---|
|datetime object as a string|API generated string|API generated string commencing "batch_"|API generated string commencing "file_"|API generated string commencing "file_"|stem of file path as a string|

If multiple batch requests have been submitted for the same case, ```classifier.batch_check``` returns the status and output file id of the most recent request. Similarly, ```classifier.batch_get``` retrieves the output of the most recent completed request. 

To check the status or get the output of an earlier request, one only needs to amend the batch_records spreadsheet.  

## Acknowledgements

This project is partially funded by a 2022 University of Sydney Research
Accelerator (SOAR) Prize awarded to Ben Chen.