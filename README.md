# codeit-project

## 기술 스택

### 프론트엔드
- React
- Next.js
- SCSS
- TanStack Query
- Zustand
- Vercel

### 백엔드
- Express
- PostgreSQL
- Swagger

### 인프라
- Docker
- Render

## 폴더 구조 (임시)

아래는 프론트엔드 기능별 구조와 백엔드 Layered Architecture를 적용한 제안 구조입니다.

```text
.
├── frontend/                    # 프론트엔드
│   ├── app/                     # Next.js 라우팅 및 페이지
│   ├── features/                # 기능별 모듈
│   │   ├── auth/
│   │   │   ├── api/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── stores/
│   │   │   └── types/
│   │   └── [feature]/
│   └── shared/                  # 공통 컴포넌트, 유틸리티, 스타일
│
├── backend/                     # 백엔드
│   ├── src/
│   │   ├── routes/              # 라우팅 계층
│   │   ├── controllers/         # 요청/응답 처리 계층
│   │   ├── services/            # 비즈니스 로직 계층
│   │   ├── repositories/        # 데이터 접근 계층
│   │   ├── middlewares/
│   │   ├── config/
│   │   ├── app.js
│   │   └── server.js
│   └── prisma/
│       └── schema.prisma
│
├── docs/                        # 프로젝트 및 API 문서
└── mock/                        # 개발용 목 데이터
```
