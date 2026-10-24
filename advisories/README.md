# Advisories

One TOML file per advisory, named after its id: `SCD-2026-0001.toml`.
An advisory says that some versions of a package are `malware`, `vulnerable`
or `broken`. sc-deken warns users who have an affected version installed, and
for `malware` it stops offering the prebuilt download.

```toml
id = "SCD-2026-0001"
package = "example-plugin"   # a recipe name; or use repo = "https://github.com/owner/name"
versions = ["0.3.0"]         # affected versions; "*" means all
severity = "vulnerable"      # malware | vulnerable | broken
summary = "One plain sentence for the user."
details = "What is wrong, who is affected, what to do."
published = 2026-10-24
```

Unknown keys are rejected. Check your file with:

    scdeken advisories validate --dir advisories

Pull requests that touch this folder need approval from the code owner, and
CI validates every pull request before it can be merged.
