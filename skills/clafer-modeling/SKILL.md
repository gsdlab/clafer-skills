---
name: clafer-modeling
description: Write, structure, fix and check Clafer models (.cfr), static or temporal, and run them with clafer, claferIG and chocosolver. Use when asked to model something in Clafer, to split a model into files, to debug a model that has no instances or traces or that does not compile, to write scenarios or assertions, or to explain what a Clafer construct means.
---

# Modeling in Clafer

Clafer models structure as a hierarchy of clafers, which are features, classes and states at once, with constraints over it.
A static model has instances; a temporal model has traces of states, each of them an instance.
The language reference is the CheatSheet (`doc/CheatSheet.md` in the clafer repository).
This guide is for Clafer 0.5.7.

## Workflow

1. Write the smallest model that shows the idea, then grow it. Most models of a few clafers solve in about a second; a model that takes minutes is usually contradictory or has scopes that are too large.
2. Compile it: `clafer model.cfr` (`-m alloy` or `-m choco` to see the generated models). Errors come as `model.cfr:line:column`.
3. Look at instances: `claferIG model.cfr` (Alloy) or `chocosolver --file model.cfr --repl --prettify` (Choco). Look at several instances (`n`), not just the first: they show what the model leaves open.
4. State what must hold as assertions and run `check` in claferIG. "No counterexample" holds only within the scopes (and the trace length).
5. For a configuration or behaviour that should be possible, write a scenario in a file that imports the model, and check that it has an instance; also check that a variant that should be impossible has none.

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
- **References:** `-> T` is a set (no repeated targets among the instances of a clafer under one parent), `->> T` a bag. Where a value is expected, a reference stands for its target (`[ speed = 3 ]` means `speed.dref = 3`), and a comparison of two references compares their targets (`this.a = parent.b.a`, Clafer 0.5.7). To compare the reference instances themselves, compare `this` or a variable, or write `dref` on one side.
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
- Abstract clafers can be nested under concrete clafers too (`System` with `abstract Port` and `p1 : Port`).

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
  - the value: an initializer `fan -> integer = 2`, or a constraint `[ speed = 3 ]`;
  - modifiers: `final` on the abstract clafer is inherited by its refinements. To add `final` in a refinement, restate the reference: `final motor : engine -> Engine`. With only `final motor : engine`, the inherited reference may still change during a trace (gsdlab/clafer#109).
  An abstract nested clafer that a subclafer does not refine has no instances in it: the subclafer simply lacks that part.
- **Constraints read like requirements.** A name alone means "some" (`[ encryption ]`), a reference stands for its value (`[ speed = 3 ]`), and names resolve from the context, so most constraints need neither quantifiers nor paths.
- **One model, many views.** The same model is a feature model to configure (claferIG, chocosolver, the configurator), a class model to instantiate, a product line with objectives to optimize (chocosolver), and, with temporal modifiers, a behaviour to simulate and check (claferIG).
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

## Temporal models

A model with temporal operators or modifiers is dynamic. Its traces are lassos: a few states, then a loop back. Only the Alloy backend (claferIG) supports them.

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
- Sibling clafers on one line (`b : B A 3`) are a parse error. A line that starts with a comment is indented as if the comment were not there.

## Tools

**claferIG** (Alloy): `n` next solution, `d` diff view, `i`/`s` scopes, `m` maxint, `tl n` trace length (default 10), `fork k`, `config` (another configuration of the `final` parts, or another structure of a static model), `check [line]`, `dot`, `f name`, `scope`, `saveScopes`. When there are no instances, it prints an unsat core, the fastest way to find a contradiction. `--all` saves instances, `-n` limits them, `--addtypes` prints the super clafers.

**chocosolver** (Choco): `--file model.cfr`, `--repl --prettify`, `-n`, `--scope`, `--maxint`, `--scope-file`, `--addtypes`; it optimizes models with objectives. It supports neither temporal models nor `check`.

**Counting instances:** neither backend removes all symmetric instances. claferIG may print several instances identically (they differ only in the atoms behind them, e.g., two systems with identical subtrees); chocosolver may give instances that differ only in which of two identical clafers a reference points to. Compare the instances, not their numbers, and fix choices so that a configuration has exactly one.

**Integers:** Alloy uses a bit width, and its integers wrap around; give the range in the model (`//# OPTIONS --maxint=1000`, and `--minint`) so that both backends use it. A generated `.als` run directly in Alloy uses 4-bit integers (−8..7) unless its scope says otherwise, which gives spurious counterexamples: prefer claferIG.

**Alloy directly**, to debug the encoding: `java -jar alloy6.2.0.jar exec -c <command|"*"> -t table -o - model.als`. It prints a table per solution or counterexample, and nothing otherwise. Its GUI (`gui`) steps through traces.

## When a model has no instances

1. Read the unsat core in claferIG; remove or weaken the listed constraints one at a time. The core leaves out the constraints of abstract clafers and their children (gsdlab/claferIG#46): if it names nothing that contradicts, look there, or move a suspect constraint into a concrete clafer to see it in the core.
2. Check the scopes (`scope`): a reference `-> T *` may need more targets than the default scope; raise it with `i`.
3. Check the integer range (`--maxint`): in Alloy, a constant out of range silently makes constraints false.
4. In temporal models: an unanchored scenario applies in every state; `[ sometime x ]` means infinitely often; a scenario whose first state the model does not force must state it; check `tl`.
5. Compare the backends: claferIG and chocosolver should agree on static models; a disagreement is a bug in one of them.
