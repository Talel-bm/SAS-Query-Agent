# SAS Query Agent: Natural Language to SAS Data Extraction (Template)

## Overview

This project provides a *template* for creating a natural language interface to SAS data extraction. Instead of requiring SAS or SQL expertise, users can ask questions in plain language, and the system will leverage a Large Language Model (LLM) to translate these questions into executable SAS code.

This approach utilizes a metadata-driven strategy, requiring users to create structured JSON metadata describing their SAS database. The LLM then uses this metadata to intelligently generate SAS queries, which are executed using the `saspy` library. The results are extracted and can be saved as an Excel file. The system also incorporates a memory component, allowing it to learn from past queries and improve its accuracy over time.

**Important:** This is a template. *You must provide your own SAS environment, API keys, metadata, and memory files.*

## Key Features

*   **Natural Language Querying:** Translate natural language questions into SAS queries.
*   **Metadata-Driven:** Relies on structured JSON metadata to guide query generation (user-provided).
*   **LLM-Powered Code Generation:** Uses a Large Language Model (Google Gemini) to generate SAS code.
*   **SAS Integration:** Executes generated code via `saspy`.
*   **Adaptive Learning:** Utilizes a memory system to improve accuracy based on past queries (user-maintained).
*   **Development Mode:** Offers interactive feedback for fine-tuning the system.

## Architecture

1.  **Database & Column Selection:** The LLM identifies relevant databases and columns using *your* metadata.
2.  **SAS Code Generation:** The LLM generates SAS code.
3.  **SAS Execution:** The generated SAS code is executed.
4.  **Data Extraction:** Results are converted to a Pandas DataFrame and can be saved as an Excel file.
5.  **Memory Update:** The system can store the query, code, and execution status in a user maintained memory file.

## Installation (Summary)

1.  Set up a Python environment (Anaconda recommended).
2.  Install required Python packages (`numpy`, `pandas`, `openpyxl`, `saspy`, `gradio`, `google-genai`).
3.  Configure `saspy` with your SAS server details ( `sascfg.py` and `_authinfo.`).
4.  Obtain and configure a Google Gemini API key.
5.  **Create and configure your metadata and memory files.**

## Usage

1.  Open the provided Jupyter Notebook (`sas query agent.ipynb`).
2.  Configure the notebook with your SAS connection details, API key, and *paths to your metadata and memory files.*
3.  Use the `DATA_RETRIEVER` function with your natural language query. Use `dev_mode=True` initially for feedback.

## Data Structures

*   **Metadata File (JSON):** Describes your SAS database tables and columns. See example format in full README.
*   **Memory File (JSON):** Stores past queries, code, and feedback. See example format in full README.

## Important Files

*   `sas query agent.ipynb`: The main notebook.
*   `sascfg.py`:  `saspy` configuration.
*   `_authinfo.`:  SAS authentication credentials.
*   `metadata.json`: **You must create this file.**
*   `agent_memory.json`: **You must create this file.**

## Contributing

Contributions are welcome (bug reports, feature requests). Do not include sensitive data files.

## License

