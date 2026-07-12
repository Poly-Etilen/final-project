# API Convention

## URL

복수형 사용

```
/users

/workspaces

/devices
```

---

## HTTP Method

GET

조회

POST

생성

PATCH

수정

DELETE

삭제

---

## Response

```json
{
    "success":true,
    "data":{},
    "message":"success"
}
```

---

## Error

```json
{
    "success":false,
    "code":"WORKSPACE_NOT_FOUND",
    "message":"Workspace를 찾을 수 없습니다."
}
```

---

## Status Code

200 OK

201 CREATED

204 NO CONTENT

400 BAD REQUEST

401 UNAUTHORIZED

403 FORBIDDEN

404 NOT FOUND

409 CONFLICT

500 INTERNAL SERVER ERROR