```bash
kubectl -n <APP> exec -it deploy/<APP> -c istio-proxy -- sh
```

```yaml
0. Check: [sh, curl, grep, sed]
1. Client:
   ClientID: CLIENT_ID
   ClientSecret: CLIENT_SECRET
   MetadataProviderURL: https://…/.well-known/openid-configuration
   Scopes: [openid, email]
redirect_uri: https://.../kiali
.well-known:
  - authorization_endpoint
  - token_endpoint (/access_token)
  - jwks_uri
  - userinfo_endpoint
  - introspection_endpoint (/introspect)

# 1. issuer & .well-known
export OIDC_ISSUER="https://cidp.com/am/oauth2/global"
export OIDC_METADATA="${OIDC_ISSUER}/.well-known/openid-configuration"

# 2. Client
export OIDC_CLIENT_ID="Cloud-Global"
export OIDC_CLIENT_SECRET="***OIDC_CLIENT_SECRET***"

# 3. Scopes
export OIDC_SCOPE="openid email az1prod"

# 4. Redirect URI CIDP for Client
export OIDC_REDIRECT_URI="https://....com/kiali"

# 5. Endpoint and .well-known
export OIDC_AUTH_ENDPOINT="https://cidp.com/am/oauth2/global/authorize"
export OIDC_TOKEN_ENDPOINT="https://cidp.com/am/oauth2/global/access_token"
export OIDC_JWKS_URI="https://cidp.com/am/oauth2/global/connect/jwk_uri"
export OIDC_USERINFO_ENDPOINT="https://cidp.com/am/oauth2/global/userinfo"
export OIDC_INTROSPECT_ENDPOINT="https://cidp.com/am/oauth2/global/introspect"
```


```bash
test_metadata() {
  echo "== METADATA (.well-known) =="
  curl -sk "$OIDC_METADATA" \
    | sed 's/[{}]//g' \
    | tr ',' '\n' \
    | grep -E 'issuer|authorization_endpoint|token_endpoint|jwks_uri|userinfo_endpoint|introspect'
}

test_metadata
```

```bash
test_token_ping() {
  echo "== TOKEN ENDPOINT PING =="
  curl -sk -w "\nHTTP_STATUS=%{http_code}\n" \
    -X POST "$OIDC_TOKEN_ENDPOINT" \
    -d "grant_type=invalid_test_grant"
}

test_token_ping
```

```bash
test_client_credentials() {
  echo "== CLIENT CREDENTIALS GRANT =="
  curl -sk -w "\nHTTP_STATUS=%{http_code}\n" \
    -X POST "$OIDC_TOKEN_ENDPOINT" \
    -d "grant_type=client_credentials" \
    -d "client_id=$OIDC_CLIENT_ID" \
    -d "client_secret=$OIDC_CLIENT_SECRET" \
    -d "scope=$OIDC_SCOPE"
}

test_client_credentials
```

```bash
https://cidp.com/am/oauth2/global/authorize
  ?client_id=<CLIENT_UD>
  &response_type=code
  &redirect_uri=https%3A%2F%2F<URL_REDIRECT>%2Fkiali
  &scope=openid%20email%20az1prod
```

```bash
echo "$OIDC_AUTH_ENDPOINT?client_id=$OIDC_CLIENT_ID&response_type=code&redirect_uri=$(printf %s "$OIDC_REDIRECT_URI" | sed 's/:/%3A/g;s/\//%2F/g')&scope=$(echo "$OIDC_SCOPE" | sed 's/ /%20/g')"
```


```bash
test_auth_code() {
  CODE="$1"
  if [ -z "$CODE" ]; then
    echo "Usage: test_auth_code <CODE_FROM_BROWSER>"
    return 1
  fi

  echo "== AUTHORIZATION_CODE GRANT =="
  curl -sk -w "\nHTTP_STATUS=%{http_code}\n" \
    -X POST "$OIDC_TOKEN_ENDPOINT" \
    -d "grant_type=authorization_code" \
    -d "client_id=$OIDC_CLIENT_ID" \
    -d "client_secret=$OIDC_CLIENT_SECRET" \
    -d "redirect_uri=$OIDC_REDIRECT_URI" \
    -d "code=$CODE"
}


test_auth_code "PASTE_CODE_HERE"

```

```bash
test_jwks() {
  echo "== JWKS =="
  curl -sk "$OIDC_JWKS_URI" | sed 's/[{}]//g' | tr ',' '\n' | head -n 20
}

test_jwks
```

```bash
test_userinfo() {
  ACCESS_TOKEN="$1"
  if [ -z "$ACCESS_TOKEN" ]; then
    echo "Usage: test_userinfo <ACCESS_TOKEN>"
    return 1
  fi

  echo "== USERINFO =="
  curl -sk -H "Authorization: Bearer $ACCESS_TOKEN" "$OIDC_USERINFO_ENDPOINT"
}

# example:
# test_userinfo "eyJhbGciOiJ..."
```


```bash
test_introspect() {
  ACCESS_TOKEN="$1"
  if [ -z "$ACCESS_TOKEN" ]; then
    echo "Usage: test_introspect <ACCESS_TOKEN>"
    return 1
  fi

  echo "== INTROSPECT =="
  curl -sk -w "\nHTTP_STATUS=%{http_code}\n" \
    -u "$OIDC_CLIENT_ID:$OIDC_CLIENT_SECRET" \
    -d "token=$ACCESS_TOKEN" \
    "$OIDC_INTROSPECT_ENDPOINT"
}
```

