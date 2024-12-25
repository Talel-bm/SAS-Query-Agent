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

-   **`MetadataVerifier`:** Critically analyzes the selected metadata to ensure it contains the necessary databases and columns to answer the query.
-   **`CachedMetadata`:** A singleton class that generates and caches the smallest set of relevant metadata extracted from a larger metadata dictionary based on the input query. It uses an LLM to identify essential databases and columns and includes a verification and regeneration mechanism.

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
