# Compass 기술 문서

Compass 서비스 개발에 필요한 기술 문서 모음.

---

## 문서 목록

### 데이터베이스

| 문서 | 설명 | 대상 |
|------|------|------|
| [database-design.md](./compass/database-design.md) | 데이터베이스 설계 배경과 원칙 | 전체 개발자 |
| [database-schema.md](./compass/database-schema.md) | 테이블 스키마 및 구현 세부사항 | 백엔드 개발자 |

---

## 프로젝트 구조

```
docs/
├── README.md              # 이 파일 (인덱스)
├── compass/               # Compass 메인 서비스
│   ├── database-design.md
│   └── database-schema.md
└── compass-engine/        # 데이터 추출/정제 서비스
    └── (예정)
```

---

## 문서 작성 규칙

### 파일명
- 소문자, 하이픈(-) 구분 사용
- 문서 성격이 드러나는 명확한 이름 사용
- 예: `database-design.md`, `api-reference.md`

### 문체
- 개조형 문체 사용 (~임, ~함, ~필요)
- 간결하고 명확한 표현 지향
- 중복 설명 지양, 필요시 링크로 참조

### 구조
- 문서 상단에 목적과 대상 독자 명시
- 관련 문서 링크 포함
- 작성일/버전 정보 문서 하단에 기재

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [compass](https://github.com/fntimes/compass) | 메인 서비스 (대시보드, 분석 기능) |
| [compass-engine](https://github.com/fntimes/compass-engine) | 데이터 추출/정제 서비스 |

---

**최종 수정일**: 2025-11-27
