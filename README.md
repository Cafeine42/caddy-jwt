# caddy-jwt

![Go Workflow](https://github.com/ggicci/caddy-jwt/actions/workflows/go.yml/badge.svg) [![codecov](https://codecov.io/gh/ggicci/caddy-jwt/branch/main/graph/badge.svg?token=4V9OX8WFAW)](https://codecov.io/gh/ggicci/caddy-jwt) [![Go Report Card](https://goreportcard.com/badge/github.com/ggicci/caddy-jwt)](https://goreportcard.com/report/github.com/ggicci/caddy-jwt) [![Go Reference](https://pkg.go.dev/badge/github.com/ggicci/caddy-jwt.svg)](https://pkg.go.dev/github.com/ggicci/caddy-jwt)

A Caddy HTTP Module - who Facilitates **JWT Authentication**

This module fulfilled [`http.handlers.authentication`](https://caddyserver.com/docs/modules/http.handlers.authentication) middleware as a provider named `jwt`.

[Documentation](https://caddyserver.com/docs/modules/http.authentication.providers.jwt)

## Install

Build this module with `caddy` at Caddy's official [download](https://caddyserver.com/download) site. Or build it with [xcaddy](https://github.com/caddyserver/xcaddy) locally by yourself:

```bash
# A caddy binary will be produced in your current directory.
xcaddy build --with github.com/ggicci/caddy-jwt
```

## Sample Caddyfile

```Caddyfile
{
	order jwtauth before basicauth
}

api.example.com {
	jwtauth {
		sign_key TkZMNSowQmMjOVU2RUB0bm1DJkU3U1VONkd3SGZMbVk=
		sign_alg HS256
		jwk_url https://api.example.com/jwk/keys
		from_query access_token token
		from_header X-Api-Token
		from_cookies user_session
		issuer_whitelist https://api.example.com
		audience_whitelist https://api.example.io https://learn.example.com
		user_claims aud uid user_id username login
		meta_claims "IsAdmin->is_admin" "settings.payout.paypal.enabled->is_paypal_enabled"
	}
	reverse_proxy http://172.16.0.14:8080
}
```

**NOTE**:

1. If you were using **symmetric** signing algorithms, e.g. `HS256`, encode your key bytes in `base64` format as `sign_key`'s value.

```text
TkZMNSowQmMjOVU2RUB0bm1DJkU3U1VONkd3SGZMbVk=
```

2. If you were using **asymmetric** signing algorithms, e.g. `RS256`, encode your public key in x.509 PEM format as `sign_key`'s value.

```text
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArzekF0pqttKNJMOiZeyt
RdYiabdyy/sdGQYWYJPGD2Q+QDU9ZqprDmKgFOTxUy/VUBnaYr7hOEMBe7I6dyaS
5G0EGr8UXAwgD5Uvhmz6gqvKTV+FyQfw0bupbcM4CdMD7wQ9uOxDdMYm7g7gdGd6
SSIVvmsGDibBI9S7nKlbcbmciCmxbAlwegTYSHHLjwWvDs2aAF8fxeRfphwQZKkd
HekSZ090/c2V4i0ju2M814QyGERMoq+cSlmikCgRWoSZeWOSTj+rAZJyEAzlVL4z
8ojzOpjmxw6pRYsS0vYIGEDuyiptf+ODC8smTbma/p3Vz+vzyLWPfReQY2RHtpUe
hwIDAQAB
-----END PUBLIC KEY-----
```

3. If you were using **JWK**, configure `jwk_url` and leave `sign_key` unset.

4. `caddy-jwt` will determine the signing algorithm by looking into the following values:

   1. `alg` value in the JWT header;
   2. `alg` value of the matched JWK if using JWK;
   3. value of the `sign_alg` config.

5. The priority of `from_xxx` is `from_query > from_header > from_cookies`.

6. Bypass the verification by turning on `skip_verification` option, [#85](/../../issues/85).

7. Instead of specifying the `sign_key` directly as a value, you can use a feature introduced in Caddy v2.8.0 to load it from a file using `sign_key {file./path/to/sign_key.txt}`.

## How to do integration test of caddy-jwt locally?

For **caddy-jwt users**, we assume you've already got a custom caddy binary built with our caddy-jwt plugin. Then you can run the test:

```bash
echo '{
        order jwtauth before basicauth
}

:8080 {
        jwtauth {
                sign_key TkZMNSowQmMjOVU2RUB0bm1DJkU3U1VONkd3SGZMbVk=
                sign_alg HS256
                from_query access_token token
                from_header X-Api-Token
                from_cookies user_session
                user_claims aud uid user_id username login
        }

        respond "User authenticated with ID: {http.auth.user.id}"
}' > /tmp/caddy-jwt-test.Caddyfile

# ./caddy is your custom caddy built, see Install section above
./caddy run --config /tmp/caddy-jwt-test.Caddyfile

# This token won't expire until year 2285.
TEST_TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjk5NTU4OTI2NzAsImp0aSI6IjgyMjk0YTYzLTk2NjAtNGM2Mi1hOGE4LTVhNjI2NWVmY2Q0ZSIsInN1YiI6IjM0MDYzMjc5NjM1MTY5MzIiLCJpc3MiOiJodHRwczovL2FwaS5leGFtcGxlLmNvbSIsImF1ZCI6WyJodHRwczovL2FwaS5leGFtcGxlLmlvIl0sInVzZXJuYW1lIjoiZ2dpY2NpIn0.O8kvRO9y6xQO3AymqdFE7DDqLRBQhkntf78O9kF71F8

curl -v "http://localhost:8080?access_token=${TEST_TOKEN}"
# You should see
# 1. caddy log:
#   http.authentication.providers.jwt       user authenticated      {"token_string": "eyJhbGciOiJIUzI1…Qhkntf78O9kF71F8", "user_claim": "username", "id": "ggicci"}
#
# 2. request response (curl command output):
# User Authenticated with ID: ggicci

# And the following command should also work:
curl -v -H"X-Api-Token: ${TEST_TOKEN}" "http://localhost:8080"
curl -v -H"Authorization: Bearer ${TEST_TOKEN}" "http://localhost:8080"
```

**NOTE**: you can decode the `${TEST_TOKEN}` above at [jwt.io](https://jwt.io/) to get human readable payload as follows:

```json
{
  "exp": 9955892670,
  "jti": "82294a63-9660-4c62-a8a8-5a6265efcd4e",
  "sub": "3406327963516932",
  "iss": "https://api.example.com",
  "aud": ["https://api.example.io"],
  "username": "ggicci"
}
```

For **caddy-jwt developers**, you need to clone this repo, and start the caddy server in the repo folder:

```bash
git clone https://github.com/ggicci/caddy-jwt.git
cd caddy-jwt

# Build a caddy with this module and run an example server at localhost.
xcaddy run --config /tmp/caddy-jwt-test.Caddyfile
```

Any local code changes should reflect immediately.

## How caddy-jwt works?

Module **caddy-jwt** behaves like a **"JWT Validator"**. The authentication flow is:

```text
   ┌──────────────────┐
   │Extract token from│
   │  1. query        │
   │  2. header       │
   │  3. cookies      │
   └────────┬─────────┘
            │
    ┌───────▼───────────┐
    │     is valid?     │
    │  using `sign_key` │
    │ or validation is  │
    │     disabled      ├─NO───────┐
    └───────┬───────────┘          │
            │YES                   │
┌───────────▼───────────┐          │
│Populate {http.user.id}│          │
│  by `user_claims`     │          │
└───────────┬───────────┘          │
            │                      │
 ┌──────────▼───────────┐          │
 │is {http.user.id} set?├──NO(empty)
 └──────────┬───────────┘       │  │
            │YES(non-empty)     │  │
 ┌──────────▼───────────┐       │  │
 │Populate {http.user.*}│       │  │
 │   by `meta_claims`   │       │  │
 └──────────┬───────────┘       │  │
            │                   │  │
   ┌────────▼──────────┐ ┌──────▼──▼─────┐
   │   Authenticated   │ │Unauthenticated│
   │ Continue to Caddy │ │      401      │
   └───────────────────┘ └───────────────┘
```

flowchart by https://asciiflow.com/

## FAQ

**Q1**: How to deal with 401 responses on OPTIONS requests? (CORS related)

It should be handled separately by Caddy. Please read [#24](https://github.com/ggicci/caddy-jwt/issues/24) for more details.

**Q2**: What to note when using a public key as the value of `sign_key` in Caddyfile?

Using multi-line content in a directive [should be quoted](https://caddyserver.com/docs/caddyfile/concepts#tokens-and-quotes) as Caddy's documentation says. And the public key should be represented in PKCS#1 PEM format. Here's a simple command to derive such a public key from an RSA private key: `openssl rsa -in input.rsa -pubout`. Related: [#36](https://github.com/ggicci/caddy-jwt/issues/36).

## References

- **MUST READ**: [JWT Security Best Practices](https://curity.io/resources/learn/jwt-best-practices/)
- Online Debugers: http://jwt.io/, https://token.dev/jwt/
