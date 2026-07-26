<img width="1329" height="471" alt="image" src="https://github.com/user-attachments/assets/b7b02b01-7017-4801-b052-2a4b334ccec4" /># WSO2-Tasks

## Task 1
<img width="825" height="674" alt="image" src="https://github.com/user-attachments/assets/2864306d-ef7b-4301-adb8-d93c13aba69f" />

<img width="1314" height="699" alt="image" src="https://github.com/user-attachments/assets/6f8a1c4f-14ff-491e-9b41-686278e1ff83" />

<img width="1314" height="699" alt="image" src="https://github.com/user-attachments/assets/adca33b4-4247-4f67-9335-aa13500093f5" />

<img width="1314" height="699" alt="image" src="https://github.com/user-attachments/assets/3391cf3d-c1dd-4ff7-b6f8-f95fc06e4aff" />

<img width="1312" height="4253" alt="image" src="https://github.com/user-attachments/assets/ec7c9861-7dd0-458a-b00e-8f1f9795479a" />

## Task 2

<img width="1168" height="486" alt="image" src="https://github.com/user-attachments/assets/e1459b85-db10-4580-a24a-471a2326d3a0" />

<img width="1203" height="607" alt="image" src="https://github.com/user-attachments/assets/9ee0fd74-f90f-4b69-a09c-debe2d301034" />

## Task 3

<img width="1084" height="1120" alt="image" src="https://github.com/user-attachments/assets/b148f0d3-27c0-4d9f-a4e1-be072696bd85" />

curl -k -i -X GET "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available"
HTTP/1.1 401 Unauthorized
activityid: 622760a6-afd5-4f90-8b2e-f7eb10f3f9be
Access-Control-Allow-Origin: *
WWW-Authenticate: Internal API Key realm="WSO2 API Manager", Bearer realm="WSO2 API Manager", error="invalid_token", error_description="The provided token is invalid"
Content-Type: application/json; charset=UTF-8
Date: Sun, 26 Jul 2026 09:53:41 GMT
Transfer-Encoding: chunked
{"code":"900902","message":"Missing Credentials","description":"Invalid Credentials. Make sure your API invocation call has a header: 'Authorization : Bearer ACCESS_TOKEN' or 'Authorization : Basic ACCESS_TOKEN' or 'ApiKey : API_KEY'"}loayelnoamani@pop-os:~$

## Task 4
<img width="1281" height="150" alt="image" src="https://github.com/user-attachments/assets/fd190d2a-1067-4277-855f-43ef1f8ca3a1" />

<img width="1329" height="471" alt="image" src="https://github.com/user-attachments/assets/4c8947c2-5089-4d3a-9f05-14ed9fbb3ab7" />
<img width="1329" height="471" alt="image" src="https://github.com/user-attachments/assets/6528178b-7d6f-40d4-b628-710777324c5a" />
loayelnoamani@pop-os:~$ TOKEN=$(curl -k -s -X POST https://localhost:9443/oauth2/token -d "grant_type=client_credentials" -u "$CONSUMER_KEY:$CONSUMER_SECRET" | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)


loayelnoamani@pop-os:~$ curl -k -s -o /dev/null -w "%{http_code}\n" "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
200
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

## Task 5
<img width="1327" height="697" alt="image" src="https://github.com/user-attachments/assets/e7893125-cdd6-4441-a505-98c03c5bceb4" />
<img width="1327" height="697" alt="image" src="https://github.com/user-attachments/assets/d1fcc282-cdbb-4e8e-86a0-7a5b4be343b3" />
https://localhost:8243/petstore/1.0.0/pet/findByStatus
https://localhost:8243/petstore/2.0.0/pet/findByStatus
same output

loayelnoamani@pop-os:~$ TOKEN=$(curl -k -s -X POST https://localhost:9443/oauth2/token -d "grant_type=client_credentials" -u "$CONSUMER_KEY:$CONSUMER_SECRET" | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)
loayelnoamani@pop-os:~$ echo "--- v1.0.0 ---"
curl -k -s -o /dev/null -w "%{http_code}\n" "https://localhost:8243/petstore/1.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
--- v1.0.0 ---
200
loayelnoamani@pop-os:~$ echo "--- v2.0.0 ---"
curl -k -s -o /dev/null -w "%{http_code}\n" "https://localhost:8243/petstore/2.0.0/pet/findByStatus?status=available" -H "Authorization: Bearer $TOKEN"
--- v2.0.0 ---
200
loayelnoamani@pop-os:~$

## Task 6
<img width="1312" height="1533" alt="image" src="https://github.com/user-attachments/assets/20e7ff6a-488f-47ef-acff-2b9d212177eb" />

<img width="1327" height="697" alt="image" src="https://github.com/user-attachments/assets/076d2bec-27c9-41b8-a27a-6a8b42349fc9" />
