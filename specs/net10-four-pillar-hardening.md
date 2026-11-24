# Spec: .NET 11 Four-Pillar Hardening

| Field | Value |
| --- | --- |
| Slug | `net10-four-pillar-hardening` |
| Created | 2026-10-25 |
| Stage | `approved` |
| Track | `full` |
| Brief | ✅ complete |
| PRD | ✅ complete (amended 2026-11-23 for AM-1, AM-2, Q-30, AM-3 and AM-4) |
| Technical Plan | ✅ complete (amended 2026-11-23 for AM-1, AM-2 and Q-30, then for AM-3 and AM-4; approved at Gate 2 on 2026-11-23) |

**Test-first:** acceptance

_`none` (default) builds tests after code, as always. `acceptance` (pilot): each AC observable below the browser is covered by a §3.4 chain of stubs → failing acceptance tests → implementation, one chain per test seam. Set by the Analyst from /spec Stage 0; read by everyone else._

> One feature, one file. Three sections, three owners. Section owners never edit
> another owner's section — contradictions are reported upward, not silently patched.

---

## 0. Kernel

_A summary written by the Architect. The body (§1 to §3) wins on any conflict._

- **Purpose:** ValhallaTech ships the fork as its own 1.0.0 packages for .NET 11, the org standard, with every behavior-changing security, performance and UX fix in one major version, released when its gates pass. (G-1, G-2, C-2, C-5, C-20, C-25, AM-2, AM-3)
- **Capabilities:** Every project targets `net11.0` only, all code is C# 15, and both packages publish to the IronSloth GitHub Packages feed. Keys never write process-wide crypto state, JWK export is public-only by default, and validation rejects unusable or mismatched keys. Base16 and Base62 are strict and lossless, and Base62 is capped. Argument errors, key sizes and try-patterns follow .NET conventions. Password rules turn on in one step, with replaceable messages. The sample runs with Bootstrap 5 and no jQuery. The CHANGELOG's breaking-change list is the only migration aid. (R-1, R-3, R-9 to R-15, R-25, R-28 to R-36, R-42, R-44, R-56, AD-14, AD-15, AD-43, AD-44, AD-51)
- **Constraints:** Test-first at acceptance and unit level; Apache-2.0 attribution kept; every direct dependency at its single latest stable version; no jQuery; the Base62 cap is fixed; process stays proportionate; no repository outside this one shapes the work. (C-13, C-16, C-17, C-23, C-24, C-25, C-27, R-17, NG-12, AD-45)
- **Non-goals:** nuget.org publication, .NET 6, 8, 9 or 10, new algorithms, staying mergeable with upstream, Branca or PASETO, ValhallaTech's application stack, and any migration guide or proof for upstream consumers. (NG-1 to NG-10, NG-12, CUT-1 to CUT-4)
- **Success signal:** 1.0.0 is on the feed only after every release gate passed on the release commit: three workflow runs, plus three commit statuses for the checks run outside a workflow. `1.x`-only checks run and are recorded but never hold it back. Each of those gates is a test-first acceptance test. A `net11.0` consumer restores the `net11.0` build and validates an EdDSA JWT, and no measured path regresses on .NET 11. The spec completes at 1.1.0, where every check blocks. (AC-4, AC-8, AC-9, AC-68, AC-121, AC-144, AC-145, AD-55, AD-60, AD-64, AD-65)

---

## 1. Brief

<!-- OWNER: Analyst. No other agent may edit this section. -->

**Status:** ✅ complete

### 1.1 Problem statement

_What is wrong / missing today, in the user's terms. No solution language._

This repository is ValhallaTech's fork of `scottbrady91/IdentityModel`, taken at upstream's last release (4.1.0, 2024-11-20) and now at `master` `555d31c`. It ships two independent libraries: `src/ScottBrady.IdentityModel` (EdDSA-signed JWT support for Microsoft.IdentityModel, plus Base16 and Base62 encoders) and `src/ScottBrady.IdentityModel.AspNetCore` (a password-rules tag helper and an ASP.NET Identity password validator). The source of the defects below is the improvement review of `555d31c` (`C:\Users\Fred\AppData\Local\Temp\claude\C--Repo-IdentityModel-Supa\ebc03158-8223-460c-98a8-90333d58c9d6\scratchpad\IdentityModel-Supa-review.md`); review IDs (S1, P1, C1, U1, …) are kept for traceability. Items the reviewer reproduced with a probe program are marked **[confirmed]**; the rest come from reading the code.

ValhallaTech wants to take this fork over as its own package, used first by its own projects and later by the public. Today it cannot, for five groups of reasons.

**1. The libraries are not built for any .NET that still gets security patches, nor for ValhallaTech's standard.** _(Amended 2026-11-23 per Fred's decisions AM-1 and AM-2, and again per AM-3 and AM-4, §1.8.)_ .NET 8 and .NET 9 both reached end of support on 2026-11-10. The spec's original hard date was that same day, and it passed with no implementation started (AM-2), so since then the libraries have targeted only runtimes that get no security patches. .NET 11 has shipped (runtime 11.0.0, below). It is an STS release, supported until 2028-11-14 (Q-27; amended 2026-11-23 (Gate 2)). As of 2026-11-23, ValhallaTech's org standard is .NET 11 with C# 15 (AM-1, C-25), and C# 15 is supported only on .NET 11 and newer (C-27). Fred has ruled that .NET 11 is the only .NET this work serves, and that repositories outside this one do not shape it (AM-3, C-2, NG-12). No project is known to consume the fork; assetvault-io, once listed here as its first consumer, uses upstream `ScottBrady.IdentityModel` 4.1.0, not the fork (AM-3). The .NET 11.0.100 SDK and 11.0.0 runtime are installed on Fred's machine (ODIN01) alongside .NET 10 *(per the /spec coordinator, 2026-11-23)*. The target frameworks and CI SDK are as they were when the brief was approved *(re-read by the Analyst, 2026-11-23, in each `.csproj` and both workflows)*:
- The core library targets `net8.0;net9.0`.
- The ASP.NET Core library targets `net6.0;net8.0;net9.0`, and `net6.0` has been out of support since November 2024.
- The tests target `net8.0;net9.0` and the sample targets `net8.0`.
- Both CI workflows install the 9.0 SDK.

Nothing in the repo builds or tests against .NET 11, so nothing shows that the libraries work there.

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

- **Developers of ValhallaTech's own projects.** They are the first consumers (R1-Q4). They need packages that work on .NET 11, the org standard (C-2, C-25). They are exposed to S1/S2 and the U1–U9 friction. No ValhallaTech project consumes the fork today. _(Amended 2026-11-23: AM-1 had added .NET 10 for assetvault-io and AM-2 dropped the 2026-11-10 date; AM-3 then dropped .NET 10, and AM-3 and AM-4 removed assetvault-io as a stakeholder.)_
  - _Superseded 2026-11-23 by AM-3 and AM-4._ This entry used to name **assetvault-io** (`C:\Repo\assetvault-io`, `net10.0`) as the first known consumer. In Fred's words, this work "does not affect them because they are still using the Scott Brady library, not the fork" (AM-3), and every assetvault-io requirement is cut (AM-4). The facts recorded about it on 2026-10-25 stay below because Q-4, Q-11 and Q-16 cite them. They no longer describe a stakeholder:
    - It uses upstream `ScottBrady.IdentityModel` 4.1.0 for EdDSA JWT signing.
    - It builds keys with `EdDsa.Create(EdDsaParameters)` and `new EdDsaSecurityKey(...) { KeyId = kid }` (`src/AssetVault.BLL/Tokens/EdDsaKeyMaterial.cs:51-72`).
    - Its signing key is built from private material only, and its verification key from public material only.
    - Every key it constructs triggers S1, through upstream's package, which this work does not change.
    - It already runs on Microsoft.IdentityModel.JsonWebTokens 8.19.1 and BouncyCastle.Cryptography 2.6.2.
- **Consumers of upstream `ScottBrady.*` packages, inside ValhallaTech or outside it, assetvault-io included.** _Added 2026-11-23._ They are not affected by this work and are not stakeholders for migration. Nothing here changes the upstream packages they use. If one of them later moves to the fork, the changelog's record of breaking changes is the only migration aid (AM-3, AM-4, NG-9, NG-12).
- **Public developers.** They are future consumers once the libraries are published to nuget.org (R1-Q4). They inherit the same defects and the same lack of documentation. Only those on .NET 11 can consume the packages (C-2; _amended 2026-11-23, AM-3_).
- **End users of apps built on the libraries.** They fill in password forms that can return server errors (U10) or English-only, inconsistent messages (U11). Scale unknown.
- **Operators of host apps** that register their own crypto provider (HSM/KMS). S1 overrides it silently.
- **Security researchers and reporters.** They are routed to the upstream author (`SECURITY.md`). The fork offers no private alternative yet (C-21).
- **The fork's maintainers (Fred / ValhallaTech).** They cannot publish under the current identity. They have no meaningful warning signal, no coverage figure, no performance baseline, and CI on an SDK that went out of support on 2026-11-10 (amended 2026-11-23: tense only).
- **The original author, Scott Brady.** He receives the security reports and is named as author of a package line he does not maintain here.

### 1.3 Goals

_Outcomes that would mean the problem is solved. Observable, not implementation._

_Platform and identity_
- **G-1** — _Amended 2026-11-23 (Fred, AM-1 and AM-2), and again the same day (Fred, AM-3)._ ValhallaTech projects running on .NET 11 can consume both libraries from the private release. Both libraries build, and the full test suite passes, on .NET 11. There is no release date (C-20 as amended). (R4-Q1, AM-2, AM-3) The AM-1 wording, superseded by AM-3: "ValhallaTech projects running on .NET 10 and ValhallaTech projects running on .NET 11 can each consume both libraries from the private release. Both libraries build, and the full test suite passes, on .NET 10 and on .NET 11." Originally: "ValhallaTech projects running on .NET 10 can consume both libraries from the private release on or before 2026-11-10. Both libraries build, and the full test suite passes, on .NET 10. (R2-Q1, R2-Q3, R4-Q1)"
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
- **G-25** — The sample app is kept and still demonstrates the password-rules tag helper. It runs on .NET 11, the org standard (_amended 2026-11-23 from ".NET 10"; AM-1, then fixed by AM-3, C-2; A-16 and Q-26 are resolved_), and uses current hosting conventions and a supported UI framework, without jQuery in the sense of C-23. Its interactive elements, such as the mobile menu, keep working. Every input on its password form has an associated label. (U12, S6, GP-8, C-23; D-8)

_Supporting_
- **G-26** — An accidental breaking change to the public API after 1.0.0 is caught before it is released. (review §5)
- **G-27** — Coverage is measured on every CI build and meets the floor in C-12. (review §5, R4-Q3)
- **G-28** — After release, dependencies stay at their latest stable versions without anyone checking for new versions by hand. Advisories against dependencies reach the maintainers the same way. Code formatting is consistent and checked automatically. (review §5, GP-1; tools are D-7)
  - This covers the sample app's front-end libraries too. Their versions are declared where an automated update check can read them, instead of existing only as copied files in the repo (S6).
- **G-29** — _Superseded 2026-11-23 by Fred's AM-4 ("Cut all of it"); no longer a goal._ No project on upstream 4.1.0 is a migration stakeholder (NG-9 as amended). The changelog's record of breaking changes, already required by G-22, is the only migration aid (AM-4). Originally: "assetvault-io, and any other ValhallaTech project on upstream 4.1.0, can move to 1.0.0 from a written account of every breaking change that affects it and what replaces each one. (Q-4 resolution; this is the consequence A-4 named if it proved false)"

### 1.4 Non-goals

_Explicitly out of scope for this feature. Each with a one-line reason._

- **NG-1** — _Amended 2026-11-23 (AM-2): the date is dropped._ Publishing 1.0.0 to nuget.org. A private-feed release counts as shipped, and the public release follows later (R4-Q1, R1-Q4, C-4). Whether that public release belongs to this spec is Q-2. Originally: "Publishing to nuget.org by 2026-11-10. The deadline counts a private-feed release as shipped, …"
- **NG-2** — Supporting .NET 6, 8, 9 or 10 consumers. No consumer needs .NET 8 after 2026-11-10 (R2-Q1), and Fred dropped .NET 10 (AM-3: "Drop .NET 10."). _Amended 2026-11-23 (AM-1), then again (AM-3):_ R2-Q1's answer was ".NET 10 only"; AM-1 made the supported set .NET 10 and .NET 11; AM-3 made it .NET 11 only (C-2 as amended).
- **NG-3** — Staying mergeable with upstream or contributing fixes back to it. The fork is independent (R2-Q4).
- **NG-4** — Restoring Branca or PASETO support. The README still documents them, but the defect is the inaccurate documentation (G-22), not the missing feature.
- **NG-5** — Adding new algorithms or token formats, such as the post-quantum algorithms that arrived with .NET 10, or anything new in .NET 11. The move to .NET 11 is a runtime change, not a feature expansion. (_Amended 2026-11-23 to add .NET 11, AM-1, and to drop .NET 10 as a target, AM-3._)
- **NG-6** — Bringing ValhallaTech's broader application stack into the libraries, meaning the database, key-value, authorization, container, SPA, logging, mapping and validation components listed in review §8. The libraries have none of those concerns, and a library must not impose them on its consumers (review §8).
- **NG-7** — Process or infrastructure on the scale of Heimdall-TicketTracker. This is a small library, and Fred asked that it stay proportionate (C-17).
- **NG-8** — Removing private key copies held inside the third-party cryptography dependency. That memory is outside the library's control (review S3), so G-6 is bounded to memory the library controls.
- **NG-9** — _Amended 2026-11-23 (Fred, AM-4)._ Migrating any consumer of the upstream `ScottBrady.*` packages, ValhallaTech's own projects included, or giving them any migration aid beyond the changelog's record of breaking changes (G-22). They are upstream's users, not this independent fork's (R2-Q4), and Fred cut every assetvault-io requirement (AM-4). Originally: "Migrating existing nuget.org consumers of the upstream `ScottBrady.*` packages. They are upstream's users, not this independent fork's (R2-Q4). ValhallaTech's own projects on upstream 4.1.0 are not part of this non-goal; G-29 covers them."
- **NG-10** — Removing the sample app, or rebuilding it on a different front-end stack. Fred chose to keep it on its current page-rendering model, because that model is what demonstrates the tag helper (GP-8).
- **NG-11** — _Added 2026-11-23 (AM-1); superseded the same day by Fred's AM-3 ("Drop .NET 10."). No longer a non-goal: a consumer of 1.0.0 must be on .NET 11 (C-2)._ It read: "Requiring any consumer to move to .NET 11. assetvault-io keeps working on .NET 10 LTS; the .NET 11 standard applies to new work (AM-1, as described in the option Fred chose)."
- **NG-12** — _Added 2026-11-23._ Letting repositories outside this one shape this work, whether through their target framework, their dependency versions or their migration needs. Fred, verbatim (AM-3): "This is the problem with worrying about repos that are outside the scope of our work. They will eventually get updated. The work we're doing here does not affect them because they are still using the Scott Brady library, not the fork."

### 1.5 Constraints

_Fixed facts the solution must live within: regulatory, platform, timeline, data,
existing-system, organizational. Mark each `hard` or `soft`._

| ID | Constraint | Type | Source |
| --- | --- | --- | --- |
| C-1 | .NET 8 and .NET 9 reached end of support on 2026-11-10 and get no security patches afterward. .NET 10 is LTS until November 2028. _(Amended 2026-11-23: tense only. The cited source does not cover .NET 11; see Q-27, resolved: amended 2026-11-23 (Gate 2).)_ | hard | Microsoft, https://devblogs.microsoft.com/dotnet/dotnet-8-9-end-of-support/ |
| C-2 | _Amended 2026-11-23 (Fred, AM-1), and again the same day (Fred, AM-3)._ .NET 11 is the only .NET supported: by both packages, by the test suite and by the sample. .NET 6, 8, 9 and 10 are not supported. The AM-1 wording, superseded by AM-3: "Both packages must support .NET 10 and .NET 11. No consumer needs .NET 8 or .NET 9." Originally: "Only .NET 10 must be supported. No consumer needs .NET 8 after 2026-11-10." | hard | Fred, grilling R2-Q1 (no .NET 8); Fred, AM-3, 2026-11-23 ("Drop .NET 10."), which supersedes the .NET 10 half of AM-1 ("net10.0 + net11.0"). The tests and the sample are covered by the coordinator's relay of AM-3 (".NET 11 only"). |
| C-3 | _Superseded 2026-11-23 by Fred's AM-2 ("No hard date"); no longer a constraint._ Originally: "2026-11-10 is a hard deadline for shipping the .NET 10 version." | — (was hard) | Fred, grilling R2-Q3; superseded by Fred, AM-2 |
| C-4 | "Shipped" for 1.0.0 means a private-feed release that ValhallaTech projects can consume. The public nuget.org release follows later. _(Amended 2026-11-23: "for the deadline" became "for 1.0.0", because AM-2 dropped the deadline. R4-Q1's meaning of "shipped" is unchanged.)_ | hard | Fred, grilling R4-Q1 |
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
| C-15 | This spec covers the whole review: .NET 11 support, build/CI foundations, all four pillars, the sample app and the docs. _(Amended 2026-11-23: AM-1 added .NET 11, and AM-3 dropped .NET 10.)_ | hard | Fred, grilling R1-Q3; Fred, AM-1 and AM-3 |
| C-16 | Anyone who distributes a derivative work must include a copy of the license, put a prominent notice on each modified file saying it was changed, and keep all copyright, patent, trademark and attribution notices from the original source. | hard | `LICENSE` (Apache-2.0) §4(a)–(c) |
| C-17 | The repo's process and infrastructure stay proportionate to a small library and do not copy the complexity of Heimdall-TicketTracker. | soft | Fred, conversation before Stage 0 (relayed by /spec) |
| C-18 | FluentAssertions 8.0 and later are under the Xceed Community License, which is free only for non-commercial use. 7.x stays Apache-2.0 but gets bug fixes only. The repo is on 6.12.2 (Apache-2.0). | hard | FluentAssertions NuGet license page (https://packages.nuget.org/packages/FluentAssertions/8.3.0/License), as cited in review §7 and restated by /spec. The Analyst did not re-read it. Version 6.12.2 is from the test `.csproj`. |
| C-19 | The work must also satisfy all four pillars: security, performance, scalability and UX. UX covers both the consuming developer and the end user of an app built with the libraries. | hard | Fred, conversation before Stage 0 (relayed by /spec) |
| C-20 | _Amended 2026-11-23 (Fred, AM-2)._ There is no hard date. The complete 1.0.0 release ships when its release gates pass, carrying the scope in A-1. A prerelease does not count as shipped. Originally: "The deadline stays hard. The complete 1.0.0 release ships by 2026-11-10, carrying the scope in A-1. A prerelease does not count as shipped." | hard | Fred, Gate 1 pre-review GP-1 (complete 1.0.0, no prerelease); Fred, AM-2, 2026-11-23 ("No hard date"), which replaces GP-1's "Keep it hard." |
| C-21 | GitHub private vulnerability reporting is disabled on `IronSloth/IdentityModel-Supa`. Enabling it is a repository setting that Fred must change himself; no agent can make that change. | hard | GitHub API, which returned `{"enabled":false}` (verified 2026-10-25 by the /spec coordinator); Fred, Gate 1 pre-review GP-7 |
| C-22 | `IronSloth/IdentityModel-Supa` is the canonical repository for package metadata and vulnerability reports. | hard | Fred, Gate 1 pre-review GP-7 |
| C-23 | jQuery is never used in any ValhallaTech work, including this repository's sample app. That means no jQuery library files, no references to jQuery, and no package that depends on it. Optional jQuery-detection code inside another library's own bundle is not use of jQuery (GP-10). | hard | Fred, Gate 1 pre-review GP-8 ("jQuery should never be used in any of our work.") |
| C-24 | Base62 has a capped input length, and the encoding decided for D-3 is fixed (GP-2). The Architect does not reopen it. | hard | Fred, Gate 1 pre-review GP-2 |
| C-25 | _Added 2026-11-23; amended the same day (AM-3)._ ValhallaTech's org standard is .NET 11 with C# 15, and C# 15 applies to all of this repository's code, with no exception. | hard | Fred, AM-1, 2026-11-23 ("the new standard is .NET 11 C# 15"); Fred, AM-3 (chose "Drop .NET 10." over keeping it at C# 15's expense) |
| C-26 | _Added 2026-11-23 (AM-1); superseded the same day by Fred's AM-3. No longer a constraint: assetvault-io uses the upstream package, not the fork, so this work does not affect it (NG-12)._ It read: "The only known consumer, assetvault-io, targets `net10.0` and keeps working on .NET 10 LTS." | — (was hard) | Fred, AM-1; superseded by Fred, AM-3 |
| C-27 | _Added 2026-11-23._ Microsoft supports C# 15 only on .NET 11 and newer. Using a language version newer than a target framework's default is unsupported. | hard | Microsoft Learn, as read by the Architect on 2026-11-23 and cited in §3 G2-1. The Analyst did not re-read it. |

### 1.6 Open questions

_Anything that would change the shape of the work. Mark `blocking` if the PRD cannot
be written without an answer._

**Open**

| ID | Question | Blocking? | Owner |
| --- | --- | --- | --- |
| Q-2 | Is the public nuget.org release part of this spec's definition of done, or a separate later piece of work? Assumed separate (NG-1). | no | Fred |
| Q-5 | How large an improvement counts for each G-13 hot path? Is any measured gain enough, or is there a minimum per path? R3-Q4 requires measurement but sets no threshold. Assumed any measured gain (A-8). | no | Fred / PM |
| Q-6 | Does the 80% line-coverage floor (C-12) apply to each packable library separately or to both combined? Is the sample app excluded? Assumed each library separately, with the sample excluded (A-6). | no | Fred |
| Q-9 | What is the largest Base62 or Base16 input that consumers realistically need to handle? Do any consumers depend on a size bound? The Base62 cap is now fixed (GP-9). This question remains only to show whether a consumer needs more than the cap allows. No consumer of the fork is known. _(Amended 2026-11-23, AM-3: this used to cite assetvault-io, which uses neither encoder, as the only known consumer.)_ | no | Fred |
| Q-10 | Grounding found two defects the review does not report: unchecked offsets and lengths in `EdDsa.Verify(byte[], int, int, byte[], int, int)`, and its rejection of zero-length input where the other overload accepts it (`Tokens/EdDsa.cs:105-125`). Both are public entry-point input handling and are assumed covered by G-18 (A-11). Is that right? | no | Fred |
| Q-29 | _Added 2026-11-23._ With the hard date gone (AM-2), does the 1.0.0 / 1.x split still stand? Q-1's resolution let non-breaking work follow in 1.x when 1.0.0 had a date. Assumed the split stands unchanged (A-19). | no | Fred / PM |
No §2.6 question is open after Gate 1. Fred's Gate 1 answers are in the Resolved table below (GP-9).

**Resolved**

| ID | Question (short) | Resolution | Source |
| --- | --- | --- | --- |
| Q-1 | Is the full scope due by 2026-11-10? | Yes. The deadline stays hard, and the complete 1.0.0 release ships by 2026-11-10, carrying every behavior-changing fix (A-1 confirmed). Non-breaking work may follow in 1.x. _Amended 2026-11-23 (AM-2): the date and its hardness are dropped. The complete 1.0.0 still carries every behavior-changing fix (C-20 as amended). Whether the 1.x split still stands is Q-29._ | Fred, Gate 1 pre-review GP-1; Fred, AM-2 |
| Q-3 | Are the new IDs available on nuget.org? | Yes, both IDs are unused (404). nuget.org has no ValhallaTech owner and no reserved `ValhallaTech` prefix. | nuget.org lookup by the /spec coordinator, 2026-10-25 |
| Q-4 | Does any ValhallaTech project use `ScottBrady.*` today? | Yes. assetvault-io (`net10.0`) uses `ScottBrady.IdentityModel` 4.1.0 (`src/AssetVault.BLL/Tokens/EdDsaKeyMaterial.cs:51-72`, re-read by the Analyst). It is the first known consumer, and it is exposed to S1. A-4 is false, which adds G-29. _Amended 2026-11-23 (AM-3, AM-4): the facts stand, but assetvault-io uses the upstream package, not the fork, so it is not a consumer of this work. G-29 is superseded._ | /spec coordinator, 2026-10-25; Analyst re-read; Fred, AM-3 and AM-4 |
| Q-7 | Must each fix also be test-first at unit level? | Yes. Each fix is written test-first at unit level, not only per acceptance criterion (C-13). | Fred, Gate 1 pre-review GP-3 |
| Q-8 | Which repository is canonical, and is private reporting enabled? | `IronSloth/IdentityModel-Supa` (C-22). Private vulnerability reporting there is disabled, and only Fred can enable it (C-21). | Fred, Gate 1 pre-review GP-7; GitHub API, 2026-10-25 |
| Q-11 | Can consumers take Microsoft.IdentityModel 8.x? | Yes. assetvault-io already uses Microsoft.IdentityModel.JsonWebTokens 8.19.1 and BouncyCastle.Cryptography 2.6.2 (A-5 confirmed for the known consumer). _Amended 2026-11-23 (AM-3): the question no longer shapes the work. No consumer's version compatibility caps G-9, because repositories outside this one do not shape it (NG-12, A-5)._ | /spec coordinator, 2026-10-25; Fred, AM-3 |
| Q-12 (§2.6) | Base62 cannot be linear while keeping today's output. Which route? | A documented maximum input length, with longer input rejected, a time budget at that maximum, and measurably faster than today (G-14). | Fred, Gate 1 pre-review GP-2 |
| Q-14 (§2.6) | Remove the leftover constants now, or mark them obsolete? | Remove the six ChaCha20 constants and `KeyTypes.Ecdh` in 1.0.0 (G-20). | Fred, Gate 1 pre-review GP-4 |
| Q-15 (§2.6) | Does G-8 cover a `D` and `X` that do not belong together? | Yes. Validation rejects them (G-8, S11). | Fred, Gate 1 pre-review GP-5 |
| Q-20 (§2.6) | Would a 1.0.0 prerelease count as shipped? | No. The complete 1.0.0 release is required (C-20). | Fred, Gate 1 pre-review GP-1 |
| Q-22 (§2.6) | Should the PEM import overrides throw `NotSupportedException`? | Yes, in 1.0.0 (G-20, U13). | Fred, Gate 1 pre-review GP-6 |
| Q-16 (§2.6) | On disposal, is the caller's own `D` array zeroed, or only the library's copies? | Only the library's own copies. A caller-supplied array is never modified. assetvault-io passes its own array as `D` (`EdDsaKeyMaterial.cs:53`). _(Amended 2026-11-23: the assetvault-io line is an illustration only, since AM-3 removed it as a consumer. The resolution stands.)_ | PM decision in §2.6, as directed by the /spec coordinator |
| Q-19 (§2.6) | Should the sample's vendored client libraries be held to an any-severity advisory rule? | Yes. Every jQuery 3.3.1 advisory is rated medium, so a High-or-Critical threshold would miss them all (S6). | `gh api /advisories?affects=jquery@3.3.1`, run by the /spec coordinator, 2026-10-25 |
| Q-23 | What are the Base62 cap and time budget? | Confirmed: a cap of 1,024 bytes for `Encode` and 1,376 characters for `Decode`, and a budget of 3 ms mean per call at the cap. Measured at 1,024 bytes on Fred's machine *(scratch probe, 2026-10-25, 2,000 calls, not in the repo)*: 1.577 ms to encode and 0.809 ms to decode with the decided encoding, against 3.555 ms for today's encode, with 1,375-character output. | Fred, Gate 1 (GP-9) |
| Q-24 (§2.6) | What does Base62 throw for invalid input? | `FormatException` for invalid characters. `ArgumentException` for input over the cap. | Fred, Gate 1 (GP-9) |
| Q-13 (§2.6) | What is Ed448's `KeySize`? | 456. | Fred, Gate 1 (GP-9) |
| Q-17 (§2.6) | How are 0 and `null` treated in `MaxLength` and `MaxConsecutiveChars`? | 0 is rejected, and `null` means no limit, for both settings. This ships in 1.0.0 however the validator is registered. | Fred, Gate 1 (GP-9) |
| Q-18 (§2.6) | What is the package icon? | The ValhallaTech company logo, a PNG that Fred supplies before release. | Fred, Gate 1 (GP-9) |
| Q-21 (§2.6) | Do the specific exception types override G-18? | Confirmed. The types set by G-7, G-8 and G-20 override G-18's general argument-exception rule. | Fred, Gate 1 (GP-9) |
| Q-25 (§2.6) | When does the one-step setup helper ship? | In 1.0.0. Bad options fail at startup through the helper, and at first use through the old two-step path. | Fred, Gate 1 (GP-9) |
| Q-26 | Which .NET does the sample app run on? _(Added 2026-11-23, resolved the same day.)_ | .NET 11, as everything in the repo does (C-2). A-16 is confirmed. | Fred, AM-3, relayed by the /spec coordinator |
| Q-27 | Until when does Microsoft support .NET 11, and does that end before .NET 10's November 2028? _(Added 2026-11-23; amended 2026-11-23 (Gate 2): moved from Open to Resolved.)_ | .NET 11 is an STS release, first released 2026-11-10, with end of support on 2028-11-14. That is the same day .NET 10's LTS ends, so it does not end sooner. A-17 is confirmed. | Architect's fact check, §3.1 grounding: read 2026-11-23 in Microsoft's `releases-index.json` (channel 11.0: `"release-type": "sts"`, `"eol-date": "2028-11-14"`) and confirmed the same day on the .NET support policy page |
| Q-28 | What does the C# 15 org standard require of a repo that must also serve .NET 10 consumers? _(Added 2026-11-23, resolved the same day; this is §3's G2-1.)_ | .NET 10 is dropped (C-2), and C# 15 applies to all of the repo's code (C-25). A-18 was false, and Fred chose .NET 10 as the side that gives. | Fred, AM-3 ("Drop .NET 10.") |

### 1.7 Assumptions

_What is being taken as true in the absence of an answer, and what breaks if it's wrong._

- **A-1** — **Confirmed** by Fred, Gate 1 pre-review GP-1 (see C-20). _Amended 2026-11-23 (Fred, AM-1 and AM-2): the date is dropped and .NET 11 is added. Amended again the same day (AM-3): .NET 10 is dropped._ The 1.0.0 private-feed release, which ships when its release gates pass, carries .NET 11 support, the new identity and every behavior-changing fix (S2's default, U5's unit, the Base62 output change, the namespace rename). Non-breaking improvements may follow in later 1.x releases (whether that still holds is Q-29). This is the only reading consistent with R2-Q2 and R4-Q2 together (Q-1). → If false, a second major version is needed, which contradicts R2-Q2. Originally: "The 1.0.0 private-feed release due by 2026-11-10 carries .NET 10 support, … This is the only reading consistent with R2-Q2, R2-Q3 and R4-Q2 together (Q-1). → If false, either the deadline slips or a second major version is needed, which contradicts R2-Q2. Sixteen days is tight for the full breaking set."
- **A-2** — Both packages take the prefix from C-7: `ValhallaTech.Security.IdentityModel` and `ValhallaTech.Security.IdentityModel.AspNetCore`. Both restart at 1.0.0, replacing upstream's separate 4.1.0 and 2.2.0 lines. Basis: R3-Q1 and R4-Q2. → If false, the identity, namespace and changelog work changes shape, and the two packages need separate version plans.
- **A-3** — ValhallaTech will not buy commercial licenses for test-only tooling, so FluentAssertions 8.x is not an option. Basis: ValhallaTech is a business (/spec context) and C-18. → If false, D-1's option set widens.
- **A-4** — **Proved false** (Q-4 resolution): assetvault-io depends on upstream 4.1.0. Its stated consequence is now G-29. _Amended 2026-11-23 (AM-4): G-29 is superseded, and no project on upstream 4.1.0 gets migration guidance beyond the changelog (NG-9 as amended)._ The original assumption read: No ValhallaTech project depends on the upstream `ScottBrady.*` packages, so nobody inside ValhallaTech needs a migration path (Q-4). → If false, those projects need migration guidance for the identity and namespace change.
- **A-5** — _Superseded 2026-11-23 by Fred's AM-3: repositories outside this one do not shape this work (NG-12), so no consumer's version compatibility caps G-9's "latest stable" target. The if-false consequence no longer applies._ It read: **Confirmed for the known consumer** (Q-11 resolution): assetvault-io is already on Microsoft.IdentityModel 8.19.1 and BouncyCastle 2.6.2. Consuming ValhallaTech apps can accept whatever minimum Microsoft.IdentityModel and BouncyCastle versions the libraries require (Q-11). → If false, G-9's "latest stable" target is capped by consumer compatibility.
- **A-6** — The 80% floor applies to each packable library separately, and the sample app is excluded (Q-6). → If false, the coverage target and its CI enforcement change.
- **A-7** — **Confirmed** by Fred, Gate 1 pre-review GP-7 (now C-22). `IronSloth/IdentityModel-Supa` is the canonical repository for package metadata and vulnerability reports (Q-8). → If false, the project URL and the security-reporting destination point to the wrong place.
- **A-8** — For G-13, any measured improvement on each hot path is enough, as long as no measured path regresses. The PM sets firmer targets if Fred wants them (Q-5). → If false, some performance items may need deeper work than a straightforward fix.
- **A-9** — The review's findings hold at `555d31c`. The Analyst re-read S1 (`EdDsaSecurityKey.cs:16`), S2 (`ExtendedJsonWebKeyConverter.cs:19`), U5 (`EdDsa.cs:85`), P3 (`EdDsa.cs:112`), and the target frameworks and package metadata in each `.csproj`. The other findings were not re-run. → If one is false, its goal is moot. The test-first discipline (C-13) shows this cheaply, because the failing test cannot be written.
- **A-10** — The C3 short-read defect is real under the documented `Stream` contract, although nobody has reproduced it here. → If false, G-15 costs a test and nothing else.
- **A-11** — The two unreported `EdDsa.Verify` defects (Q-10) fall within G-18's scope. → If false, they need their own decision (fix in their own change, or drop with a reason), and G-18 should name them as excluded.
- **A-12** — "Proportionate" (C-17) excludes added process, services or pipelines that a two-package library cannot justify. It does not exclude the supporting goals G-26 to G-28, which the review places under "kept small" and R1-Q3 brings into scope. → If false, some of G-26 to G-28 become non-goals.
- **A-13** — In G-9's "every dependency is at its latest stable version", *every* means every **direct** dependency of every project: the libraries, the tests and the sample. Transitive dependencies resolve through their direct parents and are governed by G-9's advisory and end-of-life rule. Fred's wording (GP-1) is "all of the dependencies". → If false, transitive packages need their own currency rule, and pinning them can conflict with what their parent packages declare.
- **A-14** — At release, the latest stable version of every direct dependency works on .NET 11 and works alongside the others. → If false, G-9 needs an exception rule that Fred has not given, and the conflict surfaces as a release-time decision that holds 1.0.0 back until Fred rules (C-20). _(Amended 2026-11-23: .NET 11 added per AM-1; "against a hard deadline" removed per AM-2; .NET 10 removed per AM-3.)_
- **A-15** — **Confirmed** by Fred at Gate 1 (GP-9). The Base62 cap is 1,024 bytes for encoding and 1,376 characters for decoding, with a 3 ms budget, as the PM proposed. A measurement at 1,024 bytes (1.577 ms encode, 0.809 ms decode; Q-23) supports the budget. The if-false consequence no longer applies.
- **A-16** — **Confirmed** by Fred's AM-3, 2026-11-23 (Q-26 resolution, C-2): the sample runs on .NET 11. It read: The sample app runs on .NET 11, the org standard (C-25), because no consumer depends on it and NG-11's .NET 10 carve-out covers consumers only (Q-26). → If false, G-25 names .NET 10 instead, and the .NET 11 standard is not demonstrated by any runnable app in the repo.
- **A-17** — **Confirmed**, amended 2026-11-23 (Gate 2), by the Q-27 resolution: .NET 11 (STS) reaches end of support on 2028-11-14, the same day as .NET 10 LTS *(Architect, §3.1 grounding, read 2026-11-23 in Microsoft's `releases-index.json` and on the .NET support policy page)*. The if-false consequence no longer applies. It read: _Added 2026-11-23; amended the same day (AM-3)._ .NET 11 stays in Microsoft support at least until November 2028, when .NET 10's LTS ends (Q-27). → If false, the libraries' only supported .NET loses security patches before .NET 10 does. The problem in §1.1 point 1 then comes back on that date, and the supported set in C-2 has to be revisited.
- **A-18** — **Proved false** and resolved by Fred's AM-3, 2026-11-23 (Q-28 resolution): C# 15 cannot serve .NET 10 consumers in a supported way (C-27). Fred chose .NET 10 as the side that gives, so C# 15 applies to all of the repo's code (C-25) and .NET 10 is not supported (C-2). It read: The C# 15 standard (C-25) applies to all of this repo's code, and meeting it never costs .NET 10 consumers their support (C-2, C-26). → If false, Fred must say which gives: the C# 15 standard in this repo, or .NET 10 support.
- **A-19** — _Added 2026-11-23._ The 1.0.0 / 1.x split decided under the old date (Q-1, GP-1) stands unchanged now that the date is gone (Q-29). → If false, requirements now planned for 1.x move into 1.0.0, and 1.0.0 grows.

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
- D-10 Whether to keep the three-OS CI matrix and the CodeQL workflow. — _[Annotation added after Stage 0, not part of the verbatim record: **revised by Fred, GP-11.**]_
- D-11 The mechanism for localizing password validation error messages.

#### Gate 1 pre-review decisions
_Fred's decisions of 2026-10-25, asked before Gate 1, at Gate 1 (GP-9), at Gate 2 (GP-10) and after approval (GP-11), and relayed by the /spec coordinator. Recorded as relayed, and cited elsewhere as GP-n. Three are decided and fixed for the Architect: SimpleBase and the Base62 cap (GP-2, which decides D-3), the sample's UI framework (GP-8, which decides the UI framework part of D-8), and the private feed (GP-10, which decides D-2). Every other library or tool named here is Fred's stated preference, and that choice stays with the Architect under the D-item named. For example, GP-1's Renovate is a lean for D-7._

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
- **GP-11** (D-10, Fred, 2026-10-27, relayed by the /spec coordinator). This revises the earlier D-10 resolution, which kept all three operating systems.
  - **CI operating systems:** "Ubuntu only (Recommended)".
  - **Ubuntu version:** "Follow ubuntu-latest", instead of pinning Ubuntu 26.04.
  - **CodeQL:** stays.
  - **Reasons Fred accepted:**
    - BouncyCastle is pure C# with no OS crypto, so OS-specific bugs are unlikely.
    - Windows is already covered by local runs on ODIN01.
    - macOS was the slowest job and delays Gate 0.

#### Amendment of 2026-11-23 — target frameworks, deadline and assetvault-io
_Fred's answers of 2026-11-23, before any implementation, relayed by the /spec coordinator: AM-1 and AM-2 to two questions asked after approval, and AM-3 and AM-4 to two questions asked at Gate 2 on the amended spec. Recorded as relayed, and cited elsewhere as AM-n. The grilling record, the GP decisions above and AM-1 stay as written; where a later decision changes an earlier one, the §1.1–§1.7 items it touches are marked amended or superseded._

- **AM-1** (target frameworks; amends R2-Q1, C-2, G-1 and others). Fred, verbatim: "the new standard is .NET 11 C# 15". Asked how the 1.0.0 packages should target .NET, given that the only known consumer, assetvault-io, targets net10.0, Fred chose "net10.0 + net11.0": both packages are built for .NET 10 and .NET 11. The option he picked was described as: "Multi-target. assetvault-io keeps working on .NET 10 LTS; new work builds on .NET 11 / C# 15." Facts presented to him:
  - The .NET 11.0.100 SDK and 11.0.0 runtime are installed on ODIN01 alongside .NET 10.
  - assetvault-io's projects target net10.0 (checked in `C:\Repo\assetvault-io`).
  - _[Annotation added 2026-11-23, not part of the relayed record: **superseded in part by AM-3.** The .NET 10 half is dropped. The .NET 11 / C# 15 standard stands.]_
- **AM-2** (deadline; amends R2-Q3, GP-1's "Keep it hard.", C-3, C-20, A-1). The hard 1.0.0 deadline of 2026-11-10 (C-20, GP-1) passed with no code written. Fred chose "No hard date". The coordinator recorded it as: drop the hard deadline; 1.0.0 ships when its release gates pass.
- **AM-3** (Gate 2, §3 G2-1, C# 15 versus net10.0; supersedes the .NET 10 half of AM-1; amends C-2, C-25, C-26, G-1, NG-2, NG-11 and others). Fred was told that Microsoft supports C# 15 only on .NET 11, and that the packages also had to build for net10.0 for assetvault-io. He chose to drop .NET 10. Fred, verbatim: "Drop .NET 10. This is the problem with worrying about repos that are outside the scope of our work. They will eventually get updated. The work we're doing here does not affect them because they are still using the Scott Brady library, not the fork." The coordinator relayed the result as: both packages, the tests and the sample target .NET 11 only, and C# 15 applies to all code.
- **AM-4** (Gate 2, the assetvault-io requirements; supersedes G-29; amends NG-9). Fred was asked whether the migration guide proven against assetvault-io (R-54, AC-130/AC-131, the WI-45 dry run, the WI-47 post-release proof) should stay. Of three options, "Keep guide, drop proof", "Cut all of it" and "Keep all of it", he chose "Cut all of it". The option was described as: "Remove the migration guide and every assetvault-io requirement; CHANGELOG's breaking-change list is the only migration aid."

---

## 2. PRD

<!-- OWNER: Product Manager. No other agent may edit this section. -->

**Status:** ✅ complete

### 2.1 Summary

_Two or three sentences: what gets built, for whom, and what changes for them._

ValhallaTech releases the fork as `ValhallaTech.Security.IdentityModel` and `ValhallaTech.Security.IdentityModel.AspNetCore` 1.0.0. Both target .NET 11 only, the org standard, and all code is C# 15 (AM-3, C-2, C-25). The complete 1.0.0 release (not a prerelease) goes to the private feed when its release gates pass. There is no date (AM-2, C-20).

_Amended 2026-11-23 (AM-1, AM-2): .NET 10 and .NET 11, for assetvault-io; no date. Amended 2026-11-23 (AM-3/AM-4): .NET 11 only, and the assetvault-io migration path is cut. Originally: "Both target .NET 10 only, and the complete 1.0.0 release (not a prerelease, C-20) goes to the private feed by 2026-11-10."_

That release carries:
- the new identity;
- every change that would otherwise need a second major version;
- every direct dependency at its latest stable version, kept current automatically from then on;
- a changelog whose 1.0.0 entry lists every breaking change. It is the only migration aid (AM-4); the written migration guide for assetvault-io, its ACs and its metric are cut (CUT-1 to CUT-4).

Consumers get keys that leave process-wide crypto state alone, key-set exports that are public-only by default, and key validation that rejects mismatched key pairs. They also get strict, lossless encoders, with a documented cap and time budget on Base62. Argument errors name the argument, and key sizes and try-patterns follow .NET conventions. The work that is provably non-breaking follows in 1.x releases: further measured performance gains, the sample app's hosting and labels, the repository docs, and the remaining guardrails (API compatibility, coverage, formatting).

### 2.2 Requirements

_Each requirement traces to a goal from §1.3. `MUST` / `SHOULD` / `MAY`._

**Column key.**
- **Ships in** says which release the requirement may land in. Q-1 is resolved (GP-1, C-20) and Q-29 is decided (§2.2.1), so the column is firm. §2.2.1 sets the rule behind it and justifies every `1.x` row.
  - `1.0.0` means it must be in the 1.0.0 release. (_Amended 2026-11-23, AM-2: was "the deadline release"._)
  - `1.x` means it may follow in a later 1.x release without breaking a consumer. It is still required before this spec is done.
  - `all` means it applies to every requirement.
- **Provisional on** names the §1 assumption or §1.6/§2.6 question the requirement's current wording depends on. If that answer changes, the requirement and its ACs are amended. They are appended, never renumbered.

| ID | Requirement | Priority | Ships in | Satisfies | Provisional on |
| --- | --- | --- | --- | --- | --- |
| R-1 | Both libraries, every test project and the sample app each target exactly `net11.0` (C-2, AM-3). No project targets `net6.0`, `net8.0`, `net9.0` or `net10.0`.<br>_Amended 2026-11-23 (AM-1/AM-2): libraries and library tests on `net10.0` and `net11.0`, sample on `net11.0`. Amended 2026-11-23 (AM-3/AM-4): `net10.0` dropped (AM-3), so every project targets `net11.0` only; A-16 and Q-26 are resolved and A-18 / Q-28 is closed. Originally: "Both libraries target `net10.0` and nothing else. The test project and the sample app also target `net10.0`."_ | MUST | 1.0.0 | G-1, G-25 | — |
| R-2 | The solution builds on the .NET 11 SDK, and the full test suite runs on runtime 11.0.x, locally and in CI on every OS in the CI matrix, with no test skipped.<br>**At the 1.0.0 release,** every test passes except those marked `1.x` under the §2.3 convention "`1.x`-only checks never block 1.0.0". Marked tests verify only `1.x` requirements; they run and their results are recorded, but they need not pass (§2.2.1).<br>**From the release that carries a `1.x` requirement,** that requirement's tests must pass. At the release that completes this spec, the full test suite passes, every test included.<br>No test from `555d31c` is deleted without a stated reason.<br>_Amended 2026-11-23 (AM-1/AM-2): runs on 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): .NET 11 only. Amended 2026-11-23 (Gate 2), readiness finding F-6: the `1.x` exemption is stated here, so that 1.0.0 cannot ship with R-2's text false while AC-4 and AC-7 apply §2.2.1. Was "the full test suite passes on runtime 11.0.x". Originally: "on the .NET 10 SDK and runtime"._ | MUST | 1.0.0 | G-1 | — |
| R-3 | Version 1.0.0 of both packages is published to the ValhallaTech private feed once its release gates pass (C-20 as amended). There is no date (AM-2), and a prerelease does not count. The feed is GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json` (D-2 decided, GP-10). A `net11.0` project can restore it from that feed, gets the packages' `net11.0` build, and can use it to sign and validate an EdDSA JWT.<br>_Amended 2026-11-23 (AM-1/AM-2): no date; `net10.0` and `net11.0` consumers. Amended 2026-11-23 (AM-3/AM-4): `net11.0` consumer only. Originally: "no later than 2026-11-10", with a `net10.0` consumer._ | MUST | 1.0.0 | G-1 | A-2 |
| R-4 | The package IDs are `ValhallaTech.Security.IdentityModel` and `ValhallaTech.Security.IdentityModel.AspNetCore`, and both are version 1.0.0. | MUST | 1.0.0 | G-2 | A-2 |
| R-5 | Package metadata identifies ValhallaTech's fork: authors, project URL, repository URL, release-notes URL and icon. The only place upstream may appear is the retained copyright attribution (R-7). The icon is the ValhallaTech company logo, a PNG of at least 128×128 that Fred supplies before release (Q-18, FC-6). A package with no icon fails. | MUST | 1.0.0 | G-2 | FC-6 (Fred supplies the logo) |
| R-6 | Every public type is in a `ValhallaTech.Security.IdentityModel` namespace, and each assembly's identity matches its package ID. Shipped library source and sample source contain no `ScottBrady` identifier outside attribution or modification notices. | MUST | 1.0.0 | G-2 | — |
| R-7 | The Apache-2.0 obligations are met. Each package carries the license text. The original copyright notice is retained. Every library source file derived from upstream carries a notice that it was modified. | MUST | 1.0.0 | G-2 (C-16) | — |
| R-8 | Vulnerability reports go to GitHub private vulnerability reporting on the canonical repository, `IronSloth/IdentityModel-Supa` (C-22), and that reporting is enabled. `SECURITY.md` names no upstream contact. Enabling reporting is a manual step only Fred can take (C-21). No agent can satisfy AC-20. | MUST | 1.0.0 | G-3 | C-21 (Fred's action) |
| R-9 | Constructing, converting or using an EdDSA key never writes process-wide cryptographic state. EdDSA JWTs still sign and validate through Microsoft.IdentityModel with no global registration by the app. | MUST | 1.0.0 | G-4 | — |
| R-10 | Converting a key to a JWK yields public material only, unless the developer uses an explicit opt-in to export the private key (shape is D-4). | MUST | 1.0.0 | G-5 | — |
| R-11 | Disposing an EdDSA key zeroes every buffer of private-key material that the library allocated. The library never modifies an array the caller supplied, such as the array passed as `D`. Releasing a signature provider does not damage the key it was created from.<br>_Amended 2026-11-23 (AM-3/AM-4): the illustration "assetvault-io passes its own array as `D`" is removed, because assetvault-io is no longer a consumer (AM-3). The rule is unchanged._ | MUST | 1.0.0 | G-6 | — |
| R-12 | Base62 round-trips every byte sequence, including empty ones, all-zero ones, and ones with leading zero bytes. Decoding is canonical: every string `Decode` accepts re-encodes to itself. D-3 is resolved (GP-2): SimpleBase `Base62.Default` is the implementation. Its version follows R-17's latest-stable rule like every other dependency, not a pin (GP-10). It encodes each leading zero byte as `'0'`, and for inputs without leading zeros it matches today's output. | MUST | 1.0.0 | G-7 | — |
| R-13 | Malformed encoded input is rejected, never decoded, and never fails with an unrelated exception such as `NullReferenceException` or `IndexOutOfRangeException`.<br>**Base16:** any string with a character outside the alphabet, or with an odd length, throws `FormatException`, as .NET's own `Convert.FromHexString` does.<br>**Base62:** any string with a character outside the alphabet throws `FormatException`. The decided implementation, SimpleBase `Base62.Default` (D-3 resolved, GP-2; latest stable per GP-10), throws `ArgumentException`, and the library translates that into `FormatException` (Q-24, Fred, Gate 1).<br>Exceeding the Base62 cap is a bad argument, not bad content, so it stays `ArgumentException` (R-25).<br>Fred confirmed this split under Q-21: `FormatException` for bad encoded text, `CryptographicException` for invalid keys, `ArgumentException` for bad arguments. | MUST | 1.0.0 | G-7 | — |
| R-14 | Base16 round-trips every byte sequence, including the empty one. It keeps today's lowercase output and its acceptance of either case on input. | MUST | 1.0.0 | G-7 | — |
| R-15 | Key validation is the single gate. If `Validate()` passes, `EdDsa.Create` succeeds, the key can perform every operation its material allows without throwing, and a key holding both `D` and `X` verifies its own signatures. If validation fails, it throws `CryptographicException` with a message that says why: the key part and the required length, or that `D` and `X` do not belong to the same key pair (S11, GP-5). Which lengths are valid is D-5. Fred confirmed under Q-21 that `CryptographicException` for invalid keys overrides G-18's general rule. | MUST | 1.0.0 | G-8 | — |
| R-16 | At release, no project has a High or Critical advisory or a deprecated package, direct or transitive. The unmaintained analyzer is gone. Every direct dependency is under an OSI-approved license, which keeps test tooling free for commercial use (C-18). | MUST | 1.0.0 | G-9 | — |
| R-17 | At release, every direct dependency of every project is at its latest stable version. That covers the libraries, the tests and the sample, including the sample's vendored client-side libraries (A-13). Combined with R-16's license rule, this means the test assertion library cannot stay on FluentAssertions 6.12.2, because FluentAssertions' latest stable is under the Xceed license (C-18). D-1 must choose a library whose latest stable is OSI-licensed.<br>Every project builds for `net11.0` only (R-1), so each direct NuGet dependency has one required version: its latest stable version. If that version cannot be used on `net11.0`, A-14 is false for it, and Fred rules (A-14); no other exception applies. AC-49 defines "latest stable" and how it is read.<br>_Amended 2026-11-23 (AM-1/AM-2): Q-30 added to "Provisional on". Amended 2026-11-23 (Gate 1, Q-30): a per-framework rule for framework-bound dependencies was added. Amended 2026-11-23 (AM-3/AM-4): that rule is removed, because with one target framework it has nothing to apply to. The registration-API method stays in AC-49._ | MUST | 1.0.0 | G-9 | A-13, A-14 |
| R-18 | No third-party client-side library vendored in the sample has a known advisory of any severity, or is past its maintainer's end of life, at release time. This is a floor only. R-56 (GP-8) adds the stricter rules: Bootstrap 5.x at its latest stable, and no jQuery anywhere. | MUST | 1.0.0 | G-9, G-10 | — |
| R-19 | In the sample, no key configured for token validation holds private material. PII logging and the developer exception page are active only in the Development environment. | MUST | 1.x | G-10 | — |
| R-20 | Both libraries build with active .NET code analysis, including the security rules, and with no project-level suppression of analysis or of missing-XML-doc warnings. A Release build reports 0 warnings. | MUST | 1.0.0 | G-11 | — |
| R-21 | Every workflow declares its token permissions explicitly. The build workflow's token has no write scope. | MUST | 1.x | G-12 | — |
| R-22 | A committed measurement report gives the baseline (`555d31c`) and the after-change figures for every path and input size in measurement set T (§2.3). Both sides are measured on the same machine and the same .NET 11 runtime, so the comparison measures the code change and not the runtime change.<br>_Amended 2026-11-23 (AM-1/AM-2): measured on a .NET 10 and a .NET 11 runtime. Amended 2026-11-23 (AM-3/AM-4): .NET 11 is the only shipped build (C-2), so it is the only runtime measured. Originally: "on the same machine and the same .NET 10 runtime"._ | MUST | 1.0.0 | G-13 | Q-9 |
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
| R-45 | The sample runs on .NET 11 (G-25, C-2), uses minimal hosting, and gives every input on its password form an associated label. The supported-UI-framework part of G-25 is covered by R-56, which ships in 1.0.0.<br>_Amended 2026-11-23 (AM-1/AM-2): was "runs on .NET 10". Amended 2026-11-23 (AM-3/AM-4): A-16 is confirmed and Q-26 resolved, so the requirement no longer depends on them._ | MUST | 1.x | G-25 | — |
| R-46 | An automated API compatibility check against the published 1.0.0 baseline fails the build on any breaking change to the public API. It is active before the first release after 1.0.0. | MUST | 1.x | G-26 | — |
| R-47 | Every CI build measures line coverage for each packable library separately and fails if either is below 80%.<br>_Amended 2026-11-23 (AM-1/AM-2): one figure per framework run. Amended 2026-11-23 (AM-3/AM-4): one framework, so back to one figure per library, as originally written._ | MUST | 1.x | G-27 | Q-6 |
| R-48 | From the 1.0.0 release onward, an automated tool (D-7) proposes an update whenever any direct dependency in R-17's inventory gains a newer stable version. That includes the sample's client-side libraries and the CI workflow actions. Advisories against dependencies reach the maintainers the same way.<br>_Amended 2026-11-23 (Gate 1, Q-30): a framework-bound sentence was added. Amended 2026-11-23 (AM-3/AM-4): that sentence is removed with R-17's per-framework rule, so the text is as originally written._ | MUST | 1.0.0 | G-28 | A-13 |
| R-49 | Code formatting is checked automatically in CI (tool is D-7). | SHOULD | 1.x | G-28 | — |
| R-50 | Work is test-first at two levels (C-13, GP-3). Each acceptance test that verifies a behavior change exists, and fails by assertion, at a commit before the one that implements the change. Each fix is also preceded by at least one unit test that fails by assertion at its introducing commit.<br>**Which release this evidence gates.** For work that is not `1.x`-only, it gates 1.0.0. For `1.x`-only ACs and work items (§2.3 convention "`1.x`-only checks never block 1.0.0"), the evidence is audited and recorded at 1.0.0, but it gates the release that carries their requirements (AC-121, AC-132). The rule itself applies to all work, whenever it lands.<br>_Amended 2026-11-23 (Gate 2), with readiness finding F-6: the gating sentence is added, so that R-50's text matches AC-121 and AC-132 at 1.0.0. The test-first rule is unchanged._ | MUST | all | (C-13) | — |
| R-51 | The sample and the README password section both demonstrate the single registration step (R-28). | SHOULD | 1.x | G-17 | — |
| R-52 | The signing and verifying paths in T are also measured for Ed448. | MAY | 1.x | G-13 | — |
| R-53 | `EdDsa.ImportFromPem` and both `EdDsa.ImportFromEncryptedPem` overloads throw `NotSupportedException` (U13, GP-6). | MUST | 1.0.0 | G-20 | — |
| R-54 | **Cut 2026-11-23 (AM-4, Fred); CUT-1 in §2.5.** Not required, and counted in no total. The text is kept so that earlier references resolve: "A written migration guide covers every breaking change between upstream `ScottBrady.IdentityModel` 4.1.0 and 1.0.0, with what replaces each one. Applying that guide alone is enough for assetvault-io to build and pass its token tests on 1.0.0." The changelog's breaking-change list (R-42, AC-104) is the only migration aid (AM-4, NG-9). | cut (was MUST) | cut (was 1.0.0) | G-29 (superseded) | — |
| R-55 | No path in measurement set T regresses, in time or allocation, in any release, 1.0.0 included. | MUST | 1.0.0 | G-13 | — |
| R-56 | GP-8 sets the sample's front-end rules:<br>(a) The sample is kept. It stays a Razor app, because it demonstrates the `newpassword` tag helper.<br>(b) Its UI framework is Bootstrap **5.x** at its latest stable release on the release date: 5.3.8 on npm as of 2026-10-25, per the /spec coordinator. Bootstrap 4.x, including 4.6.2, is not acceptable, and neither is a 6.x prerelease.<br>(c) No jQuery appears in any form, anywhere in the repository. That means no jQuery file, no reference to it from any view, layout or manifest, and no front-end library that depends on jQuery. Fred, verbatim: "jQuery should never be used in any of our work."<br>This supersedes the parts of R-18 that asked only for "no advisories". | MUST | 1.0.0 | G-9, G-10, G-25 (GP-8) | — |

R-50 traces to a hard constraint (C-13), not a goal. It is listed so that the Agent Evaluator can check the test-first discipline from the commit history, not only from the plan.

R-55 is split out of R-23. 1.0.0 rewrites the code of every path in T for correctness reasons (R-12, R-13, R-25, R-31, R-33, R-44), so 1.0.0 is where a regression could enter. The improvement half (R-23) is non-breaking, so it may follow in 1.x.

#### 2.2.1 Release split (firm: GP-1, C-20 as amended; Q-29 decided by the PM)

_Amended 2026-11-23 (AM-1, AM-2). The rule's three tests and the requirement lists are unchanged. What changed is the reason for the split (now exposure, not a date), the feasibility paragraph, FC-1 and FC-3, and the new FC-7. No requirement moved between `1.0.0` and `1.x`._

_Amended 2026-11-23 (AM-3/AM-4). R-54 is cut (CUT-1), so the `1.0.0` list has 39 requirements. Test (b) no longer names G-29, and R-42's reason no longer rests on it. Q-29 is re-decided without the assetvault-io exposure reason; the split still stands. FC-3 is back to .NET 11 only, and FC-7 is closed. No requirement moved between `1.0.0` and `1.x`._

**The rule.** C-20 (as amended by AM-2) requires the complete 1.0.0, carrying A-1's scope, to ship when its release gates pass. There is no date. The Q-1 resolution, as amended, still lets non-breaking work follow in 1.x (A-1, A-19). A requirement is therefore `1.0.0` if any of these holds:
- **(a) It would break a consumer after 1.0.0,** and so need a second major version (C-5). Breaking means any of:
  - removing or renaming public API;
  - changing a public signature, a returned value or an output format;
  - throwing a different exception type for an input that already threw;
  - throwing where the code used to succeed or return;
  - raising a dependency's minimum major version;
  - adding warnings that a consumer's warnings-as-errors build would turn into errors.
- **(b) A goal ties it to "release" or "after release".** G-1, G-9, G-22 (its changelog clause, "records every release") and G-28 do this. (_Amended 2026-11-23 (AM-3/AM-4): was "G-1, G-9, G-28 and G-29". G-29 is superseded by AM-4, and G-22's changelog clause is named because R-42 now rests on it._)
- **(c) It guards code that 1.0.0 rewrites anyway.** Deferring it would ship that rewrite unchecked.

Everything else may follow in 1.x. Following .NET's breaking-change conventions, making a call succeed where it used to throw is **not** breaking. `1.x` means *may* follow: it never forbids a requirement from landing in 1.0.0, provided that does not hold the 1.0.0 release gates back.

**Why the split stands without a date or a waiting consumer (Q-29, re-decided by the PM on 2026-11-23; Fred may overrule).** _Amended 2026-11-23 (AM-3/AM-4). The first decision rested on exposure: assetvault-io triggering S1 until 1.0.0 shipped. AM-3 removed that reason, since assetvault-io uses upstream's package, which this work does not change (§1.2)._
- Tests (a) and (c) never depended on the date or on any consumer. (a) follows from C-5, one major version. (c) follows from not shipping rewritten code unguarded.
- Test (b) depends on what a goal ties to the release, not on when the release happens or who is waiting. G-1, G-9, G-22's changelog clause and G-28 still name the release.
- With no date and no known consumer, the reason to keep 1.0.0 lean is delivery. G-1 is met only when ValhallaTech projects on .NET 11, the org standard (C-25), can consume the packages, and that starts at 1.0.0. Every `1.x` requirement either changes no package behavior (repository, CI and sample work) or is a non-breaking addition, so holding 1.0.0 back for it delays G-1 and gives a consumer nothing. Every `1.x` requirement is still required before the spec is done.
- So A-19 holds, and no requirement moves between `1.0.0` and `1.x`. The .NET 11 build (R-1, R-2, R-3) is `1.0.0` because AM-1 and AM-3 answer how the 1.0.0 packages target .NET.

**`1.0.0`, 39 requirements:** R-1 to R-18, R-20, R-22, R-25 to R-31, R-33 to R-38, R-42, R-44, R-48, R-53, R-55 and R-56, plus R-50 throughout. (_Amended 2026-11-23 (AM-3/AM-4): was 40, with R-54, which is cut (CUT-1)._) These moved into 1.0.0 after the first pass:
- **R-29 (invalid password options rejected), by (a). Moved at Gate 1.** Fred's Q-17 answer makes 0 invalid and requires rejection however the validator is registered. A configuration that starts today would then fail, which is breaking.
- **R-28 (single-step setup helper). Moved at Gate 1 by Fred's Q-25 answer ("Helper in 1.0.0").** It is additive, so the rule alone would allow 1.x. It is in 1.0.0 so that R-29's startup rejection (AC-76) ships in 1.0.0, not only first-use rejection. (_Amended 2026-11-23, AM-2: was "ships at the deadline"._)
- **R-51 stays `1.x` (SHOULD).** It only makes the sample and README demonstrate the helper. It changes no package behavior, and it still makes sense in a later release.
- **R-22 (measurement report), by (c).** R-25(c)–(d) needs the measurement harness and the `555d31c` baseline in 1.0.0 regardless, and R-55 needs the full set T.
- **R-26 (fragmented stream reads), by (a).** `TryRead` returning `true` where it returned `false` is a changed return value. R-31 rewrites the same method for 1.0.0 anyway.
- **R-35 (key ID in the JWK), by (a).** The converter's output changes, and R-10 rewrites the converter for 1.0.0 anyway.
- **R-42 (changelog), by (b).** G-22 requires the changelog to record every release, so the 1.0.0 entry and its breaking-change list must exist when 1.0.0 is released. Under AM-4 that list is also the only migration aid. (_Amended 2026-11-23 (AM-3/AM-4): was "G-29's migration guide (R-54) depends on an authoritative list of breaking changes at 1.0.0." G-29 is superseded and R-54 is cut._)
- **R-48 (automatic dependency updates), by (b).** G-28 says "after release", so the tool must be active from the release.
- **R-53, R-54, R-55, R-56** are new.
  - R-53 changes an exception type, so (a).
  - R-54 was tied to release by G-29, so (b). It is cut (CUT-1, AM-4).
  - R-55 is the guard described under (c).
  - R-56 is part of G-9's "latest stable at release" for the sample's front-end, so (b).

**`1.x`, 15 requirements, each with the reason it does not break C-20 or C-5:**

| Req | Why it may follow 1.0.0 |
| --- | --- |
| R-19 | Sample-only (not package content). No consumer of the packages can break. |
| R-21 | CI token scope. Not package content. |
| R-23 | Additional speed-ups do not change results. R-55 already stops 1.0.0 from regressing. |
| R-24 | Base16 linear scaling changes no result. In practice the 1.0.0 rewrite for R-13/R-14 will likely meet it early. |
| R-32 | Throw → success. Not breaking under .NET rules. (_Amended 2026-11-23 (AM-3/AM-4): the sentence "assetvault-io verifies with public-only keys, so it does not need R-32 at 1.0.0" is removed, because assetvault-io is not a consumer. The rule alone keeps R-32 in `1.x`._) |
| R-39, R-40 | Repository README content. Not package content. |
| R-41 | Adding a package readme in a later 1.x changes no API or behavior. |
| R-43 | Throw → validation result. Not breaking under .NET rules. |
| R-45 | Sample hosting style and labels. In 1.0.0, R-1, R-18 and R-56 already make the sample build on .NET 11, on Bootstrap 5.x, with no jQuery. (_Amended 2026-11-23, AM-1: was ".NET 10"._) |
| R-46 | Needs the published 1.0.0 as its baseline, so it cannot exist before 1.0.0. It is required before the first later release. |
| R-47 | Coverage measurement and the CI floor change nothing in the packages. C-12 defines when the spec is done, not what 1.0.0 contains. |
| R-49, R-51, R-52 | SHOULD/MAY. Repository and sample only. |

**Feasibility.** _Amended 2026-11-23 (AM-1, AM-2), and again (AM-3/AM-4)._ The 1.0.0 set has 39 requirements, after the AM-4 cut of R-54. The original paragraph measured the set against 2026-11-10, 16 days from the brief. That date passed with no implementation started, and AM-2 removed it, so feasibility is no longer a question of fitting the set into a window. R-28 is in by Fred's choice; every other requirement is there because C-20 and C-5 require it, so the PRD still has nothing to move out. AM-3 removed the second framework that AM-1 had added, so each test runs once, on .NET 11. What remains are release preconditions. 1.0.0 cannot pass its release gates unless all of these hold:

| ID | Condition |
| --- | --- |
| FC-1 | The IronSloth GitHub Packages NuGet feed (`https://nuget.pkg.github.com/IronSloth/index.json`, D-2 decided, GP-10) accepts a push of both 1.0.0 packages from a credential with `write:packages`, and a consumer can restore them with a `read:packages` credential. Both must be in place, and a trial push and restore done, before the release gates run. Credentials are Fred's to issue. (_Amended 2026-11-23, AM-2: was "well before 2026-11-10"._) |
| FC-2 | Fred enables private vulnerability reporting (C-21) before release. Otherwise AC-20 fails. |
| FC-3 | A-14 holds. Every latest-stable direct dependency, including the Microsoft.IdentityModel 7 → 8 major bump and the test-tooling majors, works on .NET 11 and with the others. (_Amended 2026-11-23, AM-1: ".NET 10 and .NET 11". Amended 2026-11-23 (Gate 1, Q-30): a framework-bound sentence added. Amended 2026-11-23 (AM-3/AM-4): .NET 11 only, so the framework-bound sentence is removed. Originally: ".NET 10"._) |
| FC-4 | The test-assertion library swap (D-1) is mechanical. |
| FC-5 | **Holds, as measured.** SimpleBase meets the 3 ms budget at 1,024 bytes (AC-70). On Fred's machine it averages 1.577 ms to encode and 0.809 ms to decode, against 3.555 ms for today's encode, and the output is 1,375 characters *(scratch probe on Fred's machine, 2,000 calls, 2026-10-25, relayed by the /spec coordinator)*. AC-70 and AC-127 still have to be run on the release candidate, on .NET 11 (_amended 2026-11-23, AM-3/AM-4: was "on both runtimes"_); this measurement predicts their result but does not replace them. |
| FC-6 | Fred supplies the ValhallaTech logo as a PNG of at least 128×128 before release (Q-18). Otherwise AC-12 fails. No agent can produce this asset. |
| FC-7 | _Added 2026-11-23 (AM-1). Closed 2026-11-23 (AM-3/AM-4): A-18 proved false, and Fred resolved it by dropping .NET 10 (Q-28), so this is no longer a condition._ It read: "A-18 holds: meeting the C# 15 standard (C-25) costs the `net10.0` build nothing, so R-1, R-2 and R-3 can all be met (Q-28)." |

If any of these fails, 1.0.0 cannot pass its release gates as scoped. It is held, and Fred decides what gives: C-5, C-20's complete-release rule, the scope, or the failing condition. The PRD does not choose. (_Amended 2026-11-23: was "C-3, C-5 and C-20 cannot all be met". C-3 is superseded by AM-2._)

### 2.3 Acceptance criteria

_Ground truth for evaluating execution agents. Every AC is written to be assertable._

**Conventions for every AC below.**
- **Names.** Types and members are named as at `555d31c`, inside the renamed namespaces. If the Architect renames a type or member, §3.3 must map the old name to the new one, and the AC follows the mapping.
- **Exceptions.** "Throws `X`" means `X` or a subtype. `ParamName` checks are exact and ordinal.
- **Theories.** "Both curves" means the AC is parameterized over `Ed25519` and `Ed448`, and each case must pass.
- **Frameworks.** _Added 2026-11-23 (AM-1). Amended 2026-11-23 (AM-3/AM-4): the "both frameworks" parameterization is withdrawn._ Every project targets `net11.0` only (R-1), so every AC runs on, builds for or inspects the `net11.0` build and runtime 11.0.x. No AC is parameterized over frameworks. It read: "'Both frameworks' means the AC is parameterized over `net10.0` and `net11.0` (runtimes 10.0.x and 11.0.x), and each case must pass. …"
- **Seeds.** "Fixed seed" means a deterministic random generator whose seed is recorded in the test.
- **Shared state.** ACs that touch process-wide state (AC-21 to AC-25, AC-53) need test isolation. How to isolate them is the Architect's call.
- **Network-dependent ACs.** AC-133 and AC-137 to AC-140 and AC-143 need `registry.npmjs.org`; AC-50 needs the GitHub Advisory Database; AC-49 needs `api.nuget.org` (_amended 2026-11-23, Gate 1, Q-30_). Without network access, each of these **fails** with an explicit "registry unreachable" reason naming the URL that failed. It is never passed, skipped or marked inconclusive (AC-138 gives the full rule).
- **Measurement ACs** (AC-60 to AC-70, AC-127) run in a separate measurement process against built assemblies on one named machine. Their pass or fail depends on the figures that run measures. They are classified `integration` because the template has no `measurement` level. They are evidence runs, not per-commit CI gates.
  - **AC-59 and AC-123 are not measurement ACs.** Each parses a committed report file and checks only its shape: row count, required fields, matching runtime and machine. They measure nothing, so they stay `lint`, as their `Executable as` lines say. Like every `lint` AC, they need no test-first chain.
  - _Amended 2026-11-23 (Gate 2), readiness finding F-2: AC-59 and AC-123 were removed from this list. It read "AC-59 to AC-70, AC-123, AC-127", which contradicted their `lint` level._
- **Interfaces not designed yet.** Some ACs refer to an interface that is still to be designed ("the private-export opt-in", "the single registration step", "the message replacement mechanism"). They mean whatever §3.3 defines for D-4, R-28 and D-11. The assertion is on behavior only.
- **`1.x`-only checks never block 1.0.0.** _Added 2026-11-23 (Gate 2), to make the gating ACs match §2.2.1, under which a `1.x` requirement must not hold the 1.0.0 release gates back._
  - A **`1.x`-only AC** is one whose `Verifies` line names only requirements that ship in `1.x` (§2.2). As of this amendment they are AC-52 to AC-54, AC-58, AC-60 to AC-67, AC-69, AC-87, AC-88, AC-100 to AC-103, AC-106 to AC-108, AC-112 to AC-117, AC-120, AC-122 and AC-123. An AC that also verifies any `1.0.0` or `all` requirement is not `1.x`-only.
  - A **`1.x`-only test** carries an xUnit trait, whose name and value §3 fixes, that marks it `1.x`. It also carries a trait, fixed by §3, listing the requirement IDs it verifies. A test may carry the `1.x` marker only if every requirement it lists ships in `1.x`. Every other test, including every test with no marker, is blocking.
  - A **`1.x`-only work item** is a §3.4 work item whose `satisfies` list names only `1.x`-only ACs.
  - **At the 1.0.0 release commit,** every `1.x`-only AC, test and work-item audit is still run, and never skipped, and its result is recorded with the release evidence. A failure is reported but does not fail 1.0.0 or any AC that gates it: AC-4, AC-7, AC-121, AC-132 or AC-144.
  - **From the release that carries a `1.x` requirement,** and at the release that completes the spec, its checks block like any other (_amended 2026-11-23 (Gate 2): was "the commit that completes this spec", now matching R-2_). The exemption covers only 1.0.0 and releases that do not carry that requirement.
  - **Misuse check.** Each gating AC below also asserts that no `1.x`-marked test lists a requirement shipping in `1.0.0` or `all`. A marker cannot hide a `1.0.0` failure.

#### AC-1 — Packages carry exactly the .NET 11 build

_Amended 2026-11-23 (AM-1/AM-2): the folder set was {`lib/net10.0/`, `lib/net11.0/`}. Amended 2026-11-23 (AM-3/AM-4): `net11.0` only. Originally: "only a .NET 10 build", with the folder set {`lib/net10.0/`}._

- **Verifies:** R-1
- **Given** the release commit
- **When** `dotnet pack -c Release` produces both `.nupkg` files
- **Then** each package contains exactly one target-framework folder, `net11.0`
- **Assertion:** the set of distinct `lib/<tfm>/` folders in each `.nupkg` == {`lib/net11.0/`}, and that folder contains `<package ID>.dll`. No entry exists for `net6.0`, `net8.0`, `net9.0` or `net10.0`.
- **Executable as:** build

#### AC-2 — Test projects and sample target .NET 11

_Amended 2026-11-23 (AM-1/AM-2): library tests on {`net10.0`, `net11.0`}, sample on {`net11.0`}. Amended 2026-11-23 (AM-3/AM-4): every project on {`net11.0`}; A-16 is confirmed, so the AC no longer depends on it. Originally: "`net10.0` only"._

- **Verifies:** R-1
- **Given** the release commit
- **When** the evaluated target frameworks of every test project and of the sample project are read
- **Then** each targets `net11.0` only
- **Assertion:** `dotnet msbuild <project> -getProperty:TargetFramework -getProperty:TargetFrameworks` returns {`net11.0`} as the set of frameworks for every test project and for the sample project. No project returns `net6.0`, `net8.0`, `net9.0` or `net10.0`.
- **Executable as:** build

#### AC-3 — Solution builds on the .NET 11 SDK

_Amended 2026-11-23 (AM-1/AM-2): was "a 10.0.x SDK as the only SDK selected". The solution now targets `net11.0`, which a 10.0 SDK cannot build. Amended 2026-11-23 (AM-3/AM-4): "every target framework" dropped, since there is one._

- **Verifies:** R-2
- **Given** a clean clone at the release commit and an 11.0.x SDK selected
- **When** `dotnet build -c Release` runs at the repository root
- **Then** the build succeeds for every project
- **Assertion:** exit code == 0.
- **Executable as:** build

#### AC-4 — Full suite passes on the .NET 11 runtime

_Amended 2026-11-23 (AM-1/AM-2): runs on 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): one run per test project, on 11.0.x. Originally: one run on runtime 10.0.x._

- **Verifies:** R-2
- **Given** the build from AC-3
- **When** `dotnet test -c Release` runs for every test project, with a results logger
- **Then** every blocking test passes on runtime 11.0.x, none is skipped, and every `1.x`-only test's result is recorded
- **Assertion:**
  - For every test project, the results hold a run whose recorded runtime is 11.0.x. No run records any other runtime.
  - Across all runs, skipped == 0, and failed == 0 among tests without the `1.x` marker (convention "`1.x`-only checks never block 1.0.0").
  - Every `1.x`-marked test appears in the results with an outcome of passed or failed. A failed one is listed in the release evidence and does not fail this AC at the 1.0.0 release commit.
  - The count of `1.x`-marked tests whose requirement trait names a requirement shipping in `1.0.0` or `all` == 0.
  - At the release commit of the release that completes the spec, failed == 0 across all tests. (_Amended 2026-11-23 (Gate 2): was "At the commit that completes this spec", now matching R-2._)

  _Amended 2026-11-23 (Gate 2). The assertion was "exit code == 0 … failed == 0 and skipped == 0" over every test, which let `1.x` acceptance tests block 1.0.0, against §2.2.1. The exit code is no longer asserted, because a failing `1.x`-marked test sets it non-zero. The results file is the evidence._
- **Executable as:** build

#### AC-5 — No test silently dropped

- **Verifies:** R-2
- **Given** the fully qualified test method names at `555d31c` and at the release commit, with the namespace renamed per C-8
- **When** the two name sets are compared
- **Then** every test that is missing at release is listed in §3 with the behavior it covered and the reason it was removed
- **Assertion:** (names at `555d31c` − names at release) ⊆ the removed-test list in §3. Count of unlisted removals == 0.
- **Executable as:** lint

#### AC-6 — CI selects only .NET 11

_Amended 2026-11-23 (AM-1/AM-2): 10.0 or 11.0 accepted. Amended 2026-11-23 (AM-3/AM-4): 11.0 only. Originally: "only the .NET 10 SDK", matching `^10\.0…` alone._

- **Verifies:** R-2
- **Given** `.github/workflows/*.yml` at the release commit
- **When** every .NET version that every setup-dotnet step selects is read: each line of its `dotnet-version` input, or the SDK version in the `global.json` its `global-json-file` input names
- **Then** each one selects an 11.0 version
- **Assertion:** every value matches `^11\.0(\.x|\.\*|\.\d+)?$`. The count of values selecting 6.x, 8.x, 9.x or 10.x == 0.
- **Executable as:** lint

#### AC-7 — CI is green on .NET 11

_Amended 2026-11-23 (AM-1/AM-2): library tests on 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): 11.0.x only. Originally: "on .NET 10 across the matrix", with runtime 10.0.x in each job. How jobs are split across operating systems is §3's call._

- **Verifies:** R-2
- **Given** the CI workflow run for the release commit
- **When** its jobs complete
- **Then** every job succeeds, the tests ran on runtime 11.0.x, and every blocking test passed
- **Assertion:**
  - `gh run view <run-id> --json jobs` shows every job with `conclusion == "success"`.
  - Across that run's uploaded test results, every test project ran on runtime 11.0.x, and failed == 0 among tests without the `1.x` marker.
  - Every `1.x`-marked test appears in those results with an outcome, under convention "`1.x`-only checks never block 1.0.0". A failed `1.x`-marked test, or a failed `1.x`-only check such as AC-58 or AC-120, does not fail this AC at the 1.0.0 release commit. It must therefore not turn a job's conclusion to failure; how is §3's.

  _Amended 2026-11-23 (Gate 2): was failed == 0 over every test, which let `1.x` checks block 1.0.0 through CI._
- **Executable as:** integration

#### AC-8 — 1.0.0 is on the private feed

_Amended 2026-11-23 (AM-1/AM-2): the bound "published timestamp ≤ `2026-11-10T23:59:59-05:00`" is removed, because AM-2 removed the date and C-20 no longer has one. That a release follows its gates is now AC-144._

- **Verifies:** R-3
- **Given** the ValhallaTech private feed: GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json` (D-2 decided, GP-10), queried with a `read:packages` credential
- **When** the feed's package metadata is queried for both package IDs at version 1.0.0
- **Then** both exist as the release version 1.0.0
- **Assertion:** for each ID, the metadata query returns version `1.0.0` exactly, with a published timestamp. A prerelease such as `1.0.0-rc.1` does not satisfy this AC (C-20).
- **Executable as:** integration

#### AC-9 — A .NET 11 consumer can use the feed packages

_Amended 2026-11-23 (AM-1/AM-2): `net10.0` and `net11.0` consumers. Amended 2026-11-23 (AM-3/AM-4): a `net11.0` consumer only. Originally: a `net10.0` consumer._

- **Verifies:** R-3
- **Given** a new `net11.0` console project that references both packages at 1.0.0 as package references, with no project or local-folder references, so the ValhallaTech packages resolve from the IronSloth GitHub Packages feed (GP-10) using a `read:packages` credential
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

_Amended 2026-11-23 (Gate 1, Q-30): a per-framework "required version" for framework-bound dependencies, replacing `dotnet list package --outdated`. Amended 2026-11-23 (AM-3/AM-4): every project builds for `net11.0` only, so the per-framework rule is removed. Each dependency is held to its single latest stable version, and the NuGet registration API stays as the method. Originally: "at its latest stable version", checked by `dotnet list package --outdated`._

- **Verifies:** R-17
- **Given** the release commit and the release date, and every project: both libraries, the tests and the sample. For each project and each direct `PackageReference` D (A-13), the resolved version is read from the `net11.0` entry of `targets` in the project's `obj/project.assets.json` after restore.
- **When** each D's latest stable version is read from the NuGet V3 registration resource (`RegistrationsBaseUrl` in `https://api.nuget.org/v3/index.json`), as of the release date:
  - **Stable** means a listed version with no prerelease label.
  - **Latest stable** means the highest stable version under NuGet's version ordering.
  - **Supports `net11.0`** means the version's catalog entry declares a dependency group whose `targetFramework` NuGet treats as compatible with `net11.0` (`net11.0` itself, or an earlier framework it can consume). If the catalog entry declares no dependency groups, the `lib/<tfm>/` folder names in that version's `.nupkg` are used instead, under the same rule.
- **Then** every direct dependency of every project is at its latest stable version
- **Assertion:**
  - For every (project, D), the resolved version == D's latest stable version. The count of mismatches == 0.
  - There is no recorded-gap escape: G-9 gives no exception rule (A-14).
  - If D's latest stable version does not support `net11.0`, this AC fails for D, and Fred rules under A-14.
  - At baseline, the mismatches include Microsoft.IdentityModel.Tokens 7.7.1 and BouncyCastle.Cryptography 2.4.0.
- **Executable as:** lint
- **Needs network:** yes. The check needs HTTPS access to `api.nuget.org`. If the registration resource, or a `.nupkg` needed for the `lib/` fallback, cannot be reached, the check **fails**, reporting that the NuGet registry is unreachable and naming the URL that failed. It never passes, and it is never skipped or marked inconclusive.

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
- **Given** the sample hosted in-process on `net11.0` (_amended 2026-11-23, AM-1/AM-2: was `net10.0`; R-1, A-16_)
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
- **Same conditions.** _Amended 2026-11-23 (AM-1/AM-2): measured on a .NET 10.0.x and a .NET 11.0.x runtime. Amended 2026-11-23 (AM-3/AM-4): one .NET 11.0.x runtime. Originally one .NET 10.0.x runtime._ Baseline and after run on the same machine and the same .NET 11.0.x runtime, with the same inputs, in the same session. `555d31c` does not target .NET 11, so the baseline must be rolled forward or rebuilt for .NET 11; how is the Architect's call. The point is to measure the code change, not the runtime change.
- **Per runtime.** _Added 2026-11-23 (AM-1). Withdrawn 2026-11-23 (AM-3/AM-4): there is one runtime, so every comparison in AC-60 to AC-70, AC-123 and AC-127 is made on .NET 11._
- **Report contents.** For each path and size, the report gives mean time per operation, its error margin (confidence half-width, at a stated confidence level of at least 95%), and allocated bytes per operation.
- **Improved.** Time is improved when after-mean + after-error < baseline-mean − baseline-error. Allocation is improved when after-bytes < baseline-bytes.
- **Regressed.** Time has regressed when after-mean − after-error > baseline-mean + baseline-error. Allocation has regressed when after-bytes > baseline-bytes.
- **Thresholds.** Any improvement is enough (A-8). Firmer per-path targets wait on Q-5.

#### AC-59 — The measurement report exists and is comparable

- **Verifies:** R-22
- **Given** the measurement report committed at the path §3 names
- **When** it is parsed
- **Then** it holds a baseline row and an after row for every path and size in T, taken under identical conditions on .NET 11
- **Assertion:** _(amended 2026-11-23, AM-1/AM-2: 4 × the pairs, on 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): 2 × the pairs, all on 11.0.x. Originally 2 × the pairs, all on 10.0.x.)_
  - The row count == 2 × (the number of path-size pairs in T).
  - Every row has a mean, an error margin, a confidence level of at least 95%, allocated bytes, a runtime version and a machine identifier.
  - Every row's runtime version is the same 11.0.x version, and every row's machine identifier is the same.
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
  - the calibration figure B: the mean time of the `555d31c` `Base62.Encode` on a 16,384-byte random input with a nonzero first byte, run on the same .NET 11 runtime.
- **When** the mean time per call of the release candidate's `Base62.Encode` on the 1,024-byte input, and of `Base62.Decode` on its encoding, is measured on that runtime
- **Then** each call stays within the budget, scaled to the machine's speed
- **Assertion:** for each operation, mean ≤ 3 ms × (B ÷ 580 ms). On Fred's machine, which produced the GP-2 figures and where B is about 580 ms, the budget is 3 ms. The report records B, the scale factor and both means.

  _Amended 2026-11-23 (AM-1/AM-2): measured on each of two runtimes. Amended 2026-11-23 (AM-3/AM-4): one .NET 11 runtime. Originally one measurement, with B on "the same .NET 10 runtime"._
- **Executable as:** integration

**Why these values.** Proposed by the PM (A-15) and confirmed by Fred at Gate 1 (Q-23). The reasoning below was written before the 1 KiB measurement; the measured figures follow it.

The cap is **1,024 bytes**:
- No known consumer uses Base62. No project consumes the fork (§1.2). (_Amended 2026-11-23 (AM-3/AM-4): was "assetvault-io, the only known consumer, has no Base62 use (searched 2026-10-25)"; AM-3 removed it as a consumer._)
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
- **When** each is compiled in the harness §3 defines: a `net11.0` project referencing both released packages, with each block placed as §3 specifies
- **Then** every block compiles
- **Assertion:** compiler errors == 0 for every block. At baseline, README line 152 fails this check.

  _Amended 2026-11-23 (AM-1/AM-2): compiled for `net10.0` and `net11.0`. Amended 2026-11-23 (AM-3/AM-4): a `net11.0` project only. Originally a `net10.0` project._
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
  - .NET 11 as the only target, with .NET 6, 8 and 9 dropped (_amended 2026-11-23, AM-1/AM-2: ".NET 10 and .NET 11 as the only targets"; amended 2026-11-23, AM-3/AM-4: .NET 11 only. Originally ".NET 10 as the only target". Still one topic, so the count stays 18_);
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

#### AC-112 — The sample runs on .NET 11

_Amended 2026-11-23 (AM-1/AM-2): was "on .NET 10", hosted on `net10.0`. Amended 2026-11-23 (AM-3/AM-4): no longer provisional, since AM-3 confirmed A-16 and resolved Q-26._

- **Verifies:** R-45
- **Given** the sample hosted in-process on `net11.0`
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

  _Amended 2026-11-23 (AM-1/AM-2): one report per framework run. Amended 2026-11-23 (AM-3/AM-4): one framework, so back to one report, as originally written._
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
    - Two further guards are §3 replacement tests, not ACs (Fred, 2026-10-26: "Yes, declare as guards"). They have no AC ID, so each is identified by its chain and by the removed test it replaces, as listed in §3's removed-test table:
      - **C11:** the null-only replacement for `AspNetCore.Identity.ExtendedPasswordValidatorTests.HasConsecutiveCharacters_WhenPasswordIsNullOrWhitespace_ExpectArgumentNullException`. `null` already throws at `555d31c`.
      - **C12:** the `null`-only replacement for `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxLength_ExpectNoMaxLengthAttribute`. `null` already adds no `maxlength` attribute at `555d31c`.
    - Like every guard, these must exist before the implementing commit of their chain (WI-28 for C11, WI-31 for C12), and they must pass at the release commit. They are not required to fail first.
  - **Added at Gate 2:**
    - AC-110: the two error codes already exist (`ExtendedPasswordValidator.cs:30,41`).
    - AC-112: the sample's pages already load.
  - **Removed at Gate 2: AC-29.** It fails at `555d31c`; this was found by reading the code, not by running it. `ConvertFromEdDsaSecurityKey` hands the JWK the key's factory, `CryptoProviderFactory.Default`, whose custom provider `ExtendedCryptoProvider` is set by the key's constructor (`EdDsaSecurityKey.cs:16`, `ExtendedJsonWebKeyConverter.cs:22`). That provider's `IsSupportedAlgorithm` checks only the algorithm name, so it claims `"EdDSA"` for the JWK (`ExtendedCryptoProvider.cs:9-10`). Its `Create` then accepts only an `EdDsaSecurityKey` and throws `NotSupportedException` for a `JsonWebKey` (`ExtendedCryptoProvider.cs:14-19`). Validation with the JWK therefore fails.
- **Assertion:**
  - For every non-guard AC in scope, the introducing commit comes before the implementing commit that §3.4 names.
  - At the introducing commit, the test compiles against the stubs and fails with an assertion failure, not a compile error.
  - At the release commit, every in-scope test passes.
  - **At the 1.0.0 release commit,** these three bullets are asserted for every in-scope AC that is not `1.x`-only. For the `1.x`-only ACs in scope (AC-52 to AC-54, AC-87, AC-88, AC-106 to AC-108, AC-112 and AC-114), the audit is still run and its result recorded. A failure, or an AC whose implementation has not landed yet, does not fail this AC at 1.0.0. It blocks from the release that carries the AC's requirement (convention "`1.x`-only checks never block 1.0.0").

  _Amended 2026-11-23 (Gate 2): the 1.0.0 bullet is added. Without it, `1.x` acceptance tests in scope blocked the 1.0.0 test-first audit, against §2.2.1._
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
- **Assertion:** _(amended 2026-11-23, AM-1/AM-2: 12 rows on two runtimes. Amended 2026-11-23 (AM-3/AM-4): 6 rows on .NET 11, as originally written for one runtime)_
  - The report has exactly 6 rows: a baseline row and an after row for each of P-e, P-f and P-g, with curve Ed448, all on the same 11.0.x runtime.
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

**Cut 2026-11-23 (AM-4, Fred); CUT-3 in §2.5.** This AC is not evaluated and counts in no total. Its text is kept so that earlier references resolve. Its requirement, R-54, is also cut (CUT-1).

- **Verifies:** R-54 (cut)
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

**Cut 2026-11-23 (AM-4, Fred); CUT-4 in §2.5.** This AC is not evaluated and counts in no total. Its text is kept so that earlier references resolve. Its requirement, R-54, is also cut (CUT-1).

- **Verifies:** R-54 (cut)
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

  **At the 1.0.0 release commit,** the count covers only work items that are not `1.x`-only. Each `1.x`-only work item is still audited and its result recorded if it has landed. Its failure, or its absence, does not fail this AC at 1.0.0. It blocks from the release that carries its requirements (convention "`1.x`-only checks never block 1.0.0"). R-50 still applies in full to every work item. Only the release it gates changes. _(Amended 2026-11-23 (Gate 2).)_

  A declared regression guard (AC-121's list, including the C11 and C12 replacement tests) cannot be the test that satisfies this AC, because it passes before the fix. It does not cause a failure here either. WI-28 (C11) and WI-31 (C12) are each satisfied by their chain's other, genuinely failing unit tests:
  - **WI-28 (C11):** the unit tests of AC-106 to AC-109.
  - **WI-31 (C12):** the unit tests WI-30 writes for the options rules. These are not AC-76 or AC-142, which are `integration` ACs and count only as C12's acceptance tests under AC-121. For each invalid configuration in AC-76's table, without a host or service provider:
    - one test constructs `ExtendedPasswordValidator<IdentityUser>` directly with those options and awaits `ValidateAsync` on `"Valid-Password-1"`;
    - one test constructs `NewPasswordTagHelper` directly with those options and calls `Process` on a `newpassword` element.

    Each asserts that an exception is thrown whose message (or `InnerException` chain) contains AC-76's listed names, ordinal. At WI-30's commit each test compiles and fails by assertion, because the setting is silently accepted. That makes 10 tests: 5 configurations × 2 entry points. Every one of them must meet this AC's three conditions, with WI-31 as the implementing commit.

  _Amended 2026-11-23 (Gate 2), readiness finding F-3. It read: "WI-28 (C11) and WI-31 (C12) are each satisfied by their chain's other, genuinely failing tests: AC-106 to AC-109 for C11, and AC-76 and AC-142 for C12." AC-76 and AC-142 are `integration`, which contradicted the unit-test rule for library work items above. The rule is unchanged, and the C12 unit tests are now named instead. §3's WI-30 already plans "unit tests for each options rule and message", so this names existing work and does not add scope._
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
- **Given** a caller-owned array `d` holding a valid private key, a copy `snapshot = d.ToArray()`, and `e = EdDsa.Create(new EdDsaParameters(curve) { D = d })` (both curves). (_Amended 2026-11-23 (AM-3/AM-4): the note "This mirrors assetvault-io, `EdDsaKeyMaterial.cs:51-53`" is removed, because assetvault-io is not a consumer. The AC is otherwise unchanged._)
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
- **Given** the sample hosted in-process on `net11.0` (_amended 2026-11-23, AM-1/AM-2: was `net10.0`; R-1, A-16_), every page it serves (at least the home page and the password-rules page, per AC-112), and the set V
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
- **Given** the sample hosted in-process on `net11.0` (_amended 2026-11-23, AM-1/AM-2: was `net10.0`; R-1, A-16_)
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
- **Given** the sample hosted in-process on `net11.0` (_amended 2026-11-23, AM-1/AM-2: was `net10.0`; R-1, A-16_), and the set V
- **When** the shared layout's `<script src>` assets are fetched from the home page
- **Then** Bootstrap's JavaScript bundle is loaded, served intact, and is the version AC-140 requires. That bundle is what keeps the mobile menu working.
- **Assertion:**
  - At least one loaded script returns 200 and has a SHA-256 equal to `dist/js/bootstrap.bundle.min.js` or `dist/js/bootstrap.bundle.js` from the npm `bootstrap` tarball at the version AC-140 accepts.
  - Whether the menu works in a real browser is not asserted, because that needs a browser-level test, which this spec does not use. A served, unmodified bundle is the observable evidence below the browser.
- **Executable as:** integration
- **Needs network:** yes. The test needs HTTPS access to `registry.npmjs.org`, to fetch the `bootstrap` metadata (including `dist-tags.latest`) and the release tarball it hashes against. If the registry cannot be reached, the test **fails**, reporting that the npm registry is unreachable and naming the URL that failed. It never passes silently and is never skipped. A cached tarball counts only under AC-138's integrity rule.

#### AC-144 — 1.0.0 is published only after its release gates pass

_Added 2026-11-23 (AM-2). It replaces AC-8's date bound: C-20 now says 1.0.0 ships when its release gates pass._

_Amended 2026-11-23 (Gate 2). The AC first compared only workflow-run timestamps. Three gates §3 names are not workflow runs, so they had nothing to compare: AC-20 and AC-119, which need administrator-scope API calls, and the test-first history audit (AC-121, AC-132), which runs locally. Each such gate now leaves a commit status on the release commit. GitHub sets a status's `created_at` itself, so the record is timestamped by the server and cannot be backdated by whoever posts it. The other option, narrowing this AC to the workflow runs and checking the other three after publication, was rejected. An after-publication check can show that those gates pass, but not that they passed before release, which is what C-20 requires._

- **Verifies:** R-3 (C-20 as amended)
- **Given** the release commit; the published timestamps of both 1.0.0 packages (AC-8), the earlier of which is **P**; and every release gate §3 names, in two kinds:
  - **Workflow-run gates:** the CI workflow run on the release commit (AC-7), and every other gate run §3 names for it.
  - **Non-workflow gates:** every other check §3 names as a release gate. At minimum these are AC-20 (private vulnerability reporting enabled), AC-119 (advisory alerts delivered) and the test-first history audit (AC-121 and AC-132). Each is recorded as a commit status on the release commit, under a context string §3 fixes for that gate, posted only once the check has passed. The status's `description` names the evidence log of the check.
- **When**
  - each workflow-run gate's conclusion and completion time (`updatedAt`) are read, for example `gh run list --commit <sha> --workflow <file> --json conclusion,updatedAt`; and
  - each non-workflow gate's statuses are read from `GET /repos/IronSloth/IdentityModel-Supa/commits/<sha>/statuses`, which returns each status's `context`, `state` and server-set `created_at`.
- **Then** every release gate, of both kinds, had passed before either package was published
- **Assertion:**
  - Every workflow-run gate has `conclusion == "success"` and `updatedAt` < P.
  - For every non-workflow gate's context, the newest status with `created_at` < P has `state == "success"`.
  - The count of gates that are missing, failed, or recorded only after P == 0. A non-workflow gate with no status on the release commit fails this AC.
- **Executable as:** integration

#### AC-145 — The .NET 11 consumer gets the .NET 11 build from the feed

_Added 2026-11-23 (AM-1). Amended 2026-11-23 (AM-3/AM-4): the `net10.0` consumer is removed. The AC still matters: AC-1 checks the locally packed files, and AC-9 would also pass against a package that carried only an older framework's build, so this is what shows the published packages carry the `net11.0` build._

- **Verifies:** R-3, R-1
- **Given** the `net11.0` consumer project of AC-9, after its restore from the IronSloth feed
- **When** its restore output (`obj/project.assets.json`) is read for both ValhallaTech packages
- **Then** it uses the packages' `net11.0` build
- **Assertion:** for each package, every `compile` and `runtime` asset path is under `lib/net11.0/`.
- **Executable as:** integration

### 2.4 Success metrics

_How we know it worked after it ships. Metric, current baseline, target, measurement source._

| ID | Metric | Baseline | Target | Measured by |
| --- | --- | --- | --- | --- |
| M-1 | **Cut 2026-11-23 (AM-4, Fred); CUT-2 in §2.5.** Not measured. It exists only to track migration from upstream 4.1.0, which AM-4 and NG-9 put out of scope. Kept so that references resolve: ValhallaTech projects on upstream `ScottBrady.IdentityModel` that have moved to `ValhallaTech.Security.IdentityModel` 1.x | 0 of 1 (assetvault-io, on 4.1.0; Q-4 resolution) | assetvault-io on 1.x within 14 days after 1.0.0 is published, and 0 ValhallaTech projects on `ScottBrady.*` within 51 days after it. (_Amended 2026-11-23, AM-2: was "by 2026-11-24" and "by 2026-12-31", dates counted from the removed 2026-11-10 release date. The intervals are unchanged._) | A search of ValhallaTech repositories for `ScottBrady.IdentityModel` and `ValhallaTech.Security.IdentityModel` package references, plus download statistics from the IronSloth GitHub Packages feed (GP-10) |
| M-2 | Open High or Critical dependency advisories on `master` | 2, both test-only and transitive through AutoFixture: `System.Net.Http` 4.3.0 and `System.Text.RegularExpressions` 4.3.0 (review, `555d31c`) | 0 at every release, and 0 open for more than 7 days in between | The CI advisory audit (AC-45) and the repository's advisory alerts (AC-119) |
| M-3 | Line coverage for each packable library | unknown (not collected today) | ≥ 80% on every `master` build | The CI coverage report (AC-116) |
| M-4 | Median days from a High or Critical advisory on a dependency to a merged fix on `master` | unknown, needs instrumentation | ≤ 7 days | Advisory publication dates compared with the merge dates of the update tool's PRs (D-7) |
| M-5 | 1.x releases that contain an unintended breaking API change | n/a, since no 1.x release exists yet | 0 | The API compatibility check (AC-115) on every release, cross-checked with `CHANGELOG.md` |
| M-6 | Hot-path performance in later 1.x releases | the 1.x figures recorded under R-22 | no path in T regresses against the most recent recorded release | Re-running measurement set T before each minor release |
| M-7 | Days on which a shipped target framework or the CI SDK is out of Microsoft support | > 0 (the AspNetCore library has targeted `net6.0`, out of support since November 2024, and both libraries have targeted only `net8.0` and `net9.0`, out of support since 2026-11-10; amended 2026-11-23) | 0 from 1.0.0 onward, for `net11.0`. .NET 11 is an STS release supported until 2028-11-14 (Q-27, resolved), so keeping 0 after that date needs a move to a supported .NET, which is later work. (_Amended 2026-11-23 (AM-3/AM-4): was "for both `net10.0` and `net11.0`". Amended 2026-11-23 (Gate 2): "how long .NET 11 is supported is Q-27" replaced by the resolved date._) | Target frameworks and CI SDK versions checked against Microsoft's .NET support policy |
| M-8 | HTTP 500 errors in consuming apps caused by an exception from the extended password validator | unknown, needs instrumentation in the consuming apps | 0 | The consuming apps' error logs, filtered to exceptions thrown from `ExtendedPasswordValidator` |
| M-9 | Vulnerability reports for this fork that arrive through the canonical repository's private reporting | unknown (no channel exists: private reporting returned `{"enabled":false}` on 2026-10-25, C-21, and `SECURITY.md` points to the upstream author) | every report received for the fork comes through private reporting | The GitHub security advisories list on the canonical repository (C-22) |
| M-10 | Days a direct dependency on `master` stays behind its latest stable version | > 0 at `555d31c`. For example, Microsoft.IdentityModel 7.7.1 against a current 8.x, and BouncyCastle.Cryptography 2.4.0 against 2.7.0. | 0 at every release. Between releases, no direct dependency more than 14 days behind its latest stable. | The update tool's PR history and dashboard (D-7, AC-134), plus the AC-49 check run before each release. (_Amended 2026-11-23 (Gate 1, Q-30): "the version R-17 requires", per framework. Amended 2026-11-23 (AM-3/AM-4): one framework, so back to "latest stable"; the AC-49 check stays the source. Originally measured with `dotnet list package --outdated`._) |

### 2.5 Out of scope

_Carried forward from §1.4, plus anything cut during PRD authoring._

**Carried forward from §1.4**
- NG-1: publishing 1.0.0 to nuget.org. Whether the public release belongs to this spec is Q-2. (_Amended 2026-11-23, AM-2: the date is dropped, as in §1.4._)
- NG-2: supporting .NET 6, 8, 9 or 10 consumers. .NET 11 is the only supported .NET (C-2 as amended by AM-3). (_Amended 2026-11-23 (AM-3/AM-4): was "The supported set is .NET 10 and .NET 11."_)
- NG-11: _withdrawn 2026-11-23 (AM-3/AM-4), as in §1.4._ A consumer of 1.0.0 must be on .NET 11 (C-2). It read: "requiring any consumer to move to .NET 11. assetvault-io stays on .NET 10 (C-26)."
- NG-12: letting repositories outside this one shape this work, through their target framework, dependency versions or migration needs (AM-3). (_Added 2026-11-23 (AM-3/AM-4)._)
- NG-3: staying mergeable with upstream, or contributing back to it.
- NG-4: restoring Branca or PASETO.
- NG-5: new algorithms or token formats.
- NG-6: bringing ValhallaTech's application stack into the libraries.
- NG-7: Heimdall-scale process or infrastructure.
- NG-8: private-key copies held inside BouncyCastle. AC-30 and AC-31 skip BouncyCastle-owned objects for this reason.
- NG-9: migrating any consumer of upstream `ScottBrady.*`, ValhallaTech's own projects included, or giving them any migration aid beyond the changelog's breaking-change list (R-42, AC-104). (_Amended 2026-11-23 (AM-3/AM-4): was "public nuget.org consumers … ValhallaTech's own projects are covered by G-29 and R-54." G-29 is superseded and R-54 is cut._)

**Cut or bounded during PRD authoring**

IDs are kept stable. Withdrawn items stay listed so that earlier references still resolve.

- **X-1 — withdrawn (GP-2).** A maximum Base62 input length is now required (R-25).
- **X-2 — `ObjectDisposedException` on use after `Dispose`.** No goal asks for it, and R-11 asserts only zeroing and that the caller's array is left alone.
- **X-3 — withdrawn (GP-4).** The constants are removed in 1.0.0 (R-36, AC-95).
- **X-4 — Shipped translations of the password messages.** R-44 requires that messages can be replaced. It does not require the library to ship any translations.
- **X-5 — Ed448 performance measurement.** This is `MAY` (R-52). G-13's hot paths are measured for Ed25519.
- **X-6 — Renaming repository directories, project files, the solution file or the test project.** Consumers do not see these inside the packages, so the PRD does not require it. The Architect may do it under D-9.
- **X-7 — narrowed (Q-4 resolution), then restored to full by AM-4.** A migration guide for public upstream consumers stays out (NG-9). The one for ValhallaTech's own 4.1.0 consumers (R-54, AC-130, AC-131) was required until Fred cut it (CUT-1, CUT-3, CUT-4), so no migration guide of any kind is in scope. Actually migrating assetvault-io is that project's own work. (_Amended 2026-11-23 (AM-3/AM-4)._)

**Cuts decided by Fred (AM-4, 2026-11-23)**

Fred chose "Cut all of it" for the migration guide and every assetvault-io requirement (§1.8 AM-4). The CHANGELOG's breaking-change list (R-42, AC-104) is the only migration aid. Each cut item keeps its ID and text, marked cut, so that earlier references resolve.

| CUT | What it removes | From (§1 location) |
| --- | --- | --- |
| CUT-1 | R-54: a written migration guide from upstream 4.1.0 to 1.0.0, proven sufficient for assetvault-io. | §1.3 G-29 (superseded by AM-4); §1.8 AM-4; §1.4 NG-9 as amended |
| CUT-2 | M-1: the metric counting ValhallaTech projects moved from upstream `ScottBrady.IdentityModel` to 1.x, with assetvault-io as its baseline. It exists only to track migration from upstream 4.1.0. | §1.8 AM-4 ("every assetvault-io requirement"); §1.4 NG-9 and NG-12 |
| CUT-3 | AC-130: the migration guide covers AC-104's 18 topics and the six upstream APIs assetvault-io uses. | §1.8 AM-4 (names AC-130); §1.3 G-29 |
| CUT-4 | AC-131: a scratch copy of assetvault-io builds and passes its token tests using the guide alone. | §1.8 AM-4 (names AC-131); §1.3 G-29; §1.2 (assetvault-io no longer a stakeholder) |
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
| Q-9 | What is the largest Base62 or Base16 input consumers need? No project consumes the fork, so the 1,024-byte Base62 cap has no known conflict. (_Amended 2026-11-23 (AM-3/AM-4): was "The only known consumer (assetvault-io) uses neither"._) | no | Fred | R-22, R-24, R-25, measurement set T, AC-69, AC-70 |
| Q-10 | Are the two unreported `EdDsa.Verify` defects in G-18's scope? The PRD assumes yes (A-11). | no | Fred | R-31, AC-84, AC-85 |


**Resolved**

| ID | Resolution | Source |
| --- | --- | --- |
| Q-30 | _Raised by the PM, 2026-11-23; resolved the same day (Gate 1, Q-30)._ Confirmed: `Microsoft.AspNetCore.Authentication.JwtBearer` 10.0.14 and 10.0.15 declare only a `net10.0` dependency group, and 11.0.0 declares only `net11.0`. So a `net10.0` build cannot use the latest stable 11.0.0, and AC-49 as first written could never pass. Resolution: a framework-bound dependency is held, for each framework a project builds for, to the newest stable version that supports that framework. Every other dependency stays at its single latest stable version. Amended: R-17, R-48, AC-49, FC-3 and M-10. AC-133 and AC-134 are unchanged: AC-133 covers npm client-side libraries, which have no .NET target framework, and AC-134 takes its inventory from AC-49. _Amended 2026-11-23 (AM-3/AM-4): moot. Every project now builds for `net11.0` only, so the per-framework rule is removed from R-17, R-48, AC-49, FC-3 and M-10. AC-49 keeps the registration-API method for finding the single latest stable version._ | Fred, Gate 1 ("Request changes"), relayed by the /spec coordinator; NuGet registration API, checked by the /spec coordinator, 2026-11-23 |
| Q-25 | "Helper in 1.0.0." R-28 (the single-step setup helper) moves into 1.0.0. Invalid options are rejected at startup through that step (AC-76) and at first use through upstream's two-step path (AC-142), which Fred accepts. R-51 stays a `1.x` SHOULD. | Fred, after Gate 1, relayed by the /spec coordinator |
| Q-13 | Ed448 `KeySize` is 456 bits (57-byte encoding × 8). R-34 and AC-91 already used this value. | Fred, Gate 1 |
| Q-17 | Reject 0 and negative values for `MaxLength` and `MaxConsecutiveChars`. `null` means no limit. Rejection applies however the validator is registered, which makes it breaking, so R-29 moves to 1.0.0. Covered by AC-76, AC-77 and the new AC-142. The two-step path's timing is Q-25. | Fred, Gate 1 |
| Q-18 | The icon is the ValhallaTech company logo, a PNG of at least 128×128 that Fred supplies before release. "No icon" is no longer accepted (R-5, AC-12, FC-6). | Fred, Gate 1 |
| Q-21 | Confirmed. The specific goals override G-18's general rule: `FormatException` for bad encoded text, `CryptographicException` for invalid keys, `ArgumentException` for bad arguments. | Fred, Gate 1 |
| Q-23 | Confirmed: a cap of 1,024 bytes for `Encode` and 1,376 characters for `Decode`, and a budget of 3 ms mean per call at the cap. Measured on Fred's machine at 1,024 bytes: SimpleBase takes 1.577 ms to encode and 0.809 ms to decode, against 3.555 ms for today's encode, with 1,375-character output (FC-5 holds). | Fred, Gate 1; scratch probe of 2,000 calls, 2026-10-25 |
| Q-24 | Yes. Base62 translates SimpleBase's invalid-character `ArgumentException` into `FormatException`, and AC-36 now requires `FormatException`. Over-cap input stays `ArgumentException`. X-10 is withdrawn. | Fred, Gate 1 |
| Q-1 | The full 1.0.0 scope ships when its release gates pass, with every behavior-changing fix in it, and not as a prerelease. Non-breaking work may follow in 1.x. §2.2.1 is firm. _Amended 2026-11-23 (AM-2): was "The full 1.0.0 scope ships by 2026-11-10."_ | GP-1, C-20 as amended by AM-2 |
| Q-29 | The 1.0.0 / 1.x split stands without the date or a waiting consumer, and no requirement moves (A-19 holds). Tests (a) to (c) depend on neither. The reason to keep 1.0.0 lean is delivery: G-1 is met only once 1.0.0 ships, and every `1.x` item either leaves package behavior unchanged or is non-breaking, so holding 1.0.0 for it delays G-1 for no consumer benefit. Every `1.x` item is still required before the spec is done. See §2.2.1. Fred may overrule. _Amended 2026-11-23 (AM-3/AM-4): re-decided. The first answer's reason, S1 exposure in assetvault-io, no longer applies, because assetvault-io uses the upstream package (AM-3)._ | PM decision, 2026-11-23, re-decided the same day (§1.6 names Fred / PM as owners) |
| Q-27 | Until when is .NET 11 supported? It is an STS release, and its support ends 2028-11-14, the same day as .NET 10's LTS, so A-17 is confirmed. M-7 now cites the date. No requirement or AC changes. _(Moved from open, amended 2026-11-23 (Gate 2), to match §1.6.)_ | Architect's fact check, read 2026-11-23 in Microsoft's `releases-index.json` and on the .NET support policy page, as recorded in §1.6 |
| Q-26 | Which .NET does the sample run on? `net11.0`, like every project (C-2). A-16 is confirmed. R-1, R-45, AC-2 and AC-112 no longer depend on it. _(Moved from open, 2026-11-23, AM-3/AM-4.)_ | Fred, AM-3, as recorded in §1.6 |
| Q-28 | What does the C# 15 standard require of this repo? .NET 10 is dropped (C-2), and C# 15 applies to all of the repo's code (C-25). A-18 was false. FC-7 is closed. §2 sets no language-version requirement; C-25 binds §3 as a constraint. _(Moved from open, 2026-11-23, AM-3/AM-4.)_ | Fred, AM-3, as recorded in §1.6 |
| Q-3 | Both IDs are free on nuget.org. | §1.6 |
| Q-4 | assetvault-io uses upstream 4.1.0. This added G-29 / R-54. _Amended 2026-11-23 (AM-3/AM-4): the fact stands, but assetvault-io uses upstream's package, not the fork, so it is not a consumer of this work. G-29 is superseded and R-54 is cut (CUT-1)._ | §1.6; Fred, AM-3 and AM-4 |
| Q-7 | Test-first applies at unit level too. R-50 and AC-132 cover it. | GP-3, C-13 |
| Q-8 | `IronSloth/IdentityModel-Supa` is canonical (C-22). Private reporting is disabled, and only Fred can enable it (C-21); AC-20 depends on that. | GP-7 |
| Q-11 | Yes. assetvault-io is already on Microsoft.IdentityModel 8.19.1 and BouncyCastle 2.6.2. _Amended 2026-11-23 (AM-3/AM-4): the question no longer shapes the work. No consumer's versions cap R-17, because repositories outside this one do not shape it (NG-12, A-5 superseded)._ | §1.6; Fred, AM-3 |
| Q-12 | A cap, a time budget, and faster than today (R-25). SimpleBase is decided for Base62 (D-3 resolved). | GP-2, coordinator correction 2026-10-25 |
| Q-14 | Removed in 1.0.0 (R-36, AC-95). | GP-4 |
| Q-15 | Mismatched pairs are rejected (R-15, AC-128). | GP-5 |
| Q-16 | The library zeroes only its own copies and never modifies a caller-supplied array (R-11, AC-135). _Amended 2026-11-23 (AM-3/AM-4): the illustration "assetvault-io passes its own array as `D` (`EdDsaKeyMaterial.cs:53`)" is dropped from R-11 and AC-135, since assetvault-io is not a consumer. The resolution stands on its own: a library must not alter memory a caller owns._ | PM decision on the Analyst's input, as directed by the coordinator |
| Q-19 | R-18's any-severity rule stands. Every jQuery 3.3.1 advisory is rated medium, so a High-or-Critical threshold would miss them all. | `gh api /advisories?affects=jquery@3.3.1`, run by the /spec coordinator, 2026-10-25 |
| Q-20 | A prerelease does not count as shipped. | GP-1, C-20 |
| Q-22 | `NotSupportedException` in 1.0.0 (R-53, AC-129). | GP-6 |
| GP-10 (Gate 2) | D-2 decided: GitHub Packages under IronSloth (R-3, AC-8, AC-9, FC-1). The sample ships Bootstrap 5's JavaScript (AC-143), and the no-jQuery checks exclude unmodified third-party release files (set V; AC-137, AC-138). SimpleBase follows latest stable, no longer pinned to 5.6.4 (R-12, R-13). | Fred, Gate 2, relayed by the /spec coordinator; §1.8 GP-10, C-23 |
| GP-8 (no Q) | The sample is kept as a Razor app on the latest stable Bootstrap 5.x, with no jQuery anywhere in the repository (R-56, AC-137 to AC-141). | Fred, relayed by the /spec coordinator, 2026-10-25. The Analyst records it in §1 afterwards. |

---

## 3. Technical Plan

<!-- OWNER: Architect. No other agent may edit this section. -->

**Status:** ✅ complete

**Amended 2026-11-23 (AM-1/AM-2/Q-30).** §1 and §2 were amended on 2026-11-23 and approved at Gate 1 the same day: both libraries and their tests target `net10.0` and `net11.0`, the sample targets `net11.0`, the org standard is .NET 11 with C# 15, and there is no release date. This revision adds §3.0, maps every earlier design decision into it, and amends the items below in place. Each amended passage is marked "amended 2026-11-23 (AM-1/AM-2/Q-30)". No WI-, RK-, U- or chain ID is renumbered or removed. New items are WI-53 and WI-54, and new risks are RK-17 to RK-19. No work item has started: the code still targets `net8.0`/`net9.0` *(per the /spec coordinator, 2026-11-23)*.

**Amended again 2026-11-23 (AM-3/AM-4).** Fred answered G2-1 with "Drop .NET 10." (AM-3) and cut the migration guide and every assetvault-io requirement (AM-4: R-54, M-1, AC-130, AC-131; CUT-1 to CUT-4). §1 and §2 were amended and approved at Gate 1 again. So every project, CI and the release now target `net11.0` and the .NET 11 SDK and runtime only, C# 15 applies to all code (C-25, C-27), and R-17, R-48 and AC-49 hold each dependency to one latest stable version. In this revision AD-43 to AD-54 supersede the rows AM-3 and AM-4 made wrong, which stay in the table marked superseded. WI-41, WI-45 and WI-47 are marked cut, with their IDs kept. RK-8, RK-17 and RK-18 are withdrawn and RK-19 is resolved; RK-20 is new. Passages changed in this pass are marked "amended 2026-11-23 (AM-3/AM-4)". Where a passage marked "(AM-1/AM-2/Q-30)" was rewritten again, the newer mark governs.

**Amended 2026-11-23 (Gate 2).** The readiness review failed, and Fred chose "Request changes". This pass repairs F-1 (a dependency cycle between WI-1 and WI-53, caused by a WI- token in WI-53's depends-on prose), removes the WI- tokens from the cut items' depends-on lines, carries Q-27's resolution, and follows the amended AC-144: AD-55 supersedes AD-53 and adds three commit-status gates, enforced before publishing. Changes are marked "amended 2026-11-23 (Gate 2)".

**Amended 2026-11-23 (Gate 2, second review).** The second readiness review returned CONCERNS. This pass repairs:
- **F-1:** checks of `1.x` requirements no longer gate 1.0.0. AD-56 makes them record-only in `release-gate.yml` (WI-54, WI-43) and moves the `1.x` CI checks to a separate `quality.yml` workflow (WI-7, WI-50, WI-51).
- **F-2:** the sample TRX clause moves from WI-3 to WI-34.
- **F-4:** AD-57 fixes the CHANGELOG date. WI-40 writes its own commit date, and the new WI-55 sets the final date on the release commit.
- **F-3:** carries the amended AC-132. WI-30 writes 10 unit tests that fail by assertion, and WI-31 makes them pass.

**Amended 2026-11-23 (Gate 2, §2.3 convention).** The PM added the §2.3 convention "`1.x`-only checks never block 1.0.0" and amended AC-4, AC-7, AC-121 and AC-132 to match. This pass carries them:
- AD-58 supersedes AD-56 and covers every `1.x`-only AC the convention lists. It fixes the marker trait, `[Trait("Release", "1.x")]`, and the requirement trait, `[Trait("Requirements", "<ids>")]`.
- The test step is split in `dotnetcore.yml` and `release.yml` (WI-3, WI-44), so marked failures are recorded but fail no job.
- The new WI-56 adds the misuse check, which `release-gate.yml` also runs (WI-54).
- WI-16, WI-27, WI-34 and WI-36 mark their `1.x`-only tests.
- U-6 applies the 1.0.0 scope, and WI-46 records the `1.x` results.
- WI-35 no longer gates WI-55 or WI-46, and the AC-103 check moves to `quality.yml` (WI-50).

**Amended 2026-11-23 (Gate 2, third review).** The third readiness review failed, and Fred chose "Request changes". This pass repairs F-1 to F-5: every out-of-process seam now has a test-first chain, with Pester acceptance tests (AD-59, AD-60):
- C14: the measurement reports (AC-60 to AC-70, AC-127);
- C15: the CI run (AC-7);
- C16: the release-gate runs and statuses (AC-144);
- C17: the feed and a scratch consumer (AC-8, AC-9, AC-145);
- C18: the repository's security settings (AC-20, AC-118, AC-119; AD-62);
- C19: the commit history, through an executable audit tool (AC-121, AC-132; AD-61).

The new items are WI-57 to WI-64. §3.7's exemption sentence is withdrawn. Checks that can pass only after publication or after a Fred-only action are named as release checks or Fred-state checks. The pass also carries F-6 (R-2 and R-50 as amended). The spec now completes at a 1.1.0 release, run with no exemption (AD-64; WI-65 to WI-68).

**Amended 2026-11-23 (Gate 2, fourth review).** The fourth readiness review returned CONCERNS, and Fred chose "Request changes".
- **F-1.** §3 said WI-7 did not gate 1.0.0, while WI-8 and WI-39 depend on it. AD-65 supersedes AD-58 and replaces only its point (8). WI-7, the one-time reformat and `quality.yml`, gates 1.0.0 as sequencing, and only AC-120's result is record-only. WI-7's done-when says what lets WI-8 and WI-39 start when AC-120 fails, and gives a fallback if CSharpier cannot run. Every `decisions:` line that cited AD-58 now cites AD-65.
- **F-2.** The PM narrowed the §2.3 measurement convention to AC-60 to AC-70 and AC-127. §3 already matched: AC-59 and AC-123 are `lint` with no chain, and C14 covers exactly the narrowed list. One WI-54 sentence that grouped them all is reworded.

### 3.0 Decisions

_One row per design decision. IDs are never renumbered or reused. A superseded row stays, marked at the start of its Decision cell. AD-1 to AD-31 record the decisions of the plan approved at Gate 2 on 2026-10-25 (with GP-10 and GP-11), mapped into this table on 2026-11-23 without changing their substance. AD-32 to AD-42 are new on 2026-11-23 for AM-1, AM-2 and Q-30. AD-43 to AD-54 are new on 2026-11-23 for AM-3 and AM-4. AD-55 is new on 2026-11-23 for the Gate 2 repair. AD-56 and AD-57 are new on 2026-11-23 for the second readiness review, and AD-58 (superseding AD-56) for the §2.3 convention "`1.x`-only checks never block 1.0.0". AD-59 to AD-64 are new on 2026-11-23 for the third readiness review (F-1 to F-6). AD-65, superseding AD-58, is new on 2026-11-23 for the fourth readiness review (F-1)._

| ID | Decision | Resolves | Alternatives rejected |
| --- | --- | --- | --- |
| AD-1 | Replace FluentAssertions 6.12.2 with AwesomeAssertions at its latest stable version (9.6.0 on 2026-10-25), Apache-2.0. Fred, 2026-10-25: "We're using AwesomeAssertions". | D-1 | Shouldly (every assertion rewritten); FluentAssertions 7.2.2 (not FluentAssertions' latest stable, so AC-49 fails). |
| AD-2 | The private feed is GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json`. Decided by Fred (GP-10). | D-2 | — (Fred's decision). |
| AD-3 | Base62 is `SimpleBase.Base62.Default` at its latest stable version, capped at 1,024 bytes (`Encode`) and 1,376 characters (`Decode`), as fixed by Fred (GP-2, GP-10). `Decode` checks in this order: null, length cap, one ordinal alphabet pass (`FormatException`), SimpleBase decode (any escaping `ArgumentException` becomes `FormatException`), result over 1,024 bytes (`ArgumentException`). | D-3 | An exact pre-decode overflow check against cached encodings of 256^m (same time bound, more code; R-25(b) no longer needs it). |
| AD-4 | The private-key export opt-in is a separate method, `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKeyIncludingPrivateKey(EdDsaSecurityKey)`. The default converter is public-only. | D-4 | A `bool includePrivateKey` parameter (hides intent at the call site); an export-mode enum (more surface for a two-way choice). |
| AD-5 | 64-byte Ed25519 and 114-byte Ed448 `D` values are rejected at validation, with a message telling the caller to split seed and public key into `D` and `X`. `Validate()` builds the key through the same internal path `Create` uses and checks a `D`/`X` pair in constant time. | D-5 | Supporting the concatenated form (a parse path, a second copy to zero, and a pair check for a format no known consumer uses). |
| AD-6 | BenchmarkDotNet (latest stable, MIT) measures set T, with verbatim `555d31c` source copies of each measured path compiled into the same benchmark process as the release candidate. | D-6 | Referencing upstream 4.1.0 from nuget.org (pulls Microsoft.IdentityModel 7.x beside 8.x, and is not provably `555d31c`); a `git worktree` build (two processes, two dependency graphs). |
| AD-7 | CSharpier (latest stable, as a dotnet local tool) formats the code, checked by `dotnet csharpier check .` in CI. One reformat commit lands in Phase 0. | D-7 (formatting) | `dotnet format whitespace` (style-configurable through `.editorconfig`, which invites the inventory sprawl C-17 rules out). |
| AD-8 | Renovate proposes dependency updates (Fred's lean, as in assetvault-io), with `vulnerabilityAlerts` on. GitHub Dependabot alerts (U-3) deliver advisories. | D-7 (updates) | Dependabot version updates (no custom regex manager, so `libman.json` stays invisible and AC-134 fails). |
| AD-9 | The sample's Bootstrap 5 CSS and JS bundle are restored unmodified through LibMan (`jsdelivr` provider) from `libman.json`, which a Renovate `customManagers` regex entry reads. UI framework fixed by Fred (GP-8, GP-10). | D-8 (UI) | An npm `package.json` plus a copy step (adds Node and a lockfile to a .NET build); a CDN tag with SRI (Renovate cannot recompute the hash); committed files with no manifest (invisible to update tooling). |
| AD-10 | The sample uses DryIoc through `DryIoc.Microsoft.DependencyInjection`, and one library test resolves the single step through DryIoc. Decided by Fred (2026-10-25: "Yes, sample + one test"). The libraries take no DryIoc dependency. | D-8 (DI) | — (Fred's decision). |
| AD-11 | Superseded by AD-32. Build layout: `.slnx`; `global.json` pinned to the 10.0 SDK feature band; root `Directory.Build.props` with `TargetFramework=net10.0`, `LangVersion=latest`, warnings as errors and NuGetAudit; `src/Directory.Build.props` for library settings; central package management; renamed directories. | D-9 | Keeping `.sln`; per-project versions. |
| AD-12 | CI runs on `ubuntu-latest` only, following the label; CodeQL stays. Windows is covered by local runs on ODIN01. Decided by Fred (GP-11). | D-10 | Three-OS matrix (macOS slowest; BouncyCastle is pure C#). |
| AD-13 | Messages are replaced through a new `ExtendedIdentityErrorDescriber : IdentityErrorDescriber`, registered with `AddErrorDescriber<T>()`, and a new `ExtendedPasswordValidator<TUser>(IdentityErrorDescriber? errors = null)` constructor. | D-11 | A resource file and `IStringLocalizer` inside the library (ships translations, which X-4 cuts); message templates on the options (not how ASP.NET Identity does it). |
| AD-14 | Each `EdDsaSecurityKey` gets its own `CryptoProviderFactory` whose `CustomCryptoProvider` is one shared, stateless, internal EdDSA provider, with `CacheSignatureProviders = false`. Nothing writes `CryptoProviderFactory.Default`. | — | One library-owned factory shared by all EdDSA keys (cheaper, but changing one key's factory changes every key, against G-4). |
| AD-15 | The single step is `AddExtendedPasswordRules(this IdentityBuilder, Action<ExtendedPasswordOptions>)` in `ValhallaTech.Security.IdentityModel.AspNetCore.Identity`, with `ValidateOnStart`. One internal rule evaluator feeds the options validator, the validator's first-use check and the tag helper. | — | The conventional `Microsoft.Extensions.DependencyInjection` namespace (AC-13 forbids exported types outside the ValhallaTech prefix); separate rule copies per consumer. |
| AD-16 | Test-first chain structure: where the seam already exists at `555d31c`, WI-5's rename is the chain's stubs step; a chain-specific stubs item exists only for new surface (WI-13, WI-26, WI-29); C1's tests precede the rename. | — | One empty stubs item per chain (more hand-offs, no added signal). |
| AD-17 | Base16 is re-implemented on `Convert.ToHexStringLower` and `Convert.FromHexString`. | — | Keeping the lookup table with an added length check (still allocates per byte). |
| AD-18 | `StreamExtensions.TryRead` reads with `Stream.ReadAtLeast(buffer, length, throwOnEndOfStream: false)`. | — | A hand-written read loop (same behavior, more code). |
| AD-19 | `EdDsa.Create` copies `D` and `X` into library-owned arrays, never touches the caller's arrays, and derives the public key when only `D` is given. `Dispose(bool)` zeroes the library-owned private-key buffer. | — | Holding the caller's arrays (would force zeroing them, against Q-16). |
| AD-20 | AutoFixture is removed, not upgraded (4.18.1 is already its latest stable and carries both S8 advisories). The `xunit` 2.9.x package ID stays; xunit.v3 is not a mechanical swap (changed lifetimes, an executable test host) and no AC needs it (C-17). _Reason amended 2026-11-23 (AM-2): it was "not a mechanical swap under the deadline"._ | — | Upgrading AutoFixture (no fixed version); xunit.v3 now. |
| AD-21 | The sample drops `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` and its `AddRazorRuntimeCompilation()` call; Hot Reload covers it. | — | Keeping it (adds the Roslyn payload and one more framework-bound package). |
| AD-22 | Library analysis lives in `src/Directory.Build.props`: `Nullable=enable`, `GenerateDocumentationFile=true`, `AnalysisLevel=latest-recommended`, `AnalysisModeSecurity=All`, no `NoWarn`. Tests and the sample keep default analysis. No `.editorconfig` severity inventory. | — | A per-rule `.editorconfig` inventory (C-17). |
| AD-23 | Superseded by AD-39. Set T is measured on Fred's workstation on one .NET 10.0.x runtime, baseline and after in one BenchmarkDotNet process. | — | A CI runner (noisy, and not the machine behind the GP-2 figures). |
| AD-24 | Superseded by AD-41. Coverage is gated by `build/Test-CoverageThreshold.ps1`, reading one Cobertura figure per library, with a fixed 80% floor and a committed 79.9% fixture that must fail. | — | coverlet's MSBuild threshold (cannot be fed a report, as AC-117 requires); a ratchet (C-17). |
| AD-25 | After 1.0.0, package validation runs against the 1.0.0 packages committed under `eng/baseline/` (`PackageValidationBaselinePath`). | — | `PackageValidationBaselineVersion` (every restore would need private-feed credentials). |
| AD-26 | `release.yml` publishes on a pushed `v*.*.*` tag with the workflow token (`contents: read`, `packages: write`). The FC-1 trial publishes a throwaway `ValhallaTech.FeedTrial` 0.0.1 early in Phase 1, then deletes it. | — | Publishing with a personal token (a long-lived secret); a trial under a real package ID (AC-105 would then require a changelog entry). |
| AD-27 | Superseded by AD-51. assetvault-io's migration is proven on a scratch copy of `main` at `863c0cb0cde62f820b76b9c76e1b4cf245f7dc55`, first against the release candidate from a local folder feed (before publish), then against the published package. `C:\Repo\assetvault-io` is never edited. | — | Proving it only after publish (a library-side defect would then need 1.0.1). |
| AD-28 | Branches merge without squashing, and test commits carry `test(WI-n)` and their AC IDs, so AC-121 and AC-132 can read the commit order. | — | Squash merges (destroy the order the audit reads). |
| AD-29 | The sample's in-process tests live in their own project, `test/ValhallaTech.Security.IdentityModel.Samples.Tests`, so library coverage excludes the sample (A-6). | — | Sample tests inside the library test project (pollutes the per-library figure). |
| AD-30 | Directories, project files, the solution and the test project are renamed to the ValhallaTech names (X-6), and the three repo-wide changes (rename, reformat, annotations) run serially in Phase 0. | — | Keeping two names; running repo-wide changes beside the lanes (merge conflicts, RK-9). |
| AD-31 | Superseded by AD-54. The fixed strings and paths in 3.3.5: package metadata, the two-line modification notice and its placement, the report paths, the migration-guide and CHANGELOG phrase lists, and the doc-snippet placement rules. | — | — (the ACs defer these values to §3). |
| AD-32 | Superseded by AD-44. _New 2026-11-23 (AM-1)._ Supersedes AD-11. Build layout: `ValhallaTech.Security.IdentityModel.slnx`; `global.json` pins SDK `11.0.100` with `rollForward: latestFeature`, so only an 11.0 SDK is selected; root `Directory.Build.props` holds warnings as errors and NuGetAudit only, with no target framework and no `LangVersion`; `src/Directory.Build.props` sets `TargetFrameworks=net10.0;net11.0` for both libraries; the library test project and the benchmark project set `TargetFrameworks=net10.0;net11.0`; the sample and its test project set `TargetFramework=net11.0` (AD-34); central package management; renamed directories. | D-9 | Keeping the 10.0 SDK pin (cannot build `net11.0`, so AC-3 fails); target frameworks in the root props (the sample would have to clear an inherited `TargetFrameworks`). |
| AD-33 | Superseded by AD-43. _New 2026-11-23 (C-25, Q-28)._ No project sets `LangVersion`. Each compilation uses its target framework's default: C# 15 for every `net11.0` compilation and C# 14 for every `net10.0` compilation *(Microsoft Learn, "Language versioning", read 2026-11-23: "C# 15 is supported only on .NET 11 and newer versions … Using a C# language version newer than the version associated with your target TFM is unsupported.")*. Source shared by both builds must compile as C# 14. C# 15 syntax appears in it only inside `#if NET11_0_OR_GREATER`, and only where the public API and observable behavior are the same on both builds. Projects that build only for `net11.0` (the sample and its tests) use C# 15 freely. **Fred confirms or overrules this at Gate 2 (G2-1)**, because A-18 assumed C# 15 could apply to all of the repo's code at no cost to .NET 10. | — | `LangVersion=15` on every build (unsupported on `net10.0` per Microsoft; union types need `UnionAttribute` and `IUnion`, which ship in the .NET 11 runtime); `LangVersion=latest` (Microsoft warns against it); dropping `net10.0` (breaks C-2 and C-26). |
| AD-34 | _New 2026-11-23 (A-16, Q-26)._ The sample and `test/ValhallaTech.Security.IdentityModel.Samples.Tests` target `net11.0` only, as R-1 now requires. | — | Multi-targeting the sample (no consumer needs it, and R-1 rules it out). |
| AD-35 | Superseded by AD-45. _New 2026-11-23 (Q-30)._ A framework-bound NuGet dependency (R-17's definition) gets one conditional `PackageVersion` per target framework in `Directory.Packages.props`, for example `Condition="'$(TargetFramework)' == 'net10.0'"` *(conditional `PackageVersion` per framework read 2026-11-23 in Microsoft Learn's central package management page via Context7 `/websites/learn_microsoft_en-us_nuget`)*. Today that is `Microsoft.AspNetCore.Authentication.JwtBearer` in the library test project. Renovate keeps each entry on its own line: `packageRules` matching `Microsoft.AspNetCore.**` and `Microsoft.EntityFrameworkCore.**` with `matchCurrentValue` `/^10\./` → `allowedVersions` `/^10\./`, and the same for `/^11\./` *(both options read 2026-11-23 in Renovate's configuration docs via Context7 `/websites/renovatebot`)*. The `dotnet-sdk` (`global.json`) entry gets `allowedVersions` `<12.0.0`, so AC-6 keeps holding. | — | `VersionOverride` in the project file (splits the inventory AC-49 and AC-134 read); one version for both frameworks (11.0.0 cannot be used by `net10.0`). |
| AD-36 | Superseded by AD-46. _New 2026-11-23 (Q-30)._ AC-49 is checked by `build/Test-RequiredVersions.ps1`, which reads each project's `obj/project.assets.json` per target framework and computes each direct dependency's required version from the NuGet V3 registration resource, exactly as AC-49 defines it. It fails, naming the URL, when the registry is unreachable. | — | `dotnet list package --outdated` (compares against the newest stable version whatever framework it supports, so the `net10.0` leg of a framework-bound dependency can never pass). |
| AD-37 | Superseded by AD-47. _New 2026-11-23 (AM-1)._ Every workflow that builds installs the 10.0.x and 11.0.x SDKs through `actions/setup-dotnet` (`dotnet-version` lines `10.0.x` and `11.0.x`); `global.json` selects the 11 SDK, and the 10.0 install provides runtime 10.0.x. CI keeps one `ubuntu-latest` test job: `dotnet test` runs each target framework on its own runtime and writes one TRX file per framework, uploaded as an artifact. | — | One job per framework (doubles restore and build for no added signal; C-17); installing only the 11 SDK (no 10.0 runtime for the `net10.0` test run). |
| AD-38 | Superseded by AD-53. _New 2026-11-23 (AM-2, AC-144)._ The release gate is three named GitHub Actions runs on the release commit: the `dotnetcore.yml` CI run, the `codeql-analysis.yml` run, and a `release-gate.yml` run (WI-54) that runs every scriptable release-time check through `build/Invoke-ReleaseGate.ps1`. `release.yml` starts with a `verify-gates` job that refuses to publish unless all three runs for the tagged commit concluded `success`. The checks that need Fred's administrator scope (AC-20, AC-119), WI-45's dry run and U-6's history audit run locally before `release-gate.yml` is dispatched, with evidence logs. | — | A gate job inside `release.yml` (the run's conclusion is known only after publishing, so AC-144 cannot read it); a written checklist (no run conclusion to compare). |
| AD-39 | Superseded by AD-48. _New 2026-11-23 (AM-1)._ Supersedes AD-23. The benchmark project targets `net10.0` and `net11.0`. Set T, Ed448, AC-69 and AC-70 are run on Fred's workstation once per framework (`-f net10.0`, then `-f net11.0`) in one session, each run holding its baseline copies and the release candidate in one process on one runtime. No figure is compared across runtimes. | — | One runtime only (R-22 and AC-59 now need both). |
| AD-40 | Superseded by AD-49. _New 2026-11-23 (AM-1)._ The doc-snippet harness compiles each C# block twice, in a generated `net10.0` project and a generated `net11.0` project, each at its default language version. README examples are therefore valid C# 14. | — | One `net11.0` project (AC-102 needs both frameworks). |
| AD-41 | Superseded by AD-50. _New 2026-11-23 (AM-1)._ Supersedes AD-24. The coverage gate reads one Cobertura report per framework run and fails if either library is below 80.0% in either run. The 79.9% fixture still must fail. | — | Merging the two runs into one figure (R-47 asks for each run). |
| AD-42 | Superseded by AD-52. _New 2026-11-23 (AM-2)._ No date drives the plan. Phases, lanes and gates are unchanged; 1.0.0 is published when WI-46's gates pass. The reason to keep 1.0.0 lean is exposure (§2.2.1): S1 in assetvault-io and libraries built only for unsupported targets until it ships. If a gate cannot pass, Fred decides what gives (C-5, C-20's complete-release rule, scope, or the failing condition). | — | Re-planning around a new date (none was given). |
| AD-43 | _New 2026-11-23 (AM-3)._ Supersedes AD-33. **Records Fred's answer to G2-1, "Drop .NET 10." (AM-3).** Every compilation in the repository targets `net11.0`, so all code is C# 15 (C-25). The language comes from the target framework's default, not from a `LangVersion` property: no `.csproj`, `.props` or `.targets` sets `LangVersion`, and WI-1 proves the result with a `#error version` probe that must report C# 15. **Why the default and not an explicit `LangVersion=15.0`:** on `net11.0` the default is exactly C# 15 *(Microsoft Learn, "Language versioning", read 2026-11-23: ".NET 11.x → C# 15")*, so an explicit value adds nothing today. It would be a second setting that can drift from the target framework: left at 15 after a later retarget, or raised past the framework's default, which Microsoft does not support (C-27). Microsoft's guidance is that the default "ensures you use the latest language version compatible with your target framework", and it warns against `latest`. R-1 fixes the framework, so the framework fixes the language. | G2-1 (Q-28; answered by Fred, AM-3) | An explicit `LangVersion=15.0` in `Directory.Build.props` (redundant on `net11.0`, and a second knob that can disagree with the framework); `LangVersion=latest` (Microsoft warns against it); AD-33's per-framework split (moot once `net10.0` is gone). |
| AD-44 | _New 2026-11-23 (AM-3)._ Supersedes AD-32. Build layout: `ValhallaTech.Security.IdentityModel.slnx`; `global.json` pins SDK `11.0.100` with `rollForward: latestFeature`, so only an 11.0 SDK is selected; the root `Directory.Build.props` sets `TargetFramework=net11.0` for every project (libraries, both test projects, the sample and the benchmarks), plus warnings as errors and NuGetAudit, and no `LangVersion` (AD-43); `src/Directory.Build.props` holds library-only settings; central package management with one unconditional `PackageVersion` per package (AD-45); renamed directories. | D-9 | Per-project target frameworks (AD-32's reason, the sample clearing an inherited `TargetFrameworks`, is gone with a single framework, and one root setting cannot drift between projects). |
| AD-45 | _New 2026-11-23 (AM-3; R-17, R-48 and AC-49 as amended)._ Supersedes AD-35, the framework-bound version decision. Every direct NuGet dependency has one unconditional `PackageVersion` at its single latest stable version; no dependency is treated as framework-bound, and no conditional entries exist. Renovate keeps its default behavior for NuGet packages, with no per-major `allowedVersions` rules, so it proposes every newer stable version (R-48). The only remaining rule is `dotnet-sdk` (`global.json`) `allowedVersions: "<12.0.0"`, because AC-6 accepts only an 11.0 SDK; the SDK is not in R-17's inventory (AC-134), so R-48 does not cover it. | Q-30's per-framework rule, withdrawn by AM-3 | Keeping AD-35's per-major rules (would stop Renovate from proposing a newer stable version, against R-48 as amended); restricting the SDK rule away (a `global.json` bump to 12.0 would break AC-6). |
| AD-46 | _New 2026-11-23 (AM-3)._ Supersedes AD-36. AC-49 is checked by `build/Test-RequiredVersions.ps1`, which reads each project's `obj/project.assets.json` (`net11.0` target) and compares each direct dependency's resolved version with its single latest stable version from the NuGet V3 registration resource, as AC-49 now defines it. A latest stable version that does not support `net11.0` is reported as an A-14 case for Fred, never passed. It fails, naming the URL, when the registry is unreachable. | — | `dotnet list package --outdated` (AC-49 names the registration API as the method, and `--outdated` cannot report the A-14 "does not support `net11.0`" case). |
| AD-47 | _New 2026-11-23 (AM-3)._ Supersedes AD-37. Every workflow that builds installs only the 11.0 SDK through `actions/setup-dotnet` with `global-json-file: global.json`, which also brings runtime 11.0.x. CI keeps one `ubuntu-latest` test job; `dotnet test` writes one TRX per test project, uploaded as an artifact. | — | `dotnet-version: '11.0.x'` beside `global.json` (two sources for one version); installing the 10.0 SDK (no project needs it, and AC-6 forbids it). |
| AD-48 | _New 2026-11-23 (AM-3)._ Supersedes AD-39. The benchmark project targets `net11.0`. Set T, Ed448, AC-69 and AC-70 run on Fred's workstation on one .NET 11.0.x runtime, baseline copies and release candidate in one BenchmarkDotNet process. | — | Measuring .NET 10 as well (no `net10.0` build ships, R-22). |
| AD-49 | _New 2026-11-23 (AM-3)._ Supersedes AD-40. The doc-snippet harness compiles each C# block once, in a generated `net11.0` project with no `LangVersion`, so README examples may use C# 15. | — | Keeping a `net10.0` compile (no such build ships). |
| AD-50 | _New 2026-11-23 (AM-3)._ Supersedes AD-41. The coverage gate reads the one Cobertura report of the library test run and fails if either library is below 80.0% (R-47 as amended). The 79.9% fixture still must fail. Script, floor and fixture are as in AD-24. | — | — (one framework leaves one report). |
| AD-51 | _New 2026-11-23 (AM-4)._ Supersedes AD-27. No migration guide and no assetvault-io proof of any kind: R-54, M-1, AC-130 and AC-131 are cut (CUT-1 to CUT-4). WI-41, WI-45 and WI-47 are cut, keeping their IDs. Nothing in the plan reads, copies or builds `C:\Repo\assetvault-io` (NG-12). The CHANGELOG's 1.0.0 breaking-change list (WI-40) is the only migration aid, so each breaking entry names what replaces it. | AM-4 (CUT-1 to CUT-4) | Keeping the guide without the proof ("Keep guide, drop proof", which Fred did not choose). |
| AD-52 | _New 2026-11-23 (AM-3/AM-4)._ Supersedes AD-42. No date drives the plan; 1.0.0 is published when WI-46's gates pass. The reason to keep 1.0.0 lean is delivery, as §2.2.1 now says: G-1 is met only when ValhallaTech's .NET 11 projects can consume the packages, which starts at 1.0.0. No consumer's exposure is a reason (NG-12). If a gate cannot pass, Fred decides what gives (C-5, C-20's complete-release rule, scope, or the failing condition). | — | Re-planning around a date or a consumer (none applies). |
| AD-53 | Superseded by AD-55. _New 2026-11-23 (AM-4)._ Supersedes AD-38. The release gate is the same three named GitHub Actions runs on the release commit: `dotnetcore.yml`, `codeql-analysis.yml` and `release-gate.yml` (WI-54, through `build/Invoke-ReleaseGate.ps1`). `release.yml`'s `verify-gates` job refuses to publish unless all three concluded `success` for the tagged commit. The checks that need Fred's administrator scope (AC-20, AC-119) and U-6's history audit run locally before `release-gate.yml` is dispatched, with evidence logs. WI-45's dry run is no longer among them (AM-4). | — | As AD-38. |
| AD-54 | _New 2026-11-23 (AM-3/AM-4)._ Supersedes AD-31. The fixed strings and paths in 3.3.5: package metadata, the two-line modification notice and its placement, the report paths and row counts (one runtime), the CHANGELOG phrase list (topic 3 now names .NET 11 only), and the doc-snippet placement rules. The migration-guide phrase list is withdrawn with AC-130. | — | — (the ACs defer these values to §3). |
| AD-55 | _New 2026-11-23 (Gate 2; AC-144 as amended)._ Supersedes AD-53. The release gate has two kinds of gate, both read on the release commit:<br>**Workflow-run gates:** the `dotnetcore.yml`, `codeql-analysis.yml` and `release-gate.yml` (WI-54) runs.<br>**Non-workflow gates**, each recorded as a commit status posted only after the check passes, before publication: `release-gate/ac-20` (AC-20, private vulnerability reporting), `release-gate/ac-119` (AC-119, advisory alerts) and `release-gate/test-first-audit` (U-6's audit of AC-121 and AC-132). Each status has `state: success`, `target_url` empty, and `description` exactly `evidence: <yyyy-mm-dd>/<log file name>`, the evidence log's path under `C:/Users/fsmit/.claude/evidence/`.<br>WI-46's operator posts all three with `gh api repos/IronSloth/IdentityModel-Supa/statuses/<sha>`, under Fred's `gh` authorization, which needs push access to the repository (a classic token's `repo:status` scope is enough) *(GitHub REST docs, "Commit statuses", read 2026-11-23: creating a status needs "push access", and "the `repo:status` OAuth scope grants targeted access to statuses")*. The Agent Evaluator runs U-6's audit and writes its evidence log but posts nothing, because an evaluator does not write to what it judges.<br>`release.yml`'s `verify-gates` job enforces both kinds before publishing: the three runs concluded `success`, and for each of the three contexts the newest status on the tagged commit is `success` *(statuses are listed newest first, read 2026-11-23 in the same docs)*. A skipped or failed check therefore blocks publication instead of being found afterward. | — | Checking the statuses only after publishing (a skipped check would already have shipped); letting the Agent Evaluator post its own status (an evaluator writing to the thing it judges); one combined status for all local checks (AC-144 asks for one context per gate). |
| AD-56 | Superseded by AD-58. _New 2026-11-23 (Gate 2, F-1)._ A check of a `1.x` requirement never gates 1.0.0 (§2.2.1). It still runs and is recorded, and a miss is a finding for its owning item. Two mechanisms: (1) in `release-gate.yml` (WI-54), `Invoke-ReleaseGate.ps1` reports AC-58, AC-60 to AC-67, AC-69, AC-113, AC-120 and AC-123 as record-only and exits non-zero only on a 1.0.0 check; (2) the CI checks of `1.x` requirements (CSharpier for AC-120, the coverage gate for AC-116 and AC-117, the doc-snippet compile for AC-102) run in a separate workflow, `quality.yml`, which is not one of AC-144's gate runs. `dotnetcore.yml` keeps only the build, the full test run (AC-3, AC-4, AC-57, all 1.0.0) and the pack. The full test run still includes the acceptance tests of `1.x` requirements, because AC-4 (R-2, 1.0.0) requires every test to pass. | — | `continue-on-error` on the `1.x` steps inside `dotnetcore.yml` (hides a red check inside a green run); gating every check (contradicts §2.2.1); dropping the `1.x` checks from the gate run entirely (their results would go unrecorded at release). |
| AD-57 | _New 2026-11-23 (Gate 2, F-4)._ The CHANGELOG's `## [1.0.0] - <date>` heading: WI-40 writes the date of its own commit, so AC-104's ISO-date check can pass from Phase 2 on. The final date is set by WI-55, the Documentation Agent's last commit before the gate: it changes the date to the day that commit is made, and that commit is the release commit the gates run on (WI-46 step 2). If any later change is needed, it makes a new release commit, and WI-55 runs again on that day before the gates rerun. A tag pushed on a later day than that date does not change it, because the edit would itself be a new release commit; the feed's published timestamp (AC-8) records the exact time. | — | Editing the date after the gates pass (creates an ungated commit); having WI-46 (`ci`) edit `CHANGELOG.md` (the Documentation Agent owns that file); a placeholder that is not an ISO date (fails AC-104 until release). |
| AD-58 | Superseded by AD-65. _New 2026-11-23 (Gate 2, third review; the §2.3 convention "`1.x`-only checks never block 1.0.0", and AC-4, AC-7, AC-121 and AC-132 as amended)._ Supersedes AD-56. At the 1.0.0 release commit every `1.x`-only AC, test and work-item audit runs, is never skipped, and is recorded with the release evidence. A failure, or a `1.x`-only item that has not landed, fails neither 1.0.0 nor AC-4, AC-7, AC-121, AC-132 or AC-144. The `1.x`-only ACs are the convention's list: AC-52 to AC-54, AC-58, AC-60 to AC-67, AC-69, AC-87, AC-88, AC-100 to AC-103, AC-106 to AC-108, AC-112 to AC-117, AC-120, AC-122 and AC-123.<br>**(1) One scope file.** `build/release-scope.json` holds `oneXRequirements` (R-19, R-21, R-23, R-24, R-32, R-39, R-40, R-41, R-43, R-45, R-46, R-47, R-49, R-51 and R-52, the requirements §2.2.1 does not list as `1.0.0`, less the cut R-54) and `oneXAcs` (the list above). The test script, `Invoke-ReleaseGate.ps1`, the misuse test and U-6's audit read their `1.x` lists from it, and no other file holds one. When a later release carries a `1.x` requirement, that release's prep commit removes the requirement and its ACs from the file; the misuse test then forces the marker off its tests, so from that release its checks block.<br>**(2) Traits.** The marker is `[Trait("Release", "1.x")]`. The requirement trait is `[Trait("Requirements", "<ids>")]`, whose value lists the requirement IDs the test verifies, comma-separated with no spaces, for example `"R-43"` or `"R-19,R-45"`. Both are string literals on the test method, never on a class or the assembly. A test is marked only if every ID it lists is in `oneXRequirements`. A test that verifies any `1.0.0` or `all` requirement carries no marker and blocks. An unmarked test may carry the requirement trait, but need not.<br>**(3) Misuse check.** `ReleaseScopeTraitTests`, one shared source file compiled into every test project (WI-56), reflects over its own assembly. It fails if a marked test has no requirement trait, a malformed value, or an ID outside `oneXRequirements`; if any `Release` trait has a value other than `1.x`; or if either trait is on a class or the assembly. It is unmarked, so it blocks in CI, and `Invoke-ReleaseGate.ps1` runs it as a gating check.<br>**(4) Split test runs.** `build/Invoke-Tests.ps1 -Scope Blocking` or `-Scope OneX` runs `dotnet test -c Release --no-build` once per test project with a TRX logger. `Blocking` filters `Release!=1.x` and returns `dotnet test`'s exit code. `OneX` filters `Release=1.x`, writes every failure to the job summary and as a `::warning::` annotation, and exits 0 when every project's run either wrote a TRX or reported that no test matches the filter; it exits non-zero only when a run did not complete (a build, host or abort error). `dotnetcore.yml` and `release.yml` run both as separate steps, the `OneX` step with `if: ${{ !cancelled() }}`, and upload both sets of TRX files.<br>**(5) Separate workflow.** The CI checks of `1.x` requirements run in `quality.yml`, which is not an AC-144 gate run: CSharpier (AC-120), coverage (AC-116, AC-117), doc snippets (AC-102) and the packed readme (AC-103). `dotnetcore.yml` keeps the build, the split test run (AC-3, AC-4, AC-57) and the pack. AC-115's package validation lands after 1.0.0 (WI-52), so it never runs on the 1.0.0 commit.<br>**(6) Release gate.** `Invoke-ReleaseGate.ps1` reports every check whose AC is in `oneXAcs` as record-only, and exits non-zero only on a 1.0.0 check.<br>**(7) Audit.** U-6 audits every in-scope AC and work item. Its evidence log separates the 1.0.0 scope, which decides `release-gate/test-first-audit`, from the `1.x` scope, which is recorded.<br>**(8) Sequencing.** No `1.x`-only work item (WI-7, WI-35, WI-48 to WI-52) is a prerequisite of WI-55 or WI-46. WI-7 stays in Phase 0 only to keep the reformat serial (RK-9). A `1.x`-only item that lands after 1.0.0 is audited when it lands, and this spec completes only when every item has landed and every check passes with none exempt (AC-4's last bullet). | — | `continue-on-error` on one combined test step (a 1.0.0 failure would also leave the job green, which breaks AC-7); one run with its exit code ignored and its TRX parsed for failures (a trait-to-result mapping in TRX was not verified, and the blocking verdict would rest on a parser); `[Fact(Skip = …)]` on `1.x` tests (the convention forbids skipping); a `1.x` list in each script (copies that drift); keeping WI-35 as a WI-46 prerequisite (a `1.x`-only item would hold 1.0.0 back, against §2.2.1). |
| AD-59 | _New 2026-11-23 (Gate 2, third readiness review, F-1 to F-5)._ The acceptance tests of the out-of-process seams (chains C14 to C19) are Pester tests, one file per seam under `build/tests/`:<br>- `Measurement.Tests.ps1` (C14);<br>- `CiRun.Tests.ps1` (C15);<br>- `ReleaseGates.Tests.ps1` (C16);<br>- `Feed.Tests.ps1` (C17);<br>- `RepositorySecurity.Tests.ps1` (C18);<br>- `TestFirstHistory.Tests.ps1` (C19).<br>**Inputs.** Each file declares `#Requires -Modules @{ ModuleName = 'Pester'; ModuleVersion = '5.0.0' }`. It takes its inputs (commit SHA, repository, version, feed source, paths) as `param()` values.<br>**Runner.** `build/Invoke-AcceptanceTests.ps1 -Path <file> -Data <hashtable> [-Tag] [-ExcludeTag]` passes the inputs through `New-PesterContainer -Data`, and runs with `Run.Exit = $true` and `TestResult.Enabled = $true`, so NUnit XML results are written beside the evidence log *(Pester docs, read 2026-11-23 via Context7 `/pester/docs`: `New-PesterContainer -Data` supplies parameter values to a test file, and `Run.Exit` exits non-zero when the run fails)*.<br>**Names and tags.** Every `It` name starts with the AC ID it checks, so `oneXAcs` (AD-58) decides whether a failure is record-only. `Post` marks a test that needs a published version or posted statuses. `Admin` marks a test that needs Fred's administrator read, and an `Admin` test never runs in a workflow.<br>**Owner.** The PowerShell CI Engineer owns the `tests` items. | F-1 to F-5 (third readiness review) | An xUnit project that shells out to `gh`, `git` and `dotnet` (it would put network- and credential-bound checks into every PR's blocking `dotnet test` run); Bash with `bats` (a second test toolchain, where every other script in `build/` is PowerShell). |
| AD-60 | _New 2026-11-23 (Gate 2, third readiness review)._ Every out-of-process seam gets a chain, and §3.7's sentence that exempted out-of-process `integration` ACs from chains is withdrawn.<br>**Stubs steps.** A chain's stubs step is the item that first creates the surface its tests read: WI-38's header-only report files (C14), WI-44's feed publishing path (C17) and WI-62's stub of the audit tool (C19). Where the surface already exists at `555d31c` (GitHub's Actions, commit-status and repository APIs, for C15, C16 and C18), no stubs item is added, as AD-16 rules for in-process seams.<br>**Red check.** Every acceptance test fails by assertion at its tests item.<br>**Named exemptions.** Three named classes are exempt from the chain's later green check. §3.7 lists each test with its reason and the item that turns it green, and none is dropped:<br>- **release checks** go green only on a release commit after publication or the posting of statuses (WI-46, WI-68);<br>- **Fred-state checks** pass or fail according to a repository setting only Fred can change (U-2, U-3, U-4). They are exempt from both the red and the green check, and must pass at the release gate;<br>- **`1.x` record-only checks** follow AD-58.<br>**Audit scope.** These chains change CI, feed or repository state, not library or sample behavior, and they are outside AC-121's in-scope list, so U-6 audits none of them. | F-1 to F-5 (third readiness review) | Leaving out-of-process ACs without chains (F-1 to F-5); naming release checks as guards (a guard passes against the stubs by design, and these fail by design until publication). |
| AD-61 | _New 2026-11-23 (Gate 2, F-5)._ The AC-121 and AC-132 audit becomes an executable tool, `build/Test-TestFirstHistory.ps1`, built test-first as chain C19: WI-62 stubs it, WI-63 tests it and WI-64 implements it.<br>**Inputs.** It reads `build/test-first-chains.json`: each chain from C1 to C13b, with its in-scope ACs, tests item, implementing item and guards, and for AC-132 the tests selector of each behavior-changing work item. It also reads `build/release-scope.json` (AD-58).<br>**Method.** It walks `git rev-list --reverse --topo-order 555d31c..<sha>` and finds each introducing and implementing commit by the AD-28 commit subjects. It checks each such commit out in a `git worktree` under the system temp directory, runs the named tests with `dotnet test --filter`, and reads their TRX outcomes. A build failure counts as "compile error", never as red.<br>**Output.** It writes `test-first-audit.json` and `test-first-audit.md`, with a 1.0.0-scope verdict and a `1.x`-scope verdict, or a single verdict when both scope lists are empty (AD-64).<br>**Who runs it.** U-6, the Agent Evaluator, runs it on the release commit. The evaluator reviews the report, including whether each red result was an assertion about its AC, and writes the evidence log that WI-46 reads. | F-5 (third readiness review) | The evaluator's manual audit (not an executable check, so F-5 stands); reading order from GitHub's PR API (merges without squashing keep every commit in git, and the API adds nothing git lacks). |
| AD-62 | _New 2026-11-23 (Gate 2, F-4; Security yes)._ The repository-security checks (C18: AC-20, AC-118 and AC-119) only read.<br>- `RepositorySecurity.Tests.ps1` calls `gh api` with its default GET method and never passes `--method` or `-X`. WI-61's done-when checks this with `rg`.<br>- The `Admin`-tagged tests (AC-20, AC-119) run only locally, under Fred's `gh` login, at WI-46 step 1 and WI-68. They never run in a workflow, so no workflow token gains administration read.<br>- Their output and evidence hold the endpoint, the HTTP status and the `enabled` value only, never response headers or token text.<br>- No test creates, lists or reads a security advisory or a vulnerability report, so the check cannot disclose an undisclosed report (R-8).<br>- The Security Reviewer reviews WI-61 at Gate 1. | F-4 (third readiness review) | Running AC-20 and AC-119 in `release-gate.yml` with a fine-grained token holding administration read (a long-lived secret with broad scope in CI, for a check run once per release). |
| AD-63 | _New 2026-11-23 (Gate 2, F-3)._ The feed tests (C17) build their consumer outside the repository.<br>**Consumer.** `Feed.Tests.ps1` copies the fixture `build/fixtures/feed-consumer/` (a `net11.0` console holding the AC-9 program, whose two `PackageReference`s read `Version="$(ConsumerVersion)"`) to a new directory under the system temp path. No repository `Directory.Build.props` or central package file applies there.<br>**Sources.** It writes a `nuget.config` in that directory with two sources, the feed under test and nuget.org. Package source mapping sends `ValhallaTech.*` to the feed under test only and `*` to nuget.org, so both packages can only come from the feed (AC-9).<br>**Credential.** The feed credential is the environment variable `IRONSLOTH_PACKAGES_READ_TOKEN` (U-5). It is referenced in `packageSourceCredentials` as `%IRONSLOTH_PACKAGES_READ_TOKEN%` (NuGet's environment-variable expansion in config values is inferred; WI-60 verifies it first, RK-22). No token is written to disk or printed, and the directory is deleted after the run. | F-3 (third readiness review) | `dotnet nuget add source --store-password-in-clear-text` (writes the token to a file); a consumer inside the repository (repository props and central versions would change what restores). |
| AD-64 | _New 2026-11-23 (Gate 2; R-2 and R-50 as amended, F-6)._ The release that completes this spec is **1.1.0**, published after every `1.x`-only item has landed (WI-65 to WI-68).<br>**Prep.** WI-65 first removes every `1.x` marker. WI-66 then empties `oneXRequirements` and `oneXAcs`. As a result:<br>- every test runs in the blocking step;<br>- `Invoke-ReleaseGate.ps1` has no record-only check;<br>- the C19 audit runs in one scope;<br>- `quality.yml` must also conclude `success`;<br>- the misuse check rejects any marker left.<br>**Gates.** 1.1.0 goes through the same gates as 1.0.0 (AD-55, AD-57), and WI-67 dates the `## [1.1.0]` CHANGELOG entry on its release commit.<br>**Version.** It is a minor version because every `1.x` requirement is non-breaking (§2.2.1). | — | Ending the spec at a commit with no release (R-2 now names "the release that completes this spec"); 1.0.1 (the `1.x` work adds non-breaking behavior, which SemVer counts as minor); leaving markers in place and filtering by the scope file alone (a marker that no longer exempts anything would mislead the next reader). |
| AD-65 | _New 2026-11-23 (Gate 2, fourth readiness review, F-1)._ Supersedes AD-58. AD-58's points (1) to (7) carry over unchanged, word for word, and are read from AD-58's row. Only point (8) is replaced, and it now reads:<br>**(8) Sequencing.** WI-35 and WI-48 to WI-52, the other `1.x`-only work items, are not prerequisites of WI-55 or WI-46. A `1.x`-only item that lands after 1.0.0 is audited when it lands, and this spec completes at the 1.1.0 release (AD-64).<br>**The exception is WI-7.** WI-7 holds the one-time CSharpier reformat and the creation of `quality.yml`. It stays in Phase 0, ahead of WI-8 and WI-39, because the reformat and the annotation pass each touch every file (RK-9). So WI-7 **does gate 1.0.0, as sequencing.** Only AC-120's pass-or-fail result is record-only at 1.0.0.<br>**When WI-7 is done.** WI-7 is done, and WI-8 and WI-39 may start, once its reformat commit and `quality.yml` have merged and the build and the test results are unchanged. That holds whatever AC-120's check returns. A failed AC-120 is recorded and goes back to WI-7 as a finding, fixed before WI-66.<br>**Fallback.** If CSharpier cannot format the code at all, for example because it does not parse C# 15 syntax, WI-7 merges `quality.yml` with its check step and no reformat commit, and AC-120 is recorded as failed. WI-8 and WI-39 then start, and the reformat is done in Phase 4, before WI-65, when no other lane is open. | — | Removing the WI-7 → WI-8 and WI-7 → WI-39 edges, so that no 1.0.0 item waits on WI-7 (the reformat would then run beside the annotation pass and the Phase 1 lanes, the merge-conflict risk RK-9 exists to avoid); keeping AD-58's point (8) as written (it said WI-7 does not gate 1.0.0, which WI-8's and WI-39's depends-on lines contradict). |

### 3.1 Affected systems & repos

| System / repo | Nature of change | Owner domain |
| --- | --- | --- |
| `src/ScottBrady.IdentityModel` → `src/ValhallaTech.Security.IdentityModel` | Renamed, retargeted to `net11.0` only, C# 15 (amended 2026-11-23 (AM-3/AM-4); AD-43, AD-44). EdDSA key isolation, validation, disposal, JWK conversion, Base16/Base62, stream reads. | `backend` |
| `src/ScottBrady.IdentityModel.AspNetCore` → `src/ValhallaTech.Security.IdentityModel.AspNetCore` | Renamed, retargeted to `net11.0` only (amended 2026-11-23 (AM-3/AM-4)). Validator messages through an error describer, one-step registration, options validation, tag-helper guards. | `backend` |
| `test/ScottBrady.IdentityModel.Tests` → `test/ValhallaTech.Security.IdentityModel.Tests` | Retargeted to `net11.0` only; every library test runs on runtime 11.0.x (amended 2026-11-23 (AM-3/AM-4)). FluentAssertions replaced by AwesomeAssertions, AutoFixture removed, coverlet added. Hosts every library acceptance and unit test. | `tests` |
| New `test/ValhallaTech.Security.IdentityModel.Samples.Tests` | In-process host tests of the sample (`WebApplicationFactory<Program>`), targeting `net11.0` like every project (AD-34, AD-44). Kept separate so library coverage excludes the sample (A-6). | `tests` |
| `samples/ScottBrady.IdentityModel.Samples.AspNetCore` → `samples/ValhallaTech.Security.IdentityModel.Samples.AspNetCore` | Retargeted to `net11.0` (AD-34, AD-44). Hosting: minimal hosting, DryIoc, public-only validation key, diagnostics only in Development. UI: Bootstrap 5's CSS and JS bundle, unmodified, restored through LibMan (GP-10); working mobile menu; no jQuery; fixed label. | `backend` (hosting), `frontend` (views, assets) |
| New `perf/ValhallaTech.Security.IdentityModel.Benchmarks`, `perf/results/1.0.0/` | BenchmarkDotNet harness with a source-level `555d31c` baseline, targeting `net11.0` and run on one .NET 11 runtime (amended 2026-11-23 (AM-3/AM-4); AD-48), and the committed report. | `tests` |
| Root build files: `global.json`, `Directory.Build.props`, `src/Directory.Build.props`, `Directory.Packages.props`, `ValhallaTech.Security.IdentityModel.slnx` (replaces `ScottBrady.IdentityModel.sln`), `assets/valhallatech-logo.png` | New shared build layout (D-9; AD-44, amended 2026-11-23 (AM-3/AM-4): 11.0 SDK pin, `TargetFramework=net11.0` for every project in the root props, one version per package, no `LangVersion`). The logo exists, uncommitted, since 2026-11-23 (U-1). | `backend` |
| `.config/dotnet-tools.json`, `.csharpierrc.json`, `renovate.json`, `build/*.ps1`, `build/release-scope.json`, `.github/workflows/*.yml` (new `release.yml`, `release-gate.yml` and `quality.yml`) | CI on the 11.0 SDK only (amended 2026-11-23 (AM-3/AM-4); AD-47), least-privilege tokens, format check, coverage gate, doc-snippet compile on `net11.0`, the AC-49 latest-stable check, Renovate, the named release-gate run, and release to the private feed. _Amended 2026-11-23 (Gate 2; AD-58):_ the `1.x` scope file, the split test run (`build/Invoke-Tests.ps1`), and the non-gating `quality.yml`. | `ci` |
| `test/Directory.Build.props`, `test/Shared/ReleaseScopeTraitTests.cs` | _Added 2026-11-23 (Gate 2; AD-58)._ The `1.x` marker misuse check, compiled into every test project (WI-56). | `tests` |
| `build/tests/*.Tests.ps1`, `build/fixtures/measurement-*/`, `build/fixtures/feed-consumer/` | _Added 2026-11-23 (Gate 2, third review; AD-59)._ The Pester acceptance tests of chains C14 to C19, with their fixtures. | `tests` |
| `build/Invoke-AcceptanceTests.ps1`, `build/Test-TestFirstHistory.ps1`, `build/test-first-chains.json` | _Added 2026-11-23 (Gate 2, third review; AD-59, AD-61)._ The Pester runner and the executable test-first audit. | `ci` |
| `SECURITY.md`, `README.md`, new `CHANGELOG.md`, new `src/*/README.md` | Repository and package documentation. The migration guide `docs/migration/scottbrady-4.1.0-to-1.0.0.md` is no longer created (amended 2026-11-23 (AM-3/AM-4): AM-4 cut it; AD-51). | `docs` |
| GitHub settings on `IronSloth/IdentityModel-Supa` | Private vulnerability reporting, Dependabot alerts, Renovate app install. | Unmapped (Fred) |
| Private NuGet feed: GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json` (D-2 decided, GP-10) | Hosts 1.0.0. Push from the release workflow; consumers restore with a `read:packages` credential. | `ci`, plus Unmapped (Fred: credentials) |
| `C:\Repo\assetvault-io` | **Not affected, and not used.** _Amended 2026-11-23 (AM-3/AM-4):_ it used to be the AC-131 subject, through a scratch copy of `main` at `863c0cb0cde62f820b76b9c76e1b4cf245f7dc55`. AC-131 is cut (CUT-4), and no item reads, copies or builds it (AD-51, NG-12). | none |

**Grounding: what was read and what is inferred.**

*Verified by reading the code at `555d31c` on 2026-10-25:*
- Every library source file: `EdDsa.cs`, `EdDsaParameters.cs`, `EdDsaSecurityKey.cs`, `EdDsaSignatureProvider.cs`, `ExtendedCryptoProvider.cs`, `ExtendedSecurityAlgorithms.cs`, `Base16.cs`, `Base62.cs`, `ExtendedJsonWebKeyConverter.cs`, `StreamExtensions.cs`, `ByteArrayExtensions.cs`, `ExtendedPasswordOptions.cs`, `ExtendedPasswordValidator.cs`, `NewPasswordTagHelper.cs`.
- All four `.csproj` files, the `.sln`, both workflows and `SECURITY.md`.
- The sample's `Startup.cs`, `Program.cs`, `SampleOptions.cs`, `HomeController.cs`, `_Layout.cshtml` and `PasswordRules.cshtml`, and the `wwwroot` file list.
- Every test method name (157 methods), plus the test files whose assertions this plan changes: `EdDsaParametersTests`, `EdDsaCreationTests`, `EdDsaSecurityKeyTests`, `NewPasswordTagHelperTests`, `StreamExtensionsTests`, `Base62Tests`, `JwtBearerHandlerTests`, and the AutoFixture usage (one file).
- In assetvault-io: `EdDsaKeyMaterial.cs`, `NuGet.config` (nuget.org only), `renovate.json`, and its `main` ref. _(Historical since 2026-11-23: AM-3 and AM-4 make assetvault-io irrelevant to this plan; NG-12.)_

*Verified through Context7 on 2026-10-25:*
- `JsonWebTokenHandler` signature validation uses `validationParameters.CryptoProviderFactory ?? key.CryptoProviderFactory`. Source: `JsonWebTokenHandler.ValidateToken.cs` on the library's `dev` branch, `/azuread/azure-activedirectory-identitymodel-extensions-for-dotnet`. A per-key factory is therefore honored unless the app sets one on `TokenValidationParameters`.
- SimpleBase exposes `Base62.Default` on a `DividingCoder` that implements `IBaseCoder` and `INonAllocatingBaseCoder` (`/ssg/simplebase`, `docs/simplebase.base62.md`).

*Verified on 2026-11-23 for the amendment (AM-1/AM-2/Q-30):*
- **Q-27, .NET 11 support.** .NET 11 is an STS release, first released 2026-11-10, end of support **2028-11-14**. .NET 10 is LTS with the same end of support, 2028-11-14. So A-17 holds: .NET 11 is supported exactly as long as .NET 10. *(Read 2026-11-23 in Microsoft's release metadata, `https://builds.dotnet.microsoft.com/dotnet/release-metadata/releases-index.json`, channel 11.0: `"release-type": "sts"`, `"eol-date": "2028-11-14"`, `"latest-sdk": "11.0.100"`, `"latest-runtime": "11.0.0"`; channel 10.0: `"release-type": "lts"`, `"eol-date": "2028-11-14"`, `"latest-runtime": "10.0.15"`. Confirmed the same day on the support policy page, `https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core`: ".NET 11 | November 10, 2026 | … | STS | Active | November 14, 2028" and "STS releases get free support and patches for two years.")*
- **Q-28, C# 15 on `net10.0`.** The default language is C# 15 for .NET 11.x and C# 14 for .NET 10.x. "C# 15 is supported only on .NET 11 and newer versions … Using a C# language version newer than the version associated with your target TFM is unsupported." Microsoft also warns: "Don't set the `LangVersion` element to `latest`." *(Read 2026-11-23 on Microsoft Learn, "Language versioning", `learn.microsoft.com/dotnet/csharp/language-reference/language-versioning`, updated 2026-11-12, and "Configure language version".)* The C# 15 features are collection expression arguments, union types, closed hierarchies, extension indexers, labeled `break` and `continue`, non-virtual static interface members on older runtimes, and memory safety (preview only). Union types "rely on the UnionAttribute and IUnion types in the `System.Runtime.CompilerServices` namespace. The .NET 11 runtime includes these types." *(Read 2026-11-23 on Microsoft Learn, "What's new in C# 15", updated 2026-11-19, and "Union types", updated 2026-10-23.)* The pages name no runtime need for the other features, but the "unsupported" rule covers the whole language version. That was AD-33 and G2-1; Fred answered with AM-3, and the same facts now ground AD-43: on `net11.0` the default language is C# 15, so no `LangVersion` is needed (now C-27 in §1.5).
- **Conditional `PackageVersion` per target framework** is supported by central package management *(read 2026-11-23 in Microsoft Learn's central package management page via Context7 `/websites/learn_microsoft_en-us_nuget`)*: AD-35, superseded by AD-45 (no longer used).
- **Renovate `matchCurrentValue` and `allowedVersions`** exist as `packageRules` options, with regex values *(read 2026-11-23 in Renovate's configuration-options docs via Context7 `/websites/renovatebot`)*: AD-35, superseded by AD-45, which keeps only an `allowedVersions` rule for `dotnet-sdk`.
- _Added 2026-11-23 (AM-3/AM-4):_ Q-27 still holds for the single framework: .NET 11 support ends 2028-11-14, not before .NET 10's (the releases-index and support-policy reads above). A-17 holds. _Amended 2026-11-23 (Gate 2): §1.6 and §2.6 now record Q-27 as resolved with this date, and M-7 states it._
- _Added 2026-11-23 (Gate 2):_ **Commit statuses.** Creating a status needs "push access" to the repository, and "the `repo:status` OAuth scope grants targeted access to statuses"; listing them needs pull access; the list is newest first, 30 per page by default and 100 at most; each status carries `context`, `state`, `description`, `created_at` and `creator`; there is "a limit of 1000 statuses per sha and context". The page states no description length limit. *(Read 2026-11-23 in GitHub's REST docs, `docs.github.com/en/rest/commits/statuses`, via WebFetch.)* AD-55.
- _Inferred (Gate 2):_ a workflow token with `statuses: read` can list a commit's statuses in `verify-gates`: WI-44.
- **The repository today** *(read 2026-11-23)*: the core library targets `net8.0;net9.0` and references BouncyCastle.Cryptography 2.4.0 and Microsoft.IdentityModel.Tokens/JsonWebTokens 7.7.1. The AspNetCore library targets `net6.0;net8.0;net9.0` and uses `<FrameworkReference Include="Microsoft.AspNetCore.App" />`, so it has no framework-bound NuGet dependency. The test project targets `net8.0;net9.0` with `LangVersion=latest` and references `Microsoft.AspNetCore.Authentication.JwtBearer` 8.0.11 (the one dependency that would have been framework-bound under the two-framework plan; with `net11.0` only it simply takes its latest stable 11.0.x, AD-45). The sample targets `net8.0` and references JwtBearer, Razor RuntimeCompilation, Identity.EntityFrameworkCore and EF Core InMemory, all 8.0.11. `dotnetcore.yml` still runs the three-OS matrix on the 9.0 SDK.
- **Relayed by the /spec coordinator, 2026-11-23, not re-run by the Architect:** installed SDKs 10.0.403, 10.0.404 and 11.0.100, and runtimes 10.0.14, 10.0.15 and 11.0.0, on ODIN01; U-1's logo at `assets/valhallatech-logo.png` (512×512 transparent PNG, uncommitted); private vulnerability reporting (U-2) and Dependabot alerts (U-3) still disabled, via `gh api`; JwtBearer 11.0.0 declares only `net11.0`, and 10.0.14 and 10.0.15 only `net10.0` (NuGet registration API).

*Inferred, not verified (each is checked by the work item named):*
- A new `AsymmetricSecurityKey` shares the `CryptoProviderFactory.Default` instance, which is why line 16 writes process-wide state. The review's [confirmed] probe supports this. Checked by AC-21 in WI-11.
- `new CryptoProviderFactory()` is cheap enough to create per key: RK-4, AC-25.
- At `555d31c`, `ExtendedCryptoProvider.Create` throws for a `JsonWebKey` argument, so AC-29 fails there. §2 (AC-121) now records the same conclusion from reading the code. Nobody has run it.
- .NET 9+ `Convert.ToHexStringLower` and `Convert.FromHexString` throw `FormatException` on odd or invalid input: WI-23.
- `Stream.ReadAtLeast` exists in .NET 7+: WI-25.
- jsDelivr's `/npm/` path, which LibMan's `jsdelivr` provider uses, serves npm-tarball files byte for byte, so the restored Bootstrap files are in set V: RK-6, AC-143 in WI-36. That the bundle contains jQuery-detection text is no longer inferred; GP-10 records the coordinator's measurement of 19 mentions.
- `Microsoft.Web.LibraryManager.Build`, `DryIoc.Microsoft.DependencyInjection` 6.2.0 and DryIoc.dll 5.4.3 restore and run on `net11.0` (the sample, and the DryIoc test in the library test project; amended 2026-11-23 (AM-3/AM-4)): WI-35, WI-37, WI-30.
- _Added 2026-11-23 (AM-1/AM-2/Q-30); amended the same day (AM-3/AM-4), which withdrew the two-framework inferences:_
  - The 11.0.100 SDK reads `.slnx`: WI-1.
  - `actions/setup-dotnet` installs the SDK that `global-json-file: global.json` names, honoring its `rollForward`, and the SDK brings runtime 11.0.x: WI-3.
  - The latest stable xunit 2.9.x, `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk`, Moq, AwesomeAssertions and coverlet run on `net11.0`: WI-2.
  - The latest stable BenchmarkDotNet recognizes `net11.0` as a runtime; if not, its in-process toolchain is used: WI-38, RK-12.
  - The workflow token with `actions: read` can list the runs of other workflows for a commit (`gh run list --commit`): WI-44.
  - _Withdrawn (AM-3):_ the SDK building `net10.0` through its targeting pack, multi-framework TRX output, and Renovate extracting conditional `PackageVersion` entries. No item relies on them now.
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

**Every row marked breaking goes into the CHANGELOG (WI-40), with what replaces it.** _Amended 2026-11-23 (AM-3/AM-4): was "the CHANGELOG and the migration guide (WI-40, WI-41)". The guide is cut (AD-51), so the CHANGELOG is the only migration aid._

| Surface | Change | Breaking? | Decision |
| --- | --- | --- | --- |
| All public types | Namespace and assembly rename (3.3.1). | **Yes** | C-7, C-8 |
| Target framework | `net11.0` only. .NET 6, 8, 9 and 10 are dropped, so a consumer must be on .NET 11. _Amended 2026-11-23 (AM-1/AM-2/Q-30): `net10.0` and `net11.0`. Amended 2026-11-23 (AM-3/AM-4): `net11.0` only. Originally "`net10.0` only"._ | **Yes** | C-2 (as amended by AM-3), AD-44 |
| Dependencies | Minimum versions rise to latest stable: Microsoft.IdentityModel.* 8.x (a major bump from 7.7.1), BouncyCastle.Cryptography 2.7.x (as of 2026-10-25). SimpleBase (latest stable, GP-10) is new. A consumer whose own direct reference is lower gets NU1605; the CHANGELOG tells it to raise its reference. _Amended 2026-11-23 (AM-3/AM-4): the assetvault-io example and "the guide" are removed (NG-12, AD-51)._ | **Yes** | G-9, §2.2.1(a) |
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

#### 3.3.3 Contract notes for consumers (feed into WI-40's CHANGELOG; amended 2026-11-23 (AM-3/AM-4): was "into WI-41", the cut migration guide)

- **JWKS consumers.** A `JsonWebKey` parsed from JSON carries the default `CryptoProviderFactory`, which has no EdDSA support once no global registration exists. Convert it with `TryConvertToEdDsaSecurityKey`, for example in `IssuerSigningKeyResolver`. At `555d31c` this path also failed (the provider threw for a `JsonWebKey`), so this is not a regression. It is still documented.
- **`TokenValidationParameters.CryptoProviderFactory`.** It overrides the key's factory, as the Context7 excerpt above shows. An app that sets it must include EdDSA support in it, or leave it unset.
- **One `EdDsa` instance stays safe to share across threads** (G-16). A new signer is still created per operation.

#### 3.3.4 Build layout (resolves D-9: AD-44, which supersedes AD-32 and AD-11)

_Amended 2026-11-23 (AM-1/AM-2/Q-30): a `net10.0`/`net11.0` layout with per-project frameworks and a C# 14/C# 15 split. Amended again 2026-11-23 (AM-3/AM-4): Fred dropped .NET 10 (AM-3), so every project targets `net11.0` and all code is C# 15. Originally the layout pinned the 10.0 SDK and set `TargetFramework=net10.0` and `LangVersion=latest` for every project._

- **`ValhallaTech.Security.IdentityModel.slnx`.** XML solution format, read by the .NET 11 SDK CLI that `global.json` selects. It replaces the GUID-heavy `.sln`, and diffs and merges are readable. Alternative lost: keeping `.sln`, which brings nothing a repository built only by the 11.0 SDK needs.
- **`global.json`.** Pins SDK `11.0.100` with `rollForward: latestFeature`, so any 11.0 feature band or patch is selected and nothing else. That makes AC-3's "an 11.0.x SDK selected" true by construction, and CI installs from this file (AD-47).
- **Root `Directory.Build.props`.** Holds settings shared by every project (AD-44):
  - `TargetFramework=net11.0`, for the libraries, both test projects, the sample and the benchmarks. No project sets its own framework, so AC-2 cannot drift.
  - No `LangVersion` (AD-43): on `net11.0` the compiler's default is C# 15, so all code is C# 15 (C-25) without a second setting that could disagree with the framework (C-27).
  - `TreatWarningsAsErrors=true`;
  - `NuGetAudit=true`, `NuGetAuditMode=all` and `NuGetAuditLevel=high`. A High or Critical advisory, transitive included, fails restore. That makes AC-45 a build property, not a script.
- **C# 15 (C-25, C-27, AD-43).** Every compilation is `net11.0`, so C# 15 features (for example union types or closed hierarchies) may be used anywhere, including the public API, when they suit the design. Nothing requires rewriting existing code into C# 15 constructs; C-25 sets the language version, and WI-1 proves it with a `#error version` probe.
- **`src/Directory.Build.props`.** Imports the root file and adds library-only settings:
  - package metadata (3.3.5) and `Version=1.0.0`;
  - `Nullable=enable` and `GenerateDocumentationFile=true`;
  - `AnalysisLevel=latest-recommended` and `AnalysisModeSecurity=All`;
  - `IncludeSymbols` with `SymbolPackageFormat=snupkg`;
  - `PackageLicenseExpression=Apache-2.0`, plus `LICENSE` packed as a file (AC-16), which keeps the SPDX metadata license scanners read.

  Tests and the sample keep default analysis, because test names such as `ctor_When…` would trip CA1707 and break AC-57.
- **`Directory.Packages.props`.** Central package management. The direct-dependency inventory for AC-49 and AC-134 lives in one file that Renovate reads natively. Alternative lost: per-project versions, which let `Microsoft.IdentityModel.*` drift between the library and the tests.
  - _Amended 2026-11-23 (AM-3/AM-4; AD-45 supersedes AD-35)._ Every package has one unconditional `PackageVersion` at its latest stable version, `Microsoft.AspNetCore.Authentication.JwtBearer` included (its 11.0.x line). There are no per-framework entries. (The Q-30 pass had given framework-bound dependencies one conditional entry per framework.)
- **No `.editorconfig` severity inventory.** C-17 and Fred's guidance rule it out; `AnalysisLevel` plus warnings-as-errors meets AC-55 to AC-57.

#### 3.3.5 Fixed strings and paths that the ACs defer to §3

- **Package metadata (both packages).**
  - `Authors` = `ValhallaTech`.
  - `PackageProjectUrl` and `RepositoryUrl` = `https://github.com/IronSloth/IdentityModel-Supa`.
  - `PackageReleaseNotes` = `https://github.com/IronSloth/IdentityModel-Supa/blob/master/CHANGELOG.md`.
  - `Copyright` = `Copyright 2026 (c) ValhallaTech. Portions Copyright 2022 (c) Scott Brady.` This contains the AC-17 string. No other element names Scott Brady (AC-11).
  - Descriptions: core = `EdDSA (Ed25519, Ed448) signing and validation for Microsoft.IdentityModel JWTs, plus strict Base16 and Base62 encoders.` AspNetCore = `ASP.NET Core Identity password rules: a passwordrules tag helper and an extended password validator.`
- **Icon (AC-12).** Fred supplied `assets/valhallatech-logo.png` on 2026-11-23 (U-1, complete): a 512×512 transparent PNG of the company emblem, not yet committed. WI-42 commits it byte for byte, packs it as `icon.png`, and deletes the upstream `icon.png` at the repository root. AC-12's "supplied logo" hash is the hash of that committed file. (Amended 2026-11-23 (AM-1/AM-2/Q-30).)
- **Modification notice (AC-15, AC-18).** Placed as the first two lines of every `.cs` file under `src/` and `samples/` that derives from a file at `555d31c`. The same header goes on test files and on the `perf/` baseline copies.
  - Line 1: `// Portions Copyright 2022 (c) Scott Brady, https://github.com/scottbrady91/IdentityModel, licensed under the Apache License, Version 2.0.`
  - Line 2: `// Modified by ValhallaTech: this file was changed from the original. See CHANGELOG.md.`
  - AC-18 searches for the string `Modified by ValhallaTech: this file was changed from the original.` AC-15's only permitted `scottbrady` match is Line 1.
- **Measurement report (AC-59).** _Amended 2026-11-23 (AM-1/AM-2/Q-30): 64 rows and 12 Ed448 rows on runtimes 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): one .NET 11.0.x runtime (AD-48). Originally 32 and 6 rows on one 10.0.x runtime._
  - Main report: `perf/results/1.0.0/report.csv`, with `report.md` as a readable copy. It holds exactly 2 rows (baseline and after, on one 11.0.x runtime) × 16 path-size pairs of T, so 32 rows. Each row has mean, error (BenchmarkDotNet's 99.9% confidence half-width), confidence level, allocated bytes, runtime version and machine name.
  - Companion files: the AC-123 Ed448 report in `report-ed448.csv`, exactly 6 rows (baseline and after for P-e, P-f and P-g); AC-69 in `base16-scaling.csv`; AC-70 in `base62-budget.csv`, which records `B`, the scale factor and both means.
  - The Ed448 rows are kept out of the main report so AC-59's exact row count holds.
- **Migration guide (AC-130). Withdrawn 2026-11-23 (AM-3/AM-4):** AC-130 is cut (CUT-3) and no guide is written (AD-51, AD-54). The list below is kept only so earlier references resolve; nothing checks it. It read: `docs/migration/scottbrady-4.1.0-to-1.0.0.md`. Each entry is a table row with non-empty **Before**, **After** and **Action** cells. The API phrase list, where an entry matches if it contains the phrase, case-insensitive:

  | AC-130 API | Phrase |
  | --- | --- |
  | 4.1.0 package reference | `ScottBrady.IdentityModel" Version="4.1.0"` or `ScottBrady.IdentityModel 4.1.0` |
  | Namespaces | both `ScottBrady.IdentityModel.Crypto` and `ScottBrady.IdentityModel.Tokens` |
  | `EdDsa.Create(EdDsaParameters)` | `EdDsa.Create(` |
  | `EdDsaParameters { D, X }` | `EdDsaParameters` |
  | `ExtendedSecurityAlgorithms.Curves.Ed25519` | `Curves.Ed25519` |
  | `new EdDsaSecurityKey(EdDsa) { KeyId = … }` | `new EdDsaSecurityKey(` |

- **CHANGELOG 1.0.0 phrase list (AC-104's 18 topics; AC-130 no longer uses it, since it is cut).** A topic counts as covered if its section contains any listed phrase, case-insensitive.

  | # | Topic | Phrases |
  | --- | --- | --- |
  | 1 | New package IDs | `ValhallaTech.Security.IdentityModel` |
  | 2 | Namespace rename | `namespace` |
  | 3 | .NET 11 as the only target (_amended 2026-11-23 (AM-1/AM-2/Q-30): ".NET 10 and .NET 11", with phrases `net10.0 and net11.0` and `.NET 10 and .NET 11`; amended 2026-11-23 (AM-3/AM-4): .NET 11 only. Originally ".NET 10 only", with `net10.0` and `.NET 10`_) | `net11.0`, `.NET 11` |
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

- **Doc-snippet harness (AC-102).** `build/Test-DocSnippets.ps1` extracts every fenced block tagged `csharp`, `cs` or `c#` from `README.md` and `src/*/README.md`. Each block is compiled in its own generated `net11.0` console project that references both released packages and sets no `LangVersion`, so C# 15 is allowed (AD-49, AD-43). _Amended 2026-11-23 (AM-1/AM-2/Q-30): a `net10.0` and a `net11.0` project. Amended 2026-11-23 (AM-3/AM-4): one `net11.0` project. Originally one `net10.0` project._ Placement:
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
  - **Out-of-process chains C14 to C19** _(added 2026-11-23 (Gate 2, third review); AD-59, AD-60)_. Their acceptance tests are Pester files under `build/tests/`, red by assertion at their tests item. Their stubs steps are WI-38 (C14), WI-44 (C17) and WI-62 (C19). C15, C16 and C18 have no stubs item, because GitHub's APIs exist at `555d31c`. Release checks, Fred-state checks and `1.x` record-only checks are named in §3.7 and exempt from the green check after implementation. These chains are not in AC-121's or AC-132's scope.
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
  - A tests item is done when every new test compiles and fails by assertion ("red for the right reason"). The exceptions are AC-121's regression guards, which pass: AC-24, AC-32, AC-40, AC-41, AC-72, AC-73, AC-77, AC-82, AC-90, AC-110, AC-112, AC-135 and AC-141. Two replacement tests with no AC ID are also guards (AC-121; Fred, 2026-10-26). C11 has the null-only replacement for `HasConsecutiveCharacters_WhenPasswordIsNullOrWhitespace_ExpectArgumentNullException`, written in WI-27. C12 has the `null`-only replacement for `ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxLength_ExpectNoMaxLengthAttribute`, written in WI-30. Each must exist before its chain's implementing commit (WI-28, WI-31) and pass at release; neither has to fail first.
- **Unchanged guardrails.** No test is weakened to make it pass. A finding in an item's output goes back to that item's owner.
- **Decisions.** _Added 2026-11-23._ Every item carries a `decisions:` line naming the §3.0 rows that bind it, or `none`. The chain structure above is AD-16, and the commit rules are AD-28.
- **One framework.** _Added 2026-11-23 (AM-1/AM-2/Q-30) as "Both frameworks"; amended 2026-11-23 (AM-3/AM-4)._ Every project targets `net11.0`, so every test runs once, on runtime 11.0.x, and "fails by assertion" and "passes" mean on that run. All code, tests included, is C# 15 (AD-43). The earlier rule for two framework runs is withdrawn with `net10.0`.

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
| `AspNetCore.Identity.ExtendedPasswordValidatorTests.HasConsecutiveCharacters_WhenPasswordIsNullOrWhitespace_ExpectArgumentNullException` | Null, empty and whitespace input throw | U10: only null throws now. Replaced by a null-only test (WI-27, a C11 regression guard under AC-121) plus the AC-108 test. |
| `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxLength_ExpectNoMaxLengthAttribute` | 0 and -1 silently ignored | GP-9 Q-17: they now throw (AC-142(b)). Replaced by a `null`-only test (WI-30, a C12 regression guard under AC-121) and the AC-142 tests. |
| `AspNetCore.TagHelpers.NewPasswordTagHelperTests.ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxConsecutiveChars_ExpectNoMaxConsecutiveCharsAttribute` | -1 silently ignored | Same |

Some kept tests lose theory rows or change assertions, without any name being removed:
- the 0 and -1 rows in `ValidateAsync_WhenExtendedPasswordOptionsAndPasswordIsNotTooLong_ExpectSuccess` and `…MaxConsecutiveCharactersValid_ExpectSuccess`;
- the 0 row in `ProcessIdentityPasswordRules_WhenExtendedOptionsWithMaxConsecutiveChars_ExpectMaxConsecutiveCharsAttribute`;
- the 32 → 256 `KeySize` assertions in `EdDsaSecurityKeyTests`;
- the byte → bit data in `LegalKeySizes_ExpectCorrectValues`;
- the reference-equality and `PublicKeyParameter.Should().BeNull()` assertions in `EdDsaCreationTests.FromParameters_When*PrivateKey_ExpectCorrectParameters`, which change because `Create` now copies the parameters and derives the public key.

The item making each change states it in its PR.

##### Phase 0 — Foundation (serial)

#### WI-1 — Establish the .NET 11 build foundation and move runtime and sample dependencies to latest stable

_Amended 2026-11-23 (AM-1/AM-2/Q-30): a `net10.0`/`net11.0` foundation with per-framework versions. Amended 2026-11-23 (AM-3/AM-4): `net11.0` only, C# 15 through the default language version, one latest stable version per package. Originally a ".NET 10 build foundation" with a 10.0 SDK pin and `--outdated` as the AC-49 check._

- **domain:** `backend`
- **depends-on:** WI-53 (its script is this item's AC-49 check, so it lands first; amended 2026-11-23 (Gate 2, F-1): the note "the two can be worked at the same time" contradicted this hard edge and is removed)
- **decisions:** AD-3, AD-21, AD-34, AD-43, AD-44, AD-45, AD-46
- **satisfies:** AC-1, AC-2 (sample), AC-45, AC-46, AC-47, AC-48, AC-49 (libraries and sample)
- **change:**
  - Add `global.json` (SDK `11.0.100`, `rollForward: latestFeature`) and the root `Directory.Build.props` (3.3.4) with `TargetFramework=net11.0` and no `LangVersion` (AD-43, AD-44).
  - Add `Directory.Packages.props` (CPM, one unconditional `PackageVersion` per package, AD-45) and remove every inline `Version` and every existing `TargetFrameworks`/`TargetFramework`/`LangVersion` from the project files.
  - **Libraries:** BouncyCastle.Cryptography to latest stable (2.7.x on 2026-10-25). Microsoft.IdentityModel.Tokens and .JsonWebTokens to latest stable 8.x. Add SimpleBase at latest stable. GP-2 chose the library; GP-10 removed the 5.6.4 pin, so it follows R-17 like every other dependency, and the AC-70 and AC-124 to AC-127 checks catch any regression in a newer version. Remove `SecurityCodeScan.VS2019` from both.
  - **Sample:** Microsoft.AspNetCore.* and EF Core InMemory to their latest stable (11.0.x). Remove `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` and its `AddRazorRuntimeCompilation()` call (AD-21).
  - Replace `ScottBrady.IdentityModel.sln` with an `.slnx` holding the same projects (renamed in WI-5).
  - Fix compile breaks in `src/` and `samples/` from the Microsoft.IdentityModel 7 → 8 move and from building on .NET 11. No behavior change is intended.
- **done when:**
  - `dotnet build -c Release` succeeds for each **library and the sample project**, built individually, with the 11.0 SDK that `global.json` selects. The whole solution is not expected to build yet: with NuGetAudit at high and warnings as errors, the test project's AutoFixture advisories fail its restore until WI-2 removes them, so AC-3 belongs to WI-2.
  - Each library pack's `lib/` folders are exactly `net11.0` (AC-1). `dotnet msbuild -getProperty:TargetFramework -getProperty:TargetFrameworks` returns `net11.0` for both libraries and the sample (AC-2, sample row).
  - **Language check (C-25, AD-43).** `rg -uu LangVersion -g "*.csproj" -g "*.props" -g "*.targets"` finds 0 matches. A throwaway `#error version` line, built in one library and removed before commit, reports language version 15 in its CS8304 message *(the `#error version` method is from Microsoft Learn, "Configure language version", read 2026-11-23)*. The output goes in the PR.
  - `--vulnerable` and `--deprecated --include-transitive` report nothing for the libraries and the sample (AC-45, AC-46). `build/Test-RequiredVersions.ps1` (WI-53) reports 0 mismatches for both libraries and the sample (AC-49). `rg -uu SecurityCodeScan` finds 0 matches (AC-47). Every direct license is an OSI-approved SPDX expression (AC-48).

#### WI-2 — Migrate the test project to .NET 11 and AwesomeAssertions

_Amended 2026-11-23 (AM-1/AM-2/Q-30): `net10.0` and `net11.0`, with JwtBearer per framework. Amended 2026-11-23 (AM-3/AM-4): `net11.0` only, one JwtBearer version. Originally `net10.0` only, a 10.0 SDK, xunit.v3 rejected "under the deadline", and `--outdated` as the AC-49 check._

- **domain:** `tests`
- **depends-on:** WI-1
- **decisions:** AD-1, AD-20, AD-43, AD-44, AD-45, AD-46
- **satisfies:** AC-2 (tests), AC-3, AC-4, AC-45, AC-46, AC-48, AC-49 (tests)
- **change:**
  - Retarget to `net11.0` by removing the project's own `TargetFrameworks` (the root props set it, 3.3.4) and its `LangVersion=latest` (AD-43).
  - **D-1 (AD-1):** replace FluentAssertions 6.12.2 with AwesomeAssertions at its latest stable (9.6.0 on 2026-10-25). Change `using FluentAssertions;` to `using AwesomeAssertions;` in the 19 files, and fix every 6→7 API rename the compiler reports across the 157 `.Should()` calls.
  - Remove **AutoFixture** (AD-20). It is used only in `AsymmetricAlgorithmTests.cs`, to build `PbeParameters`; replace that with `new PbeParameters(PbeEncryptionAlgorithm.Aes256Cbc, HashAlgorithmName.SHA256, 1)`. This removes both S8 High advisories.
  - Bump `xunit` (stays on the 2.9.x package ID, AD-20), `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk`, `Moq` and `System.IdentityModel.Tokens.Jwt` to latest stable. Add `coverlet.collector`.
  - `Microsoft.AspNetCore.Authentication.JwtBearer` to its latest stable (11.0.x), one unconditional entry (AD-45; amended 2026-11-23 (AM-3/AM-4): the per-framework entries are withdrawn).
  - Move `TestJwtBearerHandler` off the `ISystemClock` constructor if .NET 11 removed it.
- **done when:**
  - The full solution builds with `dotnet build -c Release` at the repository root, with the 11.0 SDK selected (AC-3).
  - `dotnet test -c Release` runs the test project on runtime 11.0.x, and all 250 tests pass, none skipped. The test-name set equals `555d31c`'s (AC-4, AC-5). No assertion is weakened. _Amended 2026-11-23 (Gate 2; R-2 as amended, F-6):_ no test is marked `1.x` yet, so R-2's exemption for marked tests does not apply here, and every test must pass.
  - `dotnet msbuild -getProperty:TargetFramework -getProperty:TargetFrameworks` on the test project returns `net11.0` (AC-2).
  - `--vulnerable` and `--deprecated --include-transitive` are clean for the test project, and `build/Test-RequiredVersions.ps1` (WI-53) reports 0 mismatches (AC-49).

#### WI-3 — Move CI to .NET 11 with least-privilege tokens

_Amended 2026-11-23 (AM-1/AM-2/Q-30): the 10.0 and 11.0 SDKs, with both runtimes in the done-when. Amended 2026-11-23 (AM-3/AM-4): the 11.0 SDK only. Originally the 10.0 SDK only._

- **domain:** `ci`
- **depends-on:** WI-1, WI-2, WI-58 (WI-58 added 2026-11-23 (Gate 2): this item implements chain C15)
- **decisions:** AD-12, AD-47, AD-60, AD-65
- **satisfies:** AC-6, AC-7, AC-58 (R-21 lands early because it is one block per file); AC-4's split run (amended 2026-11-23 (Gate 2))
- **change:**
  - `dotnetcore.yml`: top-level `permissions: contents: read`. `actions/setup-dotnet` with `global-json-file: global.json` and no `dotnet-version` (AD-47), so the 11.0 SDK and runtime 11.0.x are installed. Restore, build, test, pack and upload the artifact in one job on `runs-on: ubuntu-latest`. The label follows GitHub's current Ubuntu image (Fred, 2026-10-27: "Follow ubuntu-latest"). Remove the `matrix.os` strategy and the `if: matrix.os == 'ubuntu-latest'` conditions on the pack and upload steps.
  - **Two test steps (AD-58; amended 2026-11-23 (Gate 2): was one `dotnet test` step, whose single exit code would let a failing `1.x`-marked test fail the job, and with it AC-7 and AC-144).** Add `build/release-scope.json` with `oneXRequirements` and `oneXAcs` exactly as AD-58 lists them, and `build/Invoke-Tests.ps1`, which runs `dotnet test -c Release --no-build --logger "trx;LogFilePrefix=<scope>" --results-directory TestResults/<scope>` once per test project:
    - step **Test (blocking)** runs `-Scope Blocking` (filter `Release!=1.x`) and fails the job on any failure;
    - step **Test (1.x, recorded)** runs `-Scope OneX` (filter `Release=1.x`) with `if: ${{ !cancelled() }}`. It writes each failed test's name and requirement IDs to `$GITHUB_STEP_SUMMARY` under "1.x, recorded, not gating" and as a `::warning::` annotation, and exits 0. It exits non-zero only when a run did not complete (a build, host or abort error), because then nothing was recorded.

    Both steps upload `TestResults/` as one artifact. AC-4 and AC-7 read the recorded runtime and the per-scope results from those files.
  - `codeql-analysis.yml`: kept. The same `global-json-file` setup (AD-47); job permissions `contents: read`, `actions: read`, `security-events: write`.
  - **Network.** The test job needs outbound HTTPS to `registry.npmjs.org` (WI-36's `Category=Network` tests, AC-138 and AC-143) and to jsDelivr (LibMan restore on build, WI-37). GitHub-hosted runners have both. The job does not filter out `Category=Network` tests. A registry outage fails the run loudly, naming the URL, and the run is re-run, never muted.
  - **Resolves D-10 (Fred, 2026-10-27: "Ubuntu only (Recommended)"; CodeQL stays).**
    - **Ubuntu only.** The library's crypto is pure-C# BouncyCastle with no OS crypto, so OS-specific defects are unlikely.
    - **Windows coverage.** Windows is covered because every build and test also runs locally on ODIN01 (Windows).
    - **macOS dropped.** macOS was the slowest job to queue, and Gate 0 waits on every job.
    - **CodeQL** stays because its dataflow analysis finds things the in-build analyzers do not.
- **done when:** AC-6 and AC-58 pass: the `global.json` SDK each setup step names, and any `dotnet-version` line, match `^11\.0(\.x|\.\*|\.\d+)?$`. A branch run of the single `ubuntu-latest` job is green, and its TRX files show the library tests ran on runtime 11.0.x with 0 failed. _Amended 2026-11-23 (Gate 2, F-2): "and sample" is removed, because the sample test project does not exist until WI-34; WI-34's done-when now checks the sample TRX._ WI-46 re-checks AC-7 on the release commit, where "every job" is that one job.
  - **Split probe (AD-58; added 2026-11-23 (Gate 2)).** On a probe branch with one throwaway test marked `[Trait("Release", "1.x")]` and `[Trait("Requirements", "R-43")]` that fails by assertion, the job concludes `success`, and the summary lists that test as "1.x, recorded, not gating". With the marker removed from the same test, the job fails. On the same branch, the set of test methods in the blocking TRX files and the set in the `1.x` TRX files are disjoint, and together equal the set an unfiltered `dotnet test` run records. That proves `Release!=1.x` keeps every test that has no `Release` trait (RK-21). The probe branch is deleted afterwards.
  - **Chain C15 green (AD-60; added 2026-11-23 (Gate 2, third review)).** WI-58's `CiRun.Tests.ps1`, run with `-Sha` set to this item's branch commit, passes without edits.

#### WI-4 — Write the failing identity and API-shape tests (chain C1)

- **domain:** `tests`
- **depends-on:** WI-2 (C1 precondition; no new surface)
- **decisions:** AD-16
- **satisfies:** AC-13, AC-14, AC-95, AC-96
- **change:** Add `PublicApiShapeTests`, reflecting over `typeof(Base62).Assembly` and `typeof(NewPasswordTagHelper).Assembly`: exported-type namespaces, `GetName().Name`, assembly-attribute strings including `InternalsVisibleTo`, removed constant names, and `"OKP"` constant names. Unit tests per assertion.
- **done when:** The tests compile at the pre-rename state and fail by assertion.

#### WI-5 — Rename to the ValhallaTech identity and remove the leftover constants (implements C1; stubs step for C3, C5–C10, C13)

- **domain:** `backend`
- **depends-on:** WI-4
- **decisions:** AD-16, AD-30, AD-54
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
- **decisions:** AD-30, AD-54
- **satisfies:** AC-4, AC-5 (name mapping)
- **change:** `git mv` the test project to `test/ValhallaTech.Security.IdentityModel.Tests`, rename namespaces and usings, and add the 3.3.5 notice to derived test files. The WI-4 tests change only their `using` lines.
- **done when:** The suite is green, and the test-name set equals `555d31c`'s under the 3.3.1 mapping.

#### WI-7 — Add CSharpier and reformat once

- **domain:** `ci`
- **depends-on:** WI-6
- **decisions:** AD-7, AD-30, AD-65
- **satisfies:** AC-120 (R-49 is 1.x, but it lands in Phase 0 so the one-time reformat cannot conflict with parallel lanes; RK-9). _Amended 2026-11-23 (Gate 2, fourth review, F-1; AD-65): this line said WI-7, as a `1.x`-only item, was "not a prerequisite of WI-55 or WI-46". That was false, because WI-8 and WI-39 depend on it._ WI-7 **gates 1.0.0 as sequencing.** Its reformat and `quality.yml` come before WI-8 and WI-39, so that the reformat never runs beside another repository-wide change (RK-9). Only AC-120's pass-or-fail result is record-only at 1.0.0.
- **change:**
  - **Resolves D-7, formatting half: CSharpier**, Fred's preference. It is zero-configuration, deterministic, and one dotnet local tool that Renovate keeps current. `dotnet format whitespace` lost because it is style-configurable through `.editorconfig`, which invites the analyzer-inventory sprawl C-17 rules out.
  - Add `.config/dotnet-tools.json` with `csharpier` at latest stable, and a default `.csharpierrc.json`.
  - One `style:` commit containing only `dotnet csharpier format .` output.
  - Add a new workflow, `.github/workflows/quality.yml`, for the CI checks of `1.x` requirements (AD-58, which superseded AD-56; amended 2026-11-23 (Gate 2, F-1): this step was on the `dotnetcore.yml` job, which is a 1.0.0 gate run, so a formatting failure would have held 1.0.0 back against §2.2.1). It runs on the same triggers as `dotnetcore.yml`, on `ubuntu-latest`, with `permissions: contents: read` and `actions/setup-dotnet` from `global-json-file: global.json` (AC-6, AC-58). Its first step is `dotnet tool restore && dotnet csharpier check .`. It is not one of AC-144's gate runs, and it is never added to `verify-gates`.
- **done when:** _Amended 2026-11-23 (Gate 2, fourth review, F-1; AD-65): AC-120 no longer decides when this item is done._
  - **Done, and WI-8 and WI-39 may start,** when the reformat commit and `quality.yml` have merged, and build and test results are unchanged by the reformat commit. This holds whatever AC-120 returns.
  - **AC-120 is recorded either way.** The check should pass on the branch and fail on a one-line probe. If it does not, the result is recorded with the run, and the miss goes back to this item as a finding, fixed in its own change before WI-66. It never holds WI-8, WI-39 or 1.0.0.
  - **Fallback.** If CSharpier cannot format the code at all, for example because it does not parse C# 15 syntax, merge `quality.yml` with its check step and no reformat commit, and record AC-120 as failed. WI-8 and WI-39 then start. The one reformat commit is made in Phase 4, before WI-65, when no other lane is open (RK-9).

#### WI-8 — Write the failing nullability test (chain C2)

- **domain:** `tests`
- **depends-on:** WI-7 (done under AD-65 once its reformat and `quality.yml` have merged, whatever AC-120 returns; amended 2026-11-23 (Gate 2, fourth review))
- **decisions:** AD-16, AD-65
- **satisfies:** AC-97
- **change:** A `NullabilityInfoContext` sweep over every public parameter, return, property and field of reference type in both assemblies.
- **done when:** It fails by assertion, because nullability is `Unknown` with annotations off.

#### WI-9 — Turn on analysis, nullable annotations and XML docs across the existing surface (implements C2)

- **domain:** `backend`
- **depends-on:** WI-8
- **decisions:** AD-22, AD-43
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
- **decisions:** none
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
- **decisions:** AD-14, AD-16
- **satisfies:** AC-21, AC-22, AC-23, AC-24, AC-25, AC-32, AC-81
- **change:**
  - Tests in an xUnit collection `GlobalCryptoState` (`DisableParallelization = true`), with a fixture that saves and restores `CryptoProviderFactory.Default.CustomCryptoProvider`.
  - Add a unit test that two keys never share a factory instance.
  - AC-24 and AC-32 are regression guards.
- **done when:** AC-21, 22, 23, 25 and 81 fail by assertion; AC-24 and AC-32 pass.

#### WI-12 — Isolate each key's crypto provider (implements C3)

- **domain:** `backend`
- **depends-on:** WI-11
- **decisions:** AD-14
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
- **decisions:** AD-4, AD-16
- **satisfies:** AC-28 (surface only)
- **change:** Add `ExtendedJsonWebKeyConverter.ConvertFromEdDsaSecurityKeyIncludingPrivateKey(EdDsaSecurityKey key)`, documented, returning `new JsonWebKey()`. It does not throw (**resolves D-4**, 3.3.2).
- **done when:** It builds, and the signature matches 3.3.2.

#### WI-14 — Write the failing JWK tests (chain C4)

- **domain:** `tests`
- **depends-on:** WI-13
- **decisions:** AD-4
- **satisfies:** AC-26, AC-27, AC-28, AC-29, AC-82, AC-93, AC-94
- **change:** Acceptance and unit tests against the converter and `JsonWebTokenHandler`. AC-82 is a guard.
- **done when:** AC-26, 27, 28, 29, 93 and 94 fail by assertion; AC-82 passes.

#### WI-15 — Make JWK export public-only and carry the key ID (implements C4)

- **domain:** `backend`
- **depends-on:** WI-14, WI-12, WI-17
- **decisions:** AD-4, AD-14, AD-19
- **satisfies:** AC-26, AC-27, AC-28, AC-29, AC-82, AC-93, AC-94
- **change:**
  - The default conversion emits `X`, plus `D` only through the opt-in. `X` comes from the library-held public key, so private-only keys export their derived public key after WI-17.
  - `Kid` is copied both ways, `CryptoProviderFactory = key.CryptoProviderFactory`, and a null `key` throws.
- **done when:** WI-14 is green.

#### WI-16 — Write the failing key-material tests (chain C5)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5), WI-56 (amended 2026-11-23 (Gate 2))
- **decisions:** AD-5, AD-16, AD-19, AD-65
- **satisfies:** AC-30, AC-31, AC-42, AC-43, AC-44, AC-73, AC-87, AC-88, AC-91, AC-92, AC-128, AC-129, AC-135
- **change:**
  - Acceptance and unit tests.
  - AC-30 and AC-31 use the reflection buffer walk, skipping BouncyCastle-assembly types.
  - AC-42 uses the full combination grid with a recorded seed.
  - AC-44 asserts rejection (D-5).
  - AC-73 and AC-135 are guards.
  - **`1.x` marker (AD-58; added 2026-11-23 (Gate 2)).** The AC-87 and AC-88 tests verify only R-32, which ships in `1.x`. Each carries `[Trait("Release", "1.x")]` and `[Trait("Requirements", "R-32")]`. No other test of this item is marked.
- **done when:** Every non-guard test fails by assertion; the guards pass. The AC-87 and AC-88 tests are the only marked tests, and WI-56's misuse check passes (amended 2026-11-23 (Gate 2)).

#### WI-17 — Copy, validate, size, zero and import keys correctly (implements C5)

- **domain:** `backend`
- **depends-on:** WI-16
- **decisions:** AD-5, AD-19
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
- **decisions:** AD-16
- **satisfies:** AC-84, AC-85, AC-89, AC-90
- **change:** Acceptance and unit tests, including the RFC 8032 §7.1 TEST 1 vector for AC-85. AC-90 is a guard.
- **done when:** AC-84, 85 and 89 fail by assertion; AC-90 passes.

#### WI-19 — Validate offsets and sign or verify without copies (implements C6)

- **domain:** `backend`
- **depends-on:** WI-18, WI-17 (both edit `EdDsa.cs`)
- **decisions:** none
- **satisfies:** AC-84, AC-85, AC-89, AC-90. Also the P-e, P-f and P-g improvements measured by WI-43 (AC-64 to AC-66, AC-68).
- **change:**
  - The offset `Verify` validates every argument with `ParamName`, accepts a zero-length message, and feeds BouncyCastle through `BlockUpdate(ReadOnlySpan<byte>)` with no `Skip`/`Take`/`ToArray`.
  - In the signature provider, span `Sign` returns `false`/0 on a short destination and signs from the span. Offset `Sign` validates its arguments and calls `BlockUpdate(input, offset, count)` with no copy.
- **done when:** WI-18 is green, and the suite is green.

##### Phase 1 — Lane B: encoders and streams

#### WI-20 — Write the failing Base62 tests (chain C7)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **decisions:** AD-3, AD-16
- **satisfies:** AC-33, AC-34, AC-35, AC-36, AC-79, AC-124, AC-125, AC-126
- **change:**
  - Acceptance and unit tests with fixed seeds.
  - Run AC-124's 1 MiB case on a task with a bounded wait, so the `555d31c` behavior (minutes) fails by assertion instead of hanging the run.
- **done when:** All fail by assertion.

#### WI-21 — Replace Base62 with capped SimpleBase (implements C7)

- **domain:** `backend`
- **depends-on:** WI-20
- **decisions:** AD-3
- **satisfies:** AC-33, AC-34, AC-35, AC-36, AC-79, AC-124, AC-125, AC-126, AC-136. Feeds AC-60, AC-61, AC-70 and AC-127.
- **change:**
  - Implement the 3.3.2 `Encode`/`Decode` rows on `SimpleBase.Base62.Default` at its latest stable version (**D-3 as fixed by GP-2; version per GP-10**).
  - XML docs state the caps (`1,024` bytes; `1,376` characters) and `ArgumentException`/`FormatException`.
  - Keep `CharacterSet`.
- **done when:** WI-20 is green. AC-136 holds in the built XML doc file.

#### WI-22 — Write the failing Base16 tests (chain C8)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **decisions:** AD-16, AD-17
- **satisfies:** AC-37, AC-38, AC-39, AC-40, AC-41
- **change:** Acceptance and unit tests. AC-40 and AC-41 are guards.
- **done when:** AC-37, 38 and 39 fail by assertion; AC-40 and AC-41 pass.

#### WI-23 — Re-implement Base16 on the runtime's hex APIs (implements C8)

- **domain:** `backend`
- **depends-on:** WI-22
- **decisions:** AD-17
- **satisfies:** AC-37 to AC-41. Feeds AC-62, AC-63 and AC-69.
- **change:** `Convert.ToHexStringLower` and `Convert.FromHexString`, with the null guard. Delete the lookup table.
- **done when:** WI-22 is green.

#### WI-24 — Write the failing stream-read tests (chain C9)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5)
- **decisions:** AD-16, AD-18
- **satisfies:** AC-71, AC-72, AC-83, AC-86
- **change:** Use a test stream that returns at most 1 byte per `Read`. AC-72 is a guard.
- **done when:** AC-71, 83 and 86 fail by assertion; AC-72 passes.

#### WI-25 — Read streams until the requested length arrives (implements C9)

- **domain:** `backend`
- **depends-on:** WI-24
- **decisions:** AD-18
- **satisfies:** AC-71, AC-72, AC-83, AC-86
- **change:** The 3.3.2 `TryRead` row.
- **done when:** WI-24 is green, and the existing `TryRead` tests are unchanged and green.

##### Phase 1 — Lane C: password validation (AspNetCore)

#### WI-26 — Stub the error describer and the validator constructor (chain C11)

- **domain:** `backend`
- **depends-on:** WI-9
- **decisions:** AD-13, AD-16
- **satisfies:** AC-111 (surface only)
- **change:**
  - Add `ExtendedIdentityErrorDescriber` with the two virtual methods, each returning `new IdentityError()`.
  - Add the `ExtendedPasswordValidator(IdentityErrorDescriber? errors = null)` ctor; it stores the value and does not use it.
  - **Resolves D-11** (3.3.2).
- **done when:** It builds, and the surface matches 3.3.2.

#### WI-27 — Write the failing validator and message tests (chain C11)

- **domain:** `tests`
- **depends-on:** WI-26, WI-56 (amended 2026-11-23 (Gate 2))
- **decisions:** AD-13, AD-65
- **satisfies:** AC-106, AC-107, AC-108, AC-109, AC-110, AC-111
- **change:**
  - Acceptance and unit tests.
  - **`1.x` marker (AD-58; added 2026-11-23 (Gate 2)).** The AC-106, AC-107 and AC-108 tests verify only R-43, which ships in `1.x`. Each carries `[Trait("Release", "1.x")]` and `[Trait("Requirements", "R-43")]`. The AC-109, AC-110 and AC-111 tests and the C11 replacement test are not marked. So at 1.0.0, WI-28's AC-132 test is the AC-109 unit test (U-6).
  - AC-111 registers a French describer subclass through `AddErrorDescriber<T>()`.
  - Also writes the null-only replacement for the removed `HasConsecutiveCharacters_WhenPasswordIsNullOrWhitespace_ExpectArgumentNullException`.
  - Regression guards (AC-121): AC-110, and that C11 replacement test (`null` already throws at `555d31c`).
- **done when:** AC-106 to AC-109 and AC-111 fail by assertion. AC-110 and the C11 replacement test pass. Only the AC-106 to AC-108 tests are marked, and WI-56's misuse check passes (amended 2026-11-23 (Gate 2)).

#### WI-28 — Use the describer, fix whitespace, scan linearly (implements C11)

- **domain:** `backend`
- **depends-on:** WI-27
- **decisions:** AD-13
- **satisfies:** AC-106 to AC-111. Feeds AC-67 (P-h).
- **change:** The 3.3.2 validator, describer, default-message and `HasConsecutiveCharacters` rows. The `Regex` is replaced by a single pass.
- **done when:** WI-27 is green, and the suite is green.

#### WI-29 — Stub the one-step registration (chain C12)

- **domain:** `backend`
- **depends-on:** WI-9
- **decisions:** AD-15, AD-16
- **satisfies:** AC-74 (surface only)
- **change:** Add `ExtendedPasswordIdentityBuilderExtensions.AddExtendedPasswordRules(this IdentityBuilder, Action<ExtendedPasswordOptions>)`, documented, returning `builder` and registering nothing.
- **done when:** It builds, and the surface matches 3.3.2.

#### WI-30 — Write the failing registration and options-validation tests, plus the DryIoc test (chain C12)

- **domain:** `tests`
- **depends-on:** WI-29
- **decisions:** AD-10, AD-15
- **satisfies:** AC-74, AC-75, AC-76, AC-77, AC-142; AC-132 for WI-31 (the 10 unit tests below)
- **change:**
  - Acceptance tests, plus unit tests for each options rule and message.
  - **WI-31's test-first unit tests (AC-132 as amended, F-3; added 2026-11-23 (Gate 2)).** A test class `ExtendedPasswordOptionsFirstUseTests`, with 10 unit tests: for each of AC-76's five invalid configurations (`MaxLength = 0`, `MaxLength = -1`, `MaxConsecutiveChars = 0`, `MaxConsecutiveChars = -1`, and `RequiredLength = 12` with `MaxLength = 10`), with no host and no service provider:
    - one test constructs `new ExtendedPasswordValidator<IdentityUser>()` (the 3.3.2 constructor, `IdentityErrorDescriber? errors = null`, which takes no options) and awaits `ValidateAsync` on `"Valid-Password-1"` with a `UserManager<IdentityUser>` test double whose `Options.Password` is an `ExtendedPasswordOptions` holding that configuration. That is how the validator receives options at `555d31c` and under 3.3.2;
    - one test constructs `new NewPasswordTagHelper(htmlGenerator, Options.Create(new IdentityOptions { Password = <that configuration> }))` (the 3.3.2 constructor) and calls `Process` on a `newpassword` element.

    Each asserts that an exception is thrown whose message (or `InnerException` chain) contains AC-76's listed names, ordinal. Each test carries `[Trait("FixedBy", "WI-31")]`, its method name ends `_ExpectRejectedAtFirstUse`, and the commit that adds them is `test(WI-30): … [AC-132 → WI-31]`, so U-6 can find all 10 by trait, name or commit.
  - **One DryIoc test (D-8, Fred, 2026-10-25: "Yes, sample + one test").** It builds the AC-74 `ServiceCollection` (`AddIdentityCore<IdentityUser>()` plus the single step), creates the provider through `DryIoc.Microsoft.DependencyInjection`'s `DryIocServiceProviderFactory`, and asserts that the resolved `UserManager<IdentityUser>` reports `PasswordTooLong` for a 65-character password.
  - The test project takes `DryIoc.Microsoft.DependencyInjection` **6.2.0** and `DryIoc.dll` **5.4.3** as test-only references; the libraries take no DryIoc dependency.
  - Also writes the `null`-only replacement for the removed `ProcessIdentityPasswordRules_WhenExtendedOptionsWithInvalidMaxLength_ExpectNoMaxLengthAttribute`.
  - Regression guards (AC-121): AC-77, and that C12 replacement test (`null` already adds no `maxlength` attribute at `555d31c`).
- **done when:** AC-74, 75, 76, 142, the DryIoc test and the 10 `ExtendedPasswordOptionsFirstUseTests` fail by assertion, because the settings are silently accepted (amended 2026-11-23 (Gate 2, F-3)). AC-77 and the C12 replacement test pass.

#### WI-31 — Implement the single step and reject invalid options on both paths (implements C12)

- **domain:** `backend`
- **depends-on:** WI-30, WI-28 (both edit `ExtendedPasswordValidator.cs`)
- **decisions:** AD-15
- **satisfies:** AC-74, AC-75, AC-76, AC-77, AC-142
- **change:**
  - Implement the 3.3.2 `AddExtendedPasswordRules` and `ExtendedPasswordOptions` rule rows.
  - One internal rule evaluator feeds three consumers: the `IValidateOptions<IdentityOptions>` with `ValidateOnStart`, the validator's first-use check, and `NewPasswordTagHelper.Process`. Each throws `OptionsValidationException` naming the setting.
- **done when:** WI-30 is green, including the DryIoc test and the 10 `ExtendedPasswordOptionsFirstUseTests`, which are this item's AC-132 tests (amended 2026-11-23 (Gate 2, F-3)).

##### Phase 1 — Lane D: null-argument sweep

#### WI-32 — Write the failing null-guard sweep (chain C10)

- **domain:** `tests`
- **depends-on:** WI-9 (annotations decide which parameters are swept; stubs step: WI-5)
- **decisions:** AD-16
- **satisfies:** AC-78, AC-80
- **change:**
  - A reflection-driven sweep over every public ctor and method of both assemblies (not `Try*`), with a fixture-value table per parameter type. A parameter type with no fixture **fails** with "no fixture for type X"; it is never skipped, so members added by later items are swept.
  - Add the AC-80 test, and the replacement for the removed null-`.Value` test.
- **done when:** The sweep fails by assertion, for example on `Base62.Encode`'s `ParamName` and `ConvertFromEdDsaSecurityKey(null)`. AC-80 fails.

#### WI-33 — Close the remaining null guards (implements C10)

- **domain:** `backend`
- **depends-on:** WI-32, WI-12, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-28, WI-31
- **decisions:** none
- **satisfies:** AC-78, AC-80
- **change:**
  - `ArgumentNullException.ThrowIfNull` wherever the sweep still fails after the lanes, including `ByteArrayExtensions.Combine`.
  - The `NewPasswordTagHelper` ctor guards from 3.3.2.
- **done when:** WI-32 is green, and the full suite is green.

##### Phase 1 — Lane E: sample

#### WI-34 — Create the sample test project and write the failing hosting tests (chain C13a)

- **domain:** `tests`
- **depends-on:** WI-9 (stubs step: WI-5; the routes exist), WI-56 (amended 2026-11-23 (Gate 2))
- **decisions:** AD-16, AD-29, AD-34, AD-44, AD-65
- **satisfies:** AC-52, AC-53, AC-54, AC-112, AC-2 (sample test row)
- **change:**
  - **`1.x` marker (AD-58; added 2026-11-23 (Gate 2)).** Every test this item writes is `1.x`-only: AC-52 to AC-54 verify R-19, and AC-112 verifies R-45. Each AC-52 to AC-54 test carries `[Trait("Release", "1.x")]` and `[Trait("Requirements", "R-19")]`, and each AC-112 test `[Trait("Release", "1.x")]` and `[Trait("Requirements", "R-45")]`. The project sits under `test/`, so it compiles WI-56's `ReleaseScopeTraitTests` through `test/Directory.Build.props`, unmarked.
  - New `test/ValhallaTech.Security.IdentityModel.Samples.Tests`, on `net11.0` through the root props (AD-34, AD-44; amended 2026-11-23 (AM-3/AM-4): it no longer sets its own framework), using `Microsoft.AspNetCore.Mvc.Testing` at its latest stable and `WebApplicationFactory<Program>`, added to the `.slnx`. C# 15 like all code (AD-43).
  - AC-53 and AC-54 run in a non-parallel collection that resets `IdentityModelEventSource.ShowPII`.
  - AC-54's probe is an `IStartupFilter` that appends a terminal middleware throwing `InvalidOperationException("ac54-probe")`, after the app's own pipeline.
- **done when:** AC-52, 53 and 54 fail by assertion on `net11.0`. AC-112, a regression guard (AC-121), passes. `dotnet msbuild -getProperty:TargetFramework -getProperty:TargetFrameworks` on the new project returns `net11.0` (AC-2). The CI run on this item's branch uploads TRX files for the sample test project whose recorded runtime is 11.0.x, with exactly the failures named above (amended 2026-11-23 (Gate 2, F-2): this clause moved here from WI-3). Those failures appear in the `1.x` TRX file and the job summary, not in the blocking TRX file. The blocking run of this project, `ReleaseScopeTraitTests` included, passes, so the job concludes `success` (amended 2026-11-23 (Gate 2), AD-58).

#### WI-35 — Move the sample to minimal hosting and DryIoc, and fix its key and diagnostics (implements C13a)

- **domain:** `backend`
- **depends-on:** WI-34, WI-31
- **decisions:** AD-10, AD-15, AD-34, AD-65
- **satisfies:** AC-52, AC-53, AC-54, AC-112, AC-113, AC-122 (sample half)
- **`1.x`-only (AD-65, which superseded AD-58; added 2026-11-23 (Gate 2)).** No 1.0.0 item depends on this one (checked 2026-11-23 (Gate 2, fourth review): only WI-48 and WI-65, both `1.x`, name it). Every AC this item satisfies is `1.x`-only, so it is not a prerequisite of WI-55 or WI-46. It is still scheduled in Phase 1. If it has landed by the release commit, its results are recorded with the release evidence; if not, it lands after 1.0.0 and is audited then.
- **change:**
  - `Program.cs` uses `WebApplication.CreateBuilder` and ends with `public partial class Program;`. Delete `Startup.cs`.
  - **D-8, DryIoc part (Fred, 2026-10-25):** `builder.Host.UseServiceProviderFactory(new DryIocServiceProviderFactory())`, with every registration still made on `IServiceCollection`.
    - References: `DryIoc.Microsoft.DependencyInjection` **6.2.0** and `DryIoc.dll` **5.4.3**. Both are their packages' latest stable; `8.0.0-preview-04` and `6.0.0-preview-09` are excluded because G-9 says "stable".
    - `DryIoc.dll` is referenced directly so the container in use is its latest stable, not the adapter's declared minimum. AC-49 and Renovate then govern it.
    - If either fails on `net11.0` (amended 2026-11-23: was `net10.0`; `net11.0` is now every project's framework, AM-3), that is reported to Fred; no preview is substituted.
  - `SampleOptions` holds a signing key (`D` and `X`) and a separate validation key built from `X` only.
  - `ShowPII` and the developer exception page apply only when `IsDevelopment()`. Otherwise use `AddProblemDetails()`, `UseExceptionHandler()` and `UseHsts()`.
  - Password setup uses `AddExtendedPasswordRules` (R-51).
- **done when:** WI-34 is green: its `1.x` TRX file shows 0 failed (amended 2026-11-23 (Gate 2)). AC-113's search finds `WebApplication.CreateBuilder` and no `Startup`. AC-122's sample search is clean.

#### WI-36 — Write the failing sample-page tests (chain C13b)

- **domain:** `tests`
- **depends-on:** WI-34
- **decisions:** AD-9, AD-16, AD-34, AD-65
- **satisfies:** AC-114, AC-138, AC-141, AC-143
- **change:**
  - **`1.x` marker (AD-58; added 2026-11-23 (Gate 2)).** The AC-114 tests verify only R-45, so each carries `[Trait("Release", "1.x")]` and `[Trait("Requirements", "R-45")]`. The AC-138, AC-141 and AC-143 tests verify `1.0.0` requirements and are not marked. A test that asserts both AC-114 and a `1.0.0` AC is split in two, so that the `1.0.0` assertion blocks.
  - HTML-parsing tests over GET `/` and GET `/Home/PasswordRules`, fetching every `<script src>` and `<link href>` through the same client.
  - Set V for AC-138 and AC-143 is built at test time. Read the library name and version from the sample's `libman.json`, download that version's tarball from `https://registry.npmjs.org/<name>/-/<name>-<version>.tgz`, and hash its files. Keying on the manifest means a Renovate bump needs no test edit. While `libman.json` does not exist (the introducing commit), the test fails on an assertion that the manifest declares `bootstrap`, never on an I/O exception. AC-140's release-date check confirms the version is npm `latest`.
  - **Network (AC conventions, AC-138's rule).** These tests need HTTPS access to `registry.npmjs.org`, and carry the xUnit trait `Category=Network`. If the metadata or tarball request fails, the test fails with "npm registry unreachable" and the failing URL. It never passes, skips, or reports inconclusive. A cached tarball counts only if its SHA-256 matches the registry's published `dist.integrity` for that version.
  - **Named test file (AC-137's closed exclusion).** Every AC-138 and AC-143 assertion that names the forbidden library lives in exactly one file, `test/ValhallaTech.Security.IdentityModel.Samples.Tests/ForbiddenClientLibraryTests.cs`. This is the path AC-137 excludes. The file only names the library as the thing forbidden. It contains no library banner, and loads or imports nothing (AC-137's separate check on it). No second file of that name exists.
  - AC-141 is a guard.
- **done when:** AC-114, AC-138 and AC-143 fail by assertion: the baseline serves the old library file and Bootstrap 4.3.1, whose hash is not 5.x's. AC-141 passes. Only the AC-114 tests are marked, and WI-56's misuse check passes (amended 2026-11-23 (Gate 2)).

#### WI-37 — Move the sample UI to Bootstrap 5, CSS and JS bundle, through LibMan, with no jQuery (implements C13b)

- **domain:** `frontend`
- **depends-on:** WI-36
- **decisions:** AD-9
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

_Amended 2026-11-23 (AM-1/AM-2/Q-30): a `net10.0`/`net11.0` harness run per runtime. Amended 2026-11-23 (AM-3/AM-4): `net11.0` only. Originally a `net10.0`-only harness._

- **domain:** `tests`
- **depends-on:** WI-9
- **decisions:** AD-6, AD-48, AD-60
- **satisfies:** AC-59, AC-69, AC-70, AC-123 (harness only; numbers come from WI-43). Stubs step of chain C14 (added 2026-11-23 (Gate 2, third review)).
- **change:**
  - **D-6 (AD-6): BenchmarkDotNet** (latest stable, MIT). It reports mean, error as a 99.9% confidence half-width, and allocations natively.
  - `perf/ValhallaTech.Security.IdentityModel.Benchmarks` (`net11.0` through the root props, `IsPackable=false`, in the `.slnx`). It runs with `dotnet run -c Release` on one .NET 11.0.x runtime (AD-48), and records the runtime version in every row. If BenchmarkDotNet's latest stable does not recognize `net11.0` as a runtime, the harness uses its in-process toolchain (RK-12). `Baseline/` holds verbatim `555d31c` copies of `Base62`, `Base16`, the offset-`Verify` path, the provider's span and offset `Sign` paths, and the consecutive-character check. Each copy carries the 3.3.5 notice and the source commit.
  - Benchmarks: every T path and size; Ed448 P-e/f/g; AC-69's 8 and 64 KiB inputs; AC-70's calibration `B`. Use `[MemoryDiagnoser]`, CSV and GitHub-markdown exporters, and a machine-name column.
  - Why a source copy: baseline and after then run in one process, on one runtime and one dependency graph, so only the code differs, which is the point of the measurement rules. Alternatives lost:
    - referencing upstream 4.1.0 from nuget.org: it pulls Microsoft.IdentityModel 7.x into a process that loads 8.x, and is not provably `555d31c`;
    - a `git worktree` build: two processes, two dependency graphs.
  - **C14 stubs (AD-60; added 2026-11-23 (Gate 2, third review)).** Commit the four 3.3.5 report files under `perf/results/1.0.0/`: `report.csv`, `report-ed448.csv`, `base16-scaling.csv` and `base62-budget.csv`. Each holds its header row only, with the 3.3.5 columns and no data rows. This is the neutral default WI-57's tests read; WI-43 replaces the files with measured rows.
- **done when:** The project builds in CI, and `--list flat` enumerates all 16 T pairs plus the Ed448, scaling and calibration cases. The four header-only files exist with the 3.3.5 columns.

#### WI-39 — Configure Renovate for every direct dependency

- **domain:** `ci`
- **depends-on:** WI-3, WI-7, WI-37, WI-61 (WI-61 added 2026-11-23 (Gate 2, third review): this item implements chain C18; the WI-7 edge is met once WI-7's reformat and `quality.yml` have merged, whatever AC-120 returns, AD-65, amended 2026-11-23 (Gate 2, fourth review))
- **decisions:** AD-8, AD-9, AD-45, AD-60, AD-62
- **satisfies:** AC-134, AC-118 (with U-4), AC-119 (with U-3)
- **change:**
  - **D-7, dependency half (AD-8): Renovate**, Fred's lean and the tool assetvault-io already uses. _(Amended 2026-11-23 (AM-3/AM-4): the two assetvault-io mentions in this item are history, not reasons; the decision rests on Fred's lean and the deciding fact below, NG-12.)_ The deciding fact: neither Renovate nor Dependabot has a LibMan manager, but only Renovate offers a custom regex manager that can be pointed at `libman.json`. Without it, AC-134 fails.
  - `renovate.json` extends `config:recommended`, `:dependencyDashboard` and `:semanticCommits`, with `minimumReleaseAge: "7 days"` (as in assetvault-io) and `vulnerabilityAlerts.enabled: true`.
  - Groups: `Microsoft.IdentityModel.*` with `System.IdentityModel.*`; `Microsoft.AspNetCore.*` with `Microsoft.EntityFrameworkCore.*`.
  - Rule: `bootstrap` gets `allowedVersions: "<6.0.0"` (GP-8).
  - A `customManagers` entry with `customType: "regex"`, matched to `samples/**/libman.json` through the file-pattern key of the Renovate schema current at implementation. Its match string captures `depName` and `currentValue` from `"library": "<name>@<version>"`, with `datasourceTemplate: "npm"`. A Renovate PR then bumps `libman.json`; the build restores the new files; and WI-36's hash tests re-verify them against the new tarball in the same PR's CI.
  - Native managers cover `Directory.Packages.props`, `global.json`, `.config/dotnet-tools.json` and the workflow `uses:`.
  - **SDK line (AD-45).** _Amended 2026-11-23 (AM-3/AM-4): the per-framework `matchCurrentValue`/`allowedVersions` rules added in the Q-30 pass are withdrawn; every NuGet package follows its latest stable version (R-48 as amended)._ One rule remains: `dotnet-sdk` gets `allowedVersions: "<12.0.0"`, so a `global.json` bump can never break AC-6. A NuGet bump whose new version does not support `net11.0` fails its own PR's restore, and is not merged; it is an A-14 case for Fred (RK-20).
- **done when:** The config validates (`renovate-config-validator`). Once U-4 is done, the dependency dashboard lists every item of AC-134's inventory, including `bootstrap` under the regex manager.
  - **Chain C18 green (AD-60; added 2026-11-23 (Gate 2, third review)).** Once this item has merged and U-4 is done, WI-61's AC-118 test passes without edits. Until U-4 is done, that test is a Fred-state check and waits for it. The AC-20 and AC-119 tests are Fred-state checks for U-2 and U-3. This item cannot turn them green, and they must pass at WI-46 step 1.

##### Phase 2 — Release (1.0.0)

#### WI-40 — Write `CHANGELOG.md` with the 1.0.0 breaking changes

- **domain:** `docs`
- **depends-on:** WI-33
- **decisions:** AD-51, AD-54, AD-57
- **satisfies:** AC-104, AC-105
- **change:**
  - Keep a Changelog format, with `## [1.0.0] - <yyyy-mm-dd>`, where the date is the date of this item's own commit (AD-57; amended 2026-11-23 (Gate 2, F-4): was "`<release date, ISO>`", which no item set). WI-55 sets the final date on the release commit.
  - The target-framework entry says the packages target `net11.0` only and drop .NET 6, 8, 9 and 10, using a topic-3 phrase from 3.3.5 (amended 2026-11-23 (AM-3/AM-4); it was `net10.0` and `net11.0`).
  - A **Breaking** list covers every 3.3.2 row marked breaking, using at least one 3.3.5 phrase per topic.
  - **The only migration aid (AM-4, AD-51; added 2026-11-23).** Each breaking entry names what replaces the old behavior or API, in one line. The list also states: raise direct BouncyCastle.Cryptography and Microsoft.IdentityModel.* references to the new minimums (NU1605); convert JWKS keys with `TryConvertToEdDsaSecurityKey` (3.3.3); and an app that sets `TokenValidationParameters.CryptoProviderFactory` must include EdDSA support in it. It names the feed, `https://nuget.pkg.github.com/IronSloth/index.json`, and that restoring needs a `read:packages` credential; the README (WI-48) gives the full setup in 1.x.
  - Also record the `\n` counting change and the new SimpleBase dependency.
- **done when:** The AC-104 phrase check finds all 18 topics. AC-105 holds for every tag or feed version.

#### WI-41 — Write the migration guide from upstream 4.1.0 (cut)

**Cut 2026-11-23 (AM-4; AD-51).** Not dispatched, and counted in no total. Its only purpose, AC-130 and R-54, is cut (CUT-1, CUT-3). The ID and text are kept so earlier references resolve. Its non-guide content (NU1605, JWKS conversion, the `CryptoProviderFactory` note, the feed URL) moved to WI-40's CHANGELOG entry.

- **domain:** `docs`
- **depends-on:** none (cut; amended 2026-11-23 (Gate 2): its pre-cut edge is no longer named here, so a cut item creates no dependency edge)
- **decisions:** AD-51 (cut)
- **satisfies:** none (was AC-130, and AC-131 through WI-45 and WI-47)
- **change (as written before the cut):**
  - `docs/migration/scottbrady-4.1.0-to-1.0.0.md`, with Before/After/Action rows for the 18 topics and 6 APIs (3.3.5).
  - It must also cover:
    - that a `net10.0` project, such as assetvault-io, keeps its framework and gets the `net10.0` build, and that moving to .NET 11 is not required (NG-11; amended 2026-11-23 (AM-1/AM-2/Q-30));
    - adding the feed `https://nuget.pkg.github.com/IronSloth/index.json` (D-2, GP-10) to `NuGet.config`. That means a `packageSourceMapping` entry for `ValhallaTech.*` and a `read:packages` token supplied through an environment variable, never committed;
    - raising direct BouncyCastle.Cryptography and Microsoft.IdentityModel.* references to the library's minimums (NU1605, RK-8);
    - converting JWKS keys (3.3.3);
    - the `TokenValidationParameters.CryptoProviderFactory` note.
  - It does not mention the sample's client-library change. That is not a package change, and AC-137 excludes only `specs/` and `CHANGELOG.md` from its text search.
- **done when:** AC-130's phrase match finds 0 uncovered items, and every row's three cells are non-empty.

#### WI-42 — Pack the ValhallaTech logo as the package icon

- **domain:** `backend`
- **depends-on:** U-1 (complete 2026-11-23), WI-5
- **decisions:** AD-54
- **satisfies:** AC-12
- **change:** Commit `assets/valhallatech-logo.png` exactly as Fred supplied it (amended 2026-11-23 (AM-1/AM-2/Q-30): the file now exists, uncommitted), and record its SHA-256 in the PR. `PackageIcon` is built from it, packed as `icon.png`. Delete the root `icon.png`.
- **done when:** AC-12's checks hold: PNG signature, IHDR at least 128 pixels, and the hash equals the supplied file and differs from `555d31c`.

#### WI-43 — Measure baseline against the release candidate and commit the report

- **domain:** `tests`
- **depends-on:** WI-38, WI-33, WI-57 (WI-57 added 2026-11-23 (Gate 2, third review): this item implements chain C14)
- **decisions:** AD-48, AD-54, AD-60, AD-65
- **satisfies:** AC-59, AC-60, AC-61, AC-62, AC-63, AC-64, AC-65, AC-66, AC-67, AC-68, AC-69, AC-70, AC-123, AC-127
- **change:**
  - Run the harness `-c Release` on Fred's workstation, the machine behind the GP-2 and Q-23 figures, on the release-candidate commit, on one .NET 11.0.x runtime (AD-48). _Amended 2026-11-23 (AM-1/AM-2/Q-30): runtimes 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): 11.0.x only. Originally one .NET 10.0.x runtime._
  - Commit the 3.3.5 files under `perf/results/1.0.0/`.
- **done when:**
  - The AC-59 parse passes: 32 rows, one 11.0.x runtime version, one machine name. The AC-123 file has 6 rows.
  - AC-68 shows 0 regressions; AC-70's means are within the budget scaled by `B`; AC-127 shows a time improvement.
  - A regression goes back to the owning item as a finding. A budget overrun is reported to Fred, not tuned away. AC-60 to AC-67 (R-23), AC-69 (R-24) and AC-123 (R-52) are `1.x`: they are measured and recorded either way, and a miss never holds 1.0.0 back (§2.2.1; AD-58, which superseded AD-56 on 2026-11-23 (Gate 2)). A miss is a finding for the owning item, fixed in a later 1.x release before this spec is done. The 1.0.0 measurement ACs, AC-59, AC-68, AC-70 and AC-127, do gate 1.0.0. _(Amended 2026-11-23 (Gate 2, F-1): AC-69 and AC-123 added to the record-only list, which WI-54 now matches.)_
  - **Chain C14 green (AD-60; added 2026-11-23 (Gate 2, third review)).** WI-57's `Measurement.Tests.ps1` runs over the committed files without edits. The AC-68, AC-70 and AC-127 tests pass. The AC-60 to AC-67 and AC-69 tests are `1.x` record-only checks: they pass or fail, and their result is recorded (§3.7).

#### WI-44 — Add the release workflow for the private feed

- **domain:** `ci`
- **depends-on:** WI-3, WI-59, U-5 (consumer credential for the trial restore; WI-59 added 2026-11-23 (Gate 2, third review), because `verify-gates` runs its tests)
- **decisions:** AD-2, AD-26, AD-47, AD-55, AD-59, AD-60, AD-65
- **satisfies:** AC-8 (mechanism), AC-58, AC-144 (enforcement), FC-1 (trial push and restore)
- **change:**
  - **D-2 (AD-2, GP-10):** GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json`.
  - `.github/workflows/release.yml` triggers on a pushed `v*.*.*` tag. It sets up the 11.0 SDK from `global.json` (AD-47; amended 2026-11-23 (AM-3/AM-4): was the 10.0.x and 11.0.x SDKs; originally .NET 10 only), then restores, builds, tests on runtime 11.0.x and packs. Each package carries `lib/net11.0/` only. The test run uses WI-3's two steps, `build/Invoke-Tests.ps1 -Scope Blocking` and then `-Scope OneX` with `if: ${{ !cancelled() }}`, so a failing `1.x`-marked test is recorded in the summary but never stops the publish (AD-58; amended 2026-11-23 (Gate 2)).
  - **`verify-gates` job (AC-144, AD-55; added 2026-11-23, amended 2026-11-23 (Gate 2)).** The first job, with `actions: read`, `statuses: read` and `contents: read`. For the tagged commit it checks both kinds of gate:
    - **Workflow runs.** It reads the latest run of each of `dotnetcore.yml`, `codeql-analysis.yml` and `release-gate.yml` (`gh run list --commit "$GITHUB_SHA" --workflow <file> --json conclusion,status,updatedAt`), and fails, naming the run, unless each exists, is completed, and concluded `success`.
    - **Commit statuses.** It reads `GET /repos/IronSloth/IdentityModel-Supa/commits/$GITHUB_SHA/statuses?per_page=100`, following every page, and fails, naming the context, unless each of `release-gate/ac-20`, `release-gate/ac-119` and `release-gate/test-first-audit` has at least one status and its newest one has `state == "success"`.
    The publish job `needs: verify-gates`, so nothing is pushed before every gate of both kinds has passed. A check that was skipped leaves no status, and blocks publication.
    _Amended 2026-11-23 (Gate 2, third review; AD-59):_ `verify-gates` makes both checks by running WI-59's tests: `build/Invoke-AcceptanceTests.ps1 -Path build/tests/ReleaseGates.Tests.ps1 -Data @{ Sha = $env:GITHUB_SHA } -ExcludeTag Post`. So the enforcement and AC-144's acceptance tests are one implementation.
  - **C17 stubs step (AD-60; added 2026-11-23 (Gate 2, third review)).** This item creates the surface WI-60's feed tests read: the publish path to `https://nuget.pkg.github.com/IronSloth/index.json`, proven by the FC-1 trial.
  - It runs `dotnet nuget push "**/*.nupkg" --source https://nuget.pkg.github.com/IronSloth/index.json --api-key ${{ secrets.GITHUB_TOKEN }}`. Only `.nupkg` files are pushed; GitHub Packages does not take `.snupkg` (inferred).
  - The publish job's permissions are `contents: read` and `packages: write`, and nothing else; `verify-gates` has `contents: read`, `actions: read` and `statuses: read` (amended 2026-11-23 (Gate 2)). No other workflow gains a write scope. The packages link to the repository through `RepositoryUrl` (3.3.5), which is what lets the workflow token publish.
  - **Trial run (FC-1), done early in Phase 1, not at release.** A `workflow_dispatch` input publishes a throwaway package, ID `ValhallaTech.FeedTrial`, version `0.0.1`. A separate ID keeps the two real package IDs free of versions that AC-105 would require the CHANGELOG to record. A scratch `net11.0` console then restores it with Fred's `read:packages` credential (U-5; amended 2026-11-23 (AM-3/AM-4): was a `net10.0` and a `net11.0` console; originally `net10.0` only). Afterwards the trial package is deleted from the feed. The trial uses `workflow_dispatch`, so `verify-gates` is skipped for it, and only for it. The trial job packs and pushes only the `ValhallaTech.FeedTrial` package, never the two real packages, so the skip cannot publish them.
  - A second `workflow_dispatch` input, `verify_only`, runs `verify-gates` alone against the dispatched commit and never reaches a publish job. It exists so the gate check can be proven without a tag.
- **done when:** The trial push succeeds and the trial restore exits 0. The trial package is deleted. A `verify_only` run on a branch commit that has no `release-gate.yml` run and no `release-gate/*` statuses fails at `verify-gates`, and its output names the missing run and each of the three missing contexts; `verify-gates` reports every failing gate before it exits (amended 2026-11-23 (Gate 2)). AC-58 still holds.

#### WI-45 — Dry-run the assetvault-io migration against the release candidate (cut)

**Cut 2026-11-23 (AM-4; AD-51).** Not dispatched, and counted in no total. AC-131 is cut (CUT-4), and no item reads, copies or builds assetvault-io (NG-12). The ID and text are kept so earlier references resolve. It no longer gates WI-46.

- **domain:** `tests`
- **depends-on:** none (cut; amended 2026-11-23 (Gate 2): its pre-cut edges are no longer named here, so a cut item creates no dependency edge)
- **decisions:** AD-51 (cut)
- **satisfies:** none (was AC-131, as a pre-flight for WI-46)
- **change (as written before the cut):**
  - Pack the release candidate to a local folder feed.
  - Copy assetvault-io `main` @ `863c0cb0cde62f820b76b9c76e1b4cf245f7dc55` into the session scratchpad. **Never edit `C:\Repo\assetvault-io`.**
  - Apply only the guide's steps, then `dotnet build` and `dotnet test tests/AssetVault.BLL.Tests`.
- **done when:** Build exit code is 0, with 0 failed and 0 skipped tests, and the diff touches only guide-directed lines. Any failure goes back to WI-41 (guide) or to the owning library item.

#### WI-46 — Run the release gate and publish 1.0.0 (point of no return)

_Amended 2026-11-23 (AM-1/AM-2/Q-30): AC-8's date bound is gone (AM-2); the gate is now three named runs (AC-144); AC-9 and the new AC-145 used a `net10.0` and a `net11.0` consumer; AC-49 uses WI-53's script. Amended 2026-11-23 (AM-3/AM-4): one `net11.0` consumer; WI-41 and WI-45 are cut and no longer gate this item (AD-51, AD-53). Amended 2026-11-23 (Gate 2): the non-workflow checks now post commit statuses before publishing, and AC-144 reads both kinds of gate (AD-55)._

- **domain:** `ci`
- **depends-on:**
  - WI-10, WI-12, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-28, WI-31, WI-33, WI-37, WI-39, WI-40, WI-42, WI-43, WI-44, WI-53, WI-54, WI-55, WI-56, WI-60, WI-61, WI-64 (WI-55 added 2026-11-23 (Gate 2, F-4); amended 2026-11-23 (Gate 2, AD-58): WI-56 added, and the `1.x`-only sample item removed, since it may not hold 1.0.0 back; amended 2026-11-23 (Gate 2, third review): WI-60, WI-61 and WI-64 added, whose tests and tool this item runs, and this item implements chain C17);
  - U-1 (complete 2026-11-23), U-2, U-3, U-4, U-5, U-6.
- **decisions:** AD-26, AD-52, AD-55, AD-57, AD-59, AD-60, AD-61, AD-62, AD-63, AD-64, AD-65
- **satisfies:**
  - AC-1, AC-4, AC-5, AC-7, AC-8, AC-9, AC-10, AC-11, AC-12, AC-15, AC-16, AC-17, AC-18;
  - AC-20 (verifies U-2), AC-45, AC-46, AC-47, AC-48, AC-49, AC-57, AC-118, AC-119 (verified);
  - AC-144, AC-145 (added 2026-11-23).
- **change:**
  0. **The release commit (AD-57; added 2026-11-23 (Gate 2, F-4)).** The release commit is WI-55's commit, which sets the CHANGELOG date. Every step below runs on that commit's `<sha>`. If a step needs a change, the change goes through its owning item, WI-55 runs again, and this item restarts at step 1 on the new commit.
  1. **Non-workflow gates, before any publish (AD-55; amended 2026-11-23 (Gate 2)).** With Fred's `gh` authorization (push access, or a classic token's `repo:status` scope, for posting; administrator read for AC-20 and AC-119), on the release commit `<sha>`:
     - Run AC-20 and AC-119, which need administrator scope the workflow token cannot have. Write each result to an evidence log under `C:/Users/fsmit/.claude/evidence/<yyyy-mm-dd>/`. _Amended 2026-11-23 (Gate 2, third review; AD-62):_ run them as WI-61's `Admin`-tagged tests, `build/Invoke-AcceptanceTests.ps1 -Path build/tests/RepositorySecurity.Tests.ps1 -Tag Admin`. The log holds each endpoint, its HTTP status and the `enabled` value, and nothing else.
     - Take U-6's history audit (AC-121, AC-132) from the Agent Evaluator's evidence log for this commit. U-6 runs WI-64's `build/Test-TestFirstHistory.ps1` on `<sha>`, together with WI-63's `Release`-tagged test (AD-61; amended 2026-11-23 (Gate 2, third review)). The evaluator posts nothing. The `release-gate/test-first-audit` status is posted when the log's 1.0.0-scope verdict is pass, whatever its `1.x`-scope results are (AD-58; amended 2026-11-23 (Gate 2)).
     - Only for a check that passed, post its status: `gh api repos/IronSloth/IdentityModel-Supa/statuses/<sha> -f state=success -f context=<context> -f description="evidence: <yyyy-mm-dd>/<log file name>"`, with context `release-gate/ac-20`, `release-gate/ac-119` or `release-gate/test-first-audit`. A failed check gets no status, and the release halts.
     - A later release commit needs all three posted again on its own `<sha>`.
  2. **Named gate runs on the release commit (AC-144).** Confirm the `dotnetcore.yml` run (AC-7) and the `codeql-analysis.yml` run for the release commit concluded `success`. Then dispatch `release-gate.yml` (WI-54) on that commit and wait for `success`. Its record-only `1.x` results (AD-58) are copied into the release evidence and routed to their owning items as findings; they do not hold the release. AC-49 runs inside it on that day, with any bump going through a normal PR first, which makes a new release commit and restarts at step 0. `quality.yml`'s result is read and recorded the same way, as `1.x` evidence, not as a gate (amended 2026-11-23 (Gate 2, F-1, F-4)).
     - **Consumer pre-flight (AD-63; added 2026-11-23 (Gate 2, third review)).** Pack the release commit into a local folder, and run WI-60's tests against it: `Feed.Tests.ps1` with `Source` set to that folder and `Version` 1.0.0, excluding the `FeedMetadata` tag. A failure halts the release before the point of no return, because the same AC-9 and AC-145 checks would fail after publication.
     - **`1.x` record (AD-58; added 2026-11-23 (Gate 2); amended 2026-11-23 (Gate 2, third review; R-2 and R-50 as amended, F-6)).** Write `C:/Users/fsmit/.claude/evidence/<yyyy-mm-dd>/1.0.0-1x-results.md` for the release commit. It has four lists:
       - **Failing `1.x`-marked tests.** Every test that failed in the `dotnetcore.yml` run's `1.x` TRX files, with its `Requirements` trait and its owning item. R-2 lets 1.0.0 ship with these.
       - **All `1.x`-marked tests,** each with its outcome.
       - **`1.x`-only audits.** U-6's `1.x`-scope results: each AC-121 and AC-132 entry, pass, fail or "not landed", with its owning item. R-50 gates them at the release that carries their requirements.
       - **`1.x` checks.** Every record-only check from the `release-gate.yml` summary with its result, the `quality.yml` run's conclusion, and each `1.x`-only work item that has not landed (WI-35 and WI-48 to WI-52, as applicable; WI-7 always lands before 1.0.0, AD-65, so its entry is AC-120's recorded result; amended 2026-11-23 (Gate 2, fourth review)).

       Every `1.x`-marked test must appear with an outcome of passed or failed (AC-4), and every record-only check with a result. A missing entry means a check did not run, so the release halts until it runs. A failed entry does not halt it. Each failing entry is a finding for its owning item, and must be fixed before WI-68.
  3. **Network.** The gate needs HTTPS to `registry.npmjs.org`, `api.nuget.org`, the GitHub API and Advisory Database, and, after publish, the IronSloth feed. An unreachable endpoint is a failed check that names the URL, never a skip. The release halts until the check runs and passes.
  4. **Publish.** Fred pushes the `v1.0.0` tag. WI-44's `verify-gates` re-reads the three runs and the three statuses, and only then does the publish job push both packages.
  5. **After publish.** Verify AC-8 (both IDs at exactly `1.0.0`, with a published timestamp). Verify AC-9 with one scratch `net11.0` console restoring both packages from the feed only. Verify AC-145 from its `obj/project.assets.json`: every asset under `lib/net11.0/`. Verify AC-144 against P, the earlier of the two packages' published timestamps: each of the three runs has `conclusion == "success"` and `updatedAt` < P, and for each of the three contexts the newest status with `created_at` < P, read from `GET /repos/IronSloth/IdentityModel-Supa/commits/<sha>/statuses` (all pages), has `state == "success"` (amended 2026-11-23 (Gate 2)). _Amended 2026-11-23 (Gate 2, third review; AD-59, AD-60):_ these checks are the chains' release checks. Run `Feed.Tests.ps1` (WI-60) with `Version` 1.0.0 against the feed, which covers AC-8, AC-9 and AC-145. Run `ReleaseGates.Tests.ps1` (WI-59) with `Sha` and `PublishedAt` = P, `Post` tag included, which covers AC-144. This step is the green check of chain C17, whose implementation is this item.
- **done when:** Every listed AC passes, under the convention "`1.x`-only checks never block 1.0.0" for AC-4, AC-7, AC-121, AC-132 and AC-144, and the `1.x` record of step 2 is complete. Any failure of a 1.0.0 check before step 4 halts the release and is reported, not worked around. A failure after step 4 follows RK-15 (forward-only). _Amended 2026-11-23 (Gate 2): the convention clause and the `1.x` record are added (AD-58)._
- **Spec completion (AD-58; added 2026-11-23 (Gate 2)).** Publishing 1.0.0 does not complete this spec while any `1.x`-only item is open or any `1.x` result in step 2's record failed. Each is a finding for its owning item. _Amended 2026-11-23 (Gate 2, third review; R-2 as amended, AD-64): was "at the commit where the last item lands"._ The spec completes at the 1.1.0 release (WI-65 to WI-68). That release runs the full suite and the full AC-121 and AC-132 audit, with no exemption.

#### WI-47 — Prove the assetvault-io migration from the published package (cut)

**Cut 2026-11-23 (AM-4; AD-51).** Not dispatched, and counted in no total. AC-131 is cut (CUT-4). The ID and text are kept so earlier references resolve.

- **domain:** `tests`
- **depends-on:** none (cut; amended 2026-11-23 (Gate 2): its pre-cut edge is no longer named here, so a cut item creates no dependency edge)
- **decisions:** AD-51 (cut)
- **satisfies:** none (was AC-131)
- **change (as written before the cut):** WI-45's procedure, with the package restored from the private feed at 1.0.0.
- **done when (as written before the cut):** AC-131's assertion holds.

##### Phase 3 — 1.x, required before this spec is done

#### WI-48 — Rewrite the README and add a readme per package

- **domain:** `docs`
- **depends-on:** WI-33, WI-35
- **decisions:** AD-2, AD-49
- **satisfies:** AC-100, AC-101, AC-103 (content), AC-122 (README half)
- **change:**
  - The README describes only existing features, has no Branca or PASETO, and has no upstream package, CI, issue or release links.
  - Installation names the feed `https://nuget.pkg.github.com/IronSloth/index.json` (D-2, GP-10) and the `read:packages` credential it needs, until the public nuget.org release (Q-2).
  - No readme mentions the sample's removed client library (AC-137 searches them).
  - The password section uses `AddExtendedPasswordRules`.
  - The README states the supported framework, `net11.0` only, and WI-50 compiles every C# example as C# 15 on `net11.0` (AD-49; added 2026-11-23 (AM-1/AM-2/Q-30) for two frameworks, amended 2026-11-23 (AM-3/AM-4) to one).
  - New `src/ValhallaTech.Security.IdentityModel/README.md` and `src/ValhallaTech.Security.IdentityModel.AspNetCore/README.md`, each naming its own package ID.
- **done when:** AC-100 and AC-101 searches return 0 matches. The AC-122 README search is clean.

#### WI-49 — Pack each package readme

- **domain:** `backend`
- **depends-on:** WI-48
- **decisions:** none
- **satisfies:** AC-103
- **change:** `PackageReadmeFile=README.md` in each library project, packing its own readme.
- **done when:** AC-103 holds on both packs.

#### WI-50 — Compile every documented C# example in CI

- **domain:** `ci`
- **depends-on:** WI-48, WI-46, WI-49 (WI-49 added 2026-11-23 (Gate 2): the AC-103 step reads the readme it packs)
- **decisions:** AD-49, AD-54, AD-65
- **satisfies:** AC-102; AC-103's CI check (amended 2026-11-23 (Gate 2))
- **change:**
  - **AC-103 check (AD-58; added 2026-11-23 (Gate 2)).** A `quality.yml` step runs `dotnet pack -c Release` and checks each `.nupkg` for AC-103: a nuspec `<readme>` element, and a packed readme that contains its own package ID. This check was CI-u in `dotnetcore.yml`, a 1.0.0 gate run, and R-41 is `1.x`.
  - Add `build/Test-DocSnippets.ps1` (placement rules in 3.3.5) and a step in `quality.yml` (AD-56; amended 2026-11-23 (Gate 2, F-1): was "on the ubuntu job", which is the `dotnetcore.yml` gate run; R-40 is `1.x`). Each block compiles in one `net11.0` project (AD-49; amended 2026-11-23 (AM-3/AM-4): was a `net10.0` and a `net11.0` project; originally `net10.0` only).
- **done when:** AC-102 shows 0 compiler errors for every block. The AC-103 step passes on both packages (amended 2026-11-23 (Gate 2)).

#### WI-51 — Gate line coverage per library at 80%

- **domain:** `ci`
- **depends-on:** WI-2, WI-3, WI-7 (may land during Phase 1; amended 2026-11-23 (Gate 2): the third edge is added because that item creates `quality.yml`)
- **decisions:** AD-50, AD-65
- **satisfies:** AC-116, AC-117
- **change:**
  - `quality.yml` (created by WI-7) runs the library test project with `--collect "XPlat Code Coverage"`, which writes Cobertura through `coverlet.collector`, then runs the gate below. _Amended 2026-11-23 (Gate 2, F-1; AD-56): was "the ubuntu test step" of `dotnetcore.yml`. R-47 is `1.x`, so its gate must not fail a 1.0.0 gate run (§2.2.1)._
  - `build/Test-CoverageThreshold.ps1` reads the per-package `line-rate` for exactly the two library assemblies. It fails when either is below 80.0, printing the name and the percentage to one decimal. _Amended 2026-11-23 (AM-1/AM-2/Q-30): one report per framework run. Amended 2026-11-23 (AM-3/AM-4; AD-50): one report again, as originally written._
  - A committed fixture, `build/fixtures/coverage-79.9.cobertura.xml`, is run in the same job and **must** make the gate fail.
  - This is a fixed floor, not a ratchet (C-17). It is a script because AC-117 feeds the gate a report, which coverlet's MSBuild threshold cannot accept.
- **done when:** AC-116 is at least 80% for each library on the CI build, and the AC-117 fixture run fails naming the library and `79.9`.

#### WI-52 — Fail the build on a breaking public-API change after 1.0.0

- **domain:** `ci`
- **depends-on:** WI-46
- **decisions:** AD-25
- **satisfies:** AC-115
- **change:**
  - In `src/Directory.Build.props`, set `EnablePackageValidation=true`, with the 1.0.0 packages committed under `eng/baseline/` and referenced through `PackageValidationBaselinePath` (inferred property name; verify first).
  - A committed baseline means no developer restore needs feed credentials. Alternative lost: `PackageValidationBaselineVersion`, which needs every restore to authenticate to the private feed.
- **done when:** AC-115's probe, which removes `Base62.Encode(byte[])`, fails pack and names the member.

##### Added 2026-11-23 (AM-1/AM-2/Q-30)

#### WI-53 — Check every direct NuGet dependency against its latest stable version

_Amended 2026-11-23 (AM-3/AM-4): was "against its required version per framework", with a framework-bound rule. R-17 and AC-49 now hold each dependency to one latest stable version (AD-46). The script name is kept so no reference breaks._

- **domain:** `ci`
- **depends-on:** none (Phase 0, first item; amended 2026-11-23 (Gate 2, F-1): this line used to name the item that waits on this one, which made a false two-way edge)
- **decisions:** AD-45, AD-46, AD-59
- **satisfies:** AC-49 (the check; WI-1 and WI-2 make it pass)
- **change:**
  - **Pester runner (AD-59; added 2026-11-23 (Gate 2, third review)).** Add `build/Invoke-AcceptanceTests.ps1 -Path <file> -Data <hashtable> [-Tag <string[]>] [-ExcludeTag <string[]>] [-ResultPath <path>]`. It builds a `New-PesterContainer -Path $Path -Data $Data` and a configuration with `Run.Exit = $true`, `TestResult.Enabled = $true` and the tag filters, then runs `Invoke-Pester`. If Pester 5 or later is not installed, it installs the latest stable Pester for the current user first. The chain tests items WI-57 to WI-61 and WI-63 run through it, so it lands first.
  - Add `build/Test-RequiredVersions.ps1`. For every project and every direct `PackageReference` D, it reads the version resolved in the `net11.0` entry of `targets` in `obj/project.assets.json`. It reads D's latest stable version from the NuGet V3 registration resource (`RegistrationsBaseUrl` from `https://api.nuget.org/v3/index.json`) under AC-49's definitions of "stable" and "latest stable".
  - It also checks that the latest stable version supports `net11.0` under AC-49's definition, using the `lib/<tfm>/` fallback when a catalog entry has no dependency groups. A latest stable version that does not is reported as an A-14 case for Fred, never passed.
  - It prints one line per (project, D) with the resolved and latest stable versions, and exits non-zero on any mismatch.
  - An unreachable registry, or a `.nupkg` it cannot fetch for the fallback, fails the run with "NuGet registry unreachable" and the URL. It never passes or skips.
  - Committed fixtures under `build/fixtures/` (an assets file and recorded registration responses) prove the logic offline: one where every package is at its latest stable, which must pass; one where a package is one stable version behind, which must fail; and one where the latest stable declares only a newer framework than `net11.0`, which must fail as an A-14 case.
- **done when:** The three fixture runs give the expected exit codes. A live run over the repository at the current commit prints every direct dependency and its latest stable version, and its result is attached to WI-1's PR. The runner exits 0 on a one-test Pester file that passes and non-zero on one that fails, and passes `-Data` values to the file's `param()` (added 2026-11-23 (Gate 2, third review)).

#### WI-54 — Add the release-gate workflow that AC-144 names

- **domain:** `ci`
- **depends-on:** WI-3, WI-53, WI-56, WI-57, WI-58, WI-59, WI-61 (WI-56 added 2026-11-23 (Gate 2): the misuse check runs its test class; WI-57, WI-58, WI-59 and WI-61 added 2026-11-23 (Gate 2, third review): this item runs their tests and implements chain C16)
- **decisions:** AD-47, AD-55, AD-59, AD-60, AD-65
- **satisfies:** AC-144 (the named run); runs the release-time checks of AC-1, AC-2, AC-5, AC-6, AC-10 to AC-12, AC-15 to AC-19, AC-45 to AC-51, AC-55, AC-56, AC-58 to AC-70, AC-98, AC-99, AC-104, AC-105, AC-113, AC-118, AC-120, AC-123, AC-127, AC-133, AC-134, AC-136, AC-137, AC-139 and AC-140 (amended 2026-11-23 (AM-3/AM-4): AC-130 removed, since it is cut)
- **change:**
  - `.github/workflows/release-gate.yml`, triggered by `workflow_dispatch` only, with `permissions: contents: read`, `actions: read`, `issues: read` and `pull-requests: read`. It sets up the 11.0 SDK from `global.json` (AD-47; amended 2026-11-23 (AM-3/AM-4): was the 10.0.x and 11.0.x SDKs), builds and packs `-c Release`, then runs `build/Invoke-ReleaseGate.ps1`.
  - `Invoke-ReleaseGate.ps1` runs each check in §3.7's "Gate" rows in turn, using the commands those rows name, and prints a pass or fail line per AC. The measurement ACs (AC-60 to AC-70, AC-127) and the report-shape `lint` ACs (AC-59, AC-123) are parses and comparisons of the committed `perf/results/1.0.0/` files, not new measurements. _Amended 2026-11-23 (Gate 2, fourth review): this sentence grouped all of them as "Measurement ACs". It now follows the narrowed §2.3 convention (F-2)._
  - **Gating and record-only checks (AD-56; amended 2026-11-23 (Gate 2, F-1)).** Every check runs and its result is printed and written to the run's job summary. The script exits non-zero only if a **1.0.0** check fails, after running all of them. The checks of `1.x` requirements are **record-only**: a failure is reported in the summary as "1.x, recorded, not gating" and goes to the owning item as a finding, but never fails the run, because §2.2.1 forbids a `1.x` requirement from holding 1.0.0 back. Record-only here: AC-58 (R-21), AC-60 to AC-67 (R-23), AC-69 (R-24), AC-113 (R-45), AC-120 (R-49) and AC-123 (R-52). Every other check in this item's list gates, including AC-59, AC-68, AC-70 and AC-127. _Amended 2026-11-23 (Gate 2; AD-58, which supersedes AD-56):_ the script reads the record-only set from `oneXAcs` in `build/release-scope.json` (WI-3), the one file that holds it, and holds no list of its own. A check is record-only exactly when its AC is in that set, so a check is never moved between the lists by an agent.
  - **Marker misuse check (AD-58; added 2026-11-23 (Gate 2)).** A gating check, "1.x marker misuse", runs `dotnet test -c Release --no-build --filter "FullyQualifiedName~ReleaseScopeTraitTests"` for every test project (WI-56). It fails if any `1.x`-marked test lists a requirement shipping in `1.0.0` or `all`, which is the misuse assertion of AC-4, AC-7, AC-121 and AC-132. This workflow runs no other `dotnet test`: AC-4's and AC-7's results come from the `dotnetcore.yml` run on the same commit.
  - **Chain acceptance tests (AD-59; added 2026-11-23 (Gate 2, third review)).** The checks below are made by running the chains' Pester files through `build/Invoke-AcceptanceTests.ps1`, not by separate commands, so each AC has one implementation:
    - `Measurement.Tests.ps1` (WI-57) for AC-60 to AC-70 and AC-127;
    - `CiRun.Tests.ps1` (WI-58) for AC-7, with `Sha` set to this run's commit;
    - `RepositorySecurity.Tests.ps1` (WI-61) with `-ExcludeTag Admin`, for AC-118.

    The workflow token gets no administration scope. Results map to ACs by the `It` names, so `oneXAcs` decides which are record-only.
  - Checks that need administrator scope (AC-20, AC-119) are not in this workflow; WI-46 runs them locally and records each as a commit status (AD-55; amended 2026-11-23 (Gate 2)).
  - Network: `registry.npmjs.org`, `api.nuget.org` and the GitHub API. An unreachable endpoint fails its check, naming the URL.
- **done when:** A dispatched run on a branch commit runs every listed check and reports one line per AC. A probe branch with one deliberate defect (an extra `lib/net10.0/` folder in a pack) makes the run fail on AC-1 and still report every other check. A second probe with one record-only defect (a deliberately misformatted line, failing AC-120) leaves the run green, with AC-120 reported as "1.x, recorded, not gating" (amended 2026-11-23 (Gate 2, F-1)). A third probe marks one test of a `1.0.0` requirement, for example `[Trait("Release", "1.x")]` with `[Trait("Requirements", "R-15")]`, and makes the run fail on "1.x marker misuse" (added 2026-11-23 (Gate 2), AD-58).
  - **Chain C16 green (AD-60; added 2026-11-23 (Gate 2, third review)).** On a branch commit where `dotnetcore.yml`, `codeql-analysis.yml` and a dispatched `release-gate.yml` all concluded `success`, WI-59's untagged tests pass without edits. Its `Statuses`- and `Post`-tagged tests stay red. They are release checks, turned green by WI-46 steps 1 and 5 (§3.7).

##### Added 2026-11-23 (Gate 2)

#### WI-55 — Set the 1.0.0 release date in `CHANGELOG.md` on the release commit

- **domain:** `docs`
- **depends-on:** WI-10, WI-12, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-28, WI-31, WI-33, WI-37, WI-39, WI-40, WI-42, WI-43, WI-44, WI-53, WI-54, WI-56 (amended 2026-11-23 (Gate 2, AD-58): WI-56 added, and the `1.x`-only sample item removed)
- **decisions:** AD-57, AD-65
- **satisfies:** AC-104 (the final dated heading), AC-105
- **change:** Once every other 1.0.0 item has merged (a `1.x`-only item is not waited for; AD-58), change the date in `## [1.0.0] - <yyyy-mm-dd>` to the day of this commit, and change nothing else. This commit is the release commit that WI-46 gates. If anything must change after it, that change makes a new release commit, and this item runs again on that day, before the gates rerun (AD-57).
- **done when:** The heading's date equals the commit's date, the diff touches only that line, and the AC-104 phrase check still finds all 18 topics.

#### WI-56 — Add the `1.x` marker misuse check to every test project

_Added 2026-11-23 (Gate 2), for the §2.3 convention "`1.x`-only checks never block 1.0.0" (AD-58)._

- **domain:** `tests`
- **depends-on:** WI-3, WI-6
- **decisions:** AD-65
- **satisfies:** the misuse assertion of AC-4, AC-7, AC-121 and AC-132 (no `1.x`-marked test lists a requirement shipping in `1.0.0` or `all`)
- **change:**
  - Add `test/Directory.Build.props`. It imports the parent `Directory.Build.props` (`$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))`) and adds `<Compile Include="$(MSBuildThisFileDirectory)Shared/ReleaseScopeTraitTests.cs" Link="Shared/ReleaseScopeTraitTests.cs" />`. Every project under `test/`, including the sample test project WI-34 creates, then compiles the class.
  - Add `test/Shared/ReleaseScopeTraitTests.cs`, an unmarked xUnit class. It finds `build/release-scope.json` by walking up from `AppContext.BaseDirectory` to the directory that holds `global.json`, and reads `oneXRequirements`. It then reflects over every `[Fact]` and `[Theory]` method in its own assembly and asserts, naming each offending method:
    - every method carrying `Trait("Release", …)` has the value `1.x`;
    - every marked method also carries one `Trait("Requirements", …)` whose value matches `^R-\d+(,R-\d+)*$`;
    - every ID in a marked method's value is in `oneXRequirements`;
    - no class and no assembly carries either trait.
  - Fixtures prove the rules. Internal test-double types, which the reflection scan skips because they carry no `[Fact]` or `[Theory]`, cover four cases: a correct marker; a marker with `R-15`; a marker with no requirement trait; and a class-level marker. The class runs the same rule function over each fixture and asserts that the first passes and the other three fail with the expected reason.
- **done when:** In the library test project, `ReleaseScopeTraitTests` passes on the branch (the sample test project does not exist yet; WI-34's done-when checks it there), and the fixture assertions show the three misuse cases rejected. A `1.x`-marked test listing `R-15`, added on a probe branch, makes the blocking CI step fail by naming that method. The probe branch is deleted afterwards.

##### Added 2026-11-23 (Gate 2, third review): chains for the out-of-process seams, and the 1.1.0 release

_Chains C14 to C19 (AD-59, AD-60). In each tests item, "fails by assertion" means a Pester `Should` failure, not a script error. The release checks and Fred-state checks named in §3.7 are exempt from the green check after implementation._

#### WI-57 — Write the failing measurement-report tests (chain C14)

- **domain:** `tests`
- **depends-on:** WI-38
- **decisions:** AD-48, AD-59, AD-60, AD-65
- **satisfies:** AC-60, AC-61, AC-62, AC-63, AC-64, AC-65, AC-66, AC-67, AC-68, AC-69, AC-70, AC-127 (acceptance tests)
- **change:**
  - `build/tests/Measurement.Tests.ps1`, with `param([string] $ReportDirectory = 'perf/results/1.0.0')`, and one `It` per AC, named `AC-<n> …`. It reads the 3.3.5 files:
    - `report.csv` for AC-60 to AC-68 and AC-127;
    - `base16-scaling.csv` for AC-69;
    - `base62-budget.csv` for AC-70.
  - Each test first asserts that every row its AC compares is present, for example both the baseline and the after row of P-a at every size. It then applies the AC's comparison under §2.3's measurement rules. So a header-only file fails every test by assertion, AC-68 included, rather than letting "0 regressions" pass on no rows.
  - Two fixture directories prove the comparisons. `build/fixtures/measurement-pass/` holds synthetic rows that meet every rule. `build/fixtures/measurement-regress/` matches it except for one regressed pair.
- **done when:**
  - Against WI-38's header-only files, all 12 tests run and fail by assertion, each naming the missing rows, and none errors.
  - Against `measurement-pass`, all 12 pass.
  - Against `measurement-regress`, AC-68 fails and names the pair.

#### WI-58 — Write the failing CI-run tests (chain C15)

- **domain:** `tests`
- **depends-on:** WI-53
- **decisions:** AD-47, AD-59, AD-60, AD-65
- **satisfies:** AC-7 (acceptance tests)
- **change:** `build/tests/CiRun.Tests.ps1`, with `param([string] $Sha, [string] $Repository = 'IronSloth/IdentityModel-Supa')`.
  - **What it reads.** It finds the latest `dotnetcore.yml` run for `$Sha` (`gh run list --repo $Repository --commit $Sha --workflow dotnetcore.yml --json databaseId,status,conclusion --limit 1`) and reads its jobs (`gh run view <id> --json jobs`). It downloads the run's `TestResults` artifact (`gh run download <id>`) into a new temp directory.
  - **What it asserts.** Each AC-7 assertion, as amended, is its own `It`, named `AC-7 …`:
    - the run completed, and every job concluded `success`;
    - every test project has TRX files under both `TestResults/Blocking/` and `TestResults/OneX/`, or an explicit "no test matches" record for the `1.x` scope;
    - every recorded runtime is 11.0.x;
    - failed == 0 in the blocking TRX files;
    - every `1.x`-marked test has an outcome.
- **done when:** Run against this item's own commit, every test fails by assertion and none errors. Before WI-3 there is no per-scope TRX artifact and no 11.0.x runtime.

#### WI-59 — Write the failing release-gate tests (chain C16)

- **domain:** `tests`
- **depends-on:** WI-53
- **decisions:** AD-55, AD-59, AD-60
- **satisfies:** AC-144 (acceptance tests)
- **change:** `build/tests/ReleaseGates.Tests.ps1`, with `param([string] $Sha, [string] $Repository = 'IronSloth/IdentityModel-Supa', [Nullable[datetime]] $PublishedAt)`. It has three groups of tests:
  - **Untagged.** For each of `dotnetcore.yml`, `codeql-analysis.yml` and `release-gate.yml`, the latest run for `$Sha` exists, is completed and concluded `success`.
  - **Tagged `Statuses`.** It reads every page of `GET repos/$Repository/commits/$Sha/statuses?per_page=100`. For each of `release-gate/ac-20`, `release-gate/ac-119` and `release-gate/test-first-audit`, the newest status is `success`, and its `description` matches `^evidence: \d{4}-\d{2}-\d{2}/[^/]+$`.
  - **Tagged `Post`.** With `$PublishedAt` as P, each run's `updatedAt` < P. For each context, the newest status with `created_at` < P is `success`.
- **done when:** Against this item's own commit, every test fails by assertion: no `release-gate.yml` run and no `release-gate/*` status exists yet. None errors.

#### WI-60 — Write the failing feed and consumer tests (chain C17)

- **domain:** `tests`
- **depends-on:** WI-44, U-5
- **decisions:** AD-2, AD-59, AD-60, AD-63
- **satisfies:** AC-8, AC-9, AC-145 (acceptance tests)
- **change:**
  - **Consumer fixture.** `build/fixtures/feed-consumer/FeedConsumer.csproj` is a `net11.0` console with two `PackageReference`s whose `Version="$(ConsumerVersion)"`. Its `Program.cs` is the AC-9 program: it creates an Ed25519 `EdDsaSecurityKey`, issues a JWT through `JsonWebTokenHandler.CreateToken` with `SigningCredentials(key, "EdDSA")`, validates it with `ValidateTokenAsync` using that key, and prints `IsValid`'s value.
  - **Test file.** `build/tests/Feed.Tests.ps1`, with `param([string] $Source = 'https://nuget.pkg.github.com/IronSloth/index.json', [string] $Version = '1.0.0', [string[]] $PackageIds = @('ValhallaTech.Security.IdentityModel', 'ValhallaTech.Security.IdentityModel.AspNetCore'))`. It holds three tests:
    - **AC-8, tag `FeedMetadata`.** From the feed's service index and registration resource, each package ID has exactly `$Version` and a `published` timestamp.
    - **AC-9.** The fixture is copied out of the repository and given the AD-63 `nuget.config`. Then `dotnet restore`, `build` and `run`, each with `-p:ConsumerVersion=$Version`, each exit 0, and the output is `True`.
    - **AC-145.** In that consumer's `obj/project.assets.json`, every `compile` and `runtime` asset of both packages is under `lib/net11.0/`.
- **done when:**
  - Against the feed at 1.0.0, which is not published yet, every test fails by assertion: the version is absent, and restore exits non-zero. None errors.
  - The written `nuget.config` holds `%IRONSLOTH_PACKAGES_READ_TOKEN%` literally, and no token value appears in the output.
  - NuGet's expansion of that variable is proven first (RK-22).
  - **Harness proof, not the AC.** The two libraries, packed from this item's branch at `0.0.0-feedtest` into a local folder, pass AC-9 and AC-145 with `Source` set to that folder and `-ExcludeTag FeedMetadata`.

#### WI-61 — Write the failing repository-security tests (chain C18)

- **domain:** `tests`
- **depends-on:** WI-53
- **decisions:** AD-8, AD-59, AD-60, AD-62
- **satisfies:** AC-20, AC-118, AC-119 (acceptance tests)
- **change:** `build/tests/RepositorySecurity.Tests.ps1`, with `param([string] $Repository = 'IronSloth/IdentityModel-Supa', [Nullable[datetime]] $ReleaseDate)`. It is read-only (AD-62) and holds three tests:
  - **AC-20, tag `Admin`.** `gh api repos/$Repository/private-vulnerability-reporting` exits 0, and `.enabled` is `true`.
  - **AC-119, tag `Admin`.** `gh api repos/$Repository/vulnerability-alerts` exits 0, which means 204. An `HTTP 404` fails by assertion.
  - **AC-118.** The commit that added `renovate.json` on `master` exists (`git log master --diff-filter=A --format=%cI -- renovate.json`). `gh pr list --repo $Repository --author app/renovate --state all --json createdAt`, or `gh issue list` with the same author, returns at least one item created after that commit. When `$ReleaseDate` is given, that item was created no later than the release date.
- **done when:**
  - Every test fails by assertion today, and none errors: U-2 and U-3 are not done, and no `renovate.json` is on `master`.
  - If Fred has finished U-2 or U-3 before the red check runs, that test passes. It is then recorded as a Fred-state check (AD-60), not a halt.
  - `rg -n -e '--method' -e '\s-X\s' build/tests/RepositorySecurity.Tests.ps1` returns 0 matches.
  - The Security Reviewer has reviewed the file against AD-62.

#### WI-62 — Stub the test-first audit tool (chain C19)

- **domain:** `ci`
- **depends-on:** WI-3, WI-53
- **decisions:** AD-28, AD-61, AD-65
- **satisfies:** AC-121, AC-132 (surface only)
- **change:**
  - `build/Test-TestFirstHistory.ps1`, with parameters `-RepositoryPath`, `-BaseCommit` (default `555d31c`), `-HeadCommit`, `-ChainMap` (default `build/test-first-chains.json`), `-ScopeFile` (default `build/release-scope.json`) and `-OutDirectory`. For now it writes `test-first-audit.json` holding `{ "head": "<sha>", "scopes": [] }` and an empty `test-first-audit.md`, and exits 0.
  - `build/test-first-chains.json`, taken from §3.4 and §3.7. It lists each chain from C1 to C13b with:
    - its in-scope ACs (AC-121's list);
    - its tests item and implementing item;
    - its guards, the C11 and C12 replacement tests among them by fully qualified name;
    - for each behavior-changing work item, the selector of its AC-132 tests. For example, WI-31's selector is the trait `FixedBy=WI-31`, and WI-28's is the AC-106 to AC-109 unit tests.
- **done when:** Run on any commit, the stub writes the empty report and exits 0. The chain map names every in-scope AC of AC-121 exactly once, and every work item in §3.4's unit-level test-first list.

#### WI-63 — Write the failing audit tests (chain C19)

- **domain:** `tests`
- **depends-on:** WI-62
- **decisions:** AD-28, AD-59, AD-60, AD-61, AD-65
- **satisfies:** AC-121, AC-132 (acceptance tests)
- **change:** `build/tests/TestFirstHistory.Tests.ps1`.
  - **Fixture tests.** At test time they build throwaway git repositories under the temp directory. Each holds a one-class xUnit project on `net11.0`, a chain map and a scope file, plus a scripted commit history. Each test runs the tool on one history and asserts the report entry:
    - (a) a test commit red by assertion, then an implementing commit that turns it green: pass;
    - (b) the implementing commit before the test commit: fail, "implementation precedes test";
    - (c) a test that does not compile at its introducing commit: fail, "compile error, not assertion";
    - (d) a declared guard, green at its introducing commit: pass as a guard;
    - (e) test and fix squashed into one commit: fail;
    - (f) a failing `1.x`-only AC that is listed in `oneXAcs`: the 1.0.0 verdict passes, and the `1.x` section records the failure;
    - (g) the same history with both scope lists empty: the single verdict fails;
    - (h) a `1.x`-only work item with no commits: "not landed" in the `1.x` section.
  - **Release test, tag `Release`.** It runs the tool on the repository at `$HeadCommit` and asserts that the 1.0.0-scope verdict passes, or the single verdict when both scope lists are empty.
- **done when:** Against WI-62's stub, every fixture test fails by assertion, because the report is empty, and none errors. The `Release` test also fails, because the chains have not landed. It is a release check (§3.7).

#### WI-64 — Implement the test-first audit tool (implements C19)

- **domain:** `ci`
- **depends-on:** WI-63
- **decisions:** AD-28, AD-61, AD-65
- **satisfies:** AC-121, AC-132 (the executable audit U-6 runs)
- **change:** Implement AD-61 in `build/Test-TestFirstHistory.ps1`. Worktrees go under the system temp directory and are removed after the run. The tool never changes the repository's branches or working tree.
- **done when:**
  - WI-63's fixture tests pass without edits.
  - Its `Release` test stays red until the release commit: a release check, turned green by U-6's run at WI-46.
  - A run over the repository's current history completes, and lists every chain that has landed.

#### WI-65 — Remove the `1.x` markers before the 1.1.0 release

- **domain:** `tests`
- **depends-on:** WI-7, WI-35, WI-46, WI-48, WI-49, WI-50, WI-51, WI-52
- **decisions:** AD-64, AD-65
- **satisfies:** R-2's final clause (AC-4, last bullet), AC-7 at 1.1.0
- **change:** Remove every `[Trait("Release", "1.x")]` from the test projects, and keep every `Requirements` trait. Every test then runs in the blocking step.
- **done when:**
  - `rg -n 'Trait\("Release"' test/` returns 0 matches.
  - The branch's CI run shows failed == 0 and skipped == 0 across all tests in the blocking TRX files.
  - A test that fails goes to its owning item, whose fix lands first. No test is weakened.

#### WI-66 — Empty the `1.x` scope for the release that completes this spec

- **domain:** `ci`
- **depends-on:** WI-65
- **decisions:** AD-64, AD-65
- **satisfies:** AC-4, AC-7, AC-121, AC-132 and AC-144 at 1.1.0, with no exemption (R-2, R-50 as amended)
- **change:** Set `oneXRequirements` and `oneXAcs` in `build/release-scope.json` to `[]`. From then on, no check in `Invoke-ReleaseGate.ps1` is record-only. The audit tool returns one verdict, and the misuse check (WI-56) rejects any `1.x` marker.
- **done when:**
  - `release-gate.yml`, dispatched on the branch, prints no record-only line, and exits non-zero on any failed check.
  - `quality.yml` concludes `success`.
  - If a `1.x` measurement AC failed at 1.0.0, its fix's owner has re-run WI-43's harness and recommitted the 3.3.5 files, and `Measurement.Tests.ps1` passes in full.

#### WI-67 — Write and date the 1.1.0 CHANGELOG entry on the release commit

- **domain:** `docs`
- **depends-on:** WI-66
- **decisions:** AD-57, AD-64
- **satisfies:** AC-105 (the 1.1.0 heading)
- **change:** Add `## [1.1.0] - <yyyy-mm-dd>` above the 1.0.0 entry, listing every `1.x` change that 1.0.0 did not ship, under Keep a Changelog sections. Its date is the day of this commit, which is the 1.1.0 release commit. If anything changes after it, this item runs again, as WI-55 does (AD-57).
- **done when:** The heading's date equals the commit's date, and AC-105's heading check matches 1.1.0 once it is tagged and published.

#### WI-68 — Run the release gate and publish 1.1.0, completing this spec (point of no return)

- **domain:** `ci`
- **depends-on:** WI-67, WI-64, U-6
- **decisions:** AD-55, AD-57, AD-59, AD-60, AD-62, AD-63, AD-64, AD-65
- **satisfies:** R-2's final clause (AC-4, last bullet); AC-7, AC-121, AC-132 and AC-144 with no exemption; AC-9 and AC-145 for 1.1.0 as evidence
- **change:** Run WI-46's steps 0 to 5 on WI-67's commit, with version 1.1.0 and tag `v1.1.0`. With the scope lists empty:
  - the CI blocking step runs every test;
  - `release-gate.yml` has no record-only check;
  - U-6's audit returns one verdict over every in-scope AC and work item;
  - `quality.yml` must conclude `success` before the tag;
  - step 2's `1.x` record is replaced by a statement that no exemption applied.

  `Feed.Tests.ps1` runs with `Version` 1.1.0.
- **done when:** Every gate passes with no exemption, and both packages are on the feed at 1.1.0. The evidence folder records that the spec is complete. A failure before the tag halts the release and goes to its owning item. A failure after it follows RK-15.

#### Unmapped work items

_Work with no tag in the allowed set. Never invent a tag — list it here with a
recommended owner so the orchestrator can route by judgment._

| ID | Title | Satisfies | Recommended owner | Why unmapped |
| --- | --- | --- | --- | --- |
| U-1 | Supply the ValhallaTech logo as a PNG of at least 128×128 at `assets/valhallatech-logo.png` | AC-12 (FC-6) | Fred | A brand asset only Fred holds; no agent can produce it. **Complete 2026-11-23:** a 512×512 transparent PNG of the company emblem, at that path, not yet committed (WI-42 commits it). *(Per the /spec coordinator, 2026-11-23.)* |
| U-2 | Enable GitHub private vulnerability reporting on `IronSloth/IdentityModel-Supa` | AC-20 (C-21, FC-2) | Fred | A repository setting only Fred can change (C-21). Still disabled on 2026-11-23 *(per the /spec coordinator, via `gh api`)*. |
| U-3 | Enable Dependabot alerts (`vulnerability-alerts`) on the canonical repository | AC-119 | Fred | A repository security setting, the same class as C-21. Still disabled on 2026-11-23 *(per the /spec coordinator, via `gh api`)*. |
| U-4 | Install the Renovate GitHub App on `IronSloth/IdentityModel-Supa` | AC-118 | Fred | An account-level app installation; no repository file can do it. |
| U-5 | Issue the GitHub Packages credentials for the IronSloth feed (D-2 decided, GP-10): a `read:packages` token for consumers and for the WI-44 trial restore and AC-9. Confirm the workflow's `packages: write` publish is allowed for the IronSloth account. | AC-8, AC-9 (FC-1); unblocks WI-44's trial restore | Fred | Credentials belong to Fred's account. FC-1 says they are Fred's to issue. |
| U-6 | Audit test-first ordering from the commit history | AC-121, AC-132 | Agent Evaluator | It is an evaluation of the history the work items produce, not an artifact any domain builds (R-50 names the evaluator). _Amended 2026-11-23 (Gate 2):_ the audit runs on the release commit and writes an evidence log; WI-46 posts the `release-gate/test-first-audit` status from it. The evaluator posts nothing (AD-55). _Amended 2026-11-23 (Gate 2, AD-58):_ the audit reads `oneXAcs` from `build/release-scope.json` and audits every in-scope AC and work item. Its log has two sections, each with its own verdict. **1.0.0 scope:** AC-121 for every in-scope AC not in `oneXAcs`, and AC-132 for every behavior-changing work item that is not `1.x`-only. For WI-28 that means the AC-109 unit test, because the AC-106 to AC-108 tests are marked. **`1.x` scope:** AC-121 for AC-52 to AC-54, AC-87, AC-88, AC-106 to AC-108, AC-112 and AC-114, and AC-132 for WI-35, each audited if it has landed and listed as "not landed" if not. Only the 1.0.0-scope verdict decides the status; the `1.x` section goes into WI-46's `1.x` record. _Amended 2026-11-23 (Gate 2, third review; AD-61):_ the audit is now executable. The evaluator runs WI-64's `build/Test-TestFirstHistory.ps1` and WI-63's `Release`-tagged test on the release commit, reviews the report, and writes the evidence log from them. At 1.1.0 (WI-68) the scope lists are empty, so the tool returns one verdict over every in-scope AC and work item, with no exemption (R-50). |

### 3.5 Sequencing

_Ordered phases with the work items in each and what gates the next phase._

_Amended 2026-11-23 (AM-1/AM-2/Q-30): no date drives the plan; Gate 0 required both runtimes; WI-53 and WI-54 are added; U-1 is complete. Amended 2026-11-23 (AM-3/AM-4): one runtime, .NET 11; WI-41, WI-45 and WI-47 are cut and removed from the sequence; G2-1 is answered (AD-43); the critical path is re-derived; AD-52 replaces AD-42._

**Phase 0 — Foundation (serial, one lane):** WI-53 → WI-1 → WI-2 → WI-4 → WI-5 + WI-6 (one PR) → WI-7 → WI-8 → WI-9. _(Amended 2026-11-23 (Gate 2, F-1): was "WI-1 (with WI-53)", which the depends-on lines turned into a cycle. WI-53, a small script, now lands first, and WI-1 starts on it.)_
- These items run serially on purpose. The rename, the reformat and the annotation pass each touch every file, so running any of them beside other work guarantees merge conflicts (RK-9).
- **WI-7 gates 1.0.0 as sequencing** (AD-65; amended 2026-11-23 (Gate 2, fourth review)). It is a `1.x`-only item, because its only AC is AC-120, but WI-8 and WI-39 wait for it, so it holds up everything after it. It is done once its reformat and `quality.yml` have merged. AC-120's result is recorded and never holds WI-8, WI-39 or 1.0.0. If CSharpier cannot run at all, the reformat moves to Phase 4.
- In parallel: WI-58 (after WI-53, before WI-3), then WI-3 (after WI-2), WI-56 (after WI-3 and WI-6, before any tests item that marks tests; added 2026-11-23 (Gate 2)), WI-59 and WI-61 (after WI-53; added 2026-11-23 (Gate 2, third review)), WI-10 (any time), and the Fred-only items U-2 to U-5, which should start at Gate 2 approval. U-1 is complete. G2-1 is answered (AM-3), so nothing about the language version waits on Gate 2.
- **Gate 0:**
  - CI green on `ubuntu-latest` with every test run on runtime 11.0.x, and the suite green locally on ODIN01 (Windows);
  - all 250 tests pass under the new names;
  - 0 warnings with analyzers, nullable and docs enforced;
  - WI-7's reformat and `quality.yml` merged. The CSharpier result (AC-120) is recorded and does not hold Gate 0 (AD-65; amended 2026-11-23 (Gate 2, fourth review): was "CSharpier clean").

**Phase 1 — Fan-out (lanes in parallel, each starting at Gate 0):**

| Lane | Order inside the lane | Notes |
| --- | --- | --- |
| A — EdDSA | WI-11 → WI-12; WI-16 → WI-17; WI-13 → WI-14 → WI-15 (after WI-12 and WI-17); WI-18 → WI-19 (after WI-17) | Tests items WI-11, 13/14, 16 and 18 can all be written at once. The implementations serialize on shared files: WI-17 before WI-19 (`EdDsa.cs`); WI-12 and WI-17 before WI-15. |
| B — Encoders | WI-20 → WI-21; WI-22 → WI-23; WI-24 → WI-25 | Three independent chains. |
| C — AspNetCore | WI-26 → WI-27 → WI-28; WI-29 → WI-30 → WI-31 (after WI-28) | WI-28 and WI-31 serialize on `ExtendedPasswordValidator.cs`. |
| D — Sweep | WI-32 at lane start → WI-33 last | WI-33 waits for every library implementation item. |
| E — Sample | WI-34 → WI-36; WI-34 → WI-35 (after WI-31); WI-36 → WI-37 | WI-37 (Bootstrap 5 CSS and JS through LibMan, 1.0.0 scope) does not wait for lane C. WI-35 is `1.x`-only: scheduled here, but it holds neither Gate 1 nor the release (AD-58; amended 2026-11-23 (Gate 2)). |
| F — Measurement | WI-38 → WI-57 | The harness is built early so WI-43 only runs it. WI-57 writes chain C14's tests against WI-38's header-only files; WI-43 implements C14 in Phase 2 (amended 2026-11-23 (Gate 2, third review)). |
| G — Automation | WI-44 (start of Phase 1) → WI-60; WI-54 (after WI-3, WI-53, WI-56 to WI-59 and WI-61); WI-39 (after WI-37 and WI-61); WI-51; WI-62 → WI-63 → WI-64 (after WI-3) | D-2 is decided (GP-10), so WI-44 starts at Gate 0. Its FC-1 trial push and restore finish early in Phase 1, so a feed problem surfaces long before the release gate; it waits only for U-5's read credential. _(Amended 2026-11-23 (AM-1/AM-2/Q-30): was "well before 2026-11-10"; WI-54 added.)_ _Amended 2026-11-23 (Gate 2, third review): chains C16 to C19 added (AD-60). WI-60 writes C17's tests after WI-44; WI-46 implements C17._ |

- **Gate 1:**
  - every chain's tests green, and the full suite green, except that a `1.x`-marked test or a `1.x`-only item (WI-35) does not hold the gate; their results are recorded (AD-58; amended 2026-11-23 (Gate 2));
  - 0 warnings;
  - a Security Reviewer pass over the four security controls: WI-12 (S1), WI-15 (S2), WI-17 (S3, S5, S11) and WI-21 (S4). Findings go back to the item's owner, and the reviewer re-checks. _Amended 2026-11-23 (Gate 2, third review; AD-62):_ the same pass covers WI-61's read-only repository-security tests, and the format of their evidence.

  This review is an orchestration gate, not a work item.

**Phase 2 — Release:**
- WI-40, WI-42 and WI-43 run in parallel. _(Amended 2026-11-23 (AM-3/AM-4): WI-41, the migration guide, is cut.)_
- Then WI-55 sets the CHANGELOG date; its commit is the release commit (AD-57; added 2026-11-23 (Gate 2, F-4)).
- `1.x` checks never hold this phase back: they run and are recorded, in `release-gate.yml` as record-only and in `quality.yml`, which is not a gate (AD-56; added 2026-11-23 (Gate 2, F-1)). _Amended 2026-11-23 (Gate 2; AD-58 supersedes AD-56):_ the same holds for `1.x`-marked tests, which `dotnetcore.yml` and `release.yml` run in a separate recorded step, and for U-6's `1.x`-scope audit. WI-46 records all of them with the release evidence.
- Then **WI-46, the point of no return.** Publishing 1.0.0 is irreversible in practice, because the version is immutable once consumers restore it. WI-46 is gated by:
  - the non-workflow gates passing and each posted as a `success` commit status on the release commit: `release-gate/ac-20`, `release-gate/ac-119` and `release-gate/test-first-audit` (AD-55, AC-144; amended 2026-11-23 (Gate 2));
  - the three workflow-run gates on the release commit concluding `success`: `dotnetcore.yml`, `codeql-analysis.yml` and `release-gate.yml` (AD-55, AC-144);
  - `release.yml`'s `verify-gates` re-checking both kinds before it publishes;
  - U-2 to U-5 complete (U-1 is complete);
  - Fred pushing the tag.
- _Amended 2026-11-23 (AM-3/AM-4):_ WI-45 (the release-candidate dry run against assetvault-io) no longer precedes WI-46, and WI-47 no longer follows it. Both are cut (AD-51).

**Phase 3 — 1.x, after release and before this spec is done:** WI-48 → WI-49 and WI-50; WI-52. WI-51 may land any time after WI-3 and WI-7 (amended 2026-11-23 (Gate 2): it adds its gate to `quality.yml`, which WI-7 creates).

**Phase 4 — 1.1.0, the release that completes this spec** _(added 2026-11-23 (Gate 2, third review); AD-64; R-2 and R-50 as amended)_. It runs WI-65 → WI-66 → WI-67 → WI-68, after every `1.x`-only item has landed and every failing entry of WI-46's `1.x` record is fixed. If WI-7 took its fallback, the one reformat commit lands first, before WI-65 (AD-65; amended 2026-11-23 (Gate 2, fourth review)). **WI-68 is the second point of no return.** It is gated like WI-46, with no exemption: the full suite in the blocking step, no record-only checks, a single-scope audit, and `quality.yml` green.

**Critical path** _(re-derived 2026-11-23 (AM-3/AM-4); amended 2026-11-23 (Gate 2, F-1): WI-53 now precedes WI-1)_. WI-53 → WI-1 → WI-2 → WI-4 → WI-5/6 → WI-7 → WI-8 → WI-9 → WI-16 → WI-17 → WI-19 → WI-33 → WI-43 → WI-55 → WI-46 (with WI-54's `release-gate.yml` run inside it). _(Amended 2026-11-23 (Gate 2, F-4): WI-55 added before WI-46.)_ The new tests items, WI-57 to WI-61 and WI-63, and the audit tool, WI-62 to WI-64, run beside this path and do not lengthen it. _Added 2026-11-23 (Gate 2, third review):_ the path to the spec's completion continues WI-46 → WI-50 or WI-52 → WI-65 → WI-66 → WI-67 → WI-68. WI-50 and WI-52 wait for WI-46, and they are the longest `1.x`-only items after it.
- Lane A is the longest because `EdDsa.cs` hosts three serialized implementations (C5, C6 and C4's dependency on C5).
- Lane C (WI-27 → WI-28 → WI-31) is the nearest rival, also feeding WI-33.
- After WI-33, WI-40 (CHANGELOG), WI-42 (icon) and WI-43 (measurement) run in parallel. WI-43 is on the path because it runs the full set T on Fred's workstation against the release candidate; WI-40 is shorter. The cut WI-41 → WI-45 leg, which used to follow WI-33, is gone.
- Off the code path, **U-5 (feed credentials) gates WI-44's trial restore, and U-2 to U-4 gate WI-46** (U-1 is complete). If any Fred-only item lands late, it becomes the critical path. That is why they start at Gate 2, not in Phase 2. With no date (AM-2), a late item delays 1.0.0, and with it G-1: ValhallaTech's .NET 11 projects cannot consume the packages until 1.0.0 ships (§2.2.1, AD-52).

**Parallel capacity.** After Gate 0, up to seven lanes run at once:
- tests engineers on WI-11, 14, 16, 18, 20, 22, 24, 27, 30, 32, 34 and 36 (WI-56 lands before Gate 0; added 2026-11-23 (Gate 2));
- coding experts following each red tests item;
- the Frontend Expert on WI-37;
- CI on WI-39, 44, 51 and 54;
- docs on WI-10 (WI-55 runs last, on the release commit; amended 2026-11-23 (Gate 2, F-4)).

_Amended 2026-11-23 (AM-2; AD-42, now AD-52): was "The deadline is fixed (C-20) … Fred decides which of C-3, C-5 and C-20 gives."_ There is no date (C-20 as amended; C-3 is superseded). 1.0.0 is published when WI-46's gates pass. If a gate cannot pass as scoped, the PRD has nothing left to move out (§2.2.1), so the release is held and Fred decides what gives: C-5, C-20's complete-release rule, the scope, or the failing condition. No item is silently dropped.

### 3.6 Risks & mitigations

| ID | Risk | Likelihood | Impact | Mitigation | Affects |
| --- | --- | --- | --- | --- | --- |
| RK-1 | _Amended 2026-11-23 (AM-2): was "does not fit before 2026-11-10"; the date is gone. Amended 2026-11-23 (AM-3/AM-4): the assetvault-io exposure reason is gone (NG-12)._ The 1.0.0 scope (39 requirements, about 47 active items before Phase 3) takes long enough that G-1 is delayed: ValhallaTech's .NET 11 projects cannot consume the packages until 1.0.0 ships (§2.2.1). | med | high | Serial Phase 0 kept to mechanical work; seven parallel lanes; 1.x items in Phase 3; Fred-only items started at Gate 2. If a gate cannot pass as scoped, the release is held and Fred chooses what gives (§2.2.1, AD-52). No agent drops scope. | all |
| RK-2 | Microsoft.IdentityModel 7 → 8 or BouncyCastle 2.4 → 2.7 changes behavior or APIs that the libraries or tests rely on (FC-3). | med | high | Done first, alone, in WI-1 and WI-2, with all 250 tests run on 8.x, on runtime 11.0.x, before any behavior change. A failure is a finding for WI-1's owner, not a reason to pin old versions (G-9 has no exception rule; A-14). | WI-1, WI-2 |
| RK-3 | The AwesomeAssertions swap is not mechanical (FC-4): 19 files, 157 `.Should()` calls, and 6→7 renames such as `BeEquivalentTo` strictness on byte arrays. | med | med | An isolated item (WI-2), driven by compiler errors. Rule: no test is weakened, and a construct with no equivalent is rewritten with xUnit `Assert`, stated in the PR. The test-name set must not change. | WI-2 |
| RK-4 | A per-key `CryptoProviderFactory` is expensive (it allocates caches) when keys are built per tenant or per request (C1), or the inference that new keys share `CryptoProviderFactory.Default` is wrong. | med | med | `CacheSignatureProviders = false` and a shared, stateless provider instance. AC-21 proves the inference in WI-11, and AC-25 builds 1,000 keys concurrently. If allocation is material, WI-12's owner reads the pinned Microsoft.IdentityModel source and reports before choosing anything else, because the shared-factory alternative weakens G-4. | WI-11, WI-12 |
| RK-5 | Consumers that validate EdDSA through JWKS `JsonWebKey`s, or that set `TokenValidationParameters.CryptoProviderFactory`, find EdDSA unsupported once no global registration exists. | med | med | The provider accepts converted `JsonWebKey`s (AC-29). The CHANGELOG documents conversion and the override rule (3.3.3, WI-40). _Amended 2026-11-23 (AM-3/AM-4): was "the migration guide … (WI-41)", and an assetvault-io note, both removed (AD-51, NG-12)._ | WI-12, WI-40 |
| RK-6 | The shipped Bootstrap bundle stops matching the npm tarball, so it drops out of set V and its 19 jQuery-detection mentions fail AC-137/AC-138. Causes: a bundling or minifying step, a jsDelivr byte difference (inferred not to happen), or a Renovate bump restored without a test run. Separately, the WI-36 tests depend on network access to `registry.npmjs.org`. | med | med | No transform touches `wwwroot/lib/` (WI-37). AC-143's hash test runs in CI on every PR, including Renovate's, keyed on `libman.json`, so drift fails the PR that caused it. Network-dependent tests are marked `Category=Network` and are never skipped; an outage fails loudly and is re-run, not muted. | WI-36, WI-37, WI-39 |
| RK-7 | Fred-only actions (U-2 to U-5) arrive late: private reporting, Dependabot alerts, the Renovate app, and the feed credentials needed for the FC-1 trial. U-1, the logo, is complete (2026-11-23). U-2 and U-3 were still disabled on 2026-11-23. | high | high | All are listed with the item each one gates, and they start in parallel with Phase 0. The FC-1 feed trial (WI-44) is scheduled early in Phase 1 so a feed problem surfaces while there is time. WI-46 halts on any missing one; AC-20 and AC-12 are never faked. | WI-42, WI-44, WI-46 |
| RK-8 | **Withdrawn 2026-11-23 (AM-3/AM-4).** No consumer shapes this work (NG-12), and the guide and dry run are cut (AD-51). The CHANGELOG still tells a consumer to raise lower direct references (WI-40). It read: "Raising minimums to latest stable makes consumers with lower direct references fail restore (NU1605). assetvault-io pins BouncyCastle 2.6.2." | — | — | No action left. | WI-40 |
| RK-9 | Repo-wide mechanical changes (rename, reformat, annotations) conflict with parallel lanes. | high (if parallel) | med | All three run in serial Phase 0 before fan-out. Lanes are split by file; the implementations that share `EdDsa.cs` or `ExtendedPasswordValidator.cs` are serialized by `depends-on`. | WI-5, WI-7, WI-9 |
| RK-10 | Squash merges or rebased history destroy the commit ordering AC-121 and AC-132 audit. | med | high | No squash merges for this spec (§3.4 conventions). Test commits carry `test(WI-n)` and the AC IDs, so U-6 can find them. | all tests items, U-6 |
| RK-11 | Measurement noise, or the measuring machine being unavailable or changing between baseline and after. | med | med | Baseline and after are measured in one BenchmarkDotNet process on one .NET 11 runtime, so the same session, machine and runtime are guaranteed (WI-38, AD-48). WI-43 runs on Fred's workstation. AC-70 is calibrated by `B`. A regression or overrun is reported, never retuned by the agent that wrote the code. | WI-38, WI-43 |
| RK-12 | _Amended 2026-11-23 (AM-1/AM-2/Q-30): was "fails on `net10.0`". Amended 2026-11-23 (AM-3/AM-4): `net11.0` is the only framework._ A chosen package is deprecated, or fails on `net11.0`, before release: `xunit` 2.x, `xunit.runner.visualstudio`, Moq or AwesomeAssertions; `Microsoft.Web.LibraryManager.Build` or the DryIoc 6.2.0 adapter; BenchmarkDotNet not recognizing `net11.0` (AC-46, AC-49). | low | med | Each item's done-when runs `--deprecated` and builds and runs on `net11.0`. Fallbacks: xunit.v3; a committed CSS plus an npm `package.json` that Renovate reads; reporting DryIoc to Fred, with no preview substituted; BenchmarkDotNet's in-process toolchain. | WI-2, WI-30, WI-35, WI-37, WI-38 |
| RK-13 | A new stable release of a dependency lands between implementation and release and fails AC-49 on the day, or a new SimpleBase release changes Base62 behavior or speed. | med | med | Renovate is active before release (WI-39). WI-46 re-runs AC-49 on the release date, and bumps go through a normal PR and CI. SimpleBase is unpinned (GP-10). A bump must pass the C7 suite (AC-33 to AC-36, AC-124 to AC-126) in CI. A bump after WI-43 re-runs the P-a and P-b benchmarks (AC-70, AC-127) before WI-46; a budget overrun is reported to Fred. | WI-39, WI-43, WI-46 |
| RK-16 | **Resolved at Gate 2, round 2.** AC-137 as first written could not pass: its search also hit the test source and `.git/`. §2 now closes the exclusion list with `.git/` and the one named test file, and checks that file separately. | — | — | No action left. WI-36 places the assertions at the excluded path, and that file meets AC-137's separate check. | WI-36 |
| RK-14 | **Operations: monitoring.** A defect in 1.0.0 surfaces only in consumers; the library emits no telemetry, by design (NG-6). | med | med | Post-release signals and where they are watched: CI on every `master` push (`ubuntu-latest`, coverage, CodeQL), with Windows covered by local runs on ODIN01; Dependabot alerts and the Renovate dashboard (U-3, WI-39; M-2, M-4, M-10); private vulnerability reports (U-2; M-9). No new signal needs building. _(Amended 2026-11-23 (AM-3/AM-4): "the M-1 adoption search" removed, since M-1 is cut, CUT-2.)_ | WI-3, WI-39, WI-51 |
| RK-15 | **Operations: rollback.** A defect found after WI-46 publishes 1.0.0. | low | high | Before WI-46, rollback is `git revert`. After WI-46, rollback is **forward-only**: the 1.0.0 version is immutable once restored. Consumers pin back to upstream 4.1.0, which stays on nuget.org, and a 1.0.1 ships through the same workflow. The point of no return is WI-46's step 4 (the tag push), gated as in §3.5. | WI-46 |
| RK-17 | **Withdrawn 2026-11-23 (AM-3/AM-4):** no conditional entries exist (AD-45), so the risk cannot occur. It read: _Added 2026-11-23 (Q-30)._ Renovate's NuGet manager does not extract both conditional `PackageVersion` entries of a framework-bound dependency, or offers an 11.x version to the `net10.0` entry. Then R-48 fails, or a merged bump breaks the `net10.0` restore. | med | med | WI-39's done-when checks that the dashboard lists JwtBearer twice and that no PR offers 11.x to the `net10.0` entry. `allowedVersions` rules bound each line (AD-35). WI-53's script fails any drift in CI and at the gate. If Renovate cannot see both entries, WI-39's owner reports it before choosing another layout, because a different layout changes AC-49's and AC-134's inventory. | WI-39, WI-53 |
| RK-18 | **Withdrawn 2026-11-23 (AM-3/AM-4):** there is one runtime. It read: _Added 2026-11-23 (AM-1)._ A test passes on one runtime and fails on the other, through a .NET 11 behavior change in the runtime, BouncyCastle or Microsoft.IdentityModel. | low | med | Every library test runs on both runtimes on every PR (AD-37). A test may not be excluded from one framework, since AC-4 requires equal test-name sets (§3.4 conventions). A framework-only failure goes back to the owning implementation item. | all implementation items |
| RK-19 | **Resolved 2026-11-23 (AM-3).** Fred answered G2-1 with "Drop .NET 10."; §1 and §2 went back through `/spec`, as this row foresaw, and AD-43 records the answer. It read: _Added 2026-11-23 (C-25)._ Fred does not accept AD-33's reading of C# 15 at Gate 2 (G2-1). | — | — | No action left. | WI-1 |
| RK-20 | _Added 2026-11-23 (AM-3/AM-4)._ A direct dependency's latest stable version stops supporting `net11.0` before release, for example ASP.NET Core packages once a .NET 12 line ships with only `net12.0` assets. R-17 then cannot be met for it (A-14). Renovate's PR for that version fails restore. | low | high | Renovate's failing PR is the early signal and is not merged. WI-53's script reports the case as A-14 at the gate, never as a pass, and the release is held for Fred's ruling (A-14, AD-46). No agent pins, overrides or excludes the dependency. | WI-39, WI-53, WI-46 |
| RK-21 | _Added 2026-11-23 (Gate 2, AD-58)._ The split test run goes wrong. The filter `Release!=1.x` may drop tests that have no `Release` trait, which is inferred, not verified: the VSTest filter docs list `!=` and xUnit traits as filterable, but say nothing about a missing property. A dropped test would never run, and the blocking step would be silently green. A second way is a `1.0.0` test marked `1.x` by mistake, which would hide a `1.0.0` failure. | low | high | WI-3's split probe proves that the blocking and `1.x` test sets are disjoint and together equal an unfiltered run's set, before any test is marked. If they are not, WI-3's owner switches the blocking step to one unfiltered run plus a check that fails on any failed test absent from the `1.x` run's TRX, and reports it. The misuse check (WI-56) blocks in CI and gates `release-gate.yml` (WI-54), so a marker that lists a `1.0.0` requirement cannot reach the release commit. A marker whose requirement trait names the wrong `1.x` requirement passes that check, so the review of each tests item compares every marked test's requirement trait with its AC's `Verifies` line. | WI-3, WI-44, WI-54, WI-56 |
| RK-22 | _Added 2026-11-23 (Gate 2, third review)._ Two inferred facts about the feed tests (C17) turn out false. First, NuGet may not expand `%IRONSLOTH_PACKAGES_READ_TOKEN%` in `packageSourceCredentials`. Second, GitHub Packages' NuGet registration resource may not return a `published` timestamp. Then WI-60's tests error, or AC-8 has no timestamp to read. | med | med | WI-60 proves both before writing its tests. If expansion fails, the test passes the credential through `dotnet restore`'s environment-based credential provider, with no token on disk, and the owner reports which. If `published` is missing, AC-8's test reads `created_at` from GitHub's REST endpoint for the package's versions, and the owner reports it, since AC-8 names "published timestamp". | WI-60, WI-46, WI-68 |

**Operations:** monitoring: RK-14; backup: not applicable, since no step destroys or rewrites data (no database, cache or persisted state, §3.2; the repository rename is a `git mv` recorded in history); rollback: RK-15. _Amended 2026-11-23 (Gate 2, third review):_ RK-15's forward-only rule applies to WI-68's 1.1.0 publish as well as to WI-46's 1.0.0 publish; each is a point of no return gated in §3.5.

_M-7 (added 2026-11-23, Q-27; amended 2026-11-23 (AM-3/AM-4)):_ the only shipped framework, .NET 11 (STS), reaches end of support on 2028-11-14 (§3.1 grounding). M-7 stays at 0 until then. Microsoft's release metadata is where a change to that date would show; Renovate's `global.json` updates stay within 11.0 (AD-45).

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
| C14 | WI-38 (stubs) → WI-57 → WI-43 | `build/tests/Measurement.Tests.ps1 -ReportDirectory perf/results/1.0.0` over the 3.3.5 files `report.csv` (AC-60 to AC-68, AC-127), `base16-scaling.csv` (AC-69) and `base62-budget.csv` (AC-70), with their 3.3.5 columns |
| C15 | WI-58 → WI-3 | `build/tests/CiRun.Tests.ps1 -Sha <sha>`: `gh run list --commit <sha> --workflow dotnetcore.yml`, `gh run view <id> --json jobs`, and the `TestResults/Blocking/` and `TestResults/OneX/` TRX files from `gh run download <id>` |
| C16 | WI-59 → WI-54 | `build/tests/ReleaseGates.Tests.ps1 -Sha <sha> [-PublishedAt <P>]`: `gh run list --commit <sha> --workflow <file> --json conclusion,status,updatedAt` for the three workflows; `GET /repos/IronSloth/IdentityModel-Supa/commits/<sha>/statuses?per_page=100`, all pages |
| C17 | WI-44 (stubs) → WI-60 → WI-46 | `build/tests/Feed.Tests.ps1 -Source <feed> -Version <v>`: the feed's NuGet V3 service index and registration resource (AC-8); the copied `build/fixtures/feed-consumer/` console's `dotnet restore`, `build` and `run` exit codes and output (AC-9), and its `obj/project.assets.json` (AC-145) |
| C18 | WI-61 → WI-39 | `build/tests/RepositorySecurity.Tests.ps1`: `GET /repos/IronSloth/IdentityModel-Supa/private-vulnerability-reporting` (AC-20), `GET /repos/IronSloth/IdentityModel-Supa/vulnerability-alerts` (AC-119), and `gh pr list` / `gh issue list --author app/renovate --state all --json createdAt` against the commit that added `renovate.json` (AC-118). GET only (AD-62) |
| C19 | WI-62 (stubs) → WI-63 → WI-64 | `build/Test-TestFirstHistory.ps1 -HeadCommit <sha>` and its `test-first-audit.json`, read by `build/tests/TestFirstHistory.Tests.ps1`: fixture git repositories built at test time, and the repository's history `555d31c..<sha>` (tag `Release`) |

_Amended 2026-11-23 (Gate 2, third review; AD-60): rows C14 to C19 are added. The sentence that stood here is withdrawn. It read: "ACs without a chain are `build`, `lint`, out-of-process `integration` (CI runs, feed, GitHub API, measurement, scratch consumers), or Fred-only. The two ACs added on 2026-11-23, AC-144 and AC-145, are out-of-process `integration` checks of workflow runs and of consumer restores from the feed, so they get no chain."_ Every AC observable below the browser now has a chain, and every chain has a named seam. ACs without a chain are `build`, `lint`, `typecheck` or `manual` only.

**Tests exempt from the green check after implementation** _(added 2026-11-23 (Gate 2, third review); AD-60)_. Each test below is red by assertion at its tests item and stays red, or depends on state, after its chain's implementation item. Each is named here with its reason and the item that turns it green, and none is dropped.

| Chain | Test (`It` name prefix, tag) | Class | Why it cannot be green at the implementation item | Turned green by |
| --- | --- | --- | --- | --- |
| C14 | AC-60 to AC-67, AC-69 | `1.x` record-only (AD-58) | These verify R-23 and R-24, which ship in `1.x`. A miss is recorded and does not fail 1.0.0. | WI-43, or a `1.x` fix before WI-66; must pass at WI-68 |
| C16 | `Statuses` (AC-144) | Release check | The three statuses are posted only on the release commit, after U-2, U-3 and U-6 pass | WI-46 step 1; WI-68 |
| C16 | `Post` (AC-144) | Release check | P exists only after publication | WI-46 step 5; WI-68 |
| C17 | AC-8, AC-9, AC-145 | Release check | The version is on the feed only after publication. WI-60's local-folder harness run and WI-46's step 2 pre-flight prove the tests earlier, but are not the AC. | WI-46 step 5 (1.0.0); WI-68 (1.1.0) |
| C18 | AC-20 (`Admin`) | Fred-state check | Private vulnerability reporting is a setting only Fred can enable (U-2, C-21). It is exempt from the red and the green check, because Fred may act at any time. | U-2; must pass at WI-46 step 1 |
| C18 | AC-119 (`Admin`) | Fred-state check | Dependabot alerts are a setting only Fred can enable (U-3) | U-3; must pass at WI-46 step 1 |
| C18 | AC-118 | Fred-state check, until U-4 | Renovate acts only once its app is installed (U-4). WI-39's merge is still needed for the test to pass. | WI-39 with U-4; must pass at WI-46 step 2 (`release-gate.yml`) |
| C19 | `Release` (AC-121, AC-132) | Release check | The live audit can pass only when every chain C1 to C13b has landed | U-6's run on the release commit (WI-46; WI-68) |

**Seams on one framework.** _Added 2026-11-23 (AM-1/AM-2/Q-30) as "Seams on both frameworks"; amended 2026-11-23 (AM-3/AM-4)._ The seams above are unchanged. Every chain runs on `net11.0` and runtime 11.0.x, so "red for the right reason" and every guard are judged on that one run. The guards are unchanged: AC-24, AC-32, AC-40, AC-41, AC-72, AC-73, AC-77, AC-82, AC-90, AC-110, AC-112, AC-135 and AC-141, plus the C11 and C12 replacement tests named in §3.4. Each passes against its chain's stubs because the behavior it checks already exists at `555d31c` (AC-121 gives each reason). No new guard is introduced. Chains C14 to C19 have no guards; their exempt tests are listed in the table above (amended 2026-11-23 (Gate 2, third review)).

**AC → check.** Levels follow the PM's `Executable as` throughout; none is changed. Key to "Where it runs":
- **CI** means `dotnet test` in the single `ubuntu-latest` job (D-10), which runs every test on runtime 11.0.x (AD-47; amended 2026-11-23 (AM-3/AM-4)). It runs in two steps (AD-58; amended 2026-11-23 (Gate 2)). The blocking step runs every unmarked test and fails the job on any failure. The recorded step runs every test marked `[Trait("Release", "1.x")]`, and its failures are written to the job summary and never fail the job.
- **CI (1.x, recorded)** means the recorded step. A row's tests run there when every requirement they verify is `1.x`. They block from the release that carries their requirement, and at the release that completes this spec (§2.3 convention). _Added 2026-11-23 (Gate 2). Amended 2026-11-23 (Gate 2, third review; R-2, AD-64): "commit" became "release"; that release is 1.1.0, where WI-65 has removed every marker._
- **CI-u** means a step that runs once in that same job, after the tests. The label is kept so existing rows still read correctly. Only 1.0.0 checks run there (AD-56; amended 2026-11-23 (Gate 2, F-1)).
- **`quality.yml`** means the separate workflow for the CI checks of `1.x` requirements, created by WI-7. It runs on every push and PR, and is not one of AC-144's gate runs (AD-56; added 2026-11-23 (Gate 2, F-1)). Where a row below cites AD-56, AD-58 now governs. It supersedes AD-56, keeps every record-only check AD-56 named, and widens the rule to every `1.x`-only AC (amended 2026-11-23 (Gate 2)). AD-65 has since superseded AD-58, changing only its point (8), on WI-7's sequencing. So a row that cites AD-56 or AD-58 is governed by AD-65 (amended 2026-11-23 (Gate 2, fourth review)).
- **Gate** means the `release-gate.yml` run on the release commit (WI-54), one of the three workflow-run gates (AD-55), except AC-20 and AC-119, which WI-46 runs locally with administrator scope and records as the commit statuses `release-gate/ac-20` and `release-gate/ac-119`. _Amended 2026-11-23 (AM-1/AM-2/Q-30): was "WI-46's release-gate run". Amended 2026-11-23 (Gate 2): the commit statuses are added._
- **Bench** means WI-43 on Fred's workstation, on one .NET 11 runtime (AD-48).
- **Post** means WI-46's checks after publish. _Added 2026-11-23._

| AC | Check | Level | Where it runs |
| --- | --- | --- | --- |
| AC-1, AC-10, AC-11, AC-16, AC-17 | Unzip each `.nupkg`; read `lib/` folders and the nuspec; byte-compare the packed `LICENSE` | build | CI-u (pack step), Gate |
| AC-2 | `dotnet msbuild -getProperty:TargetFramework -getProperty:TargetFrameworks` on every test project and the sample: {`net11.0`} for each (amended 2026-11-23 (AM-3/AM-4); was {`net10.0`, `net11.0`} for the library test project) | build | CI-u, Gate |
| AC-3, AC-4, AC-57 | With the 11.0 SDK that `global.json` selects, full-solution `dotnet build -c Release --no-incremental` at the repository root (0 warnings), then `dotnet test` with one TRX per test project: failed and skipped both 0; every run on runtime 11.0.x. AC-3 and AC-4 first close in WI-2; AC-57 in WI-9. _Amended 2026-11-23 (Gate 2; AC-4 as amended, AD-58):_ the test run is WI-3's two steps, and AC-4 reads the uploaded TRX files, not the exit code. (1) Every run is on runtime 11.0.x, with skipped == 0. (2) The blocking TRX files show failed == 0. (3) Every `1.x`-marked test appears in the `1.x` TRX files as passed or failed, and its failures go into WI-46's `1.x` record. (4) The misuse count is 0 (WI-56's `ReleaseScopeTraitTests`, blocking). (5) At the release that completes this spec, 1.1.0, every test runs in the blocking step with no marker left, and failed == 0 (WI-65, WI-68). _Amended 2026-11-23 (Gate 2, third review; R-2 as amended, F-6):_ (5) read "at the commit that completes this spec". At 1.0.0, R-2 lets marked tests fail: they run and are recorded in WI-46's `1.x` record, and they need not pass. _Amended 2026-11-23 (AM-1/AM-2/Q-30): runs on 10.0.x and 11.0.x. Amended 2026-11-23 (AM-3/AM-4): 11.0.x only. Originally a 10.0 SDK and one run on 10.0.x._ | build | CI, Gate (via the CI run) |
| AC-5 | Diff test FQN sets (`555d31c` vs release, under the 3.3.1 mapping) against the §3.4 removed-test table | lint | Gate |
| AC-6, AC-58 | `yq` over `.github/workflows/*.yml`, `quality.yml` included (every `dotnet-version` line, and the SDK of any `global-json-file`, against AC-6's pattern). AC-6 gates; AC-58 (R-21, `1.x`) is record-only (AD-56; amended 2026-11-23 (Gate 2, F-1)) | lint | Gate (and on review of WI-3, WI-7, WI-44 and WI-54) |
| AC-7 | `gh run view <id> --json jobs`: every job `success`. The uploaded TRX files show every test project on runtime 11.0.x, and failed == 0 in the blocking TRX files (amended 2026-11-23 (AM-3/AM-4)). _Amended 2026-11-23 (Gate 2; AC-7 as amended, AD-58):_ every `1.x`-marked test appears in the `1.x` TRX files with an outcome. A failed one, or a failed `1.x`-only check such as AC-58 or AC-120, cannot turn the job red, because the recorded step exits 0 and those checks run in `release-gate.yml` as record-only or in `quality.yml`. The misuse count is 0. _Amended 2026-11-23 (Gate 2, third review; R-2 as amended, F-6; AD-60):_ chain C15, `CiRun.Tests.ps1` (WI-58), run by `release-gate.yml` with `Sha` set to the release commit. At 1.0.0, R-2 exempts only marked tests. At 1.1.0 there are no markers, so every test must pass (WI-68) | integration | Gate (CI run of the release commit) |
| AC-8 | Feed metadata query: both IDs at exactly `1.0.0` with a published timestamp. No date bound (amended 2026-11-23, AM-2). _Amended 2026-11-23 (Gate 2, third review):_ chain C17, `Feed.Tests.ps1` tag `FeedMetadata` (WI-60), a release check | integration | Post (WI-46 step 5) |
| AC-9, AC-145 | One scratch `net11.0` console restoring both packages from the feed only, then building and running the AC-9 program. AC-145 reads its `obj/project.assets.json`: every `compile` and `runtime` asset of both packages is under `lib/net11.0/`. (Amended 2026-11-23 (AM-1/AM-2/Q-30): two consoles. Amended 2026-11-23 (AM-3/AM-4): one `net11.0` console.) _Amended 2026-11-23 (Gate 2, third review):_ chain C17, `Feed.Tests.ps1` (WI-60), with the AD-63 consumer outside the repository; release checks, pre-flighted against a local folder at WI-46 step 2 | integration | Post (WI-46 step 5); pre-flight at WI-46 step 2 |
| AC-144 | P = the earlier of the two packages' published timestamps (AC-8). **Workflow-run gates** (`dotnetcore.yml`, `codeql-analysis.yml`, `release-gate.yml`): `gh run list --commit <sha> --workflow <file> --json conclusion,updatedAt`, each `success` with `updatedAt` < P. **Non-workflow gates** (`release-gate/ac-20`, `release-gate/ac-119`, `release-gate/test-first-audit`): `GET /repos/IronSloth/IdentityModel-Supa/commits/<sha>/statuses`, all pages; for each context, the newest status with `created_at` < P is `success`, and a context with no status fails. Each status's `description` is `evidence: <yyyy-mm-dd>/<log file name>`. Enforced before publish by `release.yml`'s `verify-gates` (WI-44), and verified after publish (WI-46 step 5). (Added 2026-11-23; amended 2026-11-23 (Gate 2), AD-55.) _Amended 2026-11-23 (Gate 2, third review):_ chain C16, `ReleaseGates.Tests.ps1` (WI-59). `verify-gates` runs it excluding `Post`, and WI-46 step 5 runs it with `Post` and `PublishedAt` = P. Its `Statuses` and `Post` tests are release checks | integration | `verify-gates`, then Post |
| AC-12 | PNG signature, IHDR size, SHA-256 against the supplied and the `555d31c` icon | build | Gate |
| AC-13, AC-14, AC-95, AC-96 | C1 reflection tests | unit | CI |
| AC-15, AC-18 | `rg -i -uu "scottbrady"`, and the notice-string search per 3.3.5 | lint | Gate |
| AC-19 | `rg` on `SECURITY.md` | lint | Gate |
| AC-20 | `gh api /repos/IronSloth/IdentityModel-Supa/private-vulnerability-reporting`; on pass, WI-46 posts the `release-gate/ac-20` status (amended 2026-11-23 (Gate 2)). _Amended 2026-11-23 (Gate 2, third review):_ chain C18, `RepositorySecurity.Tests.ps1` tag `Admin` (WI-61), GET only (AD-62); a Fred-state check | integration | WI-46 locally (after U-2) |
| AC-21, AC-23, AC-25, AC-32, AC-81 | C3 tests in the `GlobalCryptoState` collection | unit | CI |
| AC-22, AC-24 | C3 in-process JWT create and validate | integration | CI |
| AC-26 to AC-28, AC-82, AC-93, AC-94 | C4 tests | unit | CI |
| AC-29 | C4 JWT validated with a public-only JWK | integration | CI |
| AC-30, AC-31, AC-42 to AC-44, AC-73, AC-91, AC-92, AC-128, AC-129, AC-135 | C5 tests | unit | CI |
| AC-87 | C5 test, marked `1.x` with `Requirements` `R-32` (amended 2026-11-23 (Gate 2), AD-58: was in the row above) | unit | CI (1.x, recorded) |
| AC-88 | C5 JWT validated with a private-only key, marked `1.x` with `Requirements` `R-32` (amended 2026-11-23 (Gate 2), AD-58) | integration | CI (1.x, recorded) |
| AC-33 to AC-36, AC-79, AC-124 to AC-126 | C7 tests | unit | CI |
| AC-37 to AC-41 | C8 tests | unit | CI |
| AC-45, AC-46 | `dotnet list package --vulnerable` and `--deprecated --include-transitive`; NuGetAudit already fails restore on High or Critical | build | CI (restore), Gate |
| AC-47 | `rg -uu "SecurityCodeScan" -g "*.csproj" -g "*.props" -g "*.targets"` | lint | Gate |
| AC-48 | Read each direct package's nuspec `<license type="expression">` against the OSI list | lint | Gate |
| AC-49 | `build/Test-RequiredVersions.ps1` (WI-53, AD-46): every (project, direct dependency) resolved version equals its latest stable version from the NuGet registration API, on the release date; a latest stable that does not support `net11.0` fails as an A-14 case. _Amended 2026-11-23 (Q-30): a per-framework required version. Amended 2026-11-23 (AM-3/AM-4): one latest stable version. Originally `dotnet list package --outdated`._ | lint | Gate (needs `api.nuget.org`) |
| AC-50, AC-51, AC-133, AC-139, AC-140 | Needs `registry.npmjs.org` (AC-133, AC-139, AC-140) and the GitHub Advisory Database (AC-50). Without network these fail loudly, naming the URL; they are never skipped. Steps: `libman.json` library and version (the sample's only client-library inventory) → `gh api /advisories?affects=bootstrap@<v>`, Bootstrap's support status, npm `dist-tags.latest`, npm `dependencies`/`peerDependencies` | lint | Gate |
| AC-52 to AC-54, AC-112 | C13a in-process host tests, marked `1.x` with `Requirements` `R-19` (AC-52 to AC-54) or `R-45` (AC-112) (amended 2026-11-23 (Gate 2), AD-58) | integration | CI (1.x, recorded) |
| AC-55, AC-99 | Probe copy (CA5351 line; undocumented public method) built `-c Release` | build | Gate (and once in WI-9) |
| AC-56 | `dotnet msbuild -getProperty:NoWarn;RunAnalyzers;EnableNETAnalyzers;AnalysisMode;AnalysisLevel`, plus `rg` over `.editorconfig`/`.globalconfig` | lint | Gate |
| AC-59, AC-123 | Parse `perf/results/1.0.0/report.csv` (32 rows) and `report-ed448.csv` (6 rows): one 11.0.x runtime, one machine (amended 2026-11-23 (AM-3/AM-4); was 64 and 12 rows on two runtimes). AC-59 gates; AC-123 (R-52, `1.x`) is record-only (AD-56; amended 2026-11-23 (Gate 2, F-1)) | lint | Gate |
| AC-60 to AC-70, AC-127 | Compare rows under the measurement rules; AC-69 `base16-scaling.csv`; AC-70 `base62-budget.csv`, with the budget scaled by `B`. AC-68, AC-70 and AC-127 gate; AC-60 to AC-67 and AC-69 (R-23, R-24, `1.x`) are record-only (AD-56; amended 2026-11-23 (Gate 2, F-1)). _Amended 2026-11-23 (Gate 2, third review):_ chain C14, `Measurement.Tests.ps1` (WI-57), red against WI-38's header-only files and run over the committed files by WI-43 and the Gate | integration | Bench (measured); Gate (comparison re-run over the committed files) |
| AC-71, AC-72, AC-83, AC-86 | C9 tests | unit | CI |
| AC-74 to AC-77, AC-142 | C12 tests (DI container, generic host, tag helper) | integration | CI |
| AC-78, AC-80 | C10 sweep | unit | CI |
| AC-84, AC-85, AC-89, AC-90 | C6 tests | unit | CI |
| AC-97 | C2 test | unit | CI |
| AC-98 | Match public members to the built XML doc file | build | Gate |
| AC-100, AC-101 | `rg -i` per the AC on the README and package readmes | lint | Phase 3 review of WI-48 |
| AC-102 | `build/Test-DocSnippets.ps1`, compiling each block in one `net11.0` project (AD-49) | build | `quality.yml` (from WI-50; amended 2026-11-23 (Gate 2, F-1): was CI-u) |
| AC-103 | Nuspec `<readme>` and the packed file contains its package ID | build | `quality.yml` (WI-49 makes it pass; WI-50 adds the step; amended 2026-11-23 (Gate 2), AD-58: was CI-u, inside the `dotnetcore.yml` gate run, and R-41 is `1.x`) |
| AC-104, AC-105 | 3.3.5 phrase list over the `1.0.0` section; headings against tags and feed versions | lint | Gate |
| AC-106 to AC-108 | C11 tests, marked `1.x` with `Requirements` `R-43` (amended 2026-11-23 (Gate 2), AD-58: was in one row with AC-109 and AC-110) | unit | CI (1.x, recorded) |
| AC-109, AC-110 | C11 tests | unit | CI |
| AC-111 | C11 DI-resolved validator with a registered describer | integration | CI |
| AC-113, AC-122 | `rg` on the sample (and, for AC-122, the README). Both are `1.x` (R-45, R-51): AC-113 is record-only at the Gate (AD-56; amended 2026-11-23 (Gate 2, F-1)) | lint | Gate, record-only (sample); WI-48 review (README) |
| AC-114, AC-138, AC-141, AC-143 | C13b in-process host HTML and asset tests; SHA-256 against the npm tarball for set V and the Bootstrap bundle. The AC-114 tests are marked `1.x` with `Requirements` `R-45` (amended 2026-11-23 (Gate 2), AD-58) | integration | CI (needs network to `registry.npmjs.org`); AC-114 in CI (1.x, recorded) |
| AC-115 | Probe copy with `Base62.Encode(byte[])` removed, packed against the `eng/baseline/` 1.0.0 | build | WI-52, then CI-u |
| AC-116, AC-117 | Cobertura gate per library (AD-50); the 79.9% fixture must fail | build | `quality.yml` (from WI-51; AD-56; amended 2026-11-23 (Gate 2, F-1): was CI-u, inside the `dotnetcore.yml` gate run) |
| AC-118, AC-119 | AC-118: `gh pr list` / `gh issue list --author renovate[bot]`, in the Gate. AC-119: `gh api /repos/IronSloth/IdentityModel-Supa/vulnerability-alerts` returns 204, run by WI-46 locally, which then posts the `release-gate/ac-119` status (amended 2026-11-23 (Gate 2)). _Amended 2026-11-23 (Gate 2, third review):_ chain C18, `RepositorySecurity.Tests.ps1` (WI-61): AC-118 untagged, run in the Gate; AC-119 tag `Admin`, run locally. Both are Fred-state checks (U-4, U-3), GET only (AD-62) | integration | Gate (AC-118); WI-46 locally (AC-119) |
| AC-120 | `dotnet csharpier check .` on the release commit and on a probe. Record-only at the Gate (R-49, `1.x`; AD-56) | lint | `quality.yml`, Gate (record-only) (amended 2026-11-23 (Gate 2, F-1): was CI-u, Gate) |
| AC-121, AC-132 | History audit: check out each introducing commit and each implementing commit, and run the named tests. For AC-132, WI-28's tests are the AC-106 to AC-109 unit tests, and WI-31's are the 10 `ExtendedPasswordOptionsFirstUseTests` (`[Trait("FixedBy", "WI-31")]`, commit `test(WI-30): … [AC-132 → WI-31]`), each red at WI-30's commit and green at WI-31's and at release (amended 2026-11-23 (Gate 2, F-3)). On pass, WI-46 posts the `release-gate/test-first-audit` status from the audit's evidence log (amended 2026-11-23 (Gate 2)). _Amended 2026-11-23 (Gate 2; AC-121 and AC-132 as amended, AD-58):_ the log has a 1.0.0 scope and a `1.x` scope, each with its own verdict, and only the 1.0.0 verdict decides the status. **1.0.0 scope:** AC-121 for every in-scope AC not in `oneXAcs`; AC-132 for every behavior-changing work item that is not `1.x`-only. At 1.0.0, WI-28's test is the AC-109 unit test, and WI-31's are the 10 tests, which are unmarked. **`1.x` scope, recorded:** AC-121 for AC-52 to AC-54, AC-87, AC-88, AC-106 to AC-108, AC-112 and AC-114; AC-132 for WI-35. Each is audited if it has landed and listed as "not landed" if not, and goes into WI-46's `1.x` record. The misuse count is 0 (WI-56). At the release that completes this spec, both scopes must pass ("commit" became "release" on 2026-11-23 (Gate 2, third review)). _Amended 2026-11-23 (Gate 2, third review; R-50 as amended, F-6; AD-61):_ chain C19. The audit is `build/Test-TestFirstHistory.ps1` (WI-62 to WI-64), proven on fixture histories by `TestFirstHistory.Tests.ps1` (WI-63), and run by U-6 on the release commit as a release check. The completing release is 1.1.0 (WI-68), where the scope lists are empty and the tool returns one verdict, with no exemption | integration | U-6 (Agent Evaluator) |
| AC-130 | **Cut 2026-11-23 (AM-4; CUT-3).** Not checked. It was: the 3.3.5 API and topic phrase match over the guide, at the Gate. | — | — |
| AC-131 | **Cut 2026-11-23 (AM-4; CUT-4).** Not checked. It was: a scratch assetvault-io at `863c0cb`, built and tested, in WI-45 and WI-47 (both cut). | — | — |
| AC-134 | Renovate dependency dashboard set against the AC-49, AC-133 and `uses:` inventory | lint | Gate |
| AC-136 | Read the `Base62.Encode`/`Decode` `<member>` entries in the built XML doc file | build | Gate |
| AC-137 | The AC's four checks with its closed exclusion list (`bin/`, `obj/`, `.git/`, `specs/`, `CHANGELOG.md`, and `test/ValhallaTech.Security.IdentityModel.Samples.Tests/ForbiddenClientLibraryTests.cs`), plus the separate check on that file. Check 2 subtracts files whose SHA-256 matches a file in the npm tarball of a `libman.json` library (set V) at the declared version. Run after a build, so the restored `wwwroot/lib/` files are present. | lint | Gate (needs `registry.npmjs.org`) |

### 3.8 Domain tag summary

_The dispatch seam. One row per tag actually used._

| Tag | Work items | Suggested specialist |
| --- | --- | --- |
| `backend` (19) | WI-1, WI-5, WI-9, WI-12, WI-13, WI-15, WI-17, WI-19, WI-21, WI-23, WI-25, WI-26, WI-28, WI-29, WI-31, WI-33, WI-35, WI-42, WI-49 | C# Coding Expert |
| `tests` (26 active; WI-45 and WI-47 cut) | WI-2, WI-4, WI-6, WI-8, WI-11, WI-14, WI-16, WI-18, WI-20, WI-22, WI-24, WI-27, WI-30, WI-32, WI-34, WI-36, WI-38, WI-43, WI-56, WI-57, WI-58, WI-59, WI-60, WI-61, WI-63, WI-65; cut: WI-45, WI-47 | C# Unit Test Engineer. WI-34 and WI-36 are in-process host tests, not Playwright. WI-38 and WI-43 are the measurement harness and run. WI-56 is the `1.x` marker misuse check (added 2026-11-23 (Gate 2)). PowerShell CI Engineer for the Pester acceptance tests WI-57 to WI-61 and WI-63 (AD-59; added 2026-11-23 (Gate 2, third review)); C# Unit Test Engineer for WI-65. |
| `ci` (14) | WI-3, WI-7, WI-39, WI-44, WI-46, WI-50, WI-51, WI-52, WI-53, WI-54, WI-62, WI-64, WI-66, WI-68 | CI/CD Pipeline Engineer (WI-3, WI-39, WI-44, WI-46, WI-52, WI-54's workflow, WI-66 and WI-68); PowerShell CI Engineer (WI-50, WI-51 and WI-53 scripts, WI-54's `Invoke-ReleaseGate.ps1`, and the audit tool WI-62 and WI-64); WI-7 either. _WI-62, WI-64, WI-66 and WI-68 added 2026-11-23 (Gate 2, third review)._ |
| `docs` (5 active; WI-41 cut) | WI-10, WI-40, WI-48, WI-55, WI-67; cut: WI-41 | Documentation Agent |
| `frontend` (1) | WI-37 | Frontend Expert |

Unused tags: `database`, `redis`, `security`, `openfga` and `docker`. These libraries have no such artifacts (NG-6). The security controls (S1 to S5, S11) are library code, so they are tagged `backend` by artifact. The Security Reviewer reviews them at Gate 1 as a gate, not as an owner. Total: 68 tagged IDs (52 before 2026-11-23, plus WI-53 to WI-68), of which 65 are active and 3 are cut (WI-41, WI-45, WI-47; AM-4, AD-51), and 6 unmapped (U-1 to U-6; U-1 complete). _Amended 2026-11-23 (AM-3/AM-4). Amended 2026-11-23 (Gate 2, F-4): WI-55 added (AD-57). Amended 2026-11-23 (Gate 2): WI-56 added (AD-58). Amended 2026-11-23 (Gate 2, third review): WI-57 to WI-68 added (AD-59 to AD-64)._

**Track (sizing rule).** The plan needs many PRs and far more than 3 items (each TDD chain counted as one item: 20 chains, C1 to C12, C13a, C13b and C14 to C19, holding 46 items; plus 19 other active tagged items and 6 unmapped; 45 in all; cut items not counted; amended 2026-11-23 (Gate 2): was 14 chains of 31 items, 20 other items and 40 in all before WI-55 to WI-68, which also moved WI-3, WI-38, WI-39, WI-43, WI-44, WI-46 and WI-54 into chains), so the header's `Track` reads `full`.

#### D-item resolution ledger

§1.8 holds 11 D-items: all 11 are resolved by a §3.0 row whose Resolves cell names them, and none is raised. _Amended 2026-11-23: each outcome now cites its AD row._

| D-item | Outcome | Resolved by | Where |
| --- | --- | --- | --- |
| D-1 Assertion library | **Decided by Fred (2026-10-25): AwesomeAssertions**, latest stable (9.6.0 on 2026-10-25). | AD-1 | §3.4 WI-2 |
| D-2 Private NuGet feed | **Decided by Fred at Gate 2 (GP-10): GitHub Packages under IronSloth, `https://nuget.pkg.github.com/IronSloth/index.json`.** | AD-2 | §3.1, WI-44, WI-40, WI-48, U-5 |
| D-3 Base62 leading zeros | **Fixed by Fred (GP-2): SimpleBase `Base62.Default` with the cap.** Since GP-10 its version follows latest stable, with no pin. | AD-3 | §3.3.2 (`Base62` rows), WI-1, WI-21 |
| D-4 Private-export opt-in shape | **Decided:** a separate method, `ConvertFromEdDsaSecurityKeyIncludingPrivateKey`. | AD-4 | §3.3.2, WI-13 |
| D-5 64/114-byte private keys | **Decided:** rejected at validation. | AD-5 | §3.3.2 (`Validate` row), WI-17 |
| D-6 Measurement tool | **Decided:** BenchmarkDotNet, with a source-level `555d31c` baseline in the same process, on one .NET 11 runtime (AD-48, which supersedes AD-39 and AD-23). | AD-6 | §3.4 WI-38 |
| D-7 Update and format tools | **Decided:** Renovate (WI-39) for updates; CSharpier (WI-7) for formatting; GitHub Dependabot alerts for advisories (U-3). | AD-7, AD-8 | §3.4 WI-7, WI-39 |
| D-8 Sample DI and UI | **UI fixed by Fred (GP-8, GP-10):** Bootstrap 5 latest stable, CSS and JS bundle shipped unmodified, no jQuery, through LibMan with a Renovate regex manager over `libman.json`. **DryIoc decided by Fred (2026-10-25, "Yes, sample + one test"):** the sample runs on `DryIoc.Microsoft.DependencyInjection` with `DryIoc.dll` (6.2.0 and 5.4.3, their latest stable on 2026-10-25), plus one DryIoc test of the single step. | AD-9, AD-10 | §3.4 WI-35, WI-30, WI-37 |
| D-9 Solution format and build layout | **Decided, then amended twice on 2026-11-23 (AM-1, then AM-3):** `.slnx`; `global.json` pinned to the 11.0 SDK; `TargetFramework=net11.0` for every project in the root `Directory.Build.props`; no `LangVersion`, so all code is C# 15 by the framework default; a `src/` `Directory.Build.props` for library settings; central package management with one version per package; renamed directories. AD-11 records the 10.0-only layout and AD-32 the two-framework one. | AD-44 (supersedes AD-32, which superseded AD-11) | §3.3.4, WI-1, WI-2, WI-5 |
| D-10 CI matrix and CodeQL | **Decided by Fred (2026-10-27):** CI runs on `ubuntu-latest` only, following the label ("Ubuntu only (Recommended)", "Follow ubuntu-latest"). CodeQL stays. Windows is covered by local runs on ODIN01. | AD-12 | §3.4 WI-3 |
| D-11 Message localization mechanism | **Decided:** an `ExtendedIdentityErrorDescriber` subclass through `AddErrorDescriber<T>()`. | AD-13 | §3.3.2, WI-26 |

**Questions handed to the Architect on 2026-11-23** (not D-items; §1.6 owns them). _Amended 2026-11-23 (AM-3/AM-4)._
- **Q-26 (sample framework):** resolved by Fred's AM-3: every project, the sample included, targets `net11.0` (AD-34, AD-44).
- **Q-27 (.NET 11 support):** answered by fact check. .NET 11 is STS with end of support 2028-11-14, the same day as .NET 10 LTS, so A-17 holds (§3.1 grounding, read 2026-11-23 from Microsoft's release metadata and support policy page). _Amended 2026-11-23 (Gate 2):_ §1.6 and §2.6 now record it as resolved, and M-7 states the date.
- **Q-28 (C# 15):** resolved by Fred's AM-3 ("Drop .NET 10."). AD-43 records it and sets how C# 15 is reached: the `net11.0` default, with no `LangVersion`.

#### Raised for Gate 2

_§1.8 D-items the Architect could not decide, and choices that are Fred's; each is a question for Fred at Gate 2._

No D-item is raised, and no question is open. D-2, the only item raised at the first Gate 2, was decided by Fred (GP-10). G2-1, below, was answered by Fred on 2026-11-23 (AM-3) and is kept for the record.

- **G2-1 — Answered 2026-11-23 by Fred, AM-3: "Drop .NET 10." Resolved by AD-43, which supersedes AD-33.** Every project targets `net11.0`, all code is C# 15, and the language comes from the framework default with no `LangVersion` (AD-43 gives the reason). The question as raised follows.
- **G2-1 (as raised) — How does the C# 15 standard (C-25) apply to a repository that must also build for `net10.0`? (Q-28, A-18, FC-7; raised 2026-11-23.)**
  - **Why the plan cannot decide it.** A-18 assumed C# 15 applies to all of the repo's code at no cost to .NET 10 consumers, and says that if this is false, Fred chooses between the C# 15 standard in this repo and .NET 10 support. Microsoft says C# 15 is supported only on .NET 11 and newer, and that a language version newer than the target framework's default is unsupported. Union types also need `UnionAttribute` and `IUnion`, which ship in the .NET 11 runtime. *(Microsoft Learn, read 2026-11-23; §3.1 grounding.)* So C# 15 cannot be the language of the `net10.0` build without an unsupported configuration.
  - **Options.**
    1. **Recommended (AD-33):** each build uses its framework's default. Every `net11.0` compilation is C# 15, every `net10.0` compilation is C# 14, and source shared by both is written as C# 14, with C# 15 only behind `#if NET11_0_OR_GREATER` where both builds keep one API and one behavior. The `net11.0`-only sample and sample tests use C# 15 freely. Cost: the libraries' shared source cannot use C# 15 features such as union types or closed hierarchies in their public API. No cost to .NET 10 consumers, so FC-7 holds under this reading.
    2. `LangVersion=15` on every build. Cost: the `net10.0` package is built in a configuration Microsoft does not support, so .NET 10 consumers carry the risk of the "hard-to-diagnose compile-time and runtime errors" Microsoft warns of. FC-7 fails.
    3. Drop `net10.0`. Cost: breaks C-2 and C-26; assetvault-io cannot consume 1.0.0 without moving to .NET 11, against NG-11. §1 and §2 would need another amendment.
  - **What changes with the answer.** Option 1 is what the plan implements. Option 2 adds `LangVersion` to `src/Directory.Build.props` and the library test project in WI-1 and WI-2, and lifts the C# 14 rule. Option 3 sends the spec back through `/spec` from §1 (RK-19).

#### Gate 2 record

_The earlier Gate 2 approvals of this spec (2026-10-25 and later) predate this record._

- **2026-11-23: the AM-1 to AM-4 amendment.**
  - **Track:** full.
  - **Final readiness verdict:** PASS, with 0 security findings.
  - **Fred's decision:** Approve.
  - **Review history:** five readiness reviews, returning FAIL, CONCERNS, FAIL, CONCERNS and PASS. After each review except the last, the findings were repaired by their section owners before the next review.
  - **G2-1:** Fred answered "Drop .NET 10", which became AM-3.
