##### Clafer Skills Version 0.5.8, unreleased

* For Clafer 0.5.8.
* `clafer-modeling`: the initializers `=` and `:=` (the initial value), and their inheritance: a subclafer can only strengthen an inherited initializer. A super can be a path to a nested abstract clafer, e.g., `h : Person.Head`.
* `clafer-modeling`: the explicit form first (`clafer -m clafer`), with a table of the syntactic sugar it stands for: the default group cardinalities and multiplicities, paths, `dref`, initializers and transitions. Reading the explicit form explains most defaults, e.g., which `speed` a name resolves to.
* `clafer-modeling`: encapsulation of abstractions, as in object-oriented design: state and invariants inside, a small interface of attributes and references, dependencies as references instead of global names, identity (`final`) apart from mutable state, and specialization by strengthening; the counter as an example.
* `clafer-modeling`: practical consequences of the temporal semantics: where a constraint lives decides how long it lasts; an instance has a value only while it exists, so a value rule keeps its clafer alive; every new instance is initialized again; the value of `:=` keeps the parent's temporal context; rules for values need a quantifier with parentheses, and `let` cannot capture a value; behaviours share the loop of a trace; `final` is shallow. Also the clock of an inherited initializer, set reference keys per clafer, and the temporal keywords that cannot be names.

##### Clafer Skills Version 0.5.7, Nov 8, 2026

* The first release: the skill `clafer-modeling`, for Clafer 0.5.7, and a Claude Code plugin marketplace with the plugin `clafer`.
