# Security Policy

## Supported Versions

The [latest released version](https://github.com/jquery/jquery/releases) of jQuery is supported.

## Reporting a Vulnerability

Before reporting, please check your finding against the [Threat Model](#threat-model) below. Reports that fall outside it are closed as out of scope.

Please report security issues **privately**:

- Email: security@jquery.com

**Do not** file public GitHub issues for security problems.

When reporting, please include:
- Affected project/repo and version(s)
- Impact and component(s) involved
- Reproduction steps or PoC (if available)
- Your contact and preferred credit name

If you do not receive an acknowledgement of your report within **6 business days**, or if you cannot find a private security contact for the project, you may **escalate to the OpenJS Foundation CNA** at `security@lists.openjsf.org`.

If the project acknowledges your report but does not provide any further response or engagement within **14 days**, escalation is also appropriate.

## Coordination & Disclosure

Important:
- If the vulnerability is considered valid and accepted, a patch will be made for the latest jQuery version.
- If the vulnerability is deemed invalid, no further action is required.

We follow coordinated vulnerability disclosure:
- We will acknowledge your report, assess impact, and work on a fix.
- We aim to provide status updates at reasonable intervals until resolution.
- We will publish a security advisory (and **CVE via the OpenJS CNA when applicable**) once a fix or mitigation is available. We credit reporters by default unless you request otherwise.

## Threat Model

jQuery is a DOM manipulation, traversal, event, and Ajax library that runs inside the web page that loads it. There is no privilege boundary between jQuery and the page: jQuery code runs with the same authority as any other script in the document. The threat model therefore defines what jQuery trusts and what it does not trust within that shared context.

This model covers the jQuery core library (`jquery/jquery`) only. jQuery UI and jQuery Mobile have their own repositories and support policies, and the jQuery websites and infrastructure are maintained in their own repositories under the [jquery organization](https://github.com/jquery).

For a vulnerability to be considered in scope, jQuery itself must violate its documented behavior when handling untrusted input, without assuming the compromise of a trusted element such as the browser, the page's own scripts, or the distribution channel. Reports must demonstrate the violation with a working proof of concept on a [supported version](#supported-versions).

### Elements jQuery Does Not Trust

1. **Data passed to APIs that are not documented to execute code.** Selector strings, text values, attribute and property values, data objects, serialized form data, and Ajax response bodies (unless the caller explicitly requests script execution, for example with `dataType: "script"` or `jQuery.globalEval`). *If such data can cause jQuery to execute script, modify `Object.prototype`, or otherwise act beyond the documented behavior of the API, that indicates a security vulnerability.*

2. **Input reaching jQuery's documented safety guarantees.** Some APIs document an explicit safety property: a string passed to `jQuery()` that does not look like HTML is treated as a selector and never as HTML, `jQuery.parseHTML` does not run scripts unless `keepScripts` is `true` and parses into an inert document when no context is given, and `.load( url selector )` strips scripts from the fetched fragment. *If untrusted input can bypass one of these documented guarantees, that indicates a security vulnerability.*

3. **The shape of plain objects it processes.** Operations such as `jQuery.extend( true, ... )` accept arbitrary object graphs. *If a crafted object (for example one carrying a `__proto__` key) can pollute built-in prototypes or corrupt unrelated state, that indicates a security vulnerability.*

### Elements jQuery Trusts

1. **The page and the application code that calls it.** The application is responsible for validating and sanitizing untrusted data before passing it to APIs that are documented to interpret their input as HTML or code. Any scenario that requires attacker-controlled script already running in the page is outside the model, since such an attacker already has full control of the document without jQuery.

2. **HTML strings passed to HTML-accepting APIs.** APIs documented to accept HTML, such as `jQuery( html )`, `.html()`, `.append()`, and the other manipulation methods, execute `<script>` elements and inline event handlers contained in that HTML by design. jQuery is not an HTML sanitizer. See the security notice below.

3. **The browser and its HTML parser.** jQuery assumes a correct, uncompromised browser, DOM implementation, and JavaScript engine. Parsing quirks and mutation behaviors inherent to the browser's own HTML parsing (for example `innerHTML` round-trip mutations) are browser characteristics, not jQuery vulnerabilities, unless jQuery's own processing introduces the unsafe transformation.

4. **Installed package integrity.** jQuery assumes the file delivered to the page (via npm, a CDN, or bundling) is the legitimate, unmodified distribution. Tampered packages, malicious forks, and compromised delivery infrastructure are supply chain issues, not jQuery vulnerabilities. CDNs continuing to serve older jQuery releases is intentional and is not a vulnerability.

5. **The privileges of the execution context.** jQuery inherits the origin, permissions, and capabilities of the document that loads it.

### Global configuration is a documented design property

jQuery maintains page-global state by design. `jQuery.ajaxSetup()` changes the defaults of every subsequent Ajax request, `jQuery.htmlPrefilter` rewrites every HTML string before parsing, and the `jQuery.fn` and `jQuery.expr` namespaces extend behavior for the whole page. These surfaces exist so that one call can reconfigure jQuery everywhere, and they are writable trusted code, not input. Routing untrusted data into them (for example calling `jQuery.ajaxSetup()` with attacker-influenced options) is an application decision, and consequences reachable only that way are out of scope. Genuine hardening gaps in how these surfaces merge or store values may still be fixed as robustness improvements rather than security advisories.

### Security notice: HTML strings execute code

> [!NOTE]
> Most jQuery APIs that accept HTML strings will run scripts included in the HTML, including inline event handlers such as `<img onerror>` once the markup is injected into a live document. This is documented, intended behavior, and passing untrusted input to these APIs is an application vulnerability, not a jQuery one. Callers must clean or escape untrusted input from sources such as the URL or cookies before building HTML with it. `jQuery.parseHTML` does not run scripts unless `keepScripts` is explicitly `true`, and callers should not depend on the ability to run any script content when `keepScripts` is unspecified or `false`. See the [`jQuery.parseHTML` documentation](https://api.jquery.com/jQuery.parseHTML/) for details.

### Examples of Vulnerabilities (in scope)

- **Cross-site scripting through non-HTML inputs ([CWE-79](https://cwe.mitre.org/data/definitions/79.html)).** If input documented as a selector, text, or data can reach an HTML-interpreting or script-executing path, that is a jQuery vulnerability. Prior examples: selector strings interpreted as HTML ([CVE-2011-4969](https://www.cve.org/CVERecord?id=CVE-2011-4969), [CVE-2012-6708](https://www.cve.org/CVERecord?id=CVE-2012-6708)).

- **Unsafe transformation of HTML during processing ([CWE-79](https://cwe.mitre.org/data/definitions/79.html)).** jQuery's own HTML preprocessing must not add execution paths the input did not contain. Prior examples: the `htmlPrefilter` self-closing tag rewrite ([CVE-2020-11022](https://github.com/jquery/jquery/security/advisories/GHSA-gxr4-xjj5-5px2)) and `<option>` fragment handling ([CVE-2020-11023](https://github.com/jquery/jquery/security/advisories/GHSA-jpcq-cgw6-v4j6)).

- **Bypass of a documented safety guarantee ([CWE-79](https://cwe.mitre.org/data/definitions/79.html), [CWE-94](https://cwe.mitre.org/data/definitions/94.html)).** Defeating the inertness of `jQuery.parseHTML` defaults, the script stripping of `.load( url selector )` ([CVE-2020-7656](https://www.cve.org/CVERecord?id=CVE-2020-7656)), or any similar documented guarantee.

- **Undocumented automatic script execution ([CWE-94](https://cwe.mitre.org/data/definitions/94.html)).** Executing response content the caller did not explicitly request as script. Prior example: cross-domain Ajax responses auto-executed as script without `dataType: "script"` ([CVE-2015-9251](https://www.cve.org/CVERecord?id=CVE-2015-9251)).

- **Prototype pollution ([CWE-1321](https://cwe.mitre.org/data/definitions/1321.html)).** Untrusted object graphs polluting `Object.prototype` through jQuery operations. Prior example: `jQuery.extend( true, ... )` ([CVE-2019-11358](https://www.cve.org/CVERecord?id=CVE-2019-11358)).

- **Undocumented cross-origin disclosure ([CWE-200](https://cwe.mitre.org/data/definitions/200.html)).** If jQuery exposes cross-origin response data to the page beyond what the browser's same-origin policy and CORS already grant, that is a jQuery vulnerability.

- **Denial of service through logic flaws ([CWE-400](https://cwe.mitre.org/data/definitions/400.html)).** Unbounded recursion, catastrophic regular expression backtracking, or hangs triggered by otherwise valid inputs within documented usage.

### Examples of Non-Vulnerabilities (out of scope)

#### Untrusted input passed to HTML-accepting APIs

Passing unsanitized, attacker-influenced strings to `jQuery( html )`, `.html()`, `.append()`, or similar APIs is an application vulnerability. These APIs are documented to interpret their input as HTML and to run the scripts it contains.

#### Reports without a working proof of concept

A report must demonstrate the claimed behavior on a supported jQuery version with a runnable proof of concept. Claims produced by scanners or AI tooling that do not reproduce, or that misdescribe jQuery's actual source or behavior, are not vulnerabilities.

#### Known issues in unsupported versions

Only the [latest released version](https://github.com/jquery/jquery/releases) is supported. Vulnerabilities that only affect older release lines, including already-published CVEs, are historical records, not new reports. Official CDNs continuing to serve older versions for compatibility is standard practice and is not a vulnerability.

#### Browser parsing behavior

Mutation and parsing differentials inherent to the browser's HTML parser (for example `innerHTML` serialization quirks) are characteristics of the platform jQuery trusts. They are in scope only when jQuery's own processing introduces the unsafe transformation.

#### Selector injection

Building selectors from untrusted strings can throw errors or select unexpected elements, but selectors cannot execute code and there is no privileged selector context. Validating selector inputs is the application's responsibility.

#### Attacker code already running in the page

Any scenario that begins with hostile JavaScript executing in the same document (modifying jQuery internals, monkey-patching prototypes, clobbering globals used by application code) assumes a compromise of the page itself. There is no jQuery boundary left to protect at that point.

#### Vulnerability chaining and gadgets

A jQuery vulnerability must be exploitable through jQuery alone. If the impact depends on a separate defect in another library or in application code (for example a prototype pollution gadget that only matters because another component executes polluted values), the root cause lies in that component.

#### Malicious third-party plugins ([CWE-1357](https://cwe.mitre.org/data/definitions/1357.html))

jQuery plugins run as trusted page code. A malicious or vulnerable plugin is not a jQuery vulnerability.

#### Supply chain compromise

Tampering with packages on npm or a CDN, man-in-the-middle delivery attacks, and malicious clones are not vulnerabilities in jQuery itself. This is distinct from jQuery's own release and build infrastructure: a credible issue in how official jQuery artifacts are built, signed, or published (for example a released file that does not match its source tag) should still be reported privately and will be handled as an infrastructure issue rather than a library vulnerability.

#### Environmental and policy configuration ([CWE-15](https://cwe.mitre.org/data/definitions/15.html))

Missing or weak Content Security Policy, loading jQuery over insecure transport, or running it in unsupported environments are deployment choices of the embedding site.

### Summary

jQuery operates entirely within the trust boundary of the page that loads it. Vulnerabilities in scope are limited to cases where jQuery fails to uphold its documented behavior in the presence of untrusted data: script execution through inputs not documented to be HTML or code, bypass of a documented safety guarantee, prototype pollution, or resource exhaustion on valid inputs. The security of HTML that the application chooses to build from untrusted sources remains the application's responsibility.
