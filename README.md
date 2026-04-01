# Docker 기반 개발 환경 구축 실습

## 1. 프로젝트 개요
본 실습에서는 Docker를 활용하여 컨테이너 기반 개발 환경을 구축하였습니다.  
Docker 설치 및 실행 확인, Ubuntu 컨테이너 실행, Dockerfile을 이용한 이미지 생성,  
웹 서버 컨테이너 실행 및 포트 매핑, Bind Mount 실습, Docker Volume 영속성 실습을 진행하였습니다.  
추가로 Docker Compose, 환경 변수 활용, GitHub SSH 키 설정 보너스 과제도 수행하였습니다.  
실습 과정 전체를 GitHub Repository와 README 문서를 통해 정리하였습니다.

Docker를 이용하면 프로그램 실행 환경을 컨테이너로 관리할 수 있으며  
개발 환경을 동일하게 유지할 수 있다는 장점이 있습니다.  
이번 실습을 통해 Docker의 기본 구조와 동작 방식을 이해하는 것을 목표로 하였습니다.

---

## 2. 실행 환경

| 항목 | 내용 |
|------|------|
| OS | Windows 11 |
| Shell / Terminal | PowerShell 7.6.0 |
| Docker | Docker Desktop |
| Git | 2.x |
| Web Server | Nginx (alpine) |
| Language | HTML |

---

## 3. 수행 항목 체크리스트

**필수 항목**
- [x] 터미널 기본 조작 및 디렉토리 구성
- [x] 파일 / 디렉토리 권한 변경 실습
- [x] Docker 설치 및 점검 (`docker --version`, `docker info`)
- [x] hello-world 컨테이너 실행
- [x] Ubuntu 컨테이너 실행 및 내부 진입
- [x] Dockerfile 작성 및 커스텀 이미지 빌드
- [x] 웹 컨테이너 실행 및 포트 매핑 접속 확인
- [x] Bind Mount 실습 (호스트 변경 전/후 반영 확인)
- [x] Docker Volume 영속성 실습 (컨테이너 삭제 전/후 데이터 유지 확인)
- [x] 컨테이너 및 이미지 목록 확인 (`docker ps -a`, `docker images`)
- [x] Docker 운영 명령 수행 (`docker logs`, `docker stats`)
- [x] Git 설정 및 GitHub 연동

**보너스 항목**
- [x] Docker Compose 기초 (단일 서비스 실행)
- [x] Docker Compose 멀티 컨테이너 (nginx + ubuntu 2개 서비스)
- [x] Compose 운영 명령어 (`up`, `down`, `ps`, `logs`)
- [x] 환경 변수 활용 (`.env` 파일로 포트 변경)
- [x] GitHub SSH 키 설정 및 인증 확인

---

## 4. 터미널 기본 조작 로그

PowerShell에서 기본 디렉토리 탐색 및 파일 조작 명령어를 수행하였습니다.

```powershell
# 현재 위치 확인
PS C:\Users\User> pwd

# 목록 확인 (숨김 파일 포함)
PS C:\Users\User> ls -Force

# 작업 디렉토리 생성 및 이동
PS C:\Users\User> mkdir docker-web
PS C:\Users\User> cd docker-web

# 빈 파일 생성
PS C:\Users\User\docker-web> New-Item test.txt

# 파일 내용 확인
PS C:\Users\User\docker-web> cat Dockerfile

# 파일 복사, 이름 변경, 삭제
PS C:\Users\User\docker-web> cp test.txt test-copy.txt
PS C:\Users\User\docker-web> mv test-copy.txt renamed.txt
PS C:\Users\User\docker-web> rm renamed.txt
```

### 프로젝트 디렉토리 구조 및 구성 기준

```
docker-web/
├── app/
│   └── index.html       # 웹 서버가 서빙할 정적 파일 (콘텐츠 영역)
├── images/              # README에 삽입할 스크린샷 모음
├── Dockerfile           # 커스텀 이미지 빌드 설계도
├── docker-compose.yml   # Compose 실행 설정 파일
├── .env                 # 환경 변수 정의 (포트 등 설정값 분리)
└── README.md            # 실습 문서
```

**구성 기준:**

- `app/` : 컨테이너에 마운트되거나 COPY될 **서비스 소스 파일**만 모아두었습니다. Dockerfile의 `COPY app/ /usr/share/nginx/html/` 명령과 Bind Mount의 `-v ${PWD}/app:/usr/share/nginx/html` 경로가 이 폴더를 직접 참조하므로, 경로 일관성을 위해 별도 분리하였습니다.
- `Dockerfile` / `docker-compose.yml` / `.env` : 실행 환경 정의 파일은 프로젝트 루트에 위치시켜 `docker build .`, `docker compose up` 명령을 루트에서 바로 실행할 수 있도록 구성하였습니다.
- `images/` : README 문서에 삽입하는 스크린샷을 별도 폴더로 분리하여 소스 파일과 문서 자산을 구분하였습니다.
- `.env` : 포트 번호처럼 환경마다 달라질 수 있는 값은 코드에 직접 쓰지 않고 `.env`로 분리하였습니다. 포트를 바꿀 때 `.env` 파일 한 곳만 수정하면 되고, 민감한 값은 `.gitignore`에 등록해 저장소에 올라가지 않도록 관리할 수 있습니다.

### 절대 경로 vs 상대 경로

| 구분 | 설명 | 예시 |
|------|------|------|
| **절대 경로** | 루트(`/` 또는 드라이브)부터 시작하는 전체 경로 | `C:\Users\User\docker-web\app` |
| **상대 경로** | 현재 위치(`pwd`)를 기준으로 표현하는 경로 | `./app`, `../docker-web` |

**언제 어떤 경로를 쓰는가:**
- **절대 경로** : 스크립트, 설정 파일처럼 실행 위치가 달라져도 항상 같은 파일을 가리켜야 할 때 사용합니다. 단, 특정 PC의 경로에 종속되어 다른 환경에서 깨질 수 있습니다.
- **상대 경로** : `docker build .`처럼 현재 디렉토리를 기준으로 동작하는 명령이나, 프로젝트 내부 파일 참조에 사용합니다. 프로젝트 폴더를 어디로 옮겨도 내부 구조가 유지되면 경로가 깨지지 않아 **재현성**이 높아집니다.
- 이번 실습에서는 `docker build .`(상대), `COPY app/`(상대)처럼 프로젝트 내부는 상대 경로로, 볼륨 영속성 실습의 데이터 확인(`/data/test.txt`)은 컨테이너 내부 절대 경로로 사용하였습니다.

---

## 5. 파일 권한 실습

리눅스 환경(Ubuntu 컨테이너 내부)에서 파일 및 디렉토리 권한을 확인하고 변경하는 실습을 진행하였습니다.

### 권한의 핵심 원리

리눅스에서 모든 파일과 디렉토리는 **누가(who)** / **무엇을(what)** 할 수 있는지를 세 가지 대상과 세 가지 행위로 정의합니다.

**세 가지 대상 (왼쪽부터)**
```
-rwxr-xr-x
 ↑↑↑ ↑↑↑ ↑↑↑
소유자 그룹  기타(others)
```
- **소유자(owner)** : 파일을 만든 사람
- **그룹(group)** : 소유자가 속한 그룹의 다른 사용자들
- **기타(others)** : 소유자도 그룹도 아닌 모든 사람

**세 가지 행위**

| 기호 | 이름 | 숫자 | 파일에서의 의미 | 디렉토리에서의 의미 |
|------|------|------|----------------|-------------------|
| `r` | read | 4 | 파일 내용 읽기 | 디렉토리 목록(`ls`) 조회 |
| `w` | write | 2 | 파일 내용 수정 | 파일 생성/삭제 |
| `x` | execute | 1 | 파일 실행 | 디렉토리 진입(`cd`) |

**숫자 표기가 만들어지는 원리:**  
각 대상의 권한을 r(4) + w(2) + x(1)의 합으로 표현합니다.

```
755 = 소유자(7=4+2+1=rwx) / 그룹(5=4+0+1=r-x) / 기타(5=4+0+1=r-x)
644 = 소유자(6=4+2+0=rw-) / 그룹(4=4+0+0=r--) / 기타(4=4+0+0=r--)
```

**실무에서 755와 644를 쓰는 이유:**
- `644` : 일반 텍스트 파일(설정 파일, HTML 등)의 표준 권한입니다. 소유자만 수정 가능하고 나머지는 읽기만 허용하여 실수로 덮어쓰는 것을 방지합니다.
- `755` : 실행 파일이나 디렉토리의 표준 권한입니다. 소유자는 모든 권한, 그룹과 기타는 읽기와 진입만 허용합니다. 웹 서버가 디렉토리를 탐색할 때 `x`(진입) 권한이 반드시 필요합니다.

**권한 변경이 실제로 어떤 영향을 주는가:**  
디렉토리에 `x`(execute) 권한이 없으면 `cd` 명령으로 진입 자체가 불가능합니다.  
`r`만 있어도 `ls`로 목록은 볼 수 있지만, 그 안의 파일에는 접근할 수 없습니다.  
이처럼 권한은 단순히 숫자를 외우는 것이 아니라, **누가 어떤 행위를 할 수 있는가**라는 의미를 이해하고 설정해야 합니다.

```bash
# 파일 권한 확인
ls -l /data/test.txt

# 파일 권한 변경 (644 → 755)
chmod 755 /data/test.txt
ls -l /data/test.txt

# 디렉토리 권한 확인 및 변경
ls -ld /data
chmod 755 /data
ls -ld /data
```

---

## 6. Docker 설치 및 기본 점검

Docker가 정상적으로 설치되었는지 버전 및 데몬 동작 여부를 확인하였습니다.

```powershell
docker --version
docker info
```

`docker info` 명령어를 통해 Docker 데몬이 정상적으로 동작 중임을 확인하였습니다.

---

## 7. hello-world 실행

Docker가 정상적으로 설치되었는지 확인하기 위해 hello-world 컨테이너를 실행하였습니다.

```powershell
docker run hello-world
```

![hello-world](images/hello-world.png)

hello-world 컨테이너 실행을 통해 Docker 이미지 다운로드,  
컨테이너 생성 및 실행 과정이 정상적으로 이루어지는 것을 확인하였습니다.

---

## 8. Ubuntu 컨테이너 실행

Ubuntu 컨테이너를 실행하여 컨테이너 내부에서 리눅스 명령어를 실행해 보았습니다.

```bash
docker run -it ubuntu bash
ls
echo hello
exit
```

![ubuntu-container](images/ubuntu%20%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88.png)

컨테이너 내부에서 파일 목록 확인 및 문자열 출력 명령어를 실행하였으며  
컨테이너 내부는 하나의 독립된 리눅스 환경처럼 동작한다는 것을 확인하였습니다.

### attach vs exec 차이 관찰

| 방식 | 설명 | exit 시 동작 |
|------|------|-------------|
| `docker attach` | 컨테이너의 메인 프로세스에 직접 연결 | 컨테이너 **종료** |
| `docker exec -it` | 실행 중인 컨테이너에 새 프로세스로 진입 | 컨테이너 **유지** |

---

## 9. Dockerfile 작성

웹 서버 컨테이너를 만들기 위해 Dockerfile을 작성하였습니다.

```dockerfile
FROM nginx:alpine
COPY app/ /usr/share/nginx/html/
```

- `FROM nginx:alpine` : nginx가 설치된 경량 리눅스 이미지를 베이스로 사용
- `COPY app/` : 로컬의 app 폴더를 nginx 웹 루트 경로로 복사하여 커스텀 페이지 제공

![dockerfile](images/Dockerfile%20%EB%82%B4%EC%9A%A9.png)

Dockerfile을 이용하면 원하는 실행 환경을 이미지로 만들어 재사용할 수 있습니다.

---

## 10. Docker 이미지 빌드

Dockerfile을 이용하여 웹 서버 이미지를 생성하였습니다.

```powershell
docker build -t my-web:1.0 .
```

![image-build](images/%EC%9D%B4%EB%AF%B8%EC%A7%80%20%EB%B9%8C%EB%93%9C.png)

`docker build` 명령어를 통해 Dockerfile을 기반으로 새로운 이미지를 생성하였습니다.

---

## 11. 웹 컨테이너 실행 및 포트 매핑

생성한 이미지를 이용하여 웹 컨테이너를 실행하였습니다.

```powershell
docker run -d -p 8080:80 --name my-web my-web:1.0
```

브라우저에서 아래 주소로 접속하여 웹 페이지가 정상적으로 실행되는 것을 확인하였습니다.

```
http://localhost:8080
```

![web-container](images/%EC%9B%B9%20%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88%20%EC%8B%A4%ED%96%89.png)

### 포트 매핑이 필요한 이유 — 컨테이너 네트워크 격리 구조

컨테이너는 기본적으로 **Docker 내부에 격리된 가상 네트워크** 안에서 실행됩니다.  
이 내부 네트워크는 호스트(내 PC)의 네트워크와 **완전히 분리**되어 있기 때문에,  
컨테이너 내부에서 nginx가 80 포트로 서비스를 열고 있어도  
호스트 브라우저에서 `localhost:80`으로 직접 접속하면 **연결이 거부**됩니다.

**왜 직접 접속이 안 되는가:**
```
[브라우저] → localhost:80 → [호스트 네트워크] → ? → [컨테이너 내부 80포트]
                                                  ↑
                          Docker 내부 네트워크 경계 — 직접 통과 불가
```

포트 매핑(`-p 8080:80`)은 이 경계에 **터널**을 뚫는 역할을 합니다.  
호스트의 8080 포트로 들어온 요청을 Docker가 받아서 컨테이너 내부 80 포트로 전달해줍니다.

```
[브라우저] → localhost:8080 → [Docker] → 컨테이너:80 → nginx 응답
```

**포트 설정 재현 방법:**  
이번 실습에서는 동일한 이미지를 여러 포트로 실행하여 재현성을 검증하였습니다.

```powershell
# 8080 포트로 첫 번째 컨테이너 실행
docker run -d -p 8080:80 --name my-web my-web:1.0

# 8082 포트로 Bind Mount 컨테이너 실행 (동시에 독립적으로 동작)
docker run -d -p 8082:80 -v "${PWD}/app:/usr/share/nginx/html" nginx
```

두 컨테이너가 동시에 실행되어도 포트가 다르므로 충돌 없이 각각 독립적으로 접속 가능합니다.  
이처럼 포트 매핑을 문서화해두면 누구나 동일한 명령으로 환경을 재현할 수 있습니다.

---

## 12. Docker 운영 명령 수행

이미지 목록, 컨테이너 목록, 로그, 리소스 사용량을 확인하는 운영 명령어를 수행하였습니다.

```powershell
# 이미지 목록 확인
docker images

# 모든 컨테이너 목록 (종료된 컨테이너 포함)
docker ps -a

# 컨테이너 로그 확인
docker logs my-web

# 리소스 사용량 확인 (1회 출력 후 종료)
docker stats --no-stream
```

![container-list](images/%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88%20%EB%AA%A9%EB%A1%9D.png)

![image-list](images/%EC%9D%B4%EB%AF%B8%EC%A7%80%20%EB%AA%A9%EB%A1%9D.png)

---

## 13. Bind Mount 실습

Bind Mount를 이용하여 호스트 PC의 폴더와 컨테이너 내부 폴더를 연결하였습니다.

```powershell
docker run -d -p 8082:80 -v ${PWD}/app:/usr/share/nginx/html nginx
```

`index.html` 파일을 수정한 후 브라우저를 새로고침하였을 때  
컨테이너 내부 웹 페이지 내용이 즉시 변경되는 것을 확인하였습니다.

![bind-mount](images/%EB%B0%94%EC%9D%B8%EB%93%9C%20%EB%A7%88%EC%9A%B4%ED%8A%B8.png)

Bind Mount는 호스트 PC와 컨테이너가 **동일한 파일을 실시간으로 공유**할 때 사용하는 기능입니다.  
개발 중 코드를 수정하면 컨테이너 재시작 없이 바로 반영되어 개발 효율이 높아집니다.

---

## 14. Docker Volume 영속성 실습

Docker Volume을 이용하여 컨테이너 데이터를 영구적으로 저장하는 실습을 진행하였습니다.

```powershell
# 볼륨 생성
docker volume create mydata

# 첫 번째 컨테이너에 볼륨 연결 및 데이터 기록
docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
docker exec -it vol-test bash
# 컨테이너 내부
echo hello > /data/test.txt
exit

# 첫 번째 컨테이너 삭제
docker rm -f vol-test

# 두 번째 컨테이너에 동일 볼륨 연결 후 데이터 확인
docker run -d --name vol-test2 -v mydata:/data ubuntu sleep infinity
docker exec -it vol-test2 bash
# 컨테이너 내부
cat /data/test.txt
# hello → 데이터 유지 확인
```

![volume](images/%EB%B3%BC%EB%A5%A8.png)

컨테이너를 삭제하더라도 볼륨에 저장된 데이터는 유지되는 것을 확인하였습니다.

### 볼륨 없이 컨테이너를 삭제하면 어떻게 되는가 — 직접 경험

실습 초기에 볼륨을 연결하지 않은 상태에서 컨테이너 내부에 파일을 저장하고 삭제해보았습니다.

```bash
# 볼륨 없이 컨테이너 실행
docker run -d --name no-vol-test ubuntu sleep infinity
docker exec -it no-vol-test bash -c "echo 'important data' > /tmp/data.txt && cat /tmp/data.txt"
# important data 출력 확인

# 컨테이너 삭제
docker rm -f no-vol-test

# 새 컨테이너로 다시 실행 — 이전 데이터 없음
docker run -d --name no-vol-test2 ubuntu sleep infinity
docker exec -it no-vol-test2 bash -c "cat /tmp/data.txt"
# cat: /tmp/data.txt: No such file or directory → 데이터 소멸 확인
```

**이 경험에서 배운 것:**  
컨테이너는 이미지 위에 임시 쓰기 레이어를 올려서 실행됩니다.  
`docker rm`으로 컨테이너를 삭제하면 이 임시 레이어째로 데이터가 사라집니다.  
처음에는 "컨테이너가 살아있는 동안은 데이터가 유지되니까 괜찮겠지"라고 생각했지만,  
배포 시 컨테이너를 재생성하거나, 장애로 컨테이너가 재시작되면 데이터가 날아간다는 것을 깨달았습니다.  
이 문제를 해결하기 위한 방법이 바로 **Docker Volume**입니다.

### Bind Mount vs Docker Volume 비교

| 구분 | Bind Mount | Docker Volume |
|------|-----------|--------------|
| 저장 위치 | 호스트의 **지정 경로** | Docker가 관리하는 **내부 영역** |
| 주요 용도 | 개발 중 코드 실시간 반영 | DB, 로그 등 **영속 데이터** 보관 |
| 컨테이너 삭제 후 | 호스트 파일 유지 | 볼륨 별도 삭제 전까지 데이터 유지 |
| 경로 지정 | 호스트 경로를 직접 지정 | `docker volume create`로 Docker가 관리 |
| 이식성 | 호스트 경로 구조에 의존 | 경로 무관하게 이름으로 연결 가능 |

**실무에서의 선택 기준:**  
- 개발 중 코드를 수정하면서 바로 결과를 보고 싶을 때 → **Bind Mount**  
- DB 데이터나 업로드 파일처럼 컨테이너가 재생성돼도 반드시 살아남아야 할 데이터 → **Docker Volume**

---

## 15. Git 설정 및 GitHub 연동

Git 사용자 정보와 기본 브랜치를 설정하고 GitHub에 연동하였습니다.

```powershell
# 사용자 정보 설정
git config --global user.name "이름"
git config --global user.email "이메일@example.com"

# 기본 브랜치 설정
git config --global init.defaultBranch main

# 설정 전체 확인
git config --list
```

### Git과 GitHub의 역할 차이

| 구분 | Git | GitHub |
|------|-----|--------|
| 역할 | **로컬** 버전 관리 도구 | **원격** 협업 플랫폼 |
| 동작 위치 | 내 컴퓨터 | 클라우드 (웹) |
| 주요 기능 | 커밋, 브랜치, 히스토리 관리 | 원격 저장소, PR, 이슈, 팀 협업 |

---

## 16. Docker 구조 정리

Docker의 기본 구조는 다음과 같습니다.

```
Dockerfile → Image → Container → Port Mapping → Bind Mount → Volume
```

### 이미지(Image)와 컨테이너(Container)의 관계 — 빌드 / 실행 / 변경 관점

| 관점 | Image | Container |
|------|-------|-----------|
| **빌드** | `docker build`로 Dockerfile을 읽어 **레이어 형태로 생성**됨. 한 번 만들어지면 내용이 변하지 않는 읽기 전용 스냅샷 | 이미지를 기반으로 `docker run`으로 생성. 이미지 위에 **쓰기 가능한 레이어**가 한 겹 추가됨 |
| **실행** | 실행되지 않음. 실행을 위한 **설계도(템플릿)** 역할 | 실제로 **프로세스가 동작하는 런타임 환경**. 같은 이미지로 여러 컨테이너를 동시에 실행 가능 |
| **변경** | 내용을 직접 수정 불가. 변경하려면 Dockerfile을 수정하고 **다시 빌드**해야 새 이미지가 생성됨 | 컨테이너 안에서 파일을 수정할 수 있으나, `docker rm`으로 삭제하면 **변경 내용이 사라짐**. 영속적으로 유지하려면 Volume이나 Bind Mount 필요 |

**핵심 비유:**  
이미지는 붕어빵 틀(설계도), 컨테이너는 그 틀로 구운 붕어빵(실행 결과)입니다.  
틀(이미지)은 변하지 않고, 붕어빵(컨테이너)은 먹고 나면(삭제하면) 사라지지만 틀은 남습니다.  
그래서 컨테이너 안에서 저장한 데이터를 남기려면 Volume이나 Bind Mount라는 **외부 보관함**이 필요합니다.

```powershell
# 이미지 목록 — 삭제되지 않고 재사용 가능한 템플릿
docker images

# 컨테이너 실행 — 이미지를 기반으로 새 런타임 환경 생성
docker run -d -p 8080:80 --name my-web my-web:1.0

# 컨테이너 삭제 — 런타임 환경과 내부 변경 내용이 사라짐. 이미지는 유지됨
docker rm -f my-web

# 동일 이미지로 다시 실행 가능 — 항상 같은 초기 상태에서 시작
docker run -d -p 8080:80 --name my-web my-web:1.0
```

---

## 17. 트러블슈팅

### 트러블슈팅 1 — 포트 충돌로 컨테이너 실행 실패

**문제**  
`docker run -d -p 8080:80` 실행 시 아래 오류 발생:
```
Error response from daemon: Bind for 0.0.0.0:8080 failed: port is already allocated
```

**원인 가설**  
이미 8080 포트를 점유 중인 컨테이너 또는 다른 프로세스가 존재할 것이라 판단하였습니다.

**확인**  
```powershell
docker ps -a
netstat -ano | findstr :8080
```
이미 실행 중인 컨테이너가 8080 포트를 사용하고 있음을 확인하였습니다.

**해결**  
기존 컨테이너를 중지하거나, 다른 포트(8081)로 변경하여 실행하였습니다.
```powershell
docker stop my-web
docker run -d -p 8081:80 --name my-web2 my-web:1.0
```

---

### 트러블슈팅 2 — Bind Mount 경로 오류 (Windows PowerShell)

**문제**  
PowerShell에서 Bind Mount 실행 시 경로를 인식하지 못하는 문제가 발생하였습니다.
```
docker: invalid reference format
```

**원인 가설**  
Windows 경로 형식(`C:\Users\...`)을 Docker가 정상적으로 인식하지 못할 것이라 판단하였습니다.

**확인**  
`${PWD}` 를 직접 절대 경로로 입력해도 동일한 오류가 발생하였습니다.

**해결**  
PowerShell에서는 경로를 큰따옴표로 감싸 해결하였습니다.
```powershell
docker run -d -p 8082:80 -v "${PWD}/app:/usr/share/nginx/html" nginx
```

---

## 18. 검증 방법 요약

| 항목 | 검증 명령 | 확인 내용 |
|------|----------|----------|
| Docker 설치 | `docker --version` | 버전 출력 확인 |
| 데몬 동작 | `docker info` | 정상 응답 확인 |
| 이미지 빌드 | `docker images` | `my-web:1.0` 목록 확인 |
| 컨테이너 실행 | `docker ps` | STATUS = Up 확인 |
| 포트 매핑 | 브라우저 `localhost:8080` 접속 | 웹 페이지 정상 출력 확인 |
| Bind Mount | 파일 수정 후 브라우저 새로고침 | 변경 내용 즉시 반영 확인 |
| Volume 영속성 | 컨테이너 삭제 후 재생성, `cat /data/test.txt` | 데이터 유지 확인 |
| Git 설정 | `git config --list` | `user.name` / `user.email` 확인 |
| SSH 인증 | `ssh -T git@github.com` | 인증 성공 메시지 확인 |

---

## 19. 보너스 과제

### 19-1. Docker Compose 기초 (단일 서비스)

`docker-compose.yml` 파일을 작성하여 단일 nginx 서비스를 Compose로 실행하였습니다.

```yaml
version: '3'

services:
  web:
    image: nginx
    ports:
      - "8090:80"
```

![compose-yml](images/docker-compose.yml%20%ED%8C%8C%EC%9D%BC%20%ED%99%94%EB%A9%B4.png)

```powershell
# Compose로 서비스 실행
docker compose up -d

# 실행 상태 확인
docker compose ps
```

![compose-up](images/docker%20compose%20up%20-d%20%EC%8B%A4%ED%96%89%20%ED%99%94%EB%A9%B4.png)

**배움 포인트:**  
`docker run` 명령어로 매번 옵션을 입력하던 방식이 `docker-compose.yml`이라는 **문서화된 실행 설정**으로 바뀝니다.  
팀원 누구나 `docker compose up -d` 한 줄로 동일한 환경을 실행할 수 있어 재현성이 높아집니다.

---

### 19-2. Docker Compose 멀티 컨테이너

nginx 웹 서버와 ubuntu 2개의 서비스를 Compose로 함께 실행하였습니다.

```yaml
version: '3'

services:
  web:
    image: nginx
    ports:
      - "8091:80"
  ubuntu:
    image: ubuntu
    command: sleep infinity
```

```powershell
docker compose up -d
docker compose ps
```

![compose-multi](images/docker%20compose%20ps%20(%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88%202%EA%B0%9C%20%EB%96%A0%EC%9E%88%EB%8A%94%20%ED%99%94%EB%A9%B4).png)

`docker compose ps` 결과에서 `docker-web-ubuntu-1`과 `docker-web-web-1` 두 컨테이너가  
동시에 실행되는 것을 확인하였습니다.

**배움 포인트:**  
Compose로 띄운 서비스들은 자동으로 **같은 네트워크(`docker-web_default`)** 에 묶입니다.  
서비스 이름(예: `web`, `ubuntu`)을 호스트명처럼 사용하여 컨테이너 간 통신이 가능합니다.

---

### 19-3. Compose 운영 명령어

```powershell
# 서비스 실행
docker compose up -d

# 실행 중인 서비스 확인
docker compose ps

# 서비스 로그 확인
docker compose logs

# 서비스 종료 및 컨테이너 삭제
docker compose down
```

![compose-ps](images/docker%20compose%20ps%20%EA%B2%B0%EA%B3%BC%20%ED%99%94%EB%A9%B4.png)

**배움 포인트:**  
`up / down / ps / logs` 네 가지 명령으로 서비스의 **실행 → 상태 확인 → 로그 확인 → 종료** 전체 사이클을 관리할 수 있습니다.

---

### 19-4. 환경 변수 활용 (.env 파일로 포트 변경)

`.env` 파일에 환경 변수를 정의하고 `docker-compose.yml`에서 참조하여 포트를 변경하였습니다.

**.env 파일**
```
PORT=8092
```

![env-file](images/.env%ED%8C%8C%EC%9D%BC.png)

**docker-compose.yml**
```yaml
version: '3'

services:
  web:
    image: nginx
    ports:
      - "${PORT}:80"
```

```powershell
docker compose up -d
# localhost:8092 로 접속 확인
```

![env-port](images/localhost%20%EC%A0%91%EC%86%8D%20%ED%99%94%EB%A9%B4.png)

**배움 포인트:**  
포트, 환경 모드, DB 비밀번호 같은 **설정값을 코드에서 분리**할 수 있습니다.  
`.env` 파일만 바꾸면 환경(개발/스테이징/운영)별로 다른 설정을 적용할 수 있어  
실무에서 매우 널리 사용되는 패턴입니다.

---

### 19-5. GitHub SSH 키 설정

HTTPS 대신 SSH 방식으로 GitHub에 인증하도록 SSH 키를 생성하고 등록하였습니다.

```powershell
# SSH 키 생성
ssh-keygen -t ed25519 -C "이메일@example.com"

# 공개 키 확인 (GitHub에 등록할 키)
cat ~/.ssh/id_ed25519.pub

# SSH 인증 테스트
ssh -T git@github.com
```

![ssh-github](images/SSH%20GitHub%20%EC%97%B0%EA%B2%B0.png)

`Hi clae-dev! You've successfully authenticated` 메시지를 통해 SSH 인증이 정상적으로 완료된 것을 확인하였습니다.

**HTTPS vs SSH 인증 방식 비교**

| 구분 | HTTPS | SSH |
|------|-------|-----|
| 인증 방식 | 아이디 / 토큰 입력 | 키 파일로 자동 인증 |
| 매번 인증 필요 | O (토큰 캐싱 전까지) | X (키 등록 후 자동) |
| 보안 | 토큰 노출 위험 | 개인키가 로컬에만 존재 |
| 실무 권장 | 일반적 사용 | **서버/자동화 환경 권장** |

---

## 20. 미션 회고

### 가장 어려웠던 지점과 해결 과정

이번 미션에서 가장 어려웠던 부분은 **Windows PowerShell 환경에서의 경로 처리**와 **포트 충돌 디버깅**이었습니다.

**경험 1 — Bind Mount 경로 오류**

| 단계 | 내용 |
|------|------|
| 문제 | `docker run -v ${PWD}/app:...` 실행 시 `invalid reference format` 오류 |
| 가설 | `${PWD}`가 Windows 경로 형식으로 해석되어 Docker가 인식 못하는 것 같다 |
| 확인 | 절대 경로로 직접 입력해봐도 동일한 오류 발생 → 경로 형식 자체가 문제임을 확인 |
| 조치 | 큰따옴표로 감싸서 `"${PWD}/app:..."` 형식으로 변경 → 정상 동작 |
| 배움 | Windows에서 Docker를 쓸 때는 경로 표기 방식 차이를 반드시 고려해야 한다는 것을 체득 |

**경험 2 — 포트 충돌로 컨테이너 실행 실패**

| 단계 | 내용 |
|------|------|
| 문제 | `port is already allocated` 오류로 컨테이너 실행 불가 |
| 가설 | 이전에 실행한 컨테이너가 해당 포트를 점유하고 있을 것이다 |
| 확인 | `docker ps -a`로 종료되지 않은 컨테이너 확인, `netstat`으로 포트 점유 프로세스 확인 |
| 조치 | 기존 컨테이너 중지 후 재실행, 또는 포트 번호를 8081로 변경 |
| 배움 | 컨테이너를 stop해도 포트는 해제되지만, rm하지 않으면 목록에 남아 혼란을 줄 수 있다 |

### 미션 전체를 통해 달라진 시각

미션 시작 전에는 Docker를 단순히 컨테이너를 실행하는 도구라고만 알고 있었습니다.  
직접 트러블슈팅을 하면서 **왜 이런 구조가 필요한지**를 이해하게 되었습니다.

- 볼륨 없이 데이터가 사라지는 경험을 통해 → **영속성 설계의 중요성** 인식
- 포트 충돌을 진단하면서 → **컨테이너 네트워크 격리 구조** 이해
- `.env` 파일로 설정을 분리하면서 → **코드와 설정의 분리** 개념 체감
- Compose로 멀티 컨테이너를 띄우면서 → **서비스 디스커버리와 네트워크 자동 구성** 경험

이번 실습을 통해 Docker의 기본 구조와 동작 방식을 이해하였으며,  
이후 CI/CD 파이프라인, 클라우드 배포 환경으로 자연스럽게 확장할 수 있는 기반을 갖추게 되었습니다.
