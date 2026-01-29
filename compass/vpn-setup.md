# VPN 설정 가이드

외부에서 Compass에 접속하려면 VPN 연결이 필요합니다. 한 번만 설정하면 이후에는 연결 버튼만 누르면 됩니다. 불편하시더라도 양해 부탁드립니다.

> **주의사항**
> 월별 VPN 사용량이 한정되어 있습니다. **Compass 접속 외 용도로는 사용을 제한**해 주세요.
> 사용 완료 후에는 반드시 **연결 해제** 버튼을 눌러 VPN 연결을 종료해 주세요.

---

## 1. Outline 클라이언트 설치

| 운영체제 | 다운로드 링크 |
|---------|-------------|
| Windows | [Outline-Client.exe](https://s3.amazonaws.com/outline-releases/client/windows/stable/Outline-Client.exe) |
| macOS | [App Store](https://apps.apple.com/app/outline-app/id1356178125) |

---

## 2. 액세스 키 입력

**Step 1.** Outline을 실행하면 **VPN 액세스 키 추가** 화면이 나타납니다.

<img src="images/vpn-step-1.png" width="300" alt="Step 1 - 액세스 키 입력 화면">

- 전달받은 **액세스 키**를 입력창에 붙여넣고 **확인**을 클릭합니다.

> 액세스 키는 `ss://...` 형식의 문자열입니다.

**Step 2.** 서버가 추가되면 **연결** 버튼을 클릭합니다.

<img src="images/vpn-step-2.png" width="300" alt="Step 2 - 서버 추가 완료">

**Step 3.** 화면이 **녹색**으로 바뀌면 연결 완료입니다.

<img src="images/vpn-step-3.png" width="300" alt="Step 3 - VPN 연결 완료">

---

## 문제 해결

| 증상 | 해결 방법 |
|-----|----------|
| 연결 버튼을 눌러도 연결되지 않음 | V3, 알약 등 백신 프로그램을 잠시 종료 후 재시도 |
| 액세스 키가 유효하지 않다고 표시됨 | 관리자에게 새 액세스 키 요청 |
| Compass 페이지가 열리지 않음 | VPN 연결 상태 확인 (녹색 표시 여부) |
