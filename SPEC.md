# fprompting spec (v0)

Prompting in a functional programming style. This spec covers two ideas and
sketches the TypeScript API that the first library version will implement.

Out of scope for now: combinators (retry, fallback, map over lists), prompts
as data, agents as folds.

## 1. Prompts are typed pure functions

A prompt is a function from a declared input type to a declared output type.

```
summarize : Article -> Summary
```

Rules:

- **Declared types.** Every prompt names an input type and an output type. Both
  have a runtime schema, so input is validated before rendering and the model's
  reply is parsed into the output type.
- **A template, nothing else.** The text sent to the model is computed only
  from the input value (and the system prompt, see below). Rendering is a pure
  function: same input, same text.
- **No hidden context.** No chat history, no global state, no memory carried
  between calls. If a prompt needs something, it is part of its input type.
- **The model is injected, not captured.** A prompt describes a call; it does
  not own a client. Running it takes a `Model` explicitly. The model call is
  the one effect, and it lives at the edge.
- **System prompts are partially applied arguments.** A system prompt is fixed
  ahead of time, like the first argument of a curried function. Parameterized
  prompts work the same way: `translate("pl")` returns a
  `Prompt<Summary, Translation>`.

We treat the model as a function of the rendered text only. It is not truly
deterministic, but nothing else may influence the call.

## 2. Composition instead of mega-prompts

Instead of one large prompt that does everything, write small prompts and chain
them.

```
translate("pl") ∘ summarize ∘ extractFacts
```

Rules:

- **Types must line up.** Composing `Prompt<A, B>` with `Prompt<B, C>` gives
  `Pipeline<A, C>`. A mismatch (e.g. feeding `Facts` into something that
  expects `Summary`) is a TypeScript compile error, so it is caught before any
  model call.
- **Every boundary is checked at runtime too.** Each stage's reply is parsed
  with its output schema. If parsing fails, the pipeline stops with an error
  that names the stage, and the next model call never happens.
- **Pipelines are prompts.** A `Pipeline<A, C>` can be run, and composed
  further, exactly like a single `Prompt<A, C>`.
- **Each stage stays small and testable.** Every prompt can be run and tested on
  its own, with a fake `Model`.

## 3. TypeScript API sketch

```ts
// Anything with a parse method. Zod schemas satisfy this as is.
interface Schema<T> {
  parse(value: unknown): T;
}

// The only effect. Implementations wrap a real API or a test fake.
interface Model {
  complete(request: { system?: string; prompt: string }): Promise<string>;
}

interface Prompt<I, O> {
  readonly name: string;
  readonly input: Schema<I>;
  readonly output: Schema<O>;
  readonly system?: string;
  render(input: I): string; // pure
}

type Pipeline<I, O> = Prompt<I, O>; // same shape, so it composes

// Build a prompt.
function prompt<I, O>(spec: {
  name: string;
  input: Schema<I>;
  output: Schema<O>;
  template: (input: I) => string;
  system?: string;
}): Prompt<I, O>;

// Partially apply a system prompt.
function withSystem(system: string): <I, O>(p: Prompt<I, O>) => Prompt<I, O>;

// Left to right, typed with overloads. pipe(f, g, h) means h ∘ g ∘ f.
function pipe<A, B, C>(f: Prompt<A, B>, g: Prompt<B, C>): Pipeline<A, C>;
function pipe<A, B, C, D>(
  f: Prompt<A, B>, g: Prompt<B, C>, h: Prompt<C, D>,
): Pipeline<A, D>;
// ... more arities

// Right to left, for readers who prefer the math order.
function compose<A, B, C>(g: Prompt<B, C>, f: Prompt<A, B>): Pipeline<A, C>;

// Run against a model. Throws StageError { stage, cause } on a bad
// input, a failed parse, or a model error.
function run<I, O>(p: Prompt<I, O>, model: Model): (input: I) => Promise<O>;
```

Example, matching step 3 of the plan:

```ts
const extractFacts = prompt({
  name: "extractFacts",
  input: Article,
  output: Facts,
  template: (a) => `List the factual claims in this article:\n\n${a.body}`,
});

const summarize = prompt({
  name: "summarize",
  input: Facts,
  output: Summary,
  template: (f) => `Summarize these facts in 3 sentences:\n${f.items.join("\n")}`,
});

const translate = (lang: string) =>
  prompt({
    name: `translate(${lang})`,
    input: Summary,
    output: Translation,
    template: (s) => `Translate into ${lang}:\n\n${s.text}`,
  });

const polishBrief = pipe(extractFacts, summarize, translate("pl"));
// pipe(summarize, extractFacts) does not compile: Summary is not Facts.

const brief = await run(polishBrief, claude)(article);
```

## 4. Output parsing

- `run` appends a short instruction asking for JSON matching the output schema.
  The template author does not write that part.
- The reply is parsed as JSON, then with the output schema. Plain string outputs
  can skip JSON with a `text()` schema.
- No automatic retries in v0. Retrying is a combinator and comes later.

## Open questions

- Should `run` return a `Result` instead of throwing?
- Should we require Zod, or keep the minimal `Schema<T>` interface?
- How should a schema describe itself to the model (JSON Schema, an example
  value, or a hand written hint)?
