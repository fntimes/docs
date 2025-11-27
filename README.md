# Compass 기술 문서

Compass 서비스 개발에 필요한 기술 문서 모음.

---

## 문서 목록

### 데이터베이스

| 문서 | 설명 |
|------|------|
| [database-design.md](./compass/database-design.md) | 데이터베이스 설계 배경과 원칙 |
| [database-schema.md](./compass/database-schema.md) | 테이블 스키마 및 구현 세부사항 |

---

## 프로젝트 구조

```
docs/
├── README.md              # 이 파일 (인덱스)
├── compass/               # Compass 메인 서비스
│   ├── database-design.md
│   └── database-schema.md
└── compass-engine/        # 데이터 추출/정제/모니터링 서비스
    └── (예정)
```

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [compass](https://github.com/fntimes/compass) | 메인 서비스 (대시보드, 분석 기능) |
| [compass-engine](https://github.com/fntimes/compass-engine) | 데이터 추출, 정제, 모니터링 |

---

**최종 수정일**: 2025-11-27
