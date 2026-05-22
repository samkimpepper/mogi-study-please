# DNS 확인

"DNS가 안 된다"는 것은 서버가 도메인 이름을 IP 주소로 바꾸지 못한다는 뜻이다.

예를 들어 애플리케이션은 보통 외부 API나 DB를 도메인으로 호출한다.

```text
api.example.com
db.example.internal
redis.example.internal
```

실제로 네트워크 통신을 하려면 먼저 이 이름을 IP 주소로 바꿔야 한다.

```text
api.example.com -> 203.0.113.10
```

DNS가 안 되면 애플리케이션 로그에는 단순히 timeout이나 host not found처럼 보일 수 있다.

확인할 때는 보통 이런 명령어를 쓴다.

```bash
nslookup api.example.com
dig api.example.com
curl -v https://api.example.com
```

백엔드 장애에서 DNS 문제가 중요한 이유는, 애플리케이션 코드가 멀쩡해도 이름 해석이 안 되면 외부 서비스에 접근할 수 없기 때문이다.

## 확인 흐름

```text
외부 API 호출 실패
-> 애플리케이션 로그에서 host not found, timeout 확인
-> nslookup 또는 dig로 DNS 해석 확인
-> curl -v로 실제 연결 시도 확인
-> DNS 문제인지, 네트워크 문제인지, 상대 서버 문제인지 좁힘
```
