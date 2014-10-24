---
title: First-Party Cookies
abbrev: first-party-cookies
docname: draft-west-first-party-cookies-00
date: 2014
category: std
updates: 6265

ipr: trust200902
area: General
workgroup: HTTPbis
keyword: Internet-Draft

pi: [toc, tocindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
-
    ins: M. West
    name: Mike West
    organization: Google, Inc
    email: mkwst@google.com
    uri: https://mikewest.org/

normative:
  HTML:
    target: https://html.spec.whatwg.org/
    title: HTML Living Standard
    author:
    -
        ins: I. Hickson
        name: Ian Hickson
        organization: Google, Inc.
  RFC2119:
  RFC4790:
  RFC5234:
  RFC6265:
  RFC6454:

informative:
  samedomain-cookies:
    target: http://people.mozilla.org/~mgoodwin/SameDomain/samedomain-latest.txt
    title: SameDomain Cookie Flag
    author:
    -
      ins: M. Goodwin,
      name: Mark Goodwin
    -
      ins: J. Walker
      name: Joe Walker
    date: 2011
  pixel-perfect:
    target: http://www.contextis.com/documents/2/Browser_Timing_Attacks.pdf
    title: Pixel Perfect Timing Attacks with HTML5
    author:
    -
        ins: P. Stone
        name: Paul Stone

--- abstract

This document updates RFC6265, defining the `First-Party` attribute for cookies,
which allows servers to mitigate the risk of cross-site request forgery and
related information leakage attacks by asserting that a particular cookie should
only be sent in a "first-party" context.

--- middle

# Introduction

Section 8.2 of {{RFC6265}} eloquently notes that cookies are a form of ambient
authority, attached by default to requests the user agent sends on a user's
behalf. Even when an attacker doesn't know the contents of a user's cookies,
she can still execute commands on the user's behalf (and with the user's
authority) by asking the user agent to send HTTP requests to unwary servers.

Here, we update {{RFC6265}} with a simple mitigation strategy that allows
servers to declare certain cookies as "First-party cookies" which should be
attached to requests if and only if they occur in a first-party context.

Note that the mechanism outlined here is backwards compatible with the existing
cookie syntax. Servers may serve first-party cookies to all user agents; those
that do not support the `First-Party` attribute will simply store a
non-first-party cookie, just as they do today.

## Examples

First-party cookies are set via the `First-Party` attribute in the `Set-Cookie`
header field. That is, given a server's response to a user agent which contains
the following header field:

    Set-Cookie: SID=31d4d96e407aad42; First-Party

Subsequent requests from that user agent can be expected to contain the
following header field if and only if both the requested resource and the
resource in the top-level browsing context match the cookie.

# Terminology and notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{RFC2119}}.

This specification uses the Augmented Backus-Naur Form (ABNF) notation of
{{RFC5234}}.

Two sequences of octets are said to case-insensitively match each other if and
only if they are equivalent under the `i;ascii-casemap` collation defined in
{{RFC4790}}.

The terms "active document", and "top-level browsing context" are defined in
the HTML Living Standard. {{HTML}}

The term "origin" and the mechanism of deriving an origin from a URI are defined
in {{RFC6454}}.

## First-party and Third-party Requests  {#first-and-third-party}

The URL displayed in a user agent's address bar is the only security context
directly exposed to users, and therefore the only signal users can reasonably
rely upon to determine who they're talking to.

Broadly speaking, then, a "first-party" request is an HTTP request for a
resource whose URL's origin matches the origin of the URL the user sees in the
address bar. A "third-party" request is an HTTP request for a resource at any
other origin.

To be slightly more precise, given an HTTP request `request`:

1. Let `context` be the top-level browsing context in the window responsible
   for `request`.

2. Let `top-origin` be the origin of the location of the active document in
   `context`.

3. If the origin of `request`'s URL is the same as `top-origin`, `request`
   is a **first-party request**. Otherwise, `request` is a **third-party
   request**.

Note that we deal with the document's _location_ in step 2 above, not with the
document's origin. For example, a top-level document from `https://example.com`
which has been sandboxed into a unique origin still creates a non-unique
first-party context for subsequent requests.

This definition has a few implications:

* New windows create new first-party contexts.
* Full-page navigations create new first-party contexts. Notably, this
  includes both HTTP and `<meta>`-driven redirects.
* `<iframe>`s do _not_ create new first-party contexts; their requests MUST
  be considered in the context of the origin of the URL the user actually
  sees in the user agent's address bar.

# Server Requirements

This section describes extensions to {{RFC6265}} necessary to implement the
server-side requirements of the `First-Party` attribute.

## Grammar

Add `First-Party` to the list of accepted attributes in the `Set-Cookie` header
field's value by replacing the `cookie-av` token definition in Section 4.1.1 of
{{RFC6265}} with the following ABNF grammar:

    cookie-av         = expires-av / max-age-av / domain-av /
                        path-av / secure-av / httponly-av /
                        first-party-av / extension-av
    first-party-av    = "First-Party"

## Semantics of the "First-Party" Attribute (Non-Normative)

The "First-Party" attribute limits the scope of the cookie such that it will
only be attached to requests if those requests are "first-party", as described
in {{first-and-third-party}}. For example, requests for
`https://example.com/sekrit-image` will attach first-party cookies if and only
if the top-level browsing context is currently displaying a document from
`https://example.com`.

The changes to the `Cookie` header field suggested in {{cookie-header}} provide
additional detail.

# User Agent Requirements

This section describes extensions to {{RFC6265}} necessary in order to implement
the client-side requirements of the `First-Party` attribute.

## The "First-Party" attribute

The following attribute definition should be considered part of the the
`Set-Cookie` algorithm as described in Section 5.2 of {{RFC6265}}:

If the attribute-name case-insensitively matches the string "First-Party", the
user agent MUST append an attribute to the `cookie-attribute-list` with an
`attribute-name` of "First-Party" and an empty `attribute-value`.

## Monkey-patching the Storage Model

Note: There's got to be a better way to specify this. Until I figure out
what that is, monkey-patching!

Alter Section 5.3 of {{RFC6265}} as follows:

1.  Add `firstparty-flag` to the list of fields stored for each cookie.

2.  Before step 11 of the current algorithm, add the following:

    11.  If the `cookie-attribute-list` contains an attribute with an
         `attribute-name` of "First-Party", set the cookie's `first-party-flag`
         to true. Otherwise, set the cookie's `first-party-flag` to false.

    12.  If the cookie's `first-party-flag` is set to true, and the request
         which generated the cookie is not a first-party request (as defined
         in {{first-and-third-party}}), then abort these steps and ignore the
         newly created cookie entirely.

## Monkey-patching the "Cookie" header {#cookie-header}

Note: There's got to be a better way to specify this. Until I figure out
what that is, monkey-patching!

Alter Section 5.4 of {{RFC6265}} as follows:

1.  Add the following requirement to the list in step 1:

    * If the cookie's `first-party-flag` is true, then exclude the cookie if
      the HTTP request is a third-party request (see
      {{first-and-third-party}}).

Note that the modifications suggested here concern themselves only with the
origin of the top-level browsing context and the origin of the resource being
requested. The cookie's `domain`, `path`, and `secure` attributes do not come
into play for this comparison.

# Authoring Considerations

## Mashups and Widgets

The `First-Party` attribute is inappropriate for some important use-cases. In
particular, note that content which is intended to be embedded in a
third-party context (social networking widgets or commenting services, for
instance) will lose access to first-party cookies, which may prevent it from
acting in the intended way. Concretely, you wouldn't be able to share directly
to a social network from a news article if your session cookie was first-party.

# Privacy Considerations

## User Controls

First-party cookies in and of themselves don't do anything to address the
general privacy concerns outlined in Section 7.1 of {{RFC6265}}. The attribute
is set by the server, and serves to mitigate the risk of certain kinds of
attacks that the server is worried about. The user is not involved in this
decision.

User agents, however, could offer users the ability to toggle a cookie's
`first-party-flag` themselves, perhaps as part of a more general cookie
management interface. This could provide an interesting middle-ground between
the options (e.g. "Block all", "Block third-party", and "Allow all") that many
user agents offer to users.

# Security Considerations

## Limitations

It is possible to bypass the protection that first-party cookies offer against
cross-site request forgery attacks by creating first-party contexts in which to
execute the attack. Consider, for instance, the URL `https://example.com/logout`
which logs the current user out of `example.com`. If the user's session cookie
is first-party cookie, then embedding the logout URL in an `<iframe>` element or
an `<img>` element won't log her out, as the cookie won't be sent. Popping up a
new window, or doing a top-level navigation, on the other hand, will create a
first-party context, attach cookies, and perform the logout.

Note, though, that popping up a window, or doing a top-level navigation are both
significantly more visible to the user than loading a subresource. Users will
at least have the opportunity to notice that something strange is going on,
which hopefully reduces an attacker's ability to perform untargeted attacks.

Further, note that certain kinds of attacks are no longer possible if a
first-party context is required. Information leakage attacks which rely on
visible side-effects of loading a session-protected image, for example, can no
longer access those side-effects if the image is loaded in a new window. Timing
attacks like those Paul Stone outlines in {{pixel-perfect}} are no longer
possible if the session cookie is first-party, as they rely on `<iframes>` to
contain the protected content in a way the attacker can manipulate. 

# Acknowledgements

The first-party cookie concept documented here is similar to (but stricter than)
Mark Goodwin's and Joe Walker's {{samedomain-cookies}}.

--- back
