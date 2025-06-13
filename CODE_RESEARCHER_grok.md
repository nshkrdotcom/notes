https://www.microsoft.com/en-us/research/wp-content/uploads/2025/06/Code_Researcher-1.pdf

# Enhanced System for Dynamically Listing Subsets of Files for Features in an Elixir Codebase

Managing a large Elixir codebase, such as the one described, presents a challenge when interacting with tools that have limited context windows (e.g., LLMs for code analysis or generation). To address this, we need a system that dynamically identifies and lists a subset of files relevant to specific features or bugs. Drawing inspiration from the research paper "Code Researcher: Deep Research Agent for Large Systems Code and Commit History" by Ramneet Singh et al., which tackles context gathering in complex systems codebases like the Linux kernel, we can adapt its principles—deep research, structured context gathering, and historical analysis—to create an effective solution for Elixir. Below is an enhanced system tailored to the Elixir codebase, leveraging both simple tools and advanced possibilities like Elixir’s Abstract Syntax Tree (AST) analysis.

## Proposed System: Module Dependency Analysis with Historical Context

The system models the Elixir codebase as a graph of module dependencies and augments it with historical commit data to ensure a comprehensive yet focused subset of files. This approach mirrors the Code Researcher agent's ability to explore codebases deeply and synthesize relevant context, adapting its three-phase process (Analysis, Synthesis, Validation) to our needs.

### Steps of the System

1.  **Identify the Feature’s Entry Point**
    Define the feature by its primary module (e.g., `DSPEx.Teleprompter.BEACON` for the "BEACON optimization" feature) or a key function (e.g., `DSPEx.Teleprompter.BEACON.compile/5`).
    This serves as the "seed" for context gathering, similar to how Code Researcher starts with a crash report.

2.  **Generate the Dependency Graph**
    Use Elixir’s `mix xref graph` to create a dependency graph of all modules, showing which modules depend on or are depended upon by others. This is analogous to Code Researcher’s `search_definition(sym)` and `search_code(regex)` actions for tracing code relationships.

3.  **Incorporate Commit History**
    Analyze commit history using `git log` or `git grep` to identify files related to the entry point or feature name (e.g., commits mentioning "BEACON"). This reflects Code Researcher’s `search_commits(regex)` action, leveraging historical context to uncover related files not directly in the dependency graph.

4.  **Traverse the Graph with Historical Insights**
    Start at the entry point module and traverse the graph to collect dependent modules (downstream traversal) up to a specified depth (e.g., 1 or 2 levels).
    Include files from commit history that are frequently modified alongside the entry point, even if not direct dependencies. This ensures historical relevance, akin to Code Researcher’s causal analysis over commits.
    Use heuristics (e.g., commit frequency or relevance score) to limit the subset size.

5.  **Map Modules to Files**
    Convert module names to file paths (e.g., `DSPEx.Teleprompter.BEACON` → `dspex/teleprompter/beacon.ex`) and include additional files from commit history.

6.  **Return the Subset**
    Output a concise list of file paths, ensuring it fits within context window constraints while covering the feature comprehensively.

### Example Application

For the "BEACON optimization" feature:

* **Entry Point:** `DSPEx.Teleprompter.BEACON` (`dspex/teleprompter/beacon.ex`).
* **Dependency Graph:** `mix xref graph` shows dependencies like `DSPEx.Teleprompter.BEACON.BayesianOptimizer` (`dspex/teleprompter/beacon/bayesian_optimizer.ex`) and `DSPEx.Services.ConfigManager` (`dspex/services/config_manager.ex`).
* **Commit History:** Searching for "BEACON" in commit messages reveals related files like `dspex/adapters/instructor_lite_gemini.ex` (an adapter modified with BEACON changes) and `dspex/config.ex` (configuration updates).
* **Traversal Result:** Combines graph dependencies and historical files: `["dspex/teleprompter/beacon.ex", "dspex/teleprompter/beacon/bayesian_optimizer.ex", "dspex/services/config_manager.ex", "dspex/adapters/instructor_lite_gemini.ex", "dspex/config.ex"]`.

This subset (5 files) is much smaller than the full codebase (20+ files), yet it captures both structural and historical context.

### Implementation Sketch

Here’s a practical Elixir implementation leveraging `mix xref` and Git commands:

```elixir
defmodule FeatureFileSelector do
  @doc """
  Lists files relevant to a feature based on module dependencies and commit history.
  Args:
    - entry_module: Starting module (e.g., "DSPEx.Teleprompter.BEACON")
    - depth: Max traversal depth (default: 1)
    - commit_search_term: Term to search in commit history (optional)
  """
  def list_files_for_feature(entry_module, depth \\ 1, commit_search_term \\ nil) do
    # Build dependency graph
    graph = build_dependency_graph()

    # Traverse dependencies
    relevant_modules = traverse_dependencies(graph, entry_module, depth)

    # Add historical context if provided
    historical_files = if commit_search_term, do: search_commit_history(commit_search_term), else: []

    # Combine and deduplicate
    (modules_to_files(relevant_modules) ++ historical_files)
    |> Enum.uniq()
  end

  defp build_dependency_graph do
    # Simulate parsing `mix xref graph --format=pretty`
    %{
      "DSPEx.Teleprompter.BEACON" => [
        "DSPEx.Teleprompter.BEACON.BayesianOptimizer",
        "DSPEx.Services.ConfigManager"
      ],
      "DSPEx.Teleprompter.BEACON.BayesianOptimizer" => ["DSPEx.Example"],
      # Add more mappings as needed
    }
  end

  defp traverse_dependencies(graph, module, depth, visited \\ MapSet.new()) do
    if depth < 0 or MapSet.member?(visited, module) do
      visited
    else
      deps = Map.get(graph, module, [])
      Enum.reduce(deps, MapSet.put(visited, module), fn dep, acc ->
        traverse_dependencies(graph, dep, depth - 1, acc)
      end)
    end
  end

  defp search_commit_history(search_term) do
    # Simulate `git log -S #{search_term}` or `git grep #{search_term} $(git rev-list --all)`
    # Returns file paths from commits mentioning the term
    ["dspex/adapters/instructor_lite_gemini.ex", "dspex/config.ex"]
  end

  defp modules_to_files(modules) do
    modules
    |> Enum.map(fn module ->
      module
      |> String.replace(".", "/")
      |> String.replace_prefix("DSPEx/", "dspex/")
      |> Kernel.<>(".ex")
      |> String.downcase()
    end)
  end
end
```

**Usage:**

```elixir
FeatureFileSelector.list_files_for_feature("DSPEx.Teleprompter.BEACON", 1, "BEACON")
# => ["dspex/teleprompter/beacon.ex", "dspex/teleprompter/beacon/bayesian_optimizer.ex",
#     "dspex/services/config_manager.ex", "dspex/adapters/instructor_lite_gemini.ex",
#     "dspex/config.ex"]
```

### Why This Approach?

* **Simplicity:** Relies on `mix xref` and Git, tools already integrated into Elixir workflows, avoiding complex initial setups.
* **Deep Context:** Combines structural dependencies with historical insights, inspired by Code Researcher’s success (e.g., exploring 10 files per trajectory vs. SWE-agent’s 1.33).
* **Relevance:** Historical analysis ensures files contextually tied to the feature (e.g., adapters or configs) are included, even if not direct dependencies.
* **Adaptability:** Generalizes to other Elixir codebases, much like Code Researcher’s performance on FFmpeg after Linux kernel experiments.

The Code Researcher paper demonstrates that deep exploration and commit history analysis significantly improve outcomes in large codebases (58% crash resolution rate vs. 37.5% for SWE-agent). This validates our inclusion of commit history to capture the evolution of features like BEACON.

## Future Enhancements with AST Analysis

While the current system is effective, Elixir’s AST capabilities offer a path to greater precision, inspired by Code Researcher’s reasoning strategies (e.g., chasing control/data flow, pattern searching). Here’s how:

### AST-Based Dependency Tracking

Parse each `.ex` file with `Code.string_to_quoted/1` and traverse the AST using `Macro.traverse/4`.
Identify:

* **Aliases/Imports:** `alias DSPEx.Services.ConfigManager` → dependency edge.
* **Function Calls:** `ConfigManager.get/1` → link to `config_manager.ex`.
* **Structs:** `%DSPEx.Teleprompter.BEACON{}` → tie to `beacon.ex`.
* **Behaviours:** `@behaviour DSPEx.Program` → connect to `program.ex`.

Build a fine-grained graph, potentially more accurate than `mix xref` for dynamic calls.

### Commit History Analysis via AST

Parse commit diffs at the AST level to detect changes to specific functions or modules (e.g., additions to `BEACON.compile/5`).
Weight files by relevance (e.g., frequency of BEACON-related changes), refining the subset.

### Contractual Interfaces for Pruning

Detect behaviours (`@behaviour DSPEx.Teleprompter`) or use macros in the AST.
Example:

Feature uses `DSPEx.Client.request(messages, provider: :gemini)`.
AST analysis identifies `:gemini` as context, including `instructor_lite_gemini.ex` while excluding `openai.ex`.
Reduces subset size by focusing on relevant implementations, mirroring Code Researcher’s filtering in the Synthesis phase.

### Structured Context Memory

Maintain a memory structure (e.g., ETS table) of modules, files, and commit metadata, similar to Code Researcher’s structured memory. This enables iterative refinement and validation of the subset.

## Validation and Practicality

The Code Researcher paper’s Validation phase checks patch correctness. For our system, we can:

* **Verify Coverage:** Ensure all key dependencies (e.g., `BayesianOptimizer` for BEACON) are included via unit tests or manual review.
* **Limit Size:** Cap the subset (e.g., 5-10 files) to fit context windows, adjustable via depth or heuristics.

This system starts simple with `mix xref` and Git, then scales with AST enhancements, balancing practicality with depth.

## Conclusion

This enhanced system dynamically lists file subsets for Elixir features by integrating module dependencies and commit history, inspired by Code Researcher’s deep research approach. It provides a focused, relevant context for tools with limited windows, starting with a practical implementation and offering a clear path to AST-driven precision. By adapting proven strategies from systems code research to Elixir’s ecosystem, it tackles the challenges of large codebases effectively, with potential for broader applications like code reviews or refactoring.

