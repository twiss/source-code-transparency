# Source Code Transparency

Source Code Transparency is a proposed mechanism to publish (the hash
of) the source code of web apps to a transparency log, allowing auditors
to check that no malicious code was deployed for a given web app.

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

To signal to the browser that a given web app is using Source Code
Transparency, we could introduce a X.509 certificate extension, which
would automatically be included in the Certificate Transparency logs,
allowing security researchers to check that all certificates for the
domain of the web app include this extension. Browser would refuse to
load web apps without Source Code Transparency from a domain using a
certificate with this extension.

[1]: https://certificate.transparency.dev/
[2]: https://wpack-wg.github.io/bundled-responses/draft-ietf-wpack-bundled-responses.html
