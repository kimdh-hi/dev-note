# Trivy

## Trivy?
- 컨테이너 이미지, 파일시스템, git 저장소, vm, k8s 등을 대상으로 아래 사항으로부터 문제를 찾고 SBOM을 생성하는 오픈소스 보안 스캐너
  - 알려진 취약점 (CVE)
  - 잘못된 설정 (IaC)
  - 하드코딩 된 시크릿
  - 소프트웨어 라이선스 문제
- Apache-2.0 라이센스, GO 로 작성
  - Aqua Security 관리/개발
- SBOM(Software Bill OfMaterials, 소프트웨어 자산 명세서)
  - 소프트웨어 하나를 구성하는 모든 컴포넌트(OS패키지, 라이브러리, 의존성)을 버전, 라이센스, 식별자, 의존관계와 함께 기록한 문서
- Trivy 는 SBOM 파일을 생성하고 SBOM 파일을 대상으로 검사 진행
  - `trivy image --format spdx-json --output spdx.json alpine:3.16.0` # SBOM 생성
  - `trivy sbom spdx.json` # SBOM 검사

## Trivy target, scanner
- trivy 는 특정 target 에 n개 scanner 를 골라 적용하는 구조

### Trivy target

```
trivy image ...
trivy fs ...
trivy repo ...
trivy vm ...
trivy k8s ...
trivy sbom ...
```

### Trivy scanner (`--scanners`)
- 한 개 target 에 대해 n개 스캐너 지정 가능
  - `image` 는 vuln, misconfig, secret, license 허용 (기본값 vuln, secret)
  - `sbom` 은 vuln, license 허용 (기본값 vuln)

```
trivy image \
  --scanners vuln,secret,license,misconfig \
  --image-config-scanners misconfig,secret \
  --format json --output result.json \
  nginx:1.27
```

#### Vulnerability
- `trivy image [image-name] --scanners vuln --severity HIGH,CRITICAL .`
  - `--ignore-unfixed`: 패치가 없는 취약점은 무시
- 패키지와 의존성의 알려진 취약점(CVE) 탐지

#### Misconfiguration
- `trivy config Dockerfile`
- `trivy image --image-config-scanners misconfig nginx:1.27`
- Dockerfile, Terraform, CloudFormation... 등 IaC 형식을 감지하여 설정 오류 탐지

#### Secret
- `trivy image --scanners secret myapp:1.0` # 이미지 레이어 안 파일에서 시크릿만 검사
- `trivy image --image-config-scanners secret myapp:1.0` # 이미지 설정(환경 변수 등)에 들어 있는 시크릿 검사
- `trivy fs --scanners secret --secret-config ./trivy-secret.yaml .` # 사용자 정의 규칙 파일 지정
- 하드코딩 된 자격 증명 탐지
- 사용자 정의 규칙 파일 지정 가능 `--secret-config`

```
rules:
  - id: internal-api-key
    category: InternalAPI
    title: Internal API Key
    severity: HIGH
    regex: 'MYCO_[A-Z0-9]{32}'
    keywords:
      - MYCO_
allow-rules:
  - id: skip-test-fixtures
    description: 테스트 픽스처 제외
    path: '.*/testdata/.*'
```

- 사용자 정의 규칙 파일 지정하지 않아도 기본 규칙 적용
  - https://github.com/aquasecurity/trivy/blob/main/pkg/fanal/secret/builtin-rules.go

#### license
- google 라이센스 분류 체계에 따라 라이센스별 위험도를 분석
  - https://opensource.google/documentation/reference/thirdparty/licenses
- google 이 라이센스별로 등급을 정해둔 것을 trivy 는 CRITICAL/HIGH/LOW 로 매핑하여 규칙으로 사용 (커스텀 가능)


## Reference
- https://trivy.dev/docs/latest/guide/
- https://github.com/aquasecurity/trivy