# Generation-time Verification

Checks completed in the generation environment:

- all Maven `pom.xml` files parse as XML;
- the BPMN file parses as XML;
- all three DMN files parse as XML;
- Java source was passed through `javac` parsing with Java 21 available in the environment;
- no Java syntax-pattern errors were reported (expected unresolved external dependency errors remain because Maven dependencies are not installed in the generation container);
- module directories and parent-module declarations were cross-checked;
- concrete `.java` Partitioner/Supervisor classes referenced by BPMN are present;
- source/config/JSP scan confirms the implementation does not introduce React, Thymeleaf or Kafka.

## Important limitation

A dependency-resolved Maven build could not be executed inside the generation environment because Maven is not installed there and outbound Maven dependency resolution is unavailable.

Therefore the package is **source/static validated**, not claimed to have completed a full local Spring/Camunda runtime boot in the generation environment.

On the developer machine, the first acceptance gate is:

```bat
scripts\00-build.bat
```

Then follow `RUNBOOK.md`.

If Maven reports a dependency/API compatibility issue, retain the full Maven error output so the exact dependency can be adjusted without changing the architecture.
