---
name: clafer-modeling
description: Write, structure, fix and check Clafer models (.cfr), static or temporal, and run them with clafer, claferIG and clafersg (formerly chocosolver). Use when asked to model something in Clafer, to split a model into files, to debug a model that has no instances or traces or that does not compile, to write scenarios or assertions, or to explain what a Clafer construct means.
---

# Modeling in Clafer

Clafer models structure as a hierarchy of clafers, which are features, classes and states at once, with constraints over it.
A static model has instances; a temporal model has traces of states, each of them an instance.
The language reference is the CheatSheet (`doc/CheatSheet.md` in the clafer repository).
Where the tools differ from the papers on Clafer's semantics, and why, is in `doc/static-semantics-errata.md` (Bąk et al. 2016) and `doc/temporal-semantics-errata.md` (Juodisius et al. 2019) there.
This guide is for Clafer 0.6.0.

## Workflow

1. Write the smallest model that shows the idea, then grow it. Most models of a few clafers solve in about a second; a model that takes minutes is usually contradictory or has scopes that are too large.
2. Compile it: `clafer model.cfr` (`-m alloy` or `-m choco` to see the generated models). Errors come as `model.cfr:line:column`. When a construct does not mean what you expect, read its explicit form: `clafer -m clafer model.cfr` writes `model.des.cfr` (see below).
3. Look at instances: `claferIG model.cfr` (Alloy) or `clafersg --file model.cfr --repl --prettify` (Choco for a static model, Alloy for a temporal one). Look at several instances (`n`), not just the first: they show what the model leaves open.
4. State what must hold as assertions and run `check` in claferIG. "No counterexample" holds only within the scopes (and the trace length).
5. For a configuration or behaviour that should be possible, write a scenario in a file that imports the model, and check that it has an instance; also check that a variant that should be impossible has none.

## The explicit form: what a model means

Every Clafer model has an explicit form, in which nothing is left to defaults: every group cardinality, every multiplicity, every path and every `dref` is written out. `clafer -m clafer model.cfr` prints it to `model.des.cfr`, and it is itself a model that compiles to itself (`//# OPTIONS --explicit`). Learn the explicit form first: the rest of the syntax is sugar for it, and most surprises about defaults disappear when you read what a model desugars to.

A clafer in the explicit form is

```
<group cardinality> <name> [: <super>] [-> <target> | ->> <target>] <multiplicity> {
  <children>
  [ <constraints> ]
}
```

with the group cardinality on the left (how many child instances an instance has, counted over all its children) and the multiplicity on the right (how many instances under each parent instance). For example,

```
abstract Component                abstract 0..* Component 0..* {
  version -> integer                0..* version -> int 1..1 {}
  xor mode                          1..1 mode 1..1 {
    active                            0..* active 0..1 {}
    standby                           0..* standby 0..1 {}
                                    }
  extras ?                          0..* extras 0..1 {}
  ports -> Port 1..3                0..* ports -> Port 1..3 {}
Port 2                            }
  speed -> integer                0..* Port 2..2 {
camera : Component                  0..* speed -> int 1..1 {}
  [ version >= 2 ]                }
  [ extras ]                      0..* camera : Component 1..1 {
  [ speed = 3 ]                     [this.version.dref >= 2]
                                    [some this.extras]
                                    [Port.speed.dref = 3]
                                  }
```

The last constraint shows why the explicit form is worth reading: `speed` in `camera` is the `speed` of all the ports at the top level (`Port.speed`), not of the camera's own `ports` (`this.ports.speed`). Write the path you mean.

What the sugar stands for:

| Sugar | Explicit form |
|---|---|
| no group cardinality | `0..*`: no constraint on the number of children |
| `xor`, `or`, `mux`, `opt`, `n..m` | `1..1`, `1..*`, `0..1`, `0..*`, `n..m` |
| no multiplicity | `1..1` (mandatory), but `0..1` (optional) under `xor`, `or`, `mux`, `opt` or a range such as `2..3` |
| `?`, `*`, `+`, `n` | `0..1`, `0..*`, `1..*`, `n..n` |
| `-> integer` | `-> int`, one target per instance; `->` is a set reference, `->>` a bag |
| a name `x` in a constraint | a path from the context: `this.x`, `this.parent.x`, a path through the reference target, or a top-level `X.x` |
| `[ x ]`, `[ no x ]`, `[ one p ]` | `[some this.x]`, `[no this.x]`, `[one this.o.p]` |
| a reference where a value is expected | its target: `[ version >= 2 ]` is `[this.version.dref >= 2]` |
| `fan -> integer = 2` | `[(some fan) => (fan.dref = 2)]` in the parent's context |
| `level -> integer := 3` (temporal) | `[initially (this.dref = 3)]` in the context of `level` |
| `xs -> integer * := 1` | `[(some this.xs) => (this.xs.dref = 1)]` in the parent's context, under `initially` in a temporal model |
| `xs -> integer * := each 1` | `[all xs_1 : this.xs \| xs_1.dref = 1]` in the parent's context (static), `[initially (this.dref = 1)]` in `xs` (temporal) |
| `idle -[ coin ]-> brewing` | `[((some this) && (some coin)) => (X (some this.parent.brewing))]` in `idle` |
| `brewing -->> ready` | `[(some this) => ((some this) U (some this.parent.ready))]` in `brewing` |

Consequences that follow directly from the explicit form:

- A clafer is a set of instances under each parent, never a single object: `#x`, `some x` and `x.dref` are over that set, and a constraint that holds for "the" `x` holds for each instance.
- An initializer `=` is an ordinary constraint of the parent, so it holds only if the clafer exists, and a subclafer's own initializer is just another constraint added to it.
- A transition is a constraint of its source state, so it is part of that state's lifetime (see Temporal models).
- When an instance count or a name looks wrong, the explicit form shows the multiplicity, group or path that the compiler chose.

## Structure

```
abstract Component            // a type: no instances of its own
  version -> integer          // a reference to a primitive value
  xor mode                    // group: exactly one child of mode
    active
    standby
  extras ?                    // multiplicity 0..1
  ports -> Port 1..3          // a set reference: no repeated targets per parent
Port
  speed -> integer
camera : Component            // concrete, inherits children, group and reference
  [ version >= 2 ]
```

- **Multiplicity** (right of the name) bounds the instances per parent instance: `?` 0..1, `*` 0..*, `+` 1..*, `n`, `n..m`.
- **Default multiplicity of children:** mandatory (`1`) under no group or `0..*`; optional (`0..1`) under `xor`, `or`, `mux`, `opt` and any other range such as `2..3`. Write `?` only where it adds something.
- **Group cardinality** (left of the name) counts child *instances*: `xor` 1..1, `or` 1..*, `mux` 0..1 (at most one), `opt` 0..* with optional children. A child with `1..5` can satisfy `2..3` alone. A concrete clafer's group counts its own and its inherited children.
- A group whose lower bound the children cannot reach (`xor` over no children) is a compile error.
- **References:** `-> T` is a set (no repeated targets among the instances of a clafer under one parent, also when the clafer inherits the reference; instances of different clafers may share a target, Clafer 0.5.8), `->> T` a bag. Where a value is expected, a reference stands for its target (`[ speed = 3 ]` means `speed.dref = 3`), and a comparison of two references compares their targets (`this.a = parent.b.a`, Clafer 0.5.7). To compare the reference instances themselves, compare `this` or a variable, or write `dref` on one side.
- **Initializers:** `fan -> integer = 2` fixes the value: `[ some fan => fan.dref = 2 ]` in the parent's context, in every state of a temporal model. `level -> integer := 3` is the initial value (Clafer 0.5.8): the same as `=` in a static model, and in a temporal model each lifetime of `level` starts with 3, which may then change. The value is resolved in the parent's context, where it is written. A set initializer (`xs -> integer * := 1, 2`) initializes the instances under the parent when the parent appears, not those created later. On an abstract clafer, the multiplicity decides this clock, and the subclafers keep it: `abstract Val -> integer := 1` initializes each instance when it is born, `abstract Val -> integer * := 1` the instances under a parent when the parent is born (or, at the top level, in the first state). The clock can be written (Clafer 0.6.0): `:= each e` checks each instance at its birth, also one born later, and `:= together e` all the instances under the parent at its birth (all those of a top-level abstract clafer at time zero), whatever the multiplicity; without a clock, the multiplicity decides as above. A top-level abstract clafer with a multiplicity other than `*` needs a clock, e.g., `abstract Val -> integer 0..3 := together 1`. `xs -> integer * := each 1, 2` has no instance: each instance is a single reference. `finalref x -> integer := y` keeps the value `y` has when `x` appears. `:=` is not a soft default: with `[ level = 5 ]`, `:= 3` has no instance. Before 0.5.8, `:=` compiled to an assertion (gsdlab/clafer#116), so the model had instances and a counterexample.
- **Enums:** `enum Color = red | green | blue`.
- **Types:** `integer`, `string`, `decimal` (Choco only; the scale is the most fractional digits of the literals), `double`/`real` (no backend).
- **Objectives** (Choco): `<< minimize total.cost >>`, `<< maximize perf >>`.

## Best practices

### Model the hierarchy

The hierarchy is the model: decompose a system into its parts, features into subfeatures, and states into substates, rather than listing clafers flat and relating them with constraints.

- A part that belongs to one parent is a child, not a reference: nesting gives the parent for free (`parent`), keeps names local, and lets constraints be written in its context.
- Use references for what is shared or chosen from elsewhere: a connector between ports, the engine a car uses from a catalogue.
- Use groups for choices: `xor` for alternatives, `or` for at least one, `mux` for at most one, `n..m` for a count. A choice is then visible in the structure, where a constraint would hide it.
- Use multiplicities for counts (`cams : camera 6`) instead of numbered siblings.

### Abstract clafers for reuse

Put what several clafers share in an abstract clafer, and extend it:

```
abstract Car
    abstract engine -> Engine     // every car has an engine of some kind
    xor drive
        fwd
        awd
abstract Engine
    power -> integer
gas : Engine
    [ power = 100 ]
electric : Engine
    [ power = 150 ]

tesla : Car
    motor : engine -> electric    // refines the engine: a narrower target
    [ drive.awd ]
golf : Car
    motor : engine -> gas
```

- The subclafers inherit the children, the group and the reference of the super clafer, and its constraints hold for them.
- Nest abstract clafers to declare what each instance has, and refine them in the subclafers: `motor : engine -> electric` is the `engine` of `tesla`, with its target narrowed from `Engine` to `electric`. Its constraints and those of `engine` both hold.
- Give a refining clafer its own name. A redefinition with the same name as the inherited clafer (`engine : engine -> electric`) compiles, but then `tesla.engine` is ambiguous: it could be either clafer.
- A super clafer must be abstract; inheritance from concrete clafers is not supported.
- A super can be a path to a nested abstract clafer: `h : Person.Head`. Each name after the first is a child of the previous clafer, its own or else an inherited one, so under `Ella : Person`, `Ella.Head` is `Person.Head` unless Ella redefines `Head` (`abstract Head : Head`, which extends `Person.Head` and can add children). A path tells apart two abstract clafers with the same name (`Dog.Head`, `Person.Head`). Before Clafer 0.5.8, a path super crashed the compiler (`Maybe.fromJust: Nothing`) once a constraint mentioned the clafer, and was silently dropped otherwise; write the plain name there.
- Abstract clafers can be nested under concrete clafers too (`System` with `abstract Port` and `p1 : Port`).

### Encapsulate abstractions

An abstract clafer is a class: it owns its parts, its invariants and its behaviour. Design it as you would an object-oriented class.

- **Keep the state inside.** Put the parts, the constraints over them and the state machine of a concept in its abstract clafer, so every instance has them and they cannot be forgotten. A constraint in the abstract clafer is a class invariant: it holds for every instance of every subclafer.
- **A small interface.** Expose what users of the abstraction set or read: a few attributes and references, e.g., `start` and `end` of a counter. Keep the rest as children that only the abstraction's own constraints mention. A subclafer then configures the abstraction through the interface (`[ this.start = 2 ]`), not by constraining its internals.
- **No global names inside.** A constraint in an abstraction that names a top-level clafer of one model ties the abstraction to that model. Give the abstraction a reference to what it depends on (`abstract engine -> Engine`), and let the subclafer narrow it (`motor : engine -> electric`), as a constructor parameter would. Use `parent` sparingly for the same reason: it assumes where the abstraction is placed.
- **No reaching in from outside.** A top-level constraint that navigates into the internals of an abstraction's instances (`[ all c : Counter | c.val.internal … ]`) breaks when the abstraction changes. State the requirement in the abstraction, or as a constraint of the interface.
- **Identity and state apart.** In a temporal model, make the object `final` and its mutable state a non-final child: `abstract final Counter` with `final start -> integer` and `val -> integer := this.start`. The counter stays the same object during a trace, and only `val` changes.
- **Specialize by strengthening.** A subclafer adds constraints and narrows multiplicities, groups and targets; it cannot weaken or replace what it inherits. If a subclafer needs to differ, factor the shared part into an intermediate abstract clafer.

```
abstract final Counter                  // the object: fixed during a trace
    final start -> integer              // the interface, set by the subclafers
    final end -> integer
    val -> integer := this.start        // the state: starts at start, changes
        [ all v : integer | (this = v --> this = (if v < parent.end then v + 1 else parent.start)) ]

c2to6 : Counter                         // configured through its interface only
    [ this.start = 2 ]
    [ this.end = 6 ]
```

### Use the expressive power of Clafer

Clafer unifies feature models and class models: one clafer can be a feature, a class, an association, an attribute or a state. Use that instead of encoding it as in other languages.

- **Containment is built in.** Unlike Alloy, where a part is a signature plus a field plus multiplicity and ownership facts, a nested clafer is the part: it has one parent, its multiplicity is per parent, and `parent` navigates back. Do not model containment with references.
- **Features with attributes and many instances.** Unlike a feature model, a feature can have attributes (`size -> integer`), references, children of any depth, and several instances (`cams : camera 6`); a group (`xor`, `or`, `n..m`) applies to instances of classes as much as to features.
- **References can have children.** A reference clafer is a link with its own data, like an association class: no separate link class and two references are needed.

  ```
  abstract Component
      abstract link -> Port *
          latency -> integer
          [ latency >= 1 && latency <= this.speed ]   // this.speed: the linked port's speed
  ```

- **Refine by redefinition.** A clafer that extends a nested abstract clafer is its redefinition in the subclafer, and it can specialize what it inherits:
  - the multiplicity: `abstract port : Port *` becomes `p : port 2`;
  - the group: `abstract or feature` becomes `xor f : feature`;
  - the reference target: `abstract link -> Port *` becomes `l : link -> Hub.p 1`, the ports of one hub, and a set reference can refine a bag (`->>`);
  - the value: an initializer `fan -> integer = 2`, or a constraint `[ speed = 3 ]`. An inherited initializer cannot be replaced, only strengthened: the own one is added to it. Under `abstract val -> integer := 2`, `v : val = 2` fixes the value in every state, while `:= 3` or `= 3` has no instance; under `= 2`, a `:= 2` only repeats the value and a `:= 3` leaves no instance; the compiler warns that a `:=` does not redefine an inherited `=` (Clafer 0.5.8). For different initial values, put the shared one in an intermediate abstract clafer (`abstract defaultVal : val := 2`) and let the others extend `val` directly (`override : val := 3`);
  - modifiers: `final` on the abstract clafer is inherited by its refinements. A refinement can add `final`, `finalref` or `finaltarget`: `final motor : engine` fixes the inherited reference (Clafer 0.5.7). Before 0.5.7, the inherited reference could still change during a trace unless restated, `final motor : engine -> Engine` (gsdlab/clafer#109).
  An abstract nested clafer that a subclafer does not refine has no instances in it: the subclafer simply lacks that part.
- **Constraints read like requirements.** A name alone means "some" (`[ encryption ]`), a reference stands for its value (`[ speed = 3 ]`), and names resolve from the context, so most constraints need neither quantifiers nor paths.
- **One model, many views.** The same model is a feature model to configure (claferIG, clafersg, the configurator), a class model to instantiate, a product line with objectives to optimize (clafersg), and, with temporal modifiers, a behaviour to simulate and check (claferIG, clafersg).
- **Partial instances are models.** An exemplar or a test case is a concrete clafer that extends the abstract model and fixes its choices with constraints; it is checked by the same tools.

### Place constraints in their most specific context

A constraint under a clafer holds for each of its instances, with `this` as that instance, and names resolve from there. So placing a constraint in the context it is about states it for every instance without a quantifier:

```
// avoid: a top-level constraint with quantifiers and full paths
[ all c : Car | c.engine.power >= 100 ]

// better: in the context of Car, for each car
abstract Car
    abstract engine -> Engine
    [ engine.power >= 100 ]

// best: in the context of the engine itself, for each engine of each car
abstract Car
    abstract engine -> Engine
        [ power >= 100 ]          // power of the target, through the reference
```

- The deeper the context, the shorter the constraint, and the fewer instances it can be wrong about. It also moves with the clafer when the model is restructured.
- A constraint on an abstract clafer holds for all its subclafers: write it once, at the most general clafer it applies to, and as deep as possible within it.
- Use quantifiers only for relations between instances of different contexts that no single context owns, e.g., `[ all disj a; b : Task | … ]` at the level that contains both.
- Write a choice of a subclafer as constraints in the subclafer (`[ drive.awd ]`), not as conditions at the top level (`[ tesla => tesla.drive.awd ]`).
- In temporal models, the context also bounds a constraint's lifetime (see Temporal models): keep a rule about a value or a state under that value or state.

### Make choices explicit

- To fix a configuration (an exemplar, a test case), state every choice in its clafer, also the negative ones (`[ not kind.maneuvers ]`, `[ no frame ]`). A fully fixed configuration has exactly one instance; more instances mean an open choice.
- Give each concept one clafer and one name; names resolve to the nearest clafer, so short local names are fine (`mode`, `speed`) as long as they are unambiguous in their context. Qualify (`camera.extras`) when they are not.
- State the intent in comments next to the choice or constraint, e.g., why a requirement is not induced.

## Models in several files

Split a model by concern: a library of types (components, interfaces), the model of a system that uses them, and its configurations, scenarios and tests.

```
//# OPTIONS --import=lib/car.cfr
tesla : Car
    …
```

- `--import=FILE` in the first line's `//# OPTIONS` (or on the command line) compiles the model together with the imported file. A path is relative to the importing file's folder; a URL is allowed, and its imports are relative to the URL.
- Imports are transitive: when `fleet.cfr` imports `lib/car.cfr`, which imports `components.cfr`, all three are compiled. Each file is compiled once, before the files that import it, even if several files import it. An import cycle is an error.
- All files share one namespace, as if they were concatenated: a clafer of one file can be used in the others, and the same top-level name in two files is an error.
- The options of all files are combined (the largest `--maxint`, the smallest `--minint`, all imports and modes).
- Errors give the file: `Compile error at lib/car.cfr:12:5`.
- A library file that only declares abstract clafers needs `//# OPTIONS --keep-unused` to be compiled on its own: unused abstract clafers are otherwise removed.
- Keep the reusable model free of choices that belong to one use: put a configuration, a scenario or an assertion about one case in its own file that imports the model. Then the base model stays general, and each case compiles and solves on its own.
- Compile and solve each top-level file (each configuration or scenario) separately; a model with several independent configurations makes the solver enumerate their combinations.

## Constraints

- A name alone means "some": `[ extras ]` is `[ some extras ]`.
- Names resolve to the nearest clafer: the descendants of the context first, then the ancestors (and their descendants), then the children of the context's reference target, then the top level. So inside a reference clafer, a name that an ancestor also has resolves to the ancestor's: `[ latency <= speed ]` under `link -> Port`, in a component that has ports with a `speed`, compares with the component's own ports. Write `this.speed` for the target's, and `parent.speed` for the enclosing one, to make the choice explicit.
- After an explicit `dref`, the names are the target's children: `this.dref.speed` is the target's `speed`, also when the reference clafer has a `speed` of its own, and it is the same as `this.speed` otherwise. (Before Clafer 0.5.7 it did not compile, gsdlab/clafer#108; write `this.speed` there.)
- `parent` goes up; `this.parent` of a top-level abstract clafer's instance is the parent of its subclafer's instance.
- At the top level, the parent is `root` (Clafer 0.5.7, gsdlab/clafer#112): in a top-level `b`, `parent.a` is `root.a`, i.e., the top-level `a`. The root has no instance in the outputs: `some root` holds and `#root` is 1, but it cannot be used as a set otherwise. `a.parent` of an optional `a` is the root when `a` exists, so `some a.parent` is `some a`. In an abstract clafer whose subclafers are at the top level and nested, take the parent of `this`, not of a variable. Before 0.5.7, `parent` at the top level gave outputs that do not run.
- Quantifiers: `all x : X | …`, `some`, `no`, `one`, `lone`, and `disj` for distinct variables. Several variables are separated by `;`, not `,`: `all disj a; b : Task | a.d != b.d`. `not` is the quantifier `no`; boolean negation is `!`.
- Set operators: `++` union, `**` intersection (`&` is deprecated), `--` difference, `in`, `.` join.
- Arithmetic: `+ - * / %`, `sum X.cost`, `product`, `min`, `max`, `#X`.
- `if … then … else …` needs parentheses on the right of `=`: `x = (if a then 1 else 2)`.
- `no (a && b)` does not compile (a quantifier over a boolean): write `!(a && b)`.
- A tautology such as `x = x` is rejected as redundant.
- `assert [ … ]` must hold in every instance (every trace); nested assertions hold for each instance of their clafer.
- `let v = e in body` is `body` with `e` for `v` (Clafer 0.5.8): it names an expression, not a value. In a temporal formula, `v` is re-evaluated in each state, so it cannot capture the value of the current state for a later one; use a quantifier (`all v : integer | this = v --> …`).

## Temporal models

A model with temporal operators or modifiers is dynamic. Its traces are lassos: a few states, then a loop back. Only the Alloy backend supports them: claferIG, and clafersg, which solves them with Alloy.

**States are clafers that come and go.** A state machine is an `xor` group; nesting gives hierarchical states:

```
final EU ?                         // a product-line feature: fixed within a trace
coin ?                             // an event: present or not in each state
xor machine
    initial idle -[ coin ]-> brewing
    brewing -->> ready
    ready --> idle
```

- **Modifiers:** `initial` means present when the parent appears; `final` means unchanged during a trace (shallow: mark each feature `final`, not just their parent); `finalref`, `finaltarget`.
- **Transitions:** `a --> b` is `a => X b`; `a -->> b` means that a lasts until b; the guards in `a -[ g ]-> b` and `a -[ g ]->> b` strengthen the source. A clafer may declare one transition after its name; more ways out of a state go in constraints in the state, e.g., `[ this -[ milk ]->> frothing ]` in `brewing`. The same constraint in the machine (`[ brewing -[ milk ]->> frothing ]`) lasts while the machine's parent exists, so it is not preempted with the machine.
- A transition only says what happens when it fires. It does not keep other clafers from changing, nor the state from changing without it: say so explicitly if needed.
- **Every constraint holds in every state** (an implicit `G`). So `[ sometime d ]` at the top level means "again and again"; for once, write `[ initially (sometime d) ]`. `initially` anchors to the first state of the context's lifetime, `finally` to its last.
- **Lifetimes:** a constraint's obligations (`X`, `F`, `G`, `U`) last while the parent of its context instance exists. So `[ req=down --> movingDown ]` inside the state `movingUp` leads to `movingDown`, and the pending transitions of nested states lapse when their enclosing state is left. To make a constraint last longer, place it higher or at the top level. Negated obligations lapse too: negations are pushed inside the temporal operators first, so `[ !(F error) ]` is `[ G !error ]` and lets the parent disappear. A formula and its negation can then both hold on a lifetime that ends before they are due.
- Instances may disappear and reappear, which cycles need. An instance never changes its parent, also across its absences.
- **Scopes count identities across the trace,** not instances in a state: a mandatory clafer whose instance is replaced by another needs scope 2, and a property that should survive the replacement of a machine quantifies over its instances or makes it `final`. The default scope analysis infers scopes from the multiplicities, which bound one state, so raise the scopes when checking temporal properties, and the trace length too (`tl`): a scenario of four steps has no trace with 3 states.
- **Patterns** read as requirements: `never p`, `sometime p`, `always p`, `lonce p`, `p must precede q`, `p must follow q`, with the scopes `before q`, `after q`, `between q and r`, `after q until r`.
- **Values change too:** `temperature -> integer` with `[ this = 0 -[ machine.heating ]-> this = 5 ]` in its own context. Keep all rules about one value under that clafer.

### Consequences of the semantics in practice

- **Where a constraint lives decides how long it lasts.** A constraint is checked for each instance of its clafer in each state in which the instance exists, and what it requires of later states lasts while the instance's parent exists. Put a rule about a state in the state, a rule about a value under the value, and a rule that must outlive a machine above the machine.
- **An instance has a value only while it exists** (Clafer 0.5.8). A requirement on the value of an instance in a later state, e.g., `[ X (this.dref = 7) ]` or `this = v --> this = v + 1`, also requires the instance to exist then. So a rule for every value keeps its clafer alive, and an optional clafer with such a rule cannot disappear while the rule applies. A negated comparison, `X (this.dref != 7)`, holds for an absent instance.
- **Every new instance is initialized again.** `:=` applies at each birth. If a mandatory clafer may be replaced by a fresh instance (scope 2 or more), the value starts over; rules that keep the instance alive, `final`, or a constraint such as `[ all v : val | G (v in val) ]` prevent the replacement.
- **The value of `:=` is evaluated where it is written**, in the parent's context, also its temporal operators (Clafer 0.5.8): in `x -> integer := if X (some flag) then 1 else 2`, `X (some flag)` is about the parent's `flag` in the next state, also when the parent disappears then.
- **Rules for values.** A rule that depends on the current value needs a quantifier, with parentheses, since a transition binds looser than the quantifier: `[ all v : integer | (this = v --> this = v + 1) ]`. Without them, `v` is unknown after `-->`.
- **One trace is one loop.** Behaviours that run side by side share the loop of the trace, so their periods need a common multiple within the trace length: counters with periods 5 and 6 need `tl` 30. A model that should have traces and has none may only need a longer `tl`.
- **`final` is shallow, and fixes identity.** `final` on a clafer fixes its instances while the parent exists, not its children; mark each part that must not change. `final` on an object keeps the object, not its non-final state.

### Scenarios and assertions

- **A chain is a sequence:** `a --> b --> c` is `a => X (b && X c)`. Its first state is only a condition.
- **A scenario that must be possible** is a constraint anchored with `initially`, in a file that imports the model:

  ```
  //# OPTIONS --import=machine.cfr
  [ initially (machine.idle && (machine.idle --> machine.brewing --> machine.ready)) ]
  ```

  A trace means "possible"; "no instances" means "impossible", and the unsat core lists the constraints that contradict it.
- **State the first state.** With the machine above, `[ initially (machine.idle && coin --> machine.ready) ]` has traces: it holds trivially when there is no coin at the start. And `[ initially (machine.idle --> machine.ready) ]` has traces too, since without a coin nothing keeps `idle` from going to `ready`. `[ initially (machine.idle && coin && (machine.idle --> machine.ready)) ]` has none, and its core is the scenario and `idle -[ coin ]-> brewing`.
- **An assertion is a property of all traces**, checked with `check`: `assert [ machine.ready --> machine.idle ]`.
- **Negative properties:** `never (req = down --> closed)` is wrong, since a transition holds trivially where its source does not. Write `assert [ req = down --> no closed ]` or `assert [ never (req = down && X closed) ]`.
- **From the start of the trace:** `assert [ initially ϕ ]`. `assert [ never !ϕ ]` means ϕ in every state.
- **Per-instance assertions:** a mandatory clafer can be replaced by another instance between states, so `a.b.c => X a.b.d` may fail at scope > 1. Quantify: `all b1 : a.b | b1.c => X (b1.d || no (b1 ** a.b))`.
- A quantified variable stays bound to its instance after the instance disappears, so `no e` is never true for it; test its existence with `no (e ** event)`.

### Precedence pitfalls

- Transitions bind loosest: `initially green --> yellow` is `(initially green) --> yellow`. Write `initially (green --> yellow)`.
- `X a || b` is `(X a) || b`. `never p && q` is `never (p && q)`.
- A clafer declared with `xor` right after a clafer with a transition continues that transition's expression: declare it first, or write the transition as a constraint.
- `X`, `F`, `G`, `U` and `W` are temporal operators, so a clafer cannot be named `X`: `abstract X -> integer` is a syntax error. The same holds for the pattern keywords (`never`, `sometime`, `always`, …).
- Sibling clafers on one line (`b : B A 3`) are a parse error. A line that starts with a comment is indented as if the comment were not there.

## Tools

**claferIG** (Alloy): `n` next solution, `d` diff view, `i`/`s` scopes, `m` maxint, `tl n` trace length (default 10), `fork k`, `config` (another configuration of the `final` parts, or another structure of a static model), `check [line]`, `dot`, `f name`, `scope`, `saveScopes`. When there are no instances, it prints an unsat core, the fastest way to find a contradiction. `--all` saves instances, `-n` limits them, `--addtypes` prints the super clafers.

**clafersg** (the Clafer solution generator; formerly `chocosolver`, a name that Clafer 0.6.0 no longer installs): `--file model.cfr`, `--repl --prettify`, `-n`, `--scope`, `--maxint`, `--scope-file`, `--addtypes`. It solves a static model with Choco and optimizes models with objectives; it solves a temporal model with Alloy, and its `--repl` is then claferIG's REPL, with `check`, `fork`, `config` and `tl`. It does not solve `real` and `double`.

**Counting instances:** neither backend removes all symmetric instances. claferIG may print several instances identically (they differ only in the atoms behind them, e.g., two systems with identical subtrees); Choco may give instances that differ only in which of two identical clafers a reference points to. Compare the instances, not their numbers, and fix choices so that a configuration has exactly one.

**Integers:** Alloy uses a bit width, and its integers wrap around; give the range in the model (`//# OPTIONS --maxint=1000`, and `--minint`) so that both backends use it. A generated `.als` run directly in Alloy uses 4-bit integers (−8..7) unless its scope says otherwise, which gives spurious counterexamples: prefer claferIG.

**Alloy directly**, to debug the encoding: `java -jar alloy6.2.0.jar exec -c <command|"*"> -t table -o - model.als`. It prints a table per solution or counterexample, and nothing otherwise. Its GUI (`gui`) steps through traces.

## When a model has no instances

1. Read the unsat core in claferIG; remove or weaken the listed constraints one at a time. The core leaves out the constraints of abstract clafers and their children (gsdlab/claferIG#46): if it names nothing that contradicts, look there, or move a suspect constraint into a concrete clafer to see it in the core.
2. Check the scopes (`scope`): a reference `-> T *` may need more targets than the default scope; raise it with `i`.
3. Check the integer range (`--maxint`): in Alloy, a constant out of range silently makes constraints false.
4. In temporal models: an unanchored scenario applies in every state; `[ sometime x ]` means infinitely often; a scenario whose first state the model does not force must state it; check `tl`.
5. Compare the backends: claferIG (Alloy) and clafersg (Choco) should agree on static models; a disagreement is a bug in one of them.
