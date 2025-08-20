# 📗 08.20 AWS
## 📚 MySQL 이용해서 게시판 생성
### 1. VPC 생성
- VPC 생성 시 자동으로 생성하면 라우팅 테이블에서 서브넷 확인 필요

### 2. EC2 생성
- bastion, web01, web02 생성 후 보안 그룹에 인바운드 규칙 ssh 설정
- bastion은 추가로 보안 그룹에서 아웃바운드 MySQL 설정
- http, https는 로드 밸런스에서 설정

### 3. 로드밸런서 생성
- 로드밸런서 생성 후 보안 그룹에서 인바운드 규칙 HTTP, HTTPS 설정
- 대상 그룹 전달 시 포트 설정 확인 필요 (8080, 3000 등)

### 4. RDS 생성
- 파라미터 그룹 생성 시 사용할 DB 선택
- 서브넷 그룹 생성 시 서브넷 private 서버인지 확인
- 옵션 그룹 생성 시 사용할 DB 선택
- 데이터베이스 생성 후 보안 그룹에서 인바운드 규칙 MySQL/Aurora로 선택
- 보안 그룹에 default 설정 시 default들만 서로 소통하기 때문에 확인 필요

### 5. bastion, web 설정
- bastion에 MySQL 설치

```bash
sudo yum install mariadb105 -y
```

- MySQL 정상 작동 확인
```bash
mysqladmin ping -u admin -p -h database-1.crone748rvgl.ap-northeast-2.rds.amazonaws.com
```

- MySQL 시작
```bash
mysql -h database-1.crone748rvgl.ap-northeast-2.rds.amazonaws.com -P 3306 -u admin -p
```

- web01, web02에 vi를 이용하여 java, git 설치
```bash
#!/bin/bash
sudo yum install java-17-amazon-corretto-devel -y
sudo dnf install git -y
```

- web01, web02에 vi를 이용하여 git clone 자동화
```bash
#!/bin/bash
cd
sudo rm -rf swu_blog_deploy    # Permission denied 오류로 인해 sudo 사용
git clone https://github.com/dev-library/swu_blog_deploy.git
cp credential-db ./swu_blog_deploy/src/main/resources/application-db.properties
cd swu_blog_deploy
chmod +x ./gradlew
./gradlew clean build -x test
echo "build complete(ci)"
java -jar ./build/libs/blog-0.0.1-SNAPSHOT.war
echo "deploy complete(cd)"
```
## 📊 결과 확인
- 박세진 노션 : [PSJ REPOSITORY](https://psjrepository.notion.site/DAY-26-2553d86ddbdc80149675d4b760a0a0f1)