# 05-1. REST API 준비

## 애플리케이션 비밀번호 발급

1. 관리 화면에서 사용자 > 프로필로 들어갑니다.
2. 페이지 아래쪽의 애플리케이션 비밀번호 항목을 찾습니다.
3. 이름을 적습니다. 예: `autoblog-api`
4. 추가를 누르면 `abcd efgh ijkl mnop` 형태의 값이 한 번 표시됩니다.
5. 이 값을 복사해 로컬 파일에 저장합니다. 창을 닫으면 다시 볼 수 없습니다.

이 비밀번호는 사이트 로그인 비밀번호와 다릅니다. 권한은 계정과 같지만, 필요 없어지면 그 항목만 삭제해 즉시 무효화할 수 있습니다. 자동화에는 로그인 비밀번호 대신 이 값을 씁니다.

## 값 보관

```text
~/.config/autoblog/wordpress.txt
user: <관리자 아이디>
app_password: <발급받은 값>
```

파일 권한은 소유자만 읽을 수 있게 둡니다.

```bash
chmod 600 ~/.config/autoblog/wordpress.txt
```

이 파일은 저장소에 올리지 않습니다. 스크립트에는 파일 경로만 적고, 값은 실행할 때 읽어 옵니다.

## 인증 방식

워드프레스 REST API는 HTTP 기본 인증을 씁니다. 사용자 이름과 애플리케이션 비밀번호를 콜론으로 이어 붙이고 base64로 인코딩해 헤더에 넣습니다.

```python
import base64, requests

def session_for(user, app_password, base):
    token = base64.b64encode(f"{user}:{app_password}".encode()).decode()
    s = requests.Session()
    s.headers.update({"Authorization": f"Basic {token}",
                      "User-Agent": "autoblog/1.0"})
    s.base = base.rstrip("/")
    return s

s = session_for("admin", "abcd efgh ijkl mnop", "https://내도메인")
r = s.get(s.base + "/wp-json/wp/v2/users/me")
print(r.status_code, r.json().get("name"))
```

200과 함께 표시 이름이 나오면 인증이 끝난 것입니다.

## 자주 막히는 지점

| 증상 | 원인 | 확인 |
| --- | --- | --- |
| 401 | 비밀번호 오타, 공백 포함 | 값 전체를 그대로 넣었는지 |
| 403 | 보안 플러그인이 API 차단 | 플러그인 설정에서 REST 허용 |
| 404 | 고유주소 설정이 기본값 | 설정 > 고유주소를 글 이름으로 |
| 200인데 HTML | 서버 오류 페이지를 200으로 반환 | 응답 본문이 JSON인지 확인 |

마지막 항목이 가장 까다롭습니다. 호스팅이 트래픽 초과 상태일 때 오류 페이지를 200으로 돌려주는 경우가 있습니다. 상태 코드만 보면 성공으로 오해합니다. 그래서 발행 스크립트는 상태 코드와 함께 응답 본문이 JSON인지 확인합니다.

```python
import json

def is_json(resp):
    ctype = resp.headers.get("Content-Type", "")
    if "application/json" not in ctype:
        return False
    try:
        json.loads(resp.text)
        return True
    except Exception:
        return False
```
