# EditorConfig + StyleCop for ASP.NET Core

A small, reusable setup for enforcing a consistent C# code style in any .NET project. Follow the steps below to add it to a new component in about 15 minutes.

## The big picture

Three things work together. It helps to know who does what:

| Piece | Role | Analogy |
|---|---|---|
| **Analyzers** (built-in .NET analyzers, StyleCop.Analyzers) | Define the rules and do the checking. They produce the warnings. | The inspector |
| **`.editorconfig`** | Says which rules are on and how severe each one is. Also holds formatting settings like indentation. | The checklist |
| **`stylecop.json`** | Tunes how a few StyleCop rules behave (for example, how `using` directives are ordered). | The inspector's fine print |

```
Your C# code  ->  analyzers check it  ->  warning shown in IDE / build
                        ^
          .editorconfig decides which rules are on and how severe
```

Key idea: `.editorconfig` cannot invent a rule. It can only turn on, turn off, or re-grade a rule that an analyzer already provides. If you remove StyleCop but leave `SA` lines in `.editorconfig`, nothing happens.

## What is EditorConfig?

A plain text file named `.editorconfig` that lives in your repo. Visual Studio, Rider, VS Code (C# extension), `dotnet build`, and `dotnet format` all read it, so everyone gets the same rules no matter which IDE they use.

It holds two kinds of settings:

```ini
# 1. Basic formatting (almost every editor understands these)
indent_style = space
indent_size = 4

# 2. .NET style and rule severity (understood by Roslyn tools)
csharp_style_namespace_declarations = file_scoped:warning
dotnet_diagnostic.SA1600.severity = none
```

### Severity levels

| Severity | Effect |
|---|---|
| `error` | Fails the build |
| `warning` | Shows in the build output and error list |
| `suggestion` | Subtle hint in the IDE only |
| `silent` | No marker, but the quick fix is available |
| `none` | Rule is turned off |

### How lookup works

The tooling starts in the file's folder and walks **up** the directory tree, collecting every `.editorconfig` until it finds one with `root = true`. Closer files override farther ones. Inside one file, later sections override earlier ones.

## What is StyleCop?

`StyleCop.Analyzers` is a NuGet package of Roslyn analyzers. It checks naming, spacing, ordering, layout, and documentation. Its rules have IDs starting with `SA`:

| Range | Area | Example |
|---|---|---|
| SA1000s | Spacing | spaces around operators |
| SA1100s | Readability | avoid multiple blank lines |
| SA1200s | Ordering | `using` placement, member order |
| SA1300s | Naming | SA1300: names must start upper-case |
| SA1400s | Maintainability | explicit access modifiers |
| SA1500s | Layout | braces, blank lines |
| SA1600s | Documentation | XML docs, file headers |

## Setup

Do this on a feature branch, not on a protected branch.

### 1. Create `.editorconfig`

Run from the folder that contains the `.sln`:

```bash
dotnet new editorconfig
```

### 2. Test that EditorConfig works (before adding StyleCop)

Add this to the bottom of `.editorconfig`:

```ini
[*.cs]
csharp_style_namespace_declarations = file_scoped:warning
```

Write a class with a block-scoped namespace:

```csharp
namespace MyApi.Api
{
    public class WeatherForecast { }
}
```

You should see a squiggle with the quick fix **Convert to file-scoped namespace** (`Ctrl+.`). Change `:warning` to `:none` and the squiggle should disappear. If it follows your edits, EditorConfig is working.

### 3. Add StyleCop to the project

From the folder containing the `.csproj`:

```bash
dotnet add package StyleCop.Analyzers --prerelease
```

`--prerelease` gives the beta line, which is the one most people use. The older stable `1.1.118` is outdated.

To apply it to every project in a solution, put it in a `Directory.Build.props` at the solution root instead:

```xml
<Project>
  <PropertyGroup>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.556">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <AdditionalFiles Include="$(MSBuildThisFileDirectory)stylecop.json" Link="stylecop.json" />
  </ItemGroup>
</Project>
```

`PrivateAssets=all` stops the analyzer from flowing to projects that reference yours. `EnforceCodeStyleInBuild` makes IDE-style rules (like IDE0161) show up in `dotnet build`, not only in the editor.

### 4. Add `stylecop.json`

Next to the `.csproj`, or at the solution root if you use `Directory.Build.props`:

```json
{
  "$schema": "https://raw.githubusercontent.com/DotNetAnalyzers/StyleCopAnalyzers/master/StyleCop.Analyzers/StyleCop.Analyzers/Settings/stylecop.schema.json",
  "settings": {
    "orderingRules": {
      "usingDirectivesPlacement": "outsideNamespace",
      "systemUsingDirectivesFirst": true
    },
    "layoutRules": {
      "newlineAtEndOfFile": "require"
    }
  }
}
```

If you are not using `Directory.Build.props`, register it in the `.csproj`:

```xml
<ItemGroup>
  <AdditionalFiles Include="stylecop.json" />
</ItemGroup>
```

### 5. Tune the noisy rules in `.editorconfig`

StyleCop is strict by default. Add this to the `[*.cs]` section:

```ini
[*.cs]
# Documentation and headers: too noisy for most API projects
dotnet_diagnostic.SA1600.severity = none
dotnet_diagnostic.SA1601.severity = none
dotnet_diagnostic.SA1602.severity = none
dotnet_diagnostic.SA1633.severity = none
dotnet_diagnostic.SA1636.severity = none

# Conflicts with the usual ASP.NET Core style (_camelCase fields, no this.)
dotnet_diagnostic.SA1101.severity = none
dotnet_diagnostic.SA1309.severity = none

# Allow DTOs/records to share a file, and relax trailing commas
dotnet_diagnostic.SA1402.severity = none
dotnet_diagnostic.SA1649.severity = none
dotnet_diagnostic.SA1413.severity = none

# Generated code
[**/Migrations/*.cs]
generated_code = true
```

### 6. Build and look at the warnings

```bash
dotnet clean
dotnet build
```

If nothing shows up, close and reopen the solution so the IDE reloads the analyzers.

### 7. See it work

Rename a class to lowercase:

```csharp
public class weatherForecast { }
```

Build and you get:

```
warning SA1300: Element 'weatherForecast' should begin with an upper-case letter
```

Rename it back and the warning disappears.

Note: StyleCop does **not** flag `Weather_Forecast`, because it only checks for an upper-case first letter. Underscores in type names are caught by the built-in rule **CA1707**, which is off by default:

```ini
dotnet_diagnostic.CA1707.severity = warning
```

### 8. Auto-fix

```bash
dotnet format
```

In the IDE, press `Ctrl+.` on a warning for a quick fix. Review the diff afterwards.

## ASP.NET Core notes

- **Migrations and generated files** create noise. The `generated_code = true` section above handles migrations.
- **`Program.cs` with top-level statements** can trip some rules. Suppress them for that file if they are not useful:
  ```ini
  [Program.cs]
  dotnet_diagnostic.SA1200.severity = none
  ```
- **Documentation rules (SA16xx)** are usually the noisiest in API projects. Disable them for internal code.
- Use **file-scoped namespaces** with `usingDirectivesPlacement: outsideNamespace`.

## Enforcing in CI

Make violations fail the pipeline without blocking local development:

```bash
dotnet build -warnaserror
dotnet format --verify-no-changes
```

## Troubleshooting

| Problem | Fix |
|---|---|
| No warnings appear | Run `dotnet clean`, rebuild, and reopen the solution |
| `.editorconfig` seems ignored | Check the filename (leading dot, no `.txt`) and that no closer `.editorconfig` overrides it |
| `SA` warnings missing but IDE rules work | Confirm the `StyleCop.Analyzers` package reference exists |
| `stylecop.json` has no effect | Confirm it is listed as `AdditionalFiles` |
| Hundreds of warnings on an existing codebase | Start with rules at `suggestion`, then tighten gradually |

## Do you need StyleCop at all?

Not always. The built-in .NET analyzers plus a good `.editorconfig` already cover language style (`var`, namespaces, braces), naming rules, and code quality (CA rules). StyleCop adds stricter member ordering, `using` sorting, spacing and layout details, and documentation rules. Add it when you want strict, opinionated consistency or your team already uses it.

## Alternatives

| Tool | What it does |
|---|---|
| **Built-in .NET analyzers** (`CA` and `IDE` rules) | Ships with the SDK. Code quality and style, configured through `.editorconfig`. |
| **Roslynator** | Large set of analyzers and refactorings, less strict about layout than StyleCop. |
| **SonarAnalyzer.CSharp** (SonarLint) | Focuses on bugs, code smells, and security issues. |
| **Meziantou.Analyzer** | Extra rules for best practices and performance. |
| **CSharpier** | Opinionated code formatter, like Prettier for C#. Removes style debates entirely. |
| **`dotnet format`** | Applies `.editorconfig` formatting and many analyzer fixes from the command line. |
| **ReSharper / Rider inspections** | IDE-level inspections configured through `.DotSettings` files. |

Common combinations: built-in analyzers + `.editorconfig` for a light setup, add StyleCop for strict style, or use CSharpier for formatting plus Roslynator or Sonar for code quality.

## Files to copy into a new component

```
.editorconfig
Directory.Build.props
stylecop.json
```

Copy these three into the next project, adjust the rule severities, and you are done.

## References

- StyleCop.Analyzers rules: https://github.com/DotNetAnalyzers/StyleCopAnalyzers
- .NET code analysis overview: https://learn.microsoft.com/dotnet/fundamentals/code-analysis/overview
- EditorConfig for .NET: https://learn.microsoft.com/dotnet/fundamentals/code-analysis/code-style-rule-options
