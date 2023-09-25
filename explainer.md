# Source Code Transparency

## Problem Statement

Today, whenever you open a web app, the browser fetches and runs its
source code from the server. This enables the ease of deployment and
iteration that the web is known for, but can also pose security risks.
In particular, for web apps that don't want to trust the server, such
as those using client-side encryption to protect the user's data before
sending it to the server, or those processing the user's data entirely
client-side without sending any sensitive data to the server, the
current security model of the web is insufficient.

After all, if a web app claims not to send sensitive data to the server,
this is very difficult for users to check, as they would need to read
(and understand) the source every time it's loaded from the server.
Even for security researchers, such a claim is impossible to verify, as
the server could simply serve a different version of the web app to a
user it wants to target than to the security researcher.

Therefore, we would like to propose a mechanism to enable security
researchers to audit the source code of especially-sensitive web apps
that is (or was) sent to any user, not just to themselves.

```
# jka commentary
Rephrasing this: we'd like a way to publish the source for content distributed
on the web, and for recipients to confirm that the content they received from
the server matches what was expected.
```

Our goal is only to allow detection rather than prevention of malicious
code, possibly after the fact, as we believe this will still deter
servers from serving malicious code. Additionally, preventing malicious
code from being deployed would require mandating security audits prior
to deploys, which would slow down and fundamentally change the
deployment model of the web.

## High-level Proposal

To accomplish the above, the source code of (a given version of) a web
app should be published to a transparency log, similar to [Certificate
Transparency][1]. Then, when opening a web app, the browser should
check that the fetched source code was received by (and is or will be
included in) the transparency log, and only run it if so. Then,
security researchers can check the transparency log for all published
versions of a given web app, and check that none of them contain any
malicious code.

```
# jka commentary
A few observations here:

  * A second channel to obtain the 'expected' content -- or a
    cryptographically strong assertion that we can rely on to confirm whether
    we've received matching/mismatching content is required with any approach.

  * That second channel becomes an element in the chain-of-trust; we want it
    to be as difficult as possible for potential attackers to disrupt, and as
    easy as possible for our colleages / peers / security researchers to
    verify independently.

  * To some extent I'd argue that source control repositories (particularly
    `git`, `fossil` and `hg`) provide some of the transparency and
    cryptographic properties desired here.  However, there is no single
    location where repositories are aggregated - and that may be a benefit
    under some threat models.
```

To aid the auditing process, web apps may want to employ additional
existing and proposed security mechanisms, such as CSP, SRI, SBOMs,
reproducible builds, etc. These mechanisms, which currently only allow
the developer of web apps to check the security of the web app, would
additionally allow external security researchers to audit a web app, if
it used a mechanism such as Source Code Transparency.

```
# jka commentary
My - possibly incorrect - understanding/explanation of some of the acronyms:

  * CSP: Content security policy: the server can provide instructions to a
    client browser about the acceptable places that it should expect to
    retrieve additional content from; images, scripts, and so on.  This allows
    server administrators to limit the ability for attackers on their website
    to instruct client browsers to retrieve unwanted content.  However: if the
    attacker is able to modify the initial content download -- the attack that
    we're intending to defend against -- then the policy in the download may
    itself not be trustworthy.

  * SRI: SubResource Integrity: within the initial content download that the
    client browser receives, there may be hyperlinks, hyperreferences and
    relationships to other linked content.  SRI is a way for the server to
    provide the client with a hash of each item of linked content, so that the
    client can deteremine whether it has received the correct content
    if-and-when it retrieves content from those references.

    Currently, no such SRI hash is available for the initial content download
    (often an `index.html` page, or web app `webmanifest` file) - so the
    initial download could be considered a weak point.

    Additionally, browsers -- following the specifications -- do not support
    SRI for all types of content that can be referenced from a webpage, and
    some content types are susceptible to vulnerabilities (image formats, for
    example, seem to be particularly problematic).

  * SBOM: Software Bill of Materials: analogous to a list of items required to
    build a product, this is a structured listing of the software dependencies
    for a software component.  The idea is to provide assistance assessing the
    supply chain involved in a component, and to some extent to help trace the
    impact of vulnerabilities.  These are machine-readable, but for various
    reasons it is generally impossible to completely rely on computerised
    analysis to trace the effect of vulnerabilities.  Integration of software
    components is context-sensitive and complicated.  If a furniture item's
    bill of materials mentions a metallic plate, and it is later discovered
    that the plate can buckle under load, it may or may not be problem for
    the furniture item, depending on where and how the plate is used in the
    furnishing.

  * Reproducible Builds: many software components are not distributed in the
    form of source code: instead, they are 'compiled' into a binary format that
    is generally more performant and suitable for machines.  That's useful, but
    it typically increases the amount of time and expertise required to
    understand what a binary does by an order of magnitude -- and that in turn
    means that it is extremely difficult to determine whether a binary was
    built from a specific source code listing.

    Things become even more complicated when we consider that a binary may be
    compiled by composing multiple disparate source code listings (libraries)
    into a single result, and that the compiler has a lot of latitude to
    rearrange the contents of the binary without affecting the apparent
    behaviour of the program when it runs on a computer.

    Reproducible builds aims to solve this problem by placing tighter
    constraints on how programs are built, and reducing the number of elements
    of the binary that can change if the same program is built on two different
    computers.

    The result should be that if a thousand people in different locations all
    around the planet build a binary from the same reproducible build source
    code, then they should receive the exact same bit-for-bit binary, something
    that they should be able to confer about and confirm using multiple
    different communication and verification methods.

Note: some of these technologies are mentioned side-by-side with Open Source
software, however these techniques are equally applicable and no more or less
technically difficult to achieve with proprietary (closed source) software.
```

## (More) Detailed Proposal

To simplify the distribution and integrity verification of web apps'
source code, we could make use of [Web Bundles][2], and publish a hash
of the bundle to a transparency log.

The transparency log could be structured as an append-only chain of
Merkle Trees, each of which serves as a map of `(domain, version)` keys
to `hash(bundle)` values (where the key could be hashed and interpreted
as a binary path in the binary tree, with the value of the final leaf
being the hash of the web bundle). Web apps could submit the hash of
the bundle to the transparency log, which would periodically include
all (existing and new versions of each) web app in the next Merkle Tree.
Auditors could check that the transparency log is behaving consistently
and not changing the hash of old versions of web apps, for example.

Then, when loading a participating web app, the browser would check the
latest Merkle Tree and verify that the hash of the bundle received is
included in it, and warn the user if not. To still allow instantaneous
deploys, we could allow this check to be done some time after loading
the web app, at the cost of delayed warnings in case of issues.

```
# jka commentary
I think this is probably implicit/agreeing with the above, but to make sure:
in the case of web applications, I think that the client should wait until
a verification hash is available for an updated version of an application
before the corresponding content is run or loaded.
```

To signal to the browser that a given web app is using Source Code
Transparency, we could introduce a X.509 certificate extension, which
would automatically be included in the Certificate Transparency logs,
allowing security researchers to check that all certificates for the
domain of the web app include this extension. Browser would refuse to
load web apps without Source Code Transparency from a domain using a
certificate with this extension.

```
# jka commentary
This seems OK, although I'd note that many web clients don't use Transport
Layer Security (TLS - also known as HTTPS), required for X.509 certificates.

Certainly the most popular websites do use TLS -- I would argue that they have
an incentive not to allow any of their competitors or new market entrants to
inspect or learn from much, if any, of their web traffic.  Those clients and
applications could be excluded from integrity guarantees as a result.

Draft specifications to provide web request (HTTP) integrity without a
requirement for the use of TLS do exist; notably HTTP Message Signatures:
https://www.ietf.org/archive/id/draft-ietf-httpbis-message-signatures-06.html

In contrast I'd mention that all browsers already support DNS (Domain Name
System) queries, because resolving website URLs, except in rare or unusual
cases, requires DNS lookups.  A suitable DNS record could be offered to plug
the SRI (SubResource Integrity) gap that exists on initial-content download.

However: DNS also has its own limitations, and I'm not an expert with it.  I
suggest it mainly because it seems far enough outside the scope of the
webserver and source code to be difficult for an attacker to tamper with, and
replicated and inspectable enough for independent verification to be possible.
There does remain the possibility for individuals to be targeted with network-
local DNS poisioning that could be impossible for them to detect on their own.
```

[1]: https://certificate.transparency.dev/
[2]: https://wpack-wg.github.io/bundled-responses/draft-ietf-wpack-bundled-responses.html
