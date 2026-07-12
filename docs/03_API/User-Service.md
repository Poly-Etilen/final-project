# User Service API

## 내 정보 조회

GET /api/users/me

---

### Response

```json
{
    "userId":1,
    "nickname":"eco",
    "email":"test@test.com"
}
```

---

## 회원정보 수정

PATCH /api/users/me

```json
{
    "nickname":"newNick"
}
```

---

## Workspace 목록

GET /api/users/me/workspaces

```json
[
    {
        "workspaceId":1,
        "name":"우리집 몬스테라"
    }
]
```