- Start Date: 2017-05-05
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Introduce syntax for passing in multiple named template blocks into a component.

# Motivation

There are limitations to composition due to the inability to pass more than one block to a component (or 2 blocks if you include the inverse block).

The result of this is that Ember developers have this ultra-powerful, compositional API for overriding portions of a component, but they can only use it in one place in the component invocation; any remaining overrides/configuration needs to be expressed as data and passed in as attributes to the component when it'd be vastly preferable to just pass in a chunk of DOM.

Example:

```html
{{x-modal headerText=page.title as |c|}}
  <p>Modal Content {{foo}}</p>
  <button onclick={{c.close}}>
     Close modal
  </button>
{{/x-modal}}
```

This works, but the moment you need to render a component in the header (rather than just `headerText`), you end up having to add more config/data/attrs to `x-modal` just to support every one of those overrides, when really you just should be able to pass in another block of DOM to define what the header looks like. The API in this proposal would allow you to express this use case via:

```html
{{x-modal}}
  <@header as |c|>
    {{page.title}}
    {{status-indicator status=status}}
    {{close-button action=c.close}}
  </@header>

  <@main as |c|>
    <p>Modal Content {{foo}}</p>
    <button onclick={{c.close}}>
       Close modal
    </button>
  </@main>
{{/x-modal}}
```

and with Glimmer components:

```html
<x-modal>
  <@header as |c|>
    {{page.title}}
    {{status-indicator status=status}}
    {{close-button action=c.close}}
  </@header>

  <@main as |c|>
    <p>Modal Content {{foo}}</p>
    <button onclick={{c.close}}>
       Close modal
    </button>
  </@main>
</x-modal>
```

Other RFCs/addons that have attempted to address this:

- [named yields](https://github.com/emberjs/rfcs/pull/72)
- [multiple yields](https://github.com/emberjs/rfcs/pull/43)
- [yet another named yields rfc](https://github.com/emberjs/rfcs/pull/193)
- [ember-block-slots](https://github.com/ciena-blueplanet/ember-block-slots)
- [ember-named-yields](https://github.com/knownasilya/ember-named-yields)
- [local template blocks](https://github.com/emberjs/rfcs/pull/199)

# Detailed design

## Multi-block Syntax

Both curly and angle-bracket component invocation syntax will be enhanced
with a nested syntax for passing multiple blocks into a component. Here
is what that syntax looks like for angle-bracket / Glimmer components:

```html
<x-foo>
  <@header>
    Howdy.
  </@header>

  <@body as |foo|>
    Body {{foo}}
  </@body>
</x-modal>
```

and for classic curly component invocation syntax:

```html
{{#x-foo}}
  <@header>
    Howdy.
  </@header>

  <@body as |foo|>
    Body {{foo}}
  </@body>
{{/x-foo}}
```

As demonstrated above, the _nested_ syntax for both curly and
angle-bracket multi-block syntax has the format `<@blockName>...</@blockName>`.
This multi-block syntax cannot be mixed with other syntaxes; either ALL
the nested "children" of a component invocation need to be
`<@blockName>...</@blockName>` (multi-block syntax), or none of them do
(classic single-block syntax). The presence of any non-whitespace
character between or around `<@blockName>...</@blockName>`s is a
compile-time error.

The purpose of the `<@blockName>...</@blockName>` syntax is to
hint/remind/reinforce that in this new model, blocks are just `@arg`s
passed into a component; in the following example, `<x-foo>` is being
passed two `@arg`s, a `@title` arg whose value is string `"Hello"`
and a `@body` arg whose value is a template block:

```html
<x-foo @title="Hello">
  <@body>
    I am some text.
  <@body>
</x-foo>
```

The `@arg` syntax itself was developed as part of the Glimmer API to
distinguish data being passed to a component from attributes being
set on a root HTML element. This RFC marks the introduction of the `@arg` syntax
to classic curly components, but we haven't yet fleshed out how fully
we want to merge `@arg` semantics into the world of classic curly components; this
is one of the major challenges that remain with this RFC.

### Rendering Blocks

The `@`-based syntax enables us to introduce a much
simpler, cleaner syntax for rendering blocks:

```html
Render a block:
{{@blockName}}

Render a block with block params:
{{@blockName 1 2 3}}
```

One nice thing about this syntax is that it works with any kind of
"renderable", including simple strings, such that given the following
layout:

```html
<div class="modal-header">{{@header}}</div>
<div class="modal-content">{{@body}}</div>
```

the following two invocations are both valid:

```html
<x-modal @header="Error!">
  <@body>
    Something went wrong.
  </@body>
</x-modal>

<x-modal>
  <@header>
    {{fa-icon "error"}}
    Error!
  </@header>

  <@body>
    Something went wrong.
  </@body>
</x-modal>
```

This helps to unify the syntax for rendering dynamic values to DOM
and supports a common workflow where string args can be promoted
to full-on blocks without having to rework the component code
to support an alternative/parallel API.

#### OPEN QUESTION: "calling" string args

How should we handle `{{@header some blockparam data}}`
when `@header` is a non-"callable", simple value like a string? Should
we loudly error? Should it just ignore those "extra" args and render
the string without a fuss? There are arguments on both sides, but we
need to decide what our policy on quiet/soft-failures is, since this
can lead to frustrating debugging experiences where nothing renders
and there's no indication in the console as to what went wrong.

#### OPEN QUESTION: block-invocation syntax for {{@arg}}

Since an `@arg` might be a component factory, we need to decide what the
syntax for rendering `@header` and passing it a template looks like. (A
real life use case of this is how [ember-power-select let's you pass
in component factories to override portions of the
component.](https://github.com/cibernox/ember-power-select/blob/2275fbbc2c8d78c1415b3ecf92c09a4bd8247976/addon/templates/components/power-select/options.hbs#L8-L12))

There are a few candidates thus far:

```
{{#@group as |a b c|}}
  ...
{{/@group}}

{{# @group as |a b c|}}
  ...
{{/ @group}}

{{#render @group as |a b c|}}
  ...
{{/render}}
```

`render` looks the nicest, but it would mean resurrecting the `#render`
helper from the deprecated dead in order to fit this one syntactical
corner case. It might be alright to just put up with this one ugly
syntax and punt a nicer alternative (perhaps involving `render`) to a future RFC.

### @main and @else

Classically, component invocation has only supported passing up to two
blocks, the "default" block and the "inverse" block. For the purposes
of unifying rendering and compositional semantics, these blocks
will be accessible under the respect arg names `@main` and `@else`.

```hbs
{{@main}}
is equivalent to:
{{yield}}

{{@main 1 2 3}}
is equivalent to:
{{yield 1 2 3}}

{{@else}}
is equivalent to:
{{yield to="inverse"}}
```

One beneficial implication of this is a nice story for introducing
additional blocks to a component invocation that started off using
single-block syntax.  For instance, the
`<x-modal>` component above could have been written as:

```html
<x-modal @header="Error!">
  Something went wrong.
</x-modal>
```

and then expanded at a later time to

```html
<x-modal>
  <@header>
    {{fa-icon "error"}}
    Error!
  </@header>

  <@main>
    Something went wrong.
  </@main>
</x-modal>
```

To demonstrate how deeply this syntax integrates with core/classic Ember features,
it's worth pointing out the following equivalence (even though no one
would/should ever do this in practice):

```html
{{#if foo}}
  True
{{else}}
  False
{{/if}}

is equivalent to

{{#if foo}}
  <@main>
    True
  </@main>

  <@else>
    False
  </@else>
{{/if}}
```

# How We Teach This

We teach this as a followup to classic block syntax; once the user is comfortable with single-block syntax, we can introduce named block syntax for more complex use cases.

We teach that what `<@blockName></@blockName>` syntax really means is
that we're just passing in an arg named `@blockName`, which is like
any other arg we might pass into a component, but it happens to point
to a template block than, say, some simple string value.

# Drawbacks

This isn't really anything like the
[WebComponents slot syntax](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/slot)
that intends to address similar use cases, so there is some risk of
introducing an API that doesn't fit in with what the rest of the world
is doing.

Some syntax highlighters might have trouble with this syntax; all
the editors I've tried it on look reasonable, but GitHub's Handlebars
parser isn't too kind:

```hbs
<x-modal>
  <@header as |c|>
    ...
  </@header>

  <@main as |c|>
    ...
  </@main>
</x-modal>
```

### Semantics of @args + {{curly-components}}

Since this RFC is the first to introduce `@arg`s to curly components, we need to decide
on some basic semantics of integrating these two worlds, particularly in
regards to how/whether `@arg`s are exposed in JavaScript Component
definitions (e.g. via `this.get()` or some other means).

One constraint is that since `@main` is the name of the block arg that's
going to be passed to any component invoked with classic single-block syntax,
we can't just suddenly start setting the `main` property on everyone's
component instances, since that would be a breaking change.

The most agreeable rule so far is that `@args` live parallel to the
"hash" args that get set as properties directly on the component. To
demonstrate these semantics (with a pathological use case that should
never happen in practice):

JS

```
export default Component.extend({
  bar: 'jsbar',
});
```

Layout:

```hbs
bar={{bar}}
@bar={{@bar}}
```

Invocation:

```hbs
{{x-foo}}

{{x-foo bar="HBSBAR"}}

{{x-foo @bar="HBSBAR"}}
```

Result:

```
bar=jsbar
@bar=

bar=HBSBAR
@bar=

bar=jsbar
@bar=HBSBAR
```

So, while the above demonstrates that `@arg`s will technically be
supported with curly components, they should only really be used for
the purpose of rendering multiple blocks, or forwarding a block
to another component for rendering.

For the time being, there is no way, within a curly
component's JS class definition, to access `@args`; we may want to add that
functionality soon, but it is out of scope of this RFC.

Note that it might make sense to move this discussion to the
[hasBlock.js](https://github.com/emberjs/rfcs/pull/102) RFC, which
proposes adding a JS API for determining whether a block was provided;
given that blocks will now be passed in as args, this RFC might be
better generalized to providing a curly JS API for accessing `@arg`s.

# Alternatives

I'd proposed a JSX-y [attr/component-centric](https://github.com/emberjs/rfcs/pull/203) syntax for passing what are essentially DOM lambdas, rendered with `{{component}}`. Perhaps we'll add something like that feature in the future, but it's a much less natural enhancement to Ember than named blocks.

# Unresolved questions

Most unresolved questions have been listed above as "OPEN QUESTION"s.

# Considerations for Future RFCs

There's been talk of possibly including named block params as part of
this RFC. For example:

```html
// x-foo layout.hbs
{{yield @otherStuff=123 @title="hello"}}

// usage
{{#x-foo as |@title|}}
  <h1>{{@title}}</h1>
{{/x-foo}}
```

This API lays the foundation for unifying positional/named args that has
divided the block vs `attr=(component)` API. At some point in the
future, we might allow an API like this:

```html
// usage
{{#x-foo @default=(my-cool-header)}}
```

where `my-cool-header` is a component that accepts `@title` as a param.
This would provide a path toward block/component unification that would
provide nice refactoring capabilities, nudge the developer into using
consistent naming conventions, and not force the developer to
arbitrarily choose between two similar but incompatible APIs for passing
chunks of dynamic DOM into a copmonent.

