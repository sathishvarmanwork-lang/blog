# Why I built one product as two native codebases, and how I keep Kotlin and Swift mirrored under a three-layer clean architecture

I made a deliberately expensive choice on a mobile build: two native codebases, Kotlin on Android and Swift on iOS, no cross-platform framework. One product, built twice, by one person. The bet was native performance, real platform idioms, and no framework lock-in. The bill is that every feature is now two implementations that have to stay behaviourally identical, forever.

Two codebases drift. Field names diverge. A business rule gets fixed on one side and forgotten on the other. The layering rots differently on each platform until they're two different apps wearing the same name. Unless something actively holds them in a mirror, the duplication isn't a tax you pay once, it's a liability that compounds.

This piece is about the two mechanisms that make the duplication safe: a module boundary the compiler enforces, and a mirror discipline strict enough to audit. It's also honest about what isn't built yet, because the skeleton I'm describing is thin on purpose and the enforcement story is half-finished.

## Why prove the architecture before any feature

The first thing I built was not a feature. It was a walking skeleton: log in, enter an amount, see "Completed." Three screens, no real logic, on both platforms.

That sounds like wasted motion. It isn't, and the reason is the whole philosophy of the build. If you grow features first and impose structure later, you impose structure at exactly the moment it's most expensive: when there's already code in the wrong place to migrate, and a habit of putting it there. The skeleton exists so the thing being proven is the architecture itself, not any feature. If the skeleton builds identically on both platforms and both compile green, the structure is sound enough to grow real features into. If it doesn't, I find out when the fix is four files, not forty.

So the standard goes in on day one. The real Presentation, Domain, and Data modules, the full naming convention, the strict mirror, all of it, while the codebase is small enough that getting it wrong costs nothing.

## The load-bearing idea: a layer boundary you can't accidentally cross

Here is the principle the entire architecture rests on. A three-layer diagram in a wiki is advisory. A Domain module that physically cannot import the UI toolkit is mechanical. The first relies on every developer remembering the rule at every code review. The second relies on the compiler, which never forgets and never gets tired.

Clean architecture says dependencies point inward: Presentation depends on Domain, Data depends on Domain, and Domain depends on nothing. The usual way to express that is folders. A `domain` package, a `data` package, a convention that the domain package doesn't reach outward. The problem with folders is that a `domain` package can still import the UI toolkit and nothing stops it. The violation sits there until someone notices in review, and reviewers notice unevenly.

So I made each layer its own build module. On Android that's a Gradle module; on iOS it's a Swift Package Manager target. The Domain module declares zero dependencies on the UI toolkit, on HTTP, on any database. Not "shouldn't import." Cannot. The import doesn't resolve. The build fails.

On Android the Domain module's entire build file is this:

```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
}

kotlin {
    jvmToolchain(11)
}
```

That's a plain Kotlin-JVM library. No Android plugin. Because there's no Android plugin, a file in this module cannot write `import androidx.compose.material3.Text`, because the symbol isn't on the classpath. A business rule physically cannot reach for a button. The Data layer is the same plain-Kotlin shape, with exactly one dependency, pointing inward to Domain and nowhere else:

```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
}

kotlin {
    jvmToolchain(11)
}

dependencies {
    implementation(project(":domain"))
}
```

The iOS side declares the same graph as a Swift package, and the package definition reads like the architecture diagram made executable:

```swift
let package = Package(
    name: "Modules",
    platforms: [.iOS(.v16)],
    products: [
        .library(name: "Domain", targets: ["Domain"]),
        .library(name: "Data", targets: ["Data"]),
        .library(name: "DesignSystem", targets: ["DesignSystem"]),
        .library(name: "Presentation", targets: ["Presentation"]),
    ],
    targets: [
        .target(name: "Domain"),
        .target(name: "Data", dependencies: ["Domain"]),
        .target(name: "DesignSystem"),
        .target(
            name: "Presentation",
            dependencies: ["Domain", "DesignSystem"]
        ),
    ]
)
```

Same arrows as Android. Data depends on Domain. Presentation depends on Domain and the design system. Domain depends on nothing. The dependency direction isn't documented, it's compiled.

There's a cost to this, and it's worth being honest about it. In the skeleton, Domain and Data contain no logic at all, yet they exist as full build modules carrying their own build files for nothing yet. Four near-empty modules across two platforms, all ceremony, zero features. I accept that cost deliberately, because the boundary has to exist before the first business rule lands. If the boundary shows up after the first rule, the first rule is already in the wrong place. The ceremony is paid once, at zero-feature time, when it's cheapest.

## The strict mirror, and the difference between same-name and same-character

The second mechanism is the mirror. The two codebases are meant to be readable as one: same module names, same feature folders, same file names, same class and method and field names, same step order. A reader holds both in a single mental model and the only thing that changes between them is language syntax.

The discipline that makes this survivable is choosing the right level to enforce it at. I audit the mirror at the name level, not the character level. Some things are required to match: class names, method names, field names, the order of steps in a flow. Other things are allowed to differ because the language forces them to: Kotlin's `Flow` against Swift's `AsyncSequence`, a `sealed class` against an `enum` with associated values, `suspend fun` against `async func`, `mutableStateOf` against `@Published`. A character-level diff of the two codebases would drown in that syntax noise and tell you nothing. A name-level diff, extract the type and method and field names from both sides and compare those, catches real drift while ignoring the differences that don't matter.

The shared ViewModel is the cleanest example. Android:

```kotlin
class TransferViewModel : ViewModel() {
    var amount by mutableStateOf("")

    fun reset() {
        amount = ""
    }
}
```

iOS, with a doc comment that names the mirror explicitly so the next reader knows the two are intentionally paired:

```swift
public final class TransferViewModel: ObservableObject {
    @Published public var amount: String = ""

    public init() {}

    public func reset() {
        amount = ""
    }
}
```

Name-level, these are identical: type `TransferViewModel`, field `amount`, method `reset()`. The allowed wrappers are everything else: `ViewModel()` against `ObservableObject`, `mutableStateOf` against `@Published`, plus the `public init()` that Swift's module boundary demands and Kotlin doesn't. That's the mirror rule working exactly as intended.

The navigation entries are the sharpest version of the same idea, because the mechanisms look completely different and the contract is still identical. Android scopes the ViewModel to a navigation graph so the framework owns its lifecycle and auto-clears it when you leave the flow. iOS holds it as a `@StateObject` on the flow view and resets it by hand. Different plumbing, same behaviour: one shared transfer state across all three screens, cleared when you start a new transfer. The contract mirrors; the platform idioms underneath it are each platform's own.

## Where the mirror genuinely breaks: empty layers and a thin design system

Two places the mirror doesn't hold character-for-character, and both are instructive.

The first is the empty lower layers, and it's a clean case of "behaviourally mirrored, not character-mirrored." Android's Domain and Data modules are genuinely empty, zero source files, just a build file. Swift can't do that: an SPM target needs at least one source file to compile. So each empty Swift layer holds a one-line marker whose comment documents the parity intent:

```swift
// Domain layer: entities, use cases, repository protocols (mirrors Android :domain).
// Empty for the walking skeleton; grows as business rules land.
public enum DomainModule {}
```

Both platforms have an empty Domain layer. Gradle expresses empty as zero files; SPM expresses it as a one-line marker. A character diff flags this as a difference. A name-and-behaviour audit correctly passes it, because the two layers are the same thing. This is exactly why the audit lives at the name level and not the character level.

The second divergence is a real one, not a language artifact, and I'm calling it out because it's the honest weak point. The Android design system inherited a full Material-3 theme from the Studio scaffold: a palette, typography, and dynamic color theming that has no iOS equivalent. The iOS design system is a single token:

```swift
public enum DS {
    public static let screenPadding: CGFloat = 24
}
```

That asymmetry didn't come from a decision. It crept in through the scaffold. Android got a rich theme for free and iOS got one padding value, and the two sides are not at parity. Worse, the screens hard-code the padding value directly instead of reading even the one token that exists, so the single shared value isn't actually sourced from one place. This is the first thing that should tighten when design tokens start carrying real weight, and right now it's a drift the mirror discipline is supposed to catch but hasn't yet.

## The honest state: one enforcement net of three is live

Here's where I have to be straight about what's built versus what's specced, because the gap is the most important thing in this whole writeup.

The intended design has three enforcement nets. One: the module boundary, which makes the worst violation (UI imported into Domain) a compile error. Two: architecture tests that assert the intra-layer rules, the ones the module boundary can't see, like "no validation logic in Presentation" or "DTOs never cross into Domain." Three: forbidden-import linters as a second mechanical check.

Only the first net is actually live in the code. The architecture tests and the linters are specced in my standards doc and absent from the build. Which means right now, "Domain imports nothing from the UI" is guaranteed by the module boundary, which is real and mechanical. But the finer rules, and the strict mirror itself, are currently enforced by me reading both trees and diffing the names by eye. The automation that would make the mirror audit a CI check that fails on drift is designed and not written.

So the measurable outcome of this skeleton is narrow and I won't inflate it. It is "both platforms build green and run identically." Android builds and runs on a real device through Firebase App Distribution; iOS builds first try and runs on the simulator. The structure is proven to compile and run the same on both sides. The enforcement of that structure past the module boundary is debt, not a finished state, and the lower layers are empty so there's no business logic to measure yet.

There's a sharper admission inside this one. The central claim of the entire architecture is "a UI import into Domain won't compile." I asserted that. I never actually tried it. I never added `import androidx.compose.material3.Text` to a Domain file to watch it fail. The guarantee is structurally sound and I believe it holds, but I haven't witnessed it, and that's the difference between "should hold" and "proven." It's the same discipline I apply on the classifier work, where I make the model fail on purpose to prove the guardrail catches it. I did that there. I skipped it here, and I shouldn't have.

## What I'd do differently

Wire the architecture-test and linter nets with the skeleton, not "later." When the codebase is four files, adding them is trivial. After forty feature files, adding them means retro-fitting and discovering the violations that already crept in. The expensive moment to install enforcement is exactly the moment most people defer it to.

Make the design-system divergence a logged exception, not an accident. The Android-ahead asymmetry should either be brought to parity now or explicitly recorded as "design tokens are Android-ahead until milestone N," so the one current hole in the mirror is a tracked decision rather than a silent drift that I happen to know about.

Don't hard-code copy and spacing in the screens. The strings and the padding values are duplicated literals in two languages, and duplicated literals in two languages are precisely what diverge first. The mirror is only as strong as its least-shared value, and right now its least-shared values are a few string literals and a number.

Prove the boundary with one throwaway violation. Thirty seconds: add a UI import to a Domain file, watch it fail to resolve, delete it. That turns the load-bearing claim of the architecture from an assertion into something witnessed. I prove guardrails by tripping them everywhere else in this project. The architecture deserved the same test and didn't get it.

## The shape of it

Two native codebases, three layers each, every layer its own build module so the dependency direction is compiled rather than documented. A strict mirror audited at the name level, where class and method and field names and step order must match and language syntax is free to differ. One enforcement net of three actually live, the empty lower layers honest about the SPM-versus-Gradle divergence, and a design system that's the known weak point.

The bet was that native duplication is safe if something mechanical holds the two sides in a mirror. The module boundary delivers on that for the most dangerous rule. The rest of the enforcement is designed and waiting, and until it's wired, the mirror holds because I'm watching it, which is exactly the kind of guarantee this whole architecture was supposed to replace.
