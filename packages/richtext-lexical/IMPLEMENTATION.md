Hello!

This implementation adds **React Native rendering support** to the `@payloadcms/richtext-lexical` package by adding a similar entry point `@payloadcms/richtext-lexical/react-native` as the existing renderers (and specifically the React renderer) but with RN primitives.

> **Note:** This is my first contribution, so I would really appreciate feedback on both implementation decisions and documentation quality! I used the existing React renderer implementation as the base and inspiration for this RN renderer. If any direction here is not aligned with contribution expectations, I would appreciate pointers on the preferred approach!

The RN entrypoint is intended to provide a renderer-only API for serialized Lexical content in RN applications. I tried to keep the exposed API as close as possible to the existing React renderer so we can use it as a drop-in replacement in most cases, while still allowing for platform-specific behavior through converter and primitive overrides.

> **Note:** This implementation intentionally focuses on rendering serialized Lexical content in RN. It does not include editor UI components, plugin ports, or image responsiveness behavior. Mostly because I'm not too sure I could do it properly.

## Primitives

Unlike the React renderer, I introduced a layer of abstraction for primitives in the RN renderer. Since React can be expected to run in a DOM environment, the React renderer can safely assume that primitives like `div`, `span`, and `img` are available. In contrast, RN has a different set of primitives (`View`, `Text`, `Image`, etc.) that are not globally available in the same way. They need to be "manually" imported from `react-native` and can also be wrapped or customized by applications. And even then, some primitives (like `Text`) have specific behavior and nesting rules embedded in app-specific components.

So, instead of importing RN primitives directly in each converter, I created a `resolvePrimitive` utility that maps abstract primitive names (like `Text`, `View`, `Image`, etc.) to actual RN components. This mapping can be overridden through converter context, allowing for primitive injection for customization (such as _theming, analytics instrumentation, accessibility conventions, or navigation integration without having to rewrite all converters_).

Primitive resolution is performed once and passed through converter context. In practical terms, centralized primitive resolution improves consistency when users partially override primitives (for example, only replacing `Text` and `Pressable`) instead of having to reimplement the entire converter set. It also keeps converter implementations focused on node-specific logic rather than platform-specific component management.

## Supported Nodes and Converters

## Exposed API

To align with the API exposed by the React renderer, I've tried to mimic the same structure. The main entry point is the `RichText` component, which accepts serialized Lexical content and renders it using the RN converters and primitives. For users who need more control, the lower-level conversion functions (`convertLexicalToReactNative` and `convertLexicalNodesToReactNative`) are also exposed for direct use. I also expose primitive utilities and converter types to keep custom integrations type-safe and consistent with package defaults.

## Expected Usage Flow

The expected integration flow is kept similarly aligned with the React renderer. Data fetching is left to the user, and the package focuses on rendering serialized Lexical content via the `RichText` component. Developers can optionally provide `primitives` and `converters` overrides for customization, but the default set should cover most use cases. They can also define an `onExternalLinkPress` handler to manage external URL behavior explicitly, which is important in RN where URL handling can vary by environment. By default, external links will attempt to open using `Linking.openURL`, but providing an explicit handler allows for more control and consistency across platforms. **So, to recap:**

1. Users fetch serialized Lexical data from Payload.
2. Render it with `RichText` from `@payloadcms/richtext-lexical/react-native`.
3. Provide `onExternalLinkPress` for explicit external URL behavior.
4. Add `primitives` overrides when integrating with an app design system.
5. Add `converters` overrides when default node behavior is insufficient.

## Edge cases, risks, and concerns

### Unknown or custom node types

If stored content includes node types with no registered converter, the output may be missing for those nodes. Currently, the default behavior is to render nothing for unsupported nodes, which could lead to silent content loss if not carefully managed. I haven't had time to look into the other implementation yet for how other implementations handle this, but I would be open to implementing a stricter default behavior (for example, rendering a placeholder or warning in development) if you think that would be more appropriate.

### Internal links

Internal links are not resolved by default in the RN renderer. I've left it for the developers to override the default `Link` converter and implement app-specific navigation logic, as internal link resolution is highly dependent on app routing structure and navigation libraries. In the React renderer, the exposed Link converter allows to provide a `internalDocToHref` prop for internal link resolution for simplicity. I'm not sure whether to implement this or not. The only reason I can see to implement it is to keep feature parity between renderers. I've elected not to implement it for now, due to lack of time.

### Large document trees and render performance

Obviously enough, the recursive conversion of large trees increases render cost and affects scroll performance on low-end devices. This is a general concern for rich text rendering in any environment, but especially in RN where JS thread performance is usually(?) more constrained. I will try to add profiling tests and metrics to identify specific bottlenecks and optimize converter implementations where possible, but I'm not confident enough, nor am I sure if this falls within the scope of this feature request itself. Would be happy to hear feedback on this.

### Tables

Tables in the renderer are currently implemented with basic `View` and `Text` primitives, which may not support all desired table features (like fixed headers, responsive layouts, or complex cell spanning). Cards on the table (no pun intended), it's a barebones implementation that covers basic rendering but may not meet all use cases. That's done intentionally. I hoped to get feedback on whether this is sufficient for the initial implementation or if someone could help with a better approach.

### Images

Web-specific responsive image behavior is intentionally not replicated in RN. The React renderer benefits from browser-native features like CSS media queries and automatic image scaling through srcset attributes. React Native has no equivalent mechanism. Image sizing must be explicitly specified or calculated at runtime. Additionally, Payload's upload metadata (dimensions, orientation, file size) should be leveraged by application code to implement appropriate RN-specific behaviors (such as preloading, caching strategies, or adaptive quality selection based on network conditions). Document this gap clearly so developers don't expect parity with web renderers.

### URL Handling Defaults

When `onExternalLinkPress` is omitted, runtime-level URL handling may vary by environment. Recommend always providing an explicit handler in production applications. React Native's `Linking.openURL` behavior differs across iOS and Android, and behavior is undefined in non-native environments (web, SSR contexts, testing). Defaulting to `Linking.openURL` without an explicit handler can lead to silent failures or platform-specific bugs. By requiring developers to provide an `onExternalLinkPress` handler, the package shifts responsibility to the consumer while maintaining safety and predictability.

### Schema and Converter Drift Over Time

If Lexical node schema evolves and converter coverage is not updated in lockstep, render gaps can appear. Keep converter updates part of schema change workflows and add compatibility checks in tests where possible. This is especially critical for a rendering-only package: if Payload's editor supports a new node type but the RN renderer lacks a converter, existing content will silently fail to render. Establish a maintenance pattern where Lexical schema changes trigger corresponding converter additions and backward-compatibility tests. Consider adding a development-time warning or error boundary to help developers identify missing converters during local testing.

## Questions for Maintainers

I would appreciate maintainer guidance on the following points:

1. Do we want a stricter default behavior for unsupported node types (for example, warnings in development), or should silent fallback remain the default?
2. Should internal link resolution remain entirely app-defined, or do maintainers want an optional helper pattern in this package?
3. Is the current API split between high-level and low-level conversion functions aligned with long-term package direction?
4. Are there project-specific performance thresholds or benchmark fixtures you would like this renderer to satisfy before broader rollout?
5. Should table and upload defaults remain intentionally minimal, or is there interest in additional opinionated native defaults?

If any of these questions indicate the wrong direction, I would appreciate pointers before we expand behavior further.

## Current Status

The documentation now includes:

- Technical rationale for architecture decisions.
- Public API summary and intended usage.
- A simple reproducible usage example.
- Expected integration flow.
- Known edge cases and operational concerns.

I can follow up with targeted tests or additional implementation refinements based on maintainer feedback.
