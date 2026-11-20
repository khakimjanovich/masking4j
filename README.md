# masking

Used by observability (every log line, every client call) and error (its log line). The library
knows only generic names; a service configures its logging masking via `org4j.observability.logging.masking.*` in YAML or Vault,
and no app's field name is hardcoded here.

```groovy
implementation libs.masking
```

Masks sensitive fields across JSON, XML, forms, URI query parameters, and plain text patterns (`key=value`, `key: value`, `key value`, `/key/value`).

### What a service configures

Nothing is masked by default (`Masking.defaults()` is `Masking.empty()`). The library knows no field
name and no value pattern: each service sets all of them (`MaskingRules`), through
`org4j.observability.logging.masking.*` in YAML or Vault.

| Rule | What it masks | How |
|---|---|---|
| `fields` | values under these names (case-insensitive; `card.pan` matches `pan`) | the base hook: 10 characters or more keep the first 6 and last 4 (`860030......7890`); shorter ones are masked whole (`...`) |
| `fullFields` | values under these names | every character, at most 10 mask characters (`......` for six, `..........` for 32); a name in both lists is masked whole |
| `values` | every match of these patterns, in any text, with or without a name | the base hook |

Names are found in JSON, XML (elements and `<name>`/`<value>` items), forms, URI queries and paths
(`/pan/8600…`, at any depth), and plain text (`key=value`, `key: value`, `key value`). Patterns run
on every result: text, URIs and JSON, XML and form bodies. The mask character is configurable (`*`).

```java
var masking =
    new Masking(
        new MaskingRules(
            ".",
            Set.of("pan", "cardToken"),
            Set.of("password", "client_secret", "cvv"),
            List.of(Pattern.compile("(?<!\\d)\\d{13,19}(?!\\d)"))));
masking.body(bytes, "application/json");
masking.uri(uri);
masking.text(message);
masking.mask("client_secret", value); // one value under a name, by that name's rule
```

## Tests

Unit tests only (`./gradlew test`): the library consists of pure functions with no
infrastructure dependencies. `MaskingTest` is the specification, one nested group per behaviour:
nothing configured, the base hook, full fields, value patterns, and every format (JSON, XML,
forms, plain text, URI queries and paths, other content types, cutting). Wiring into logs is
verified by observability's integration test.
