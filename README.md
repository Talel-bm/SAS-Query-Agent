# SAS-Query-Agent

This project implements a modified version of the CodeTree framework, as described in the paper "CodeTree: Agent-guided Tree Search for Code Generation with Large Language Models" by Li et al., tailored for generating and refining SAS code to answer data-related queries within an actuarial context, specifically focusing on automobile insurance.

## Overview

The CodeTree framework utilizes a tree search algorithm guided by multiple specialized Large Language Model (LLM) agents to explore, generate, evaluate, and refine code solutions. This implementation adapts the framework to work with SAS code, leveraging a metadata JSON dictionary that describes available databases and their schemas.

## Core Components

### Agents

The system employs four specialized LLM agents, each with a distinct role:

1. **Thinker:** Generates multiple strategies for solving a given data query using SAS code. It analyzes the query and metadata to devise different logical approaches.
2. **Solver:** Generates an initial SAS code solution based on a selected strategy and the provided metadata.
3. **Debugger:** Refines the SAS code solution by incorporating feedback from the Critic, addressing errors, and improving the code's adherence to the original query and strategy.
4. **Critic:** Evaluates the generated SAS code, provides feedback, assigns a numerical score, and determines whether the solution should be refined, aborted, or accepted.

### Metadata Management

-   **`MetadataSelector`:** This class orchestrates the entire metadata selection process. It leverages a language model (LLM) to intelligently choose relevant databases and columns from a larger metadata pool based on a user's natural language query. It features an iterative approach with verification and refinement at each step to ensure accuracy.
    -   **Database Selection:**  Uses the LLM to analyze the query and identify potentially relevant databases. It then employs `MetadataVerifier` to validate these choices and iteratively refines the selection based on feedback, up to a maximum number of iterations.
    -   **Column Selection:** For each selected database, it performs a two-stage column selection:
        1. **Analysis:** The LLM analyzes the query in the context of the chosen database to identify key entities, filter conditions, and required calculations.
        2. **Selection:** Based on the analysis, the LLM selects specific columns from the database, adhering to predefined rules (e.g., column equivalence, calculation shortcuts). `MetadataVerifier` then validates these choices, and the selection is refined iteratively.
    -   **Question Expansion:** Before database selection, the `MetadataSelector` can expand the original user question using an LLM. This expansion clarifies ambiguities, incorporates domain-specific knowledge, and makes the question more amenable to metadata selection.
    - The `MetadataSelector` class also includes several helper methods:
        - `_format_database_descriptions`: Formats database descriptions for use in prompts.
        - `_get_llm_response`: Gets a raw response from the LLM given a prompt.
        - `_parse_database_selection`: Parses the LLM's database selection response.
        - `_perform_column_analysis`: Performs the column analysis step using the LLM.
        - `_perform_column_selection`: Performs the column selection step using the LLM.
        - `_refine_databases`: Refines the database selection based on verification feedback.
        - `_refine_columns`: Refines the column selection based on verification feedback.
        - `_build_final_metadata`: Builds the final metadata dictionary from the selected databases and columns.

-   **`MetadataVerifier`:** This class is responsible for critically evaluating the selected metadata (both databases and columns) to ensure it aligns with the query's requirements.
    -   **Database Verification:** It uses the LLM to assess aspects like temporal coverage, processing requirements, and overall completeness of the chosen databases.
    -   **Column Verification:** It checks for the presence of mandatory columns, fulfillment of query requirements, and adherence to specific rules.
    -   **Feedback Mechanism:** Provides detailed feedback on the selection, including analysis, suggested improvements (additions/removals), and an explanation of the verification result. It also stores the raw LLM response for debugging.

-   **`ResponseSchemas`:** This class defines the structure and format of the expected responses from the LLM during various stages (database selection, column analysis, verification). It uses `ResponseSchema` objects from `langchain.output_parsers` to specify field names, types, and descriptions, facilitating structured output parsing.

-   **`PromptTemplates`:** This class stores the templates for the prompts used to interact with the LLM. Each template is designed for a specific task (e.g., database selection, column analysis, verification) and includes placeholders for dynamic content like the question, available databases/columns, and previous verification feedback. It also enforces specific rules and instructions for the LLM to follow.

-   **`DatabaseMetadata` and `ColumnMetadata`:** Simple classes to represent a database and a column, respectively, holding the name and description of each.

-   **`VerificationResult`:** A Pydantic `BaseModel` to encapsulate the result of a verification step. It includes fields like `is_valid` (boolean), `analysis` (dictionary), `improvements` (dictionary), `explanation` (string), and `raw_response` (string, optional).

-   **`format_metadata_to_string`:** A utility function to convert the metadata dictionary into a human-readable string, useful for debugging or displaying the selected metadata.

### Tree Search

The core of the system is a tree search algorithm that explores the solution space:

-   **`Node`:** Represents a node in the search tree, storing a strategy, a SAS code solution, its evaluation ("Expand," "Refine," "Abort," or "Accept"), a critic score, and other relevant information.
-   **`TreeState`:** Represents the state of the search tree, containing the root node and the input query.
-   **`generate_initial_strategies`:** Creates the root node and generates initial strategies, along with their corresponding solutions and evaluations.
-   **`expand_tree`:** Expands the tree by selecting a node and either generating a solution (if "Expand"), refining the solution (if "Refine"), or potentially generating new strategies based on the Critic's feedback.
-   **`select_node_to_expand`:** Selects the most promising node to expand using the Upper Confidence Bound (UCB) algorithm.
-   **`should_continue`:** Determines whether the search should continue or terminate based on finding a solution, reaching maximum depth, or other criteria.

### Utility Functions

-   **`execute_sas_code`:** Executes the generated SAS code using the `saspy` library and returns the log.
-   **`extract_data`:** Executes SAS code and saves the resulting dataframe to an Excel file.

## Workflow

1. **Initialization:**
    -   The user provides a data-related query and a metadata JSON dictionary.
    -   The `CachedMetadata` singleton generates and caches the relevant subset of metadata.
    -   Configuration parameters are set (e.g., weights for scoring, thresholds for aborting/expanding).

2. **Initial Strategy Generation:**
    -   The `generate_initial_strategies` function uses the Thinker agent to generate multiple strategies for answering the query.
    -   For each strategy, the Solver generates an initial SAS code solution.
    -   The Critic evaluates each solution, assigns a score, and provides feedback.
    -   Nodes are created for each strategy and its corresponding solution, evaluation, and score, forming the initial branches of the search tree.

3. **Tree Expansion and Refinement:**
    -   The `should_continue` function determines if the search should proceed.
    -   The `expand_tree` function iteratively expands the tree:
        -   `select_node_to_expand` chooses a node to expand based on UCB and evaluation status.
        -   If the node is marked "Expand," the Solver generates a solution, and the Critic evaluates it.
        -   If the node is marked "Refine," the Thinker generates reflections based on the Critic's feedback and the execution log. The Debugger then refines the solution based on these reflections.
        -   New nodes are created for new or refined solutions.
        -   The process continues until a solution is accepted, the maximum depth is reached, or other termination criteria are met.

4. **Solution Selection and Data Extraction:**
    -   After the search terminates, the `get_best_solution` method of the root node retrieves the best solution found.
    -   If a solution is found and accepted, the `extract_data` function can be used to execute the SAS code and save the results to an Excel file.

## Key Improvements and Adaptations

-   **SAS Code Generation:** The system is specifically designed to generate SAS code, leveraging the `saspy` library for execution.
-   **Actuarial Context:** The agents are tailored to understand actuarial concepts, particularly in automobile insurance, through specific instructions in their system prompts.
-   **Metadata Handling:** The `CachedMetadata` class and `MetadataVerifier` ensure efficient and accurate use of metadata, crucial for generating relevant SAS code.
-   **Error Categorization:** The Critic's `_get_error_reflection` function now attempts to categorize errors (syntax, runtime, logical) for more targeted debugging.
-   **Requirements Reflection:** The Critic's `_get_requirement_reflection` function uses a structured output (via `PydanticOutputParser`) to determine whether the generated code meets the query's requirements.
-   **Strategy Adherence Reflection:** The Critic's `_get_strategy_adherence_reflection` function also uses a structured output to provide a numerical score and reasoning for how well the code adheres to the given strategy.
-   **Dynamic Expansion:** The tree expansion can dynamically generate new strategies based on the Critic's score, allowing for more exploration when promising solutions are found.
-   **Verification:** The Critic includes a `verify_solution` function to assess the robustness of solutions that pass visible test cases, reducing the risk of overfitting.

## Setup and Usage

1. **Install Dependencies:**
    ```bash
    pip install -U --quiet langchain langgraph langchain_google_genai saspy pandas
    ```

2. **Set up Google API Key:**
    -   Obtain a Google API key that has access to the Gemini models.
    -   Replace `"YOUR_GOOGLE_API_KEY"` in the code with your actual API key.

3. **Prepare Metadata:**
    -   Create a JSON file (e.g., `metadata.json`) containing the metadata of your SAS databases. The format should be a dictionary where keys are database names, and values are dictionaries containing table names, descriptions, column names, descriptions, types, and potential values (see example in the code).
    -   Update the `metadata_file_path` variable to point to your metadata file.

4. **Configure SASPy:**
    -   Ensure that `saspy` is properly configured to connect to your SAS environment. This typically involves setting up a configuration file (e.g., `sascfg_personal.py`). Refer to the `saspy` documentation for detailed instructions.

5. **Run the Code:**
    -   Modify the `query` variable in the `main` function to your desired data query.
    -   Adjust the `config` dictionary to tune the search parameters as needed.
    -   Execute the script: `python your_script_name.py`

## Notes

-   The code includes extensive logging to help understand the execution flow and debug any issues.
-   The `client` variable in the `main` function is currently mocked for testing purposes. Replace it with an actual client if you need to use its functionality (e.g., for extracting the final dataframe name).
-   The performance of the system depends heavily on the quality of the LLMs used and the accuracy of the metadata.
-   The system prompts are designed for a specific actuarial context (automobile insurance). You may need to modify them for different domains or use cases.
