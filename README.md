email-validator: Validate Email Addresses
=========================================

A robust email address syntax and deliverability validation library for
Python 3.8+ by [Joshua Tauberer](https://joshdata.me).

This library validates that a string is of the form `name@example.com`
and optionally checks that the domain name is set up to receive email.
This is the sort of validation you would want when you are identifying
users by their email address like on a registration/login form (but not
necessarily for composing an email message, see below).

Key features:

* Checks that an email address has the correct syntax --- great for
  email-based registration/login forms or validing data.
* Gives friendly English error messages when validation fails that you
  can display to end-users.
* Checks deliverability (optional): Does the domain name resolve?
  (You can override the default DNS resolver to add query caching.)
* Can be called asynchronously with `await`.
* Supports internationalized domain names and internationalized local parts.
* Rejects addresses with unsafe Unicode characters, obsolete email address
  syntax that you'd find unexpected, special use domain names like
  `@localhost`, and domains without a dot by default. This is an
  opinionated library!
* Normalizes email addresses (important for internationalized
  and quoted-string addresses! see below).
* Python type annotations are used.

This is an opinionated library. You should definitely also consider using
the less-opinionated [pyIsEmail](https://github.com/michaelherold/pyIsEmail) and
[flanker](https://github.com/mailgun/flanker) if they are better for your
use case.

[![Build Status](https://github.com/JoshData/python-email-validator/actions/workflows/test_and_build.yaml/badge.svg)](https://github.com/JoshData/python-email-validator/actions/workflows/test_and_build.yaml)

View the [CHANGELOG / Release Notes](CHANGELOG.md) for the version history of changes in the library. Occasionally this README is ahead of the latest published package --- see the CHANGELOG for details.

---

Installation
------------

This package [is on PyPI](https://pypi.org/project/email-validator/), so:

```sh
pip install email-validator
```

(You might need to use `pip3` depending on your local environment.)

Quick Start
-----------

If you're validating a user's email address before creating a user
account in your application, you might do this:

```python
from email_validator import validate_email, EmailNotValidError

email = "my+address@example.org"

try:

  # Check that the email address is valid. Turn on check_deliverability
  # for first-time validations like on account creation pages (but not
  # login pages).
  emailinfo = validate_email(email, check_deliverability=False)

  # After this point, use only the normalized form of the email address,
  # especially before going to a database query.
  email = emailinfo.normalized

except EmailNotValidError as e:

  # The exception message is human-readable explanation of why it's
  # not a valid (or deliverable) email address.
  print(str(e))
```

This validates the address and gives you its normalized form. You should
**put the normalized form in your database** and always normalize before
checking if an address is in your database. When using this in a login form,
set `check_deliverability` to `False` to avoid unnecessary DNS queries.

See below for examples for caching DNS queries and calling the library
asynchronously with `await`.

Usage
-----

### Overview

The module provides a function `validate_email(email_address)` which
takes an email address and:

- Raises a `EmailNotValidError` with a helpful, human-readable error
  message explaining why the email address is not valid, or
- Returns an object with a normalized form of the email address (which
  you should use!) and other information about it.

When an email address is not valid, `validate_email` raises either an
`EmailSyntaxError` if the form of the address is invalid or an
`EmailUndeliverableError` if the domain name fails DNS checks. Both
exception classes are subclasses of `EmailNotValidError`, which in turn
is a subclass of `ValueError`.

But when an email address is valid, an object is returned containing
a normalized form of the email address (which you should use!) and
other information.

The validator doesn't, by default, permit obsoleted forms of email addresses
that no one uses anymore even though they are still valid and deliverable, since
they will probably give you grief if you're using email for login. (See
later in the document about how to allow some obsolete forms.)

The validator optionally checks that the domain name in the email address has
a DNS MX record indicating that it can receive email. (Except a Null MX record.
If there is no MX record, a fallback A/AAAA-record is permitted, unless
a reject-all SPF record is present.) DNS is slow and sometimes unavailable or
unreliable, so consider whether these checks are useful for your use case and
turn them off if they aren't.
There is nothing to be gained by trying to actually contact an SMTP server, so
that's not done here. For privacy, security, and practicality reasons, servers
are good at not giving away whether an address is
deliverable or not: email addresses that appear to accept mail at first
can bounce mail after a delay, and bounced mail may indicate a temporary
failure of a good email address (sometimes an intentional failure, like
greylisting).

### Options

The `validate_email` function also accepts the following keyword arguments
(defaults are as shown below):

`check_deliverability=True`: If true, DNS queries are made to check that the domain name in the email address (the part after the @-sign) can receive mail, as described above. Set to `False` to skip this DNS-based check. It is recommended to pass `False` when performing validation for login pages (but not account creation pages) since re-validation of a previously validated domain in your database by querying DNS at every login is probably undesirable. You can also set `email_validator.CHECK_DELIVERABILITY` to `False` to turn this off for all calls by default.

`dns_resolver=None`: Pass an instance of [dns.resolver.Resolver](https://dnspython.readthedocs.io/en/latest/resolver-class.html) to control the DNS resolver including setting a timeout and [a cache](https://dnspython.readthedocs.io/en/latest/resolver-caching.html). The `caching_resolver` function shown below is a helper function to construct a dns.resolver.Resolver with a [LRUCache](https://dnspython.readthedocs.io/en/latest/resolver-caching.html#dns.resolver.LRUCache). Reuse the same resolver instance across calls to `validate_email` to make use of the cache.

`test_environment=False`: If `True`, DNS-based deliverability checks are disabled and  `test` and `**.test` domain names are permitted (see below). You can also set `email_validator.TEST_ENVIRONMENT` to `True` to turn it on for all calls by default.

`allow_smtputf8=True`: Set to `False` to prohibit internationalized addresses that would
    require the
    [SMTPUTF8](https://tools.ietf.org/html/rfc6531) extension. You can also set `email_validator.ALLOW_SMTPUTF8` to `False` to turn it off for all calls by default.

`allow_quoted_local=False`: Set to `True` to allow obscure and potentially problematic email addresses in which the part of the address before the @-sign contains spaces, @-signs, or other surprising characters when the local part is surrounded in quotes (so-called quoted-string local parts). In the object returned by `validate_email`, the normalized local part removes any unnecessary backslash-escaping and even removes the surrounding quotes if the address would be valid without them. You can also set `email_validator.ALLOW_QUOTED_LOCAL` to `True` to turn this on for all calls by default.

`allow_domain_literal=False`: Set to `True` to allow bracketed IPv4 and "IPv6:"-prefixd IPv6 addresses in the domain part of the email address. No deliverability checks are performed for these addresses. In the object returned by `validate_email`, the normalized domain will use the condensed IPv6 format, if applicable. The object's `domain_address` attribute will hold the parsed `ipaddress.IPv4Address` or `ipaddress.IPv6Address` object if applicable. You can also set `email_validator.ALLOW_DOMAIN_LITERAL` to `True` to turn this on for all calls by default.

`allow_empty_local=False`: Set to `True` to allow an empty local part (i.e.
    `@example.com`), e.g. for validating Postfix aliases.
    

### DNS timeout and cache

When validating many email addresses or to control the timeout (the default is 15 seconds), create a caching [dns.resolver.Resolver](https://dnspython.readthedocs.io/en/latest/resolver-class.html) to reuse in each call. The `caching_resolver` function returns one easily for you:

```python
from email_validator import validate_email, caching_resolver

resolver = caching_resolver(timeout=10)

while True:
  validate_email(email, dns_resolver=resolver)
```

### Asynchronous call

The library has an alternative, asynchronous method named `validate_email_async` which must be called with `await`. This method uses an [asynchronous DNS resolver](https://dnspython.readthedocs.io/en/latest/async.html) so that multiple DNS-based deliverability checks can be performed in parallel.

Here how to use it. In this example, `import ... as` is used to alias the async method to the usual method name `validate_email`.

```python
from email_validator import validate_email_async as validate_email, \
  EmailNotValidError, caching_async_resolver

resolver = caching_async_resolver(timeout=10)

email = "my+address@example.org"
try:
  emailinfo = await validate_email(email)
  email = emailinfo.normalized
except EmailNotValidError as e:
  print(str(e))
```

Note that to create a caching asynchronous resolver, use `caching_async_resolver`. As with the synchronous version, creating a resolver is optional.

When processing batches of email addresses, I found that chunking around 25 email addresses at a time (using e.g. `asyncio.gather()`) resulted in the highest performance. I tested on a residential Internet connection with valid addresses.

### Test addresses

This library rejects email addresses that use the [Special Use Domain Names](https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml) `invalid`, `localhost`, `test`, and some others by raising `EmailSyntaxError`. This is to protect your system from abuse: You probably don't want a user to be able to cause an email to be sent to `localhost` (although they might be able to still do so via a malicious MX record). However, in your non-production test environments you may want to use `@test` or `@myname.test` email addresses. There are three ways you can allow this:

1. Add `test_environment=True` to the call to `validate_email` (see above).
2. Set `email_validator.TEST_ENVIRONMENT` to `True` globally.
3. Remove the special-use domain name that you want to use from `email_validator.SPECIAL_USE_DOMAIN_NAMES`, e.g.:

```python
import email_validator
email_validator.SPECIAL_USE_DOMAIN_NAMES.remove("test")
```

It is tempting to use `@example.com/net/org` in tests. They are *not* in this library's `SPECIAL_USE_DOMAIN_NAMES` list so you can, but shouldn't, use them. These domains are reserved to IANA for use in documentation so there is no risk of accidentally emailing someone at those domains. But beware that this library will nevertheless reject these domain names if DNS-based deliverability checks are not disabled because these domains do not resolve to domains that accept email. In tests, consider using your own domain name or `@test` or `@myname.test` instead.

Internationalized email addresses
---------------------------------

The email protocol SMTP and the domain name system DNS have historically
only allowed English (ASCII) characters in email addresses and domain names,
respectively. Each has adapted to internationalization in a separate
way, creating two separate aspects to email address
internationalization.

### Internationalized domain names (IDN)

The first is [internationalized domain names (RFC
5891)](https://tools.ietf.org/html/rfc5891), a.k.a IDNA 2008. The DNS
system has not been updated with Unicode support. Instead, internationalized
domain names are converted into a special IDNA ASCII "[Punycode](https://www.rfc-editor.org/rfc/rfc3492.txt)"
form starting with `xn--`. When an email address has non-ASCII
characters in its domain part, the domain part is replaced with its IDNA
ASCII equivalent form in the process of mail transmission. Your mail
submission library probably does this for you transparently. ([Compliance
around the web is not very good though](http://archives.miloush.net/michkap/archive/2012/02/27/10273315.html).) This library conforms to IDNA 2008
using the [idna](https://github.com/kjd/idna) module by Kim Davies.

### Internationalized local parts

The second sort of internationalization is internationalization in the
*local* part of the address (before the @-sign). In non-internationalized
email addresses, only English letters, numbers, and some punctuation
(`._!#$%&'^``*+-=~/?{|}`) are allowed. In internationalized email address
local parts, a wider range of Unicode characters are allowed.

A surprisingly large number of Unicode characters are not safe to display,
especially when the email address is concatenated with other text, so this
library tries to protect you by not permitting reserved, non-, private use,
formatting (which can be used to alter the display order of characters),
whitespace, and control characters, and combining characters
as the first character of the local part and the domain name (so that they
cannot combine with something outside of the email address string or with
the @-sign). See https://qntm.org/safe and https://trojansource.codes/
for relevant prior work. (Other than whitespace, these are checks that
you should be applying to nearly all user inputs in a security-sensitive
context.)

These character checks are performed after Unicode normalization (see below),
so you are only fully protected if you replace all user-provided email addresses
with the normalized email address string returned by this library. This does not
guard against the well known problem that many Unicode characters look alike
(or are identical), which can be used to fool humans reading displayed text.

Email addresses with these non-ASCII characters require that your mail
submission library and the mail servers along the route to the destination,
including your own outbound mail server, all support the
[SMTPUTF8 (RFC 6531)](https://tools.ietf.org/html/rfc6531) extension.
Support for SMTPUTF8 varies. See the `allow_smtputf8` parameter.

### If you know ahead of time that SMTPUTF8 is not supported by your mail submission stack

By default all internationalized forms are accepted by the validator.
But if you know ahead of time that SMTPUTF8 is not supported by your
mail submission stack, then you must filter out addresses that require
SMTPUTF8 using the `allow_smtputf8=False` keyword argument (see above).
This will cause the validation function to raise a `EmailSyntaxError` if
delivery would require SMTPUTF8. That's just in those cases where
non-ASCII characters appear before the @-sign. If you do not set
`allow_smtputf8=False`, you can also check the value of the `smtputf8`
field in the returned object.

If your mail submission library doesn't support Unicode at all --- even
in the domain part of the address --- then immediately prior to mail
submission you must replace the email address with its ASCII-ized form.
This library gives you back the ASCII-ized form in the `ascii_email`
field in the returned object, which you can get like this:

```python
emailinfo = validate_email(email, allow_smtputf8=False)
email = emailinfo.ascii_email
```

The local part is left alone (if it has internationalized characters
`allow_smtputf8=False` will force validation to fail) and the domain
part is converted to [IDNA ASCII](https://tools.ietf.org/html/rfc5891).
(You probably should not do this at account creation time so you don't
change the user's login information without telling them.)

Normalization
-------------

### Unicode Normalization

The use of Unicode in email addresses introduced a normalization
problem. Different Unicode strings can look identical and have the same
semantic meaning to the user. The `normalized` field returned on successful
validation provides the correctly normalized form of the given email
address.

For example, the CJK fullwidth Latin letters are considered semantically
equivalent in domain names to their ASCII counterparts. This library
normalizes them to their ASCII counterparts:

```python
emailinfo = validate_email("me@Ｄｏｍａｉｎ.com")
print(emailinfo.normalized)
print(emailinfo.ascii_email)
# prints "me@domain.com" twice
```

Because an end-user might type their email address in different (but
equivalent) un-normalized forms at different times, you ought to
replace what they enter with the normalized form immediately prior to
going into your database (during account creation), querying your database
(during login), or sending outbound mail. Normalization may also change
the length of an email address, and this may affect whether it is valid
and acceptable by your SMTP provider.

The normalizations include lowercasing the domain part of the email
address (domain names are case-insensitive), [Unicode "NFC"
normalization](https://en.wikipedia.org/wiki/Unicode_equivalence) of the
whole address (which turns characters plus [combining
characters](https://en.wikipedia.org/wiki/Combining_character) into
precomposed characters where possible, replacement of [fullwidth and
halfwidth
characters](https://en.wikipedia.org/wiki/Halfwidth_and_fullwidth_forms)
in the domain part, possibly other
[UTS46](http://unicode.org/reports/tr46) mappings on the domain part,
and conversion from Punycode to Unicode characters.

(See [RFC 6532 (internationalized email) section
3.1](https://tools.ietf.org/html/rfc6532#section-3.1) and [RFC 5895
(IDNA 2008) section 2](http://www.ietf.org/rfc/rfc5895.txt).)

### Other Normalization

Normalization is also applied to quoted-string local parts and domain
literal IPv6 addresses if you have allowed them by the `allow_quoted_local`
and `allow_domain_literal` options. In quoted-string local parts, unnecessary
backslash escaping is removed and even the surrounding quotes are removed if
they are unnecessary. For IPv6 domain literals, the IPv6 address is
normalized to condensed form. [RFC 2142](https://datatracker.ietf.org/doc/html/rfc2142)
also requires lowercase normalization for some specific mailbox names like `postmaster@`.

### Length checks

This library checks that the length of the email address is not longer than
the maximum length. The check is performed on the normalized form of the
address, which might be different from a string provided by a user. If you
send email to the original string and not the normalized address, the email
might be rejected because the original address could be too long.

Examples
--------

For the email address `test@joshdata.me`, the returned object is:

```python
ValidatedEmail(
  normalized='test@joshdata.me',
  local_part='test',
  domain='joshdata.me',
  ascii_email='test@joshdata.me',
  ascii_local_part='test',
  ascii_domain='joshdata.me',
  smtputf8=False)
```

For the fictitious but valid address `example@ツ.ⓁⒾⒻⒺ`, which has an
internationalized domain but ASCII local part, the returned object is:

```python
ValidatedEmail(
  normalized='example@ツ.life',
  local_part='example',
  domain='ツ.life',
  ascii_email='example@xn--bdk.life',
  ascii_local_part='example',
  ascii_domain='xn--bdk.life',
  smtputf8=False)

```

Note that `normalized` and other fields provide a normalized form of the
email address, domain name, and (in other cases) local part (see earlier
discussion of normalization), which you should use in your database.

Calling `validate_email` with the ASCII form of the above email address,
`example@xn--bdk.life`, returns the exact same information (i.e., the
`normalized` field always will contain Unicode characters, not Punycode).

For the fictitious address `ツ-test@joshdata.me`, which has an
internationalized local part, the returned object is:

```python
ValidatedEmail(
  normalized='ツ-test@joshdata.me',
  local_part='ツ-test',
  domain='joshdata.me',
  ascii_email=None,
  ascii_local_part=None,
  ascii_domain='joshdata.me',
  smtputf8=True)
```

Now `smtputf8` is `True` and `ascii_email` is `None` because the local
part of the address is internationalized. The `local_part` and `normalized` fields
return the normalized form of the address.

Return value
------------

When an email address passes validation, the fields in the returned object
are:

| Field | Value |
| -----:|-------|
| `normalized` | The normalized form of the email address that you should put in your database. This combines the `local_part` and `domain` fields (see below). |
| `ascii_email` | If set, an ASCII-only form of the normalized email address by replacing the domain part with [IDNA](https://tools.ietf.org/html/rfc5891) [Punycode](https://www.rfc-editor.org/rfc/rfc3492.txt). This field will be present when an ASCII-only form of the email address exists (including if the email address is already ASCII). If the local part of the email address contains internationalized characters, `ascii_email` will be `None`. If set, it merely combines `ascii_local_part` and `ascii_domain`. |
| `local_part` | The normalized local part of the given email address (before the @-sign). Normalization includes Unicode NFC normalization and removing unnecessary quoted-string quotes and backslashes. If `allow_quoted_local` is True and the surrounding quotes are necessary, the quotes _will_ be present in this field. |
| `ascii_local_part` | If set, the local part, which is composed of ASCII characters only. |
| `domain` | The canonical internationalized Unicode form of the domain part of the email address. If the returned string contains non-ASCII characters, either the [SMTPUTF8](https://tools.ietf.org/html/rfc6531) feature of your mail relay will be required to transmit the message or else the email address's domain part must be converted to IDNA ASCII first: Use `ascii_domain` field instead. |
| `ascii_domain` | The [IDNA](https://tools.ietf.org/html/rfc5891) [Punycode](https://www.rfc-editor.org/rfc/rfc3492.txt)-encoded form of the domain part of the given email address, as it would be transmitted on the wire. |
| `domain_address` | If domain literals are allowed and if the email address contains one, an `ipaddress.IPv4Address` or `ipaddress.IPv6Address` object. |
| `smtputf8` | A boolean indicating that the [SMTPUTF8](https://tools.ietf.org/html/rfc6531) feature of your mail relay will be required to transmit messages to this address because the local part of the address has non-ASCII characters (the local part cannot be IDNA-encoded). If `allow_smtputf8=False` is passed as an argument, this flag will always be false because an exception is raised if it would have been true. |
| `mx` | A list of (priority, domain) tuples of MX records specified in the DNS for the domain (see [RFC 5321 section 5](https://tools.ietf.org/html/rfc5321#section-5)). May be `None` if the deliverability check could not be completed because of a temporary issue like a timeout. |
| `mx_fallback_type` | `None` if an `MX` record is found. If no MX records are actually specified in DNS and instead are inferred, through an obsolete mechanism, from A or AAAA records, the value is the type of DNS record used instead (`A` or `AAAA`). May be `None` if the deliverability check could not be completed because of a temporary issue like a timeout. |
| `spf` | Any SPF record found while checking deliverability. Only set if the SPF record is queried. |

Assumptions
-----------

By design, this validator does not pass all email addresses that
strictly conform to the standards. Many email address forms are obsolete
or likely to cause trouble:

* The validator assumes the email address is intended to be
  usable on the public Internet. The domain part
  of the email address must be a resolvable domain name
  (see the deliverability checks described above).
  Most [Special Use Domain Names](https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml)
  and their subdomains, as well as
  domain names without a `.`, are rejected as a syntax error
  (except see the `test_environment` parameter above).
* Obsolete email syntaxes are rejected:
  The unusual ["(comment)" syntax](https://github.com/JoshData/python-email-validator/issues/77)
  is rejected. Extremely old obsolete syntaxes are
  rejected. Quoted-string local parts and domain-literal addresses
  are rejected by default, but there are options to allow them (see above).
  No one uses these forms anymore, and I can't think of any reason why anyone
  using this library would need to accept them.

Testing
-------

Tests can be run using

```sh
pip install -r test_requirements.txt 
make test
```

Tests run with mocked DNS responses. When adding or changing tests, temporarily turn on the `BUILD_MOCKED_DNS_RESPONSE_DATA` flag in `tests/mocked_dns_responses.py` to re-build the database of mocked responses from live queries.

For Project Maintainers
-----------------------

The package is distributed as a universal wheel and as a source package.

To release:

* Update CHANGELOG.md.
* Update the version number in `email_validator/version.py`.
* Make & push a commit with the new version number and make sure tests pass.
* Make & push a tag (see command below).
* Make a release at https://github.com/JoshData/python-email-validator/releases/new.
* Publish a source and wheel distribution to pypi (see command below).

```sh
git tag v$(cat email_validator/version.py  | sed "s/.* = //" | sed 's/"//g')
git push --tags
./release_to_pypi.sh
```

License
-------

This project is free of any copyright restrictions per the [Unlicense](https://unlicense.org/). (Prior to Feb. 4, 2024, the project was made available under the terms of the [CC0 1.0 Universal public domain dedication](http://creativecommons.org/publicdomain/zero/1.0/).) See [LICENSE](LICENSE) and [CONTRIBUTING.md](CONTRIBUTING.md).