# Actor Processes

## runtime-server

The program that serves the container execution protocol and runs step implementations; local tooling can embed it in the same process.

## syntara-plugin

The command-line program used to develop, package, publish, and locally run plugins.

## ep-worker

The Execution Plane process that launches workloads, collects results, and reports completion.

# Libraries

## syntara-plugin-sdk

The authoring library providing plugin development commands and testing support.

## syntara-plugin-contracts

The shared library providing schemas, protocol bindings, and runtime support, including `ContainerClient`.

# Plugin Terms

| Term | Meaning |
| --- | --- |
| Plugin | A versioned release declaring step types, integration types, credential recipes, and documentation; its inventory is `plugin.yaml`. |
| Step type | A catalog capability, such as **Create issue**, declared by a step manifest. |
| Category | A step type's role: `action`, `control`, or `trigger`. |
| Runner | The execution ownership selected by `platform-runner` or `custom-runner`; these are manifest values, not process names. |
| Entry point | The `module:Class` reference identifying a custom-runner implementation. A container entrypoint separately identifies its startup command. |
| Workload image | A container image containing executable step code and its dependencies. |
| Step identity | The stable name `namespace.plugin.step`. |

See the [execution model](plugin-language.md#terms-and-execution-model) for how these terms fit together.
