- Start Date: 2017-05-05
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Introduce syntax for passing in multiple named template blocks into a component.

# Motivation

There are limitations to composition due to the inability to pass more than one block to a component (or 2 blocks if you include the inverse block).

The result of this is that Ember developers have this ultra-powerful, compositional API for overriding portions of a component, but they can only use it in one place in the component invocation; any remaining overrides/configuration needs to be expressed as data and passed in as attributes to the component when it'd be vastly preferable to just pass in a chunk of DOM.

Example:

```
{{x-modal headerText=page.title as |c|}}
  <p>Modal Content {{foo}}</p>
  <button onclick={{c.close}}>
     Close modal
  </button>
{{/x-modal}}
```

This works, but the moment you need to render a component in the header (rather than just `headerText`), you end up having to add more config/data/attrs to `x-modal` just to support every one of those overrides, when really you just should be able to pass in another block of DOM to define what the header looks like. The API in this proposal would allow you to express this use case via:

```
{{x-modal as |c|}}
  <p>Modal Content {{foo}}</p>
  <button onclick={{c.close}}>
     Close modal
  </button>
{{as @header |c|}}
  {{page.title}}
  {{status-indicator status=status}}
  {{close-button action=c.close}}
{{/x-modal}}
```

Other RFCs/addons that have attempted to address this:

- [named yields](https://github.com/emberjs/rfcs/pull/72)
- [multiple yields](https://github.com/emberjs/rfcs/pull/43)
- [yet another named yields rfc](https://github.com/emberjs/rfcs/pull/193)
- [ember-block-slots](https://github.com/ciena-blueplanet/ember-block-slots)
- [ember-named-yields](https://github.com/knownasilya/ember-named-yields)
- [local template blocks](https://github.com/emberjs/rfcs/pull/199)

# Detailed design

## Curly + Angle-bracket Syntax

The Glimmer/Handlebars parser will need to be enhanced to support the following curly/angle-bracket syntax (both examples are functionally equivalent):

```hbs
{{#x-foo as |a|}}
  @default block {{a}}
{{as @header |h|}}
  @header block. Referenced via @header in x-foo's layout
{{as @footer}}
  @footer block. No block params.
{{/x-foo}}

{{#x-foo as @header |h}}
  @header block.
  This invocation of x-foo omits the @default block.
{{/x-foo}}
```

```hbs
<x-foo as |a|>
  @default block {{a}}
<as @header |h|>
  Header block. Referenced via @header in x-foo's layout
<as @footer>
  @footer block. No block params.
</x-foo>

<x-foo as @header |h|>
  @header block.
  This invocation of x-foo omits the @default block.
</x-foo>
```

## Blocks as `@arg`s

Presently, blocks passed to templates exist in their own namespace, separate from properties that might also be passed into a component. This RFC proposes that blocks are passed in as any other argument passed to a component. We're using the `@`-prefixed args syntax introduced by Glimmer, even for classic curly components.

Two results of this is that it's trivial to forward a block to an inner component, and that you don't need to use `hasBlock` to check if a block was provided, e.g.

```hbs
// components/x-modal.hbs
{{#if @header}}
  {{fancy-header @default=@header}}
{{/if}}
```

## Yielding to a named block

```
{{yield data to=@header}}
{{yield data to=@footer}}
```

## `{{else}}`, and `@default/@else`

`{{else}}` is sugar for `{{as @else}}`. The classic `{{else}}` syntax will not be expanded to support block params; if you want block params use `{{as @else |a b c|}}`.

The angle-bracket form `<as @else>` or `<as @else |a b c|>` can also be used, but we will _not_ support angle-bracket `<else>` since it might syntactically collide with a future HTML element named `<else>`. This shouldn't be an issue though since curly `{{else}}` can still be used for `{{#if ...}} {{else}} {{/if}}` constructs, and any use cases where `{{else}}` + `{{yield to="inverse"}}` had been used with custom components would be better served by choosing a more fitting name for the block and using `<as ...>`, e.g. `<as @empty>No items to display...`.

That said, `{{else}}` does have the benefit of being a generic, conventional catch-all (even if it is a bit of a stretch for how some components use it), and we'd risk opening the door to addon authors all choosing slightly different names for the same-ish thing.

# How We Teach This

We teach this as a followup to classic block syntax; once the user is comfortable with single-block syntax, we can introduce named block syntax for more complex use cases.

# Drawbacks

The `<as @foo>` separator is an unusual departure from how HTML normally looks; we can argue that unclosed `<p>` and `<li>` tags are similar spec-compliant HTML syntax, but given that Glimmer doesn't presently parse those, it's fair to say these separators will catch people off guard at first. We should weigh in community response to this to see if there's possible something less jarring.

Also, there are early reports that some editors' auto-indentation logic
doesn't play nicely with this new syntax, so the various Handlebars
formatting plugins would probably need to be upgraded to accommodate
this syntax.

# Alternatives

I'd proposed a JSX-y [attr/component-centric](https://github.com/emberjs/rfcs/pull/203) syntax for passing what are essentially DOM lambdas, rendered with `{{component}}`. Perhaps we'll add something like that feature in the future, but it's a much less natural enhancement to Ember than named blocks.

# Unresolved questions

Curly components will need to support `@`-prefixed `@args` to some degree in order to support the proposed named block syntax, but I'm not sure how deep an impact that would have, or whether `@args` will be merged with classic props.

# Considerations for Future RFCs

There's been talk of possibly including named block params as part of
this RFC. For example:

```hbs
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

```hbs
// usage
{{#x-foo @default=(my-cool-header)}}
```

where `my-cool-header` is a component that accepts `@title` as a param.
This would provide a path toward block/component unification that would
provide nice refactoring capabilities, nudge the developer into using
consistent naming conventions, and not force the developer to
arbitrarily choose between two similar but incompatible APIs for passing
chunks of dynamic DOM into a copmonent.

