# 기술 문서

프로젝트별 기술 문서 모음.

---

## Compass

Compass 시스템 기술 문서.

### 문서 목록

| 문서 | 설명 |
|------|------|
| [development-roadmap.md](./compass/development-roadmap.md) | 개발 로드맵 |
| [database-design.md](./compass/database-design.md) | 설계 철학, 핵심 과제와 접근 방식 |
| [database-schema.md](./compass/database-schema.md) | 테이블 구조, 컬럼 명세, 쿼리 예시 |
| [data-extraction-workflow.md](./compass/data-extraction-workflow.md) | 데이터 추출 서비스 구조 |

---

## FnBook

FnBook 시스템 문서.

### 문서 목록

| 문서 | 설명 |
|------|------|
| [service-overview.md](./fnbook/service-overview.md) | 서비스 개요 (메뉴 구성, 권한 체계) |
| [user-guide.md](./fnbook/user-guide.md) | 사용자 가이드 (기능 안내 및 사용법) |

---

## 문서 구조

```
docs/
├── README.md              # 이 파일 (인덱스)
├── compass/               # Compass 서비스
│   ├── development-roadmap.md
│   ├── development-roadmap.xlsx
│   ├── database-design.md
│   ├── database-schema.md
│   ├── data-extraction-workflow.md
│   └── images/
│       └── development-roadmap.png
└── fnbook/                # FnBook 서비스
    ├── service-overview.md
    └── user-guide.md
```

---

**최종 수정일**: 2026-01-16
