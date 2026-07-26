# WSO2-Tasks

A hands-on walkthrough of the WSO2 API Manager 4.7.0 platform, completing ten tasks that cover the full API lifecycle — from creating and securing an API to versioning, rate limiting, import/export automation, and observability.
 
The backend used throughout is the public **Swagger Petstore** (`https://petstore3.swagger.io/api/v3`). WSO2 API Manager sits *in front of* this backend as a managed gateway, adding OAuth2 security, rate limiting, versioning, documentation, and analytics — **without changing a single line of the backend's code**. That separation is the core idea an API gateway demonstrates, and every task below is a facet of it.
 
**Environment:** WSO2 API Manager 4.7.0 · apictl 4.7.1 · JDK 21 · Pop!_OS
 
The three consoles used, each mapping to a role:
- **Publisher** (`:9443/publisher`) — where API *producers* create, configure, and publish APIs
- **Developer Portal** (`:9443/devportal`) — where *consumers* discover, subscribe, and get keys
- **Admin** (`:9443/admin`) — platform governance: throttling policies, key managers
A recurring detail across the CLI tests: `-k` tells `curl`/`apictl` to accept WSO2's self-signed certificate, and the gateway runs two ports — `:9443` (OAuth token endpoint) and `:8243` (the API gateway that serves invocations).
 
---

<img width="1329" height="471" alt="image" src="https://github.com/user-attachments/assets/b7b02b01-7017-4801-b052-2a4b334ccec4" /># WSO2-Tasks

## Task 1
Created an API by importing the Petstore OpenAPI definition (reusing the existing contract rather than hand-defining every endpoint), set its identity (name, context `/petstore`, version `1.0.0`), and pointed it at the backend. Then walked the lifecycle: **CREATED → deploy a revision to the gateway → PUBLISHED**. Only a published, deployed API is reachable by consumers. Verified end-to-end by obtaining an OAuth2 token and calling the API for a `200` with live pet data.
 
*Debugging note:* the full security chain revealed itself as a sequence of errors — `NetworkError` until the gateway's self-signed cert was trusted, `invalid_client` from a lookalike-character typo in the consumer key (`I`/`l`/`i`), and `900910` scope failure until the application was subscribed to the API. Each error maps to a distinct gate: cert trust → valid token → active subscription.

<img width="825" height="674" alt="image" src="https://github.com/user-attachments/assets/2864306d-ef7b-4301-adb8-d93c13aba69f" />

<img width="1314" height="699" alt="image" src="https://github.com/user-attachments/assets/6f8a1c4f-14ff-491e-9b41-686278e1ff83" />



<img width="1314" height="699" alt="image" src="https://github.com/user-attachments/assets/adca33b4-4247-4f67-9335-aa13500093f5" />

<img width="1314" height="699" alt="image" src="https://github.com/user-attachments/assets/3391cf3d-c1dd-4ff7-b6f8-f95fc06e4aff" />

<img width="1312" height="4253" alt="image" src="https://github.com/user-attachments/assets/ec7c9861-7dd0-458a-b00e-8f1f9795479a" />

## Task 2
Added a resource to the API and redeployed a new revision. A **resource** is an HTTP method + URI pattern, and it defines *what the gateway will route*. Proved the concept with a clean contrast: a call to the newly-added path returned a **400 from the backend** (gateway forwarded it, backend rejected the value), while a call to a genuinely unknown path returned a **404 from the gateway** ("No matching resource"). Same-looking failures, different sources — that difference *is* what adding a resource controls. The redeploy-a-new-revision step is what makes any change live; editing the design alone changes nothing on the gateway.
<img width="1168" height="486" alt="image" src="https://github.com/user-attachments/assets/e1459b85-db10-4580-a24a-471a2326d3a0" />

<img width="1203" height="607" alt="image" src="https://github.com/user-attachments/assets/9ee0fd74-f90f-4b69-a09c-debe2d301034" />

## Task 3
Confirmed OAuth2 as the API's application-level security (Publisher → Runtime Configurations) and verified the full security guarantee from both sides: a valid, subscribed token returns `200`; a request with **no token** is rejected at the gateway with `401 / 900902 Missing Credentials`, never reaching the backend. This makes concrete the three-object model — **API** (declares security), **Application** (holds credentials), **Subscription** (the required link between them) — and the OAuth2 client-credentials flow (key + secret → token → `Authorization: Bearer`). The gateway also returns a `WWW-Authenticate` header telling callers *how* to authenticate.
<img width="1084" height="1120" alt="image" src="https://github.com/user-attachments/assets/b148f0d3-27c0-4d9f-a4e1-be072696bd85" />
```console 
curl -k -i -X GET "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available"
HTTP/1.1 401 Unauthorized
activityid: 622760a6-afd5-4f90-8b2e-f7eb10f3f9be
Access-Control-Allow-Origin: *
WWW-Authenticate: Internal API Key realm="WSO2 API Manager", Bearer realm="WSO2 API Manager", error="invalid_token", error_description="The provided token is invalid"
Content-Type: application/json; charset=UTF-8
Date: Sun, 26 Jul 2026 09:53:41 GMT
Transfer-Encoding: chunked
{"code":"900902","message":"Missing Credentials","description":"Invalid Credentials. Make sure your API invocation call has a header: 'Authorization : Bearer ACCESS_TOKEN' or 'Authorization : Basic ACCESS_TOKEN' or 'ApiKey : API_KEY'"}loayelnoamani@pop-os:~$
```
## Task 4
Created a custom `10PerMin` subscription policy in the Admin Portal, attached it to the API's Business Plans (Publisher), and selected it when subscribing (Dev Portal) — three consoles, three roles. Then fired 15 requests in a loop and watched the gateway flip from `200` to **`429 Too Many Requests`** once the per-minute allowance was spent. Throttling is a protective valve: once exceeded, the gateway stops forwarding to the backend.
 
*Debugging note:* a valid, subscribed token was still rejected with `900910`. Decoding the JWT payload showed `"scope":"default"` instead of the `read:pets write:pets` the pet resources required — the resource-level scopes (inherited from the OpenAPI import) were blocking a client-credentials token. Since the task's security model is OAuth2 + subscription rather than fine-grained per-scope access, removing the resource scopes was the correct, simplest fix — after which the throttling test produced the expected `200 → 429` transition.

<img width="1281" height="150" alt="image" src="https://github.com/user-attachments/assets/fd190d2a-1067-4277-855f-43ef1f8ca3a1" />

<img width="1329" height="471" alt="image" src="https://github.com/user-attachments/assets/4c8947c2-5089-4d3a-9f05-14ed9fbb3ab7" />
<img width="1329" height="471" alt="image" src="https://github.com/user-attachments/assets/6528178b-7d6f-40d4-b628-710777324c5a" />
```console
loayelnoamani@pop-os:~$ TOKEN=$(curl -k -s -X POST https://localhost:9443/oauth2/token -d "grant_type=client_credentials" -u "$CONSUMER_KEY:$CONSUMER_SECRET" | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)
```
```console
loayelnoamani@pop-os:~$ curl -k -s -o /dev/null -w "%{http_code}\n" "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
200
```
```console
loayelnoamani@pop-os:~$ for i in $(seq 1 15); do
  code=$(curl -k -s -o /dev/null -w "%{http_code}" \
    "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available" \
    -H "Authorization: Bearer $TOKEN")
  echo "Request $i -> $code"
done
Request 1 -> 200
Request 2 -> 200
Request 3 -> 200
Request 4 -> 200
Request 5 -> 200
Request 6 -> 200
Request 7 -> 200
Request 8 -> 200
Request 9 -> 200
Request 10 -> 200
Request 11 -> 200
Request 12 -> 200
Request 13 -> 429
Request 14 -> 429
Request 15 -> 429
loayelnoamani@pop-os:~$
```
## Task 5
Cloned `1.0.0` into a new, independent `2.0.0` (carrying all config forward), then deployed and published it alongside the original. Verified both versions respond independently — the only difference between the two working calls being `1.0.0` vs `2.0.0` in the gateway path. Because each version is a separate object, they're subscribed to independently; but a single token from one application authorized **both**, since a token belongs to the application, not the API. This is how APIs evolve without breaking existing consumers: old and new run side by side.
<img width="1327" height="697" alt="image" src="https://github.com/user-attachments/assets/e7893125-cdd6-4441-a505-98c03c5bceb4" />
<img width="1327" height="697" alt="image" src="https://github.com/user-attachments/assets/d1fcc282-cdbb-4e8e-86a0-7a5b4be343b3" />
https://localhost:8243/petstore/1.0.0/pet/findByStatus
https://localhost:8243/petstore/2.0.0/pet/findByStatus
same output
```console
loayelnoamani@pop-os:~$ TOKEN=$(curl -k -s -X POST https://localhost:9443/oauth2/token -d "grant_type=client_credentials" -u "$CONSUMER_KEY:$CONSUMER_SECRET" | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)
```
```console
loayelnoamani@pop-os:~$ echo "--- v1.0.0 ---"
curl -k -s -o /dev/null -w "%{http_code}\n" "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
--- v1.0.0 ---
200
```
```console
loayelnoamani@pop-os:~$ echo "--- v2.0.0 ---"
curl -k -s -o /dev/null -w "%{http_code}\n" "https://localhost:8243/petstore/2.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
--- v2.0.0 ---
200
loayelnoamani@pop-os:~$
```
## Task 6
Updated the consumer-facing metadata: a clear description (framing the API as a *managed proxy* rather than the raw Petstore), tags for discoverability, an uploaded icon, and business-owner/technical-owner contact info. Metadata is catalog/display information (not gateway routing), so it surfaces in the Developer Portal on save without a redeploy. Good metadata is the difference between an API that looks maintained and one consumers pass over.
<img width="1312" height="1533" alt="image" src="https://github.com/user-attachments/assets/20e7ff6a-488f-47ef-acff-2b9d212177eb" />

<img width="1327" height="697" alt="image" src="https://github.com/user-attachments/assets/076d2bec-27c9-41b8-a27a-6a8b42349fc9" />

# Task 7
Stepped fully into the consumer's role: created a **new** application (`LoayTestApp`), subscribed it to two APIs (versions `1.0.0` and `2.0.0`), generated its own **production** keys, and tested both — all `200`. Because this is a second, independent application, it has entirely different credentials from `DefaultApplication` — reinforcing that keys belong to the application. One app, many APIs, one token authorizing all of them.
 

<img width="1272" height="70" alt="image" src="https://github.com/user-attachments/assets/c775695f-6220-412d-b2ca-5aa7c00c1337" />

<img width="876" height="497" alt="image" src="https://github.com/user-attachments/assets/71e034f4-cfe4-45fc-927f-fe4793e61864" />

## Task 8
 
Used the WSO2 API Controller (`apictl`) to run the professional dev-first workflow: **export → delete → import → verify**. Exported `1.0.0` as a portable "API source project" (a zip containing the OpenAPI definition, WSO2 config, icon, and metadata), deleted it from the server, then re-imported it from the zip and confirmed it came back fully intact and serving traffic (`200`). This is how APIs move between environments (dev → staging → prod) and get version-controlled in Git.
 
*Debugging notes:* two real-world issues surfaced. First, `apictl 4.0.5` returned empty API lists and `404`s — verbose mode showed it calling the `v2` Publisher REST endpoint, which the `4.7.0` server doesn't expose; matching the tool to the server (`apictl 4.7.1`, hitting `v4`) fixed it. Second, deleting the API failed with `409 Conflict` — "active subscriptions exist" — because WSO2 refuses to orphan active consumers. Removing the subscriptions first allowed the delete. The re-imported API also received a *new* internal UUID while keeping its consumer-facing identity (`/petstore/1.0.0`) unchanged — the import is a faithful restore of the artifact, not the database row.

<img width="882" height="192" alt="image" src="https://github.com/user-attachments/assets/4f0d0851-6426-4673-8836-01a9d054d9be" />

<img width="854" height="231" alt="image" src="https://github.com/user-attachments/assets/ad549c00-6697-40f4-80fe-6e78dd6b673f" />
<img width="834" height="150" alt="image" src="https://github.com/user-attachments/assets/84455b81-f6a9-42c0-8fb7-d5e9cc0ceecf" />

## Task 9
Authored an inline "Getting Started" document on the API (Publisher → Documents) — an authentication how-to plus a worked example of the `GET /pet/findByStatus` endpoint with sample requests — and verified it renders on the consumer-facing Developer Portal. Documentation is content authored by the producer to help consumers adopt the API; it publishes with the API and needs no separate lifecycle step.
<img width="1072" height="286" alt="image" src="https://github.com/user-attachments/assets/c7cd4a42-7ce5-405e-980b-5dc6993c4bc7" />
<img width="1134" height="589" alt="image" src="https://github.com/user-attachments/assets/0a00a6d0-7da1-42a8-83fa-c817f8473f2b" />

## Task 10
Generated a mix of API traffic (successes, unknown-path calls, and a no-token call) and inspected the gateway logs. The default `wso2carbon.log` records auth/authorization *failures* at WARN with the request path, outcome reason, and timestamp. For complete per-request observability, enabled **per-API FULL logging** via apictl — without restarting the server:
```console 
loayelnoamani@pop-os:~/wso2-export$ grep "findByStatus\|fakepath\|/pet/" ~/wso2am-4.7.0/repository/logs/wso2-apigw-service.log | tail -20
2026-07-26T18:35:08,455 [-] [PassThroughMessageProcessor-1]  INFO __SynapseService STATUS = Message dispatched to the main sequence. Invalid URL., RESOURCE = /petstore/1.0.0/pet/findByStatus?status=available, HEALTH CHECK URL = /petstore/1.0.0/pet/findByStatus?status=available
loayelnoamani@pop-os:~/wso2-export$ grep "petstore/1.0.0/pet" ~/wso2am-4.7.0/repository/logs/wso2carbon.log | tail -20
TID: [] [] [2026-07-26 15:27:30,869]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,887]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,903]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,919]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,935]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,952]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,968]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:30,984]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:31,000]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:31,017]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:31,033]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:27:31,049]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:28:39,475]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:29:47,453]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:39:41,509]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 15:40:40,960]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=DefaultApplication for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 16:02:13,540]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to Missing Credentials for requestURI=/petstore/1.0.0/pet/findByStatus
TID: [] [] [2026-07-26 18:35:08,455]  INFO {org.apache.synapse.mediators.builtin.LogMediator} - STATUS = Message dispatched to the main sequence. Invalid URL., RESOURCE = /petstore/1.0.0/pet/findByStatus?status=available, HEALTH CHECK URL = /petstore/1.0.0/pet/findByStatus?status=available
TID: [] [] [2026-07-26 18:46:02,828]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to The access token does not allow you to access the requested resource for appName=LoayTestApp for requestURI=/petstore/1.0.0/pet/fakepath123
TID: [] [] [2026-07-26 18:46:15,557]  WARN {org.wso2.carbon.apimgt.gateway.handlers.security.APIAuthenticationHandler} - API authentication failure due to Missing Credentials for requestURI=/petstore/1.0.0/pet/findByStatus?status=available
loayelnoamani@pop-os:~/wso2-export$
```
```console
loayelnoamani@pop-os:~/wso2-export$ TOKEN=$(curl -k -s -X POST https://localhost:9443/oauth2/token -d "grant_type=client_credentials" -u "$TESTAPP_KEY:$TESTAPP_SECRET" | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

# 5 successful calls
for i in $(seq 1 5); do
  curl -k -s -o /dev/null "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
done

# 1 no-token call (401) for variety
curl -k -s -o /dev/null "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available"

echo "Traffic sent."
Traffic sent.
loayelnoamani@pop-os:~/wso2-export$ tail -20 ~/wso2am-4.7.0/repository/logs/api.log
[2026-07-26 18:57:09,447]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Accept=*/*","activityid=3004cafa-8d90-4205-b6a5-8f49f4838adf","Authorization=Bearer eyJ4NXQiOiJZMkUxWXpsaU5UUm1PV0kyWkRNNVpqTTVObUV4WVdGaE4yRTNPREEwT0RFek9UTmpNV1JtWW1JMU5qZGpabUZqWm1JNU5qSmxObUU0T0dGa05qVTRPQSIsImtpZCI6IlkyRTFZemxpTlRSbU9XSTJaRE01WmpNNU5tRXhZV0ZoTjJFM09EQTBPREV6T1ROak1XUm1ZbUkxTmpkalptRmpabUk1TmpKbE5tRTRPR0ZrTmpVNE9BX1JTMjU2IiwidHlwIjoiYXQrand0IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJmMzRlOTdjYi0yNzRkLTRiNDQtYWQwOC05NGM0ODIxNzFhYjkiLCJhdXQiOiJBUFBMSUNBVElPTiIsImlzcyI6Imh0dHBzOi8vbG9jYWxob3N0Ojk0NDMvb2F1dGgyL3Rva2VuIiwiZW50aXR5X2lkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsImNsaWVudF9pZCI6IjhZZlJoclNmMkNYRFh6UTlMU0dTUzFTQ0o1SWEiLCJ1c2lkIjoiNGEzNzI1YzgtNjU2Ny00YjM5LWI1NzEtN2FiYmQ2ZjgyYWMxIiwiYXVkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsIm5iZiI6MTc4NTA4MTQyOSwiYXpwIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsInNjb3BlIjoiZGVmYXVsdCIsImV4cCI6MTc4NTA4NTAyOSwiaWF0IjoxNzg1MDgxNDI5LCJqdGkiOiJiODBlZTNiOC00YTBlLTQ0ZTEtYTljYS1mMTllOWNjYjU1NzgifQ.X37Z7Y_t038Obi2CPQkuJb_E6QWSX8WhXPdHmxD-s_-clbV1Hx4RjQ46vYkV6ubiQWPNbBS8suiNzWk4orVDwgyjytxLi40U8Gf-M2pRbYMiqaY33uPcr6Hu_MF0nbqws0E7qsP15QPIwv3YsL7BkVcHwNJNXbPLoDMFSAbYvHldQl8o9cWbBB9M4QPIdgqbpKr8gucjN7R0vP3OvVbx8sZ3r5tzOVhePRm1yu6NAoqDa4fVJxxorm6NIkTb9uWthtqoM6QBplSQ_xCg1EMnr3J91-3OmjeVbTjrU3EFubAwT3nNxHc7QXxHGKxd9Wbk2rtueF1_NyAokaVB8bykCQ","Host=localhost:8243","User-Agent=curl/8.5.0"],"sourceIP":"127.0.0.1","payload":"<?xml version='1.0' encoding='utf-8'?><soapenv:Envelope xmlns:soapenv=\"http://www.w3.org/2003/05/soap-envelope\"><soapenv:Body/><\/soapenv:Envelope>","verb":"GET","correlationId":"3004cafa-8d90-4205-b6a5-8f49f4838adf","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"REQUEST_IN"}
[2026-07-26 18:57:09,449]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Access-Control-Allow-Origin=*","activityid=3004cafa-8d90-4205-b6a5-8f49f4838adf","Content-Type=application/json","User-Agent=curl/8.5.0"],"payload":"{\"code\":\"900910\",\"message\":\"The access token does not allow you to access the requested resource\",\"description\":\"User is NOT authorized to access the Resource: /pet/{petId}. Scope validation failed.\"}","verb":"GET","correlationId":"3004cafa-8d90-4205-b6a5-8f49f4838adf","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"RESPONSE_OUT","statusCode":403}
[2026-07-26 18:57:09,465]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Accept=*/*","activityid=17c7da5b-ddd1-4d70-ac81-e212765b622b","Authorization=Bearer eyJ4NXQiOiJZMkUxWXpsaU5UUm1PV0kyWkRNNVpqTTVObUV4WVdGaE4yRTNPREEwT0RFek9UTmpNV1JtWW1JMU5qZGpabUZqWm1JNU5qSmxObUU0T0dGa05qVTRPQSIsImtpZCI6IlkyRTFZemxpTlRSbU9XSTJaRE01WmpNNU5tRXhZV0ZoTjJFM09EQTBPREV6T1ROak1XUm1ZbUkxTmpkalptRmpabUk1TmpKbE5tRTRPR0ZrTmpVNE9BX1JTMjU2IiwidHlwIjoiYXQrand0IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJmMzRlOTdjYi0yNzRkLTRiNDQtYWQwOC05NGM0ODIxNzFhYjkiLCJhdXQiOiJBUFBMSUNBVElPTiIsImlzcyI6Imh0dHBzOi8vbG9jYWxob3N0Ojk0NDMvb2F1dGgyL3Rva2VuIiwiZW50aXR5X2lkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsImNsaWVudF9pZCI6IjhZZlJoclNmMkNYRFh6UTlMU0dTUzFTQ0o1SWEiLCJ1c2lkIjoiNGEzNzI1YzgtNjU2Ny00YjM5LWI1NzEtN2FiYmQ2ZjgyYWMxIiwiYXVkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsIm5iZiI6MTc4NTA4MTQyOSwiYXpwIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsInNjb3BlIjoiZGVmYXVsdCIsImV4cCI6MTc4NTA4NTAyOSwiaWF0IjoxNzg1MDgxNDI5LCJqdGkiOiJiODBlZTNiOC00YTBlLTQ0ZTEtYTljYS1mMTllOWNjYjU1NzgifQ.X37Z7Y_t038Obi2CPQkuJb_E6QWSX8WhXPdHmxD-s_-clbV1Hx4RjQ46vYkV6ubiQWPNbBS8suiNzWk4orVDwgyjytxLi40U8Gf-M2pRbYMiqaY33uPcr6Hu_MF0nbqws0E7qsP15QPIwv3YsL7BkVcHwNJNXbPLoDMFSAbYvHldQl8o9cWbBB9M4QPIdgqbpKr8gucjN7R0vP3OvVbx8sZ3r5tzOVhePRm1yu6NAoqDa4fVJxxorm6NIkTb9uWthtqoM6QBplSQ_xCg1EMnr3J91-3OmjeVbTjrU3EFubAwT3nNxHc7QXxHGKxd9Wbk2rtueF1_NyAokaVB8bykCQ","Host=localhost:8243","User-Agent=curl/8.5.0"],"sourceIP":"127.0.0.1","payload":"<?xml version='1.0' encoding='utf-8'?><soapenv:Envelope xmlns:soapenv=\"http://www.w3.org/2003/05/soap-envelope\"><soapenv:Body/><\/soapenv:Envelope>","verb":"GET","correlationId":"17c7da5b-ddd1-4d70-ac81-e212765b622b","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"REQUEST_IN"}
[2026-07-26 18:57:09,466]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Access-Control-Allow-Origin=*","activityid=17c7da5b-ddd1-4d70-ac81-e212765b622b","Content-Type=application/json","User-Agent=curl/8.5.0"],"payload":"{\"code\":\"900910\",\"message\":\"The access token does not allow you to access the requested resource\",\"description\":\"User is NOT authorized to access the Resource: /pet/{petId}. Scope validation failed.\"}","verb":"GET","correlationId":"17c7da5b-ddd1-4d70-ac81-e212765b622b","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"RESPONSE_OUT","statusCode":403}
[2026-07-26 18:57:09,481]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Accept=*/*","activityid=dab4ff9a-e41d-418f-9af0-32b21c36b43d","Authorization=Bearer eyJ4NXQiOiJZMkUxWXpsaU5UUm1PV0kyWkRNNVpqTTVObUV4WVdGaE4yRTNPREEwT0RFek9UTmpNV1JtWW1JMU5qZGpabUZqWm1JNU5qSmxObUU0T0dGa05qVTRPQSIsImtpZCI6IlkyRTFZemxpTlRSbU9XSTJaRE01WmpNNU5tRXhZV0ZoTjJFM09EQTBPREV6T1ROak1XUm1ZbUkxTmpkalptRmpabUk1TmpKbE5tRTRPR0ZrTmpVNE9BX1JTMjU2IiwidHlwIjoiYXQrand0IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJmMzRlOTdjYi0yNzRkLTRiNDQtYWQwOC05NGM0ODIxNzFhYjkiLCJhdXQiOiJBUFBMSUNBVElPTiIsImlzcyI6Imh0dHBzOi8vbG9jYWxob3N0Ojk0NDMvb2F1dGgyL3Rva2VuIiwiZW50aXR5X2lkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsImNsaWVudF9pZCI6IjhZZlJoclNmMkNYRFh6UTlMU0dTUzFTQ0o1SWEiLCJ1c2lkIjoiNGEzNzI1YzgtNjU2Ny00YjM5LWI1NzEtN2FiYmQ2ZjgyYWMxIiwiYXVkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsIm5iZiI6MTc4NTA4MTQyOSwiYXpwIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsInNjb3BlIjoiZGVmYXVsdCIsImV4cCI6MTc4NTA4NTAyOSwiaWF0IjoxNzg1MDgxNDI5LCJqdGkiOiJiODBlZTNiOC00YTBlLTQ0ZTEtYTljYS1mMTllOWNjYjU1NzgifQ.X37Z7Y_t038Obi2CPQkuJb_E6QWSX8WhXPdHmxD-s_-clbV1Hx4RjQ46vYkV6ubiQWPNbBS8suiNzWk4orVDwgyjytxLi40U8Gf-M2pRbYMiqaY33uPcr6Hu_MF0nbqws0E7qsP15QPIwv3YsL7BkVcHwNJNXbPLoDMFSAbYvHldQl8o9cWbBB9M4QPIdgqbpKr8gucjN7R0vP3OvVbx8sZ3r5tzOVhePRm1yu6NAoqDa4fVJxxorm6NIkTb9uWthtqoM6QBplSQ_xCg1EMnr3J91-3OmjeVbTjrU3EFubAwT3nNxHc7QXxHGKxd9Wbk2rtueF1_NyAokaVB8bykCQ","Host=localhost:8243","User-Agent=curl/8.5.0"],"sourceIP":"127.0.0.1","payload":"<?xml version='1.0' encoding='utf-8'?><soapenv:Envelope xmlns:soapenv=\"http://www.w3.org/2003/05/soap-envelope\"><soapenv:Body/><\/soapenv:Envelope>","verb":"GET","correlationId":"dab4ff9a-e41d-418f-9af0-32b21c36b43d","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"REQUEST_IN"}
[2026-07-26 18:57:09,483]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Access-Control-Allow-Origin=*","activityid=dab4ff9a-e41d-418f-9af0-32b21c36b43d","Content-Type=application/json","User-Agent=curl/8.5.0"],"payload":"{\"code\":\"900910\",\"message\":\"The access token does not allow you to access the requested resource\",\"description\":\"User is NOT authorized to access the Resource: /pet/{petId}. Scope validation failed.\"}","verb":"GET","correlationId":"dab4ff9a-e41d-418f-9af0-32b21c36b43d","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"RESPONSE_OUT","statusCode":403}
[2026-07-26 18:57:09,498]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Accept=*/*","activityid=0ed9f67f-9ee9-4565-9185-efbb909dda19","Authorization=Bearer eyJ4NXQiOiJZMkUxWXpsaU5UUm1PV0kyWkRNNVpqTTVObUV4WVdGaE4yRTNPREEwT0RFek9UTmpNV1JtWW1JMU5qZGpabUZqWm1JNU5qSmxObUU0T0dGa05qVTRPQSIsImtpZCI6IlkyRTFZemxpTlRSbU9XSTJaRE01WmpNNU5tRXhZV0ZoTjJFM09EQTBPREV6T1ROak1XUm1ZbUkxTmpkalptRmpabUk1TmpKbE5tRTRPR0ZrTmpVNE9BX1JTMjU2IiwidHlwIjoiYXQrand0IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJmMzRlOTdjYi0yNzRkLTRiNDQtYWQwOC05NGM0ODIxNzFhYjkiLCJhdXQiOiJBUFBMSUNBVElPTiIsImlzcyI6Imh0dHBzOi8vbG9jYWxob3N0Ojk0NDMvb2F1dGgyL3Rva2VuIiwiZW50aXR5X2lkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsImNsaWVudF9pZCI6IjhZZlJoclNmMkNYRFh6UTlMU0dTUzFTQ0o1SWEiLCJ1c2lkIjoiNGEzNzI1YzgtNjU2Ny00YjM5LWI1NzEtN2FiYmQ2ZjgyYWMxIiwiYXVkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsIm5iZiI6MTc4NTA4MTQyOSwiYXpwIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsInNjb3BlIjoiZGVmYXVsdCIsImV4cCI6MTc4NTA4NTAyOSwiaWF0IjoxNzg1MDgxNDI5LCJqdGkiOiJiODBlZTNiOC00YTBlLTQ0ZTEtYTljYS1mMTllOWNjYjU1NzgifQ.X37Z7Y_t038Obi2CPQkuJb_E6QWSX8WhXPdHmxD-s_-clbV1Hx4RjQ46vYkV6ubiQWPNbBS8suiNzWk4orVDwgyjytxLi40U8Gf-M2pRbYMiqaY33uPcr6Hu_MF0nbqws0E7qsP15QPIwv3YsL7BkVcHwNJNXbPLoDMFSAbYvHldQl8o9cWbBB9M4QPIdgqbpKr8gucjN7R0vP3OvVbx8sZ3r5tzOVhePRm1yu6NAoqDa4fVJxxorm6NIkTb9uWthtqoM6QBplSQ_xCg1EMnr3J91-3OmjeVbTjrU3EFubAwT3nNxHc7QXxHGKxd9Wbk2rtueF1_NyAokaVB8bykCQ","Host=localhost:8243","User-Agent=curl/8.5.0"],"sourceIP":"127.0.0.1","payload":"<?xml version='1.0' encoding='utf-8'?><soapenv:Envelope xmlns:soapenv=\"http://www.w3.org/2003/05/soap-envelope\"><soapenv:Body/><\/soapenv:Envelope>","verb":"GET","correlationId":"0ed9f67f-9ee9-4565-9185-efbb909dda19","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"REQUEST_IN"}
[2026-07-26 18:57:09,499]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Access-Control-Allow-Origin=*","activityid=0ed9f67f-9ee9-4565-9185-efbb909dda19","Content-Type=application/json","User-Agent=curl/8.5.0"],"payload":"{\"code\":\"900910\",\"message\":\"The access token does not allow you to access the requested resource\",\"description\":\"User is NOT authorized to access the Resource: /pet/{petId}. Scope validation failed.\"}","verb":"GET","correlationId":"0ed9f67f-9ee9-4565-9185-efbb909dda19","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"RESPONSE_OUT","statusCode":403}
[2026-07-26 18:57:09,514]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Accept=*/*","activityid=186309a5-f10a-41bd-8328-f8edebe4996e","Authorization=Bearer eyJ4NXQiOiJZMkUxWXpsaU5UUm1PV0kyWkRNNVpqTTVObUV4WVdGaE4yRTNPREEwT0RFek9UTmpNV1JtWW1JMU5qZGpabUZqWm1JNU5qSmxObUU0T0dGa05qVTRPQSIsImtpZCI6IlkyRTFZemxpTlRSbU9XSTJaRE01WmpNNU5tRXhZV0ZoTjJFM09EQTBPREV6T1ROak1XUm1ZbUkxTmpkalptRmpabUk1TmpKbE5tRTRPR0ZrTmpVNE9BX1JTMjU2IiwidHlwIjoiYXQrand0IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJmMzRlOTdjYi0yNzRkLTRiNDQtYWQwOC05NGM0ODIxNzFhYjkiLCJhdXQiOiJBUFBMSUNBVElPTiIsImlzcyI6Imh0dHBzOi8vbG9jYWxob3N0Ojk0NDMvb2F1dGgyL3Rva2VuIiwiZW50aXR5X2lkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsImNsaWVudF9pZCI6IjhZZlJoclNmMkNYRFh6UTlMU0dTUzFTQ0o1SWEiLCJ1c2lkIjoiNGEzNzI1YzgtNjU2Ny00YjM5LWI1NzEtN2FiYmQ2ZjgyYWMxIiwiYXVkIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsIm5iZiI6MTc4NTA4MTQyOSwiYXpwIjoiOFlmUmhyU2YyQ1hEWHpROUxTR1NTMVNDSjVJYSIsInNjb3BlIjoiZGVmYXVsdCIsImV4cCI6MTc4NTA4NTAyOSwiaWF0IjoxNzg1MDgxNDI5LCJqdGkiOiJiODBlZTNiOC00YTBlLTQ0ZTEtYTljYS1mMTllOWNjYjU1NzgifQ.X37Z7Y_t038Obi2CPQkuJb_E6QWSX8WhXPdHmxD-s_-clbV1Hx4RjQ46vYkV6ubiQWPNbBS8suiNzWk4orVDwgyjytxLi40U8Gf-M2pRbYMiqaY33uPcr6Hu_MF0nbqws0E7qsP15QPIwv3YsL7BkVcHwNJNXbPLoDMFSAbYvHldQl8o9cWbBB9M4QPIdgqbpKr8gucjN7R0vP3OvVbx8sZ3r5tzOVhePRm1yu6NAoqDa4fVJxxorm6NIkTb9uWthtqoM6QBplSQ_xCg1EMnr3J91-3OmjeVbTjrU3EFubAwT3nNxHc7QXxHGKxd9Wbk2rtueF1_NyAokaVB8bykCQ","Host=localhost:8243","User-Agent=curl/8.5.0"],"sourceIP":"127.0.0.1","payload":"<?xml version='1.0' encoding='utf-8'?><soapenv:Envelope xmlns:soapenv=\"http://www.w3.org/2003/05/soap-envelope\"><soapenv:Body/><\/soapenv:Envelope>","verb":"GET","correlationId":"186309a5-f10a-41bd-8328-f8edebe4996e","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"REQUEST_IN"}
[2026-07-26 18:57:09,516]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Access-Control-Allow-Origin=*","activityid=186309a5-f10a-41bd-8328-f8edebe4996e","Content-Type=application/json","User-Agent=curl/8.5.0"],"payload":"{\"code\":\"900910\",\"message\":\"The access token does not allow you to access the requested resource\",\"description\":\"User is NOT authorized to access the Resource: /pet/{petId}. Scope validation failed.\"}","verb":"GET","correlationId":"186309a5-f10a-41bd-8328-f8edebe4996e","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"RESPONSE_OUT","statusCode":403}
[2026-07-26 18:57:09,538]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Accept=*/*","activityid=dfe5facf-1688-4763-a149-6e5f1798f7ce","Host=localhost:8243","User-Agent=curl/8.5.0"],"sourceIP":"127.0.0.1","payload":"<?xml version='1.0' encoding='utf-8'?><soapenv:Envelope xmlns:soapenv=\"http://www.w3.org/2003/05/soap-envelope\"><soapenv:Body/><\/soapenv:Envelope>","verb":"GET","correlationId":"dfe5facf-1688-4763-a149-6e5f1798f7ce","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"REQUEST_IN"}
[2026-07-26 18:57:09,539]  INFO {API_LOG} SwaggerPetstore-OpenAPI30 - {"headers":["Access-Control-Allow-Origin=*","activityid=dfe5facf-1688-4763-a149-6e5f1798f7ce","Content-Type=application/json","User-Agent=curl/8.5.0","WWW-Authenticate=Internal API Key realm=\"WSO2 API Manager\", Bearer realm=\"WSO2 API Manager\", error=\"invalid_token\", error_description=\"The provided token is invalid\""],"payload":"{\"code\":\"900902\",\"message\":\"Missing Credentials\",\"description\":\"Invalid Credentials. Make sure your API invocation call has a header: 'Authorization : Bearer ACCESS_TOKEN' or 'Authorization : Basic ACCESS_TOKEN' or 'ApiKey : API_KEY'\"}","verb":"GET","correlationId":"dfe5facf-1688-4763-a149-6e5f1798f7ce","apiTo":"/petstore/1.0.0/pet/findByStatus?status=available","flow":"RESPONSE_OUT","statusCode":401}
loayelnoamani@pop-os:~/wso2-export$
```

## Task 10 — Analytics & Logs Summary

I enabled per-API FULL logging on the Petstore API (1.0.0) using apictl:

    apictl set api-logging --api-id <API-UUID> --log-level FULL -e local -k

Per-API logs are written to repository/logs/api.log as JSON, one entry per
REQUEST_IN and RESPONSE_OUT, capturing the fields the task asks for:

- Request path:    "apiTo":"/petstore/1.0.0/pet/findByStatus?status=available"
- HTTP method:     "verb":"GET"
- Response status: "statusCode":403 / 401
- Invocation time: [2026-07-26 18:57:09,447]
- Correlation ID:  links each request to its response

Observed outcomes from my test traffic:
- 403 (900910, "Scope validation failed") — token lacked the resource scope
- 401 (900902, "Missing Credentials")     — call sent without a token

Where invocations are logged (three complementary logs):
- api.log             — per-API FULL logs (enabled via apictl); richest, per-request
- wso2carbon.log      — operational log; auth/authz failures at WARN
- http_access_*.log   — servlet-transport access log (internal metadata calls)

Note: per-API logging is disabled by default for performance and enabled
on demand per API — a lightweight alternative to full analytics.
