---
layout: post
title: "Cognito - API Gateway - bash"
date: 2022-01-13
categories: scripts, cognito, api, bash
github_comments_issueid: 6
---

I needed to do some testing of one of our APIs that uses Cognito machine to machine authentication and mTLS but without using Postman - far too easy. API Gateway uses the mTLS certificate chain of public keys, the request uses the private key and certificate authority certificate.

Getting the access token from Cognito.

```(bash)
CLIENT_ID='4cu8xxxxxxxxxxxxxxx4kcl6q'
CLIENT_SECRET='vh8a2xxxxxxxxxxxxxxxxxxxxcak2crpkt6lnq87iahp3gbn'
SCOPE='xxxxx/yyyyy'

TOKEN=$(curl --location --request POST "https://xxxxxxxxxx.auth.eu-west-1.amazoncognito.com/oauth2/token?grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&scope=$SCOPE" \
--header 'Content-Type: application/x-www-form-urlencoded' | jq .access_token | tr -d '"')
```

The pipe to tr removes the quotes around the token.

Using the token in an API call and mtls certs.

```(bash)

API_REQUEST_URL='https://my_api_endpoint.mydomain.com/app_path'

curl --cert "client.crt" --key "client.key" --cacert "ca.pem" \
--location --request GET "$API_REQUEST_URL" \
--header "Authorization: Bearer $TOKEN"
```

<script src="https://utteranc.es/client.js"
        repo="indigo-tangerine/indigo-tangerine.github.io"
        issue-term="title"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
