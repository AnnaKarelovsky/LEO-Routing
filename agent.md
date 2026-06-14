# AGENTS.md

This repository is being used to reproduce a satellite-network research paper inside an existing open-source satellite routing simulation framework.

Core rule: do not rewrite the simulator from scratch. Reuse existing simulation, topology, packet forwarding, routing, delay, metrics, and experiment infrastructure whenever possible.

Research reproduction priorities:

1. Faithfulness to the paper.
2. Minimal invasive integration with the existing framework.
3. Deterministic and reproducible experiments.
4. Clear separation between paper-specific modules and generic framework modules.
5. Explicit assumptions for every missing or underspecified paper detail.

Before implementing any large change:

* Read the relevant paper section.
* Identify the corresponding existing framework module.
* Write a small implementation plan.
* Prefer adapter modules, routing strategy classes, config files, and experiment scripts over modifications to core simulation logic.
* Ask for confirmation before large architectural changes.

Required documentation for this reproduction:

* docs/reproduction_plan.md
* docs/paper_to_framework_mapping.md
* docs/equations_and_metrics.md
* docs/assumptions_and_unknowns.md
* docs/experiment_matrix.md
* docs/implementation_stages.md

Testing and validation:

* Every new metric must have a small sanity test.
* Every routing policy must have a minimal deterministic toy-topology test.
* Every experiment must support fixed random seeds.
* All generated figures must be reproducible from saved raw metrics.
* Do not claim a figure is reproduced unless the corresponding script runs and produces the plot.

Implementation style:

* Keep paper-specific code under a clearly named module, such as lisdm/ or semantic_delivery/.
* Keep baseline algorithms separate from proposed algorithms.
* Keep neural-network training code separate from simulation loop code.
* Keep plotting code separate from experiment execution.
* Avoid hidden global state.
* Avoid hard-coded paper parameters inside algorithm logic; place them in config files.

When paper details are missing:

* Do not guess silently.
* Record the gap in docs/assumptions_and_unknowns.md.
* Use a configurable default only after documenting it.
