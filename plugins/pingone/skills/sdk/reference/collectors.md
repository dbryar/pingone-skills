# Collector catalogue

Read alongside `SKILL.md`. This file is the shape tables; the decisions are there.

Three clients build collectors out of the same `form.components.fields` response and expose three different vocabularies from it. A form is renderable only to the extent every client that will render it has a collector for every field.

**For which field maps to which collector, on which platform, at which SDK version, use Ping's maintained matrix: <https://developer.pingidentity.com/orchsdks/davinci/compatibility.html>.** This file is the other half: how a collector is read and written once you have one, and the property and method names that are wrong if guessed. The matrix carries neither.

## Versions these tables were read against

| Client | Artefact | Version | Read from |
| --- | --- | --- | --- |
| Web | `@forgerock/davinci-client` | 2.1.1 | `collector.types.d.ts` and the implementation switch, 02-09-2026 |
| Android | `com.pingidentity.sdks:davinci` | 2.1.0 | `CollectorRegistry` and `Form.parse`, 31-08-2026 |
| iOS | `Ping/ping-ios-sdk` | 2.x | Ping's published agent skills, with collector names corroborated by the compatibility matrix |

`@forgerock` is legacy branding on a current package, not a legacy package. `@pingidentity/davinci-client` does not exist. The package to avoid is `@forgerock/javascript-sdk` (the older Auth Journey SDK), and on Android `org.forgerock:forgerock-auth`.

## Web: the role dispatch

Every collector maps to exactly one role, and the renderer has one function per role. The dispatch keys on `type` first and `category` only as a fallback, because `SubmitCollector` and `FlowCollector` share `category: "ActionCollector"` and must not share a role.

| Role | Collector types (2.1.1) | Draws as | Reports through |
| --- | --- | --- | --- |
| `text` | `TextCollector`, `ValidatedTextCollector`, `SingleValueCollector` | text input | value change |
| `password` | `PasswordCollector`, `ValidatedPasswordCollector` | password input | value change |
| `boolean` | `BooleanCollector`, `ValidatedBooleanCollector` | checkbox | value change, as a boolean |
| `select` | `SingleSelectCollector`, `SingleSelectObjectCollector`, `ObjectSelectCollector`, `DeviceAuthenticationCollector`, `DeviceRegistrationCollector` | select over `output.options` | the chosen option's `value` |
| `multiSelect` | `MultiSelectCollector`, `MultiValueCollector` | checkbox group over `output.options` | an array |
| `phone` | `PhoneNumberCollector`, `PhoneNumberExtensionCollector` | grouped inputs, one per part | the object the SDK's updater expects |
| `submit` | `SubmitCollector`, `ActionCollector` | button | **form submission**, `next()` |
| `flow` | `FlowCollector` | button | **named exit**, `flow({ action })()` |
| `idp` | `IdpCollector` | anchor carrying `output.url` | browser navigation, not a callback |
| `readOnly` | `ReadOnlyCollector`, `NoValueCollector` | static text | nothing |
| `richText` | `RichTextCollector` | text with each `{{key}}` replaced from `richContent.replacements` | nothing |
| `image` | `ImageCollector`, `QrCodeCollector` | image, wrapped in an anchor when it carries an `href` | nothing |
| `deferred` | `ProtectCollector`, `PollingCollector`, `FidoRegistrationCollector`, `FidoAuthenticationCollector` | nothing | reported as deferred, never dropped |
| `unknown` | `UnknownCollector`, and anything unrecognised | text input | value change |

Names that do **not** exist in 2.1.1: `CheckboxCollector`, `ErrorDisplayCollector`. Both are 1.2.0 names. A checkbox is a `BooleanCollector` whose `output` carries an `appearance`, and an error is `error: string | null` on every collector rather than a collector of its own.

Declared in the `StandardField` type union with no implementation: `BUTTON`, `SINGLE_SELECT`. Neither collects anything.

### Reading and writing a collector

```javascript
const collectors = davinciClient.getCollectors()   // for the current node

const updater = davinciClient.update(collector)    // value collectors
updater('a value')

const validator = davinciClient.validate(collector) // validated collectors only
const errors = validator('a value')                 // string[]

await davinciClient.next()                          // submit the form
await davinciClient.flow({ action: collector.output.key })()  // claim a named exit
```

Collectors are plain objects, not class instances. `PasswordCollector` does not echo `input.value` back.

FIDO collectors carry their WebAuthn options at `output.config.publicKeyCredentialCreationOptions` (registration) or `...RequestOptions` (authentication); run them through the SDK's own `fido()` helper and push the result back through the updater.

## Android: collector categories

`CollectorRegistry` registers fifteen categories at 2.1.0 where 1.2.0 registered eleven, with none removed. The four added are `BOOLEAN`, `POLLING`, `QR_CODE` and `READ_ONLY_TEXT`.

| Collector | Key properties | How a value goes back |
| --- | --- | --- |
| `TextCollector` | `label`, `value`, `required` | `collector.value = text` |
| `PasswordCollector` | `label`, `value` | `collector.validate(pwd)` first, then `collector.value = pwd` |
| `LabelCollector` | `content` | display only |
| `SingleSelectCollector` | `label`, `options: List<Option>`, `value: String` | `collector.value = option.value` |
| `MultiSelectCollector` | `label`, `options: List<Option>`, `value: List<String>` | `collector.value = listOf(...)` |
| `PhoneNumberCollector` | `label`, `defaultCountryCode`, `countryCode`, `phoneNumber` | set both parts |
| `SubmitCollector`, `FlowCollector` | `label` | call `onNext` |
| `DeviceRegistrationCollector`, `DeviceAuthenticationCollector` | `devices: List<Device>` | `collector.value = device`, then `onNext` |
| `ProtectCollector` | - | `suspend collector.collect()`, then `onNext` |
| `FidoRegistrationCollector` | - | `suspend collector.register()`, then `onNext` |
| `FidoAuthenticationCollector` | - | `suspend collector.authenticate()`, then `onNext` |
| `IdpCollector` | `label`, `type`, `iconUrl` | `suspend collector.authorize(redirectUri)` |

Compose gotchas that cost a build or a loop:

- `Node` has no `equals()`, so `node ==` never fires and `.onChange(of: node)` has no Android equivalent worth relying on. Drive recomposition off an `Int` counter.
- A `SuccessNode` navigation must be wrapped in `LaunchedEffect(Unit)` or it recomposes in a loop.
- An auto-advancing collector still draws the Next button unless you suppress it. Wrap in `LaunchedEffect(collector)` and set the flag.
- `com.pingidentity.davinci.module.Oidc` and `com.pingidentity.journey.module.Oidc` are different classes with the same simple name. Journey is not this platform; if a Journey import appears in a DaVinci file it is a mistake, not a variant.
- SDK 2.0.0 and later is on Maven Central. A custom `maven { }` block for Ping artefacts causes a 401.

## iOS: the property names that are wrong if guessed

Taken from Ping's own agent skills. The compatibility matrix confirms these collector classes exist on iOS at the versions it gives; the property and method names below are not covered by it and are unconfirmed here. Treat them as a starting point to verify, not as established fact.

| Symptom | Cause |
| --- | --- |
| `actionKey` and `eventType` absent from the POST body | `SubmitCollector.value` / `FlowCollector.value` not set to `collector.id` before `onNext()` |
| `ForEach requires Option to conform to Hashable` | `Option` is not `Hashable`. Use `collector.options.indices, id: \.self` |
| `PhoneNumberCollector has no member 'value'` | use `.phoneNumber`, plus `.countryCode` |
| `ProtectCollector has no member 'start'` | use `await collector.collect()` |
| `extra argument 'deviceName'` on `FidoRegistrationCollector` | the DaVinci variant takes only `window:`, unlike Journey's |
| `DeviceRegistrationCollector has no member 'register'` | set `collector.value = device`, then `onNext()` |
| `Device has no member 'name'` | use `.title`. Also `.type`, `.id`, `.description`, `.iconSrc`, `.isDefault`, and not `Hashable` |
| `cannot find type 'Collector' in scope` | missing `import PingDavinciPlugin` |
| `cannot find type 'ContinueNode' / 'SuccessNode'` | missing `import PingOrchestrate` |
| tokens overwritten between client instances | each needs a unique `account` string on `KeychainStorage<Token>` |

`Option` is `.label` for display and `.value` for submission.
