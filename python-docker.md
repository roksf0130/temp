# Docker 기반 Python 3.12 개발환경 구축 및 관리 가이드

> 대상 환경: macOS(Apple Silicon 포함) + Docker Desktop + Visual Studio Code + Dev Containers
> 개발 도구: Python 3.12, Pylance, Python Debugger, Ruff, pip
> 목표: **Mac에는 VS Code, Dev Containers 확장, Docker 실행 환경만 설치**하고 Python 및 나머지 개발 도구는 컨테이너 내부에서 사용한다.

## 목차

1. [구성 개요](#1-구성-개요)
2. [사전 준비](#2-사전-준비)
3. [프로젝트 파일 만들기](#3-프로젝트-파일-만들기)
4. [Dev Container 실행 및 검증](#4-dev-container-실행-및-검증)
5. [패키지 및 라이브러리 추가·삭제·업데이트](#5-패키지-및-라이브러리-추가삭제업데이트)
6. [개발 중 반드시 알아둘 내용](#6-개발-중-반드시-알아둘-내용)
7. [Docker 자원 확인 및 일상 정리](#7-docker-자원-확인-및-일상-정리)
8. [프로젝트 개발환경 삭제](#8-프로젝트-개발환경-삭제)
9. [Docker 전체 정리 및 공장 초기화](#9-docker-전체-정리-및-공장-초기화)
10. [문제 해결 및 재구축 체크리스트](#10-문제-해결-및-재구축-체크리스트)

---

## 1. 구성 개요

```text
Mac (로컬)
├── Docker Desktop / Docker Engine
├── VS Code
├── VS Code 확장: Dev Containers (로컬에만 설치)
└── python-docker/ (소스코드 및 설정 파일은 로컬에 보관)
    └── Dockerfile(python:3.12-slim 기반)으로 빌드한 개발 컨테이너에 Bind Mount
        ├── Python 3.12
        ├── pip 및 프로젝트 패키지
        ├── Ruff (린터 + 포맷터)
        └── VS Code 원격 확장: Python / Pylance / Python Debugger / Ruff
```

- 로컬 Mac에 Python 3.12, Ruff, Pylance, Python Debugger를 별도 설치할 필요가 없다.
- **Dev Containers 확장 자체는 VS Code 로컬에 설치**되어 있어야 한다.
- 개발 컨테이너 확장은 컨테이너 쪽 확장 호스트에 설치된다. 일부 UI 확장은 특성에 따라 로컬에서 실행될 수 있다.
- VS Code가 열어 놓은 프로젝트 폴더를 컨테이너에 연결하므로 **컨테이너를 삭제해도 Mac의 프로젝트 파일은 보통 남는다.** 다만 컨테이너 내부에만 만든 파일은 사라질 수 있다.
- 컨테이너마다 OS 환경이 분리된다. Mac이더라도 컨테이너 내부는 Linux다.

## 2. 사전 준비

1. Docker Desktop(또는 호환 Docker Engine)을 설치하고 실행한다.
2. VS Code를 설치한다.
3. VS Code 로컬 확장에서 **Dev Containers** (`ms-vscode-remote.remote-containers`)만 필수로 설치한다.
4. Mac 터미널에서 아래를 확인한다.

```bash
docker --version
docker info
```

`docker --version`은 CLI 설치 여부만 보여 준다. `docker info`가 정상 실행되어야 Docker 엔진에 연결된 상태다.

> Mac에 VS Code와 Dev Containers 외의 **개발 도구**를 설치하지 않는 구성이다. Docker 자체는 컨테이너 실행에 필요하므로 예외다.

## 3. 프로젝트 파일 만들기

Mac 터미널에서:

```bash
mkdir -p python-docker/.devcontainer
cd python-docker
code .
```

`code` 명령이 없다면 VS Code의 **Shell Command: Install 'code' command in PATH**를 실행하거나 VS Code 메뉴에서 폴더를 직접 연다.

구조:

```text
python-docker/
├── .devcontainer/
│   └── devcontainer.json
├── Dockerfile
├── .dockerignore         # 권장
├── main.py
├── pyproject.toml
├── requirements.txt      # 런타임 의존성
├── requirements-dev.txt  # 개발 도구(Ruff 등)
└── .gitignore            # 권장
```

### 3.1 `Dockerfile`

프로젝트 루트에 만든다. `mcr.microsoft.com/devcontainers/python` 같은 편의 이미지 대신 공식 `python:3.12-slim`을 직접 사용한다.

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PATH="/home/vscode/.local/bin:${PATH}"

# 개발에 필요한 최소 도구. 필요 없으면 제거
RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

# non-root 사용자 (UID 1000)
ARG USERNAME=vscode
ARG UID=1000
ARG GID=1000
RUN groupadd --gid ${GID} ${USERNAME} \
    && useradd --uid ${UID} --gid ${GID} -m -s /bin/bash ${USERNAME}

USER ${USERNAME}
```

- `FROM python:3.12-slim`: Python 3.12 최소 이미지. Node, ESLint 등 불필요한 도구와 VS Code 확장이 딸려 오지 않는다.
- `PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED`: `.pyc` 생성을 막고 로그를 즉시 출력한다.
- `PATH`에 `~/.local/bin`을 추가한다. non-root 사용자가 `pip install --user`로 설치한 `ruff` 같은 실행 파일을 찾기 위해 필요하다.
- non-root 사용자(`vscode`)를 만들어 컨테이너 내부에서 root로 작업하지 않는다. `devcontainer.json`의 `remoteUser`와 이름이 일치해야 한다.
- 작업 디렉터리는 Dev Containers가 자동으로 `/workspaces/<폴더명>`에 마운트하므로 `WORKDIR`를 지정하지 않는다.
- `git` 외 도구(`curl`, `zsh` 등)가 필요하면 `apt-get install` 목록에 추가한다.

> **참고: 이전 이미지에서 autopep8/ESLint가 설치되던 이유**
> `mcr.microsoft.com/devcontainers/python` 이미지는 `devcontainer.metadata` 라벨로 node·python·git feature를 포함한다. 이 feature들이 `devcontainer.json`에 적지 않은 `ms-python.autopep8`, `dbaeumer.vscode-eslint` 확장과 `[python]` 기본 포매터(autopep8) 설정을 함께 적용한다. `python:3.12-slim`에는 이런 메타데이터가 없으므로 `devcontainer.json`에 적은 확장과 설정만 적용된다.

### 3.2 `.devcontainer/devcontainer.json`

```json
{
  "name": "Python 3.12 Dev",
  "build": {
    "dockerfile": "../Dockerfile",
    "context": ".."
  },
  "remoteUser": "vscode",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "ms-python.debugpy",
        "charliermarsh.ruff"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python",
        "[python]": {
          "editor.defaultFormatter": "charliermarsh.ruff",
          "editor.formatOnSave": true,
          "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
          }
        },
        "ruff.importStrategy": "fromEnvironment",
        "ruff.nativeServer": "on"
      }
    }
  },
  "postCreateCommand": "python -m pip install --user -r requirements-dev.txt"
}
```

- `build`: 프로젝트 루트의 `Dockerfile`로 이미지를 빌드한다. `dockerfile` 경로는 `devcontainer.json` 기준 상대 경로이고, `context`는 프로젝트 루트(`..`)다. 최초 실행 시 베이스 이미지를 다운로드하고 빌드한다.
- `remoteUser`: Dockerfile에서 만든 non-root 사용자와 일치시킨다.
- `extensions`: 로컬이 아닌 **개발 컨테이너 환경에** 설치할 VS Code 확장 목록.
- `settings`: Ruff를 Python 파일의 기본 포맷터로 사용하고, 파일 저장 시 포맷팅한다.
- `editor.codeActionsOnSave`의 `explicit`는 사용자가 직접 저장할 때 Ruff의 자동 수정과 import 정리를 수행하도록 한다. 자동 저장에서도 실행하려면 설정을 별도로 조정한다.
- `ruff.importStrategy: fromEnvironment`: 확장이 환경에 설치된 Ruff 실행 파일을 사용한다. 따라서 `requirements-dev.txt`에 `ruff`를 포함하고 버전을 고정해 CLI와 에디터의 Ruff 버전을 일치시킨다. 설치가 끝나기 전에는 확장이 Ruff를 못 찾는다는 오류가 잠깐 보일 수 있다.
- `postCreateCommand`: 컨테이너 생성 후 개발 의존성(`requirements-dev.txt`, 런타임 의존성 포함)을 설치한다. non-root 사용자는 `/usr/local`에 쓸 수 없으므로 `--user`로 설치한다. **평소 requirements 변경만으로 이 명령이 자동 재실행되지는 않는다.** 아래의 `pip install -r` 명령으로 직접 동기화한다.

> **참고:** 위 `python.defaultInterpreterPath`는 `python:3.12-slim` 이미지의 Python 위치(`/usr/local/bin/python`)를 전제로 한다. `which python` 결과가 다르다면 해당 경로로 변경한다. Python 인터프리터 선택이 이미 저장되어 있다면 VS Code에서 **Python: Select Interpreter**로 다시 선택할 수 있다.

### 3.3 `pyproject.toml`: Ruff 규칙

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F", "I", "B", "UP"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

- `F`: 흔한 코드 오류, `I`: import 정리, `B`: 버그 가능성, `UP`: 최신 Python 문법 권장.
- Ruff는 린팅과 포맷팅을 모두 수행하므로 별도 Flake8/Black/isort가 필수는 아니다.
- `line-length = 88`은 기본값이라 생략해도 동작은 같다. 명시해 두는 것은 의도를 드러내기 위한 것이다.
- 필요하면 `SIM`, `C4`, `PTH`, `RUF` 같은 규칙을 `select`에 추가한다. 패키지 구조를 만든 뒤에는 `[tool.ruff.lint.isort] known-first-party = [...]`도 설정한다.

### 3.4 `requirements.txt`와 `requirements-dev.txt`: Python 패키지

런타임 의존성과 개발 도구를 분리하고 버전을 고정한다.

`requirements.txt` (앱 실행에 필요한 패키지):

```text
# 예: requests==2.32.3
```

`requirements-dev.txt` (개발 도구):

```text
-r requirements.txt
ruff==0.17.6
```

- `requirements-dev.txt`가 `-r requirements.txt`로 런타임 의존성을 포함하므로 개발 컨테이너에서는 이 파일 하나만 설치하면 된다.
- 위 Ruff 버전은 예시다. 사용할 버전을 정해 고정하고, 올릴 때는 이 파일만 수정한다.
- 나중에 `requests`, `pandas` 등 필요한 패키지는 `requirements.txt`에 추가한다. 테스트·타입 체크 도구(`pytest`, `mypy` 등)는 `requirements-dev.txt`에 추가한다.

### 3.5 `main.py`: 실행 테스트

```python
import sys


def main() -> None:
    print("Hello, Docker Python!")
    print("Python:", sys.version)


if __name__ == "__main__":
    main()
```

`main()` 함수와 `__main__` 가드로 감싸면 import 시 부수효과가 없고 테스트하기 쉽다.

### 3.6 `.gitignore` (권장)

```gitignore
__pycache__/
*.py[cod]
.venv/
.env
.env.*
!.env.example
.pytest_cache/
.ruff_cache/
.mypy_cache/
.coverage
htmlcov/
dist/
build/
*.egg-info/
.DS_Store
```

비밀 키가 들어 있는 `.env` 파일은 Git에 올리지 않는다. 필요한 경우 키 이름만 적힌 `.env.example`을 공유한다. `!.env.example` 예외 규칙을 두었다면 실제 `.env.example` 파일도 만들어 둔다.

### 3.7 `.dockerignore` (권장)

```text
.git
.venv
__pycache__/
*.py[cod]
.ruff_cache/
.pytest_cache/
.mypy_cache/
.env
.env.*
.DS_Store
```

이미지 빌드 컨텍스트(`context: ".."`)에서 불필요한 파일과 비밀 파일을 제외해 빌드를 가볍게 하고 `.env` 같은 파일이 이미지에 들어가지 않게 한다.

## 4. Dev Container 실행 및 검증

1. Docker Desktop이 실행 중인지 확인한다.
2. VS Code에서 `python-docker` 폴더를 연다.
3. `Cmd + Shift + P` → **Dev Containers: Reopen in Container**를 선택한다.
4. 최초에는 이미지 다운로드와 컨테이너 생성, 확장·패키지 설치가 진행된다.
5. VS Code의 새 터미널을 열어 다음 명령을 실행한다.

```bash
whoami
python --version
which python
python -m pip --version
ruff --version
ruff check .
ruff format --check .
python main.py
```

`whoami`는 `vscode`, `python --version`은 `Python 3.12.x`가 나오고 코드가 실행되면 정상이다. `ruff format --check .`는 포맷이 맞지 않으면 종료 코드가 0이 아닐 수 있다. 수정하려면 `ruff format .` 실행.

**확장 위치 확인:** VS Code 확장 화면에서 Dev Containers는 로컬에, Python/Pylance/Python Debugger/Ruff는 **Dev Container** 측에 설치되어 있는지 확인한다.

> VS Code에서 **Dev Containers: Reopen Folder Locally**를 선택하면 로컬 폴더로 돌아올 수 있다. 이는 컨테이너 삭제가 아니다.

## 5. 패키지 및 라이브러리 추가·삭제·업데이트

**모든 Python 패키지 명령은 Dev Container에 연결된 VS Code 터미널에서 실행한다.**

### 5.1 가장 추천하는 방법: 목록에 먼저 기록

1. 앱 실행에 필요한 라이브러리는 `requirements.txt`에, 개발 도구는 `requirements-dev.txt`에 추가한다. 가능하면 버전을 고정한다.

```text
# requirements.txt
pandas==2.2.3
requests==2.32.3
```

```text
# requirements-dev.txt
-r requirements.txt
ruff==0.17.6
```

2. 컨테이너 터미널에서 설치한다.

```bash
python -m pip install -r requirements-dev.txt
```

3. 확인한다.

```bash
python -m pip show pandas
python -c "import pandas; print(pandas.__version__)"
```

이 방식은 실제 설치와 기록을 함께 관리하기 쉽다. **패키지만 추가할 때는 Docker 이미지 재빌드가 필요 없다.**

### 5.2 자주 쓰는 pip 명령

| 할 일 | 명령어 |
|---|---|
| 하나 설치 | `python -m pip install pandas` |
| 여러 개 설치 | `python -m pip install pandas requests` |
| 특정 버전 설치 | `python -m pip install 'pandas==2.2.3'` |
| 패키지 업그레이드 | `python -m pip install --upgrade pandas` |
| 패키지 삭제 | `python -m pip uninstall pandas` |
| 목록 확인 | `python -m pip list` |
| 상세 정보 확인 | `python -m pip show pandas` |
| requirements 전체 설치 | `python -m pip install -r requirements-dev.txt` |

**중요:** 설치·삭제·업데이트를 수행했다면 `requirements.txt`(개발 도구는 `requirements-dev.txt`)도 변경한다. 이 파일에서 패키지를 지우는 것만으로 현재 컨테이너에서 자동 삭제되지는 않는다. 필요하면 `pip uninstall`을 실행한다.

정확한 버전 재현이 중요하면 패키지 버전을 고정하거나 잠금(lock) 파일을 도입한다. 단순한 `requirements.txt`에 버전을 적지 않으면 재설치 시 더 최신 버전이 설치될 수 있다.

### 5.3 pip와 uv 중 무엇을 쓸까?

이 가이드는 **pip 기반 최소 구성**이다. 나중에 의존성 잠금과 프로젝트 관리를 강화하고 싶다면 `uv`를 컨테이너 내부에만 도입할 수 있다. 로컬에 uv를 설치할 필요는 없다. uv 도입 시 `uv.lock`, `pyproject.toml`, 가상환경 위치 등을 다시 설계하는 것이 좋다.

### 5.4 패키지 설치와 재빌드 차이

| 변경 | 일반적으로 필요한 작업 |
|---|---|
| Python 소스 수정 | 저장 후 바로 실행 |
| `requirements.txt` / `requirements-dev.txt` 수정 | 컨테이너에서 `python -m pip install -r requirements-dev.txt` |
| Ruff 설정(`pyproject.toml`) 수정 | 대부분 즉시 반영 또는 확장 재시작 |
| `Dockerfile` 또는 `.devcontainer/devcontainer.json`의 빌드·확장 설정 수정 | **Dev Containers: Rebuild Container** |
| Python 3.12 → 3.13 변경 | `Dockerfile`의 `FROM` 태그와 `pyproject.toml`의 `target-version` 변경 후 Rebuild |
| 컨테이너에만 수동 설치한 패키지 | Rebuild 시 사라질 수 있으므로 설정에 기록 |

## 6. 개발 중 반드시 알아둘 내용

### 6.1 Docker 자원 5가지

| 자원 | 의미 | 주의점 |
|---|---|---|
| 이미지(Image) | 컨테이너 생성용 템플릿 | 삭제하면 다시 다운로드/빌드 필요 |
| 컨테이너(Container) | 이미지로 만든 실행환경 | 컨테이너 내부에만 기록한 데이터는 삭제 시 유실될 수 있음 |
| 볼륨(Volume) | Docker가 관리하는 영속 저장소 | 볼륨 삭제는 데이터 영구 삭제 가능 |
| Bind Mount | Mac의 디렉터리를 컨테이너에 연결 | 컨테이너 삭제와 로컬 폴더 삭제는 별개 |
| 네트워크(Network) | 컨테이너 간 통신 | Docker 기본 네트워크는 원래 존재 |

추가로 **빌드 캐시(Build Cache)**는 이미지 빌드에 사용한 중간 데이터를 저장한다. 삭제하면 재빌드가 느려질 수 있지만 소스코드를 직접 삭제하지는 않는다.

### 6.2 Stop, Start, Remove, Rebuild

- `stop`: 실행 중지. 컨테이너는 남는다.
- `start`: 기존 컨테이너 실행.
- `restart`: 기존 컨테이너 재시작.
- `rm`: 컨테이너 제거. 컨테이너의 쓰기 가능 레이어 데이터도 제거된다.
- Dev Containers `Rebuild Container`: 개발 컨테이너 재생성. 수동 설치 내역은 사라질 수 있다.

### 6.3 로컬 터미널과 컨테이너 터미널

- **Mac 로컬:** `docker ps`, `docker images`, `docker volume ls`, `docker system df`, 삭제/초기화 명령.
- **Dev Container:** `python`, `pip`, `ruff` 및 앱 실행.

`pwd`, `uname -a`, `which python`으로 현재 위치·환경을 확인할 수 있다.

### 6.4 포트, localhost, 파일 경로

- 컨테이너의 `localhost`는 기본적으로 컨테이너 자신이다.
- 컨테이너에서 웹 서버 실행 시 VS Code의 포트 포워딩을 쓸 수 있다. 예: `devcontainer.json` 최상위에 `"forwardPorts": [8000]` 추가.
- 웹 서버가 모든 인터페이스에서 연결을 받아야 하는 상황에서는 앱을 `0.0.0.0`에 바인딩해야 할 수 있다.
- Linux 컨테이너에서는 `/workspaces/...` 같은 경로를 쓰며 Mac의 `/Users/...`와 다르다.
- Apple Silicon Mac에서는 `arm64` 기반 이미지를 기본 사용하되, 일부 의존성의 아키텍처 호환성을 확인한다.

### 6.5 보안과 재현성

- API 키, DB 비밀번호, 토큰은 소스에 하드코딩하지 않는다.
- `.env`를 `.gitignore`에 추가한다. `.env` 파일이 있다는 사실만으로 컨테이너 환경변수로 자동 전달되지는 않는다.
- `.env`와 `.env.*`는 `.dockerignore`에도 넣어 이미지 빌드 컨텍스트에 포함되지 않게 한다.
- 컨테이너는 non-root 사용자(`vscode`)로 실행한다. root 권한이 필요한 작업은 `Dockerfile` 빌드 단계에서 처리한다.
- 이미지 버전, Python 버전, 라이브러리 버전을 기록한다. 정말 동일한 이미지 재현이 필요하면 digest 고정을 검토한다.
- 컨테이너 데이터베이스는 볼륨 백업이 필요하다. `docker volume rm` 및 공장 초기화는 데이터 복구를 어렵게 한다.
- **Bind Mount는 삭제 방지 장치가 아니다.** 컨테이너 내부 프로그램이 마운트된 소스 파일을 수정·삭제하면 Mac 원본에도 반영된다. Git 백업을 유지한다.

## 7. Docker 자원 확인 및 일상 정리

아래 명령은 **Mac 로컬 터미널**에서 실행한다.

```bash
docker ps                 # 실행 중인 컨테이너
docker ps -a              # 모든 컨테이너
docker images             # 이미지
docker volume ls          # 볼륨
docker network ls         # 네트워크
docker system df          # Docker 디스크 사용량
docker system df -v       # 상세 디스크 사용량
docker stats              # 실행 중 컨테이너 자원 사용
```

특정 컨테이너의 정보가 필요할 때:

```bash
docker logs <CONTAINER_NAME_OR_ID>
docker inspect <CONTAINER_NAME_OR_ID>
docker exec -it <RUNNING_CONTAINER_NAME_OR_ID> bash
```

사용하지 않는 빌드 캐시만 정리하는 예:

```bash
docker builder prune
```

사용하지 않는 이미지 정리:

```bash
docker image prune -a
```

**주의:** `prune` 명령은 Docker 전체 범위에서 사용되지 않는 자원을 정리할 수 있다. 다른 프로젝트의 재빌드나 기동에 영향을 줄 수 있다.

## 8. 프로젝트 개발환경 삭제

**다른 Docker 프로젝트를 보존하려면 이 방법을 사용한다.**

1. VS Code에서 **Dev Containers: Reopen Folder Locally**를 실행한다.
2. Mac 터미널에서 컨테이너를 식별한다.

```bash
docker ps -a
```

3. 대상이 맞는지 확인 후 **해당 컨테이너만** 삭제한다.

```bash
docker rm -f <CONTAINER_ID>
```

4. 관련 이미지와 볼륨을 조회한다.

```bash
docker images
docker volume ls
```

5. 해당 프로젝트 전용이고 다른 곳에서 쓰지 않는 것이 확실할 경우에만 삭제한다.

```bash
docker rmi <IMAGE_ID>
docker volume rm <VOLUME_NAME>
```

**주의 사항**

- 이미지/볼륨은 다른 프로젝트에서 공유할 수 있다. 이름만 보고 지우지 않는다.
- `Dockerfile`로 빌드한 개발 이미지는 보통 `vsc-<프로젝트명>-<해시>` 형태의 이름으로 로컬에 생성된다. `docker images`에서 확인하고, 베이스 이미지(`python:3.12-slim`)는 다른 프로젝트가 쓸 수 있으니 함께 지우지 않도록 주의한다.
- Dev Containers 확장 관련 공유 볼륨도 있을 수 있다.
- Mac의 `python-docker` 폴더는 위 Docker 명령으로 삭제되지 않는다.
- 로컬 프로젝트 폴더 자체를 제거하려면 별도의 파일 삭제가 필요하지만, 중요한 코드라면 우선 Git 또는 백업으로 보관한다.

## 9. Docker 전체 정리 및 공장 초기화

> ⚠️ **파괴적 작업**: 아래 전체 삭제는 현재 Python 프로젝트뿐 아니라 **다른 모든 Docker 프로젝트**에 영향을 줄 수 있다. DB 볼륨, 인증 상태, 컨테이너 데이터가 영구 삭제될 수 있다. 백업 확인 전에는 실행하지 않는다.

### 9.1 CLI로 Docker 자원 정리 (설정은 유지)

Mac 터미널에서 자원을 먼저 확인한다.

```bash
docker ps -a
docker images
docker volume ls
docker network ls
docker system df
```

**1단계 — 모든 컨테이너 삭제:**

```bash
# 우선 삭제 대상 목록 확인
docker ps -aq

# 모든 컨테이너 ID가 나열되는 것이 확인되면 삭제
ids=$(docker ps -aq)
if [ -n "$ids" ]; then
  docker rm -f $ids
fi
```

이는 현재 실행 중인 컨테이너까지 강제로 제거한다.

**2단계 — 사용하지 않는 이미지/네트워크/빌드 캐시 정리:**

```bash
docker system prune -a --volumes
```

`docker system prune --volumes`는 **모든 named volume 삭제를 보장하지 않는다.**

**3단계 — 사용하지 않는 볼륨(named volume 포함) 정리:**

```bash
docker volume prune -a
```

`docker volume prune -a`는 Docker 버전에 따라 지원 여부를 확인해야 한다. 지원하지 않는 버전에서는 `docker volume prune --help`를 확인한다. **컨테이너를 전부 삭제했다면 볼륨은 대부분 더 이상 참조되지 않으므로 데이터 유실 위험이 매우 크다.**

**4단계 — 결과 확인:**

```bash
docker ps -a
docker images
docker volume ls
docker network ls
docker system df
```

기본 네트워크(`bridge`, `host`, `none` 등)는 남는 것이 정상이다. 이 방법으로 Docker Desktop의 모든 설정 및 내부 상태가 초기화되는 것은 아니다.

### 9.2 Docker Desktop 공장 초기화 (가장 확실한 초기화)

설치 직후와 유사한 상태로 되돌리려면 Docker Desktop을 실행하고 **Troubleshoot(문제 해결)** 메뉴를 찾는다.

- **Clean / Purge data** 등 데이터 정리 기능: Docker 데이터 삭제(메뉴 명칭은 버전별로 상이할 수 있음).
- **Reset to factory defaults**: Docker Desktop 설정까지 초기화.

버전에 따라 메뉴 명칭·위치가 다를 수 있으므로 초기화 전에 화면에 표시된 삭제 범위를 읽고 확인한다. 필요한 데이터와 볼륨을 백업한 뒤 진행한다.

이 작업은 Docker가 관리하는 이미지·컨테이너·볼륨 등에 영향을 줄 수 있지만 일반적인 Bind Mount 원본인 Mac 프로젝트 폴더 자체를 삭제하는 작업은 아니다.

### 9.3 삭제 범위별 추천

| 목적 | 권장 방법 |
|---|---|
| 현재 Python 프로젝트만 제거 | [8장 프로젝트 개발환경 삭제](#8-프로젝트-개발환경-삭제) |
| 빌드 캐시 정리 | `docker builder prune` |
| 사용하지 않는 이미지 정리 | `docker image prune -a` |
| Docker 자원 전반 정리 | [9.1 CLI 정리](#91-cli로-docker-자원-정리-설정은-유지) |
| Docker Desktop까지 공장 초기화 | [9.2 Factory Reset](#92-docker-desktop-공장-초기화-가장-확실한-초기화) |

## 10. 문제 해결 및 재구축 체크리스트

### 10.1 자주 겪는 문제

| 증상 | 확인 사항/조치 |
|---|---|
| `docker info` 오류 | Docker Desktop 실행 여부와 엔진 연결 상태 확인 |
| `code: command not found` | VS Code에서 `code` CLI 설치 또는 폴더 수동 열기 |
| Dev Container 생성 실패 | Docker 로그, VS Code **Dev Containers: Show Container Log**, 이미지 다운로드/네트워크 확인 |
| `python` 버전이 다름 | 컨테이너에 연결됐는지, `which python`, **Python: Select Interpreter** 확인 |
| Ruff 명령을 못 찾음 | `python -m pip install -r requirements-dev.txt`, `ruff --version` 확인. 그래도 안 되면 `echo $PATH`에 `/home/vscode/.local/bin`이 있는지(Dockerfile의 `ENV PATH`) 확인 |
| 컨테이너 안에서 파일 쓰기 권한 오류 | non-root 사용자(`vscode`, UID 1000)로 실행 중이다. 시스템 경로에는 쓸 수 없으므로 `pip install --user`를 쓰고, 시스템 패키지 설치는 `Dockerfile`에 추가한다 |
| 확장이 예상과 다르게 설치됨 | 베이스 이미지 메타데이터가 확장을 추가할 수 있다. `python:3.12-slim`은 없지만 이미지를 바꿨다면 `docker image inspect <이미지> --format '{{index .Config.Labels "devcontainer.metadata"}}'`로 확인 |
| Ruff 확장이 작동하지 않음 | 확장이 컨테이너 측에 설치됐는지, Ruff 출력 패널과 `ruff.importStrategy` 확인 |
| 라이브러리를 설치했는데 import 실패 | 설치한 터미널과 VS Code가 선택한 Python 인터프리터가 동일한지 확인 |
| Rebuild 후 패키지가 사라짐 | `requirements.txt`, `requirements-dev.txt`와 `postCreateCommand`를 확인 |
| 이미지 빌드 실패 | **Dev Containers: Show Container Log**에서 `Dockerfile` 오류 확인. 캐시 문제가 의심되면 **Rebuild Without Cache** |
| Mac 브라우저에서 웹 서버 접근 불가 | 서버 바인딩 주소 및 VS Code 포트 포워딩 확인 |
| 디스크 용량 부족 | 먼저 `docker system df -v` 확인 후 필요한 자원만 정리 |

### 10.2 다음에 처음부터 다시 구축할 때

- [ ] Docker Desktop을 설치하고 실행한다.
- [ ] VS Code와 **로컬 Dev Containers 확장**을 설치한다.
- [ ] Mac에 기존 프로젝트 파일이 있다면 폴더를 VS Code로 연다.
- [ ] 프로젝트에 `.devcontainer/devcontainer.json`, `Dockerfile`, `requirements.txt`, `requirements-dev.txt`, `pyproject.toml`이 있는지 확인한다.
- [ ] VS Code 명령 팔레트 → **Dev Containers: Reopen in Container**.
- [ ] 컨테이너 터미널에서 `python --version`, `ruff --version`을 확인한다.
- [ ] `python -m pip install -r requirements-dev.txt`를 실행해 필요한 패키지를 확인한다.
- [ ] `ruff check .`, `ruff format --check .`, `python main.py`를 실행한다.
- [ ] 개발 중 새 패키지가 필요하면 `requirements.txt`를 수정한 다음 `python -m pip install -r requirements-dev.txt`를 실행한다.

### 10.3 전체 삭제 전 필수 체크리스트

- [ ] 프로젝트 소스코드가 Mac에 남아 있고 Git/백업이 준비되었다.
- [ ] `.devcontainer`, `Dockerfile`, `requirements.txt`, `requirements-dev.txt`, `pyproject.toml` 파일을 보관했다.
- [ ] Docker 볼륨에만 있는 데이터(DB 등)를 별도 백업했다.
- [ ] 다른 프로젝트의 컨테이너·이미지·볼륨까지 삭제되는지 확인했다.
- [ ] 프로젝트만 지울지, 자원 전체를 지울지, Desktop 공장 초기화까지 할지 결정했다.

---

## 공식 문서

- [VS Code: Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)
- [Dev Container specification](https://containers.dev/implementors/json_reference/)
- [Ruff documentation](https://docs.astral.sh/ruff/)
- [Docker: docker system prune](https://docs.docker.com/reference/cli/docker/system/prune/)
- [Docker: docker volume prune](https://docs.docker.com/reference/cli/docker/volume/prune/)
- [Docker Desktop troubleshooting](https://docs.docker.com/desktop/troubleshoot/)

> 이 문서는 **로컬 Python 개발 도구를 설치하지 않는 Dev Containers 중심 구성**을 기준으로 한다. Docker, VS Code 및 확장 기능은 업데이트되므로, 명령/설정이 달라졌다면 공식 문서를 함께 확인한다.
