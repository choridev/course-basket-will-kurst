# 수구니는 터질 KU야

## 1. 프로젝트 개요

이 프로젝트는 대학교에서 제공하는 수강바구니 신청 인원 데이터를 보기 쉽게 가공하여 자신이 신청할 강의의 경쟁률을 파악할 수 있는 웹 사이트입니다.

이 서비스는 AWS 기반의 서버리스 아키텍처로 구축되었으며 정적 웹 호스팅 서비스를 사용합니다. 전체 인프라는 Terraform을 사용하여 관리되며 배포는 GitHub Actions를 사용하는 CI/CD 파이프라인으로 자동화됩니다.

## 2. 아키텍처

이 프로젝트는 현대적인 서버리스 및 IaC 중심 아키텍처를 따릅니다:

-   **프론트엔드**: HTML, CSS, 바닐라 JavaScript로 구축된 정적 페이지입니다.
-   **호스팅**: 정적 사이트는 정적 웹사이트 호스팅을 위해 구성된 AWS S3 버킷에 호스팅됩니다.
-   **CDN**: AWS CloudFront는 CDN으로 사용되어 엣지 로케이션에서 사이트를 캐시하여 전 세계 사용자에게 낮은 지연 시간으로 액세스를 제공합니다.
-   **DNS & SSL**: AWS Route 53은 사용자 지정 도메인을 관리하고, AWS Certificate Manager (ACM)는 HTTPS를 위한 SSL/TLS 인증서를 제공합니다.
-   **백엔드 상태 및 데이터**:
    -   Terraform 상태를 저장하기 위해 `backend-s3` Terraform 구성에 의해 별도의 S3 버킷과 DynamoDB 테이블이 관리되어 일관되고 잠금 가능한 상태 백엔드를 보장합니다.
    -   과목 데이터는 메인 S3 버킷 내의 JSON 파일에 저장되며 클라이언트 측 JavaScript에 의해 가져와집니다.
-   **IaC & CI/CD**:
    -   **Terraform**: 모든 AWS 리소스는 `.tf` 파일에 코드로 정의됩니다.
    -   **GitHub Actions**: `.github/workflows/terraform.yml`에 정의된 워크플로우는 `main` 브랜치에 푸시될 때 `terraform plan` 및 `terraform apply`를 자동으로 실행하여 지속적인 배포를 가능하게 합니다.

## 3. 디렉토리 구조

저장소는 다음과 같이 기능 단위로 구성됩니다:

```
/
├── .github/workflows/       # 자동화된 Terraform 배포를 위한 CI/CD 파이프라인
│   └── terraform.yml
├── app/                     # 주요 애플리케이션 인프라 및 소스 코드 포함
│   ├── data/                # JSON 형식으로 전처리된 과목 데이터
│   ├── infra/               # 프론트엔드 인프라(S3, CloudFront 등)를 위한 Terraform 코드
│   ├── media/               # 이미지와 같은 정적 리소스
│   └── source-code/         # 프론트엔드 HTML, CSS, JavaScript 파일
├── backend-s3/              # 백엔드 인프라(Terraform 상태 버킷, DynamoDB)를 위한 Terraform 코드
├── util/                    # 데이터 처리를 위한 스크립트 및 원본 데이터
│   ├── *.xlsx               # Excel 형식의 원본 과목 데이터
│   └── data_preprocessor.py # Excel 파일을 JSON으로 변환하는 Python 스크립트
└── README.md
```

## 4. 주요 구성 요소

### 프론트엔드 (`app/source-code/`)

-   **HTML**: `index.html` (메인 진입점), `home.html`, `result.html`.
-   **CSS**: 다른 페이지를 위한 스타일시트 (`index.css`, `home.css`, `result.css`).
-   **JavaScript**:
    -   `lecture.js`: 강의 데이터를 가져오고 표시하는 역할을 할 것으로 예상됩니다.
    -   `calculator.js`: 학점 또는 기타 지표를 계산하기 위한 클라이언트 측 로직을 구현합니다.

### Infrastructure as Code (`app/infra/` 및 `backend-s3/`)

-   **`app/infra/`**: 사용자에게 노출되는 인프라를 관리합니다.
    -   `s3.tf`: 웹 호스팅을 위한 S3 버킷을 정의합니다.
    -   `cloudfront.tf`: CloudFront 배포를 정의합니다.
    -   `route53.tf` & `acm.tf`: DNS 및 SSL 인증서를 관리합니다.
    -   `objects.tf`: 프론트엔드 소스 코드 및 데이터 파일을 S3 버킷에 업로드합니다.
-   **`backend-s3/`**: Terraform을 안전하게 실행하는 데 필요한 인프라를 관리합니다.
    -   `s3.tf`: Terraform의 원격 상태(`.tfstate`)를 저장할 S3 버킷을 정의합니다.
    -   `danamodb_table.tf`: 상태 잠금을 위한 DynamoDB 테이블을 정의하여 동시 상태 수정 방지합니다.

### 데이터 파이프라인 (`util/`)

-   프로세스는 Excel 파일(`24-1.xlsx`, `24-2.xlsx`)에 저장된 원본 과목 데이터로 시작합니다.
-   `data_preprocessor.py` 스크립트는 이 파일을 읽고 데이터를 처리하여 구조화된 JSON 형식(`24-1.json`, `24-2.json`)으로 내보냅니다.
-   이 JSON 데이터는 Terraform (`objects.tf`)에 의해 S3 버킷에 업로드되어 프론트엔드 애플리케이션에서 사용됩니다.

## 5. 배포 방법

1.  **AWS 자격 증명 구성**: GitHub 저장소에 AWS 액세스 키를 시크릿으로 설정합니다 (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).
2.  **Terraform 변수 업데이트**: `app/infra/terraform.tfvars` 파일을 사용자 지정 도메인 이름 및 기타 변수로 수정합니다.
3.  **GitHub에 푸시**: `main` 브랜치에 변경 사항을 푸시하면 GitHub Actions 워크플로우가 트리거됩니다.
4.  **워크플로우 실행**: 워크플로우는 Terraform을 자동으로 초기화하고 계획을 생성하며 AWS 계정에 변경 사항을 적용하여 인프라를 배포하거나 업데이트합니다.

이 자동화된 설정은 애플리케이션이 항상 저장소의 코드와 동기화되도록 보장합니다.
