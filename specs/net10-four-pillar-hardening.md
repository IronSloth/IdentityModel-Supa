# Spec: .NET 10 Four-Pillar Hardening

| Field | Value |
| --- | --- |
| Slug | `net10-four-pillar-hardening` |
| Created | 2026-10-25 |
| Stage | `approved` |
| Brief | ✅ complete |
| PRD | ✅ complete |
| Technical Plan | ✅ complete |

**Test-first:** acceptance

_`none` (default) builds tests after code, as always. `acceptance` (pilot): each AC observable below the browser is covered by a §3.4 chain of stubs → failing acceptance tests → implementation, one chain per test seam. Set by the Analyst from /spec Stage 0; read by everyone else._

> One feature, one file. Three sections, three owners. Section owners never edit
> another owner's section — contradictions are reported upward, not silently patched.

---

## 1. Brief

<!-- OWNER: Analyst. No other agent may edit this section. -->

**Status:** ✅ complete

### 1.1 Problem statement

_What is wrong / missing today, in the user's terms. No solution language._

This repository is ValhallaTech's fork of `scottbrady91/IdentityModel`, taken at upstream's last release (4.1.0, 2024-11-20) and now at `master` `555d31c`. It ships two independent libraries: `src/ScottBrady.IdentityModel` (EdDSA-signed JWT support for Microsoft.IdentityModel, plus Base16 and Base62 encoders) and `src/ScottBrady.IdentityModel.AspNetCore` (a password-rules tag helper and an ASP.NET Identity password validator). The source of the defects below is the improvement review of `555d31c` (`C:\Users\Fred\AppData\Local\Temp\claude\C--Repo-IdentityModel-Supa\ebc03158-8223-460c-98a8-90333d58c9d6\scratchpad\IdentityModel-Supa-review.md`); review IDs (S1, P1, C1, U1, …) are kept for traceability. Items the reviewer reproduced with a probe program are marked **[confirmed]**; the rest come from reading the code.

ValhallaTech wants to take this fork over as its own package, used first by its own projects and later by the public. Today it cannot, for five groups of reasons.

**1. The libraries are not built for the only .NET that will still get security patches.** .NET 8 and .NET 9 both reach end of support on 2026-11-10, 16 days from this brief. After that date only .NET 10 (LTS to November 2028) gets security patches. The core library targets `net8.0;net9.0`. The ASP.NET Core library targets `net6.0;net8.0;net9.0`, and `net6.0` has been out of support since November 2024. The tests target `net8.0;net9.0` and the sample targets `net8.0`. Both CI workflows install the 9.0 SDK. Nothing in the repo builds or tests against .NET 10, so nothing shows that the libraries work there.

**2. The packages still present themselves as upstream's.** Both projects default their package ID to `ScottBrady.*` (no explicit `PackageId`). They name Scott Brady as author and copyright holder (`Copyright 2022 (c) Scott Brady`) and point `PackageProjectUrl` at the upstream repo. All code sits in `ScottBrady.*` namespaces. `SECURITY.md` sends vulnerability reports to `scott@scottbrady91.com`, so anyone who finds a flaw in this fork reports it to someone who does not maintain it. ValhallaTech cannot publish under the current identity. Upstream released the core library as 4.1.0 and the ASP.NET Core library as 2.2.0.

The new identity is free to take. Neither `ValhallaTech.Security.IdentityModel` nor `ValhallaTech.Security.IdentityModel.AspNetCore` exists on nuget.org (both return 404). nuget.org has no ValhallaTech owner and no reserved `ValhallaTech` prefix *(checked 2026-10-25 by the /spec coordinator against nuget.org)*.

There is also no private channel to the fork's maintainers yet. On the canonical repository, `IronSloth/IdentityModel-Supa`, GitHub private vulnerability reporting is disabled *(verified 2026-10-25 by the /spec coordinator via the GitHub API, which returned `{"enabled":false}`)*.

**3. Security defects.**
- **S1** (High, **[confirmed]**). Each `EdDsaSecurityKey` construction writes a custom provider onto the process-wide `CryptoProviderFactory.Default` (`Tokens/EdDsaSecurityKey.cs:16`). That silently replaces any custom crypto provider the host app registered, such as one backed by an HSM or a KMS. Merely creating a key changes cryptographic behavior for unrelated code in the same process.
- **S2** (High). `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKey` copies the private key (`D`) into the JWK whenever the key holds one (`Extensions/ExtendedJsonWebKeyConverter.cs:19`). A developer who builds a public key-set endpoint from their signing key publishes the private key without being told.
- **S3** (Medium). Private key bytes are never cleared. `EdDsa` does not override disposal and the signature provider's `Dispose` is empty, so key material stays in memory after the objects are disposed.
- **S4** (Medium, **[confirmed]**). Base62 decoding is not strict. Characters outside the alphabet are accepted (`"!!"` decodes to `3EC1`). Leading zero bytes are lost (`[00 00 01]` comes back as `[01]`). Different strings therefore decode to the same bytes, and some byte arrays cannot round-trip. That is unsafe for anything used as an identifier or token.
- **S5** (Medium, **[confirmed]**). `EdDsaParameters.Validate()` accepts a 64-byte Ed25519 or 114-byte Ed448 private key, and key creation then fails inside the third-party crypto library. Validation passes keys that cannot be used.
- **S11** (raised by the PM in §2.6 Q-15). `Validate()` checks only lengths and never checks that `D` and `X` belong to the same key pair (`Tokens/EdDsaParameters.cs:22-37`, confirmed by reading). A key built from a mismatched pair passes validation. Per the PM, such a key signs but its signatures never verify; the Analyst has not run this.
- **S6** (Medium). The sample app ships jQuery 3.3.1, which has known XSS advisories, and Bootstrap 4.3.1, which is end of life. Developers copy samples.
  - Every jQuery 3.3.1 advisory is rated medium *(GitHub advisory database, `gh api /advisories?affects=jquery@3.3.1`, run by the /spec coordinator 2026-10-25)*. A High-or-Critical threshold alone would therefore miss all of them.
  - Both libraries exist only as copied files under `wwwroot/js` and `wwwroot/css`. No manifest declares their versions, so nothing can tell that they are outdated (confirmed by file search).
- **S7** (Medium). In the sample, the object labeled as the public key holds the private key: `EdDsaPublicKey` and `EdDsaPrivateKey` wrap the same private `EdDsa` instance (`SampleOptions.cs:10-11`). PII logging (`ShowPII = true`, `Startup.cs:15`) and the developer exception page (`Startup.cs:54`) are on in every environment. These line references were corrected after the PM found that the review's sample line numbers were wrong; the Analyst re-read them.
- **S8** (Low). The test project's AutoFixture 4.18.1 brings in `System.Net.Http` 4.3.0 and `System.Text.RegularExpressions` 4.3.0. Both carry High advisories (test-only).
- **S9** (Low). The only security analyzer, SecurityCodeScan.VS2019 5.6.7, is unmaintained. No other code analysis is turned on, and missing-XML-doc warnings are suppressed (`NoWarn 1591`). The build's "0 warnings" is therefore no evidence of quality.
- **S10** (Low). The build workflow (`.github/workflows/dotnetcore.yml`) has no `permissions:` block, so its token gets the repository default rather than only what the job needs.

**4. Performance and scalability defects.** There are no benchmarks or performance baselines in the repo today. Nothing measured by hand is recorded either.
- **P1 / C2.** Base62 conversion is quadratic in input length (`Encoding/Base62.cs:39-64`). One long input, which may come from an attacker, can hold a thread for a disproportionate time.
  - Encoding a 16 KB input takes about 580 ms at `555d31c` *(scratch probe by the /spec coordinator, 2026-10-25; not in the repo)*.
  - Encoding a 1,024-byte input takes 3.555 ms on average at `555d31c` *(scratch probe on Fred's machine, 2026-10-25; not in the repo)*.
  - This Base62 output treats the whole input as a single number and converts its radix. The cost of producing that output therefore grows faster than input length, however it is implemented (PM, §2.6 Q-12). A second implementation probed on 2026-10-25 produced the same output and was also quadratic, at about 1.5x the speed.
- **P2.** Base16 encoding builds strings per byte, and decoding takes a substring per byte. An odd-length input crashes with `ArgumentOutOfRangeException` (**[confirmed]** on `"abc"`) instead of being rejected as malformed.
- **P3.** Verifying with offsets copies the input and the signature (`Tokens/EdDsa.cs:112`, via `Skip/Take/ToArray`). Signing over spans or offsets copies the input first (`EdDsaSignatureProvider.cs:24,32`).
- **P4.** Every password check builds a new pattern and evaluates it (`Identity/ExtendedPasswordValidator.cs:62`).
- **C1.** Same root cause as S1. A service that builds keys per tenant or per request writes to one shared process-wide object on every construction.
- **C3.** `Extensions/StreamExtensions.cs:13` treats a short read as failure. A stream may return fewer bytes than requested even when more are coming, as network streams under load do, so valid key data can be reported as unreadable. The reviewer did not probe this; it follows from the documented `Stream` contract.
- One property worth keeping: `EdDsa` creates a new signer for each operation (`Tokens/EdDsa.cs:151-159`), so one instance is safe to share across threads.

**5. UX defects.** Two audiences: the developer consuming the libraries, and the end user of an app built with them.

_Developer experience:_
- **U1.** Turning on the extended password rules takes two separate steps that the developer has to already know about: replacing the password options and registering the validator. The sample shows both in `Startup.cs:31-47`.
- **U2.** Invalid settings in `ExtendedPasswordOptions` are silently ignored. Examples are negative values, or a maximum length below the required length.
- **U3.** Several failures throw the wrong exception, or throw too late (**[confirmed]**):
  - `Base62.Decode(null)` throws `NullReferenceException`.
  - Verifying with a private-only key throws `InvalidOperationException` from the third-party library. A key holding only private material cannot verify at all.
  - `ExtendedCryptoProvider.cs:14` throws `IndexOutOfRangeException`.
  - `NewPasswordTagHelper.cs:48` throws a late `NullReferenceException`.

  Grounding also found two input-handling defects the review does not mention; they were found by reading the code, not by running it (see Q-10):
  - `EdDsa.Verify(byte[], int, int, byte[], int, int)` does not check that offset plus length fits inside the array. Out-of-range or negative offsets are silently truncated or clamped by `Skip/Take` instead of rejected (`Tokens/EdDsa.cs:105-113`).
  - That overload rejects a zero-length input, while `Verify(byte[], byte[])` accepts one (`Tokens/EdDsa.cs:109`, `:115-125`).
- **U4** (**[confirmed]**). The span `Sign` implements a try-pattern but throws when the destination is too small, instead of returning `false` (`Tokens/EdDsaSignatureProvider.cs:22-28`).
- **U5** (**[confirmed]**). `KeySize` returns bytes (32). The .NET convention, which consumers rely on, is bits (256) (`Tokens/EdDsa.cs:85`).
- **U6** (**[confirmed]**). The key ID (`kid`) is dropped on conversion to a JWK, so consumers cannot select the right key during rotation.
- **U7.** Six ChaCha20 constants are left over from the removed Branca/PASETO support. The key-type constant for `"OKP"` (Octet Key Pair) is named `Ecdh` (`Crypto/ExtendedSecurityAlgorithms.cs:5-11, 28`).
- **U13** (raised by the PM in §2.6 Q-22). `EdDsa`'s `ImportFromPem` and both `ImportFromEncryptedPem` overrides throw `NotImplementedException` (`Tokens/EdDsa.cs:161-163`). That reads as unfinished code. It does not say the operation is unsupported, which in .NET is signaled with `NotSupportedException`.
- **U8.** Nullable annotations are off and most public members have no XML docs, so IntelliSense tells the developer little.
- **U9.**
  - `README.md` documents Branca and PASETO, which were removed.
  - It links to upstream.
  - It contains a non-compiling example (line 152).
  - There is no package readme and no changelog.

_End-user experience (people filling in a password form):_
- **U10** (**[confirmed]**). With `MaxConsecutiveChars` set, a whitespace-only password throws instead of failing validation. The user sees a server error instead of "invalid password" (`Identity/ExtendedPasswordValidator.cs:57`).
- **U11.** Validation messages are hard-coded English with inconsistent wording ("Passwords must…" in one place, "Password must…" in another; `ExtendedPasswordValidator.cs:28-43`). The consuming app cannot reword or translate them the way it can the built-in ASP.NET Identity messages.
- **U12.** The sample uses the older `Startup` hosting style and end-of-life Bootstrap 4 styling. On the password form, the label targets `Password` while the input's id is `password` (`Views/Home/PasswordRules.cshtml:34-37`). Because HTML ids are case-sensitive, the label appears not to be associated with the field. This was observed by reading the view, not by rendering it.

**Supporting gaps.**
- The test project collects no coverage, so current coverage is unknown.
- Nothing detects an accidental breaking change to the public API.
- Dependencies are behind:
  - Microsoft.IdentityModel 7.7.1, where 8.x is current.
  - BouncyCastle.Cryptography 2.4.0, where 2.7.0 is current. The repo's `.csproj` pins 2.4.0. The 2.6.2 Fred mentions in §1.8 GP-1 is the version assetvault-io uses.
  - The test and ASP.NET Core packages are on 8.0.x.
- Nothing tells the maintainers when a dependency falls behind or gains an advisory.

The assertion library, FluentAssertions 6.12.2 (Apache-2.0), has no license problem today. Its 8.x line is not free for commercial use, and ValhallaTech is a business, so the test suite's assertion library has no free upgrade path (C-18).

### 1.2 Who is affected

_Roles, systems, or teams that feel the problem. Rough scale if known._

- **Developers of ValhallaTech's own projects.** They are the first consumers (R1-Q4). They need .NET 10-ready packages by 2026-11-10, and they are exposed to S1/S2 and the U1–U9 friction. The first known consumer is **assetvault-io** (`C:\Repo\assetvault-io`, `net10.0`):
  - It uses upstream `ScottBrady.IdentityModel` 4.1.0 for EdDSA JWT signing.
  - It builds keys with `EdDsa.Create(EdDsaParameters)` and `new EdDsaSecurityKey(...) { KeyId = kid }` (`src/AssetVault.BLL/Tokens/EdDsaKeyMaterial.cs:51-72`).
  - Its signing key is built from private material only, and its verification key from public material only.
  - Every key it constructs triggers S1.
  - It already runs on Microsoft.IdentityModel.JsonWebTokens 8.19.1 and BouncyCastle.Cryptography 2.6.2.
- **Public developers.** They are future consumers once the libraries are published to nuget.org (R1-Q4). They inherit the same defects and the same lack of documentation.
- **End users of apps built on the libraries.** They fill in password forms that can return server errors (U10) or English-only, inconsistent messages (U11). Scale unknown.
- **Operators of host apps** that register their own crypto provider (HSM/KMS). S1 overrides it silently.
- **Security researchers and reporters.** They are routed to the upstream author (`SECURITY.md`). The fork offers no private alternative yet (C-21).
- **The fork's maintainers (Fred / ValhallaTech).** They cannot publish under the current identity. They have no meaningful warning signal, no coverage figure, no performance baseline, and CI on an SDK that goes out of support.
- **The original author, Scott Brady.** He receives the security reports and is named as author of a package line he does not maintain here.

### 1.3 Goals

_Outcomes that would mean the problem is solved. Observable, not implementation._

_Platform and identity_
- **G-1** — ValhallaTech projects running on .NET 10 can consume both libraries from the private release on or before 2026-11-10. Both libraries build, and the full test suite passes, on .NET 10. (R2-Q1, R2-Q3, R4-Q1)
- **G-2** — Everything a consumer sees identifies ValhallaTech's fork, not upstream: package ID, namespaces, author, project URL, copyright, description and icon. The icon is the ValhallaTech company logo, which Fred supplies (GP-9). The original author's attribution that the license requires is kept. (R3-Q1, R3-Q2, R2-Q4, C-16)
- **G-3** — A person who finds a vulnerability is directed to a private channel that reaches this fork's maintainers, and that channel is open. (R3-Q3, GP-7. Opening it depends on a repository setting only Fred can change; see C-21.)

_Security_
- **G-4** — Constructing or using a key from these libraries never changes cryptographic behavior for any other key, provider, or code in the same process. (S1, C1)
- **G-5** — Private key material leaves the library only when the developer has explicitly asked for it. Converting a key for publication yields only public material unless the developer explicitly asks otherwise. (S2; the shape of the request is D-4)
- **G-6** — Once a key is disposed, no copy of its private key material remains in memory the library itself controls. (S3; see NG-8)
- **G-7** — Encoded data is unambiguous:
  - Every byte sequence, including one with leading zero bytes, decodes back to exactly the bytes that were encoded.
  - Any string with a character outside the encoding's alphabet, or with an invalid length, is rejected with an exception that identifies the input as invalid. It is never decoded, and it never fails with an unrelated exception. Both Base16 and Base62 use `FormatException`. Base62 input over the cap fails with `ArgumentException` (G-14; GP-9).

  (S4, P2; representation is D-3)
- **G-8** — Key validation is the single gate. Parameters that pass validation always produce a usable key. Parameters that would fail later are rejected at validation, with a message that says why. That includes private and public material that do not belong to the same key pair. (S5, S11, GP-5; which lengths are valid is D-5)
- **G-9** — At release time, no dependency carries a known High or Critical advisory or is past end of life. That covers runtime, test-only and sample dependencies, direct or transitive. Every dependency is also at its latest stable version at release, across the runtime, test-only and sample sets. (S6, S8, review §6, GP-1; scope of "every" is A-13) This rule has two bounds:
  - **License.** A dependency whose latest stable version would need a paid commercial license is replaced, not upgraded (C-18, A-3). For the test assertion library, the replacement is D-1.
  - **Sample UI framework.** "Latest stable" applies within the major version Fred fixed in GP-8. A new stable major version that appears before release does not apply to the sample.
- **G-10** — The sample app models safe practice. No object presented as public holds private key material. Diagnostic detail (PII in logs, developer error pages) appears only in a development environment. (S7)
- **G-11** — A clean build means the code passed active security and quality analysis. It never means the analysis was missing or suppressed. (S9)
- **G-12** — CI automation runs with only the repository permissions its jobs need. (S10)

_Performance and scalability_
- **G-13** — Each reported hot path is shown by before/after measurement against `555d31c` to be faster or to allocate less, and no measured path regresses. The hot paths are Base62 encode/decode, Base16 encode/decode, signing and verifying over offsets or spans, and the repeated-character password check. (P1–P4, R3-Q4; tool is D-6; thresholds Q-5)
- **G-14** — No single input can occupy a thread for a disproportionate time. (P1, C2, GP-2)
  - **Base16:** encode and decode time grows in proportion to input length.
  - **Base62:**
    - The library documents a maximum input length.
    - Longer input is rejected rather than processed.
    - At that maximum, encoding and decoding each finish within a stated time budget.
    - Both are measurably faster than at `555d31c`.
    - The values are confirmed (GP-9): the maximum is 1,024 bytes for encoding and 1,376 characters for decoding, and the budget is 3 ms mean per call at that maximum.
- **G-15** — Reading key material from a stream succeeds whenever the stream eventually supplies the data, even when it arrives in fragments. (C3)
- **G-16** — A signing-key object stays safe to share across threads, as it is today. (review §3; `Tokens/EdDsa.cs:151-159`)

_Developer experience_
- **G-17** — Turning on the extended password rules is a single, discoverable step in 1.0.0. An invalid configuration fails with a message that names the offending setting. Through the single step it fails at startup. Through the old two-step setup it fails at first use. A value of 0 for `MaxLength` or `MaxConsecutiveChars` is invalid, and `null` means no limit. (U1, U2, GP-9)
- **G-18** — Every public entry point fails immediately on invalid input with an argument exception that names the argument. No null-reference, index-out-of-range, or third-party exceptions reach the caller. Where G-7, G-8 or G-20 name a specific exception type, that type overrides this rule (GP-9). (U3; Q-10)
- **G-19** — A key that holds only private material can verify signatures. (U3)
- **G-20** — The public API follows the .NET conventions consumers rely on (U4–U7, U13):
  - Try-pattern members report failure through their return value.
  - Key size is reported in bits. Ed448 reports 456 (GP-9).
  - The key ID survives conversion to and from a JWK.
  - The six leftover ChaCha20 constants and the misnamed `KeyTypes.Ecdh` are absent from the 1.0.0 public API (GP-4). If the `"OKP"` value stays public, its name matches its meaning.
  - The PEM import members fail with `NotSupportedException` in 1.0.0 (GP-6).
- **G-21** — IntelliSense shows nullability and a description for every public member. (U8)
- **G-22** — Documentation is accurate and complete (U9):
  - The repo and package documentation describe only features that exist.
  - Every code example in them compiles.
  - Each package carries its own readme.
  - A changelog records every release, including its breaking changes.

_End-user experience_
- **G-23** — Every password input, including a whitespace-only one, produces a validation result, never a server error. (U10)
- **G-24** — Password validation messages are worded consistently. The consuming app can replace or translate them the same way it can the built-in ASP.NET Identity messages. (U11; mechanism is D-11)
- **G-25** — The sample app is kept and still demonstrates the password-rules tag helper. It runs on .NET 10 and uses current hosting conventions and a supported UI framework, without jQuery in the sense of C-23. Its interactive elements, such as the mobile menu, keep working. Every input on its password form has an associated label. (U12, S6, GP-8, C-23; D-8)

_Supporting_
- **G-26** — An accidental breaking change to the public API after 1.0.0 is caught before it is released. (review §5)
- **G-27** — Coverage is measured on every CI build and meets the floor in C-12. (review §5, R4-Q3)
- **G-28** — After release, dependencies stay at their latest stable versions without anyone checking for new versions by hand. Advisories against dependencies reach the maintainers the same way. Code formatting is consistent and checked automatically. (review §5, GP-1; tools are D-7)
  - This covers the sample app's front-end libraries too. Their versions are declared where an automated update check can read them, instead of existing only as copied files in the repo (S6).
- **G-29** — assetvault-io, and any other ValhallaTech project on upstream 4.1.0, can move to 1.0.0 from a written account of every breaking change that affects it and what replaces each one. (Q-4 resolution; this is the consequence A-4 named if it proved false)

### 1.4 Non-goals

_Explicitly out of scope for this feature. Each with a one-line reason._

- **NG-1** — Publishing to nuget.org by 2026-11-10. The deadline counts a private-feed release as shipped, and the public release follows later (R4-Q1, R1-Q4). Whether that public release belongs to this spec is Q-2.
- **NG-2** — Supporting .NET 6, 8 or 9 consumers. No consumer needs .NET 8 after 2026-11-10, and the answer was ".NET 10 only" (R2-Q1).
- **NG-3** — Staying mergeable with upstream or contributing fixes back to it. The fork is independent (R2-Q4).
- **NG-4** — Restoring Branca or PASETO support. The README still documents them, but the defect is the inaccurate documentation (G-22), not the missing feature.
- **NG-5** — Adding new algorithms or token formats, such as the post-quantum algorithms that arrive with .NET 10. The .NET 10 move is a runtime change, not a feature expansion.
- **NG-6** — Bringing ValhallaTech's broader application stack into the libraries, meaning the database, key-value, authorization, container, SPA, logging, mapping and validation components listed in review §8. The libraries have none of those concerns, and a library must not impose them on its consumers (review §8).
- **NG-7** — Process or infrastructure on the scale of Heimdall-TicketTracker. This is a small library, and Fred asked that it stay proportionate (C-17).
- **NG-8** — Removing private key copies held inside the third-party cryptography dependency. That memory is outside the library's control (review S3), so G-6 is bounded to memory the library controls.
- **NG-9** — Migrating existing nuget.org consumers of the upstream `ScottBrady.*` packages. They are upstream's users, not this independent fork's (R2-Q4). ValhallaTech's own projects on upstream 4.1.0 are not part of this non-goal; G-29 covers them.
- **NG-10** — Removing the sample app, or rebuilding it on a different front-end stack. Fred chose to keep it on its current page-rendering model, because that model is what demonstrates the tag helper (GP-8).

### 1.5 Constraints

_Fixed facts the solution must live within: regulatory, platform, timeline, data,
existing-system, organizational. Mark each `hard` or `soft`._

| ID | Constraint | Type | Source |
| --- | --- | --- | --- |
| C-1 | .NET 8 and .NET 9 reach end of support on 2026-11-10 and get no security patches afterward. .NET 10 is LTS until November 2028. | hard | Microsoft, https://devblogs.microsoft.com/dotnet/dotnet-8-9-end-of-support/ |
| C-2 | Only .NET 10 must be supported. No consumer needs .NET 8 after 2026-11-10. | hard | Fred, grilling R2-Q1 |
| C-3 | 2026-11-10 is a hard deadline for shipping the .NET 10 version. | hard | Fred, grilling R2-Q3 |
| C-4 | "Shipped" for the deadline means a private-feed release that ValhallaTech projects can consume. The public nuget.org release follows later. | hard | Fred, grilling R4-Q1 |
| C-5 | The behavior changes are bundled into one major version bump. | hard | Fred, grilling R2-Q2 |
| C-6 | The first release under the new package ID is 1.0.0. | hard | Fred, grilling R4-Q2 |
| C-7 | The package ID prefix is `ValhallaTech.Security.IdentityModel`. | hard | Fred, grilling R3-Q1 |
| C-8 | The `ScottBrady.*` namespaces are renamed to match the new package ID. | hard | Fred, grilling R3-Q2 |
| C-9 | The repo is an independent fork of `scottbrady91/IdentityModel`. | hard | Fred, grilling R2-Q4 |
| C-10 | Security issues are reported through GitHub private reporting. | hard | Fred, grilling R3-Q3 |
| C-11 | Every performance improvement must be proven with before/after measurements. | hard | Fred, grilling R3-Q4 |
| C-12 | The minimum test coverage that counts as done is 80% line coverage. | hard | Fred, grilling R4-Q3 |
| C-13 | The work is run test-first ("let's use TDD"). Each fix is written test-first at unit level as well as per acceptance criterion. | hard | Fred, grilling R1-Q2; unit-level scope from Fred, Gate 1 pre-review GP-3 |
| C-14 | The packages serve both ValhallaTech's own projects and public consumers. | hard | Fred, grilling R1-Q4 |
| C-15 | This spec covers the whole review: .NET 10, build/CI foundations, all four pillars, the sample app and the docs. | hard | Fred, grilling R1-Q3 |
| C-16 | Anyone who distributes a derivative work must include a copy of the license, put a prominent notice on each modified file saying it was changed, and keep all copyright, patent, trademark and attribution notices from the original source. | hard | `LICENSE` (Apache-2.0) §4(a)–(c) |
| C-17 | The repo's process and infrastructure stay proportionate to a small library and do not copy the complexity of Heimdall-TicketTracker. | soft | Fred, conversation before Stage 0 (relayed by /spec) |
| C-18 | FluentAssertions 8.0 and later are under the Xceed Community License, which is free only for non-commercial use. 7.x stays Apache-2.0 but gets bug fixes only. The repo is on 6.12.2 (Apache-2.0). | hard | FluentAssertions NuGet license page (https://packages.nuget.org/packages/FluentAssertions/8.3.0/License), as cited in review §7 and restated by /spec. The Analyst did not re-read it. Version 6.12.2 is from the test `.csproj`. |
| C-19 | The work must also satisfy all four pillars: security, performance, scalability and UX. UX covers both the consuming developer and the end user of an app built with the libraries. | hard | Fred, conversation before Stage 0 (relayed by /spec) |
| C-20 | The deadline stays hard. The complete 1.0.0 release ships by 2026-11-10, carrying the scope in A-1. A prerelease does not count as shipped. | hard | Fred, Gate 1 pre-review GP-1 ("Keep it hard.") |
| C-21 | GitHub private vulnerability reporting is disabled on `IronSloth/IdentityModel-Supa`. Enabling it is a repository setting that Fred must change himself; no agent can make that change. | hard | GitHub API, which returned `{"enabled":false}` (verified 2026-10-25 by the /spec coordinator); Fred, Gate 1 pre-review GP-7 |
| C-22 | `IronSloth/IdentityModel-Supa` is the canonical repository for package metadata and vulnerability reports. | hard | Fred, Gate 1 pre-review GP-7 |
| C-23 | jQuery is never used in any ValhallaTech work, including this repository's sample app. That means no jQuery library files, no references to jQuery, and no package that depends on it. Optional jQuery-detection code inside another library's own bundle is not use of jQuery (GP-10). | hard | Fred, Gate 1 pre-review GP-8 ("jQuery should never be used in any of our work.") |
| C-24 | Base62 has a capped input length, and the encoding decided for D-3 is fixed (GP-2). The Architect does not reopen it. | hard | Fred, Gate 1 pre-review GP-2 |

### 1.6 Open questions

_Anything that would change the shape of the work. Mark `blocking` if the PRD cannot
be written without an answer._

**Open**

| ID | Question | Blocking? | Owner |
| --- | --- | --- | --- |
| Q-2 | Is the public nuget.org release part of this spec's definition of done, or a separate later piece of work? Assumed separate (NG-1). | no | Fred |
| Q-5 | How large an improvement counts for each G-13 hot path? Is any measured gain enough, or is there a minimum per path? R3-Q4 requires measurement but sets no threshold. Assumed any measured gain (A-8). | no | Fred / PM |
| Q-6 | Does the 80% line-coverage floor (C-12) apply to each packable library separately or to both combined? Is the sample app excluded? Assumed each library separately, with the sample excluded (A-6). | no | Fred |
| Q-9 | What is the largest Base62 or Base16 input that consumers realistically need to handle? Do any consumers depend on a size bound? The Base62 cap is now fixed (GP-9). This question remains only to show whether a consumer needs more than the cap allows. The only known consumer, assetvault-io, uses neither encoder. | no | Fred |
| Q-10 | Grounding found two defects the review does not report: unchecked offsets and lengths in `EdDsa.Verify(byte[], int, int, byte[], int, int)`, and its rejection of zero-length input where the other overload accepts it (`Tokens/EdDsa.cs:105-125`). Both are public entry-point input handling and are assumed covered by G-18 (A-11). Is that right? | no | Fred |
No §2.6 question is open after Gate 1. Fred's Gate 1 answers are in the Resolved table below (GP-9).

**Resolved**

| ID | Question (short) | Resolution | Source |
| --- | --- | --- | --- |
| Q-1 | Is the full scope due by 2026-11-10? | Yes. The deadline stays hard, and the complete 1.0.0 release ships by 2026-11-10, carrying every behavior-changing fix (A-1 confirmed). Non-breaking work may follow in 1.x. | Fred, Gate 1 pre-review GP-1 |
| Q-3 | Are the new IDs available on nuget.org? | Yes, both IDs are unused (404). nuget.org has no ValhallaTech owner and no reserved `ValhallaTech` prefix. | nuget.org lookup by the /spec coordinator, 2026-10-25 |
| Q-4 | Does any ValhallaTech project use `ScottBrady.*` today? | Yes. assetvault-io (`net10.0`) uses `ScottBrady.IdentityModel` 4.1.0 (`src/AssetVault.BLL/Tokens/EdDsaKeyMaterial.cs:51-72`, re-read by the Analyst). It is the first known consumer, and it is exposed to S1. A-4 is false, which adds G-29. | /spec coordinator, 2026-10-25; Analyst re-read |
| Q-7 | Must each fix also be test-first at unit level? | Yes. Each fix is written test-first at unit level, not only per acceptance criterion (C-13). | Fred, Gate 1 pre-review GP-3 |
| Q-8 | Which repository is canonical, and is private reporting enabled? | `IronSloth/IdentityModel-Supa` (C-22). Private vulnerability reporting there is disabled, and only Fred can enable it (C-21). | Fred, Gate 1 pre-review GP-7; GitHub API, 2026-10-25 |
| Q-11 | Can consumers take Microsoft.IdentityModel 8.x? | Yes. assetvault-io already uses Microsoft.IdentityModel.JsonWebTokens 8.19.1 and BouncyCastle.Cryptography 2.6.2 (A-5 confirmed for the known consumer). | /spec coordinator, 2026-10-25 |
| Q-12 (§2.6) | Base62 cannot be linear while keeping today's output. Which route? | A documented maximum input length, with longer input rejected, a time budget at that maximum, and measurably faster than today (G-14). | Fred, Gate 1 pre-review GP-2 |
| Q-14 (§2.6) | Remove the leftover constants now, or mark them obsolete? | Remove the six ChaCha20 constants and `KeyTypes.Ecdh` in 1.0.0 (G-20). | Fred, Gate 1 pre-review GP-4 |
| Q-15 (§2.6) | Does G-8 cover a `D` and `X` that do not belong together? | Yes. Validation rejects them (G-8, S11). | Fred, Gate 1 pre-review GP-5 |
| Q-20 (§2.6) | Would a 1.0.0 prerelease count as shipped? | No. The complete 1.0.0 release is required (C-20). | Fred, Gate 1 pre-review GP-1 |
| Q-22 (§2.6) | Should the PEM import overrides throw `NotSupportedException`? | Yes, in 1.0.0 (G-20, U13). | Fred, Gate 1 pre-review GP-6 |
| Q-16 (§2.6) | On disposal, is the caller's own `D` array zeroed, or only the library's copies? | Only the library's own copies. A caller-supplied array is never modified. assetvault-io passes its own array as `D` (`EdDsaKeyMaterial.cs:53`). | PM decision in §2.6, as directed by the /spec coordinator |
| Q-19 (§2.6) | Should the sample's vendored client libraries be held to an any-severity advisory rule? | Yes. Every jQuery 3.3.1 advisory is rated medium, so a High-or-Critical threshold would miss them all (S6). | `gh api /advisories?affects=jquery@3.3.1`, run by the /spec coordinator, 2026-10-25 |
| Q-23 | What are the Base62 cap and time budget? | Confirmed: a cap of 1,024 bytes for `Encode` and 1,376 characters for `Decode`, and a budget of 3 ms mean per call at the cap. Measured at 1,024 bytes on Fred's machine *(scratch probe, 2026-10-25, 2,000 calls, not in the repo)*: 1.577 ms to encode and 0.809 ms to decode with the decided encoding, against 3.555 ms for today's encode, with 1,375-character output. | Fred, Gate 1 (GP-9) |
| Q-24 (§2.6) | What does Base62 throw for invalid input? | `FormatException` for invalid characters. `ArgumentException` for input over the cap. | Fred, Gate 1 (GP-9) |
| Q-13 (§2.6) | What is Ed448's `KeySize`? | 456. | Fred, Gate 1 (GP-9) |
| Q-17 (§2.6) | How are 0 and `null` treated in `MaxLength` and `MaxConsecutiveChars`? | 0 is rejected, and `null` means no limit, for both settings. This ships in 1.0.0 however the validator is registered. | Fred, Gate 1 (GP-9) |
| Q-18 (§2.6) | What is the package icon? | The ValhallaTech company logo, a PNG that Fred supplies before release. | Fred, Gate 1 (GP-9) |
| Q-21 (§2.6) | Do the specific exception types override G-18? | Confirmed. The types set by G-7, G-8 and G-20 override G-18's general argument-exception rule. | Fred, Gate 1 (GP-9) |
| Q-25 (§2.6) | When does the one-step setup helper ship? | In 1.0.0. Bad options fail at startup through the helper, and at first use through the old two-step path. | Fred, Gate 1 (GP-9) |

### 1.7 Assumptions

_What is being taken as true in the absence of an answer, and what breaks if it's wrong._

- **A-1** — **Confirmed** by Fred, Gate 1 pre-review GP-1 (see C-20). The 1.0.0 private-feed release due by 2026-11-10 carries .NET 10 support, the new identity and every behavior-changing fix (S2's default, U5's unit, the Base62 output change, the namespace rename). Non-breaking improvements may follow in later 1.x releases. This is the only reading consistent with R2-Q2, R2-Q3 and R4-Q2 together (Q-1). → If false, either the deadline slips or a second major version is needed, which contradicts R2-Q2. Sixteen days is tight for the full breaking set.
- **A-2** — Both packages take the prefix from C-7: `ValhallaTech.Security.IdentityModel` and `ValhallaTech.Security.IdentityModel.AspNetCore`. Both restart at 1.0.0, replacing upstream's separate 4.1.0 and 2.2.0 lines. Basis: R3-Q1 and R4-Q2. → If false, the identity, namespace and changelog work changes shape, and the two packages need separate version plans.
- **A-3** — ValhallaTech will not buy commercial licenses for test-only tooling, so FluentAssertions 8.x is not an option. Basis: ValhallaTech is a business (/spec context) and C-18. → If false, D-1's option set widens.
- **A-4** — **Proved false** (Q-4 resolution): assetvault-io depends on upstream 4.1.0. Its stated consequence is now G-29. The original assumption read: No ValhallaTech project depends on the upstream `ScottBrady.*` packages, so nobody inside ValhallaTech needs a migration path (Q-4). → If false, those projects need migration guidance for the identity and namespace change.
- **A-5** — **Confirmed for the known consumer** (Q-11 resolution): assetvault-io is already on Microsoft.IdentityModel 8.19.1 and BouncyCastle 2.6.2. Consuming ValhallaTech apps can accept whatever minimum Microsoft.IdentityModel and BouncyCastle versions the libraries require (Q-11). → If false, G-9's "latest stable" target is capped by consumer compatibility.
- **A-6** — The 80% floor applies to each packable library separately, and the sample app is excluded (Q-6). → If false, the coverage target and its CI enforcement change.
- **A-7** — **Confirmed** by Fred, Gate 1 pre-review GP-7 (now C-22). `IronSloth/IdentityModel-Supa` is the canonical repository for package metadata and vulnerability reports (Q-8). → If false, the project URL and the security-reporting destination point to the wrong place.
- **A-8** — For G-13, any measured improvement on each hot path is enough, as long as no measured path regresses. The PM sets firmer targets if Fred wants them (Q-5). → If false, some performance items may need deeper work than a straightforward fix.
- **A-9** — The review's findings hold at `555d31c`. The Analyst re-read S1 (`EdDsaSecurityKey.cs:16`), S2 (`ExtendedJsonWebKeyConverter.cs:19`), U5 (`EdDsa.cs:85`), P3 (`EdDsa.cs:112`), and the target frameworks and package metadata in each `.csproj`. The other findings were not re-run. → If one is false, its goal is moot. The test-first discipline (C-13) shows this cheaply, because the failing test cannot be written.
- **A-10** — The C3 short-read defect is real under the documented `Stream` contract, although nobody has reproduced it here. → If false, G-15 costs a test and nothing else.
- **A-11** — The two unreported `EdDsa.Verify` defects (Q-10) fall within G-18's scope. → If false, they need their own decision (fix in their own change, or drop with a reason), and G-18 should name them as excluded.
- **A-12** — "Proportionate" (C-17) excludes added process, services or pipelines that a two-package library cannot justify. It does not exclude the supporting goals G-26 to G-28, which the review places under "kept small" and R1-Q3 brings into scope. → If false, some of G-26 to G-28 become non-goals.
- **A-13** — In G-9's "every dependency is at its latest stable version", *every* means every **direct** dependency of every project: the libraries, the tests and the sample. Transitive dependencies resolve through their direct parents and are governed by G-9's advisory and end-of-life rule. Fred's wording (GP-1) is "all of the dependencies". → If false, transitive packages need their own currency rule, and pinning them can conflict with what their parent packages declare.
- **A-14** — At release, the latest stable version of every direct dependency works on .NET 10 and works alongside the others. → If false, G-9 needs an exception rule that Fred has not given, and the conflict surfaces as a release-time decision against a hard deadline (C-20).
- **A-15** — **Confirmed** by Fred at Gate 1 (GP-9). The Base62 cap is 1,024 bytes for encoding and 1,376 characters for decoding, with a 3 ms budget, as the PM proposed. A measurement at 1,024 bytes (1.577 ms encode, 0.809 ms decode; Q-23) supports the budget. The if-false consequence no longer applies.

### 1.8 Grilling record
_Written by the Analyst, verbatim from /spec Stage 0. Decision IDs are cited as sources elsewhere in the spec._

| ID | Question | Answer | Recommendation |
|---|---|---|---|
| R1-Q1 | What slug should this spec use? | net10-four-pillar-hardening (Recommended) | took |
| R1-Q2 | Run this spec test-first (acceptance tests before code)? | "I'm assuming you're asking about TDD. If so, the answer is yes, let's use TDD." | overrode |
| R1-Q3 | How much of the review should this spec cover? | Everything (Recommended) | took |
| R1-Q4 | Who will consume this package? | Both | none offered |
| R2-Q1 | Do any consumers need to stay on .NET 8 after 2026-11-10? | No, .NET 10 only (Recommended) | took |
| R2-Q2 | Is a new major version with breaking changes acceptable? | Yes, one major bump (Recommended) | took |
| R2-Q3 | Is 2026-11-10 a hard deadline for shipping the .NET 10 version? | Hard deadline | none offered |
| R2-Q4 | How will this fork relate to upstream (scottbrady91/IdentityModel)? | Independent fork (Recommended) | took |
| R3-Q1 | What should the package ID prefix be? | ValhallaTech.Security.IdentityModel | overrode |
| R3-Q2 | Should the ScottBrady.* namespaces be renamed to match the new package ID? | Yes, rename (Recommended) | took |
| R3-Q3 | How should people report security issues? | GitHub private reporting (Recommended) | took |
| R3-Q4 | Must performance improvements be proven with before/after measurements? | Yes, measured (Recommended) | took |
| R4-Q1 | What counts as "shipped" for the 2026-11-10 deadline? | Private feed release (Recommended) | took |
| R4-Q2 | What version should the first release under the new ID be? | 1.0.0 (Recommended) | took |
| R4-Q3 | What minimum test coverage counts as done? | 80% line (Recommended) | took |

#### Deferred to Technical Plan
_Solution-space questions raised during grilling and not asked. The Architect resolves each in §3 or raises it at Gate 2._
- D-1 Which assertion library replaces or updates FluentAssertions 6.12.2 for the tests.
- D-2 Which private NuGet feed hosts the ValhallaTech release. — _[Annotation added after Stage 0, not part of the verbatim record: **decided by Fred, GP-10.** The Architect treats it as fixed.]_
- D-3 How Base62 should represent leading zero bytes so encoding round-trips. — _[Annotation added after Stage 0, not part of the verbatim record: **decided by Fred, GP-2.** The Architect treats it as fixed.]_
- D-4 The shape of the opt-in for exporting a private key into a JWK.
- D-5 Whether 64-byte Ed25519 / 114-byte Ed448 private keys are supported or rejected.
- D-6 Which tool produces the before/after performance measurements.
- D-7 Which tools handle dependency updates and code formatting.
- D-8 Whether the sample app uses DryIoc, and which UI framework version it uses. — _[Annotation added after Stage 0, not part of the verbatim record: **the UI framework part is decided by Fred, GP-8.** Whether to use DryIoc is still open.]_
- D-9 Solution file format and shared build-property layout.
- D-10 Whether to keep the three-OS CI matrix and the CodeQL workflow.
- D-11 The mechanism for localizing password validation error messages.

#### Gate 1 pre-review decisions
_Fred's decisions of 2026-10-25, asked before Gate 1, at Gate 1 (GP-9) and at Gate 2 (GP-10), and relayed by the /spec coordinator. Recorded as relayed, and cited elsewhere as GP-n. Three are decided and fixed for the Architect: SimpleBase and the Base62 cap (GP-2, which decides D-3), the sample's UI framework (GP-8, which decides the UI framework part of D-8), and the private feed (GP-10, which decides D-2). Every other library or tool named here is Fred's stated preference, and that choice stays with the Architect under the D-item named. For example, GP-1's Renovate is a lean for D-7._

- **GP-1** (Q-1, Q-20). The deadline stays hard, and the full 1.0.0 scope ships by 2026-11-10, not a prerelease. Fred, verbatim: "Keep it hard. Addendum: with BouncyCastle being version 2.6.2 and the current C# version being 2.7.0, we should probably make sure all of the dependencies are up-to-date, or we can have RenovateBot take care of it." The coordinator recorded the new goal as: every dependency is at its latest stable version at release, and stays current automatically afterwards. Fred leans toward Renovate for D-7.
- **GP-2** (Q-12, D-3). **Decided by Fred:** Base62 uses SimpleBase (https://github.com/ssg/SimpleBase), meaning `SimpleBase.Base62.Default`, and its input length is capped. The version was 5.6.4 when the decision was made. Under GP-10 it now follows G-9's latest-stable rule and is not pinned. This decides D-3, and the Architect treats it as fixed. Fred first suggested SimpleBase. On learning that it is still quadratic, he chose the cap plus a time budget: a documented maximum input length, longer input rejected, a time budget at that maximum, and measurably faster than today. He then confirmed both as decisions, verbatim: "Make sure we're aware that we're using SimpleBase and the cap decision. I've only seen SimpleBase mentioned once." Facts presented to him (coordinator's scratch probe, 2026-10-25, not in the repo):
  - SimpleBase 5.6.4 is Apache-2.0, targets net8.0/net9.0/net10.0, and was published 2026-08-09.
  - For 5,120 random inputs without leading zeros, `Base62.Default` gives the same output as the current `Base62.Encode`, and the README sample `AAwf93rvy4aWQVw` is unchanged.
  - It encodes leading zero bytes as '0' and round-trips them.
  - It rejects invalid characters with `ArgumentException`.
  - It is still quadratic, at about 1.5x the speed: encoding 16 KB takes 580 ms today and 380 ms with SimpleBase.
- **GP-3** (Q-7). Yes. Each fix is also written test-first at unit level, not only per acceptance criterion.
- **GP-4** (Q-14). Remove the six ChaCha20 constants and `KeyTypes.Ecdh` in 1.0.0. Do not merely mark them obsolete.
- **GP-5** (Q-15). Yes. Key validation rejects a private key (D) and a public key (X) that do not belong together.
- **GP-6** (Q-22). Yes. The PEM import overrides throw `NotSupportedException` in 1.0.0.
- **GP-7** (Q-8). The canonical repository is IronSloth/IdentityModel-Supa. GitHub private vulnerability reporting is currently disabled there (GitHub API: `{"enabled":false}`). Enabling it is a repository setting Fred must change himself.
- **GP-8** (D-8, S6). Keep the sample app. It stays Razor because it demonstrates the tag helper, and it moves to the latest stable Bootstrap 5 (5.3.8 on npm on 2026-10-25; 6.0.0 exists only as an alpha). Fred, verbatim: "If we're going to keep the sample app, then we should definitely upgrade to the latest version of Bootstrap 5 or use the latest versions of Svelte/Sveltekit. jQuery should never be used in any of our work." He then chose "Keep it, Bootstrap 5" over a SvelteKit rebuild or removing the sample. The major version is fixed at 5. If Bootstrap 6 becomes stable before release, the sample still takes the latest stable 5.x (see G-9's sample bound).
- **GP-9** (Gate 1 answers, Fred, 2026-10-25, relayed by the /spec coordinator).
  - **Q-23:** the cap of 1,024 bytes for `Encode` / 1,376 characters for `Decode` and the 3 ms budget are confirmed.
  - **Q-24:** Base62 translates invalid characters into `FormatException`; over-cap input stays `ArgumentException`.
  - **Q-13:** Ed448 `KeySize` = 456.
  - **Q-17:** 0 is rejected and `null` means no limit, for both `MaxLength` and `MaxConsecutiveChars`. This ships in 1.0.0 however the validator is registered.
  - **Q-18:** the package icon is the ValhallaTech company logo, a PNG Fred supplies before release.
  - **Q-21:** confirmed. The specific exception types override G-18's general rule.
  - **Q-25:** the one-step setup helper ships in 1.0.0. Bad options fail at startup through the helper, and at first use through the old two-step path.
- **GP-10** (Gate 2 change requests, Fred, 2026-10-25, relayed by the /spec coordinator).
  - **D-2, private feed:** "GitHub Packages (Recommended)". The feed is GitHub Packages under IronSloth, next to the repository. This decides D-2.
  - **Bootstrap JS:** "Ship it, fix the checks (Recommended)".
    - The sample ships Bootstrap 5's JavaScript so the mobile menu keeps working.
    - Fred's no-jQuery rule means no jQuery library files, no references to jQuery, and no package that depends on it. It does not cover the optional detection text inside Bootstrap's own bundle.
    - Measured on 2026-10-25 *(per the /spec coordinator)*: `bootstrap.bundle.min.js` 5.3.8 contains 19 "jQuery" mentions, all of them `window.jQuery` checks or the optional `jQueryInterface` hooks. `bootstrap.min.css` contains none. Bootstrap 5 does not need jQuery.
  - **SimpleBase version:** "Latest stable (Recommended)". SimpleBase follows the G-9 latest-stable rule like every other dependency and is no longer pinned to 5.6.4. The library choice itself (GP-2) is unchanged.

---

## 2. PRD

<!-- OWNER: Product Manager. No other agent may edit this section. -->

**Status:** ✅ complete

### 2.1 Summary

_Two or three sentences: what gets built, for whom, and what changes for them._

ValhallaTech releases the fork as `ValhallaTech.Security.IdentityModel` and `ValhallaTech.Security.IdentityModel.AspNetCore` 1.0.0. Both target .NET 10 only, and the complete 1.0.0 release (not a prerelease, C-20) goes to the private feed by 2026-11-10.

That release carries:
- the new identity;
- every change that would otherwise need a second major version;
- every direct dependency at its latest stable version, kept current automatically from then on;
- a written migration path for assetvault-io, the first known consumer.

Consumers get keys that leave process-wide crypto state alone, key-set exports that are public-only by default, and key validation that rejects mismatched key pairs. They also get strict, lossless encoders, with a documented cap and time budget on Base62. Argument errors name the argument, and key sizes and try-patterns follow .NET conventions. The work that is provably non-breaking follows in 1.x releases: further measured performance gains, the sample app's hosting and labels, the repository docs, and the remaining guardrails (API compatibility, coverage, formatting).

### 2.2 Requirements

_Each requirement traces to a goal from §1.3. `MUST` / `SHOULD` / `MAY`._

**Column key.**
- **Ships in** says which release the requirement may land in. Q-1 is resolved (GP-1, C-20), so the column is firm. §2.2.1 sets the rule behind it and justifies every `1.x` row.
  - `1.0.0` means it must be in the deadline release.
  - `1.x` means it may follow in a later 1.x release without breaking a consumer. It is still required before this spec is done.
  - `all` means it applies to every requirement.
- **Provisional on** names the §1 assumption or §1.6/§2.6 question the requirement's current wording depends on. If that answer changes, the requirement and its ACs are amended. They are appended, never renumbered.

| ID | Requirement | Priority | Ships in | Satisfies | Provisional on |
| --- | --- | --- | --- | --- | --- |
| R-1 | Both libraries target `net10.0` and nothing else. The test project and the sample app also target `net10.0`. | MUST | 1.0.0 | G-1 | — |
| R-2 | The solution builds and the full test suite passes on the .NET 10 SDK and runtime, locally and in CI on every OS in the CI matrix. No test from `555d31c` is deleted without a stated reason. | MUST | 1.0.0 | G-1 | — |
| R-3 | Version 1.0.0 of both packages is published to the ValhallaTech private feed no later than 2026-11-10. The feed is GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json` (D-2 decided, GP-10). A `net10.0` project can restore it from that feed and use it to sign and validate an EdDSA JWT. | MUST | 1.0.0 | G-1 | A-2 |
| R-4 | The package IDs are `ValhallaTech.Security.IdentityModel` and `ValhallaTech.Security.IdentityModel.AspNetCore`, and both are version 1.0.0. | MUST | 1.0.0 | G-2 | A-2 |
| R-5 | Package metadata identifies ValhallaTech's fork: authors, project URL, repository URL, release-notes URL and icon. The only place upstream may appear is the retained copyright attribution (R-7). The icon is the ValhallaTech company logo, a PNG of at least 128×128 that Fred supplies before release (Q-18, FC-6). A package with no icon fails. | MUST | 1.0.0 | G-2 | FC-6 (Fred supplies the logo) |
| R-6 | Every public type is in a `ValhallaTech.Security.IdentityModel` namespace, and each assembly's identity matches its package ID. Shipped library source and sample source contain no `ScottBrady` identifier outside attribution or modification notices. | MUST | 1.0.0 | G-2 | — |
| R-7 | The Apache-2.0 obligations are met. Each package carries the license text. The original copyright notice is retained. Every library source file derived from upstream carries a notice that it was modified. | MUST | 1.0.0 | G-2 (C-16) | — |
| R-8 | Vulnerability reports go to GitHub private vulnerability reporting on the canonical repository, `IronSloth/IdentityModel-Supa` (C-22), and that reporting is enabled. `SECURITY.md` names no upstream contact. Enabling reporting is a manual step only Fred can take (C-21). No agent can satisfy AC-20. | MUST | 1.0.0 | G-3 | C-21 (Fred's action) |
| R-9 | Constructing, converting or using an EdDSA key never writes process-wide cryptographic state. EdDSA JWTs still sign and validate through Microsoft.IdentityModel with no global registration by the app. | MUST | 1.0.0 | G-4 | — |
| R-10 | Converting a key to a JWK yields public material only, unless the developer uses an explicit opt-in to export the private key (shape is D-4). | MUST | 1.0.0 | G-5 | — |
| R-11 | Disposing an EdDSA key zeroes every buffer of private-key material that the library allocated. The library never modifies an array the caller supplied: assetvault-io passes its own array as `D`. Releasing a signature provider does not damage the key it was created from. | MUST | 1.0.0 | G-6 | — |
| R-12 | Base62 round-trips every byte sequence, including empty ones, all-zero ones, and ones with leading zero bytes. Decoding is canonical: every string `Decode` accepts re-encodes to itself. D-3 is resolved (GP-2): SimpleBase `Base62.Default` is the implementation. Its version follows R-17's latest-stable rule like every other dependency, not a pin (GP-10). It encodes each leading zero byte as `'0'`, and for inputs without leading zeros it matches today's output. | MUST | 1.0.0 | G-7 | — |
| R-13 | Malformed encoded input is rejected, never decoded, and never fails with an unrelated exception such as `NullReferenceException` or `IndexOutOfRangeException`.<br>**Base16:** any string with a character outside the alphabet, or with an odd length, throws `FormatException`, as .NET's own `Convert.FromHexString` does.<br>**Base62:** any string with a character outside the alphabet throws `FormatException`. The decided implementation, SimpleBase `Base62.Default` (D-3 resolved, GP-2; latest stable per GP-10), throws `ArgumentException`, and the library translates that into `FormatException` (Q-24, Fred, Gate 1).<br>Exceeding the Base62 cap is a bad argument, not bad content, so it stays `ArgumentException` (R-25).<br>Fred confirmed this split under Q-21: `FormatException` for bad encoded text, `CryptographicException` for invalid keys, `ArgumentException` for bad arguments. | MUST | 1.0.0 | G-7 | — |
| R-14 | Base16 round-trips every byte sequence, including the empty one. It keeps today's lowercase output and its acceptance of either case on input. | MUST | 1.0.0 | G-7 | — |
| R-15 | Key validation is the single gate. If `Validate()` passes, `EdDsa.Create` succeeds, the key can perform every operation its material allows without throwing, and a key holding both `D` and `X` verifies its own signatures. If validation fails, it throws `CryptographicException` with a message that says why: the key part and the required length, or that `D` and `X` do not belong to the same key pair (S11, GP-5). Which lengths are valid is D-5. Fred confirmed under Q-21 that `CryptographicException` for invalid keys overrides G-18's general rule. | MUST | 1.0.0 | G-8 | — |
| R-16 | At release, no project has a High or Critical advisory or a deprecated package, direct or transitive. The unmaintained analyzer is gone. Every direct dependency is under an OSI-approved license, which keeps test tooling free for commercial use (C-18). | MUST | 1.0.0 | G-9 | — |
| R-17 | At release, every direct dependency of every project is at its latest stable version. That covers the libraries, the tests and the sample, including the sample's vendored client-side libraries (A-13). Combined with R-16's license rule, this means the test assertion library cannot stay on FluentAssertions 6.12.2, because FluentAssertions' latest stable is under the Xceed license (C-18). D-1 must choose a library whose latest stable is OSI-licensed. | MUST | 1.0.0 | G-9 | A-13, A-14 |
| R-18 | No third-party client-side library vendored in the sample has a known advisory of any severity, or is past its maintainer's end of life, at release time. This is a floor only. R-56 (GP-8) adds the stricter rules: Bootstrap 5.x at its latest stable, and no jQuery anywhere. | MUST | 1.0.0 | G-9, G-10 | — |
| R-19 | In the sample, no key configured for token validation holds private material. PII logging and the developer exception page are active only in the Development environment. | MUST | 1.x | G-10 | — |
| R-20 | Both libraries build with active .NET code analysis, including the security rules, and with no project-level suppression of analysis or of missing-XML-doc warnings. A Release build reports 0 warnings. | MUST | 1.0.0 | G-11 | — |
| R-21 | Every workflow declares its token permissions explicitly. The build workflow's token has no write scope. | MUST | 1.x | G-12 | — |
| R-22 | A committed measurement report gives the baseline (`555d31c`) and the after-change figures for every path and input size in measurement set T (§2.3). Both sides are measured on the same machine and the same .NET 10 runtime. | MUST | 1.0.0 | G-13 | Q-9 |
| R-23 | Every path in T improves on its baseline, in time or allocation, at every input size. | MUST | 1.x | G-13 | Q-5 |
| R-24 | Base16 encode and decode time grows linearly with input length. | MUST | 1.x | G-14 | Q-9 |
| R-25 | Base62 bounds how long one call can hold a thread. The cap and SimpleBase are decided (GP-2, D-3 resolved). The values below were proposed by the PM (A-15) and **confirmed by Fred at Gate 1 (Q-23)**.<br>(a) It documents a maximum input of **1,024 bytes** for `Encode`. For `Decode` the maximum is the longest string SimpleBase's `Encode` can produce from 1,024 bytes: **1,376 characters**.<br>(b) It rejects longer input, and any string that would decode to more than 1,024 bytes, with `ArgumentException` naming `value`. `Decode` never returns more than 1,024 bytes. Whether the over-size check runs before or after SimpleBase decodes is an implementation choice. The 1,376-character cap already bounds decode time, so either order is acceptable as long as (c) and AC-70 hold (Gate 2).<br>(c) At the maximum, each `Encode` and `Decode` call has a mean time of at most **3 ms**, calibrated to the measuring machine (AC-70).<br>(d) At the maximum it is measurably faster than `555d31c`. | MUST | 1.0.0 | G-14 | Q-9 |
| R-26 | Reading key material from a stream succeeds whenever the stream eventually supplies the requested bytes, even in fragments. It fails only when the stream ends first. | MUST | 1.0.0 | G-15 | — |
| R-27 | One `EdDsa` instance stays safe for concurrent signing and verifying. | MUST | 1.0.0 | G-16 | — |
| R-28 | A single registration step turns on the extended password options, the extended validator, and the tag helper's rendering of those rules. Fred put it in 1.0.0 under Q-25 ("Helper in 1.0.0"). | MUST | 1.0.0 | G-17 | — |
| R-29 | Invalid `ExtendedPasswordOptions` are rejected with a message naming the offending setting, however the validator is registered (Q-17, Fred, Gate 1).<br>**Invalid** means `MaxLength` or `MaxConsecutiveChars` set to 0 or a negative value, or `MaxLength` < `RequiredLength`. `null` means no limit, for both.<br>**Through the single step (R-28, also 1.0.0):** the host fails at startup (AC-76).<br>**Through upstream's two-step pattern** (`options.Password = new ExtendedPasswordOptions { … }` plus ASP.NET Identity's own `AddPasswordValidator<…>()`): the library has no startup hook, so it rejects no later than its first use, either the first `ValidateAsync` or the first tag-helper render, by throwing instead of silently ignoring the setting (AC-142). Fred accepted first-use rejection on this path (Q-25). | MUST | 1.0.0 | G-17 | — |
| R-30 | Every public constructor and method rejects a null for a non-nullable reference parameter with `ArgumentNullException`, whose `ParamName` is that parameter. `Try`-prefixed members are exempt from that rule.<br>**The `Try`-member contract, stated once for both libraries:** a `Try` member given `null` for any reference parameter returns `false`, sets its `out` value to `null`, and never throws. This applies to `ExtendedJsonWebKeyConverter.TryConvertToEdDsaSecurityKey` (AC-82) and `StreamExtensions.TryRead` (AC-83). A numeric argument outside its valid range, such as a negative length, still throws `ArgumentOutOfRangeException` (R-31, AC-86), because that makes the call itself invalid rather than being data the member tries to process. | MUST | 1.0.0 | G-18 | — |
| R-31 | Public members that take offsets, lengths or argument arrays reject out-of-range or missing values with an `ArgumentException` (or subtype) whose `ParamName` names the argument. Both `Verify` overloads treat a zero-length message the same way. | MUST | 1.0.0 | G-18 | Q-10 |
| R-32 | A key built from private material only can verify signatures, both directly and through Microsoft.IdentityModel token validation. | MUST | 1.x | G-19 | — |
| R-33 | The try-pattern span `Sign` returns `false` with `bytesWritten == 0` when the destination is too small. It does not throw. | MUST | 1.0.0 | G-20 | — |
| R-34 | `KeySize` is reported in bits on both `EdDsa` and `EdDsaSecurityKey`, and `LegalKeySizes` is in bits and agrees with `KeySize`. Ed25519 is 256 and Ed448 is 456, which is the 57-byte encoding × 8 (Q-13, confirmed). | MUST | 1.0.0 | G-20 | — |
| R-35 | The key ID survives conversion to a JWK and back. | MUST | 1.0.0 | G-20 | — |
| R-36 | The six leftover ChaCha20 constants and `KeyTypes.Ecdh` are absent from the 1.0.0 public API (GP-4). Any public constant whose value is `"OKP"` has a name that matches its meaning (Octet Key Pair). | MUST | 1.0.0 | G-20 | — |
| R-37 | Every public member of both libraries carries nullability metadata. | MUST | 1.0.0 | G-21 | — |
| R-38 | Every public type and every public or protected member of both libraries has an XML doc summary, or inherits one. | MUST | 1.0.0 | G-21 | — |
| R-39 | The repository README and the package readmes describe only features that exist, and do not point consumers to upstream's packages, CI or issue tracker. | MUST | 1.x | G-22 | — |
| R-40 | Every C# example in the README and package readmes compiles against the released packages. | MUST | 1.x | G-22 | — |
| R-41 | Each package carries its own readme. | MUST | 1.x | G-22 | — |
| R-42 | `CHANGELOG.md` records every release, and its 1.0.0 entry lists every breaking change. | MUST | 1.0.0 | G-22 | — |
| R-43 | Every password input, including whitespace-only and empty ones, produces an `IdentityResult` from the extended validator and never an exception. | MUST | 1.x | G-23 | — |
| R-44 | The extended validator's default messages use the built-in ASP.NET Identity phrasing ("Passwords must …", ending with a period) and keep their error codes. The app can replace or translate them (mechanism is D-11). | MUST | 1.0.0 | G-24 | — |
| R-45 | The sample runs on .NET 10, uses minimal hosting, and gives every input on its password form an associated label. The supported-UI-framework part of G-25 is covered by R-56, which ships in 1.0.0. | MUST | 1.x | G-25 | — |
| R-46 | An automated API compatibility check against the published 1.0.0 baseline fails the build on any breaking change to the public API. It is active before the first release after 1.0.0. | MUST | 1.x | G-26 | — |
| R-47 | Every CI build measures line coverage for each packable library separately and fails if either is below 80%. | MUST | 1.x | G-27 | Q-6 |
| R-48 | From the 1.0.0 release onward, an automated tool (D-7) proposes an update whenever any direct dependency in R-17's inventory gains a newer stable version. That includes the sample's client-side libraries and the CI workflow actions. Advisories against dependencies reach the maintainers the same way. | MUST | 1.0.0 | G-28 | A-13 |
| R-49 | Code formatting is checked automatically in CI (tool is D-7). | SHOULD | 1.x | G-28 | — |
| R-50 | Work is test-first at two levels (C-13, GP-3). Each acceptance test that verifies a behavior change exists, and fails by assertion, at a commit before the one that implements the change. Each fix is also preceded by at least one unit test that fails by assertion at its introducing commit. | MUST | all | (C-13) | — |
| R-51 | The sample and the README password section both demonstrate the single registration step (R-28). | SHOULD | 1.x | G-17 | — |
| R-52 | The signing and verifying paths in T are also measured for Ed448. | MAY | 1.x | G-13 | — |
| R-53 | `EdDsa.ImportFromPem` and both `EdDsa.ImportFromEncryptedPem` overloads throw `NotSupportedException` (U13, GP-6). | MUST | 1.0.0 | G-20 | — |
| R-54 | A written migration guide covers every breaking change between upstream `ScottBrady.IdentityModel` 4.1.0 and 1.0.0, with what replaces each one. Applying that guide alone is enough for assetvault-io to build and pass its token tests on 1.0.0. | MUST | 1.0.0 | G-29 | — |
| R-55 | No path in measurement set T regresses, in time or allocation, in any release, 1.0.0 included. | MUST | 1.0.0 | G-13 | — |
| R-56 | GP-8 sets the sample's front-end rules:<br>(a) The sample is kept. It stays a Razor app, because it demonstrates the `newpassword` tag helper.<br>(b) Its UI framework is Bootstrap **5.x** at its latest stable release on the release date: 5.3.8 on npm as of 2026-10-25, per the /spec coordinator. Bootstrap 4.x, including 4.6.2, is not acceptable, and neither is a 6.x prerelease.<br>(c) No jQuery appears in any form, anywhere in the repository. That means no jQuery file, no reference to it from any view, layout or manifest, and no front-end library that depends on jQuery. Fred, verbatim: "jQuery should never be used in any of our work."<br>This supersedes the parts of R-18 that asked only for "no advisories". | MUST | 1.0.0 | G-9, G-10, G-25 (GP-8) | — |

R-50 traces to a hard constraint (C-13), not a goal. It is listed so that the Agent Evaluator can check the test-first discipline from the commit history, not only from the plan.

R-55 is split out of R-23. 1.0.0 rewrites the code of every path in T for correctness reasons (R-12, R-13, R-25, R-31, R-33, R-44), so 1.0.0 is where a regression could enter. The improvement half (R-23) is non-breaking, so it may follow in 1.x.

#### 2.2.1 Release split (firm: GP-1, C-20)

**The rule.** C-20 requires the complete 1.0.0 by 2026-11-10, carrying A-1's scope, and the Q-1 resolution lets non-breaking work follow in 1.x. A requirement is therefore `1.0.0` if any of these holds:
- **(a) It would break a consumer after 1.0.0,** and so need a second major version (C-5). Breaking means any of:
  - removing or renaming public API;
  - changing a public signature, a returned value or an output format;
  - throwing a different exception type for an input that already threw;
  - throwing where the code used to succeed or return;
  - raising a dependency's minimum major version;
  - adding warnings that a consumer's warnings-as-errors build would turn into errors.
- **(b) A goal ties it to "release" or "after release".** G-1, G-9, G-28 and G-29 do this.
- **(c) It guards code that 1.0.0 rewrites anyway.** Deferring it would ship that rewrite unchecked.

Everything else may follow in 1.x. Following .NET's breaking-change conventions, making a call succeed where it used to throw is **not** breaking.

**`1.0.0` (deadline release), 40 requirements:** R-1 to R-18, R-20, R-22, R-25 to R-31, R-33 to R-38, R-42, R-44, R-48, R-53 to R-56, plus R-50 throughout. These moved into 1.0.0 after the first pass:
- **R-29 (invalid password options rejected), by (a). Moved at Gate 1.** Fred's Q-17 answer makes 0 invalid and requires rejection however the validator is registered. A configuration that starts today would then fail, which is breaking.
- **R-28 (single-step setup helper). Moved at Gate 1 by Fred's Q-25 answer ("Helper in 1.0.0").** It is additive, so the rule alone would allow 1.x. It is in 1.0.0 so that R-29's startup rejection (AC-76) ships at the deadline, not only first-use rejection.
- **R-51 stays `1.x` (SHOULD).** It only makes the sample and README demonstrate the helper. It changes no package behavior, and it still makes sense in a later release.
- **R-22 (measurement report), by (c).** R-25(c)–(d) needs the measurement harness and the `555d31c` baseline in 1.0.0 regardless, and R-55 needs the full set T.
- **R-26 (fragmented stream reads), by (a).** `TryRead` returning `true` where it returned `false` is a changed return value. R-31 rewrites the same method for 1.0.0 anyway.
- **R-35 (key ID in the JWK), by (a).** The converter's output changes, and R-10 rewrites the converter for 1.0.0 anyway.
- **R-42 (changelog), by (b).** G-29's migration guide (R-54) depends on an authoritative list of breaking changes at 1.0.0.
- **R-48 (automatic dependency updates), by (b).** G-28 says "after release", so the tool must be active from the release.
- **R-53, R-54, R-55, R-56** are new.
  - R-53 changes an exception type, so (a).
  - R-54 is tied to release by G-29, so (b).
  - R-55 is the guard described under (c).
  - R-56 is part of G-9's "latest stable at release" for the sample's front-end, so (b).

**`1.x`, 15 requirements, each with the reason it does not break C-20 or C-5:**

| Req | Why it may follow 1.0.0 |
| --- | --- |
| R-19 | Sample-only (not package content). No consumer of the packages can break. |
| R-21 | CI token scope. Not package content. |
| R-23 | Additional speed-ups do not change results. R-55 already stops 1.0.0 from regressing. |
| R-24 | Base16 linear scaling changes no result. In practice the 1.0.0 rewrite for R-13/R-14 will likely meet it early. |
| R-32 | Throw → success. Not breaking under .NET rules. assetvault-io verifies with public-only keys, so it does not need R-32 at 1.0.0. |
| R-39, R-40 | Repository README content. Not package content. |
| R-41 | Adding a package readme in a later 1.x changes no API or behavior. |
| R-43 | Throw → validation result. Not breaking under .NET rules. |
| R-45 | Sample hosting style and labels. In 1.0.0, R-1, R-18 and R-56 already make the sample build on .NET 10, on Bootstrap 5.x, with no jQuery. |
| R-46 | Needs the published 1.0.0 as its baseline, so it cannot exist before 1.0.0. It is required before the first later release. |
| R-47 | Coverage measurement and the CI floor change nothing in the packages. C-12 defines when the spec is done, not what 1.0.0 contains. |
| R-49, R-51, R-52 | SHOULD/MAY. Repository and sample only. |

**Feasibility.** The 1.0.0 set has grown from 29 to 40 requirements. It is due 2026-11-10, 16 days from the brief. R-28 is in by Fred's choice; every other requirement is there because C-20 and C-5 require it, so the PRD has nothing left to move out. R-28 and R-29 add little work: one registration helper and one options validator. The set is feasible only if all of these hold:

| ID | Condition |
| --- | --- |
| FC-1 | The IronSloth GitHub Packages NuGet feed (`https://nuget.pkg.github.com/IronSloth/index.json`, D-2 decided, GP-10) accepts a push of both 1.0.0 packages from a credential with `write:packages`, and a consumer can restore them with a `read:packages` credential. Both must be in place, and a trial push and restore done, well before 2026-11-10. Credentials are Fred's to issue. |
| FC-2 | Fred enables private vulnerability reporting (C-21) before release. Otherwise AC-20 fails. |
| FC-3 | A-14 holds. Every latest-stable direct dependency, including the Microsoft.IdentityModel 7 → 8 major bump and the test-tooling majors, works on .NET 10 and with the others. |
| FC-4 | The test-assertion library swap (D-1) is mechanical. |
| FC-5 | **Holds, as measured.** SimpleBase meets the 3 ms budget at 1,024 bytes (AC-70). On Fred's machine it averages 1.577 ms to encode and 0.809 ms to decode, against 3.555 ms for today's encode, and the output is 1,375 characters *(scratch probe on Fred's machine, 2,000 calls, 2026-10-25, relayed by the /spec coordinator)*. AC-70 and AC-127 still have to be run on the release candidate; this measurement predicts their result but does not replace them. |
| FC-6 | Fred supplies the ValhallaTech logo as a PNG of at least 128×128 before release (Q-18). Otherwise AC-12 fails. No agent can produce this asset. |

If any of these fails, C-3, C-5 and C-20 cannot all be met, and Fred must choose which one gives. The PRD does not choose.

### 2.3 Acceptance criteria

_Ground truth for evaluating execution agents. Every AC is written to be assertable._

**Conventions for every AC below.**
- **Names.** Types and members are named as at `555d31c`, inside the renamed namespaces. If the Architect renames a type or member, §3.3 must map the old name to the new one, and the AC follows the mapping.
- **Exceptions.** "Throws `X`" means `X` or a subtype. `ParamName` checks are exact and ordinal.
- **Theories.** "Both curves" means the AC is parameterized over `Ed25519` and `Ed448`, and each case must pass.
- **Seeds.** "Fixed seed" means a deterministic random generator whose seed is recorded in the test.
- **Shared state.** ACs that touch process-wide state (AC-21 to AC-25, AC-53) need test isolation. How to isolate them is the Architect's call.
- **Network-dependent ACs.** AC-133 and AC-137 to AC-140 and AC-143 need `registry.npmjs.org`; AC-50 needs the GitHub Advisory Database. Without network access, each of these **fails** with an explicit "registry unreachable" reason naming the URL that failed. It is never passed, skipped or marked inconclusive (AC-138 gives the full rule).
- **Measurement ACs** (AC-59 to AC-70, AC-123, AC-127) run in a separate measurement process against built assemblies on one named machine. They are classified `integration` because the template has no `measurement` level. They are evidence runs, not per-commit CI gates.
- **Interfaces not designed yet.** Some ACs refer to an interface that is still to be designed ("the private-export opt-in", "the single registration step", "the message replacement mechanism"). They mean whatever §3.3 defines for D-4, R-28 and D-11. The assertion is on behavior only.

#### AC-1 — Packages carry only a .NET 10 build

- **Verifies:** R-1
- **Given** the release commit
- **When** `dotnet pack -c Release` produces both `.nupkg` files
- **Then** each package contains exactly one target-framework folder, `net10.0`
- **Assertion:** the set of distinct `lib/<tfm>/` folders in each `.nupkg` == {`lib/net10.0/`}. No entry exists for `net6.0`, `net8.0` or `net9.0`.
- **Executable as:** build

#### AC-2 — Tests and sample target .NET 10

- **Verifies:** R-1
- **Given** the release commit
- **When** the evaluated target frameworks of the test project and the sample project are read
- **Then** each is `net10.0` only
- **Assertion:** `dotnet msbuild <project> -getProperty:TargetFramework -getProperty:TargetFrameworks` returns `net10.0` as the only framework for both projects.
- **Executable as:** build

#### AC-3 — Solution builds on the .NET 10 SDK

- **Verifies:** R-2
- **Given** a clean clone at the release commit and a 10.0.x SDK as the only SDK selected
- **When** `dotnet build -c Release` runs at the repository root
- **Then** the build succeeds
- **Assertion:** exit code == 0.
- **Executable as:** build

#### AC-4 — Full suite passes on the .NET 10 runtime

- **Verifies:** R-2
- **Given** the build from AC-3
- **When** `dotnet test -c Release` runs with a results logger
- **Then** every test passes on runtime 10.0.x, with none skipped
- **Assertion:** exit code == 0. In the test results, failed == 0, skipped == 0, and the runtime recorded for the run is 10.0.x.
- **Executable as:** build

#### AC-5 — No test silently dropped

- **Verifies:** R-2
- **Given** the fully qualified test method names at `555d31c` and at the release commit, with the namespace renamed per C-8
- **When** the two name sets are compared
- **Then** every test that is missing at release is listed in §3 with the behavior it covered and the reason it was removed
- **Assertion:** (names at `555d31c` − names at release) ⊆ the removed-test list in §3. Count of unlisted removals == 0.
- **Executable as:** lint

#### AC-6 — CI installs only the .NET 10 SDK

- **Verifies:** R-2
- **Given** `.github/workflows/*.yml` at the release commit
- **When** every `dotnet-version` input of every setup-dotnet step is read
- **Then** each one selects a 10.0 SDK
- **Assertion:** every value matches `^10\.0(\.x|\.\*|\.\d+)?$`. The count of values selecting 6.x, 8.x or 9.x == 0.
- **Executable as:** lint

#### AC-7 — CI is green on .NET 10 across the matrix

- **Verifies:** R-2
- **Given** the CI workflow run for the release commit
- **When** its jobs complete
- **Then** every matrix job (every OS kept under D-10) succeeds
- **Assertion:** `gh run view <run-id> --json jobs` shows every job with `conclusion == "success"`, and the test step log of each job shows runtime 10.0.x.
- **Executable as:** integration

#### AC-8 — 1.0.0 is on the private feed by the deadline

- **Verifies:** R-3
- **Given** the ValhallaTech private feed: GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json` (D-2 decided, GP-10), queried with a `read:packages` credential
- **When** the feed's package metadata is queried for both package IDs at version 1.0.0
- **Then** both exist, each published no later than the end of 2026-11-10, US Eastern time
- **Assertion:** for each ID, the metadata query returns version `1.0.0` exactly, with a published timestamp ≤ `2026-11-10T23:59:59-05:00`. A prerelease such as `1.0.0-rc.1` does not satisfy this AC (C-20).
- **Executable as:** integration

#### AC-9 — A .NET 10 consumer can use the feed packages

- **Verifies:** R-3
- **Given** a new `net10.0` console project that references both packages at 1.0.0 as package references, with no project or local-folder references, so the ValhallaTech packages resolve from the IronSloth GitHub Packages feed (GP-10) using a `read:packages` credential
- **When** it is restored, built and run. The program creates an Ed25519 `EdDsaSecurityKey`, issues a JWT through `JsonWebTokenHandler.CreateToken` with `SigningCredentials(key, "EdDSA")`, and validates it with `ValidateTokenAsync` using that key.
- **Then** the token validates
- **Assertion:** restore, build and run each exit with code 0, and the program prints `TokenValidationResult.IsValid == True`.
- **Executable as:** integration

#### AC-10 — Package IDs and version

- **Verifies:** R-4
- **Given** the two `.nupkg` files from AC-1
- **When** each `.nuspec` is read
- **Then** the IDs and versions are the new ones
- **Assertion:** the `<id>` values are exactly {`ValhallaTech.Security.IdentityModel`, `ValhallaTech.Security.IdentityModel.AspNetCore`}, and `<version>` == `1.0.0` for both.
- **Executable as:** build

#### AC-11 — Package metadata identifies ValhallaTech

- **Verifies:** R-5
- **Given** each `.nuspec` from AC-10
- **When** its metadata elements are read
- **Then** they identify the fork and do not mention upstream outside the copyright
- **Assertion:**
  - `<authors>` contains `ValhallaTech`.
  - `<projectUrl>` == `https://github.com/IronSloth/IdentityModel-Supa` (C-22).
  - `<repository url>` == the same URL.
  - No element other than `<copyright>` contains `scottbrady` or `Scott Brady` (case-insensitive). This covers `<releaseNotes>` and `<description>`.
- **Executable as:** build

#### AC-12 — Package icon is the ValhallaTech logo

- **Verifies:** R-5
- **Given** each `.nupkg`, the ValhallaTech logo PNG Fred supplies (FC-6) at the repository path §3 names, and the file `icon.png` at `555d31c`, which is upstream's "SB" logo
- **When** each package's icon entry is read and hashed
- **Then** every package carries the ValhallaTech logo
- **Assertion:**
  - The nuspec has an `<icon>` element naming a file present in the package.
  - That file is a PNG: its first 8 bytes are `89 50 4E 47 0D 0A 1A 0A`.
  - Its IHDR width and height are each ≥ 128.
  - SHA-256(packaged icon) == SHA-256(the supplied logo), and ≠ SHA-256(`icon.png` at `555d31c`).
  - A package with no icon fails.
- **Executable as:** build
- **Depends on:** Fred supplying the logo file before release (FC-6).

#### AC-13 — Public namespaces are renamed

- **Verifies:** R-6
- **Given** both built library assemblies
- **When** every exported type is enumerated by reflection
- **Then** every namespace is under the new prefix
- **Assertion:** for every exported type, `Namespace.StartsWith("ValhallaTech.Security.IdentityModel", Ordinal)` is true. The count of exported types whose namespace starts with `ScottBrady` == 0.
- **Executable as:** unit

#### AC-14 — Assembly identity matches the package

- **Verifies:** R-6
- **Given** both built library assemblies
- **When** their `AssemblyName` and assembly-level attributes are read
- **Then** each assembly is named after its package and carries no upstream identifier outside its copyright
- **Assertion:** `GetName().Name` == the package ID from AC-10. No assembly-level attribute's string arguments contain `ScottBrady` (ordinal, case-insensitive), except `AssemblyCopyrightAttribute`. This includes `InternalsVisibleTo`.
- **Executable as:** unit

#### AC-15 — No residual upstream identifiers in shipped or copied source

- **Verifies:** R-6
- **Given** the release commit
- **When** `rg -i -uu "scottbrady"` runs over `src/**/*.cs`, `samples/**/*.cs` and `samples/**/*.cshtml`, excluding `bin/` and `obj/`
- **Then** every match is part of an attribution or modification notice
- **Assertion:** every matching line is a line of the notice text fixed in §3 (AC-18). The count of other matches == 0.
- **Executable as:** lint

#### AC-16 — License text ships in each package

- **Verifies:** R-7
- **Given** each `.nupkg`
- **When** its entries are listed
- **Then** it contains the Apache-2.0 license text
- **Assertion:** the package contains a license file whose content equals the repository's `LICENSE` byte for byte, ignoring line-ending differences.
- **Executable as:** build

#### AC-17 — Original copyright is retained alongside ValhallaTech's

- **Verifies:** R-7
- **Given** each `.nuspec` and package
- **When** the copyright and attribution text is read
- **Then** both ValhallaTech and the original author are credited
- **Assertion:** `<copyright>` contains `ValhallaTech`. The string `Copyright 2022 (c) Scott Brady` appears in `<copyright>` or in a NOTICE file inside the package.
- **Executable as:** build

#### AC-18 — Modified upstream files carry a modification notice

- **Verifies:** R-7
- **Given** the library source files at the release commit that derive from a file present at `555d31c`. The C-8 namespace rename modifies every one of them.
- **When** each is searched for the modification-notice string that §3 fixes
- **Then** every derived file carries it
- **Assertion:** for each derived `.cs` file under the library source trees, a search for the notice string finds a match. Count of misses == 0.
- **Executable as:** lint

#### AC-19 — SECURITY.md routes to private reporting

- **Verifies:** R-8
- **Given** `SECURITY.md` at the release commit
- **When** its content is searched
- **Then** it links to private reporting on the canonical repository and names no upstream contact
- **Assertion:** the file contains `https://github.com/IronSloth/IdentityModel-Supa/security/advisories/new` (C-22). It has 0 matches for `scottbrady91` and 0 matches for an email-address pattern.
- **Executable as:** lint

#### AC-20 — Private vulnerability reporting is enabled

- **Verifies:** R-8
- **Given** the canonical repository (C-22), after Fred has enabled private vulnerability reporting in its settings
- **When** `GET /repos/IronSloth/IdentityModel-Supa/private-vulnerability-reporting` is called with an authorized token
- **Then** reporting is enabled
- **Assertion:** HTTP 200 and `enabled == true`. On 2026-10-25 the endpoint returned `{"enabled":false}` (C-21).
- **Executable as:** integration
- **Depends on:** a manual repository-settings change that only Fred can make (C-21). The check itself is automated, but no execution agent can make it pass. If Fred has not acted, this AC fails, and that failure is reported, not worked around.

#### AC-21 — Constructing a key leaves the global provider untouched

- **Verifies:** R-9
- **Given** `CryptoProviderFactory.Default.CustomCryptoProvider` is set to a sentinel `ICryptoProvider` S, whose `IsSupportedAlgorithm` always returns `false`
- **When** `new EdDsaSecurityKey(EdDsa.Create("Ed25519"))` is constructed
- **Then** the global custom provider is still S
- **Assertion:** `ReferenceEquals(CryptoProviderFactory.Default.CustomCryptoProvider, S)` is `true`.
- **Executable as:** unit

#### AC-22 — Using a key leaves the global provider untouched

- **Verifies:** R-9
- **Given** sentinel S registered as in AC-21, and an Ed25519 `EdDsaSecurityKey`
- **When** a JWT is created with `JsonWebTokenHandler.CreateToken` using `SigningCredentials(key, "EdDSA")`, then validated with `ValidateTokenAsync` using the key as `IssuerSigningKey`, and the key is converted with `ConvertFromEdDsaSecurityKey`
- **Then** the global custom provider is still S
- **Assertion:** `ReferenceEquals(CryptoProviderFactory.Default.CustomCryptoProvider, S)` is `true` after all three operations.
- **Executable as:** integration

#### AC-23 — An unset global provider stays unset

- **Verifies:** R-9
- **Given** `CryptoProviderFactory.Default.CustomCryptoProvider == null`
- **When** an `EdDsaSecurityKey` is constructed for each curve
- **Then** the global custom provider is still unset
- **Assertion:** `CryptoProviderFactory.Default.CustomCryptoProvider == null`.
- **Executable as:** unit

#### AC-24 — EdDSA JWTs work with no global registration

- **Verifies:** R-9
- **Given** `CryptoProviderFactory.Default.CustomCryptoProvider == null`, with no EdDSA setup beyond constructing the key (both curves)
- **When** a JWT is created and validated as in AC-22
- **Then** the token validates
- **Assertion:** `TokenValidationResult.IsValid == true`. The header `alg` == `"EdDSA"`.
- **Executable as:** integration

#### AC-25 — Concurrent key construction shares no state

- **Verifies:** R-9
- **Given** sentinel S registered as in AC-21
- **When** 1,000 `EdDsaSecurityKey` instances (Ed25519) are constructed concurrently on at least 8 threads, and each signs and verifies a distinct 32-byte message through `key.CryptoProviderFactory.CreateForSigning` and `CreateForVerifying`
- **Then** the global provider is unchanged and every key works
- **Assertion:** `ReferenceEquals(CryptoProviderFactory.Default.CustomCryptoProvider, S)` is `true`. All 1,000 verifications return `true`. 0 exceptions.
- **Executable as:** unit

#### AC-26 — Default JWK conversion excludes the private key

- **Verifies:** R-10
- **Given** an `EdDsaSecurityKey` over `EdDsa.Create(curve)`, which holds a private key (both curves)
- **When** `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKey(key)` is called with no private-export opt-in
- **Then** the JWK carries no private material
- **Assertion:** `jwk.D == null`. `jwk.X` == `Base64UrlEncoder.Encode(<public key bytes>)`.
- **Executable as:** unit

#### AC-27 — A default JWK converts back to a public-only key

- **Verifies:** R-10
- **Given** the JWK from AC-26
- **When** `TryConvertToEdDsaSecurityKey(jwk, out var k)` is called
- **Then** it succeeds with a key that has no private material
- **Assertion:** the call returns `true`, and `k.PrivateKeyStatus == PrivateKeyStatus.DoesNotExist`.
- **Executable as:** unit

#### AC-28 — The explicit opt-in exports the private key

- **Verifies:** R-10
- **Given** the key from AC-26
- **When** it is converted through the private-export opt-in (D-4)
- **Then** the JWK carries the private key
- **Assertion:** `jwk.D` == `Base64UrlEncoder.Encode(<private key bytes>)`.
- **Executable as:** unit

#### AC-29 — A public-only JWK validates tokens signed by the private key

- **Verifies:** R-10
- **Given** an Ed25519 key holding a private key, and its default (public-only) JWK from AC-26
- **When** a JWT signed with the key is validated with `IssuerSigningKey = jwk`
- **Then** the token validates
- **Assertion:** `TokenValidationResult.IsValid == true`.
- **Executable as:** integration

#### AC-30 — Disposing a generated key zeroes its private-key buffers

- **Verifies:** R-11
- **Given** `e = EdDsa.Create(curve)` (both curves). Before disposal, the test collects every `byte[]` reachable by reflection from `e` whose content equals the private key, skipping objects whose type is defined in the BouncyCastle assembly (NG-8).
- **When** `e.Dispose()` is called
- **Then** each collected buffer is zeroed
- **Assertion:** at least one buffer was collected, and every byte of every collected buffer == `0x00`.
- **Executable as:** unit

#### AC-31 — Disposing a parameter-built key zeroes the library's copies

- **Verifies:** R-11
- **Given** `e = EdDsa.Create(new EdDsaParameters(curve) { D = d, X = x })` (both curves), with buffers collected as in AC-30 except the caller's own array `d`, matched by `ReferenceEquals`
- **When** `e.Dispose()` is called
- **Then** each collected buffer is zeroed
- **Assertion:** every byte of every collected buffer == `0x00`. AC-135 checks that the caller's array `d` is left unchanged.
- **Executable as:** unit

#### AC-32 — Releasing a signature provider does not damage the key

- **Verifies:** R-11
- **Given** an Ed25519 `EdDsaSecurityKey` and `p = key.CryptoProviderFactory.CreateForSigning(key, "EdDSA")`
- **When** `key.CryptoProviderFactory.ReleaseSignatureProvider(p)` is called, then `p.Dispose()`, and then a new provider signs a 32-byte message
- **Then** the new signature verifies
- **Assertion:** `key.EdDsa.Verify(message, signature) == true`.
- **Executable as:** unit

#### AC-33 — Base62 keeps leading zero bytes

- **Verifies:** R-12
- **Given** the input `[0x00, 0x00, 0x01]`
- **When** `Base62.Decode(Base62.Encode(input))` is called
- **Then** the original bytes come back
- **Assertion:** the result sequence-equals `[0x00, 0x00, 0x01]`.
- **Executable as:** unit

#### AC-34 — Base62 round-trips every byte sequence

- **Verifies:** R-12
- **Given** the empty array, all-zero arrays of lengths 1 to 8, and 1,000 random arrays of lengths 0 to 256 (fixed seed, at least 25% starting with one or more `0x00` bytes)
- **When** each array `b` is round-tripped
- **Then** every array comes back unchanged
- **Assertion:** for every `b`, `Base62.Decode(Base62.Encode(b))` sequence-equals `b`.
- **Executable as:** unit

#### AC-35 — Base62 decoding is canonical

- **Verifies:** R-12
- **Given** 1,000 random strings over `Base62.CharacterSet`, lengths 0 to 64 (fixed seed)
- **When** each string `s` is decoded and re-encoded
- **Then** every string `Decode` accepts re-encodes to itself, so no two accepted strings decode to the same bytes
- **Assertion:** for every `s` where `Base62.Decode(s)` does not throw, `Base62.Encode(Base62.Decode(s)) == s` (ordinal).
- **Executable as:** unit

#### AC-36 — Base62 rejects characters outside its alphabet

- **Verifies:** R-13
- **Given** the inputs `"!!"`, `"a b"`, `"abc-"`, `"é"`, `"+/"`
- **When** `Base62.Decode(input)` is called for each
- **Then** each is rejected as malformed
- **Assertion:** each call throws `FormatException` (Q-24, Fred, Gate 1). SimpleBase's own `ArgumentException` escaping untranslated fails this AC, as does any other exception or a return value. At baseline, `"!!"` returns `3EC1`.
- **Executable as:** unit

#### AC-37 — Base16 rejects odd-length input

- **Verifies:** R-13
- **Given** the input `"abc"`
- **When** `Base16.Decode("abc")` is called
- **Then** it is rejected as malformed
- **Assertion:** throws `FormatException`. At baseline this throws `ArgumentOutOfRangeException`.
- **Executable as:** unit

#### AC-38 — Base16 rejects characters outside its alphabet

- **Verifies:** R-13
- **Given** the inputs `"zz"`, `"0g"`, `"  "` (two spaces)
- **When** `Base16.Decode(input)` is called for each
- **Then** each is rejected as malformed
- **Assertion:** each call throws `FormatException`.
- **Executable as:** unit

#### AC-39 — Base16 round-trips every byte sequence, including empty

- **Verifies:** R-14
- **Given** the empty array and 1,000 random arrays of lengths 0 to 256 (fixed seed)
- **When** each array `b` is round-tripped
- **Then** every array comes back unchanged, and none throws
- **Assertion:** for every `b`, `Base16.Decode(Base16.Encode(b))` sequence-equals `b`. The empty case returns a zero-length array; at baseline `Decode("")` throws `ArgumentNullException`.
- **Executable as:** unit

#### AC-40 — Base16 output stays lowercase

- **Verifies:** R-14
- **Given** the input `[0xAB, 0x01, 0xFF]`
- **When** `Base16.Encode(input)` is called
- **Then** the output is lowercase hex
- **Assertion:** returns `"ab01ff"` (ordinal).
- **Executable as:** unit

#### AC-41 — Base16 still accepts uppercase input

- **Verifies:** R-14
- **Given** the input `"AB01FF"`
- **When** `Base16.Decode("AB01FF")` is called
- **Then** the matching bytes come back
- **Assertion:** the result sequence-equals `[0xAB, 0x01, 0xFF]`.
- **Executable as:** unit

#### AC-42 — Passing validation always yields a usable key

- **Verifies:** R-15
- **Given** every combination over:
  - curve ∈ {Ed25519, Ed448};
  - D ∈ {null} ∪ random bytes of length L;
  - X ∈ {null} ∪ random bytes of length L ∪ {a real public key from an unrelated freshly generated pair on that curve} ∪ {the public key that belongs to D, whenever D is a valid private key};
  - L ∈ {0, 1, 31, 32, 33, 56, 57, 58, 63, 64, 65, 113, 114, 115, 128} (fixed seed).
- **When** `p.Validate()` is called, and if it returns, `EdDsa.Create(p)` is called
- **Then** every parameter set that passes validation produces a key that works
- **Assertion:** for every `p` where `Validate()` does not throw:
  - `EdDsa.Create(p)` does not throw.
  - If `D != null`, `Sign(new byte[32])` returns a non-empty array without throwing.
  - If `D != null` and `X != null`, `Verify(m, Sign(m)) == true` for a 32-byte `m`.
  - If `D == null`, `Verify(new byte[32], new byte[sigLen])` returns a `bool` without throwing, where `sigLen` = 64 for Ed25519 and 114 for Ed448.

  The combination of D with its own public key must pass `Validate()` and so is always exercised.
- **Executable as:** unit

#### AC-43 — Validation failures say why

- **Verifies:** R-15
- **Given** an Ed25519 parameter set with a 31-byte `D`, and an Ed448 parameter set with a 56-byte `X`
- **When** `Validate()` is called on each
- **Then** each is rejected with a message naming the key part and the required length
- **Assertion:**
  - Ed25519 case: throws `CryptographicException`. The message contains `32` and contains `private` (case-insensitive).
  - Ed448 case: throws `CryptographicException`. The message contains `57` and contains `public` (case-insensitive).
- **Executable as:** unit

#### AC-44 — The S5 lengths are rejected or fully supported

- **Verifies:** R-15
- **Given** an Ed25519 `D` of 64 bytes and an Ed448 `D` of 114 bytes, each built as the private key followed by its public key
- **When** `Validate()` is called, and if it returns, `EdDsa.Create(p)` is called and the key signs a 32-byte message
- **Then** the key is rejected at validation, or it is fully usable, as decided under D-5
- **Assertion:** either `Validate()` throws `CryptographicException`, or `Create` and `Sign` succeed and the signature verifies with the trailing public-key half. At baseline `Validate()` passes and `Create` throws.
- **Executable as:** unit

#### AC-45 — No High or Critical advisories

- **Verifies:** R-16
- **Given** every project at the release commit (both libraries, the tests, the sample)
- **When** the NuGet advisory audit runs, including transitive packages (for example `dotnet list package --vulnerable --include-transitive`)
- **Then** nothing High or Critical is reported
- **Assertion:** the count of reported advisories with severity High or Critical == 0. At baseline this count is 2: `System.Net.Http` 4.3.0 and `System.Text.RegularExpressions` 4.3.0.
- **Executable as:** build

#### AC-46 — No deprecated packages

- **Verifies:** R-16
- **Given** every project at the release commit
- **When** `dotnet list package --deprecated --include-transitive` runs
- **Then** no deprecated package is reported
- **Assertion:** the count of deprecated packages == 0 for every project.
- **Executable as:** build

#### AC-47 — The unmaintained analyzer is gone

- **Verifies:** R-16
- **Given** every project file and shared build file at the release commit
- **When** they are searched for `SecurityCodeScan`
- **Then** nothing references it
- **Assertion:** `rg -uu "SecurityCodeScan" -g "*.csproj" -g "*.props" -g "*.targets"` finds 0 matches.
- **Executable as:** lint

#### AC-48 — Every direct dependency is under an OSI-approved license

- **Verifies:** R-16
- **Given** every direct `PackageReference` across all projects, resolved to its restored version
- **When** each package's nuspec license is read
- **Then** each is an SPDX expression made only of OSI-approved licenses. OSI approval rules out field-of-use restrictions, so the Xceed license on FluentAssertions 8.x and later fails this check (C-18).
- **Assertion:** for every direct dependency, `<license type="expression">` exists and every SPDX identifier in it is on the OSI-approved list. Count of failures == 0.
- **Executable as:** lint

#### AC-49 — Every direct NuGet dependency is at its latest stable version

- **Verifies:** R-17
- **Given** the release commit and the release date, and every project: both libraries, the tests and the sample
- **When** `dotnet list package --outdated` runs for each project, stable versions only, direct references only (A-13)
- **Then** no direct NuGet dependency has a newer stable version
- **Assertion:** the count of outdated direct package references == 0 across all projects. There is no recorded-gap escape: G-9 gives no exception rule (A-14). At baseline this includes Microsoft.IdentityModel.Tokens 7.7.1 and BouncyCastle.Cryptography 2.4.0.
- **Executable as:** lint

#### AC-50 — Sample client-side libraries have no advisories

- **Verifies:** R-18
- **Given** every third-party client-side library file vendored under the sample's web root, each identified by name and version
- **When** each name and version is looked up in the GitHub Advisory Database (npm ecosystem) on the release date
- **Then** none has an advisory
- **Assertion:** the count of advisories of any severity affecting a vendored library version == 0. At baseline, jQuery 3.3.1 fails this check: every one of its advisories is rated medium, for example GHSA-gxr4-xjj5-5px2, GHSA-jpcq-cgw6-v4j6 and GHSA-6c3j-c64m-qhgq *(verified 2026-10-25 by the /spec coordinator via `gh api /advisories?affects=jquery@3.3.1`)*. A High-or-Critical threshold would miss them, which is why R-18 uses any severity.
- **Executable as:** lint

#### AC-51 — Sample client-side libraries are supported

- **Verifies:** R-18
- **Given** the same libraries as AC-50
- **When** each major version is checked against its maintainer's published support or end-of-life status on the release date
- **Then** none is past end of life
- **Assertion:** the count of vendored libraries whose major version is end of life == 0. At baseline, Bootstrap 4.3.1 fails this check.
- **Executable as:** lint

#### AC-52 — The sample's validation keys are public-only

- **Verifies:** R-19
- **Given** the sample hosted in-process on `net10.0`
- **When** every JWT bearer scheme the sample registers is resolved through `IOptionsMonitor<JwtBearerOptions>.Get(scheme)`, and each `TokenValidationParameters.IssuerSigningKey` (or entry of `IssuerSigningKeys`) is inspected
- **Then** no validation key holds private material
- **Assertion:** every such key that is an `EdDsaSecurityKey` has `PrivateKeyStatus == PrivateKeyStatus.DoesNotExist`.
- **Executable as:** integration

#### AC-53 — PII logging is off outside Development

- **Verifies:** R-19
- **Given** the sample hosted in-process with environment `Production`
- **When** the host has started
- **Then** PII logging is off
- **Assertion:** `IdentityModelEventSource.ShowPII == false`.
- **Executable as:** integration

#### AC-54 — No developer exception page outside Development

- **Verifies:** R-19
- **Given** the sample hosted in-process with environment `Production`, and a request that raises an unhandled exception with the message `"ac54-probe"` (the test host provokes it)
- **When** the response is read
- **Then** it does not expose exception detail
- **Assertion:** status == 500. The body contains neither `ac54-probe` nor `Stack`.
- **Executable as:** integration

#### AC-55 — Security analysis is active

- **Verifies:** R-20
- **Given** a probe copy of each library with one added line that breaks a .NET security analyzer rule, for example `_ = System.Security.Cryptography.MD5.Create();` (CA5351)
- **When** the probe is built with `-c Release`
- **Then** the analyzer reports the rule
- **Assertion:** the build output contains the probed rule ID (`CA5351`) as a warning or an error.
- **Executable as:** build

#### AC-56 — No analysis or doc-warning suppression at project level

- **Verifies:** R-20
- **Given** the evaluated MSBuild properties of each library, and every `.editorconfig` and `.globalconfig` that applies to it
- **When** they are inspected
- **Then** nothing switches analysis off or silences missing docs or security rules
- **Assertion:**
  - `NoWarn` contains neither `1591` / `CS1591` nor any `CA5` ID.
  - `RunAnalyzers` ≠ `false`, and `EnableNETAnalyzers` ≠ `false`.
  - `AnalysisMode` / `AnalysisLevel` ≠ `None` / `none`.
  - No config file sets `dotnet_diagnostic.CS1591.severity` or `dotnet_diagnostic.CA5*.severity` to `none` or `silent`.
- **Executable as:** lint

#### AC-57 — A clean Release build has zero warnings

- **Verifies:** R-20
- **Given** the release commit
- **When** `dotnet build -c Release --no-incremental` runs at the repository root
- **Then** the build reports no warnings and no errors
- **Assertion:** exit code == 0, and the summary reports `0 Warning(s)` and `0 Error(s)`.
- **Executable as:** build

#### AC-58 — Workflows declare least-privilege permissions

- **Verifies:** R-21
- **Given** every file under `.github/workflows/`
- **When** each is parsed (for example `yq '.permissions, .jobs[].permissions'`)
- **Then** every job runs under explicitly declared permissions, and the build workflow can write nothing
- **Assertion:** every workflow has a top-level `permissions` key, or every one of its jobs has one. In `dotnetcore.yml` (or whatever replaces it), no effective permission has the value `write`.
- **Executable as:** lint

#### Measurement set T and rules (apply to AC-59 to AC-70, AC-123 and AC-127)

| Path | Operation | Inputs |
| --- | --- | --- |
| P-a | `Base62.Encode` | random bytes with a nonzero first byte, 32 B, 256 B, 1 KiB (1,024 B, the R-25 cap; fixed seed). 4 KiB was dropped in this revision because it is over the cap. A nonzero first byte keeps the output identical at baseline and after (GP-2). |
| P-b | `Base62.Decode` | the Base62 encodings of the P-a inputs |
| P-c | `Base16.Encode` | random bytes, 32 B, 256 B, 4 KiB (fixed seed) |
| P-d | `Base16.Decode` | the lowercase hex of the P-c inputs |
| P-e | `EdDsa.Verify(byte[], int, int, byte[], int, int)` | Ed25519; a 1 KiB message at offset 512 of a 2 KiB buffer; the signature at offset 16 of a 128-byte buffer |
| P-f | EdDSA signature provider `Sign(ReadOnlySpan<byte>, Span<byte>, out int)` | Ed25519; a 1 KiB message; a 64-byte destination |
| P-g | EdDSA signature provider `Sign(byte[], int, int)` | Ed25519; a 1 KiB message at offset 512 of a 2 KiB buffer |
| P-h | `ExtendedPasswordValidator<TUser>.ValidateAsync` | a 64-character password with no run longer than 2; `MaxConsecutiveChars = 3`; `MaxLength = 128` |

**Rules.**
- **Baseline and after.** The baseline is the source at `555d31c`. "After" is the release candidate.
- **Same conditions.** Both are run on the same machine and the same .NET 10.0.x runtime, with the same inputs, in the same session. `555d31c` does not target .NET 10, so the baseline must be rolled forward or rebuilt for .NET 10; how is the Architect's call. The point is to measure the code change, not the runtime change.
- **Report contents.** For each path and size, the report gives mean time per operation, its error margin (confidence half-width, at a stated confidence level of at least 95%), and allocated bytes per operation.
- **Improved.** Time is improved when after-mean + after-error < baseline-mean − baseline-error. Allocation is improved when after-bytes < baseline-bytes.
- **Regressed.** Time has regressed when after-mean − after-error > baseline-mean + baseline-error. Allocation has regressed when after-bytes > baseline-bytes.
- **Thresholds.** Any improvement is enough (A-8). Firmer per-path targets wait on Q-5.

#### AC-59 — The measurement report exists and is comparable

- **Verifies:** R-22
- **Given** the measurement report committed at the path §3 names
- **When** it is parsed
- **Then** it holds a baseline row and an after row for every path and size in T, taken under identical conditions
- **Assertion:**
  - The row count == 2 × (the number of path-size pairs in T).
  - Every row has a mean, an error margin, a confidence level of at least 95%, allocated bytes, a runtime version and a machine identifier.
  - Within each pair, the runtime version (10.0.x) and the machine identifier are equal.
  - The report holds only set T (Ed25519 for P-e to P-g). Ed448 rows, if R-52 is done, go in the separate companion report of AC-123 and are not counted here.
- **Executable as:** lint

#### AC-60 — Base62 encode improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-a rows are compared
- **Then** P-a improves at every size
- **Assertion:** at 32 B, 256 B and 1 KiB, time is improved or allocation is improved, as defined in the rules.
- **Executable as:** integration

#### AC-61 — Base62 decode improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-b rows are compared
- **Then** P-b improves at every size
- **Assertion:** as in AC-60, for P-b.
- **Executable as:** integration

#### AC-62 — Base16 encode improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-c rows are compared
- **Then** P-c improves at every size
- **Assertion:** as in AC-60, for P-c.
- **Executable as:** integration

#### AC-63 — Base16 decode improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-d rows are compared
- **Then** P-d improves at every size
- **Assertion:** as in AC-60, for P-d.
- **Executable as:** integration

#### AC-64 — Verify with offsets improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-e rows are compared
- **Then** P-e improves
- **Assertion:** time is improved or allocation is improved.
- **Executable as:** integration

#### AC-65 — Span signing improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-f rows are compared
- **Then** P-f improves
- **Assertion:** time is improved or allocation is improved.
- **Executable as:** integration

#### AC-66 — Offset signing improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-g rows are compared
- **Then** P-g improves
- **Assertion:** time is improved or allocation is improved.
- **Executable as:** integration

#### AC-67 — The repeated-character password check improves

- **Verifies:** R-23
- **Given** the AC-59 report
- **When** the P-h rows are compared
- **Then** P-h improves
- **Assertion:** time is improved or allocation is improved.
- **Executable as:** integration

#### AC-68 — No measured path regresses

- **Verifies:** R-55 (moved from R-23 when R-55 was split out)
- **Given** the AC-59 report
- **When** every path-size pair is compared
- **Then** nothing gets worse on either metric
- **Assertion:** for every pair, neither time nor allocation has regressed, as defined in the rules. Count of regressions == 0.
- **Executable as:** integration

#### AC-69 — Base16 time scales linearly

- **Verifies:** R-24
- **Given** random inputs of 8 KiB and 64 KiB (fixed seed) on the measurement machine
- **When** the median time per call of `Base16.Encode`, and of `Base16.Decode` on the matching hex, is measured at both sizes
- **Then** an 8× larger input costs no more than 1.5× the linear expectation
- **Assertion:** for each operation, median(64 KiB) ≤ 12 × median(8 KiB).
- **Executable as:** integration

#### AC-70 — Base62 meets its time budget at the cap

_Replaces this revision's earlier linear-scaling check. Q-12 / GP-2 chose a cap and a time budget instead of linear time._

- **Verifies:** R-25
- **Given** on the measurement machine, in one session:
  - a 1,024-byte random input with a nonzero first byte, and its Base62 encoding (fixed seed);
  - the calibration figure B: the mean time of the `555d31c` `Base62.Encode` on a 16,384-byte random input with a nonzero first byte, run on the same .NET 10 runtime.
- **When** the mean time per call of the release candidate's `Base62.Encode` on the 1,024-byte input, and of `Base62.Decode` on its encoding, is measured
- **Then** each call stays within the budget, scaled to the machine's speed
- **Assertion:** for each operation, mean ≤ 3 ms × (B ÷ 580 ms). On Fred's machine, which produced the GP-2 figures and where B is about 580 ms, the budget is 3 ms. The report records B, the scale factor and both means.
- **Executable as:** integration

**Why these values.** Proposed by the PM (A-15) and confirmed by Fred at Gate 1 (Q-23). The reasoning below was written before the 1 KiB measurement; the measured figures follow it.

The cap is **1,024 bytes**:
- No known consumer uses Base62. assetvault-io, the only known consumer, has no Base62 use (searched 2026-10-25).
- Its realistic uses are identifiers and keys of 16 to 64 bytes, so 1 KiB is at least 16 times that.
- It stays far below the sizes where SimpleBase's quadratic cost bites: 4 KB takes about 24 ms, and 16 KB about 380 ms, on Fred's machine.

The budget is **3 ms**:
- SimpleBase's two measured points are consistent with quadratic growth: 380 ÷ 24 ≈ 15.8, against 16 expected for a 4× larger input.
- Scaling down to 1 KiB by that factor estimates about 1.5 ms per `Encode` (24 ms ÷ 16, or 380 ms ÷ 256).
- A 3 ms budget gives 2× headroom over that estimate. The headroom covers the fact that `Decode` has not been measured at all, and that fixed per-call overheads matter more at small sizes.
- At 3 ms, one maximum-size call is a small fraction of a typical web request's budget, so no single input can hold a thread disproportionately.

The budget alone is not a speed-up requirement. R-25(d) and AC-127 separately require being faster than `555d31c`, whose 1 KiB cost is estimated at about 2.4 ms (38 ms ÷ 16).

Calibrating to B makes the budget portable between machines without changing its meaning.

**Measured at 1,024 bytes** *(scratch probe on Fred's machine, 2,000 calls, 2026-10-25, relayed by the /spec coordinator)*:

| Implementation | Encode | Decode |
| --- | --- | --- |
| SimpleBase | 1.577 ms | 0.809 ms |
| Today's code (`555d31c`) | 3.555 ms | not measured |

SimpleBase's output was 1,375 characters, within the 1,376-character decode cap.

So the budget holds with about 1.9× headroom on encode and 3.7× on decode, and encode is about 2.25× faster than today. The extrapolated 2.4 ms for today's code was low; the measured 3.555 ms replaces it.

These figures predict AC-70's and AC-127's results. They do not replace running those ACs on the release candidate. If a release-candidate run exceeds the budget, that is reported to Fred, not resolved by an execution agent.

#### AC-71 — Fragmented stream reads succeed

- **Verifies:** R-26
- **Given** a 32-byte source behind a stream that returns at most 1 byte per `Read` call
- **When** `stream.TryRead(32, out var bytes)` is called
- **Then** the read succeeds with all the data
- **Assertion:** returns `true`, and `bytes` sequence-equals the source.
- **Executable as:** unit

#### AC-72 — A stream that ends early still fails

- **Verifies:** R-26
- **Given** a stream holding 10 bytes
- **When** `stream.TryRead(32, out _)` is called
- **Then** the read reports failure
- **Assertion:** returns `false`.
- **Executable as:** unit

#### AC-73 — One EdDsa instance is safe across threads

- **Verifies:** R-27
- **Given** one `EdDsa` instance holding a key pair (both curves)
- **When** 16 concurrent tasks each sign and verify 500 distinct 64-byte messages with that instance
- **Then** every operation succeeds
- **Assertion:** all 8,000 verifications return `true`. 0 exceptions.
- **Executable as:** unit

#### AC-74 — The single step turns on extended validation

- **Verifies:** R-28
- **Given** a `ServiceCollection` with `AddIdentityCore<IdentityUser>()` and the single registration step, configured as `RequiredLength = 8`, `MaxLength = 64`, `MaxConsecutiveChars = 3`, with no other password configuration and no `AddPasswordValidator` call
- **When** the resolved `UserManager<IdentityUser>` runs each of its `PasswordValidators` against a 65-character password with no repeated runs
- **Then** the extended rule fires
- **Assertion:** the combined errors include `Code == "PasswordTooLong"`.
- **Executable as:** integration

#### AC-75 — The single step also drives the tag helper

- **Verifies:** R-28
- **Given** the AC-74 service provider
- **When** a `NewPasswordTagHelper` built from that provider's `IOptions<IdentityOptions>` processes a `newpassword` element
- **Then** the rendered element carries the extended rules
- **Assertion:** the output attribute `passwordrules` contains `maxlength: 64;` and `max-consecutive: 3;`, and the attribute `maxlength` == `64`.
- **Executable as:** integration

#### AC-76 — Invalid options stop the host at startup (single step)

- **Verifies:** R-29 (1.0.0, through R-28's single step, also 1.0.0)
- **Given** a generic host that uses the single registration step with each invalid configuration below
- **When** `host.StartAsync()` is called
- **Then** startup fails, and the exception message (or an inner one) names the offending setting

  | Configuration | Message contains |
  | --- | --- |
  | `MaxLength = 0` | `MaxLength` |
  | `MaxLength = -1` | `MaxLength` |
  | `MaxConsecutiveChars = 0` | `MaxConsecutiveChars` |
  | `MaxConsecutiveChars = -1` | `MaxConsecutiveChars` |
  | `RequiredLength = 12`, `MaxLength = 10` | `MaxLength` and `RequiredLength` |

- **Assertion:** for each row, `StartAsync` throws, and the exception's message (or its `InnerException` chain) contains the listed names, ordinal. At baseline every row is silently accepted; the two 0 rows are Fred's Q-17 answer.
- **Executable as:** integration

#### AC-77 — Valid options, including null limits, start normally

- **Verifies:** R-29
- **Given** a generic host using the single registration step, run twice: once with the AC-74 configuration, and once with `MaxLength = null` and `MaxConsecutiveChars = null`
- **When** `host.StartAsync()` is called in each case
- **Then** both hosts start, and `null` means no limit
- **Assertion:** no exception is thrown. In the `null` host, the extended validator accepts a 10,000-character password with no repeated run (`Succeeded == true`), and accepts `"aaaaaaaa"` (8 identical characters).
- **Executable as:** integration

#### AC-78 — Null-argument guard sweep across the public API

- **Verifies:** R-30
- **Given** every public constructor and public method declared in both library assemblies, excluding `Try`-prefixed methods, property accessors and compiler-generated members. Each member is called with valid fixture values for every argument except the one under test.
- **When** each parameter that `NullabilityInfoContext` reports as non-nullable reference type is passed `null`, one at a time
- **Then** each call fails immediately with an argument error naming that parameter
- **Assertion:** every call throws `ArgumentNullException` with `ParamName` == the parameter's name. Count of failures == 0. At baseline, for example, `Base62.Encode(null)` reports `ParamName == "source"` and `ConvertFromEdDsaSecurityKey(null)` throws `NullReferenceException`.
- **Executable as:** unit

#### AC-79 — Base62.Decode(null) names its argument

- **Verifies:** R-30
- **Given** a `null` input
- **When** `Base62.Decode(null)` is called
- **Then** it fails with an argument error
- **Assertion:** throws `ArgumentNullException` with `ParamName == "value"`. At baseline this throws `NullReferenceException`.
- **Executable as:** unit

#### AC-80 — The tag helper rejects missing options at construction

- **Verifies:** R-30
- **Given** a valid `IHtmlGenerator`
- **When** `new NewPasswordTagHelper(generator, null)` is called
- **Then** construction fails, rather than a later failure in `Process`
- **Assertion:** throws `ArgumentNullException` with `ParamName == "optionsAccessor"`.
- **Executable as:** unit

#### AC-81 — The EdDSA crypto provider rejects missing arguments

- **Verifies:** R-31
- **Given** the `ICryptoProvider` the library supplies for EdDSA, reached as `key.CryptoProviderFactory.CustomCryptoProvider` or however §3.3 exposes it
- **When** `Create("EdDSA")` is called with an empty `args` array
- **Then** it fails with an argument error
- **Assertion:** throws `ArgumentException` with `ParamName == "args"`. At baseline this throws `IndexOutOfRangeException`.
- **Executable as:** unit

#### AC-82 — Converting a null JWK still returns false

- **Verifies:** R-30
- **Given** a `null` JWK
- **When** `TryConvertToEdDsaSecurityKey(null, out var k)` is called
- **Then** the try-pattern reports failure
- **Assertion:** returns `false`, `k == null`, and nothing is thrown.
- **Executable as:** unit

#### AC-83 — TryRead reports a null stream through its return value

- **Verifies:** R-30 (the `Try`-member contract)
- **Given** a `null` stream
- **When** `StreamExtensions.TryRead(null, 32, out var bytes)` is called
- **Then** the try-pattern reports failure, exactly as AC-82 does for a null JWK
- **Assertion:** returns `false`, `bytes == null`, and nothing is thrown. At baseline this throws `NullReferenceException`.
- **Executable as:** unit

#### AC-84 — Verify rejects out-of-range offsets and lengths

- **Verifies:** R-31
- **Given** an Ed25519 key, a 100-byte `input`, a 64-byte `signature`, and each row below
- **When** `EdDsa.Verify(input, inputOffset, inputLength, signature, signatureOffset, signatureLength)` is called
- **Then** each call fails with an argument error naming the argument

  | inputOffset | inputLength | signatureOffset | signatureLength | `ParamName` ∈ |
  | --- | --- | --- | --- | --- |
  | -1 | 10 | 0 | 64 | {`inputOffset`} |
  | 95 | 10 | 0 | 64 | {`inputOffset`, `inputLength`} |
  | 0 | -1 | 0 | 64 | {`inputLength`} |
  | 0 | 10 | -1 | 64 | {`signatureOffset`} |
  | 0 | 10 | 10 | 64 | {`signatureOffset`, `signatureLength`} |

- **Assertion:** each row throws `ArgumentException` with `ParamName` in the listed set. At baseline, rows 1, 2, 4 and 5 are silently clamped, and row 3 throws without a `ParamName`.
- **Executable as:** unit

#### AC-85 — An empty message verifies through both overloads

- **Verifies:** R-31
- **Given** RFC 8032 §7.1 TEST 1 (Ed25519, empty message, with its published public key and signature)
- **When** `Verify(new byte[0], 0, 0, sig, 0, 64)` and `Verify(new byte[0], sig)` are called
- **Then** both overloads accept the valid signature over the empty message
- **Assertion:** both return `true`. At baseline the offset overload throws `ArgumentException`.
- **Executable as:** unit

#### AC-86 — TryRead rejects a negative length

- **Verifies:** R-31
- **Given** any readable stream
- **When** `stream.TryRead(-1, out _)` is called
- **Then** it fails with an argument error
- **Assertion:** throws `ArgumentOutOfRangeException` with `ParamName == "length"`. At baseline this throws `OverflowException`.
- **Executable as:** unit

#### AC-87 — A private-only key verifies

- **Verifies:** R-32
- **Given** `e = EdDsa.Create(new EdDsaParameters(curve) { D = d })`, with no `X` (both curves)
- **When** `e.Verify(message, e.Sign(message))` is called for a 32-byte message
- **Then** the signature verifies
- **Assertion:** returns `true`. At baseline this throws `InvalidOperationException`.
- **Executable as:** unit

#### AC-88 — A private-only security key validates a JWT

- **Verifies:** R-32
- **Given** an `EdDsaSecurityKey` over the private-only `EdDsa` from AC-87 (Ed25519)
- **When** a JWT signed with that key is validated with the same key as `IssuerSigningKey`
- **Then** the token validates
- **Assertion:** `TokenValidationResult.IsValid == true`.
- **Executable as:** integration

#### AC-89 — Span Sign reports a small destination through its return value

- **Verifies:** R-33
- **Given** `p = key.CryptoProviderFactory.CreateForSigning(key, "EdDSA")`, a 32-byte message, and a destination one byte shorter than a signature: 63 bytes for Ed25519, 113 for Ed448 (both curves)
- **When** `p.Sign(message, destination, out var written)` is called
- **Then** the try-pattern reports failure without throwing
- **Assertion:** returns `false`, `written == 0`, and nothing is thrown.
- **Executable as:** unit

#### AC-90 — Span Sign still works when the destination fits

- **Verifies:** R-33
- **Given** the AC-89 setup with a destination of exactly the signature length: 64 or 114 bytes (both curves)
- **When** `p.Sign(message, destination, out var written)` is called
- **Then** the signature is written and verifies
- **Assertion:** returns `true`, `written` == the signature length, and `key.EdDsa.Verify(message, destination[..written]) == true`.
- **Executable as:** unit

#### AC-91 — KeySize is in bits

- **Verifies:** R-34
- **Given** keys for each curve holding (D and X), D only, and X only
- **When** `EdDsa.KeySize` and `EdDsaSecurityKey.KeySize` are read
- **Then** both report bits
- **Assertion:**
  - Ed25519: both == `256`.
  - Ed448: both == `456`, the 57-byte key encoding of RFC 8032 (Q-13, confirmed by Fred at Gate 1).
  - At baseline they are `32` and `57`.
- **Executable as:** unit

#### AC-92 — LegalKeySizes agrees with KeySize

- **Verifies:** R-34
- **Given** an `EdDsa` for each curve
- **When** `LegalKeySizes` is read
- **Then** it describes exactly the reported key size, in bits
- **Assertion:** it has exactly one entry, with `MinSize == MaxSize == KeySize` and `SkipSize == 0`.
- **Executable as:** unit

#### AC-93 — The key ID survives conversion to a JWK

- **Verifies:** R-35
- **Given** an Ed25519 `EdDsaSecurityKey` with `KeyId = "key-2026-10"`
- **When** `ConvertFromEdDsaSecurityKey(key)` is called
- **Then** the JWK carries the key ID
- **Assertion:** `jwk.Kid == "key-2026-10"`.
- **Executable as:** unit

#### AC-94 — The key ID survives conversion from a JWK

- **Verifies:** R-35
- **Given** a `JsonWebKey` with `Kty = "OKP"`, `Crv = "Ed25519"`, a valid `X`, and `Kid = "key-2026-10"`
- **When** `TryConvertToEdDsaSecurityKey(jwk, out var k)` is called
- **Then** the key carries the key ID
- **Assertion:** returns `true`, and `k.KeyId == "key-2026-10"`.
- **Executable as:** unit

#### AC-95 — Leftover and misnamed constants are removed

- **Verifies:** R-36
- **Given** both library assemblies at the release commit
- **When** every public field of every exported type is reflected
- **Then** none of the removed members remains (GP-4)
- **Assertion:** the count of public fields named any of `ChaCha20Poly1305`, `XChaCha20Poly1305`, `ChaCha20Poly1305KeyWrap`, `XChaCha20Poly1305KeyWrap`, `EchdEsWithChaCha20Poly1305`, `EchdEsWithXChaCha20Poly1305`, or `Ecdh` on a type named `KeyTypes`, == 0. Marking them `[Obsolete]` does not pass.
- **Executable as:** unit

#### AC-96 — Any public OKP constant is correctly named

- **Verifies:** R-36
- **Given** both library assemblies at the release commit
- **When** every public `const string` field of every exported type is reflected
- **Then** every field whose value is `"OKP"` has a name that matches its meaning
- **Assertion:** for every public const field with value `"OKP"`, the name does not contain `Ecdh`, and contains `Okp` or `OctetKeyPair` (case-insensitive). The AC passes vacuously if no such field exists (G-20: "if the OKP value stays public"). JWK round trips keep `Kty == "OKP"` either way (AC-26, AC-94).
- **Executable as:** unit

#### AC-97 — Public API carries nullability metadata

- **Verifies:** R-37
- **Given** both library assemblies
- **When** `NullabilityInfoContext` reads every public method parameter and return value, property and field of reference type, declared in a public type
- **Then** every one has a known nullability
- **Assertion:** `ReadState != NullabilityState.Unknown` for every item. Count of `Unknown` == 0.
- **Executable as:** unit

#### AC-98 — Every public member is documented

- **Verifies:** R-38
- **Given** each library assembly and the XML documentation file built with it
- **When** every public type and every public or protected member declared in the assembly is matched to its documentation ID
- **Then** each one has documentation
- **Assertion:** for every item, a `<member name="…">` entry exists with a non-empty `<summary>` or an `<inheritdoc/>`. Count missing == 0.
- **Executable as:** build

#### AC-99 — An undocumented public member is reported

- **Verifies:** R-38
- **Given** a probe copy of each library with one added public method that has no doc comment
- **When** the probe is built with `-c Release`
- **Then** the build reports the missing documentation
- **Assertion:** the build output contains `CS1591`.
- **Executable as:** build

#### AC-100 — Docs describe no removed features

- **Verifies:** R-39
- **Given** `README.md` and each package readme at the release commit
- **When** they are searched for the removed features
- **Then** neither is mentioned
- **Assertion:** `rg -i "branca|paseto"` finds 0 matches in those files. `CHANGELOG.md` is excluded.
- **Executable as:** lint

#### AC-101 — Docs do not send consumers to upstream

- **Verifies:** R-39
- **Given** the same files as AC-100
- **When** they are searched for links to upstream's packages, badges, CI, issues or releases
- **Then** none is found
- **Assertion:** these patterns find 0 matches:
  - `nuget\.org/packages/ScottBrady`
  - `shields\.io/[^ )]*ScottBrady`
  - `scottbrady91/IdentityModel/(actions|workflows|issues|releases)`
- **Executable as:** lint

#### AC-102 — Every documented C# example compiles

- **Verifies:** R-40
- **Given** every fenced code block with the info string `csharp`, `cs` or `c#` in `README.md` and the package readmes
- **When** each is compiled in the harness §3 defines: a `net10.0` project referencing both released packages, with each block placed as §3 specifies
- **Then** every block compiles
- **Assertion:** compiler errors == 0 for every block. At baseline, README line 152 fails this check.
- **Executable as:** build

#### AC-103 — Each package carries its own readme

- **Verifies:** R-41
- **Given** each `.nupkg`
- **When** its nuspec and entries are read
- **Then** the package has a readme that is about this package
- **Assertion:** `<readme>` names a file that exists in the package, and that file contains the package's own ID.
- **Executable as:** build

#### AC-104 — The 1.0.0 changelog entry lists every breaking change

- **Verifies:** R-42
- **Given** `CHANGELOG.md` at the release commit
- **When** its `1.0.0` section is read
- **Then** it has a dated heading and a breaking-changes list covering each topic below, each detected by a phrase list that §3 fixes:
  - the new package IDs;
  - the namespace rename;
  - .NET 10 as the only target;
  - JWK export now public-only by default;
  - `KeySize` now in bits;
  - Base62 output for leading zero bytes;
  - malformed encoded input now rejected with `FormatException`, from Base16 and Base62 alike;
  - argument-exception changes for invalid arguments;
  - span `Sign` now returning `false` when the destination is too small;
  - the password message wording;
  - invalid `ExtendedPasswordOptions` now rejected, including 0 for `MaxLength` and `MaxConsecutiveChars` (use `null` for no limit);
  - removal of the six ChaCha20 constants and `KeyTypes.Ecdh`;
  - PEM import members now throwing `NotSupportedException`;
  - the Base62 maximum input length and its rejection of longer input;
  - key validation now rejecting mismatched `D`/`X` pairs, and the D-5 length rule;
  - `EdDsaSecurityKey` no longer modifying `CryptoProviderFactory.Default`;
  - the key ID now carried through JWK conversion;
  - `TryRead` now succeeding on fragmented streams.
- **Assertion:** a `1.0.0` heading with an ISO date exists, and each of the 18 topics has at least one matching phrase within that section.
- **Executable as:** lint

#### AC-105 — Every release is recorded

- **Verifies:** R-42
- **Given** every version of either package published to the private feed, or tagged in git
- **When** `CHANGELOG.md` headings are read
- **Then** each version has an entry
- **Assertion:** for each version, a heading containing that version string exists. Count missing == 0.
- **Executable as:** lint

#### AC-106 — A whitespace-only password yields a validation failure, not an error

- **Verifies:** R-43
- **Given** `ExtendedPasswordOptions { MaxConsecutiveChars = 3 }`
- **When** `ExtendedPasswordValidator<IdentityUser>.ValidateAsync(manager, user, "    ")` (4 spaces) is awaited
- **Then** it returns a failed result instead of throwing
- **Assertion:** no exception, `result.Succeeded == false`, and `result.Errors` contains `Code == "TooManyConsecutiveCharacters"`. At baseline this throws `ArgumentNullException`.
- **Executable as:** unit

#### AC-107 — Short whitespace and empty passwords pass this validator

- **Verifies:** R-43
- **Given** the AC-106 options
- **When** `ValidateAsync` is awaited with `"  "` (2 spaces) and with `""`
- **Then** each returns success from this validator. Minimum length is checked elsewhere.
- **Assertion:** no exception, and `result.Succeeded == true` for both.
- **Executable as:** unit

#### AC-108 — The public consecutive-character check handles whitespace

- **Verifies:** R-43
- **Given** a validator instance
- **When** `HasConsecutiveCharacters("    ", 3)` is called
- **Then** it detects the run instead of throwing
- **Assertion:** returns `true`, and nothing is thrown.
- **Executable as:** unit

#### AC-109 — Default messages use the built-in phrasing

- **Verifies:** R-44
- **Given** options `MaxLength = 64` and `MaxConsecutiveChars = 3`, and no replacement messages registered
- **When** `ValidateAsync` is awaited with a 65-character password, and separately with `"aaaa"`
- **Then** both error descriptions follow ASP.NET Identity's phrasing
- **Assertion:** each `Description` starts with `"Passwords must "`, ends with `"."`, and contains its configured number: `64` for the length error, `3` for the consecutive-characters error.
- **Executable as:** unit

#### AC-110 — Error codes are unchanged

- **Verifies:** R-44
- **Given** the AC-109 inputs
- **When** the error codes are read
- **Then** they match the existing codes that consumers key on
- **Assertion:** the codes are exactly `"PasswordTooLong"` and `"TooManyConsecutiveCharacters"`.
- **Executable as:** unit

#### AC-111 — The app can replace the messages

- **Verifies:** R-44
- **Given** a service provider where the app has registered replacements through the message mechanism (D-11):
  - `"Le mot de passe ne doit pas dépasser 64 caractères."` for the length error;
  - `"Le mot de passe ne doit pas contenir plus de 3 caractères identiques consécutifs."` for the consecutive-characters error.
- **When** the DI-resolved validator runs with the AC-109 inputs
- **Then** the replacements are used and the codes stay the same
- **Assertion:** each `Description` equals its replacement string exactly, and the codes are as in AC-110.
- **Executable as:** integration

#### AC-112 — The sample runs on .NET 10

- **Verifies:** R-45
- **Given** the sample hosted in-process on `net10.0`
- **When** its home page and its password-rules page are requested (`GET /` and `GET /Home/PasswordRules`, or the routes §3 maps them to)
- **Then** both pages load
- **Assertion:** both responses are 200.
- **Executable as:** integration

#### AC-113 — The sample uses minimal hosting

- **Verifies:** R-45
- **Given** the sample's source at the release commit
- **When** it is searched
- **Then** it uses `WebApplication` hosting and no `Startup` class. This asserts mechanism, which is allowed here because G-25 is about the hosting convention itself.
- **Assertion:** 0 matches for `class Startup` and for `UseStartup`, and at least 1 match for `WebApplication.CreateBuilder`.
- **Executable as:** lint

#### AC-114 — Every input on the password form has a label

- **Verifies:** R-45
- **Given** the HTML returned by the password-rules page from AC-112
- **When** every `<input>` whose type is not `hidden`, `submit` or `button` is matched to `<label for>` attributes, case-sensitively
- **Then** each input has exactly one associated label
- **Assertion:** each such input has a non-empty `id`, and exactly one `<label>` has `for` == that `id`. At baseline, `for="Password"` does not match `id="password"`.
- **Executable as:** integration

#### AC-115 — A breaking API change fails the build

- **Verifies:** R-46
- **Given** a probe copy of the post-1.0.0 source with one public member removed (for example `Base62.Encode(byte[])`), and the published 1.0.0 package as the compatibility baseline
- **When** the probe is built and packed with `-c Release`
- **Then** the compatibility check fails and names the change
- **Assertion:** exit code ≠ 0, and the output contains the removed member's name.
- **Executable as:** build

#### AC-116 — Line coverage meets the floor for each library

- **Verifies:** R-47
- **Given** the CI build of the release commit
- **When** its coverage report is read
- **Then** each library meets the floor on its own (A-6)
- **Assertion:** line coverage ≥ 80.0% for `ValhallaTech.Security.IdentityModel`, and ≥ 80.0% for `ValhallaTech.Security.IdentityModel.AspNetCore`. The sample is excluded.
- **Executable as:** build

#### AC-117 — The coverage floor is enforced

- **Verifies:** R-47
- **Given** the coverage gate that CI runs, fed a coverage report showing one library at 79.9% line coverage
- **When** the gate runs
- **Then** it fails and names that library
- **Assertion:** exit code ≠ 0, and the output contains the library's name and `79.9`.
- **Executable as:** build

#### AC-118 — Dependency updates arrive automatically

- **Verifies:** R-48
- **Given** the dependency-update tool's configuration (D-7) merged to `master` before the 1.0.0 release. Its coverage is checked by AC-134.
- **When** the repository is queried on the release date
- **Then** the tool has already acted on the repository on its own
- **Assertion:** `gh pr list --author <tool bot> --state all` or `gh issue list --author <tool bot>` returns at least 1 item, created after the merge and no later than the release. An onboarding PR or a dependency dashboard issue counts.
- **Executable as:** integration

#### AC-119 — Advisories reach the maintainers automatically

- **Verifies:** R-48
- **Given** the canonical repository (C-22)
- **When** advisory alerting is queried
- **Then** alerts on dependencies are delivered without manual checking
- **Assertion:** either `GET /repos/IronSloth/IdentityModel-Supa/vulnerability-alerts` returns 204 (enabled), or the D-7 tool's configuration turns on its vulnerability alerts and §3 names where they are delivered.
- **Executable as:** integration

#### AC-120 — Formatting is checked in CI

- **Verifies:** R-49
- **Given** the CI formatting check (D-7)
- **When** it runs on the release commit, and separately on a probe copy with one misformatted line
- **Then** it passes on the release and fails on the probe
- **Assertion:** exit code == 0 on the release commit, and exit code ≠ 0 on the probe. The CI workflow contains that check as a step.
- **Executable as:** lint

#### AC-121 — Acceptance tests fail before their implementation lands

- **Verifies:** R-50
- **Given** the commit history from `555d31c` to the release commit, and the §3.4 chains. The ACs in scope are every `unit` AC, plus these in-process `integration` ACs: AC-22, AC-24, AC-29, AC-52 to AC-54, AC-74 to AC-77, AC-88, AC-111, AC-112, AC-114, AC-138, AC-141, AC-142, AC-143.
- **When** each in-scope AC's acceptance test is run at the commit that introduced it
- **Then** each test either fails by assertion there or is a declared regression guard. Unit-level test-first, beyond these acceptance tests, is AC-132.
  - **Regression guards,** expected to pass at `555d31c`: AC-24, AC-32, AC-40, AC-41, AC-72, AC-73, AC-77, AC-82, AC-90, AC-110, AC-112, AC-135 and AC-141.
  - **Added at Gate 2:**
    - AC-110: the two error codes already exist (`ExtendedPasswordValidator.cs:30,41`).
    - AC-112: the sample's pages already load.
  - **Removed at Gate 2: AC-29.** It fails at `555d31c`; this was found by reading the code, not by running it. `ConvertFromEdDsaSecurityKey` hands the JWK the key's factory, `CryptoProviderFactory.Default`, whose custom provider `ExtendedCryptoProvider` is set by the key's constructor (`EdDsaSecurityKey.cs:16`, `ExtendedJsonWebKeyConverter.cs:22`). That provider's `IsSupportedAlgorithm` checks only the algorithm name, so it claims `"EdDSA"` for the JWK (`ExtendedCryptoProvider.cs:9-10`). Its `Create` then accepts only an `EdDsaSecurityKey` and throws `NotSupportedException` for a `JsonWebKey` (`ExtendedCryptoProvider.cs:14-19`). Validation with the JWK therefore fails.
- **Assertion:**
  - For every non-guard AC in scope, the introducing commit comes before the implementing commit that §3.4 names.
  - At the introducing commit, the test compiles against the stubs and fails with an assertion failure, not a compile error.
  - At the release commit, every in-scope test passes.
- **Executable as:** integration

#### AC-122 — Sample and README show the single step

- **Verifies:** R-51
- **Given** the sample source and the README's password-validation example at the release commit
- **When** they are searched
- **Then** both use only the single registration step
- **Assertion:** 0 matches for `AddPasswordValidator<ExtendedPasswordValidator` and for `Password = new ExtendedPasswordOptions` in either. Both contain the single step's call, as named in §3.3.
- **Executable as:** lint

#### AC-123 — Ed448 signing paths are measured

- **Verifies:** R-52
- **Given** a companion Ed448 measurement report, separate from the AC-59 report, committed at the path §3 names
- **When** it is parsed
- **Then** it covers P-e, P-f and P-g on Ed448, under the same rules as AC-59
- **Assertion:**
  - The report has exactly 6 rows: a baseline row and an after row for each of P-e, P-f and P-g, with curve Ed448.
  - Each row has the fields AC-59 requires, and each pair has equal runtime and machine identifiers.
  - Ed448 signatures are 114 bytes, so the signature buffer is enlarged to fit.
  - The AC-59 report's row count is unaffected.
- **Executable as:** lint

#### AC-124 — Base62 rejects input over the encode cap

- **Verifies:** R-25
- **Given** byte arrays of 1,025 bytes and of 1,048,576 bytes
- **When** `Base62.Encode(value)` is called for each
- **Then** each is rejected before any conversion work
- **Assertion:** each call throws `ArgumentException` (or subtype) with `ParamName == "value"`. The 1,048,576-byte call returns, by throwing, in under 50 ms. At baseline it runs for minutes.
- **Executable as:** unit

#### AC-125 — Base62 rejects input over the decode cap

- **Verifies:** R-25
- **Given** a string of 1,377 `'1'` characters (over the 1,376-character maximum), and a string of 1,376 `'z'` characters (within the length limit, but its value needs more than 1,024 bytes)
- **When** `Base62.Decode(value)` is called for each
- **Then** each is rejected, and no result over 1,024 bytes is ever returned
- **Assertion:** each call throws `ArgumentException` (or subtype) with `ParamName == "value"`.
- **Executable as:** unit

#### AC-126 — Base62 round-trips at the cap

- **Verifies:** R-25
- **Given** a 1,024-byte array whose first two bytes are `0x00` and the rest random, plus a 1,024-byte random array with a nonzero first byte (fixed seed)
- **When** each is encoded and the result decoded
- **Then** both are accepted and come back unchanged, so the cap admits its own boundary
- **Assertion:** for each, `Base62.Decode(Base62.Encode(b))` sequence-equals `b`, and `Base62.Encode(b).Length ≤ 1376`.
- **Executable as:** unit

#### AC-127 — Base62 is faster than `555d31c` at the cap

- **Verifies:** R-25
- **Given** the AC-59 report's P-a and P-b rows at 1 KiB
- **When** the after rows are compared with the baseline rows
- **Then** both `Encode` and `Decode` are faster at the cap
- **Assertion:** for P-a and P-b at 1 KiB, time is improved, as defined in the measurement rules. A lower allocation count alone does not satisfy this AC.
- **Executable as:** integration

#### AC-128 — Validation rejects a mismatched key pair

- **Verifies:** R-15
- **Given** for each curve, two freshly generated key pairs A and B, and `p = new EdDsaParameters(curve) { D = A.D, X = B.X }`
- **When** `p.Validate()` is called
- **Then** it is rejected at validation, with a reason (S11, GP-5)
- **Assertion:** throws `CryptographicException`, and the message contains `match` (case-insensitive). With `X = A.X` instead, `Validate()` does not throw. At baseline the mismatched set passes.
- **Executable as:** unit

#### AC-129 — PEM import is reported as unsupported

- **Verifies:** R-53
- **Given** an `EdDsa` for each curve
- **When** `ImportFromPem("x")`, `ImportFromEncryptedPem("x", "p")` (char password) and `ImportFromEncryptedPem("x", new byte[] { 1 })` (byte password) are called
- **Then** each signals an unsupported operation
- **Assertion:** each throws `NotSupportedException`, exactly that type or a subtype that is not `NotImplementedException`. At baseline each throws `NotImplementedException`.
- **Executable as:** unit

#### AC-130 — The migration guide covers every breaking change

- **Verifies:** R-54
- **Given** the migration guide at the path §3 names. Its owner is the Documentation Agent under the repository-docs rule.
- **When** its entries are matched against AC-104's 18 topics, and against this list of upstream 4.1.0 APIs that assetvault-io uses (`src/AssetVault.BLL/Tokens/EdDsaKeyMaterial.cs`, `src/AssetVault.BLL/AssetVault.BLL.csproj`, read 2026-10-25):
  - the `ScottBrady.IdentityModel` 4.1.0 package reference;
  - the namespaces `ScottBrady.IdentityModel.Crypto` and `ScottBrady.IdentityModel.Tokens`;
  - `EdDsa.Create(EdDsaParameters)`;
  - `EdDsaParameters { D, X }`;
  - `ExtendedSecurityAlgorithms.Curves.Ed25519`;
  - `new EdDsaSecurityKey(EdDsa) { KeyId = … }`.
- **Then** every topic and every listed API has an entry stating the old form, the 1.0.0 form, and the action required (or "no change")
- **Assertion:** for each of the 18 topics and 6 APIs, at least one entry matches (by the phrase list §3 fixes), and each entry has non-empty "before", "after" and "action" parts. Count of uncovered items == 0.
- **Executable as:** lint

#### AC-131 — assetvault-io migrates using the guide alone

- **Verifies:** R-54
- **Given** a scratch copy of assetvault-io at a commit §3 records, made outside both repositories. assetvault-io itself is never edited.
- **When** only the steps in the migration guide are applied: the package reference is switched to `ValhallaTech.Security.IdentityModel` 1.0.0 from the private feed, and the documented replacements are made. Then `dotnet build` runs on its solution and `dotnet test` on `tests/AssetVault.BLL.Tests`.
- **Then** it builds and its token tests pass with no change beyond what the guide states
- **Assertion:** build exit code == 0. Test results show failed == 0 and skipped == 0. The diff of the copy against the recorded commit touches only lines the guide's steps call for.
- **Executable as:** integration

#### AC-132 — Each fix is driven by a failing test first

- **Verifies:** R-50
- **Given** the commit history from `555d31c` to the release commit, and every §3.4 work item that changes library or sample behavior
- **When** for each such work item, the test commit that §3.4 associates with it is checked out and the suite is run, and then the implementing commit
- **Then** each fix is preceded by at least one test that failed before it and passes after it (C-13, GP-3)
  - **Library work items:** that test is a unit test.
  - **Sample work items:** it may be an in-process `integration` test, because Razor views, layouts and static assets can be observed only through the hosted sample, not in isolation.
- **Assertion:** for every such work item:
  - at least one test of the allowed level is introduced in a commit before the implementing commit;
  - at its introducing commit, that test compiles and fails with an assertion failure;
  - at the implementing commit and at the release commit, it passes.

  Count of work items without such a test == 0.
- **Executable as:** integration

#### AC-133 — Sample client-side libraries are at their latest stable version

- **Verifies:** R-17
- **Given** every third-party client-side library vendored in the sample (as in AC-50), identified by name and version
- **When** each is compared with the latest stable release that its maintainer has published on the release date
- **Then** each is current
- **Assertion:** the count of vendored libraries below their latest stable version == 0.
- **Executable as:** lint

#### AC-134 — The update tool covers every direct dependency

- **Verifies:** R-48
- **Given** the dependency-update tool's configuration (D-7) at the release commit, and the inventory of direct dependencies from AC-49 (NuGet) and AC-133 (sample client-side libraries), plus every `uses:` action in `.github/workflows/`
- **When** the tool's resolved list of managed dependencies (for example its dependency dashboard, or a dry-run log) is compared with that inventory
- **Then** every inventory item is managed by the tool
- **Assertion:** inventory − managed set == ∅. A vendored file the tool cannot see fails this check. G-28 then needs the sample's client libraries declared in a form the tool reads.
- **Executable as:** lint

#### AC-135 — Disposal leaves the caller's array untouched

- **Verifies:** R-11
- **Given** a caller-owned array `d` holding a valid private key, a copy `snapshot = d.ToArray()`, and `e = EdDsa.Create(new EdDsaParameters(curve) { D = d })` (both curves). This mirrors assetvault-io, `EdDsaKeyMaterial.cs:51-53`.
- **When** `e.Dispose()` is called
- **Then** the library has not modified the caller's array
- **Assertion:** `d` sequence-equals `snapshot`.
- **Executable as:** unit

#### AC-136 — The Base62 cap is documented

- **Verifies:** R-25
- **Given** the XML documentation file built with the core library
- **When** the `<member>` entries for `Base62.Encode(System.Byte[])` and `Base62.Decode(System.String)` are read
- **Then** each documents its limit
- **Assertion:** the `Encode` entry contains `1024` or `1,024`, and the `Decode` entry contains `1376` or `1,376`. Each also names the exception thrown when the limit is exceeded (`ArgumentException`).
- **Executable as:** build

#### Definition used by AC-137, AC-138 and AC-143: unmodified release files (set V)

The set **V** contains every file in the repository, outside `bin/` and `obj/`, whose SHA-256 equals the SHA-256 of a file in the published npm tarball of a library at the exact version listed in AC-133's inventory. A file in V is an unmodified third-party release file, and its text belongs to that library.

C-23 / GP-10: optional jQuery-detection text inside such a library's own bundle is not use of jQuery. For example, `bootstrap.bundle.min.js` 5.3.8 contains 19 `window.jQuery` / `jQueryInterface` mentions. Any file not in V is the repository's own and is searched in full.

#### AC-137 — The repository contains no jQuery file, reference or dependency

- **Verifies:** R-56 (C-23, GP-10)
- **Given** the release commit, including ignored and hidden files, and the set V defined above. The exclusion list below is **closed**: nothing else may be excluded. A jQuery file, reference or dependency in any other place fails this AC.

  | Excluded | Why it is not use of jQuery (C-23) |
  | --- | --- |
  | `bin/`, `obj/` | Build output, regenerated from the source that is checked. |
  | `.git/` | Git metadata such as commit subjects, the reflog and branch names. History is not shipped content. |
  | `specs/` | Prose about the decision itself. |
  | `CHANGELOG.md` | A line recording jQuery's removal. |
  | The one test file `ForbiddenClientLibraryTests.cs`, at the test-project path §3 names | It must name jQuery in order to check for it (AC-138, AC-143). Exactly one file with that name may exist. A second one, or one outside the test project, is not excluded. |
- **When** these four checks run:
  1. **File names:** `fd -H -I -i jquery -E bin -E obj -E .git`.
  2. **Content outside V:** `rg -i -uu -l "jquery" -g "!**/bin/**" -g "!**/obj/**" -g "!.git/**" -g "!specs/**" -g "!CHANGELOG.md" -g "!<test-project path>/ForbiddenClientLibraryTests.cs"`, minus the files in V. This covers every view, layout, `.cs` file, hand-written script, stylesheet, manifest (`package.json`, `libman.json`, `.csproj`, `.props`) and config file.
  3. **Inventory names:** the npm names of the libraries in AC-133's inventory are checked for `jquery` or a `jquery`-prefixed name.
  4. **Dependencies:** AC-139's dependency check, which is referenced here and not repeated.
- **Then** no jQuery library file exists under any name, nothing in the repository's own files references jQuery, and no listed library is jQuery or a jQuery plugin
- **Assertion:**
  - Checks 1, 2 and 3 each return 0 results.
  - A jQuery library file renamed to anything else fails check 2. Its SHA-256 matches no inventory library's release file, so it is not in V and its jQuery banner text is searched.
  - Bootstrap 5.3.8's unmodified `bootstrap.bundle.min.js` is in V and passes.
  - The excluded test file is itself checked separately: it must not contain a jQuery library banner (`jQuery (JavaScript Library )?v\d` or `jquery\.org/license`), and it must not load or import jQuery. It may only name jQuery as the thing being forbidden.
  - Building V needs the npm registry; see the network rule in AC-138.
  - At baseline, check 1 finds `samples/…/wwwroot/js/jquery.min.js`, and check 2 finds that file and the reference in `Views/Shared/_Layout.cshtml`.
- **Executable as:** lint

#### AC-138 — No page served by the sample loads jQuery

- **Verifies:** R-56 (C-23, GP-10)
- **Given** the sample hosted in-process on `net10.0`, every page it serves (at least the home page and the password-rules page, per AC-112), and the set V
- **When** every `<script src>` and `<link href>` on each page is fetched from the same host, each body is hashed and read, and each inline `<script>` block is read
- **Then** nothing the page loads is jQuery or references it. Bootstrap's own unmodified bundle may still be served.
- **Assertion:**
  - The count of loaded assets whose URL path contains `jquery` (case-insensitive) == 0.
  - Every loaded asset whose body contains `jquery` (case-insensitive) must have a SHA-256 equal to a file in V. The count of loaded assets that contain `jquery` and are not in V == 0.
  - Inline `<script>` blocks contain neither `jquery` (case-insensitive) nor a `$(` call.
  - At baseline the layout loads `js/jquery.min.js`, which fails the first bullet.
- **Executable as:** integration
- **Needs network:** yes. Building V needs HTTPS access to `registry.npmjs.org`, to read each listed library's version metadata and download its release tarball for hashing.
  - If the registry cannot be reached, or a tarball or metadata request fails, the test **fails**. It reports that the npm registry is unreachable and names the URL that failed. It never passes, and it is never skipped or marked inconclusive.
  - A cached tarball counts only if its own SHA-256 matches the `dist.integrity` value the registry publishes for that version. Checking that still needs the registry.
  - The same rule applies to every other AC that uses V or npm metadata: AC-133, AC-137, AC-139, AC-140 and AC-143.

#### AC-139 — No sample front-end library depends on jQuery

- **Verifies:** R-56
- **Given** every third-party client-side library the sample ships, from the inventory of AC-133, identified by npm name and version
- **When** each one's published npm metadata (`dependencies` and `peerDependencies`) is read for that exact version
- **Then** none depends on jQuery
- **Assertion:** for every library, neither `dependencies` nor `peerDependencies` has a key `jquery`. Count of failures == 0.
- **Executable as:** lint

#### AC-140 — The sample uses the latest stable Bootstrap 5

- **Verifies:** R-56
- **Given** the Bootstrap files the sample ships, and npm's `bootstrap` dist-tag `latest` on the release date (`5.3.8` as of 2026-10-25, per the /spec coordinator; 6.0.0 exists only as an alpha)
- **When** the shipped Bootstrap version is read, from its file banner or the manifest that declares it, and compared with that tag
- **Then** it is the latest stable 5.x
- **Assertion:** the shipped major version == 5, and the shipped version == npm `dist-tags.latest`. Bootstrap 4.x (4.6.2 included) fails, and so does any prerelease.

  If Bootstrap 6 becomes the stable `latest` before release, the two halves conflict. That is reported to Fred, not resolved by an agent.
- **Executable as:** lint

#### AC-141 — The sample stays a Razor demo of the tag helper

- **Verifies:** R-56
- **Given** the sample hosted in-process on `net10.0`
- **When** its password-rules page is requested
- **Then** the page is rendered server-side by Razor, and shows the `newpassword` tag helper's output
- **Assertion:** status 200. The HTML contains an `<input>` with `type="password"`, `autocomplete="new-password"` and a non-empty `passwordrules` attribute. The sample project contains at least one `.cshtml` view using `<newpassword`.
- **Executable as:** integration

#### AC-142 — Invalid options are rejected on the two-step path too

- **Verifies:** R-29 (1.0.0)
- **Given** a service provider built with upstream's two-step pattern and no single step:
  - `AddIdentityCore<IdentityUser>(o => o.Password = new ExtendedPasswordOptions { … })`;
  - `.AddPasswordValidator<ExtendedPasswordValidator<IdentityUser>>()`, ASP.NET Identity's own method.

  Each invalid configuration from AC-76's table is tried in turn.
- **When**
  - (a) the resolved validator's `ValidateAsync` is awaited with the password `"Valid-Password-1"`, and
  - (b) a `NewPasswordTagHelper` built from the same provider processes a `newpassword` element
- **Then** each use fails loudly, naming the offending setting. It is never silently ignored, and never turned into a validation result the end user would see.
- **Assertion:** for every row and for both (a) and (b), an exception is thrown whose message (or `InnerException` chain) contains the names listed in AC-76, ordinal. Neither call returns normally. At baseline both return normally and ignore the setting.

  Fred accepted first-use rejection on this path (Q-25).
- **Executable as:** integration

#### AC-143 — The sample serves Bootstrap 5's JavaScript unmodified

- **Verifies:** R-56 (GP-10: "Ship it")
- **Given** the sample hosted in-process on `net10.0`, and the set V
- **When** the shared layout's `<script src>` assets are fetched from the home page
- **Then** Bootstrap's JavaScript bundle is loaded, served intact, and is the version AC-140 requires. That bundle is what keeps the mobile menu working.
- **Assertion:**
  - At least one loaded script returns 200 and has a SHA-256 equal to `dist/js/bootstrap.bundle.min.js` or `dist/js/bootstrap.bundle.js` from the npm `bootstrap` tarball at the version AC-140 accepts.
  - Whether the menu works in a real browser is not asserted, because that needs a browser-level test, which this spec does not use. A served, unmodified bundle is the observable evidence below the browser.
- **Executable as:** integration
- **Needs network:** yes. The test needs HTTPS access to `registry.npmjs.org`, to fetch the `bootstrap` metadata (including `dist-tags.latest`) and the release tarball it hashes against. If the registry cannot be reached, the test **fails**, reporting that the npm registry is unreachable and naming the URL that failed. It never passes silently and is never skipped. A cached tarball counts only under AC-138's integrity rule.

### 2.4 Success metrics

_How we know it worked after it ships. Metric, current baseline, target, measurement source._

| ID | Metric | Baseline | Target | Measured by |
| --- | --- | --- | --- | --- |
| M-1 | ValhallaTech projects on upstream `ScottBrady.IdentityModel` that have moved to `ValhallaTech.Security.IdentityModel` 1.x | 0 of 1 (assetvault-io, on 4.1.0; Q-4 resolution) | assetvault-io on 1.x by 2026-11-24 (14 days after release), and 0 ValhallaTech projects on `ScottBrady.*` by 2026-12-31 | A search of ValhallaTech repositories for `ScottBrady.IdentityModel` and `ValhallaTech.Security.IdentityModel` package references, plus download statistics from the IronSloth GitHub Packages feed (GP-10) |
| M-2 | Open High or Critical dependency advisories on `master` | 2, both test-only and transitive through AutoFixture: `System.Net.Http` 4.3.0 and `System.Text.RegularExpressions` 4.3.0 (review, `555d31c`) | 0 at every release, and 0 open for more than 7 days in between | The CI advisory audit (AC-45) and the repository's advisory alerts (AC-119) |
| M-3 | Line coverage for each packable library | unknown (not collected today) | ≥ 80% on every `master` build | The CI coverage report (AC-116) |
| M-4 | Median days from a High or Critical advisory on a dependency to a merged fix on `master` | unknown, needs instrumentation | ≤ 7 days | Advisory publication dates compared with the merge dates of the update tool's PRs (D-7) |
| M-5 | 1.x releases that contain an unintended breaking API change | n/a, since no 1.x release exists yet | 0 | The API compatibility check (AC-115) on every release, cross-checked with `CHANGELOG.md` |
| M-6 | Hot-path performance in later 1.x releases | the 1.x figures recorded under R-22 | no path in T regresses against the most recent recorded release | Re-running measurement set T before each minor release |
| M-7 | Days on which a shipped target framework or the CI SDK is out of Microsoft support | > 0 (the AspNetCore library has targeted `net6.0`, out of support since November 2024) | 0 from 1.0.0 onward | Target frameworks and CI SDK versions checked against Microsoft's .NET support policy |
| M-8 | HTTP 500 errors in consuming apps caused by an exception from the extended password validator | unknown, needs instrumentation in the consuming apps | 0 | The consuming apps' error logs, filtered to exceptions thrown from `ExtendedPasswordValidator` |
| M-9 | Vulnerability reports for this fork that arrive through the canonical repository's private reporting | unknown (no channel exists: private reporting returned `{"enabled":false}` on 2026-10-25, C-21, and `SECURITY.md` points to the upstream author) | every report received for the fork comes through private reporting | The GitHub security advisories list on the canonical repository (C-22) |
| M-10 | Days a direct dependency on `master` stays behind its latest stable version | > 0 at `555d31c`. For example, Microsoft.IdentityModel 7.7.1 against a current 8.x, and BouncyCastle.Cryptography 2.4.0 against 2.7.0. | 0 at every release. Between releases, no direct dependency more than 14 days behind its latest stable. | The update tool's PR history and dashboard (D-7, AC-134), plus `dotnet list package --outdated` run before each release (AC-49) |

### 2.5 Out of scope

_Carried forward from §1.4, plus anything cut during PRD authoring._

**Carried forward from §1.4**
- NG-1: publishing to nuget.org by 2026-11-10. Whether the public release belongs to this spec is Q-2.
- NG-2: supporting .NET 6, 8 or 9 consumers.
- NG-3: staying mergeable with upstream, or contributing back to it.
- NG-4: restoring Branca or PASETO.
- NG-5: new algorithms or token formats.
- NG-6: bringing ValhallaTech's application stack into the libraries.
- NG-7: Heimdall-scale process or infrastructure.
- NG-8: private-key copies held inside BouncyCastle. AC-30 and AC-31 skip BouncyCastle-owned objects for this reason.
- NG-9: migrating public nuget.org consumers of upstream `ScottBrady.*`. ValhallaTech's own projects are covered by G-29 and R-54.

**Cut or bounded during PRD authoring**

IDs are kept stable. Withdrawn items stay listed so that earlier references still resolve.

- **X-1 — withdrawn (GP-2).** A maximum Base62 input length is now required (R-25).
- **X-2 — `ObjectDisposedException` on use after `Dispose`.** No goal asks for it, and R-11 asserts only zeroing and that the caller's array is left alone.
- **X-3 — withdrawn (GP-4).** The constants are removed in 1.0.0 (R-36, AC-95).
- **X-4 — Shipped translations of the password messages.** R-44 requires that messages can be replaced. It does not require the library to ship any translations.
- **X-5 — Ed448 performance measurement.** This is `MAY` (R-52). G-13's hot paths are measured for Ed25519.
- **X-6 — Renaming repository directories, project files, the solution file or the test project.** Consumers do not see these inside the packages, so the PRD does not require it. The Architect may do it under D-9.
- **X-7 — narrowed (Q-4 resolution).** A migration guide for public upstream consumers stays out (NG-9). One for ValhallaTech's own 4.1.0 consumers is now required (R-54). Actually migrating assetvault-io is that project's own work; this spec proves the guide only against a scratch copy (AC-131).
- **X-8 — withdrawn (GP-6).** The PEM import members now throw `NotSupportedException` (R-53, AC-129).
- **X-9 — withdrawn (GP-5).** Validation now rejects mismatched `D`/`X` (R-15, AC-128).
- **X-10 — withdrawn (Q-24, Fred, Gate 1).** Base62 now translates SimpleBase's invalid-character `ArgumentException` into `FormatException` (R-13, AC-36).

### 2.6 Open questions carried forward

_Unresolved §1.6 questions still relevant, plus new ones raised by the PRD._

| ID | Question | Blocking? | Owner | Affects |
| --- | --- | --- | --- | --- |
| Q-2 | Is the public nuget.org release part of this spec's definition of done? Assumed separate (NG-1). | no | Fred | none yet (NG-1) |
| Q-5 | Is any measured gain enough for each hot path, or is there a minimum per path? The PRD assumes any gain (A-8). R-25(c)–(d) set Base62's targets separately. | no | Fred | R-23, AC-60 to AC-67 |
| Q-6 | Does the 80% floor apply per library, with the sample excluded? The PRD assumes yes (A-6). | no | Fred | R-47, AC-116 |
| Q-9 | What is the largest Base62 or Base16 input consumers need? The only known consumer (assetvault-io) uses neither, so the PRD's proposed 1,024-byte Base62 cap has no known conflict. | no | Fred | R-22, R-24, R-25, measurement set T, AC-69, AC-70 |
| Q-10 | Are the two unreported `EdDsa.Verify` defects in G-18's scope? The PRD assumes yes (A-11). | no | Fred | R-31, AC-84, AC-85 |


**Resolved**

| ID | Resolution | Source |
| --- | --- | --- |
| Q-25 | "Helper in 1.0.0." R-28 (the single-step setup helper) moves into 1.0.0. Invalid options are rejected at startup through that step (AC-76) and at first use through upstream's two-step path (AC-142), which Fred accepts. R-51 stays a `1.x` SHOULD. | Fred, after Gate 1, relayed by the /spec coordinator |
| Q-13 | Ed448 `KeySize` is 456 bits (57-byte encoding × 8). R-34 and AC-91 already used this value. | Fred, Gate 1 |
| Q-17 | Reject 0 and negative values for `MaxLength` and `MaxConsecutiveChars`. `null` means no limit. Rejection applies however the validator is registered, which makes it breaking, so R-29 moves to 1.0.0. Covered by AC-76, AC-77 and the new AC-142. The two-step path's timing is Q-25. | Fred, Gate 1 |
| Q-18 | The icon is the ValhallaTech company logo, a PNG of at least 128×128 that Fred supplies before release. "No icon" is no longer accepted (R-5, AC-12, FC-6). | Fred, Gate 1 |
| Q-21 | Confirmed. The specific goals override G-18's general rule: `FormatException` for bad encoded text, `CryptographicException` for invalid keys, `ArgumentException` for bad arguments. | Fred, Gate 1 |
| Q-23 | Confirmed: a cap of 1,024 bytes for `Encode` and 1,376 characters for `Decode`, and a budget of 3 ms mean per call at the cap. Measured on Fred's machine at 1,024 bytes: SimpleBase takes 1.577 ms to encode and 0.809 ms to decode, against 3.555 ms for today's encode, with 1,375-character output (FC-5 holds). | Fred, Gate 1; scratch probe of 2,000 calls, 2026-10-25 |
| Q-24 | Yes. Base62 translates SimpleBase's invalid-character `ArgumentException` into `FormatException`, and AC-36 now requires `FormatException`. Over-cap input stays `ArgumentException`. X-10 is withdrawn. | Fred, Gate 1 |
| Q-1 | The full 1.0.0 scope ships by 2026-11-10. §2.2.1 is now firm. | GP-1, C-20 |
| Q-3 | Both IDs are free on nuget.org. | §1.6 |
| Q-4 | assetvault-io uses upstream 4.1.0. This adds G-29 / R-54. | §1.6 |
| Q-7 | Test-first applies at unit level too. R-50 and AC-132 cover it. | GP-3, C-13 |
| Q-8 | `IronSloth/IdentityModel-Supa` is canonical (C-22). Private reporting is disabled, and only Fred can enable it (C-21); AC-20 depends on that. | GP-7 |
| Q-11 | Yes. assetvault-io is already on Microsoft.IdentityModel 8.19.1 and BouncyCastle 2.6.2. | §1.6 |
| Q-12 | A cap, a time budget, and faster than today (R-25). SimpleBase is decided for Base62 (D-3 resolved). | GP-2, coordinator correction 2026-10-25 |
| Q-14 | Removed in 1.0.0 (R-36, AC-95). | GP-4 |
| Q-15 | Mismatched pairs are rejected (R-15, AC-128). | GP-5 |
| Q-16 | The library zeroes only its own copies and never modifies a caller-supplied array (R-11, AC-135). assetvault-io passes its own array as `D` (`EdDsaKeyMaterial.cs:53`, Analyst). | PM decision on the Analyst's input, as directed by the coordinator |
| Q-19 | R-18's any-severity rule stands. Every jQuery 3.3.1 advisory is rated medium, so a High-or-Critical threshold would miss them all. | `gh api /advisories?affects=jquery@3.3.1`, run by the /spec coordinator, 2026-10-25 |
| Q-20 | A prerelease does not count as shipped. | GP-1, C-20 |
| Q-22 | `NotSupportedException` in 1.0.0 (R-53, AC-129). | GP-6 |
| GP-10 (Gate 2) | D-2 decided: GitHub Packages under IronSloth (R-3, AC-8, AC-9, FC-1). The sample ships Bootstrap 5's JavaScript (AC-143), and the no-jQuery checks exclude unmodified third-party release files (set V; AC-137, AC-138). SimpleBase follows latest stable, no longer pinned to 5.6.4 (R-12, R-13). | Fred, Gate 2, relayed by the /spec coordinator; §1.8 GP-10, C-23 |
| GP-8 (no Q) | The sample is kept as a Razor app on the latest stable Bootstrap 5.x, with no jQuery anywhere in the repository (R-56, AC-137 to AC-141). | Fred, relayed by the /spec coordinator, 2026-10-25. The Analyst records it in §1 afterwards. |

---

## 3. Technical Plan

<!-- OWNER: Architect. No other agent may edit this section. -->

**Status:** ✅ complete

### 3.1 Affected systems & repos

| System / repo | Nature of change | Owner domain |
| --- | --- | --- |
| `src/ScottBrady.IdentityModel` → `src/ValhallaTech.Security.IdentityModel` | Renamed, retargeted to `net10.0`. EdDSA key isolation, validation, disposal, JWK conversion, Base16/Base62, stream reads. | `backend` |
| `src/ScottBrady.IdentityModel.AspNetCore` → `src/ValhallaTech.Security.IdentityModel.AspNetCore` | Renamed, retargeted. Validator messages through an error describer, one-step registration, options validation, tag-helper guards. | `backend` |
| `test/ScottBrady.IdentityModel.Tests` → `test/ValhallaTech.Security.IdentityModel.Tests` | Retargeted. FluentAssertions replaced by AwesomeAssertions, AutoFixture removed, coverlet added. Hosts every library acceptance and unit test. | `tests` |
| New `test/ValhallaTech.Security.IdentityModel.Samples.Tests` | In-process host tests of the sample (`WebApplicationFactory<Program>`). Kept separate so library coverage excludes the sample (A-6). | `tests` |
| `samples/ScottBrady.IdentityModel.Samples.AspNetCore` → `samples/ValhallaTech.Security.IdentityModel.Samples.AspNetCore` | Hosting: minimal hosting, DryIoc, public-only validation key, diagnostics only in Development. UI: Bootstrap 5's CSS and JS bundle, unmodified, restored through LibMan (GP-10); working mobile menu; no jQuery; fixed label. | `backend` (hosting), `frontend` (views, assets) |
| New `perf/ValhallaTech.Security.IdentityModel.Benchmarks`, `perf/results/1.0.0/` | BenchmarkDotNet harness with a source-level `555d31c` baseline, and the committed report. | `tests` |
| Root build files: `global.json`, `Directory.Build.props`, `src/Directory.Build.props`, `Directory.Packages.props`, `ValhallaTech.Security.IdentityModel.slnx` (replaces `ScottBrady.IdentityModel.sln`), `assets/valhallatech-logo.png` | New shared build layout (D-9). | `backend` |
| `.config/dotnet-tools.json`, `.csharpierrc.json`, `renovate.json`, `build/*.ps1`, `.github/workflows/*.yml` (new `release.yml`) | .NET 10 CI, least-privilege tokens, format check, coverage gate, doc-snippet compile, Renovate, release to the private feed. | `ci` |
| `SECURITY.md`, `README.md`, new `CHANGELOG.md`, new `src/*/README.md`, new `docs/migration/scottbrady-4.1.0-to-1.0.0.md` | Repository and package documentation. | `docs` |
| GitHub settings on `IronSloth/IdentityModel-Supa` | Private vulnerability reporting, Dependabot alerts, Renovate app install. | Unmapped (Fred) |
| Private NuGet feed: GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json` (D-2 decided, GP-10) | Hosts 1.0.0. Push from the release workflow; consumers restore with a `read:packages` credential. | `ci`, plus Unmapped (Fred: credentials) |
| `C:\Repo\assetvault-io` | **Read-only, never edited.** A scratch copy of `main` at `863c0cb0cde62f820b76b9c76e1b4cf245f7dc55` (read from `.git/refs/heads/main` on 2026-10-25) is the AC-131 subject. | `tests` (scratch copy only) |

**Grounding: what was read and what is inferred.**

*Verified by reading the code at `555d31c` on 2026-10-25:*
- Every library source file: `EdDsa.cs`, `EdDsaParameters.cs`, `EdDsaSecurityKey.cs`, `EdDsaSignatureProvider.cs`, `ExtendedCryptoProvider.cs`, `ExtendedSecurityAlgorithms.cs`, `Base16.cs`, `Base62.cs`, `ExtendedJsonWebKeyConverter.cs`, `StreamExtensions.cs`, `ByteArrayExtensions.cs`, `ExtendedPasswordOptions.cs`, `ExtendedPasswordValidator.cs`, `NewPasswordTagHelper.cs`.
- All four `.csproj` files, the `.sln`, both workflows and `SECURITY.md`.
- The sample's `Startup.cs`, `Program.cs`, `SampleOptions.cs`, `HomeController.cs`, `_Layout.cshtml` and `PasswordRules.cshtml`, and the `wwwroot` file list.
- Every test method name (157 methods), plus the test files whose assertions this plan changes: `EdDsaParametersTests`, `EdDsaCreationTests`, `EdDsaSecurityKeyTests`, `NewPasswordTagHelperTests`, `StreamExtensionsTests`, `Base62Tests`, `JwtBearerHandlerTests`, and the AutoFixture usage (one file).
- In assetvault-io: `EdDsaKeyMaterial.cs`, `NuGet.config` (nuget.org only), `renovate.json`, and its `main` ref.

*Verified through Context7 on 2026-10-25:*
- `JsonWebTokenHandler` signature validation uses `validationParameters.CryptoProviderFactory ?? key.CryptoProviderFactory`. Source: `JsonWebTokenHandler.ValidateToken.cs` on the library's `dev` branch, `/azuread/azure-activedirectory-identitymodel-extensions-for-dotnet`. A per-key factory is therefore honored unless the app sets one on `TokenValidationParameters`.
- SimpleBase exposes `Base62.Default` on a `DividingCoder` that implements `IBaseCoder` and `INonAllocatingBaseCoder` (`/ssg/simplebase`, `docs/simplebase.base62.md`).

*Inferred, not verified (each is checked by the work item named):*
- A new `AsymmetricSecurityKey` shares the `CryptoProviderFactory.Default` instance, which is why line 16 writes process-wide state. The review's [confirmed] probe supports this. Checked by AC-21 in WI-11.
- `new CryptoProviderFactory()` is cheap enough to create per key: RK-4, AC-25.
- At `555d31c`, `ExtendedCryptoProvider.Create` throws for a `JsonWebKey` argument, so AC-29 fails there. §2 (AC-121) now records the same conclusion from reading the code. Nobody has run it.
- .NET 9+ `Convert.ToHexStringLower` and `Convert.FromHexString` throw `FormatException` on odd or invalid input: WI-23.
- `Stream.ReadAtLeast` exists in .NET 7+: WI-25.
- jsDelivr's `/npm/` path, which LibMan's `jsdelivr` provider uses, serves npm-tarball files byte for byte, so the restored Bootstrap files are in set V: RK-6, AC-143 in WI-36. That the bundle contains jQuery-detection text is no longer inferred; GP-10 records the coordinator's measurement of 19 mentions.
- `Microsoft.Web.LibraryManager.Build`, `DryIoc.Microsoft.DependencyInjection` 6.2.0 and DryIoc.dll 5.4.3 restore and run on `net10.0`: WI-35, WI-37.
- The workflow's `GITHUB_TOKEN` with `packages: write` can push a NuGet package whose `RepositoryUrl` names this repository to the IronSloth feed: WI-44's trial push (FC-1).
- Renovate has no LibMan manager (*per the /spec coordinator, read 2026-10-25 from `lib/modules/manager/api.ts` on Renovate `main`*). Renovate's `customManagers` regex type can read `libman.json`: WI-39.
- `PackageValidationBaselinePath` is the package-validation property for a local baseline: WI-52.
- `xunit` 2.9.x is not marked deprecated: WI-2 (AC-46).

### 3.2 Data changes

_Schema, migrations, indexes, cache keys and TTLs, retention. `none` if none._

**None.** No database, cache or persisted state exists in either library or the sample (the sample's EF store is in-memory and discarded on exit). Wire-format changes are API contract, so they are listed in §3.3.2: Base62 output for inputs with leading zero bytes, and the JWK's content.

### 3.3 API & contract changes

_New/changed endpoints, payloads, events, auth model. Flag breaking changes explicitly._

#### 3.3.1 Package, assembly and namespace map

Type names are unchanged, so the AC naming convention ("named as at `555d31c`, inside the renamed namespaces") applies directly. The only member rename is `KeyTypes.Ecdh` (see 3.3.2).

| At `555d31c` | At 1.0.0 |
| --- | --- |
| Package, assembly `ScottBrady.IdentityModel` 4.1.0 | `ValhallaTech.Security.IdentityModel` 1.0.0 |
| Package, assembly `ScottBrady.IdentityModel.AspNetCore` 2.2.0 | `ValhallaTech.Security.IdentityModel.AspNetCore` 1.0.0 |
| `ScottBrady.IdentityModel` (`Base16`, `Base62`, `ByteArrayExtensions`, `StreamExtensions`, `ExtendedJsonWebKeyConverter`) | `ValhallaTech.Security.IdentityModel` |
| `ScottBrady.IdentityModel.Crypto` | `ValhallaTech.Security.IdentityModel.Crypto` |
| `ScottBrady.IdentityModel.Tokens` | `ValhallaTech.Security.IdentityModel.Tokens` |
| `ScottBrady.IdentityModel.AspNetCore.Identity` | `ValhallaTech.Security.IdentityModel.AspNetCore.Identity` |
| `ScottBrady.IdentityModel.AspNetCore.TagHelpers` | `ValhallaTech.Security.IdentityModel.AspNetCore.TagHelpers` |
| Test project `ScottBrady.IdentityModel.Tests` | `ValhallaTech.Security.IdentityModel.Tests` (AC-5 compares names under this mapping) |
| Sample `ScottBrady.IdentityModel.Samples.AspNetCore` | `ValhallaTech.Security.IdentityModel.Samples.AspNetCore` |

Directories, project files and the solution are renamed too (X-6 permits it; D-9). One rename in Phase 0 is cheaper than carrying two names. The AspNetCore package keeps its separate package ID, and both packages version together from `src/Directory.Build.props` (A-2).

#### 3.3.2 Public API and behavior changes

**Every row marked breaking goes into the CHANGELOG and the migration guide (WI-40, WI-41).**

| Surface | Change | Breaking? | Decision |
| --- | --- | --- | --- |
| All public types | Namespace and assembly rename (3.3.1). | **Yes** | C-7, C-8 |
| Target framework | `net10.0` only. | **Yes** | C-2 |
| Dependencies | Minimum versions rise to latest stable: Microsoft.IdentityModel.* 8.x (a major bump from 7.7.1), BouncyCastle.Cryptography 2.7.x. SimpleBase (latest stable, GP-10) is new. A consumer whose own direct reference is lower gets NU1605 (assetvault-io pins BouncyCastle 2.6.2); the guide tells it to raise its reference. | **Yes** | G-9, §2.2.1(a) |
| `EdDsaSecurityKey` ctor | Never writes `CryptoProviderFactory.Default`. Each key gets its own `CryptoProviderFactory` whose `CustomCryptoProvider` is the library's EdDSA provider, with `CacheSignatureProviders = false`. Apps that relied on the global side effect, for example by validating EdDSA through some other key type, lose it. | **Yes** | S1/C1. Alternative lost: one library-owned factory shared by all EdDSA keys. It is cheaper, but a consumer changing one key's factory would change every key (G-4). |
| EdDSA `ICryptoProvider.Create` / `IsSupportedAlgorithm` | `args` null or empty → `ArgumentException` (`ParamName "args"`). Also accepts a `JsonWebKey` with `Kty "OKP"` and an Ed25519/Ed448 `Crv`, converted through `TryConvertToEdDsaSecurityKey`. It reports support only for those argument shapes. | **Yes** (exception type) | R-31, AC-81, AC-29 |
| `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKey(EdDsaSecurityKey)` | Public material only (`D == null`), even for a key holding a private key. `Kid` is copied. Null `key` → `ArgumentNullException`. | **Yes** | S2, R-10, R-35 |
| New `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKeyIncludingPrivateKey(EdDsaSecurityKey)` | The explicit private-export opt-in. It also sets `D`. | No (additive) | **Resolves D-4.** A separately named method is greppable in code review and audits. Its call site states the intent, and the default overload keeps its signature. Alternatives lost: a `bool includePrivateKey = false` parameter (`true` at a call site hides the intent), and an export-mode enum (more surface for a two-way choice). |
| `TryConvertToEdDsaSecurityKey` | Copies `jwk.Kid` to `KeyId`. Null `jwk` still returns `false` (AC-82). | **Yes** (output) | R-35 |
| `ExtendedSecurityAlgorithms` | Removes `ChaCha20Poly1305`, `XChaCha20Poly1305`, `ChaCha20Poly1305KeyWrap`, `XChaCha20Poly1305KeyWrap`, `EchdEsWithChaCha20Poly1305` and `EchdEsWithXChaCha20Poly1305`. Renames `KeyTypes.Ecdh` to `KeyTypes.OctetKeyPair` (value `"OKP"`). | **Yes** | GP-4, R-36 |
| `EdDsaParameters.Validate()` | `D` must be exactly 32 bytes (Ed25519) or 57 (Ed448). `X` likewise. Each message names "private key" or "public key" and the length. Validation builds the BouncyCastle key parameters through the same internal factory `Create` uses, and maps any failure to `CryptographicException`, so passing validation guarantees `Create` succeeds. When both are present, it derives the public key from `D` and compares it with `X` using `CryptographicOperations.FixedTimeEquals`. On mismatch the message contains "do not match". | **Yes** | **Resolves D-5: 64-byte Ed25519 and 114-byte Ed448 `D` are rejected.** The message tells the caller to split seed‖public into `D` and `X`. Supporting the concatenated form would add a parse path, a second key copy to zero, and a pair check for a format no known consumer uses (assetvault-io passes 32 bytes). S5, S11, GP-5. |
| `EdDsa.Create(EdDsaParameters)` | Copies `D` and `X` into arrays the library owns, and never retains or modifies the caller's arrays (Q-16). Derives and stores the public key when only `D` is given, so a private-only key verifies (R-32). R-32 may ship in 1.x, but the S11 check needs the same derivation, so it lands in 1.0.0 at no extra cost. Making a call succeed where it used to throw is non-breaking. | No | R-11, R-15, R-32 |
| `EdDsa.KeySize`, `LegalKeySizes`, `EdDsaSecurityKey.KeySize` | Reported in bits: 256 for Ed25519 and 456 for Ed448. `LegalKeySizes` has one entry, `(k, k, 0)`. | **Yes** | U5, Q-13 |
| `EdDsa.Dispose` | New `Dispose(bool)` override zeroes the library-owned private-key buffer with `CryptographicOperations.ZeroMemory`. Copies held by BouncyCastle are out of scope (NG-8). | No | S3 |
| `EdDsa.ImportFromPem`, both `ImportFromEncryptedPem` | Throw `NotSupportedException`. | **Yes** | GP-6 |
| `EdDsa.Verify(byte[], int, int, byte[], int, int)` | A negative or out-of-range offset or length → `ArgumentOutOfRangeException` naming the argument. A zero-length message is accepted, as in the two-array overload. No copies: it uses the span `BlockUpdate`. | **Yes** | R-31, Q-10 (A-11) |
| Signature provider `Sign(ReadOnlySpan<byte>, Span<byte>, out int)` | Returns `false` with `bytesWritten == 0` when the destination is too small. No input copy. | **Yes** | U4 |
| Signature provider `Sign(byte[], int, int)` | Validates its arguments and does not copy the input. | **Yes** (argument exceptions) | P3, R-31 |
| `Base62.Encode(byte[])` | Null → `ArgumentNullException("value")` (was `"source"`). More than 1,024 bytes → `ArgumentException("value")`. Otherwise `SimpleBase.Base62.Default.Encode`. Each leading zero byte becomes `'0'`; output is otherwise identical to `555d31c` (GP-2). | **Yes** | D-3 (GP-2), R-12, R-25 |
| `Base62.Decode(string)` | Decode, then check (R-25(b) as amended at Gate 2):<br>1. Null → `ArgumentNullException("value")`.<br>2. Longer than 1,376 characters → `ArgumentException("value")`. This bounds the decode time.<br>3. Any character outside `CharacterSet` (one ordinal pass) → `FormatException`.<br>4. `SimpleBase.Base62.Default.Decode`. Any SimpleBase `ArgumentException` that still escapes is translated to `FormatException` (Q-24).<br>5. A result longer than 1,024 bytes → `ArgumentException("value")`, so `Decode` never returns more than 1,024 bytes (AC-125).<br>The worst case is a full decode of a 1,376-character string. AC-70 measures decode at the cap, and the 0.809 ms measured at 1,024 bytes (Q-23) leaves headroom under the 3 ms budget. | **Yes** | R-12, R-13, R-25. Alternative lost: an exact pre-decode overflow check (ordinal comparison against cached encodings of 256^m). The time bound is the same, but it adds a cache and a comparison path that R-25(b) no longer requires. |
| `Base16.Encode` / `Decode` | `Convert.ToHexStringLower` / `Convert.FromHexString`. Odd or invalid input → `FormatException`. `""` → empty array. Uppercase is still accepted, and time is linear. | **Yes** | R-13, R-14, R-24 |
| `StreamExtensions.TryRead` | Null stream → returns `false` with `bytes == null`, and never throws (R-30's `Try`-member contract, AC-83). Negative length → `ArgumentOutOfRangeException("length")` (R-31, AC-86). Reads with `Stream.ReadAtLeast(buffer, length, throwOnEndOfStream: false)`. Returns `true` once `length` bytes have arrived; on early end of stream the `out` array stays partially filled, as today. | **Yes** | C3, R-26, R-30, R-31 |
| New `ExtendedIdentityErrorDescriber : IdentityErrorDescriber` | `virtual IdentityError PasswordTooLong(int maxLength)` and `virtual IdentityError TooManyConsecutiveCharacters(int maxConsecutiveChars)`. The default texts are below. | No | **Resolves D-11.** This is how ASP.NET Identity's own messages are replaced. The app derives from it and calls `IdentityBuilder.AddErrorDescriber<TDescriber>()`, and one describer then covers both the built-in and the extended messages. Translation works through any `IStringLocalizer` the subclass uses. Alternatives lost: a resource file plus `IStringLocalizer` inside the library (it ships translations, which X-4 cuts), and message templates on the options (not how Identity does it). |
| `ExtendedPasswordValidator<TUser>` | New ctor `(IdentityErrorDescriber? errors = null)`, mirroring the built-in `PasswordValidator<TUser>`. It uses the describer when that is an `ExtendedIdentityErrorDescriber`, and the default one otherwise. The consecutive check is now a linear scan instead of a `Regex` (P4). | **Yes** (wording) | R-44 |
| Default messages | `"Passwords must be no longer than {0} characters."` and `"Passwords must not contain more than {0} consecutive identical characters."`. Codes are unchanged. | **Yes** | R-44, AC-109, AC-110 |
| `HasConsecutiveCharacters(string, int)` | Null → `ArgumentNullException`. Empty or whitespace input returns a result instead of throwing. Threshold `max(1, maxConsecutive)`, as today. Every UTF-16 code unit counts, including `\n`, which the old regex `.` skipped. | **Yes** | U10, R-43 |
| `ExtendedPasswordOptions` rules | `MaxLength ≤ 0`, `MaxConsecutiveChars ≤ 0`, and `MaxLength < RequiredLength` are invalid. `null` means no limit. Each failure message names the setting or settings. Through the single step they fail at startup (`OptionsValidationException`); through the two-step path they fail at first `ValidateAsync` or tag-helper render, with the same exception. | **Yes** | GP-9 Q-17, Q-25, R-29 |
| New `ExtendedPasswordIdentityBuilderExtensions.AddExtendedPasswordRules(this IdentityBuilder builder, Action<ExtendedPasswordOptions> configure)` in `ValhallaTech.Security.IdentityModel.AspNetCore.Identity` | The single step (R-28). It:<br>- copies the existing base `PasswordOptions` values into a new `ExtendedPasswordOptions`, applies `configure`, and assigns `IdentityOptions.Password`;<br>- registers `IPasswordValidator<TUser>` → `ExtendedPasswordValidator<TUser>` for `builder.UserType` with `TryAddEnumerable`, so no duplicate if the app also registers it;<br>- registers an internal `IValidateOptions<IdentityOptions>`, plus `AddOptions<IdentityOptions>().ValidateOnStart()`.<br>It lives in a ValhallaTech namespace because AC-13 forbids exported types elsewhere, which rules out the conventional `Microsoft.Extensions.DependencyInjection` placement. | No (additive) | R-28, AC-74 to AC-77 |
| `NewPasswordTagHelper` ctor | Null `optionsAccessor` → `ArgumentNullException("optionsAccessor")`. Null `.Value` → `ArgumentException("optionsAccessor")`. `Process` applies the option rules above. | **Yes** | AC-80, AC-142(b) |
| Every public ctor/method | Null for a non-nullable reference parameter → `ArgumentNullException` naming it. `Try`-prefixed members keep their contracts. | **Yes** | R-30 |

#### 3.3.3 Contract notes for consumers (feed into WI-41)

- **JWKS consumers.** A `JsonWebKey` parsed from JSON carries the default `CryptoProviderFactory`, which has no EdDSA support once no global registration exists. Convert it with `TryConvertToEdDsaSecurityKey`, for example in `IssuerSigningKeyResolver`. At `555d31c` this path also failed (the provider threw for a `JsonWebKey`), so this is not a regression. It is still documented.
- **`TokenValidationParameters.CryptoProviderFactory`.** It overrides the key's factory, as the Context7 excerpt above shows. An app that sets it must include EdDSA support in it, or leave it unset.
- **One `EdDsa` instance stays safe to share across threads** (G-16). A new signer is still created per operation.

#### 3.3.4 Build layout (resolves D-9)

- **`ValhallaTech.Security.IdentityModel.slnx`.** XML solution format, read by the .NET 10 SDK CLI. It replaces the GUID-heavy `.sln`, and diffs and merges are readable. Alternative lost: keeping `.sln`, which has no benefit on an SDK-10-only repo.
- **`global.json`.** Pins the 10.0 SDK feature band with `rollForward: latestFeature`. It makes AC-3's "10.0.x is the only SDK selected" true by construction.
- **Root `Directory.Build.props`.** Holds settings shared by every project:
  - `TargetFramework=net10.0` and `LangVersion=latest`;
  - `TreatWarningsAsErrors=true`;
  - `NuGetAudit=true`, `NuGetAuditMode=all` and `NuGetAuditLevel=high`. A High or Critical advisory, transitive included, fails restore. That makes AC-45 a build property, not a script.
- **`src/Directory.Build.props`.** Imports the root file and adds library-only settings:
  - package metadata (3.3.5) and `Version=1.0.0`;
  - `Nullable=enable` and `GenerateDocumentationFile=true`;
  - `AnalysisLevel=latest-recommended` and `AnalysisModeSecurity=All`;
  - `IncludeSymbols` with `SymbolPackageFormat=snupkg`;
  - `PackageLicenseExpression=Apache-2.0`, plus `LICENSE` packed as a file (AC-16), which keeps the SPDX metadata license scanners read.

  Tests and the sample keep default analysis, because test names such as `ctor_When…` would trip CA1707 and break AC-57.
- **`Directory.Packages.props`.** Central package management. The direct-dependency inventory for AC-49 and AC-134 lives in one file that Renovate reads natively. Alternative lost: per-project versions, which let `Microsoft.IdentityModel.*` drift between the library and the tests.
- **No `.editorconfig` severity inventory.** C-17 and Fred's guidance rule it out; `AnalysisLevel` plus warnings-as-errors meets AC-55 to AC-57.

#### 3.3.5 Fixed strings and paths that the ACs defer to §3

- **Package metadata (both packages).**
  - `Authors` = `ValhallaTech`.
  - `PackageProjectUrl` and `RepositoryUrl` = `https://github.com/IronSloth/IdentityModel-Supa`.
  - `PackageReleaseNotes` = `https://github.com/IronSloth/IdentityModel-Supa/blob/master/CHANGELOG.md`.
  - `Copyright` = `Copyright 2026 (c) ValhallaTech. Portions Copyright 2022 (c) Scott Brady.` This contains the AC-17 string. No other element names Scott Brady (AC-11).
  - Descriptions: core = `EdDSA (Ed25519, Ed448) signing and validation for Microsoft.IdentityModel JWTs, plus strict Base16 and Base62 encoders.` AspNetCore = `ASP.NET Core Identity password rules: a passwordrules tag helper and an extended password validator.`
- **Icon (AC-12).** Fred supplies `assets/valhallatech-logo.png` (U-1). It is packed as `icon.png`, and the upstream `icon.png` at the repository root is deleted (WI-42).
- **Modification notice (AC-15, AC-18).** Placed as the first two lines of every `.cs` file under `src/` and `samples/` that derives from a file at `555d31c`. The same header goes on test files and on the `perf/` baseline copies.
  - Line 1: `// Portions Copyright 2022 (c) Scott Brady, https://github.com/scottbrady91/IdentityModel, licensed under the Apache License, Version 2.0.`
  - Line 2: `// Modified by ValhallaTech: this file was changed from the original. See CHANGELOG.md.`
  - AC-18 searches for the string `Modified by ValhallaTech: this file was changed from the original.` AC-15's only permitted `scottbrady` match is Line 1.
- **Measurement report (AC-59).**
  - Main report: `perf/results/1.0.0/report.csv`, with `report.md` as a readable copy. It holds exactly 2 rows × 16 path-size pairs of T, so 32 rows. Each row has mean, error (BenchmarkDotNet's 99.9% confidence half-width), confidence level, allocated bytes, runtime and machine name.
  - Companion files: the AC-123 Ed448 report in `report-ed448.csv`, exactly 6 rows (baseline and after for P-e, P-f and P-g); AC-69 in `base16-scaling.csv`; AC-70 in `base62-budget.csv`, which records `B`, the scale factor and both means.
  - The Ed448 rows are kept out of the main report so AC-59's exact row count holds.
- **Migration guide (AC-130).** `docs/migration/scottbrady-4.1.0-to-1.0.0.md`. Each entry is a table row with non-empty **Before**, **After** and **Action** cells. The API phrase list, where an entry matches if it contains the phrase, case-insensitive:

  | AC-130 API | Phrase |
  | --- | --- |
  | 4.1.0 package reference | `ScottBrady.IdentityModel" Version="4.1.0"` or `ScottBrady.IdentityModel 4.1.0` |
  | Namespaces | both `ScottBrady.IdentityModel.Crypto` and `ScottBrady.IdentityModel.Tokens` |
  | `EdDsa.Create(EdDsaParameters)` | `EdDsa.Create(` |
  | `EdDsaParameters { D, X }` | `EdDsaParameters` |
  | `ExtendedSecurityAlgorithms.Curves.Ed25519` | `Curves.Ed25519` |
  | `new EdDsaSecurityKey(EdDsa) { KeyId = … }` | `new EdDsaSecurityKey(` |

- **CHANGELOG 1.0.0 phrase list (AC-104 and AC-130's 18 topics).** A topic counts as covered if its section contains any listed phrase, case-insensitive.

  | # | Topic | Phrases |
  | --- | --- | --- |
  | 1 | New package IDs | `ValhallaTech.Security.IdentityModel` |
  | 2 | Namespace rename | `namespace` |
  | 3 | .NET 10 only | `net10.0`, `.NET 10` |
  | 4 | JWK public-only by default | `public-only`, `IncludingPrivateKey` |
  | 5 | `KeySize` in bits | `KeySize` |
  | 6 | Base62 leading zero bytes | `leading zero` |
  | 7 | Malformed input → `FormatException` | `FormatException` |
  | 8 | Argument-exception changes | `ArgumentNullException`, `ArgumentOutOfRangeException` |
  | 9 | Span `Sign` returns `false` | `destination is too small` |
  | 10 | Password message wording | `Passwords must` |
  | 11 | Invalid `ExtendedPasswordOptions` rejected, 0 invalid | `MaxConsecutiveChars` |
  | 12 | ChaCha20 constants and `KeyTypes.Ecdh` removed | `ChaCha20`, `KeyTypes.Ecdh` |
  | 13 | PEM import → `NotSupportedException` | `NotSupportedException` |
  | 14 | Base62 maximum input | `1,024 bytes`, `1024 bytes` |
  | 15 | Mismatched `D`/`X` and the D-5 length rule | `do not match`, `64-byte` |
  | 16 | No write to `CryptoProviderFactory.Default` | `CryptoProviderFactory.Default` |
  | 17 | Key ID through JWK conversion | `kid`, `KeyId` |
  | 18 | `TryRead` on fragmented streams | `TryRead` |

- **Doc-snippet harness (AC-102).** `build/Test-DocSnippets.ps1` extracts every fenced block tagged `csharp`, `cs` or `c#` from `README.md` and `src/*/README.md`. Each block is compiled in its own generated `net10.0` console project that references both released packages:
  - A block that declares a `namespace` or a top-level type becomes a source file, alongside an empty `Program.cs`.
  - Any other block becomes the `Program.cs` of top-level statements.
  - A block whose info string is suffixed `ignore` (for example `csharp ignore`) is skipped. The Documentation Agent may use this only for fragments that are not meant to compile, and must state why in the surrounding text.


### 3.4 Work items

_Every work item carries exactly one domain tag from:_
`frontend` `backend` `database` `redis` `security` `openfga` `docker` `tests` `ci` `docs`

_Production source and its tests are separate items: the component is `frontend`, its spec is
`tests`, the workflow job that runs it is `ci`._

**Conventions for every item below.**

- **Test-first chains (`**Test-first:** acceptance`).** Each chain is named C1 to C13, and §3.7 names its test seam.
  - A chain-specific stubs item exists only where the AC needs public surface that does not exist yet (WI-13, WI-26, WI-29).
  - Where the seam already exists at `555d31c`, the chain's stubs step is **WI-5**. WI-5 renames that surface and changes no behavior, so tests written after it compile against the final names and fail by assertion. Adding empty stubs items would only multiply hand-offs.
  - C1 is the exception: its tests must be red before the rename, so they precede WI-5. C1's precondition is WI-1/WI-2, which create no surface.
- **Unit-level test-first (C-13, GP-3, AC-132).** Every chain's tests item also contains the unit tests that drive its implementation item, committed before the implementation commit. The items that change library or sample behavior, with their preceding tests item, are:
  - WI-5 ← WI-4;
  - WI-12 ← WI-11;
  - WI-15 ← WI-14;
  - WI-17 ← WI-16;
  - WI-19 ← WI-18;
  - WI-21 ← WI-20;
  - WI-23 ← WI-22;
  - WI-25 ← WI-24;
  - WI-28 ← WI-27;
  - WI-31 ← WI-30;
  - WI-33 ← WI-32;
  - WI-35 ← WI-34;
  - WI-37 ← WI-36.

  For WI-35 and WI-37 (sample items), the preceding test is an in-process `integration` test, which AC-132 now permits for sample work items.
- **Commits.**
  - Test commits use `test(WI-n): … [AC-x, AC-y]`; implementation commits use `feat(WI-n)` or `fix(WI-n)`.
  - **Branches merge without squashing.** AC-121 and AC-132 read the commit order (RK-10).
  - A tests item is done when every new test compiles and fails by assertion ("red for the right reason"). The exceptions are AC-121's regression guards, which pass: AC-24, AC-32, AC-40, AC-41, AC-72, AC-73, AC-77, AC-82, AC-90, AC-110, AC-112, AC-135 and AC-141.- **Unchanged guardrails.** No test is weakened to make it pass. A finding in an item's output goes back to that item's owner.

**Tests removed or renamed (AC-5).** These names exist at `555d31c` and not at release, under the 3.3.1 mapping (prefix `ValhallaTech.Security.IdentityModel.Tests.`). Every other test name survives.

| Removed test | Behavior it covered | Why removed, and what replaces it |
| --- | --- | --- |
| `Tokens.EdDSA.AsymmetricAlgorithm.AsymmetricAlgorithmTests.KeySize_Expect32` | `KeySize` in bytes (32/57) | U5: now bits. Replaced by the AC-91 test. |
| `…AsymmetricAlgorithmTests.ImportFromEncryptedPem_WithPasswordString_ExpectNotImplementedException` | PEM import throws `NotImplementedException` | GP-6: now `NotSupportedException`. Replaced by the AC-129 test. |
| `…AsymmetricAlgorithmTests.ImportFromEncryptedPem_WithPasswordBytes_ExpectNotImplementedException` | Same | Same |
| `…AsymmetricAlgorithmTests.ImportFromPem_WithPasswordBytes_ExpectNotImplementedException` | Same | Same |
| `Tokens.EdDSA.AsymmetricAlgorithm.EdDsaSigningTests.VerifyWithOffsets_WhenInputLengthZero_ExpectArgumentException` | Zero-length message rejected by the offset overload | Contradicts R-31 and AC-85. Replaced by the AC-85 test. |
| `Tokens.EdDSA.EdDsaParametersTests.Validate_WhenEd25519SigningKey_ExpectValid` | 64-byte Ed25519 `D` accepted | D-5: now rejected. Replaced by the AC-44 test. |
| `Tokens.EdDSA.EdDsaParametersTests.Validate_WhenEd448SigningKey_ExpectValid` | 114-byte Ed448 `D` accepted | Same |
| `Extensions.ExtendedJsonWebKeyConverterTests.JsonWebKeyConverter_WithPrivateKey_ConvertFromEdDsaSecurityKey` | Default conversion exports `D` | S2: the default is now public-only. Replaced by the AC-26 and AC-28 tests. |
| `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ctor_WhenOptionsAccessorIsNull_ExpectNullOptions` | Null accessor accepted | Contradicts AC-80. Replaced by the AC-80 test. |
| `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ctor_WhenOptionsAccessorReturnsNullIsNull_ExpectNullOptions` | Null `.Value` accepted, which led to the late `NullReferenceException` (U3) | G-18: now rejected at construction (3.3.2). Replaced by a WI-32 unit test. |
| `AspNetCore.Identity.ExtendedPasswordValidatorTests.HasConsecutiveCharacters_WhenPasswordIsNullOrWhitespace_ExpectArgumentNullException` | Null, empty and whitespace input throw | U10: only null throws now. Replaced by a null-only test plus the AC-108 test. |
| `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxLength_ExpectNoMaxLengthAttribute` | 0 and -1 silently ignored | GP-9 Q-17: they now throw (AC-142(b)). Replaced by a `null`-only test and the AC-142 tests. |
| `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxConsecutiveChars_ExpectNoMaxConsecutiveCharsAttribute` | -1 silently ignored | Same |

Some kept tests lose theory rows or change assertions, without any name being removed:
- the 0 and -1 rows in `ValidateAsync_WhenExtendedPasswordOptionsAndPasswordIsNotTooLong_ExpectSuccess` and `…MaxConsecutiveCharactersValid_ExpectSuccess`;
- the 0 row in `ProcessIdentityPasswordRules_WhenExtendedOptionsWithMaxConsecutiveChars_ExpectMaxConsecutiveCharsAttribute`;
- the 32 → 256 `KeySize` assertions in `EdDsaSecurityKeyTests`;
- the byte → bit data in `LegalKeySizes_ExpectCorrectValues`;
- the reference-equality and `PublicKeyParameter.Should().BeNull()` assertions in `EdDsaCreationTests.FromParameters_When*PrivateKey_ExpectCorrectParameters`, which change because `Create` now copies the parameters and derives the public key.

The item making each change states it in its PR.

##### Phase 0 — Foundation (serial)

#### WI-1 — Establish the .NET 10 build foundation and move runtime and sample dependencies to latest stable

- **domain:** `backend`
- **depends-on:** none
- **satisfies:** AC-1, AC-2 (sample), AC-3, AC-45, AC-46, AC-47, AC-48, AC-49 (libraries and sample)
- **change:**
  - Add `global.json` and the root `Directory.Build.props` (3.3.4).
  - Add `Directory.Packages.props` (CPM) and remove every `TargetFrameworks` and inline `Version`.
  - **Libraries:** BouncyCastle.Cryptography to latest stable (2.7.x). Microsoft.IdentityModel.Tokens and .JsonWebTokens to latest stable 8.x. Add SimpleBase at latest stable. GP-2 chose the library; GP-10 removed the 5.6.4 pin, so it follows R-17 like every other dependency, and the AC-70 and AC-124 to AC-127 checks catch any regression in a newer version. Remove `SecurityCodeScan.VS2019` from both.
  - **Sample:** Microsoft.AspNetCore.* and EF Core InMemory to latest 10.0.x. Remove `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` and its `AddRazorRuntimeCompilation()` call; Hot Reload covers it, and removing it drops the Roslyn payload.
  - Replace `ScottBrady.IdentityModel.sln` with an `.slnx` holding the same projects (renamed in WI-5).
  - Fix compile breaks in `src/` and `samples/` from the Microsoft.IdentityModel 7 → 8 move. No behavior change is intended.
- **done when:** `dotnet build -c Release` succeeds with only a 10.0 SDK (AC-3). Every `lib/` folder in the packs is `net10.0` (AC-1). `--vulnerable`, `--deprecated` and `--outdated` (direct, stable) report nothing for the libraries and sample (AC-45, AC-46, AC-49). `rg -uu SecurityCodeScan` finds 0 matches (AC-47). Every direct license is an OSI-approved SPDX expression (AC-48).

#### WI-2 — Migrate the test project to .NET 10 and AwesomeAssertions

- **domain:** `tests`
- **depends-on:** WI-1
- **satisfies:** AC-2 (tests), AC-4, AC-45, AC-46, AC-48, AC-49 (tests)
- **change:**
  - Retarget through the props.
  - **Resolves D-1 (Fred, 2026-10-25: "We're using AwesomeAssertions"):** replace FluentAssertions 6.12.2 with AwesomeAssertions **9.6.0** (Apache-2.0), its latest stable. Change `using FluentAssertions;` to `using AwesomeAssertions;` in the 19 files, and fix every 6→7 API rename the compiler reports across the 157 `.Should()` calls. Shouldly and FluentAssertions 7.2.2 lose: Shouldly needs every assertion rewritten, and 7.2.2 is not FluentAssertions' latest stable (AC-49).
  - Remove **AutoFixture**. It is used only in `AsymmetricAlgorithmTests.cs`, to build `PbeParameters`; replace that with `new PbeParameters(PbeEncryptionAlgorithm.Aes256Cbc, HashAlgorithmName.SHA256, 1)`. This removes both S8 High advisories, which is why it is removed rather than upgraded: 4.18.1 is already AutoFixture's latest stable.
  - Bump `xunit` (stays on the 2.9.x package ID), `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk`, `Moq`, `System.IdentityModel.Tokens.Jwt` and `Microsoft.AspNetCore.Authentication.JwtBearer` to latest stable. Add `coverlet.collector`.
  - Move `TestJwtBearerHandler` off the `ISystemClock` constructor if .NET 10 removed it.
  - xunit.v3 was considered and lost for now: changed lifetimes and an executable test host are not a mechanical swap under the deadline (RK-12).
- **done when:** All 250 tests pass on runtime 10.0.x (AC-4). The test-name set is identical to `555d31c`'s, and no assertion is weakened. `--vulnerable`, `--deprecated` and `--outdated` are clean for the test project.

#### WI-3 — Move CI to .NET 10 with least-privilege tokens

- **domain:** `ci`
- **depends-on:** WI-1, WI-2
- **satisfies:** AC-6, AC-7, AC-58 (R-21 lands early because it is one block per file)
- **change:**
  - `dotnetcore.yml`: top-level `permissions: contents: read`. `actions/setup-dotnet` with `dotnet-version: '10.0.x'`. Restore, build and test on the **kept** `ubuntu-latest`/`macos-latest`/`windows-latest` matrix. Pack and upload the artifact on ubuntu only.
  - `codeql-analysis.yml`: kept. `dotnet-version: '10.0.x'`; job permissions `contents: read`, `actions: read`, `security-events: write`.
  - **Network.** The test job needs outbound HTTPS to `registry.npmjs.org` (WI-36's `Category=Network` tests, AC-138 and AC-143) and to jsDelivr (LibMan restore on build, WI-37). GitHub-hosted runners have both. The job does not filter out `Category=Network` tests. A registry outage fails the run loudly, naming the URL, and the run is re-run, never muted.
  - **Resolves D-10.** The three-OS matrix stays because consumers run the libraries on all three, and its cost is nil on a public repository. CodeQL stays because its dataflow analysis finds things the in-build analyzers do not.
- **done when:** AC-6 and AC-58 pass, and a branch run is green on all three OSes with runtime 10.0.x. WI-46 re-checks AC-7 on the release commit.

#### WI-4 — Write the failing identity and API-shape tests (chain C1)

- **domain:** `tests`
- **depends-on:** WI-2 (C1 precondition; no new surface)
- **satisfies:** AC-13, AC-14, AC-95, AC-96
- **change:** Add `PublicApiShapeTests`, reflecting over `typeof(Base62).Assembly` and `typeof(NewPasswordTagHelper).Assembly`: exported-type namespaces, `GetName().Name`, assembly-attribute strings including `InternalsVisibleTo`, removed constant names, and `"OKP"` constant names. Unit tests per assertion.
- **done when:** The tests compile at the pre-rename state and fail by assertion.

#### WI-5 — Rename to the ValhallaTech identity and remove the leftover constants (implements C1; stubs step for C3, C5–C10, C13)

- **domain:** `backend`
- **depends-on:** WI-4
- **satisfies:** AC-10, AC-11, AC-13, AC-14, AC-15, AC-16, AC-17, AC-18, AC-95, AC-96
- **change:**
  - `git mv` the library and sample directories and project files, and the solution, per 3.3.1. Update every namespace in `src/` and `samples/`.
  - Add `src/Directory.Build.props` with the 3.3.5 metadata, `Version` 1.0.0, and the packed `LICENSE`. The upstream `icon.png` stays packed until WI-42.
  - Rename `InternalsVisibleTo` to `ValhallaTech.Security.IdentityModel.Tests` (keeping `DynamicProxyGenAssembly2`).
  - Delete the six ChaCha20 constants. Rename `KeyTypes.Ecdh` to `KeyTypes.OctetKeyPair`, and update the converter.
  - Add the 3.3.5 notice to every derived `.cs` file in `src/` and `samples/`.
  - Retitle the sample's layout strings.
  - No behavior change beyond AC-95 and AC-96.
- **done when:** WI-4's tests pass. AC-10, AC-11, AC-16 and AC-17 hold on the packed nuspecs. AC-15 and AC-18 `rg` checks return 0 misses. The full suite is green with WI-6 in the same PR.

#### WI-6 — Rename the test project and test namespaces

- **domain:** `tests`
- **depends-on:** WI-5 (same PR, so `master` never sees a broken build)
- **satisfies:** AC-4, AC-5 (name mapping)
- **change:** `git mv` the test project to `test/ValhallaTech.Security.IdentityModel.Tests`, rename namespaces and usings, and add the 3.3.5 notice to derived test files. The WI-4 tests change only their `using` lines.
- **done when:** The suite is green, and the test-name set equals `555d31c`'s under the 3.3.1 mapping.

#### WI-7 — Add CSharpier and reformat once

- **domain:** `ci`
- **depends-on:** WI-6
- **satisfies:** AC-120 (R-49 is 1.x, but it lands in Phase 0 so the one-time reformat cannot conflict with parallel lanes; RK-9)
- **change:**
  - **Resolves D-7, formatting half: CSharpier**, Fred's preference. It is zero-configuration, deterministic, and one dotnet local tool that Renovate keeps current. `dotnet format whitespace` lost because it is style-configurable through `.editorconfig`, which invites the analyzer-inventory sprawl C-17 rules out.
  - Add `.config/dotnet-tools.json` with `csharpier` at latest stable, and a default `.csharpierrc.json`.
  - One `style:` commit containing only `dotnet csharpier format .` output.
  - Add a CI step `dotnet tool restore && dotnet csharpier check .` on the ubuntu job.
- **done when:** AC-120 passes: the check passes on the branch and fails on a one-line probe. Build and test results are unchanged by the reformat commit.

#### WI-8 — Write the failing nullability test (chain C2)

- **domain:** `tests`
- **depends-on:** WI-7
- **satisfies:** AC-97
- **change:** A `NullabilityInfoContext` sweep over every public parameter, return, property and field of reference type in both assemblies.
- **done when:** It fails by assertion, because nullability is `Unknown` with annotations off.

#### WI-9 — Turn on analysis, nullable annotations and XML docs across the existing surface (implements C2)

- **domain:** `backend`
- **depends-on:** WI-8
- **satisfies:** AC-55, AC-56, AC-57, AC-97, AC-98, AC-99
- **change:**
  - In `src/Directory.Build.props`: `Nullable=enable`, `GenerateDocumentationFile=true`, `AnalysisLevel=latest-recommended`, `AnalysisModeSecurity=All`, and no `NoWarn`.
  - Annotate and document every existing public or protected member of both libraries, using `<inheritdoc/>` on overrides.
  - Fix every analyzer finding in source. A member-level `[SuppressMessage]` with a written justification is allowed only for a demonstrated false positive; no project-level suppression.
  - From here on, every new public member is born documented and annotated, because warnings are errors.
- **done when:** WI-8 passes. The AC-55 (CA5351) and AC-99 (CS1591) probes report. The AC-56 MSBuild inspection is clean. AC-57 shows 0 warnings. AC-98's doc check passes.

#### WI-10 — Route vulnerability reports to the fork's private reporting

- **domain:** `docs`
- **depends-on:** none (runs in parallel with Phase 0)
- **satisfies:** AC-19 (AC-20 needs U-2)
- **change:** Rewrite `SECURITY.md`:
  - link to `https://github.com/IronSloth/IdentityModel-Supa/security/advisories/new`;
  - add a supported-versions line (1.x);
  - no email address and no upstream contact.
- **done when:** AC-19's search finds the link, 0 `scottbrady91` matches and 0 email addresses.

##### Phase 1 — Lane A: EdDSA keys and crypto provider

#### WI-11 — Write the failing crypto-isolation tests (chain C3)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **satisfies:** AC-21, AC-22, AC-23, AC-24, AC-25, AC-32, AC-81
- **change:**
  - Tests in an xUnit collection `GlobalCryptoState` (`DisableParallelization = true`), with a fixture that saves and restores `CryptoProviderFactory.Default.CustomCryptoProvider`.
  - Add a unit test that two keys never share a factory instance.
  - AC-24 and AC-32 are regression guards.
- **done when:** AC-21, 22, 23, 25 and 81 fail by assertion; AC-24 and AC-32 pass.

#### WI-12 — Isolate each key's crypto provider (implements C3)

- **domain:** `backend`
- **depends-on:** WI-11
- **satisfies:** AC-21, AC-22, AC-23, AC-24, AC-25, AC-32, AC-81
- **change:**
  - In `EdDsaSecurityKey`'s ctor, assign a new per-key `CryptoProviderFactory` with a shared, stateless, static EdDSA provider and `CacheSignatureProviders = false`. Remove the line-16 write.
  - In `ExtendedCryptoProvider`, add the `args` guards and `JsonWebKey` acceptance from 3.3.2.
  - The provider stays internal; AC-81 reaches it through `key.CryptoProviderFactory.CustomCryptoProvider`.
  - Signature-provider caching is off because `EdDsaSignatureProvider` is a stateless wrapper (`EdDsa` creates a signer per operation), so caching saves nothing and per-key caches would cost memory under C1.
- **done when:** WI-11 is green, including AC-25's 1,000 concurrent keys. The full suite is green.

#### WI-13 — Stub the private-key export opt-in (chain C4)

- **domain:** `backend`
- **depends-on:** WI-9
- **satisfies:** AC-28 (surface only)
- **change:** Add `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKeyIncludingPrivateKey(EdDsaSecurityKey key)`, documented, returning `new JsonWebKey()`. It does not throw (**resolves D-4**, 3.3.2).
- **done when:** It builds, and the signature matches 3.3.2.

#### WI-14 — Write the failing JWK tests (chain C4)

- **domain:** `tests`
- **depends-on:** WI-13
- **satisfies:** AC-26, AC-27, AC-28, AC-29, AC-82, AC-93, AC-94
- **change:** Acceptance and unit tests against the converter and `JsonWebTokenHandler`. AC-82 is a guard.
- **done when:** AC-26, 27, 28, 29, 93 and 94 fail by assertion; AC-82 passes.

#### WI-15 — Make JWK export public-only and carry the key ID (implements C4)

- **domain:** `backend`
- **depends-on:** WI-14, WI-12, WI-17
- **satisfies:** AC-26, AC-27, AC-28, AC-29, AC-82, AC-93, AC-94
- **change:**
  - The default conversion emits `X`, plus `D` only through the opt-in. `X` comes from the library-held public key, so private-only keys export their derived public key after WI-17.
  - `Kid` is copied both ways, `CryptoProviderFactory = key.CryptoProviderFactory`, and a null `key` throws.
- **done when:** WI-14 is green.

#### WI-16 — Write the failing key-material tests (chain C5)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **satisfies:** AC-30, AC-31, AC-42, AC-43, AC-44, AC-73, AC-87, AC-88, AC-91, AC-92, AC-128, AC-129, AC-135
- **change:**
  - Acceptance and unit tests.
  - AC-30 and AC-31 use the reflection buffer walk, skipping BouncyCastle-assembly types.
  - AC-42 uses the full combination grid with a recorded seed.
  - AC-44 asserts rejection (D-5).
  - AC-73 and AC-135 are guards.
- **done when:** Every non-guard test fails by assertion; the guards pass.

#### WI-17 — Copy, validate, size, zero and import keys correctly (implements C5)

- **domain:** `backend`
- **depends-on:** WI-16
- **satisfies:** AC-30, AC-31, AC-42, AC-43, AC-44, AC-73, AC-87, AC-88, AC-91, AC-92, AC-128, AC-129, AC-135
- **change:** In `EdDsa.cs` and `EdDsaParameters.cs`, implement the 3.3.2 rows:
  - library-owned copies in `Create`;
  - public-key derivation;
  - `Validate()` with exact lengths, a shared construction path, and the constant-time pair check (**D-5: reject**);
  - `KeySize` and `LegalKeySizes` in bits;
  - the `Dispose(bool)` zeroing;
  - PEM members throwing `NotSupportedException`.

  Remove the TODO at `EdDsaParameters.cs:16`. Keep the per-operation signer (G-16).
- **done when:** WI-16 is green, and the full suite is green.

#### WI-18 — Write the failing sign/verify argument tests (chain C6)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **satisfies:** AC-84, AC-85, AC-89, AC-90
- **change:** Acceptance and unit tests, including the RFC 8032 §7.1 TEST 1 vector for AC-85. AC-90 is a guard.
- **done when:** AC-84, 85 and 89 fail by assertion; AC-90 passes.

#### WI-19 — Validate offsets and sign or verify without copies (implements C6)

- **domain:** `backend`
- **depends-on:** WI-18, WI-17 (both edit `EdDsa.cs`)
- **satisfies:** AC-84, AC-85, AC-89, AC-90. Also the P-e, P-f and P-g improvements measured by WI-43 (AC-64 to AC-66, AC-68).
- **change:**
  - The offset `Verify` validates every argument with `ParamName`, accepts a zero-length message, and feeds BouncyCastle through `BlockUpdate(ReadOnlySpan<byte>)` with no `Skip`/`Take`/`ToArray`.
  - In the signature provider, span `Sign` returns `false`/0 on a short destination and signs from the span. Offset `Sign` validates its arguments and calls `BlockUpdate(input, offset, count)` with no copy.
- **done when:** WI-18 is green, and the suite is green.

##### Phase 1 — Lane B: encoders and streams

#### WI-20 — Write the failing Base62 tests (chain C7)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **satisfies:** AC-33, AC-34, AC-35, AC-36, AC-79, AC-124, AC-125, AC-126
- **change:**
  - Acceptance and unit tests with fixed seeds.
  - Run AC-124's 1 MiB case on a task with a bounded wait, so the `555d31c` behavior (minutes) fails by assertion instead of hanging the run.
- **done when:** All fail by assertion.

#### WI-21 — Replace Base62 with capped SimpleBase (implements C7)

- **domain:** `backend`
- **depends-on:** WI-20
- **satisfies:** AC-33, AC-34, AC-35, AC-36, AC-79, AC-124, AC-125, AC-126, AC-136. Feeds AC-60, AC-61, AC-70 and AC-127.
- **change:**
  - Implement the 3.3.2 `Encode`/`Decode` rows on `SimpleBase.Base62.Default` at its latest stable version (**D-3 as fixed by GP-2; version per GP-10**).
  - XML docs state the caps (`1,024` bytes; `1,376` characters) and `ArgumentException`/`FormatException`.
  - Keep `CharacterSet`.
- **done when:** WI-20 is green. AC-136 holds in the built XML doc file.

#### WI-22 — Write the failing Base16 tests (chain C8)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **satisfies:** AC-37, AC-38, AC-39, AC-40, AC-41
- **change:** Acceptance and unit tests. AC-40 and AC-41 are guards.
- **done when:** AC-37, 38 and 39 fail by assertion; AC-40 and AC-41 pass.

#### WI-23 — Re-implement Base16 on the runtime's hex APIs (implements C8)

- **domain:** `backend`
- **depends-on:** WI-22
- **satisfies:** AC-37 to AC-41. Feeds AC-62, AC-63 and AC-69.
- **change:** `Convert.ToHexStringLower` and `Convert.FromHexString`, with the null guard. Delete the lookup table.
- **done when:** WI-22 is green.

#### WI-24 — Write the failing stream-read tests (chain C9)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **satisfies:** AC-71, AC-72, AC-83, AC-86
- **change:** Use a test stream that returns at most 1 byte per `Read`. AC-72 is a guard.
- **done when:** AC-71, 83 and 86 fail by assertion; AC-72 passes.

#### WI-25 — Read streams until the requested length arrives (implements C9)

- **domain:** `backend`
- **depends-on:** WI-24
- **satisfies:** AC-71, AC-72, AC-83, AC-86
- **change:** The 3.3.2 `TryRead` row.
- **done when:** WI-24 is green, and the existing `TryRead` tests are unchanged and green.

##### Phase 1 — Lane C: password validation (AspNetCore)

#### WI-26 — Stub the error describer and the validator constructor (chain C11)

- **domain:** `backend`
- **depends-on:** WI-9
- **satisfies:** AC-111 (surface only)
- **change:**
  - Add `ExtendedIdentityErrorDescriber` with the two virtual methods, each returning `new IdentityError()`.
  - Add the `ExtendedPasswordValidator(IdentityErrorDescriber? errors = null)` ctor; it stores the value and does not use it.
  - **Resolves D-11** (3.3.2).
- **done when:** It builds, and the surface matches 3.3.2.

#### WI-27 — Write the failing validator and message tests (chain C11)

- **domain:** `tests`
- **depends-on:** WI-26
- **satisfies:** AC-106, AC-107, AC-108, AC-109, AC-110, AC-111
- **change:**
  - Acceptance and unit tests.
  - AC-111 registers a French describer subclass through `AddErrorDescriber<T>()`.
  - AC-110 is a regression guard (AC-121).
- **done when:** AC-106 to AC-109 and AC-111 fail by assertion; AC-110 passes.

#### WI-28 — Use the describer, fix whitespace, scan linearly (implements C11)

- **domain:** `backend`
- **depends-on:** WI-27
- **satisfies:** AC-106 to AC-111. Feeds AC-67 (P-h).
- **change:** The 3.3.2 validator, describer, default-message and `HasConsecutiveCharacters` rows. The `Regex` is replaced by a single pass.
- **done when:** WI-27 is green, and the suite is green.

#### WI-29 — Stub the one-step registration (chain C12)

- **domain:** `backend`
- **depends-on:** WI-9
- **satisfies:** AC-74 (surface only)
- **change:** Add `ExtendedPasswordIdentityBuilderExtensions.AddExtendedPasswordRules(this IdentityBuilder, Action<ExtendedPasswordOptions>)`, documented, returning `builder` and registering nothing.
- **done when:** It builds, and the surface matches 3.3.2.

#### WI-30 — Write the failing registration and options-validation tests, plus the DryIoc test (chain C12)

- **domain:** `tests`
- **depends-on:** WI-29
- **satisfies:** AC-74, AC-75, AC-76, AC-77, AC-142
- **change:**
  - Acceptance tests, plus unit tests for each options rule and message.
  - **One DryIoc test (D-8, Fred, 2026-10-25: "Yes, sample + one test").** It builds the AC-74 `ServiceCollection` (`AddIdentityCore<IdentityUser>()` plus the single step), creates the provider through `DryIoc.Microsoft.DependencyInjection`'s `DryIocServiceProviderFactory`, and asserts that the resolved `UserManager<IdentityUser>` reports `PasswordTooLong` for a 65-character password.
  - The test project takes `DryIoc.Microsoft.DependencyInjection` **6.2.0** and `DryIoc.dll` **5.4.3** as test-only references; the libraries take no DryIoc dependency.
  - AC-77 is a guard.
- **done when:** AC-74, 75, 76, 142 and the DryIoc test fail by assertion; AC-77 passes.

#### WI-31 — Implement the single step and reject invalid options on both paths (implements C12)

- **domain:** `backend`
- **depends-on:** WI-30, WI-28 (both edit `ExtendedPasswordValidator.cs`)
- **satisfies:** AC-74, AC-75, AC-76, AC-77, AC-142
- **change:**
  - Implement the 3.3.2 `AddExtendedPasswordRules` and `ExtendedPasswordOptions` rule rows.
  - One internal rule evaluator feeds three consumers: the `IValidateOptions<IdentityOptions>` with `ValidateOnStart`, the validator's first-use check, and `NewPasswordTagHelper.Process`. Each throws `OptionsValidationException` naming the setting.
- **done when:** WI-30 is green, including the DryIoc test.

##### Phase 1 — Lane D: null-argument sweep

#### WI-32 — Write the failing null-guard sweep (chain C10)

- **domain:** `tests`
- **depends-on:** WI-9 (annotations decide which parameters are swept; stubs step: WI-5)
- **satisfies:** AC-78, AC-80
- **change:**
  - A reflection-driven sweep over every public ctor and method of both assemblies (not `Try*`), with a fixture-value table per parameter type. A parameter type with no fixture **fails** with "no fixture for type X"; it is never skipped, so members added by later items are swept.
  - Add the AC-80 test, and the replacement for the removed null-`.Value` test.
- **done when:** The sweep fails by assertion, for example on `Base62.Encode`'s `ParamName` and `ConvertFromEdDsaSecurityKey(null)`. AC-80 fails.

#### WI-33 — Close the remaining null guards (implements C10)

- **domain:** `backend`
- **depends-on:** WI-32, WI-12, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-28, WI-31
- **satisfies:** AC-78, AC-80
- **change:**
  - `ArgumentNullException.ThrowIfNull` wherever the sweep still fails after the lanes, including `ByteArrayExtensions.Combine`.
  - The `NewPasswordTagHelper` ctor guards from 3.3.2.
- **done when:** WI-32 is green, and the full suite is green.

##### Phase 1 — Lane E: sample

#### WI-34 — Create the sample test project and write the failing hosting tests (chain C13a)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5; the routes exist)
- **satisfies:** AC-52, AC-53, AC-54, AC-112
- **change:**
  - New `test/ValhallaTech.Security.IdentityModel.Samples.Tests`, using `Microsoft.AspNetCore.Mvc.Testing` and `WebApplicationFactory<Program>`, added to the `.slnx`.
  - AC-53 and AC-54 run in a non-parallel collection that resets `IdentityModelEventSource.ShowPII`.
  - AC-54's probe is an `IStartupFilter` that appends a terminal middleware throwing `InvalidOperationException("ac54-probe")`, after the app's own pipeline.
- **done when:** AC-52, 53 and 54 fail by assertion. AC-112, a regression guard (AC-121), passes.

#### WI-35 — Move the sample to minimal hosting and DryIoc, and fix its key and diagnostics (implements C13a)

- **domain:** `backend`
- **depends-on:** WI-34, WI-31
- **satisfies:** AC-52, AC-53, AC-54, AC-112, AC-113, AC-122 (sample half)
- **change:**
  - `Program.cs` uses `WebApplication.CreateBuilder` and ends with `public partial class Program;`. Delete `Startup.cs`.
  - **D-8, DryIoc part (Fred, 2026-10-25):** `builder.Host.UseServiceProviderFactory(new DryIocServiceProviderFactory())`, with every registration still made on `IServiceCollection`.
    - References: `DryIoc.Microsoft.DependencyInjection` **6.2.0** and `DryIoc.dll` **5.4.3**. Both are their packages' latest stable; `8.0.0-preview-04` and `6.0.0-preview-09` are excluded because G-9 says "stable".
    - `DryIoc.dll` is referenced directly so the container in use is its latest stable, not the adapter's declared minimum. AC-49 and Renovate then govern it.
    - If either fails on `net10.0`, that is reported to Fred; no preview is substituted.
  - `SampleOptions` holds a signing key (`D` and `X`) and a separate validation key built from `X` only.
  - `ShowPII` and the developer exception page apply only when `IsDevelopment()`. Otherwise use `AddProblemDetails()`, `UseExceptionHandler()` and `UseHsts()`.
  - Password setup uses `AddExtendedPasswordRules` (R-51).
- **done when:** WI-34 is green. AC-113's search finds `WebApplication.CreateBuilder` and no `Startup`. AC-122's sample search is clean.

#### WI-36 — Write the failing sample-page tests (chain C13b)

- **domain:** `tests`
- **depends-on:** WI-34
- **satisfies:** AC-114, AC-138, AC-141, AC-143
- **change:**
  - HTML-parsing tests over GET `/` and GET `/Home/PasswordRules`, fetching every `<script src>` and `<link href>` through the same client.
  - Set V for AC-138 and AC-143 is built at test time. Read the library name and version from the sample's `libman.json`, download that version's tarball from `https://registry.npmjs.org/<name>/-/<name>-<version>.tgz`, and hash its files. Keying on the manifest means a Renovate bump needs no test edit. While `libman.json` does not exist (the introducing commit), the test fails on an assertion that the manifest declares `bootstrap`, never on an I/O exception. AC-140's release-date check confirms the version is npm `latest`.
  - **Network (AC conventions, AC-138's rule).** These tests need HTTPS access to `registry.npmjs.org`, and carry the xUnit trait `Category=Network`. If the metadata or tarball request fails, the test fails with "npm registry unreachable" and the failing URL. It never passes, skips, or reports inconclusive. A cached tarball counts only if its SHA-256 matches the registry's published `dist.integrity` for that version.
  - **Named test file (AC-137's closed exclusion).** Every AC-138 and AC-143 assertion that names the forbidden library lives in exactly one file, `test/ValhallaTech.Security.IdentityModel.Samples.Tests/ForbiddenClientLibraryTests.cs`. This is the path AC-137 excludes. The file only names the library as the thing forbidden. It contains no library banner, and loads or imports nothing (AC-137's separate check on it). No second file of that name exists.
  - AC-141 is a guard.
- **done when:** AC-114, AC-138 and AC-143 fail by assertion: the baseline serves the old library file and Bootstrap 4.3.1, whose hash is not 5.x's. AC-141 passes.

#### WI-37 — Move the sample UI to Bootstrap 5, CSS and JS bundle, through LibMan, with no jQuery (implements C13b)

- **domain:** `frontend`
- **depends-on:** WI-36
- **satisfies:** AC-50, AC-51, AC-114, AC-133, AC-137, AC-138, AC-139, AC-140, AC-141, AC-143
- **change:**
  - **Delivery (GP-10: "Ship it, fix the checks").** Add `samples/ValhallaTech.Security.IdentityModel.Samples.AspNetCore/libman.json` with:
    - provider `jsdelivr`;
    - one library, `bootstrap@<latest stable 5.x>` (5.3.8 on 2026-10-25);
    - files `dist/css/bootstrap.min.css` and `dist/js/bootstrap.bundle.min.js`;
    - destination `wwwroot/lib/bootstrap/`.

    Add a `Microsoft.Web.LibraryManager.Build` reference so files are restored on build, and gitignore `wwwroot/lib/`. Files are restored byte for byte from the npm package (jsDelivr `/npm/` path), so both are in set V.
  - **Served unmodified (AC-143).** No bundling, minification or rewriting step touches `wwwroot/lib/`. Fingerprinted URLs (`asp-append-version` or `MapStaticAssets`) are fine, because they change the URL and not the bytes. Response compression must decompress to identical bytes.
  - **Remove the old files.** Delete `wwwroot/js/jquery.min.js`, `wwwroot/js/bootstrap.bundle.min.js` (4.3.1) and `wwwroot/css/bootstrap.css` (4.3.1). Nothing in the sample's own files names the removed library afterwards (AC-137 check 2).
  - **`_Layout.cshtml`.**
    - `<link>` to `~/lib/bootstrap/dist/css/bootstrap.min.css`, and one `<script>` to `~/lib/bootstrap/dist/js/bootstrap.bundle.min.js`.
    - A Bootstrap 5 navbar whose toggler uses `data-bs-toggle="collapse"` and `data-bs-target` (Bootstrap 4's `data-toggle` does not work in 5), so the mobile menu works (G-25).
    - Bootstrap 5 class changes, such as `text-right` → `text-end`.
  - **`PasswordRules.cshtml`.** Drop the explicit `id="password"`, so `asp-for` generates matching `for` and `id`. The inline script uses the generated id, and contains no `$(`.
  - **Why LibMan with a Renovate regex manager (closes the coordinator's AC-134 gap; Renovate side in WI-39).** It is the smallest option that keeps a declared version, unmodified files, and no new toolchain: one JSON manifest, one MSBuild package, and one `customManagers` block in `renovate.json`. Renovate has no LibMan manager, so the regex manager is what makes `libman.json` visible. Alternatives lost:
    - an npm `package.json` plus a copy step: Renovate reads it natively, but it adds Node and a lockfile to every build and CI leg of a .NET repository, plus an MSBuild copy target to maintain;
    - a CDN `<link>`/`<script>` with SRI: Renovate cannot recompute the integrity hash;
    - committed vendored files with no manifest: invisible to update tooling (S6, AC-134).
  - **D-8, UI part:** fixed by GP-8 and GP-10.
- **done when:** WI-36 is green, including AC-143's hash match. AC-140: the version in `libman.json` equals npm `latest`, major 5. AC-50, AC-51, AC-133 and AC-139 lookups are clean. AC-137 checks 1, 2 and 3 return 0 under its closed exclusion list, and the named test file passes its separate check.

##### Phase 1 — Lanes F and G: measurement and automation

#### WI-38 — Build the BenchmarkDotNet harness with a source-level baseline

- **domain:** `tests`
- **depends-on:** WI-9
- **satisfies:** AC-59, AC-69, AC-70, AC-123 (harness only; numbers come from WI-43)
- **change:**
  - **Resolves D-6: BenchmarkDotNet** (latest stable, MIT). It reports mean, error as a 99.9% confidence half-width, and allocations natively.
  - `perf/ValhallaTech.Security.IdentityModel.Benchmarks` (`net10.0`, `IsPackable=false`, in the `.slnx`). `Baseline/` holds verbatim `555d31c` copies of `Base62`, `Base16`, the offset-`Verify` path, the provider's span and offset `Sign` paths, and the consecutive-character check. Each copy carries the 3.3.5 notice and the source commit.
  - Benchmarks: every T path and size; Ed448 P-e/f/g; AC-69's 8 and 64 KiB inputs; AC-70's calibration `B`. Use `[MemoryDiagnoser]`, CSV and GitHub-markdown exporters, and a machine-name column.
  - Why a source copy: baseline and after then run in one process, on one runtime and one dependency graph, so only the code differs, which is the point of the measurement rules. Alternatives lost:
    - referencing upstream 4.1.0 from nuget.org: it pulls Microsoft.IdentityModel 7.x into a process that loads 8.x, and is not provably `555d31c`;
    - a `git worktree` build: two processes, two dependency graphs.
- **done when:** The project builds in CI, and `--list flat` enumerates all 16 T pairs plus the Ed448, scaling and calibration cases.

#### WI-39 — Configure Renovate for every direct dependency

- **domain:** `ci`
- **depends-on:** WI-3, WI-7, WI-37
- **satisfies:** AC-134, AC-118 (with U-4), AC-119 (with U-3)
- **change:**
  - **Resolves D-7, dependency half: Renovate**, Fred's lean and the tool assetvault-io already uses. The deciding fact: neither Renovate nor Dependabot has a LibMan manager, but only Renovate offers a custom regex manager that can be pointed at `libman.json`. Without it, AC-134 fails.
  - `renovate.json` extends `config:recommended`, `:dependencyDashboard` and `:semanticCommits`, with `minimumReleaseAge: "7 days"` (as in assetvault-io) and `vulnerabilityAlerts.enabled: true`.
  - Groups: `Microsoft.IdentityModel.*` with `System.IdentityModel.*`; `Microsoft.AspNetCore.*` with `Microsoft.EntityFrameworkCore.*`.
  - Rule: `bootstrap` gets `allowedVersions: "<6.0.0"` (GP-8).
  - A `customManagers` entry with `customType: "regex"`, matched to `samples/**/libman.json` through the file-pattern key of the Renovate schema current at implementation. Its match string captures `depName` and `currentValue` from `"library": "<name>@<version>"`, with `datasourceTemplate: "npm"`. A Renovate PR then bumps `libman.json`; the build restores the new files; and WI-36's hash tests re-verify them against the new tarball in the same PR's CI.
  - Native managers cover `Directory.Packages.props`, `global.json`, `.config/dotnet-tools.json` and the workflow `uses:`.
- **done when:** The config validates (`renovate-config-validator`). Once U-4 is done, the dependency dashboard lists every item of AC-134's inventory, including `bootstrap` under the regex manager.

##### Phase 2 — Release (1.0.0)

#### WI-40 — Write `CHANGELOG.md` with the 1.0.0 breaking changes

- **domain:** `docs`
- **depends-on:** WI-33
- **satisfies:** AC-104, AC-105
- **change:**
  - Keep a Changelog format, with `## [1.0.0] - <release date, ISO>`.
  - A **Breaking** list covers every 3.3.2 row marked breaking, using at least one 3.3.5 phrase per topic.
  - Also record the `\n` counting change and the new SimpleBase dependency.
- **done when:** The AC-104 phrase check finds all 18 topics. AC-105 holds for every tag or feed version.

#### WI-41 — Write the migration guide from upstream 4.1.0

- **domain:** `docs`
- **depends-on:** WI-33
- **satisfies:** AC-130 (AC-131 through WI-45 and WI-47)
- **change:**
  - `docs/migration/scottbrady-4.1.0-to-1.0.0.md`, with Before/After/Action rows for the 18 topics and 6 APIs (3.3.5).
  - It must also cover:
    - adding the feed `https://nuget.pkg.github.com/IronSloth/index.json` (D-2, GP-10) to `NuGet.config`. That means a `packageSourceMapping` entry for `ValhallaTech.*` and a `read:packages` token supplied through an environment variable, never committed;
    - raising direct BouncyCastle.Cryptography and Microsoft.IdentityModel.* references to the library's minimums (NU1605, RK-8);
    - converting JWKS keys (3.3.3);
    - the `TokenValidationParameters.CryptoProviderFactory` note.
  - It does not mention the sample's client-library change. That is not a package change, and AC-137 excludes only `specs/` and `CHANGELOG.md` from its text search.
- **done when:** AC-130's phrase match finds 0 uncovered items, and every row's three cells are non-empty.

#### WI-42 — Pack the ValhallaTech logo as the package icon

- **domain:** `backend`
- **depends-on:** U-1, WI-5
- **satisfies:** AC-12
- **change:** `PackageIcon` is built from `assets/valhallatech-logo.png`, packed as `icon.png`. Delete the root `icon.png`.
- **done when:** AC-12's checks hold: PNG signature, IHDR at least 128 pixels, and the hash equals the supplied file and differs from `555d31c`.

#### WI-43 — Measure baseline against the release candidate and commit the report

- **domain:** `tests`
- **depends-on:** WI-38, WI-33
- **satisfies:** AC-59, AC-60, AC-61, AC-62, AC-63, AC-64, AC-65, AC-66, AC-67, AC-68, AC-69, AC-70, AC-123, AC-127
- **change:**
  - Run the harness `-c Release` on Fred's workstation, the machine behind the GP-2 and Q-23 figures, on the release-candidate commit with one .NET 10.0.x runtime.
  - Commit the 3.3.5 files under `perf/results/1.0.0/`.
- **done when:**
  - The AC-59 parse passes.
  - AC-68 shows 0 regressions; AC-70's means are within budget; AC-127 shows a time improvement.
  - A regression goes back to the owning item as a finding. A budget overrun is reported to Fred, not tuned away. AC-60 to AC-67 (R-23, 1.x) are recorded either way.

#### WI-44 — Add the release workflow for the private feed

- **domain:** `ci`
- **depends-on:** WI-3, U-5 (consumer credential for the trial restore)
- **satisfies:** AC-8 (mechanism), AC-58, FC-1 (trial push and restore)
- **change:**
  - **D-2 decided (GP-10):** GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json`.
  - `.github/workflows/release.yml` triggers on a pushed `v*.*.*` tag. It sets up .NET 10, then restores, builds, tests and packs.
  - It runs `dotnet nuget push "**/*.nupkg" --source https://nuget.pkg.github.com/IronSloth/index.json --api-key ${{ secrets.GITHUB_TOKEN }}`. Only `.nupkg` files are pushed; GitHub Packages does not take `.snupkg` (inferred).
  - Job permissions are `contents: read` and `packages: write`, and nothing else. No other workflow gains a write scope. The packages link to the repository through `RepositoryUrl` (3.3.5), which is what lets the workflow token publish.
  - **Trial run (FC-1), done early in Phase 1, not at release.** A `workflow_dispatch` input publishes a throwaway package, ID `ValhallaTech.FeedTrial`, version `0.0.1`. A separate ID keeps the two real package IDs free of versions that AC-105 would require the CHANGELOG to record. A scratch `net10.0` console then restores it with Fred's `read:packages` credential (U-5). Afterwards the trial package is deleted from the feed.
- **done when:** The trial push succeeds and the trial restore exits 0. The trial package is deleted. AC-58 still holds.

#### WI-45 — Dry-run the assetvault-io migration against the release candidate

- **domain:** `tests`
- **depends-on:** WI-41, WI-33
- **satisfies:** AC-131 (pre-flight; gates WI-46)
- **change:**
  - Pack the release candidate to a local folder feed.
  - Copy assetvault-io `main` @ `863c0cb0cde62f820b76b9c76e1b4cf245f7dc55` into the session scratchpad. **Never edit `C:\Repo\assetvault-io`.**
  - Apply only the guide's steps, then `dotnet build` and `dotnet test tests/AssetVault.BLL.Tests`.
- **done when:** Build exit code is 0, with 0 failed and 0 skipped tests, and the diff touches only guide-directed lines. Any failure goes back to WI-41 (guide) or to the owning library item.

#### WI-46 — Run the release gate and publish 1.0.0 (point of no return)

- **domain:** `ci`
- **depends-on:**
  - WI-10, WI-12, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-28, WI-31, WI-33, WI-35, WI-37, WI-39, WI-40, WI-41, WI-42, WI-43, WI-44, WI-45;
  - U-1, U-2, U-3, U-4, U-5.
- **satisfies:**
  - AC-1, AC-4, AC-5, AC-7, AC-8, AC-9, AC-10, AC-11, AC-12, AC-15, AC-16, AC-17, AC-18;
  - AC-20 (verifies U-2), AC-45, AC-46, AC-47, AC-48, AC-49, AC-57, AC-118, AC-119 (verified).
- **change:**
  - On the release commit, run every release-time check in §3.7's "release gate" rows. AC-49 runs on the release date, with any bump going through a normal PR first.
  - **Network.** The gate host needs HTTPS to `registry.npmjs.org` (AC-133, AC-137, AC-139, AC-140; AC-138 and AC-143 via the CI run), to the GitHub API and Advisory Database (AC-20, AC-50, AC-118, AC-119), and to the IronSloth feed (AC-8, AC-9). An unreachable endpoint is a failed check that names the URL, never a skip. The release halts until the check runs and passes.
  - Confirm the Fred-only items.
  - Tag `v1.0.0` so WI-44 publishes.
  - Then verify AC-8 (published timestamp ≤ `2026-11-10T23:59:59-05:00`) and AC-9 (a scratch `net10.0` console restored from the feed only).
- **done when:** Every listed AC passes. Any failure halts the release and is reported, not worked around.

#### WI-47 — Prove the assetvault-io migration from the published package

- **domain:** `tests`
- **depends-on:** WI-46
- **satisfies:** AC-131
- **change:** WI-45's procedure, with the package restored from the private feed at 1.0.0.
- **done when:** AC-131's assertion holds.

##### Phase 3 — 1.x, required before this spec is done

#### WI-48 — Rewrite the README and add a readme per package

- **domain:** `docs`
- **depends-on:** WI-33, WI-35
- **satisfies:** AC-100, AC-101, AC-103 (content), AC-122 (README half)
- **change:**
  - The README describes only existing features, has no Branca or PASETO, and has no upstream package, CI, issue or release links.
  - Installation names the feed `https://nuget.pkg.github.com/IronSloth/index.json` (D-2, GP-10) and the `read:packages` credential it needs, until the public nuget.org release (Q-2).
  - No readme mentions the sample's removed client library (AC-137 searches them).
  - The password section uses `AddExtendedPasswordRules`.
  - New `src/ValhallaTech.Security.IdentityModel/README.md` and `src/ValhallaTech.Security.IdentityModel.AspNetCore/README.md`, each naming its own package ID.
- **done when:** AC-100 and AC-101 searches return 0 matches. The AC-122 README search is clean.

#### WI-49 — Pack each package readme

- **domain:** `backend`
- **depends-on:** WI-48
- **satisfies:** AC-103
- **change:** `PackageReadmeFile=README.md` in each library project, packing its own readme.
- **done when:** AC-103 holds on both packs.

#### WI-50 — Compile every documented C# example in CI

- **domain:** `ci`
- **depends-on:** WI-48, WI-46
- **satisfies:** AC-102
- **change:** Add `build/Test-DocSnippets.ps1` (placement rules in 3.3.5) and a CI step on the ubuntu job.
- **done when:** AC-102 shows 0 compiler errors for every block.

#### WI-51 — Gate line coverage per library at 80%

- **domain:** `ci`
- **depends-on:** WI-2, WI-3 (may land during Phase 1)
- **satisfies:** AC-116, AC-117
- **change:**
  - The ubuntu test step collects Cobertura through `coverlet.collector`.
  - `build/Test-CoverageThreshold.ps1` reads the per-package `line-rate` for exactly the two library assemblies. It fails when either is below 80.0, printing the name and the percentage to one decimal.
  - A committed fixture, `build/fixtures/coverage-79.9.cobertura.xml`, is run in the same job and **must** make the gate fail.
  - This is a fixed floor, not a ratchet (C-17). It is a script because AC-117 feeds the gate a report, which coverlet's MSBuild threshold cannot accept.
- **done when:** AC-116 is at least 80% for each library on the CI build, and the AC-117 fixture run fails naming the library and `79.9`.

#### WI-52 — Fail the build on a breaking public-API change after 1.0.0

- **domain:** `ci`
- **depends-on:** WI-46
- **satisfies:** AC-115
- **change:**
  - In `src/Directory.Build.props`, set `EnablePackageValidation=true`, with the 1.0.0 packages committed under `eng/baseline/` and referenced through `PackageValidationBaselinePath` (inferred property name; verify first).
  - A committed baseline means no developer restore needs feed credentials. Alternative lost: `PackageValidationBaselineVersion`, which needs every restore to authenticate to the private feed.
- **done when:** AC-115's probe, which removes `Base62.Encode(byte[])`, fails pack and names the member.

#### Unmapped work items

_Work with no tag in the allowed set. Never invent a tag — list it here with a
recommended owner so the orchestrator can route by judgment._

| ID | Title | Satisfies | Recommended owner | Why unmapped |
| --- | --- | --- | --- | --- |
| U-1 | Supply the ValhallaTech logo as a PNG of at least 128×128 at `assets/valhallatech-logo.png` | AC-12 (FC-6) | Fred | A brand asset only Fred holds; no agent can produce it. |
| U-2 | Enable GitHub private vulnerability reporting on `IronSloth/IdentityModel-Supa` | AC-20 (C-21, FC-2) | Fred | A repository setting only Fred can change (C-21). |
| U-3 | Enable Dependabot alerts (`vulnerability-alerts`) on the canonical repository | AC-119 | Fred | A repository security setting, the same class as C-21. |
| U-4 | Install the Renovate GitHub App on `IronSloth/IdentityModel-Supa` | AC-118 | Fred | An account-level app installation; no repository file can do it. |
| U-5 | Issue the GitHub Packages credentials for the IronSloth feed (D-2 decided, GP-10): a `read:packages` token for consumers and for the WI-44 trial restore and AC-9. Confirm the workflow's `packages: write` publish is allowed for the IronSloth account. | AC-8, AC-9 (FC-1); unblocks WI-44's trial restore | Fred | Credentials belong to Fred's account. FC-1 says they are Fred's to issue. |
| U-6 | Audit test-first ordering from the commit history | AC-121, AC-132 | Agent Evaluator | It is an evaluation of the history the work items produce, not an artifact any domain builds (R-50 names the evaluator). |

### 3.5 Sequencing

_Ordered phases with the work items in each and what gates the next phase._

**Phase 0 — Foundation (serial, one lane):** WI-1 → WI-2 → WI-4 → WI-5 + WI-6 (one PR) → WI-7 → WI-8 → WI-9.
- These items run serially on purpose. The rename, the reformat and the annotation pass each touch every file, so running any of them beside other work guarantees merge conflicts (RK-9).
- In parallel: WI-3 (after WI-2), WI-10 (any time), and the Fred-only items U-1 to U-5, which should start at Gate 2 approval.
- **Gate 0:**
  - CI green on all three OSes on .NET 10;
  - all 250 tests pass under the new names;
  - 0 warnings with analyzers, nullable and docs enforced;
  - CSharpier clean.

**Phase 1 — Fan-out (lanes in parallel, each starting at Gate 0):**

| Lane | Order inside the lane | Notes |
| --- | --- | --- |
| A — EdDSA | WI-11 → WI-12; WI-16 → WI-17; WI-13 → WI-14 → WI-15 (after WI-12 and WI-17); WI-18 → WI-19 (after WI-17) | Tests items WI-11, 13/14, 16 and 18 can all be written at once. The implementations serialize on shared files: WI-17 before WI-19 (`EdDsa.cs`); WI-12 and WI-17 before WI-15. |
| B — Encoders | WI-20 → WI-21; WI-22 → WI-23; WI-24 → WI-25 | Three independent chains. |
| C — AspNetCore | WI-26 → WI-27 → WI-28; WI-29 → WI-30 → WI-31 (after WI-28) | WI-28 and WI-31 serialize on `ExtendedPasswordValidator.cs`. |
| D — Sweep | WI-32 at lane start → WI-33 last | WI-33 waits for every library implementation item. |
| E — Sample | WI-34 → WI-36; WI-34 → WI-35 (after WI-31); WI-36 → WI-37 | WI-37 (Bootstrap 5 CSS and JS through LibMan, 1.0.0 scope) does not wait for lane C. |
| F — Measurement | WI-38 | The harness is built early so WI-43 only runs it. |
| G — Automation | WI-44 (start of Phase 1), WI-39 (after WI-37), WI-51 | D-2 is decided (GP-10), so WI-44 starts at Gate 0. Its FC-1 trial push and restore must finish early in Phase 1, well before 2026-11-10; it waits only for U-5's read credential. |

- **Gate 1:**
  - every chain's tests green, and the full suite green;
  - 0 warnings;
  - a Security Reviewer pass over the four security controls: WI-12 (S1), WI-15 (S2), WI-17 (S3, S5, S11) and WI-21 (S4). Findings go back to the item's owner, and the reviewer re-checks.

  This review is an orchestration gate, not a work item.

**Phase 2 — Release:**
- WI-40, WI-41, WI-42 and WI-43 run in parallel.
- Then WI-45, the release-candidate dry run.
- Then **WI-46, the point of no return.** Publishing 1.0.0 is irreversible in practice, because the version is immutable once consumers restore it. WI-46 is gated by:
  - every 1.0.0 AC passing on the release commit;
  - WI-45 passing;
  - U-1 to U-5 complete;
  - Fred pushing the tag.
- Then WI-47.

**Phase 3 — 1.x, after release and before this spec is done:** WI-48 → WI-49 and WI-50; WI-52. WI-51 may land any time after WI-3.

**Critical path.** WI-1 → WI-2 → WI-4 → WI-5/6 → WI-7 → WI-8 → WI-9 → WI-16 → WI-17 → WI-19 → WI-33 → WI-41 → WI-45 → WI-46.
- Lane A is the longest because `EdDsa.cs` hosts three serialized implementations (C5, C6 and C4's dependency on C5).
- Lane C (WI-27 → WI-28 → WI-31) is the nearest rival, also feeding WI-33.
- Off the code path, **U-5 (feed credentials) gates WI-44's trial restore, and U-1 to U-4 gate WI-46.** If any Fred-only item lands late, it becomes the critical path. That is why they start at Gate 2, not in Phase 2.

**Parallel capacity.** After Gate 0, up to seven lanes run at once:
- tests engineers on WI-11, 14, 16, 18, 20, 22, 24, 27, 30, 32, 34 and 36;
- coding experts following each red tests item;
- the Frontend Expert on WI-37;
- CI on WI-39, 44 and 51;
- docs on WI-10.

The deadline is fixed (C-20). If Gate 1 is at risk, the PRD has nothing left to move out (§2.2.1), so Fred decides which of C-3, C-5 and C-20 gives. No item is silently dropped.

### 3.6 Risks & mitigations

| ID | Risk | Likelihood | Impact | Mitigation | Affects |
| --- | --- | --- | --- | --- | --- |
| RK-1 | The 1.0.0 scope (40 requirements, about 47 items before Phase 3) does not fit before 2026-11-10. | high | high | Serial Phase 0 kept to mechanical work; seven parallel lanes; 1.x items in Phase 3. If Gate 1 slips, Fred chooses which constraint gives (§2.2.1 feasibility). No agent drops scope. | all |
| RK-2 | Microsoft.IdentityModel 7 → 8 or BouncyCastle 2.4 → 2.7 changes behavior or APIs that the libraries or tests rely on (FC-3). | med | high | Done first, alone, in WI-1 and WI-2, with all 250 tests run on 8.x before any behavior change. A failure is a finding for WI-1's owner, not a reason to pin old versions (G-9 has no exception rule; A-14). | WI-1, WI-2 |
| RK-3 | The AwesomeAssertions swap is not mechanical (FC-4): 19 files, 157 `.Should()` calls, and 6→7 renames such as `BeEquivalentTo` strictness on byte arrays. | med | med | An isolated item (WI-2), driven by compiler errors. Rule: no test is weakened, and a construct with no equivalent is rewritten with xUnit `Assert`, stated in the PR. The test-name set must not change. | WI-2 |
| RK-4 | A per-key `CryptoProviderFactory` is expensive (it allocates caches) when keys are built per tenant or per request (C1), or the inference that new keys share `CryptoProviderFactory.Default` is wrong. | med | med | `CacheSignatureProviders = false` and a shared, stateless provider instance. AC-21 proves the inference in WI-11, and AC-25 builds 1,000 keys concurrently. If allocation is material, WI-12's owner reads the pinned Microsoft.IdentityModel source and reports before choosing anything else, because the shared-factory alternative weakens G-4. | WI-11, WI-12 |
| RK-5 | Consumers that validate EdDSA through JWKS `JsonWebKey`s, or that set `TokenValidationParameters.CryptoProviderFactory`, find EdDSA unsupported once no global registration exists. | med | med | The provider accepts converted `JsonWebKey`s (AC-29). The migration guide documents conversion and the override rule (3.3.3, WI-41). assetvault-io builds `EdDsaSecurityKey`s directly, so it is unaffected. | WI-12, WI-41 |
| RK-6 | The shipped Bootstrap bundle stops matching the npm tarball, so it drops out of set V and its 19 jQuery-detection mentions fail AC-137/AC-138. Causes: a bundling or minifying step, a jsDelivr byte difference (inferred not to happen), or a Renovate bump restored without a test run. Separately, the WI-36 tests depend on network access to `registry.npmjs.org`. | med | med | No transform touches `wwwroot/lib/` (WI-37). AC-143's hash test runs in CI on every PR, including Renovate's, keyed on `libman.json`, so drift fails the PR that caused it. Network-dependent tests are marked `Category=Network` and are never skipped; an outage fails loudly and is re-run, not muted. | WI-36, WI-37, WI-39 |
| RK-7 | Fred-only actions (U-1 to U-5) arrive late: the logo, private reporting, Dependabot alerts, the Renovate app, and the feed credentials needed for the FC-1 trial. | high | high | All are listed with the item each one gates, and they start in parallel with Phase 0. The FC-1 feed trial (WI-44) is scheduled early in Phase 1 so a feed problem surfaces while there is time. WI-46 halts on any missing one; AC-20 and AC-12 are never faked. | WI-42, WI-44, WI-46 |
| RK-8 | Raising minimums to latest stable makes consumers with lower direct references fail restore (NU1605). assetvault-io pins BouncyCastle 2.6.2. | high | med | The guide makes the bump an explicit step. WI-45 runs the real assetvault-io migration against the release candidate **before** publish, so a library-side surprise is found while it is still fixable. | WI-41, WI-45 |
| RK-9 | Repo-wide mechanical changes (rename, reformat, annotations) conflict with parallel lanes. | high (if parallel) | med | All three run in serial Phase 0 before fan-out. Lanes are split by file; the implementations that share `EdDsa.cs` or `ExtendedPasswordValidator.cs` are serialized by `depends-on`. | WI-5, WI-7, WI-9 |
| RK-10 | Squash merges or rebased history destroy the commit ordering AC-121 and AC-132 audit. | med | high | No squash merges for this spec (§3.4 conventions). Test commits carry `test(WI-n)` and the AC IDs, so U-6 can find them. | all tests items, U-6 |
| RK-11 | Measurement noise, or the measuring machine being unavailable or changing between baseline and after. | med | med | Baseline and after are measured in one BenchmarkDotNet process, so the same session, machine and runtime are guaranteed (WI-38). WI-43 runs on Fred's workstation. AC-70 is calibrated by `B`. A regression or overrun is reported, never retuned by the agent that wrote the code. | WI-38, WI-43 |
| RK-12 | A chosen package is deprecated or fails on `net10.0` before release: `xunit` 2.x, `Microsoft.Web.LibraryManager.Build`, or the DryIoc 6.2.0 adapter (AC-46, AC-49). | low | med | Each item's done-when runs `--deprecated` and a `net10.0` build. Fallbacks: xunit.v3; a committed CSS plus an npm `package.json` that Renovate reads; reporting DryIoc to Fred, with no preview substituted. | WI-2, WI-35, WI-37 |
| RK-13 | A new stable release of a dependency lands between implementation and release and fails AC-49 on the day, or a new SimpleBase release changes Base62 behavior or speed. | med | med | Renovate is active before release (WI-39). WI-46 re-runs AC-49 on the release date, and bumps go through a normal PR and CI. SimpleBase is unpinned (GP-10). A bump must pass the C7 suite (AC-33 to AC-36, AC-124 to AC-126) in CI. A bump after WI-43 re-runs the P-a and P-b benchmarks (AC-70, AC-127) before WI-46; a budget overrun is reported to Fred. | WI-39, WI-43, WI-46 |
| RK-16 | **Resolved at Gate 2, round 2.** AC-137 as first written could not pass: its search also hit the test source and `.git/`. §2 now closes the exclusion list with `.git/` and the one named test file, and checks that file separately. | — | — | No action left. WI-36 places the assertions at the excluded path, and that file meets AC-137's separate check. | WI-36 |
| RK-14 | **Operations: monitoring.** A defect in 1.0.0 surfaces only in consumers; the library emits no telemetry, by design (NG-6). | med | med | Post-release signals and where they are watched: CI on every `master` push (all three OSes, coverage, CodeQL); Dependabot alerts and the Renovate dashboard (U-3, WI-39; M-2, M-4, M-10); private vulnerability reports (U-2; M-9); the M-1 adoption search. No new signal needs building. | WI-3, WI-39, WI-51 |
| RK-15 | **Operations: rollback.** A defect found after WI-46 publishes 1.0.0. | low | high | Before WI-46, rollback is `git revert`. After WI-46, rollback is **forward-only**: the 1.0.0 version is immutable once restored. Consumers pin back to upstream 4.1.0, which stays on nuget.org, and a 1.0.1 ships through the same workflow. The point of no return is WI-46, gated as in §3.5. | WI-46 |

**Operations:** monitoring: RK-14; backup: not applicable, since no step destroys or rewrites data (no database, cache or persisted state, §3.2; the repository rename is a `git mv` recorded in history); rollback: RK-15.

### 3.7 Test strategy

_Every AC maps to an executable check, or is explicitly recorded as manual._

**Test seams for the test-first chains.** The stubs step creates or names these surfaces, and acceptance tests target nothing else.

| Chain | Tests item → implementation item | Test seam (exact surface) |
| --- | --- | --- |
| C1 | WI-4 → WI-5 | `Assembly.GetExportedTypes()`, `Assembly.GetName()` and `Assembly.GetCustomAttributesData()` on `typeof(Base62).Assembly` and `typeof(NewPasswordTagHelper).Assembly`; public `const` fields of every exported type |
| C2 | WI-8 → WI-9 | `new NullabilityInfoContext().Create(ParameterInfo / PropertyInfo / FieldInfo)` over public members of the two assemblies |
| C3 | WI-11 → WI-12 | `new EdDsaSecurityKey(EdDsa)`; `EdDsaSecurityKey.CryptoProviderFactory` (`.CreateForSigning(SecurityKey, string)`, `.CreateForVerifying(SecurityKey, string)`, `.ReleaseSignatureProvider(SignatureProvider)`, `.CustomCryptoProvider.Create(string, params object[])`); `JsonWebTokenHandler.CreateToken(SecurityTokenDescriptor)` and `.ValidateTokenAsync(string, TokenValidationParameters)`; static `CryptoProviderFactory.Default.CustomCryptoProvider` |
| C4 | WI-13 (stubs) → WI-14 → WI-15 | `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKey(EdDsaSecurityKey)`, `.ConvertFromEdDsaSecurityKeyIncludingPrivateKey(EdDsaSecurityKey)`, `.TryConvertToEdDsaSecurityKey(JsonWebKey, out EdDsaSecurityKey)`; AC-29 through `JsonWebTokenHandler.ValidateTokenAsync` with `IssuerSigningKey = jwk` |
| C5 | WI-16 → WI-17 | `EdDsaParameters.Validate()`; `EdDsa.Create(EdDsaParameters)`, `EdDsa.Create(string)`, `EdDsa.Sign(byte[])`, `EdDsa.Verify(byte[], byte[])`, `EdDsa.Dispose()`, `EdDsa.KeySize`, `EdDsa.LegalKeySizes`, `EdDsa.ImportFromPem(ReadOnlySpan<char>)`, both `EdDsa.ImportFromEncryptedPem` overloads; `EdDsaSecurityKey.KeySize`; for AC-30 and AC-31, reflection over instance fields reachable from the `EdDsa` |
| C6 | WI-18 → WI-19 | `EdDsa.Verify(byte[], int, int, byte[], int, int)`; `SignatureProvider.Sign(ReadOnlySpan<byte>, Span<byte>, out int)` on the provider from `key.CryptoProviderFactory.CreateForSigning(key, "EdDSA")` |
| C7 | WI-20 → WI-21 | `Base62.Encode(byte[])`, `Base62.Decode(string)` |
| C8 | WI-22 → WI-23 | `Base16.Encode(byte[])`, `Base16.Decode(string)` |
| C9 | WI-24 → WI-25 | `StreamExtensions.TryRead(this Stream, int, out byte[])` |
| C10 | WI-32 → WI-33 | `ConstructorInfo.Invoke` and `MethodInfo.Invoke` over every public, non-`Try` ctor and method of both assemblies; `new NewPasswordTagHelper(IHtmlGenerator, IOptions<IdentityOptions>)` |
| C11 | WI-26 (stubs) → WI-27 → WI-28 | `ExtendedPasswordValidator<TUser>.ValidateAsync(UserManager<TUser>, TUser, string)`; `.HasConsecutiveCharacters(string, int)`; `ExtendedIdentityErrorDescriber.PasswordTooLong(int)` and `.TooManyConsecutiveCharacters(int)` registered through `IdentityBuilder.AddErrorDescriber<TDescriber>()` |
| C12 | WI-29 (stubs) → WI-30 → WI-31 | `IdentityBuilder.AddExtendedPasswordRules(Action<ExtendedPasswordOptions>)`; `IHost.StartAsync(CancellationToken)`; `UserManager<TUser>.PasswordValidators`; `NewPasswordTagHelper.Process(TagHelperContext, TagHelperOutput)`; `DryIocServiceProviderFactory` for the DryIoc test |
| C13a | WI-34 → WI-35 | `WebApplicationFactory<Program>`: GET `/`, GET `/Home/PasswordRules`; `IOptionsMonitor<JwtBearerOptions>.Get("eddsa")`; `IdentityModelEventSource.ShowPII`; the AC-54 `IStartupFilter` probe |
| C13b | WI-36 → WI-37 | The HTML of GET `/` and GET `/Home/PasswordRules` from `WebApplicationFactory<Program>`, and every `<script src>` and `<link href>` asset fetched through the same `HttpClient`. Set V comes from the sample's `libman.json` and the matching npm tarball at `https://registry.npmjs.org/<name>/-/<name>-<version>.tgz` (AC-138, AC-143). |

No AC observable below the browser lacks a chain, and every chain has a named seam. ACs without a chain are `build`, `lint`, out-of-process `integration` (CI runs, feed, GitHub API, measurement, scratch consumers), or Fred-only.

**AC → check.** Levels follow the PM's `Executable as` throughout; none is changed. Key to "Where it runs":
- **CI** means `dotnet test` on the three-OS matrix.
- **CI-u** means the ubuntu CI job only.
- **Gate** means WI-46's release-gate run on the release commit.
- **Bench** means WI-43 on Fred's workstation.

| AC | Check | Level | Where it runs |
| --- | --- | --- | --- |
| AC-1, AC-10, AC-11, AC-16, AC-17 | Unzip each `.nupkg`; read `lib/` folders and the nuspec; byte-compare the packed `LICENSE` | build | CI-u (pack step), Gate |
| AC-2 | `dotnet msbuild -getProperty:TargetFramework -getProperty:TargetFrameworks` on the test and sample projects | build | CI-u, Gate |
| AC-3, AC-4, AC-57 | `dotnet build -c Release --no-incremental` (0 warnings), then `dotnet test` with a TRX logger: failed and skipped both 0, runtime 10.0.x | build | CI, Gate |
| AC-5 | Diff test FQN sets (`555d31c` vs release, under the 3.3.1 mapping) against the §3.4 removed-test table | lint | Gate |
| AC-6, AC-58 | `yq` over `.github/workflows/*.yml` | lint | Gate (and on review of WI-3 and WI-44) |
| AC-7 | `gh run view <id> --json jobs`, plus the test-step logs | integration | Gate (CI run of the release commit) |
| AC-8, AC-9 | Feed metadata query; a scratch `net10.0` console restoring only from the feed | integration | Gate, after publish |
| AC-12 | PNG signature, IHDR size, SHA-256 against the supplied and the `555d31c` icon | build | Gate |
| AC-13, AC-14, AC-95, AC-96 | C1 reflection tests | unit | CI |
| AC-15, AC-18 | `rg -i -uu "scottbrady"`, and the notice-string search per 3.3.5 | lint | Gate |
| AC-19 | `rg` on `SECURITY.md` | lint | Gate |
| AC-20 | `gh api /repos/IronSloth/IdentityModel-Supa/private-vulnerability-reporting` | integration | Gate (after U-2) |
| AC-21, AC-23, AC-25, AC-32, AC-81 | C3 tests in the `GlobalCryptoState` collection | unit | CI |
| AC-22, AC-24 | C3 in-process JWT create and validate | integration | CI |
| AC-26 to AC-28, AC-82, AC-93, AC-94 | C4 tests | unit | CI |
| AC-29 | C4 JWT validated with a public-only JWK | integration | CI |
| AC-30, AC-31, AC-42 to AC-44, AC-73, AC-87, AC-91, AC-92, AC-128, AC-129, AC-135 | C5 tests | unit | CI |
| AC-88 | C5 JWT validated with a private-only key | integration | CI |
| AC-33 to AC-36, AC-79, AC-124 to AC-126 | C7 tests | unit | CI |
| AC-37 to AC-41 | C8 tests | unit | CI |
| AC-45, AC-46 | `dotnet list package --vulnerable` and `--deprecated --include-transitive`; NuGetAudit already fails restore on High or Critical | build | CI (restore), Gate |
| AC-47 | `rg -uu "SecurityCodeScan" -g "*.csproj" -g "*.props" -g "*.targets"` | lint | Gate |
| AC-48 | Read each direct package's nuspec `<license type="expression">` against the OSI list | lint | Gate |
| AC-49 | `dotnet list package --outdated` (direct, stable) for every project, on the release date | lint | Gate |
| AC-50, AC-51, AC-133, AC-139, AC-140 | Needs `registry.npmjs.org` (AC-133, AC-139, AC-140) and the GitHub Advisory Database (AC-50). Without network these fail loudly, naming the URL; they are never skipped. Steps: `libman.json` library and version (the sample's only client-library inventory) → `gh api /advisories?affects=bootstrap@<v>`, Bootstrap's support status, npm `dist-tags.latest`, npm `dependencies`/`peerDependencies` | lint | Gate |
| AC-52 to AC-54, AC-112 | C13a in-process host tests | integration | CI |
| AC-55, AC-99 | Probe copy (CA5351 line; undocumented public method) built `-c Release` | build | Gate (and once in WI-9) |
| AC-56 | `dotnet msbuild -getProperty:NoWarn;RunAnalyzers;EnableNETAnalyzers;AnalysisMode;AnalysisLevel`, plus `rg` over `.editorconfig`/`.globalconfig` | lint | Gate |
| AC-59, AC-123 | Parse `perf/results/1.0.0/report.csv` and `report-ed448.csv` | lint | Gate |
| AC-60 to AC-70, AC-127 | Compare rows under the measurement rules; AC-69 `base16-scaling.csv`; AC-70 `base62-budget.csv` | integration | Bench |
| AC-71, AC-72, AC-83, AC-86 | C9 tests | unit | CI |
| AC-74 to AC-77, AC-142 | C12 tests (DI container, generic host, tag helper) | integration | CI |
| AC-78, AC-80 | C10 sweep | unit | CI |
| AC-84, AC-85, AC-89, AC-90 | C6 tests | unit | CI |
| AC-97 | C2 test | unit | CI |
| AC-98 | Match public members to the built XML doc file | build | Gate |
| AC-100, AC-101 | `rg -i` per the AC on the README and package readmes | lint | Phase 3 review of WI-48 |
| AC-102 | `build/Test-DocSnippets.ps1` | build | CI-u (from WI-50) |
| AC-103 | Nuspec `<readme>` and the packed file contains its package ID | build | CI-u (from WI-49) |
| AC-104, AC-105 | 3.3.5 phrase list over the `1.0.0` section; headings against tags and feed versions | lint | Gate |
| AC-106 to AC-110 | C11 tests | unit | CI |
| AC-111 | C11 DI-resolved validator with a registered describer | integration | CI |
| AC-113, AC-122 | `rg` on the sample (and, for AC-122, the README) | lint | Gate (sample); WI-48 review (README) |
| AC-114, AC-138, AC-141, AC-143 | C13b in-process host HTML and asset tests; SHA-256 against the npm tarball for set V and the Bootstrap bundle | integration | CI (needs network to `registry.npmjs.org`) |
| AC-115 | Probe copy with `Base62.Encode(byte[])` removed, packed against the `eng/baseline/` 1.0.0 | build | WI-52, then CI-u |
| AC-116, AC-117 | Cobertura gate per library; the 79.9% fixture must fail | build | CI-u (from WI-51) |
| AC-118, AC-119 | `gh pr list` / `gh issue list --author renovate[bot]`; `gh api /repos/IronSloth/IdentityModel-Supa/vulnerability-alerts` returns 204 | integration | Gate |
| AC-120 | `dotnet csharpier check .` on the release commit and on a probe | lint | CI-u, Gate |
| AC-121, AC-132 | History audit: check out each introducing commit and each implementing commit, and run the named tests | integration | U-6 (Agent Evaluator) |
| AC-130 | 3.3.5 API and topic phrase match over the guide | lint | Gate |
| AC-131 | Scratch assetvault-io at `863c0cb`: build plus `tests/AssetVault.BLL.Tests` | integration | WI-45 (release candidate), WI-47 (published) |
| AC-134 | Renovate dependency dashboard set against the AC-49, AC-133 and `uses:` inventory | lint | Gate |
| AC-136 | Read the `Base62.Encode`/`Decode` `<member>` entries in the built XML doc file | build | Gate |
| AC-137 | The AC's four checks with its closed exclusion list (`bin/`, `obj/`, `.git/`, `specs/`, `CHANGELOG.md`, and `test/ValhallaTech.Security.IdentityModel.Samples.Tests/ForbiddenClientLibraryTests.cs`), plus the separate check on that file. Check 2 subtracts files whose SHA-256 matches a file in the npm tarball of a `libman.json` library (set V) at the declared version. Run after a build, so the restored `wwwroot/lib/` files are present. | lint | Gate (needs `registry.npmjs.org`) |

### 3.8 Domain tag summary

_The dispatch seam. One row per tag actually used._

| Tag | Work items | Suggested specialist |
| --- | --- | --- |
| `backend` (19) | WI-1, WI-5, WI-9, WI-12, WI-13, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-26, WI-28, WI-29, WI-31, WI-33, WI-35, WI-42, WI-49 | C# Coding Expert |
| `tests` (20) | WI-2, WI-4, WI-6, WI-8, WI-11, WI-14, WI-16, WI-18, WI-20, WI-22, WI-24, WI-27, WI-30, WI-32, WI-34, WI-36, WI-38, WI-43, WI-45, WI-47 | C# Unit Test Engineer. WI-34 and WI-36 are in-process host tests, not Playwright. WI-38 and WI-43 are the measurement harness and run. |
| `ci` (8) | WI-3, WI-7, WI-39, WI-44, WI-46, WI-50, WI-51, WI-52 | CI/CD Pipeline Engineer (WI-3, WI-39, WI-44, WI-46, WI-52); PowerShell CI Engineer (WI-50, WI-51 scripts); WI-7 either |
| `docs` (4) | WI-10, WI-40, WI-41, WI-48 | Documentation Agent |
| `frontend` (1) | WI-37 | Frontend Expert |

Unused tags: `database`, `redis`, `security`, `openfga` and `docker`. These libraries have no such artifacts (NG-6). The security controls (S1 to S5, S11) are library code, so they are tagged `backend` by artifact. The Security Reviewer reviews them at Gate 1 as a gate, not as an owner. Total: 52 tagged items and 6 unmapped (U-1 to U-6).

#### D-item resolution ledger

§1.8 holds 11 D-items: all 11 are resolved and none is raised.

| D-item | Outcome | Where |
| --- | --- | --- |
| D-1 Assertion library | **Decided by Fred (2026-10-25): AwesomeAssertions 9.6.0.** | §3.4 WI-2 |
| D-2 Private NuGet feed | **Decided by Fred at Gate 2 (GP-10): GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json`.** | §3.1, WI-44, WI-41, WI-48, U-5 |
| D-3 Base62 leading zeros | **Fixed by Fred (GP-2): SimpleBase `Base62.Default` with the cap.** Since GP-10 its version follows latest stable, with no pin. | §3.3.2 (`Base62` rows), WI-1, WI-21 |
| D-4 Private-export opt-in shape | **Decided:** a separate method, `ConvertFromEdDsaSecurityKeyIncludingPrivateKey`. | §3.3.2, WI-13 |
| D-5 64/114-byte private keys | **Decided:** rejected at validation. | §3.3.2 (`Validate` row), WI-17 |
| D-6 Measurement tool | **Decided:** BenchmarkDotNet, with a source-level `555d31c` baseline in the same process. | §3.4 WI-38 |
| D-7 Update and format tools | **Decided:** Renovate (WI-39) for updates; CSharpier (WI-7) for formatting; GitHub Dependabot alerts for advisories (U-3). | §3.4 WI-7, WI-39 |
| D-8 Sample DI and UI | **UI fixed by Fred (GP-8, GP-10):** Bootstrap 5 latest stable, CSS and JS bundle shipped unmodified, no jQuery. Delivered through LibMan, with a Renovate regex manager over `libman.json`. **DryIoc decided by Fred (2026-10-25, "Yes, sample + one test"):** the sample runs on `DryIoc.Microsoft.DependencyInjection` 6.2.0 with `DryIoc.dll` 5.4.3, plus one DryIoc test of the single step. | §3.4 WI-35, WI-30, WI-37 |
| D-9 Solution format and build layout | **Decided:** `.slnx`, `global.json`, a root plus a `src/` `Directory.Build.props`, central package management, and renamed directories. | §3.3.4, WI-1, WI-5 |
| D-10 CI matrix and CodeQL | **Decided:** keep both. | §3.4 WI-3 |
| D-11 Message localization mechanism | **Decided:** an `ExtendedIdentityErrorDescriber` subclass through `AddErrorDescriber<T>()`. | §3.3.2, WI-26 |

#### Raised for Gate 2

_§1.8 D-items the Architect could not decide; each is a question for Fred at Gate 2._

None. D-2, the only item raised at Gate 2, was decided by Fred (GP-10) and is recorded in the ledger above.
