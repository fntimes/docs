# Compass VPN 접속 가이드

Compass는 사내망 또는 VPN을 통해서만 접속 가능합니다. 외부에서 접속하려면 아래 가이드에 따라 VPN을 설정해 주세요.

> **주의사항**
> 월별 VPN 사용량이 한정되어 있습니다. **Compass 접속 외 용도로는 사용을 제한**해 주세요.
> 사용 완료 후에는 반드시 **연결 해제** 버튼을 눌러 VPN 연결을 종료해 주세요.

---

## 1. Outline VPN 클라이언트 설치

아래 링크에서 본인의 운영체제에 맞는 클라이언트를 다운로드하여 설치합니다.

| 운영체제 | 다운로드 링크 |
|---------|-------------|
| Windows | [Outline-Client.exe](https://s3.amazonaws.com/outline-releases/client/windows/stable/Outline-Client.exe) |
| macOS | [App Store](https://apps.apple.com/app/outline-app/id1356178125) |

---

## 2. 액세스 키 입력

### Step 1. Outline 실행

Outline 클라이언트를 실행하면 아래와 같이 **VPN 액세스 키 추가** 화면이 나타납니다.

<img src="images/vpn-step-1.png" width="300" alt="Step 1 - 액세스 키 입력 화면">

- 관리자에게 전달받은 **액세스 키**를 입력창에 붙여넣습니다.
- **확인** 버튼을 클릭합니다.

> 액세스 키는 `ss://...` 형식의 문자열입니다.

---

### Step 2. 서버 추가 완료

액세스 키가 정상적으로 입력되면 아래와 같이 **Outline 서버**가 추가됩니다.

<img src="images/vpn-step-2.png" width="300" alt="Step 2 - 서버 추가 완료">

- 화면에 "연결 해제됨" 상태로 표시됩니다.
- 우측 하단의 **연결** 버튼을 클릭합니다.

---

### Step 3. VPN 연결 완료

연결이 성공하면 아래와 같이 화면이 **녹색**으로 변경되고, "연결됨" 상태가 표시됩니다.

<img src="images/vpn-step-3.png" width="300" alt="Step 3 - VPN 연결 완료">

- 하단에 **'Outline 서버'에 연결됨** 메시지가 표시됩니다.
- 연결이 완료되면 외부에서 Compass에 접속할 수 있습니다.

---

## 문제 해결

| 증상 | 해결 방법 |
|-----|----------|
| 연결 버튼을 눌러도 연결되지 않음 | V3, 알약 등 백신 프로그램을 잠시 종료 후 재시도 |
| 액세스 키가 유효하지 않다고 표시됨 | 관리자에게 새 액세스 키 요청 |
| Compass 페이지가 열리지 않음 | VPN 연결 상태 확인 (녹색 표시 여부) |

---

## 문의

VPN 접속 관련 문의는 관리자에게 연락해 주세요.
