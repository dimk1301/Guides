# Upgrading Spring Framework, Spring Boot, and React with AI Coding Agents
### A developer's playbook for Spring Framework 6→7, Spring Boot 3→4, and React 18→19 — using Claude Code, Kiro CLI, and GitHub Copilot
> 🔑 **The one rule that matters more than any tool in this guide:** let a deterministic migration tool (OpenRewrite, the official React codemod) do the mechanical rewriting first. Only let the AI agent touch what the tool couldn't fix automatically, and only after you've told it exactly which known breaking changes to look for. Never ask an agent to "just upgrade X" from a blank prompt.

**Legend:** 🔒 hard rule · 💡 tip · 🧩 example prompt · 🛠️ command · ⚠️ silent-failure risk (compiles/builds, breaks at runtime) · 🔗 primary source

***

## Table of contents

- [Part 0 — Golden Constraints](#part-0--golden-constraints-put-this-in-place-before-you-start)
- [Part 1 — Spring Framework 6 → 7](#part-1--spring-framework-6--7)
  - [Fast path (10 minutes)](#fast-path-10-minutes)
- [Part 2 — Spring Boot 3 → 4](#part-2--spring-boot-3--4)
  - [Fast path (10 minutes)](#fast-path-10-minutes-1)
- [Part 3 — React 18 → 19](#part-3--react-18--19)
  - [Fast path (10 minutes)](#fast-path-10-minutes-2)
- [Part 4 — Side-by-side cheat sheet](#part-4--side-by-side-cheat-sheet)
- [Part 5 — Rollout strategy](#part-5--rollout-strategy-all-three)
- [Part 6 — Troubleshooting](#part-6--troubleshooting)
- [Final Checklist](#final-checklist)

***

## Part 0 — Golden Constraints (put this in place before you start)

Add this to your agent's persistent-instructions file — `CLAUDE.md` for Claude Code, `.kiro/steering/` for Kiro CLI, `.github/copilot-instructions.md` for GitHub Copilot:

```
Major Upgrade Golden Constraints:

0. HUMAN APPROVAL REQUIRED for every file change. Show the diff, wait for
   explicit approval, before writing to disk.

1. TOOL FIRST: Always run the official migration tool (OpenRewrite for
   Spring Framework/Boot, the official codemod for React) BEFORE making any
   manual edit. Only touch files the tool did not fully fix.

2. SILENT FAILURES OUTRANK COMPILE ERRORS: Actively search for patterns that
   compile/build successfully but change behavior at runtime (see the
   ⚠️ callouts in each section below) before declaring the migration done.
   A clean build is not proof the migration is correct.

3. ONE CONCERN PER COMMIT: Group changes by root cause (e.g. "all javax.*
   to jakarta.* annotation replacements") not by file. Never bundle
   unrelated changes into one diff.

4. NO SILENT BEHAVIOR CHANGES: If a fix changes runtime behavior (not just
   syntax), flag it explicitly and explain the difference — don't just
   make it compile.

5. TEST BEFORE AND AFTER: Confirm the test suite passes before starting.
   If it doesn't, stop and say so — do not "fix" pre-existing failures as
   part of the migration diff.

6. NO VERSION GUESSING: Confirm the exact target version and its release
   notes before generating any change tied to that version's behavior.

7. SCOPE LIMIT: Stay inside the module/package explicitly given for this
   session. Don't wander into unrelated services.

8. CHECK FOR OVERLAP: Before manually fixing anything, confirm the
   deterministic tool didn't already cover it (see the note on recipe
   overlap in Part 1, Step 7). Re-doing work the tool already did is
   wasted review time and a source of unnecessary diff noise.
```

💡 Why constraint #2 exists: unlike a CVE fix, several of the changes below (Framework's `javax.annotation` removal, React's ref-cleanup change) do not throw an error — they just stop doing what they used to do. That's the specific failure mode this guide is built to catch.

***

## Part 1 — Spring Framework 6 → 7

### Fast path (10 minutes)

If you just want the fastest safe first pass: run the OpenRewrite command in Step 3, then run the single grep in Step 4. Those two actions catch the mechanical rewrites and the #1 silent failure. Everything else in this section is what to do with what's left over.

### 1.1 Mental model: Framework 7 is not Boot 4

Spring Boot 4 gets the attention, but it sits on top of Spring Framework 7 — and most "Boot 4 broke my code" surprises actually trace back to a Framework 7 decision: the Jakarta cutover, bean lifecycle changes, AOT metadata format, and null-safety migration. Rule of thumb while triaging: a class in `org.springframework.*` is a Framework 7 concern (this section); a class in `org.springframework.boot.*` is a Boot 4 concern (Part 2).

### 1.2 Baseline requirements — check these before touching code

| Component | Framework 6.2 | Framework 7.0 |
|---|---|---|
| JDK | 17 baseline, 21 recommended | 17 baseline, 25 recommended |
| Jakarta EE | 9/10 | 11 |
| Servlet | 6.0 (Tomcat 10.1, Jetty 12) | 6.1 (Tomcat 11, Jetty 12.1) |
| JPA | 3.1 | 3.2 (Hibernate ORM 7.1/7.2) |
| Bean Validation | 3.0 | 3.1 (Hibernate Validator 9.x) |
| JUnit | 5 (Jupiter) | 6 |
| Jackson | 2.x | 3.x default, 2.x deprecated |

🔗 [Spring Framework 7.0 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes) · [Spring Boot System Requirements](https://docs.spring.io/spring-boot/system-requirements.html)

💡 **A note on the JDK row, since it's a common point of confusion online:** you'll find blog posts claiming Spring Boot 4 "requires" Java 21. The official Spring Boot system-requirements page and Framework 7 release notes both confirm the *hard floor* is still Java 17 — Java 21/25 is strongly recommended (mainly for virtual threads), not required. Don't let a CI pipeline upgrade turn into an unplanned JDK migration; it's optional here.

🔒 **Undertow users: stop and read this first.** Undertow does not yet support Servlet 6.1, so Framework 7 dropped its Undertow-specific WebSocket/WebFlux classes. There is no flag to work around this — you're blocked until Undertow ships Servlet 6.1 support, or you move to Tomcat 11+/Jetty 12.1+.

### 1.3 Step-by-step

**Step 1 — Get to Framework 6.2 first, cleanly.** If you're on an older 6.x, upgrade to 6.2 and clear its deprecation warnings before jumping to 7. Jumping straight from 6.0/6.1 to 7.0 stacks two migrations into one diff.

**Step 2 — Baseline test run:**
```bash
mvn clean test
```
🔒 If this fails already, stop and fix it first — a migration diff on top of a red baseline is undebuggable.

> 💡 **New to OpenRewrite? Read this first.** OpenRewrite is a deterministic code-rewrite tool for Java/JVM projects — it parses your code into an actual syntax tree and applies precise, rule-based transformations, the same way every time. It is **not** an AI model; it doesn't guess or infer intent, it only changes code that matches a rule it has. A **recipe** is just a named, versioned bundle of those rules (e.g. `UpgradeSpringFramework_7_0` bundles dozens of individual rewrite rules for this specific migration). You add it as a Maven/Gradle plugin, run one command, and it edits your working directory directly — nothing is applied until you say so, and `git diff` afterward shows you exactly what changed, file by file, so you can review it like any other diff before committing. This determinism is exactly why Golden Constraint #1 says "tool first, agent second": the recipe either has a rule for something or it doesn't — there's no partial credit or creative reinterpretation to double-check the way there is with an AI agent's output.

**Step 3 — Run the OpenRewrite recipe.** Check the current Maven Central version before pinning — don't hardcode a version from any guide, including this one:
```bash
curl -s "https://search.maven.org/solrsearch/select?q=g:org.openrewrite.maven+AND+a:rewrite-maven-plugin&rows=1&wt=json" | grep -o '"v":"[^"]*"'
curl -s "https://search.maven.org/solrsearch/select?q=g:org.openrewrite.recipe+AND+a:rewrite-spring&rows=1&wt=json" | grep -o '"v":"[^"]*"'
```
Add to `pom.xml` (temporary, removable after migration) using the versions you just retrieved:
```xml
<plugin>
  <groupId>org.openrewrite.maven</groupId>
  <artifactId>rewrite-maven-plugin</artifactId>
  <version><!-- version from curl command above --></version>
  <configuration>
    <activeRecipes>
      <recipe>org.openrewrite.java.spring.framework.UpgradeSpringFramework_7_0</recipe>
    </activeRecipes>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>org.openrewrite.recipe</groupId>
      <artifactId>rewrite-spring</artifactId>
      <version><!-- version from curl command above --></version>
    </dependency>
  </dependencies>
</plugin>
```
```bash
mvn rewrite:run
git diff
```
This recipe handles the mechanical parts — `ResponseStatusException` API changes, `WebMvcConfigurerAdapter` removal, deprecated `UriComponentsBuilder` methods, and similar find-and-replace-safe changes.

🔗 [`UpgradeSpringFramework_7_0` recipe reference](https://docs.openrewrite.org/recipes/java/spring/framework/upgradespringframework_7_0)

⚠️ **Recipe overlap alert — read before Step 7:** this recipe's own chain already includes `JUnit5to6Migration`, `UpgradeJackson_2_3`, and `MigrateFromSpringFrameworkAnnotations` (the `org.springframework.lang.Nullable` → JSpecify move). That means the JUnit deprecation, the Jackson 2→3 shift, and Step 7's null-safety migration below may **already be done** by the time you reach them. Diff first, act second — don't manually redo what the tool already touched.

**Step 4 — Hunt the silent failure before trusting any test result.** This is the step most guides skip, and it's the most dangerous change in the release:

> ⚠️ **`javax.annotation` and `javax.inject` are silently ignored, not removed as a compile error.** Code like `@javax.annotation.PostConstruct` still compiles (the annotation class is on the classpath transitively) — but Spring 7 no longer recognizes it, so the method just never runs. No exception, no log line.
>
> 🔗 [Confirmed in the official release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes): "Annotations in the `javax.annotation` and `javax.inject` packages are no longer supported."

🛠️ Run this grep before anything else, and don't skip it even if your build is green:
```bash
grep -rn "javax.annotation\|javax.inject" --include="*.java" src/
```

> 🧩 Prompt for your agent (Claude Code / Kiro CLI / Copilot):
> *"Search this codebase for every use of javax.annotation.* and javax.inject.* (PostConstruct, PreDestroy, Resource, Inject). For each hit, show the file, the line, and the exact jakarta.* replacement. Do not apply any change — just list findings grouped by file, ready for review."*

**Step 5 — Fix the hard compile errors** (the easy part — the compiler tells you where):

| Removed | Replace with |
|---|---|
| `ListenableFuture` | `CompletableFuture` |
| `suffixPatternMatch`, `trailingSlashMatch`, `favorPathExtension` | `PathPattern`-based matching (default) |
| `HttpHeaders` cast to `MultiValueMap` | Header-specific methods: `.add()`, `.set()`, `.get()` |
| `spring-jcl` module | Apache Commons Logging (usually transparent) |
| Undertow WebSocket/WebFlux classes | Tomcat 11+ or Jetty 12.1+ |

⚠️ **`HttpHeaders` no longer implements `MultiValueMap`** — any utility code, filters, or tests that cast headers to a map will fail to compile. Search specifically for that cast.

**Step 6 — Review bean lifecycle and proxy changes** that don't throw errors but change behavior:

- ⚠️ **Proxy defaulting is now consistently CGLIB** across all proxy processors (including `@Async`), where 6.x behavior was inconsistent. If code relied on a JDK dynamic proxy for an `@Async` bean, that assumption may silently no longer hold. Use `@Proxyable(INTERFACES)` to opt a specific bean back to interface proxying.
- New `BeanRegistrar` contract replaces verbose `BeanDefinitionRegistryPostProcessor` code for programmatic bean registration, and is AOT-analyzable (important if you build native images).

**Step 7 — Confirm the null-safety migration, don't assume it's manual:**
```
// Old (deprecated):
import org.springframework.lang.Nullable;
// New:
import org.jspecify.annotations.Nullable;
```
As noted in Step 3, the `UpgradeSpringFramework_7_0` recipe already runs `MigrateFromSpringFrameworkAnnotations` for you. Diff first — this step may already be checked off. What the recipe *won't* catch is Kotlin nullability drift:

⚠️ **Kotlin teams: budget extra time here regardless.** JSpecify's more precise nullness on Spring's own APIs can turn previously-compiling Kotlin into a compile error (or vice versa), and that's a manual review the tool can't do for you.

**Step 8 — If you build native images**, update `RuntimeHints`:
```
// Spring 6 (regex-based, matched nested folders too):
hints.resources().registerPattern("/files/*.ext");
// Spring 7 (glob-based, single segment only — be explicit for nested paths):
hints.resources().registerPattern("/files/**/*.ext");
```
Also simplify reflection hints — a type hint alone now implies methods/constructors/fields.

**Step 9 — Re-test the security layer as its own pass, not as a side effect of "the build passed":**

- ⚠️ **CORS pre-flight requests are no longer rejected when the CORS config is empty** — if you relied on that rejection as an implicit guard, it's gone. 🔗 [Tracked in spring-framework#31839](https://github.com/spring-projects/spring-framework/issues/31839), shipped in 7.0.
- ⚠️ **Ant-style path patterns in security rules may match different URLs** now that `PathMatcher`/`AntPathMatcher` is deprecated in favor of `PathPattern` — a mismatch here means a security rule can silently stop protecting an endpoint. 🔗 Spring Security is tracking its own migration off `AntPathRequestMatcher` in [spring-boot#45150](https://github.com/spring-projects/spring-boot/issues/45150) — worth checking if your Boot version has already picked up the fix.

**Step 10 — Validate:**
```bash
mvn dependency:tree
mvn clean test
```

### 1.4 Use your agent to scan for everything above, in one pass

> 🧩 Full audit prompt (works in Claude Code, Kiro CLI, or Copilot Agent Mode):
> *"Audit this Java codebase for a Spring Framework 6.2 to 7.0 migration. Report file:line for each of the following, ordered by severity: (1) SILENT FAILURES — javax.annotation.* or javax.inject.* usage; (2) HARD COMPILE ERRORS — ListenableFuture, HttpHeaders cast to MultiValueMap, removed path-mapping options, Undertow-specific config; (3) DEPRECATIONS WITH DEADLINES — Jackson 2.x (com.fasterxml.jackson) imports, RestTemplate usage, AntPathMatcher in MVC or security config, JUnit 4 + SpringRunner; (4) NULL SAFETY — org.springframework.lang.Nullable usage that the OpenRewrite recipe did not already convert; (5) SECURITY RISK — Ant-style security path patterns, reliance on empty-CORS rejection. For each, give the exact replacement code. Do not apply anything — output a report only."*

***

## Part 2 — Spring Boot 3 → 4

Boot 4 depends on Framework 7 — do Part 1 first, or expect Boot-layer errors to actually be Framework-layer errors in disguise. This section covers what's specific to Boot itself: autoconfiguration, starters, and properties.

### Fast path (10 minutes)

Run the Boot-specific OpenRewrite recipe in Step 2, diff it, then hand your agent only the files still failing to build (Step 3's prompt). Don't hand-audit the whole module first — let the recipe narrow the surface area for you.

### 2.1 Step-by-step

**Step 1 — Baseline test run** (same as Part 1, Step 2, if not already done):
```bash
mvn clean test
```

**Step 2 — Run the Boot-specific OpenRewrite recipe.** (New to OpenRewrite? See the primer in Part 1, Step 3 — same tool, different recipe.) Re-check current versions the same way as Part 1, Step 3 — Boot recipes ship on their own release cadence:
```xml
<plugin>
  <groupId>org.openrewrite.maven</groupId>
  <artifactId>rewrite-maven-plugin</artifactId>
  <version><!-- current version, verified via Maven Central --></version>
  <configuration>
    <activeRecipes>
      <recipe>org.openrewrite.java.spring.boot4.UpgradeSpringBoot_4_0</recipe>
    </activeRecipes>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>org.openrewrite.recipe</groupId>
      <artifactId>rewrite-spring</artifactId>
      <version><!-- current version, verified via Maven Central --></version>
    </dependency>
  </dependencies>
</plugin>
```
```bash
mvn rewrite:run
git diff
```
This rewrites `pom.xml` (parent/BOM version, starter coordinates) and known Boot-level source changes in one pass.

🔗 [`UpgradeSpringBoot_4_0` recipe reference](https://docs.openrewrite.org/recipes/java/spring/boot4/upgradespringboot_4_0-community-edition)

💡 Worth knowing what this recipe chains together on your behalf: it already pulls in `UpgradeSpringFramework_7_0`, `UpgradeSpringSecurity_7_0`, a Spring Batch 5→6 migration, `MigrateToHibernate71`, a Testcontainers migration, and a modular-starters migration. If a manual step below looks like it should already be fixed, check the diff before assuming it needs a hand edit.

**Step 3 — Bring in your agent for what's left.** OpenRewrite recipes don't cover custom integrations — a hand-rolled `WebSecurityConfigurerAdapter` subclass, a custom `HttpSecurity` chain, deprecated bean definitions, or third-party starters without a Boot 4 release yet.

> 🧩 Prompt: *"I ran the OpenRewrite Spring Boot 4 migration recipe. These files still fail to build: [list]. For each, explain what changed between Spring Boot 3 and 4 that caused it (check if the root cause is actually a Spring Framework 7 change, not Boot itself), and propose a fix. Show me the diff for each file separately — do not apply anything yet."*

**Step 4 — Check for removed/renamed autoconfiguration and properties**, common in this jump:
- Actuator endpoint and property renames.
- Removed auto-configuration classes for third-party libraries that haven't shipped a Boot-4-compatible release.
- Starter artifact ID or grouping changes (Boot 4 introduced modular starters — a library you used to get transitively may now need its own starter declared explicitly).

**Step 5 — Review, approve, validate per file** — not one giant approval for the whole batch.

**Step 6 — Validate:**
```bash
mvn dependency:tree
mvn clean test
```

***

## Part 3 — React 18 → 19

### Fast path (10 minutes)

Run the two codemod commands in Steps 3 and 4 back to back, then grep for the legacy patterns in Step 5. That combination catches almost everything mechanical; what's left is genuinely a case-by-case judgment call for your agent.

### 3.1 Before touching React itself: check dependency readiness

The most common cause of "upgraded React, broke everything else" is a third-party library (state management, UI kit, testing utilities) that doesn't yet support React 19. Confirm compatibility for your key dependencies before running anything.

### 3.2 Step-by-step

**Step 1 — Baseline:**
```bash
npm run test
```
🔒 Don't proceed on a red baseline.

**Step 2 — Confirm what's actually published**, rather than trusting a fixed version number from any guide:
```bash
npm view react versions --json | tail -20
npm view react-dom versions --json | tail -20
```

> 💡 **New to codemods? Read this first.** A codemod is the JavaScript-ecosystem equivalent of an OpenRewrite recipe: a script that parses your code (via its AST, not plain text) and applies mechanical, rule-based rewrites — not an AI guess. `npx codemod@latest react/19/migration-recipe` doesn't install anything permanent; `npx` downloads and runs the tool once, applies its rewrites directly to your files, and exits. As always, nothing is "safe" until you've looked at `git diff` yourself — the codemod is reliable for the specific patterns it knows about (like `ReactDOM.render`), but silent, behavior-only changes (like the ref-cleanup and `act()` changes in Step 5 below) are exactly the kind of thing no codemod can catch, because there's no syntax pattern to rewrite — the code still compiles, it just runs differently.

**Step 3 — Run the official codemod:**
```bash
npx codemod@latest react/19/migration-recipe
```
This mechanically rewrites the known breaking changes across your codebase — removed `ReactDOM.render`, string refs replaced with callback refs, the `act` import moved from `react-dom/test-utils` to `react`, `useFormState` renamed to `useActionState`, and other syntax-level shifts.

🔗 [Official React 19 Upgrade Guide](https://react.dev/blog/2024/04/25/react-19-upgrade-guide) · [Codemod registry entry](https://app.codemod.com/registry/react/19/migration-recipe)

**Step 4 — Run the TypeScript types codemod, if applicable:**
```bash
npx types-react-codemod@latest preset-19 ./src
```
This fixes `@types/react` typing breaks the main codemod doesn't touch — including the stricter ref-callback-return rule, where returning anything other than a cleanup function from a ref callback is now rejected by TypeScript.

**Step 5 — Hunt the changes that build clean but shift behavior:**

| Change | Risk |
|---|---|
| `ReactDOM.render` removed | Hard error if missed — codemod should catch this |
| Ref cleanup functions now supported and called on unmount | ⚠️ Code that assumed refs never needed cleanup may leak or behave differently |
| Legacy Context API removed (`contextTypes`, `getChildContext`) | Hard error if still in use |
| `act()` behavior changed in tests, now imported from `react` | ⚠️ Tests may pass but no longer assert what you think they assert |
| `useFormState` renamed to `useActionState` | Codemod-covered, but check call sites in older tutorials/copy-pasted code |

🛠️ Grep for legacy patterns before trusting the codemod caught everything:
```bash
grep -rn "ReactDOM.render\|contextTypes\|getChildContext" --include="*.jsx" --include="*.tsx" src/
```

**Step 6 — Bring in your agent for what's left:**

> 🧩 Prompt: *"I ran the React 19 codemod and the types-react-codemod. These files still fail to build or fail tests: [list]. For each, explain what React 19 behavior change is responsible, then propose the smallest fix that preserves current behavior. Flag any fix where the runtime behavior differs from React 18, even if the code now compiles. Diff per file, no auto-apply."*

**Step 7 — Review each diff individually, approve one at a time.**

**Step 8 — Validate:**
```bash
npm run build
npm run test
```

***

## Part 4 — Side-by-side cheat sheet

| | Spring Framework 6→7 | Spring Boot 3→4 | React 18→19 |
|---|---|---|---|
| Deterministic tool | OpenRewrite `UpgradeSpringFramework_7_0` | OpenRewrite `UpgradeSpringBoot_4_0` | `npx codemod react/19/migration-recipe` |
| #1 silent-failure risk | `javax.annotation`/`javax.inject` silently ignored | Inherited from Framework layer + removed autoconfig | Ref cleanup / `act()` behavior changes |
| What the tool won't fix | Custom security chains, proxy-type assumptions, native-image hints | Custom security config, unsupported 3rd-party starters | Behavior-dependent code, non-standard render setups |
| Validate with | `mvn clean test`, security regression pass | `mvn dependency:tree`, `mvn clean test` | `npm run build`, `npm run test` |
| Rollback | Per-file `git checkout --`, not whole-repo revert | Same | Same |

***

## Part 5 — Rollout strategy (all three)

- **One migration layer per commit.** Do Framework 7 fixes and Boot 4 fixes as separate commits even though they land together — a Framework-layer regression should be traceable without wading through Boot changes too.
- **Never skip the baseline test step.** A migration diff on top of already-broken tests is undebuggable.
- **Treat silent-failure hunts (Step 4 in Parts 1 and 3) as mandatory, not optional** — a green build is not proof of correctness for this class of migration.
- **Run the security regression pass separately**, with its own test cases, after everything else compiles and builds.
- **Feature-flag risky React changes** if deploying gradually, rather than shipping the full jump to 100% of users at once.

***

## Part 6 — Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| `@PostConstruct` method silently doesn't run after Framework 7 upgrade | Check for `javax.annotation.PostConstruct` instead of `jakarta.annotation.PostConstruct` — this is the #1 silent failure |
| Agent tries to rewrite files OpenRewrite already covers | Re-state Golden Constraint #1 — remind it to check `git diff` from the tool run first |
| Build fails only on native-image/GraalVM step | Check `RuntimeHints` for old regex-style resource patterns — Framework 7 uses glob patterns |
| Security rule stops matching a URL after upgrade | Ant-style pattern vs `PathPattern` mismatch — verify with an explicit test request against that endpoint |
| CORS suddenly allows requests it used to reject | Expected in Framework 7 — empty CORS config no longer rejects pre-flight; add explicit config if you need the old behavior |
| React codemod skips a file | Usually a non-standard JSX transform config — hand that file to the agent directly |
| Kotlin build breaks only after JSpecify migration | Expected — nullness is now stricter/more precise; review `!!` and nullable usages the compiler flags |
| Maven can't resolve `LATEST` as a version | `LATEST` is a deprecated Maven metaversion — always pin an actual version number, verified against Maven Central, not the literal string `LATEST` |

***

## Final Checklist

**Spring Framework 6→7**
- [ ] On Framework 6.2 (not older 6.x) before starting
- [ ] Servlet container confirmed Servlet-6.1-compatible (not Undertow, unless already patched)
- [ ] OpenRewrite `UpgradeSpringFramework_7_0` run, diff reviewed
- [ ] Grep for `javax.annotation`/`javax.inject` completed — zero hits confirmed
- [ ] `HttpHeaders`-as-`MultiValueMap` casts checked
- [ ] Proxy-type assumptions reviewed (`@Proxyable` added where needed)
- [ ] Null-safety migration to JSpecify confirmed (recipe-applied or manually done), Kotlin nullability re-checked
- [ ] Security layer re-tested separately (path matching, CORS)

**Spring Boot 3→4**
- [ ] Framework 6→7 steps completed first
- [ ] OpenRewrite `UpgradeSpringBoot_4_0` run, diff reviewed
- [ ] Third-party starters confirmed Boot-4-compatible
- [ ] Actuator/property renames checked

**React 18→19**
- [ ] Third-party dependency readiness confirmed before starting
- [ ] Official codemod + types codemod run, diff reviewed
- [ ] Ref cleanup and `act()` behavior changes checked, not just build success
- [ ] Full test suite + build passes after all changes

**All three**
- [ ] Golden Constraints file in place for your chosen agent (Claude Code / Kiro CLI / Copilot)
- [ ] Changes split into reviewable, revertible commits — not one giant PR
