# Terraform AWS S3 작업

## 🚪 들어가기 전
현대 소프트웨어 개발 및 배포 환경에서 인프라 관리의 중요성을 느끼고 있습니다. 특히 클라우드 서비스가 보편화되면서, 인프라를 코드로 정의하고 자동화하는 도구가 필요하다는 생각이 들었습니다. 여러 가지 도구를 조사해본 결과, Terraform이 이러한 요구를 충족시켜 줄 수 있는 강력한 도구라는 것을 알게 되어 이를 위해 다음과 같은 단계로 작업을 진행하였습니다.

<br/>

## 💻 시스템 환경 및 소프트웨어

| **항목**            | **상세 정보**                                                                                  |
|---------------------|-----------------------------------------------------------------------------------------------|
| **운영 체제**       | ![](https://img.shields.io/badge/Ubuntu%2022.04.5%20LTS-E95420?style=flat-square&logo=Ubuntu&logoColor=white)  |
| **CLI 도구**       | ![](https://img.shields.io/badge/AWS_CLI_v2.18.0-232F3E?style=flat-square&logo=amazonaws&logoColor=white)  <br> ![](https://img.shields.io/badge/Terraform_v1.9.7-7B42BC?style=flat-square&logo=terraform&logoColor=white)  |


<br/>

## 🧩 작업 전1:Terraform Install
```bash
sudo su - # root 계정 접속

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg


echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

apt-get update && apt-get install terraform -y
```

<br/>

## 🧩 작업 전2:Terraform 주요 명령어
```bash
terraform init # Terraform 프로젝트를 초기화

terraform plan # 현재 상태와 정의된 인프라를 비교하여 실행할 변경 사항을 미리 보여줌

terraform apply # 변경 사항을 실제로 적용하여 인프라를 업데이트

terraform apply -auto-approve # 사용자로부터 적용 확인을 받지 않고 자동으로 승인하여 바로 인프라 변경을 실행

terraform destroy # 모든 인프라 리소스 삭제

terraform validate # Terraform 코드가 문법적으로 유효한지 확인

terraform output # output 블록에서 정의된 값 터미널에 출력

terraform apply -target={리소스 유형}.{리소스 이름} # 파일에 정의된 해당 리소스만 적용하여 인프라를 업데이트
```

<br/>

## 🧩 작업 전3:S3 버킷 사용 권한 부여 설정
```tf
# IAM 역할 생성
resource "aws_iam_role" "s3_create_bucket_role" {
  name = "s3-create-bucket-role"
  
  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# IAM 정책 정의 (S3에 대한 모든 권한 부여)
resource "aws_iam_policy" "s3_full_access_policy" {
  name        = "s3-full-access-policy"
  description = "Full access to S3 resources"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:*"  # 모든 S3 액세스 허용
        ]
        Resource = [
          "*"  # 모든 S3 리소스에 대한 권한
        ]
      }
    ]
  })
}

# IAM 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "attach_s3_policy" {
  role       = aws_iam_role.s3_create_bucket_role.name
  policy_arn = aws_iam_policy.s3_full_access_policy.arn
}
```

<br/>

## 🧩 작업 전4:파일 구조
```bash
username@awsclient:~/s3bucket$ tree
.
├── index.html
├── main.html
├── provider.txt
├── resource.tf
├── updateIndex.tf
└── updateMain.tf
```

<br/>

## 1️⃣ 작업 1:S3 버킷 생성

### 1. resource.tf 생성
```tf
# S3 버킷 생성
resource "aws_s3_bucket" "bucket1" {
  bucket = "{S3 버킷 이름}"  # 생성하고자 하는 S3 버킷 이름
  force_destroy =  true 
}

# S3 버킷의 public access block 설정
resource "aws_s3_bucket_public_access_block" "bucket1_public_access_block" {
  bucket = aws_s3_bucket.bucket1.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}


# S3 버킷의 웹사이트 호스팅 설정
resource "aws_s3_bucket_website_configuration" "xweb_bucket_website" {
  bucket = aws_s3_bucket.bucket1.id  # 생성된 S3 버킷 이름 사용

  index_document {
    suffix = "index.html"
  }
}

# S3 버킷의 public read 정책 설정
resource "aws_s3_bucket_policy" "public_read_access" {
  bucket = aws_s3_bucket.bucket1.id  # 생성된 S3 버킷 이름 사용

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [ "s3:GetObject" ],
      "Resource": [
        "arn:aws:s3:::{S3 버킷 이름}",
        "arn:aws:s3:::{S3 버킷 이름}/*"
      ]
    }
  ]
}
EOF
}

```

<br>

## 2️⃣ 작업 2:새로운 index.html 업로드

### 1. index.html 생성
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    ce06
</body>
</html>
```

### 2. updateindex.tf 생성
```tf
resource "aws_s3_bucket_object" "main_html" {
  bucket       = "{S3 버킷 이름}" 
  key          = "main.html"     
  source       = "main.html"      
  content_type = "text/html"
}

output "s3_object_url_main" {
  value = "https://${aws_s3_bucket_object.main_html.bucket}.s3.amazonaws.com/${aws_s3_bucket_object.main_html.key}"
}
```
### 3. Terraform 리소스를 성공적으로 배포한 후, output 블록에서 생성된 URL에 직접 접속하여 배포된 결과를 확인
<img src="https://github.com/user-attachments/assets/2a38193e-e8b9-4ff1-9178-6c6f4ba50989" width="70%">


<br><br/>

## 3️⃣ 작업 3:이미 존재하는 index.html 수정 후 업로드 및 main.html 추가
### 1. index.html 수정
```html
# 수정 된 html파일은 git에서 다운로드 후 사용해주시면 됩니다.
```
### 2. updateIndex.tf 수정
```tf

resource "aws_s3_bucket_object" "main_html" {
  bucket       = "{S3 버킷 이름}" 
  key          = "main.html"     
  source       = "main.html"      
  content_type = "text/html"

  etag = filemd5("main.html")  # 해당 코드 추가 =>  파일의 MD5 해시를 사용하여 변경 여부를 감지
}

output "s3_object_url_main" {
  value = "https://${aws_s3_bucket_object.main_html.bucket}.s3.amazonaws.com/${aws_s3_bucket_object.main_html.key}"
}
```

### 3. main.html 추가
```html
#  html파일은 git에서 다운로드 후 사용해주시면 됩니다.
```

### 4. updateMain.tf 수정
```tf

resource "aws_s3_bucket_object" "main_html" {
  bucket       = "{S3 버킷 이름}" 
  key          = "main.html"      
  source       = "main.html"     
  content_type = "text/html"

  etag = filemd5("main.html") 
}

output "s3_object_url_main" {
  value = "https://${aws_s3_bucket_object.main_html.bucket}.s3.amazonaws.com/${aws_s3_bucket_object.main_html.key}"
}
```

### 5. Terraform 리소스를 성공적으로 배포한 후, output 블록에서 생성된 URL에 직접 접속하여 배포된 결과를 확인
1. index.html 접속 후 버튼을 클릭
<img src="https://github.com/user-attachments/assets/dbdc6535-7119-4ce1-9cbd-01a18323229f" width="70%">

<br>

2. main.html 접속 됨
<img src="https://github.com/user-attachments/assets/9d64ff6c-394d-4adb-87a4-8016526d3ce4" width="70%">

<br/>

## 🎈 느낀점
Terraform을 사용하여 S3 버킷에 파일을 간편하게 업로드하고 업데이트할 수 있었음. 코드로 정의된 인프라를 통해 설정과 배포를 쉽게 처리할 수 있었고, 개발 과정에서 발생할 수 있는 오류를 줄이는 데 큰 도움이 될 거라 생각이 들었음. Terraform의 변경 감지 및 업데이트 기능 덕분에, S3 버킷의 파일이 변경되었을 때만 업데이트가 이루어지는 것을 보며 자원 관리를 효율적으로 할 수 있다는 점이 매우 유익하게 느껴졌음.
