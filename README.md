# drf-httpsig

| Field | Value |
|---|---|
| **Owner** | INE Engineering |
| **Slack** | [#all-developers](https://slack.com/app_redirect?channel=GS8K9MV0V) · [#dev-team](https://slack.com/app_redirect?channel=C05UPTNP7EV) |
| **Status** | Active |
| **Type** | Python package |
| **Last reviewed** | 2026-06-25 — @jkahgee |

## Overview

`drf-httpsig` adds [HTTP Signature](https://datatracker.ietf.org/doc/draft-cavage-http-signatures/)
authentication to the [Django REST framework](https://www.django-rest-framework.org/).
The HTTP Signature scheme gives an HTTP request origin authentication and message
integrity by signing a chosen set of request headers with a shared secret, in a
manner conceptually similar to AWS request signing.

This package is a thin Django REST framework (DRF) adapter: the cryptographic work
of building and verifying the canonical signing string lives in the upstream
[`httpsig`](https://github.com/ahknight/httpsig) library, and this repository
provides the DRF authentication-class glue around it. The entire public surface is
a single class, `SignatureAuthentication`, in
[`drf_httpsig/authentication.py`](drf_httpsig/authentication.py).

This is an INE fork (hosted in the `Internetworkexpert` GitHub org) of the original
`ahknight/drf-httpsig`. Relative to upstream, this fork adds two behaviours, both
implemented in `authentication.py` and covered by the test suite:

- **Signature expiry.** When the request carries an `(expires)` header, the
  authenticator rejects the request once the current time passes that Unix
  timestamp.
- **On-behalf-of impersonation.** When the request carries an `On-Behalf-Of`
  header, the authenticator resolves the impersonated user via a
  `fetch_on_behalf_of_user(user_id)` hook (which you implement in your subclass)
  and authenticates the request as that user.

### What the authenticator does

`SignatureAuthentication` subclasses DRF's `BaseAuthentication`. On each request its
`authenticate()` method:

1. Reads the `Authorization` header and returns `None` (declines, does not fail) if
   it is absent or is not a `Signature` scheme header — letting other authenticators
   run.
2. Fails (`AuthenticationFailed`) if the `Signature` header is malformed or is
   missing any of the required `keyId`, `algorithm`, or `signature` fields.
3. Calls your `fetch_user_data(keyId, algorithm)` to resolve the `(User, secret)`
   pair for the supplied key ID, and fails if no user/secret is found.
4. Verifies the signature over the configured `required_headers` using
   `httpsig.HeaderVerifier`.
5. Enforces `(expires)` and, if present, `On-Behalf-Of` as described above.
6. On success returns `(user, keyId)` (or `(user, None)` for an on-behalf-of
   request), which DRF assigns to `request.user` / `request.auth`.

Every signature/key-resolution failure raises the same shared
`AuthenticationFailed("Invalid signature.")` instance on purpose, so the API does
not leak whether a given key ID exists. (The one distinct message is on the
on-behalf-of path: if the `On-Behalf-Of` lookup returns no user, the request fails
with `"On behalf of user was not found."`.)

## Getting started

### Prerequisites

- Python 3.6–3.8. The repository pins interpreters for local development in
  [`.python-version`](.python-version) (3.8.0 / 3.7.5 / 3.6.9) and the test matrices
  in [`tox.ini`](tox.ini) and [`.travis.yml`](.travis.yml) target py36–py38.
- A Django REST framework project. Runtime dependencies are declared in
  [`setup.py`](setup.py) and [`requirements.txt`](requirements.txt):
  `django>=1.11`, `djangorestframework>3`, and `httpsig>=1.3.0`.

### Install

This package is published to INE's AWS CodeArtifact (domain `ine`, repository
`python`) rather than to public PyPI — see [Deployment](#deployment). Install it the
same way you install any other INE internal Python package, from that index, e.g.:

```text
pip install drf-httpsig
```

To work against a checkout of this repository, install it in editable mode from the
repository root:

```text
pip install -e .
```

`setup.py` derives the package version from Git tags via `setuptools_scm`
(`use_scm_version=True`), so an installable build needs the repository's tags
present.

### Use it in your project

`SignatureAuthentication` is abstract: you must subclass it and implement
`fetch_user_data()` to map an incoming key ID (and the algorithm the client claims)
to a Django user and that user's shared secret. Implement
`fetch_on_behalf_of_user()` as well only if you use the `On-Behalf-Of` feature.

```python
# my_api/auth.py
from django.contrib.auth.models import User
from drf_httpsig.authentication import SignatureAuthentication


class MyAPISignatureAuthentication(SignatureAuthentication):
    # Optional overrides:
    #   www_authenticate_realm  (default "api")
    #   required_headers        (default ["(request-target)", "date"])

    def fetch_user_data(self, key_id, algorithm="hmac-sha256"):
        # key_id and algorithm come from the client and are UNTRUSTED;
        # validate them here. Return (User, secret) or (None, None).
        try:
            user = User.objects.get(keyId=key_id, algo=algorithm)
            return (user, user.secret)
        except User.DoesNotExist:
            return (None, None)
```

Then register the class with DRF:

```python
# my_project/settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "my_api.auth.MyAPISignatureAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
}
```

With the permission class above, every request must carry a valid signature.

## Usage examples

### With Python `requests`

Use the `httpsig` request-auth helper to sign outgoing requests:

```python
import requests
from httpsig.requests_auth import HTTPSignatureAuth

KEY_ID = "su-key"
SECRET = "my secret string"

signature_headers = ["(request-target)", "accept", "date", "host"]
headers = {
    "Host": "localhost:8000",
    "Accept": "application/json",
    "Date": "Mon, 17 Feb 2014 06:11:05 GMT",
}

auth = HTTPSignatureAuth(
    key_id=KEY_ID,
    secret=SECRET,
    algorithm="hmac-sha256",
    headers=signature_headers,
)
resp = requests.get("http://localhost:8000/resource/", auth=auth, headers=headers)
print(resp.content)
```

### With cURL

The `signature` value must be precomputed (a base64 HMAC of the canonical signing
string built from the listed headers). The signed `headers` list must cover every
header in the server's `required_headers` — by default `(request-target)` and
`date`, so the example below signs both. `(request-target)` is not a header you
send: the verifier synthesizes it from the request method and path, so the only
literal header to transmit here is `Date` (it must be byte-for-byte the value you
signed):

```text
curl -v \
  -H 'Date: Mon, 17 Feb 2014 06:11:05 GMT' \
  -H 'Authorization: Signature keyId="my-key",algorithm="hmac-sha256",headers="(request-target) date",signature="<base64-hmac>"' \
  http://localhost:8000/resource/
```

[`generate_test_data.py`](generate_test_data.py) is a small helper that signs a
fixed request with `httpsig.HeaderSigner` and prints the resulting headers, useful
for producing example/fixture signatures.

## Testing

Tests live in [`drf_httpsig/tests.py`](drf_httpsig/tests.py) and run against an
in-memory SQLite database configured by [`test_settings.py`](test_settings.py). The
suite uses `pytest` with `pytest-django` and `freezegun` (the latter to test
`(expires)` handling at a frozen time); these test dependencies are declared under
`tests_require` in [`setup.py`](setup.py).

Run the tests across the supported interpreters with tox:

```text
tox
```

`tox.ini` sets `DJANGO_SETTINGS_MODULE=test_settings` and runs `./setup.py test`
per environment (py36, py37, py38). [`manage.py`](manage.py) is provided for ad hoc
Django management commands against the same test settings.

## Deployment

Publishing is automated by
[`.github/workflows/publish-package.yml`](.github/workflows/publish-package.yml),
which runs when a GitHub Release is published (and can also be triggered manually via
`workflow_dispatch`, with an optional version override). The workflow:

- assumes AWS credentials from repository secrets and fetches a CodeArtifact
  authorization token, and
- builds and uploads the package to INE's CodeArtifact (domain `ine`, domain owner
  `324321837501`, repository `python`, region `us-east-1`).

> **Note:** the publish workflow drives the build and upload with **Poetry**
> (`poetry version` / `poetry build` / `poetry publish`), but this repository
> currently ships only setuptools packaging — `setup.py` and `setup.cfg`, with no
> `pyproject.toml` or `poetry.lock`. The Poetry build steps therefore expect Poetry
> packaging metadata that is not yet present in the repo. Reconcile the packaging
> (add a `pyproject.toml`, or switch the workflow to the setuptools build) before
> relying on an automated release.

## Continuous integration

This repository has no automated build, test, or deploy-on-merge pipeline under
`.github/workflows`: the only operational workflow is the release publisher
described above, which runs on a published GitHub Release rather than on push or
pull request. A legacy [`.travis.yml`](.travis.yml) (Travis CI, py36–py38, running
`setup.py test`) is still present in the tree but is not wired into GitHub Actions.

This repository also participates in the org-wide README compliance check via
`.github/workflows/readme-compliance.yml`. That check omits an explicit mode and
follows the org-wide `README_GATE_MODE` variable: `report` (non-blocking) or `block`
(enforced).

## Links

- [HTTP Signatures IETF draft](https://datatracker.ietf.org/doc/draft-cavage-http-signatures/)
- [Django REST framework](https://www.django-rest-framework.org/)
- [`httpsig` library (upstream signing/verification)](https://github.com/ahknight/httpsig)
- [`ahknight/drf-httpsig` (upstream project)](https://github.com/ahknight/drf-httpsig)
- [`CHANGELOG.rst`](CHANGELOG.rst)
- [`LICENSE.txt`](LICENSE.txt) (MIT)
