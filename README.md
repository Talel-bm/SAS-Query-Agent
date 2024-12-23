# SAS-Query-Agent

This project implements an agent that generates and optimizes SAS code to answer data-related queries using a tree-based search algorithm. It leverages large language models (LLMs) for code generation, error correction, and requirement analysis, and incorporates techniques like caching and reflection to improve efficiency and accuracy.

## Technical Details

### Core Components

1. **Metadata Verification:**
    *   The `MetadataVerifier` class uses an LLM to critically assess the relevance of selected metadata (databases and columns) for a given query.
    *   It checks for missing essential columns, completeness of columns needed for calculations, filtering, grouping, and output, and adherence to business rules.
    *   The verifier provides feedback on whether the metadata is approved and suggests improvements if needed.

2. **Cached Metadata Generation:**
    *   The `CachedMetadata` class is a singleton that intelligently identifies the smallest set of databases and columns necessary to answer a query based on provided metadata.
    *   It uses an LLM to generate an initial metadata selection, then employs the `MetadataVerifier` to check its validity.
    *   If the initial selection is not approved, it regenerates the metadata based on the verifier's feedback.
    *   This class caches the generated metadata to avoid redundant computations for the same query and metadata.

3. **System Prompt Generation:**
    *   `get_system_prompt` creates a detailed system prompt for the LLM, instructing it to act as "StarData," an expert SAS coding agent.
    *   The prompt provides the necessary metadata, outlines steps for code generation, and includes numerous very important notes such as column naming conventions, type consistency, error handling.
    *   The prompt is tailored to the specific problem domain (automobile insurance).

4. **Code Correction and Requirement Analysis:**
    *   `give_requirement_corrector_system_prompt` creates a prompt for correcting code that does not meet query requirements.
    *   `error_corrector_system_prompt` is designed for fixing errors found in the SAS log. It includes detailed instructions for debugging common SAS errors and best practices for `UNION ALL` operations.

5. **Reflection Mechanism:**
    *   `ErrorReflection` and `RequirementReflection` are base models used to capture reflection information about the generated code.
    *   `ErrorReflection` stores the number of errors, specific fixes for each error, and whether all errors were resolved.
    *   `RequirementReflection` stores a list of unmet requirements, suggested fixes, and whether all requirements were ultimately met.
    *   `Reflector` uses an LLM to analyze the code and log, generating instances of `ErrorReflection` and `RequirementReflection`.
        *   It caches reflection results to avoid repeated analysis.
        *   It uses a `gradio_client` to interact with a requirement checking API, which determines if the code meets the query's requirements.

6. **Code Corrector:**
    *   The `CodeCorrector` class utilizes the `Reflector`'s output to refine the generated SAS code.
    *   It iteratively corrects errors and addresses unmet requirements based on the feedback from `ErrorReflection` and `RequirementReflection`.
    *   It caches corrected code to avoid redundant corrections.

7. **Tree-Based Search:**
    *   `Node` represents a node in the search tree, storing the SAS code, error and requirement reflections, parent and children nodes, and other relevant information.
    *   Nodes have a value based on their error and requirement scores, and they track their visit count.
    *   The `upper_confidence_bound` method implements the UCB1 algorithm for balancing exploration and exploitation during tree traversal.
    *   `backpropagate` updates the value and visit count of ancestor nodes when a child node is visited.
    *   `is_solved` indicates whether a node represents a solution that meets all requirements and has no errors.
    *   `get_best_solution` returns the best solution found in the tree.

8. **Graph Definition and Execution:**
    *   `TreeState` is a `TypedDict` defining the state of the search, including the root node, input query, and a termination flag.
    *   `generate_initial_response` generates the first SAS code using the LLM, along with metadata, and creates the root node.
    *   `expand` selects the best candidate node using `select`, generates a correction using `CodeCorrector`, and creates a new child node if requirements are met.
    *   `select` uses the UCB algorithm to choose the most promising node for expansion.
    *   `should_loop` determines whether to continue the search based on the `terminate` flag in the state.
    *   `execute_sas_code` submits the SAS code to a SAS session and returns the log.
    *   `extract_data` extracts the final dataframe name from the SAS code, retrieves the data using `sas.sd2df`, and saves it to an Excel file.
    *   `graph_builder` constructs the state graph using `langgraph`, defining nodes and edges for the search process.
    *   `run_sas_agent` orchestrates the entire process, running the graph for a specified number of steps or until a solution is found.

### Metadata Dictionary

The agent relies on a carefully constructed metadata dictionary that provides a comprehensive description of the SAS databases. This dictionary is created by extracting metadata directly from the SAS server, ensuring accuracy and completeness.

**Content of the Metadata Dictionary:**

*   **Database Names:**  All database names present on the SAS server are included as keys in the dictionary.
*   **General Description:** Each database entry contains a general description of its purpose and the type of data it stores.
*   **Columns:** For each database, the dictionary includes a list of all its columns.
*   **Column Descriptions:** Each column is accompanied by a detailed description explaining its meaning and usage.
*   **Column Types:** The data type of each column (e.g., numeric, character, date) is specified.
*   **Category Values (if applicable):** If a column represents a categorical variable, the dictionary lists its possible values.

**Example Structure of the Metadata Dictionary:**

```json
{
  "database_name_1": {
    "description": "This database contains information about...",
    "columns": {
      "column_name_1": {
        "description": "Description of column 1",
        "type": "numeric",
        "values": []
      },
      "column_name_2": {
        "description": "Description of column 2",
        "type": "character",
        "values": ["value1", "value2", "value3"]
      }
    }
  },
  "database_name_2": {
    "description": "This database contains information about...",
    "columns": {
      "column_name_3": {
        "description": "...",
        "type": "date",
        "values": []
      }
    }
  }
}
```
This detailed metadata dictionary is essential for the agent's ability to understand the structure and content of the SAS databases, enabling it to generate accurate and relevant SAS code.

### Workflow

1. **Initialization:**
    *   The user provides a data-related query and a path to a JSON file containing the database metadata.
    *   The `CachedMetadata` class generates the necessary metadata for the query.
    *   An LLM (e.g., `ChatGoogleGenerativeAI`) and a `gradio_client` are initialized.

2. **Graph Execution:**
    *   The `run_sas_agent` function starts the graph execution.
    *   `generate_initial_response` uses the LLM to generate the first SAS code based on the query and metadata, creating the root node of the search tree.
    *   The graph iteratively expands the tree by selecting the best candidate node (using `select` and UCB), correcting its code (using `CodeCorrector`), and creating a new child node if requirements are met (using `expand`).
    *   The `Reflector` analyzes the code and log to provide feedback on errors and unmet requirements.
    *   The search continues until a solution is found (a node with no errors and all requirements met) or the maximum number of steps is reached.

3. **Output:**
    *   If a solution is found, the `extract_data` function extracts the final dataframe and saves it to an Excel file.
    *   The final SAS code is also printed to the console.

### Dependencies

*   `langchain`: For interacting with LLMs and building the state graph.
*   `langgraph`: For defining and executing the state graph.
*   `langchain_google_genai`: For using Google's Generative AI models.
*   `saspy`: For interacting with a SAS session.
*   `pandas`: For data manipulation and saving results to Excel.
*   `gradio_client`: For interacting with the requirement checking API.
*   `pydantic`: For data validation and defining base models.
*   `typing_extensions`: For using `TypedDict`.

### Setup

1. **Install Dependencies:**
    ```bash
    pip install -U langchain langgraph langchain_google_genai saspy pandas gradio_client pydantic typing_extensions
    ```

2. **Set Google API Key:**
    ```python
    import os
    import getpass
    
    def _set_if_undefined(var: str) -> None:
        """Set environment variable if not already defined."""
        if os.environ.get(var):
            return
        os.environ[var] = getpass.getpass(var)
    
    _set_if_undefined("GOOGLE_API_KEY")
    ```
    This will prompt you to enter your Google API key.

3. **Prepare Metadata:**
    *   Create a JSON file (`meta_data.json` in this example) containing the database metadata. The format should be a dictionary where keys are database names and values are dictionaries containing column names, descriptions, types, and optional values.

4. **Configure SAS Session:**
    *   Create a SAS configuration file (`sascfg.py`) for `saspy`. Refer to the `saspy` documentation for details on configuration options.
    *   Ensure that you have access to a SAS server and that the necessary libraries are defined.

5. **Run the Agent:**
    *   Modify the `question` variable in the script to your desired query.
    *   Run the script. It will output the search progress, the final SAS code, and save the results to an Excel file if a solution is found.
