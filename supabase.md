
# Supabase?

- 오픈소스 기반 BaaS(Backend as a Service) 플랫폼
- postgres 개반 백엔드 개발 플랫폼
- 기능
    - Database (postgreSQL, vectorDB)
    - Auth
    - Storage
    - Realtime
    - Edge Functions
    - ...

## env

- supabase: organization > projects 구조
- 동일 organization 이하 dev/alpha/real project 생성
- 클라이언트 측에서는 각 project의 projectURL, key 값으로 접근
- project 이하로 Database, Auth, Storage, Realtime 등 독립적으로 설정

## Database
- postgreSQL RDBMS 기반
    - postgreSQL 기능 완전 지원
    - firebase의 경우 NoSQL(filestroe?) 사용
- 추가 기능
    - Realtime: insert, update, delete 이벤트 pub/sub 기능 제공
    - sql editor: web console 기반 SQL 작업 툴 제공
    - mysql, mariadb 등 db migration 기능 제공

CRUD example
```js
// create
const { data, error } = await supabase
  .from('posts')
  .insert([
    { title: 'title', content: 'content' }
  ])

// read
const { data, error } = await supabase
  .from('posts')
  .select('*')

// update
const { data, error } = await supabase
  .from('posts')
  .update({ content: 'updated content' })
  .eq('id', 1)

// delete
const { data, error } = await supabase
  .from('posts')
  .delete()
  .eq('id', 1)

// subscribe
const subscription = supabase
  .from('posts')
  .on('*', payload => {
    console.log('posts updated..', payload)
  })
  .subscribe()
```


```
RLS? (Row Level Security)
- db 수준에서 사용자의 권한에 따라 행 접근을 제어
- Grant, Invoke 와 같은 테이블 수준  접근제어가 아닌 행 단위 조회, 수정, 삭제에 대한 권한 설정
- `pg_policy` 통해 행별 접근 제거 규칙 생성

...
```
- https://supabase.com/docs/guides/database/postgres/row-level-security

## Auth

### 인증 지원
- email/password
- OAuth (google, apple, github..)

### JWT 기반 인증
- JWT + RLS (Row Level Security,  행단위 보안) 통한 인가 처리

### Auth Flow
1. (client) 로그인 요청
2. (supabase) 인증 및 JWT 발급
3. (client) JWT 세션 포함 요청
4. (supabase) JWT, RLS 기반 접근 제어

### 구성
- User
    - auth.users 테이블에 저장 (컬럼 커스텀 추가 불가)
    - UUID 기반 식별
    - email, phoneNumber 등 추가 정보 가질 수 있음
- Identity
    - auth.identities 테이블에 저장
    - User 의 로그인 수단 (계정 연동 등)
    - 한 명의 User 는 여러 Identity 를 갖는다

```
Identity Linking
- 하나의 supabase user(auth.users) 에 여러 인증수단(providers)을 연결하는 기능
- Automatinc Linking
  - Identity Linking default 전략
  - 동일 email 를 가진 auth.users 가 있는 경우 생성되는 identity 를 자동 연결
- Manual Linking (20250210 기준 Beta)
  - 서로 다른 email 의 identity 를 하나의 user 에 연결
```

```
google oauth example

0. google cloud console 앱 생성 및 기본 설정

1. Authentication > Sign In/Providers > Auth Providers
- google 활성화

2. Authentication > URL Configuration 
- site URL
- Redirect URLs 추가
```

## Storage
- 파일 저장, 관리 및 서빙 기능 제공
- S3 호환 인터페이스(api) 제공, SDK 방식도 제공
- postgres RLS 과 결합된 접근 제어 가능
- Vector Bucket 제공


## Realtime
- websocket 기반 실시간 메세지 발송
    - supabase 내부 websocket 서버 기반
- broascast
- presence
    - presence(접속 상태 동기화): channel 에 접속한 user 상태 실시간 동기화 (online 여부 등)

postgres changes
- db 변경 이벤트 감지하여 websocket 메세지 발송
- postgres Logical Replication 기반 (https://www.postgresql.org/docs/current/logical-replication.html)
- WAL(Write-Ahread Logging) 의 데이터를 디코딩하여 테이블 변경사항 추출 및 subscriber 에게 대상 추출 데이터 publish
- realtime 서버는 logical replication 에 의해 스트리밍 되는 WAL 의 디코딩 된 변경사항에 대한 데이터를 지속적으로 수신한다.

reference
- https://github.com/supabase/realtime


## Edge Function
- 기본적인 개념은 aws lambda 와 같은 서버리스 함수와 같음
- 서버리스 함수 실행 (Deno, Typescript 기반)
    - 함수는 각 edge location 에서 실행 (호출자와 가까운 곳)

## supabase mcp
- tools (DB)
    - list_tables - List all tables in the database
    - list_extensions - List available/installed Postgres extensions
    - list_migrations - List database migrations
    - apply_migration - Apply a database migration
    - execute_sql - Execute SQL queries

```
1. users 테이블에 sort delete 용 deleted_at 컬럼 추가
2. email 에 unique index 생성
```

- tools (Edge functions)
    - list_edge_functions (get_edge_function)
    - deoloy_edge_function

```
user profile 조회용 edge function 하나 만들어서 배포해.
응답은 xxx, xxx, xxx 등이 필요해
```

- tools (logs)
    - get_logs
    - get_advisors - Get security and performance advisors

```
//get_logs
최근 1시간 auth 관련 에러 로그 조회

//get_advisors
성능 이슈 있는 쿼리 정리해서 알려줘
```

reference
- https://supabase.com/docs/guides/getting-started/mcp

## Supabase CLI

로컬 개발환경 구성 (docker 기반)
- postgres, auth, realtime, storage 등을 docker 로 로컬 실행 및 supabase linking
```
supabase init
supabase start
supabase stop
supabase status
supabase link
```

Declarative database schemas
- “어떻게 변경할지(How)”가 아니라 “최종적으로 어떤 스키마 상태여야 하는지(What)”를 선언
- 선언된 스키마를 기준으로 CLI가 자동으로 migration(diff)을 생성
- 로컬 ↔ 운영 DB 간 스키마 일관성 유지

vs spring/jpa

| 구분 | Spring/JPA 세계 | Supabase Declarative |
|------|-----------------|---------------------|
| **스키마 정의 방식** | `@Entity` | `supabase/schemas/*.sql`에 스키마 정의 |
| **마이그레이션** | Flyway: `V1__create_table.sql`, `V2__add_column.sql` | `supabase db diff`로 자동 생성 |
| **로컬 DB** | Docker Compose 직접 설정 | `supabase start` 한 번에 구축 |
| **타입 안전성** | JPA Entity 클래스 | `supabase gen types`로 TS 타입 자동 생성 |
| **변경사항 푸시** | `flyway migrate` 또는 앱 배포 시 자동 실행 | `supabase db push --dry-run` 확인 후 `supabase db push` |

reference
- https://supabase.com/docs/reference/cli/introduction

## 여러 환경 클라이언트가 있는 경우

- FE 환경에서만 독립적으로 프로젝트를 진행하는 경우 비즈니스 로직을 FE 코드에 구현해도 무관
- Mobile 환경까지 있는 경우 비즈니스 로직이 클라이언트 측에 있는 경우 유지보수 및 기능 일원화 어려움

중앙 집중식 로직
- Database Function, Edge Function
- 주요 로직을 데이터베이스 함수로 작성
- 무거운 작업의 경우 Edge Function 으로 분리

```
Database Function, Edge Function..
- 백엔드 로직 구현 복잡도는 오히려 올라갈 듯. (framework 의 도움 X)
- 바이브 코딩에 적합하지 않을 듯. (바이브 코딩 개발 환경 구성에 어려움 예상, IDE 등 부재)
```


---


