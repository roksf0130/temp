# Docker 기반 Rust 개발환경 구축 및 관리 가이드

> 대상 환경: macOS(Apple Silicon 포함) + Docker Desktop + Visual Studio Code + Dev Containers
> 개발 도구: Rust stable, rustup, Cargo, rust-analyzer, Clippy, rustfmt
> 목표: **Mac에는 VS Code, Dev Containers 확장, Docker 실행 환경만 설치**하고 Rust 컴파일러·라이브러리·나머지 개발 도구는 컨테이너 내부에서 사용한다.

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
└── rust-docker/ (소스코드 및 설정 파일은 로컬에 보관)
    └── Dockerfile(rust:1-bookworm 기반)으로 빌드한 개발 컨테이너에 Bind Mount
        ├── Linux + Rust stable (rustc / rustup)
        ├── Cargo (빌드·의존성·테스트)
        ├── Clippy (린터) / rustfmt (포맷터)
        ├── Docker 볼륨: target/, Cargo 레지스트리 캐시
        └── VS Code 원격 확장: rust-analyzer / Even Better TOML
```

- 로컬 Mac에 Rust, Cargo, Clippy, rustfmt, rust-analyzer를 별도 설치할 필요가 없다.
- **Dev Containers 확장 자체는 VS Code 로컬에 설치**되어 있어야 한다.
- 개발 컨테이너 확장은 컨테이너 쪽 확장 호스트에 설치된다. 일부 UI 확장은 특성에 따라 로컬에서 실행될 수 있다.
- VS Code가 열어 놓은 프로젝트 폴더를 컨테이너에 연결하므로 **컨테이너를 삭제해도 Mac의 프로젝트 파일은 보통 남는다.** 다만 컨테이너 내부에만 만든 파일은 사라질 수 있다.
- 컨테이너마다 OS 환경이 분리된다. Mac(Apple Silicon 포함)이더라도 컨테이너 내부는 Linux다. 즉 **Linux용 실행 파일을 빌드**하며 macOS 실행 파일이 자동 생성되는 것은 아니다.

### Python 개발환경과 비교

| Python 환경 | Rust 환경 | 역할 |
|---|---|---|
| `python` | `rustc` | 언어 실행/컴파일 도구 (동작 방식은 다름) |
| `pip` | `cargo add` / Cargo | 외부 라이브러리(의존성) 관리 |
| `requirements.txt` | `Cargo.toml` | 프로젝트 의존성 선언 |
| 잠금 파일(별도 관리 가능) | `Cargo.lock` | 의존성 버전 확정 |
| Ruff | Clippy + rustfmt | 린트 및 포맷팅 |
| Pylance | rust-analyzer | 자동 완성, 타입·코드 분석 |

Rust에서는 Python과 달리 대개 **코드를 컴파일한 다음 실행**한다. `cargo run`이 빌드와 실행을 함께 처리해 준다.

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
mkdir -p rust-docker/.devcontainer rust-docker/src
cd rust-docker
git init
code .
```

`code` 명령이 없다면 VS Code의 **Shell Command: Install 'code' command in PATH**를 실행하거나 VS Code 메뉴에서 폴더를 직접 연다.

구조:

```text
rust-docker/
├── .devcontainer/
│   └── devcontainer.json
├── src/
│   └── main.rs
├── Dockerfile
├── .dockerignore         # 권장
├── Cargo.toml
├── rust-toolchain.toml
└── .gitignore            # 권장
```

`Cargo.lock`은 첫 Cargo 빌드에서 생성된다. `target/`도 빌드 과정에서 생성된다.

> **Mac에서 Rust를 직접 실행해 `target/`이 이미 생겼다면 삭제**한다. macOS용 빌드 결과가 Linux 컨테이너의 빌드와 섞이면 불필요한 전체 재빌드나 충돌이 생긴다.

### 3.1 `Dockerfile`

프로젝트 루트에 만든다. `mcr.microsoft.com/devcontainers/rust` 같은 편의 이미지 대신 공식 `rust:1-bookworm`을 직접 사용한다.

```dockerfile
FROM rust:1-bookworm

ENV CARGO_TERM_COLOR=always

# 개발에 필요한 최소 도구. sudo는 postCreateCommand의 chown용
RUN apt-get update \
    && apt-get install -y --no-install-recommends sudo \
    && rm -rf /var/lib/apt/lists/*

# non-root 사용자 (UID 1000)
ARG USERNAME=vscode
ARG UID=1000
ARG GID=1000
RUN groupadd --gid ${GID} ${USERNAME} \
    && useradd --uid ${UID} --gid ${GID} -m -s /bin/bash ${USERNAME} \
    && echo "${USERNAME} ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME} \
    && chown -R ${USERNAME}:${USERNAME} /usr/local/cargo /usr/local/rustup

USER ${USERNAME}
```

- `FROM rust:1-bookworm`: Rust 공식 이미지(Debian bookworm 기반). `git`과 C 빌드 도구(gcc 등)가 포함되어 있어 별도 `apt-get install` 없이 일반적인 크레이트를 빌드할 수 있다. 이미지는 갱신되므로 정확한 Rust 버전은 `rust-toolchain.toml`로 관리한다.
- non-root 사용자(`vscode`)를 만들어 컨테이너 내부에서 root로 작업하지 않는다. `devcontainer.json`의 `remoteUser`와 이름이 일치해야 한다.
- `chown`: 이미지의 `CARGO_HOME`(`/usr/local/cargo`)과 `RUSTUP_HOME`(`/usr/local/rustup`)은 root 소유다. non-root 사용자가 `cargo add`, `rustup`의 toolchain 설치(`rust-toolchain.toml` 적용)를 하려면 소유권을 넘겨야 한다.
- 작업 디렉터리는 Dev Containers가 자동으로 `/workspaces/<폴더명>`에 마운트하므로 `WORKDIR`를 지정하지 않는다.
- `sudo`: 새로 만들어진 볼륨(`target/` 등)의 소유권을 `postCreateCommand`에서 바꾸기 위해 설치한다. 필요 없다고 판단되면 `sudo`와 `chown` 단계를 함께 제거한다.
- OpenSSL 등 추가 시스템 패키지가 필요하면 `apt-get install` 단계를 이 파일에 추가한다. (예: `pkg-config`, `libssl-dev`)

> **참고: 이전 이미지에서 확장·권한이 추가되던 이유**
> `mcr.microsoft.com/devcontainers/rust` 이미지는 `devcontainer.metadata` 라벨로 feature와 설정을 포함한다. 이 메타데이터가 `devcontainer.json`에 적지 않은 확장(CodeLLDB, Dependi 등)과 디버깅용 권한 옵션(`SYS_PTRACE`, `seccomp=unconfined` 등)을 함께 적용할 수 있다. `rust:1-bookworm`에는 이런 메타데이터가 없으므로 `devcontainer.json`에 적은 확장과 설정만 적용된다.

### 3.2 `.devcontainer/devcontainer.json`

```json
{
  "name": "Rust Dev",
  "build": {
    "dockerfile": "../Dockerfile",
    "context": ".."
  },
  "remoteUser": "vscode",
  "mounts": [
    "source=${devcontainerId}-cargo-registry,target=/usr/local/cargo/registry,type=volume",
    "source=${devcontainerId}-target,target=${containerWorkspaceFolder}/target,type=volume"
  ],
  "customizations": {
    "vscode": {
      "extensions": [
        "rust-lang.rust-analyzer",
        "tamasfe.even-better-toml"
      ],
      "settings": {
        "[rust]": {
          "editor.defaultFormatter": "rust-lang.rust-analyzer",
          "editor.formatOnSave": true
        },
        "rust-analyzer.check.command": "clippy",
        "rust-analyzer.cargo.allTargets": true,
        "files.watcherExclude": {
          "**/target/**": true
        }
      }
    }
  },
  "postCreateCommand": "sudo chown -R vscode: /usr/local/cargo/registry \"${containerWorkspaceFolder}/target\" && rustup show && cargo --version && rustc --version"
}
```

- `build`: 프로젝트 루트의 `Dockerfile`로 이미지를 빌드한다. `dockerfile` 경로는 `devcontainer.json` 기준 상대 경로이고, `context`는 프로젝트 루트(`..`)다. 최초 실행 시 베이스 이미지를 다운로드하고 빌드한다.
- `remoteUser`: Dockerfile에서 만든 non-root 사용자와 일치시킨다.
- `mounts`: **`target/`과 Cargo 레지스트리 캐시를 Docker named volume으로 분리**한다. macOS의 Bind Mount는 작은 파일 I/O가 느려 `target/`을 Mac 폴더에 두면 빌드가 크게 느려진다. `${devcontainerId}`는 프로젝트별로 고유한 값이라 프로젝트 간 볼륨이 섞이지 않는다. (Mac 폴더에는 빈 `target/` 디렉터리만 마운트 지점으로 남는다.)
- `extensions`: 로컬이 아닌 **개발 컨테이너 환경에** 설치할 VS Code 확장 목록.
- `settings`: rust-analyzer를 Rust 파일의 기본 포맷터로 사용하고(내부적으로 rustfmt), 파일 저장 시 포맷팅한다. `rust-analyzer.check.command: clippy`는 진단에 `cargo clippy`를 사용한다.
- `postCreateCommand`: 컨테이너 생성 후 (1) 새로 만들어진 `target/` 볼륨은 root 소유이므로 `vscode`로 소유권을 바꾸고, (2) `rustup show`로 `rust-toolchain.toml`에 선언된 toolchain·컴포넌트(rustfmt, clippy)를 **미리 설치**한다. 이렇게 하지 않으면 첫 `cargo` 실행 때 toolchain을 내려받느라 지연된다. `rust:1-bookworm`에는 `sudo`가 없으므로 3.1의 `Dockerfile`에서 설치하고 `vscode` 사용자에게 비밀번호 없는 sudo를 허용했다. (개발 컨테이너의 일반적인 구성이다.)

> **참고:** 레지스트리 볼륨 경로(`/usr/local/cargo/registry`)는 이미지의 `CARGO_HOME` 기준이다. 이미지를 바꿨다면 `echo $CARGO_HOME`로 확인한다.

### 3.3 `rust-toolchain.toml`: Rust 버전과 필수 컴포넌트

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
profile = "minimal"
```

- `stable`: 안정화된 최신 Rust를 사용한다. 이는 **정확한 버전 고정이 아니다**. 서로 완전히 동일한 컴파일러 버전이 필요하면 `channel`을 `1.xx.y` 형태의 **실제로 존재하는 버전**으로 바꾼다.
- **권장:** 팀/CI와 공유하게 되면 버전을 고정한다. 컨테이너에서 `rustc --version`으로 현재 버전을 확인한 뒤 그 값을 `channel = "1.xx.y"`로 적고, 업그레이드는 의도적으로 올린다.
- `components`: 포맷터와 린터를 프로젝트 필수 구성 요소로 선언한다.
- `profile`: 기본 설치 구성을 간소화한다.
- 이 파일이 있는 폴더에서 `cargo`/`rustc`/`rustup`을 실행하면 해당 프로젝트 toolchain이 적용된다.

> 이미지에 설치된 toolchain과 `rust-toolchain.toml`이 다르면 컨테이너 안에서 해당 toolchain을 추가로 내려받는다. 위 `postCreateCommand`의 `rustup show`가 이 작업을 컨테이너 생성 시점에 미리 수행한다.

### 3.4 `Cargo.toml`: 프로젝트 및 의존성 선언

```toml
[package]
name = "rust-docker"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"

[dependencies]

[lints.rust]
unsafe_code = "forbid"

[lints.clippy]
all = { level = "warn", priority = -1 }
```

- `name`: 프로젝트(크레이트) 이름. 프로젝트 폴더명과 맞춰 두는 것이 좋다. 빌드되는 바이너리 이름도 이 값을 따른다.
- `edition`: Rust 언어 에디션. 현대의 안정화된 툴체인을 사용할 때 `2024`를 선택할 수 있다. 이전 프로젝트와 호환성 문제가 있으면 `2021`도 가능하다.
- `rust-version`: 이 프로젝트를 빌드하는 데 필요한 최소 Rust 버전. edition 2024는 1.85 이상이 필요하다.
- `[dependencies]`: 나중에 `serde`, `reqwest` 같은 외부 라이브러리가 들어갈 자리이다.
- `[lints]`: 린트 설정을 코드 저장소에 선언한다. `unsafe` 코드를 금지하고 Clippy 기본 그룹을 경고로 켠다. `unsafe`가 필요한 프로젝트라면 `unsafe_code` 줄을 제거한다.

### 3.5 `src/main.rs`: 실행 테스트

```rust
fn main() {
    println!("Hello, Docker Rust!");
    println!("2 + 3 = {}", 2 + 3);
}
```

### 3.6 `.gitignore` (권장)

```gitignore
/target/
.env
.env.*
!.env.example
.idea/
.DS_Store
```

- `target/`: Cargo 컴파일 결과 및 중간 파일이 저장되므로 Git에서 제외한다. 이 구성에서는 Docker 볼륨으로 분리되어 있으나 마운트 지점 디렉터리가 Mac에 남으므로 계속 제외한다.
- 비밀 키가 들어 있는 `.env` 파일은 Git에 올리지 않는다. 필요한 경우 키 이름만 적힌 `.env.example`을 공유한다. `!.env.example` 예외 규칙을 두었다면 실제 `.env.example` 파일도 만들어 둔다.
- **`Cargo.lock`은 `.gitignore`에 넣지 않는다.** 특히 실행 프로그램(application/binary)은 의존성 재현성을 위해 Git에 커밋하는 것이 일반적이다. 재사용 라이브러리의 lockfile 처리 정책은 프로젝트 성격에 따라 다를 수 있다.

### 3.7 `.dockerignore` (권장)

```text
.git
target/
.env
.env.*
.DS_Store
```

이미지 빌드 컨텍스트(`context: ".."`)에서 불필요한 파일과 비밀 파일을 제외해 빌드를 가볍게 하고 `.env` 같은 파일이 이미지에 들어가지 않게 한다. 특히 Mac에 남은 `target/`이 빌드 컨텍스트로 전송되지 않도록 반드시 포함한다.

## 4. Dev Container 실행 및 검증

1. Docker Desktop이 실행 중인지 확인한다.
2. VS Code에서 `rust-docker` 폴더를 연다.
3. `Cmd + Shift + P` → **Dev Containers: Reopen in Container**를 선택한다.
4. 최초에는 이미지 다운로드와 컨테이너 생성, Rust toolchain·확장 설치가 진행된다.
5. VS Code의 새 터미널을 열어 다음 명령을 실행한다.

```bash
whoami
rustc --version
rustup show active-toolchain
cargo --version
cargo fmt --version
cargo clippy --version
cargo run
cargo check
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

`whoami`는 `vscode`가 나오고 `cargo run`이 아래처럼 출력되면 정상이다.

```text
Hello, Docker Rust!
2 + 3 = 5
```

- `cargo run`: 프로젝트를 빌드하고 실행한다.
- `cargo check`: 최종 실행 파일 생성 없이 빠르게 컴파일 오류를 검사한다.
- `cargo fmt --all -- --check`: 포맷이 올바른지만 확인한다. 포맷이 맞지 않으면 종료 코드가 0이 아닐 수 있다. 수정하려면 `cargo fmt --all` 실행.
- `cargo clippy --all-targets -- -D warnings`: lint 경고도 실패로 처리한다. 학습 단계에서는 `-D warnings`를 빼도 된다.
- `cargo test`: 테스트를 실행한다.

**확장 위치 확인:** VS Code 확장 화면에서 Dev Containers는 로컬에, rust-analyzer와 Even Better TOML은 **Dev Container** 측에 설치되어 있는지 확인한다.

> VS Code에서 **Dev Containers: Reopen Folder Locally**를 선택하면 로컬 폴더로 돌아올 수 있다. 이는 컨테이너 삭제가 아니다.

## 5. 패키지 및 라이브러리 추가·삭제·업데이트

Rust에서는 외부 라이브러리를 보통 **크레이트(crate)**라고 부른다. Cargo가 의존성 선언, 다운로드, 버전 해석, 빌드까지 관리한다.

**모든 Cargo 명령은 Dev Container에 연결된 VS Code 터미널에서 실행한다.**

### 5.1 가장 추천하는 방법: `cargo add`

JSON 데이터 처리용 `serde`와 `serde_json`을 예로 든다.

```bash
cargo add serde --features derive
cargo add serde_json
cargo check
```

`Cargo.toml`에 대략 다음과 같은 의존성이 추가된다(정확한 버전 번호는 명령 실행 시점에 따라 다르다).

```toml
[dependencies]
serde = { version = "...", features = ["derive"] }
serde_json = "..."
```

이때 `Cargo.lock`도 의존성 해석 시 생성 또는 갱신된다. 애플리케이션 프로젝트라면 **Cargo.toml과 Cargo.lock 둘 다 Git에 커밋**한다. 이 방식은 실제 설치와 기록이 함께 이루어진다. **패키지만 추가할 때는 Docker 이미지 재빌드가 필요 없다.**

라이브러리를 사용하는 예제로 `src/main.rs`를 바꿔 확인한다.

```rust
use serde::Serialize;

#[derive(Serialize)]
struct Person {
    name: String,
    score: u32,
}

fn main() {
    let person = Person {
        name: "Alice".to_string(),
        score: 90,
    };

    let json = serde_json::to_string_pretty(&person).expect("JSON 변환 실패");
    println!("{json}");
}
```

```bash
cargo run
```

### 5.2 자주 쓰는 Cargo 명령

| 할 일 | 명령어 |
|---|---|
| 빌드하고 실행 | `cargo run` |
| 빠른 타입·컴파일 검사 | `cargo check` |
| 개발용 빌드 | `cargo build` |
| 최적화된 빌드 | `cargo build --release` |
| 테스트 | `cargo test` |
| 린트 | `cargo clippy --all-targets` |
| 코드 포맷팅 | `cargo fmt --all` |
| 포맷 검사만 | `cargo fmt --all -- --check` |
| 의존성 추가 | `cargo add <crate>` |
| 개발 의존성 추가 | `cargo add --dev <crate>` |
| 의존성 제거 | `cargo remove <crate>` |
| 특정 크레이트 업데이트 | `cargo update -p <crate>` |
| lockfile 의존성 업데이트 | `cargo update` |
| 빌드 결과 삭제 | `cargo clean` |

- 개발할 때만 필요한 패키지(테스트용 등)는 `cargo add --dev tempfile`처럼 추가한다. `Cargo.toml`의 `[dev-dependencies]`에 기록된다.
- 패키지를 제거(`cargo remove serde_json`)했으면 코드에서 해당 패키지를 사용하는 부분도 수정해야 한다.
- `cargo update`는 **현재 `Cargo.toml`이 허용하는 버전 범위** 내에서 `Cargo.lock`의 의존성 버전을 갱신한다. 새 메이저 버전 등 선언 범위 밖의 버전이 필요하면 `cargo add serde_json@<원하는_버전>` 등으로 `Cargo.toml`도 갱신한다. 변경 후에는 `cargo check`, `cargo test`, `cargo clippy`를 실행한다.

**중요:** `cargo add`/`cargo remove`는 `Cargo.toml`을 자동으로 수정하므로 pip와 달리 목록 파일을 따로 고칠 필요가 없다. 반면 `Cargo.toml`을 손으로 수정했다면 `cargo check`로 의존성을 동기화한다.

정확한 버전 재현이 중요하면 `Cargo.lock`을 Git에 커밋하고, CI에서는 `--locked` 옵션을 사용한다.

### 5.3 `cargo add`와 `cargo install`의 차이

`cargo add`는 **프로젝트에 의존성을 추가**하는 명령이다. CLI 유틸리티를 환경에 설치하는 `cargo install <crate>`와 다르다. 컨테이너에 `cargo install`로 수동 설치한 도구는 Rebuild 시 사라질 수 있으니 `Dockerfile`이나 `postCreateCommand`에 설치 절차를 기록한다.

### 5.4 패키지 설치와 재빌드 차이

| 변경 | 일반적으로 필요한 작업 |
|---|---|
| Rust 소스 수정 | 저장 후 바로 `cargo run` |
| `Cargo.toml` 수정 / `cargo add` | 컨테이너에서 `cargo check`. 재빌드 불필요 |
| 린트 설정(`[lints]`) 수정 | 즉시 반영 또는 rust-analyzer 재시작 |
| `rust-toolchain.toml` 변경 | 대개 불필요(컨테이너 내부에서 toolchain 다운로드). `rustup show`로 미리 설치 가능 |
| `Dockerfile` 또는 `.devcontainer/devcontainer.json`의 빌드·확장·`mounts` 설정 수정 | **Dev Containers: Rebuild Container** |
| 시스템 패키지(OpenSSL 등) 추가 | `Dockerfile` 수정 후 Rebuild |
| 컨테이너에만 수동 설치한 도구(`cargo install` 등) | Rebuild 시 사라질 수 있으므로 설정에 기록 |

## 6. 개발 중 반드시 알아둘 내용

### 6.1 Docker 자원 5가지

| 자원 | 의미 | 주의점 |
|---|---|---|
| 이미지(Image) | 컨테이너 생성용 템플릿 | 삭제하면 다시 다운로드/빌드 필요 |
| 컨테이너(Container) | 이미지로 만든 실행환경 | 컨테이너 내부에만 기록한 데이터는 삭제 시 유실될 수 있음 |
| 볼륨(Volume) | Docker가 관리하는 영속 저장소 | 이 구성에서는 `target/`·Cargo 캐시가 저장됨. 삭제하면 캐시 유실 |
| Bind Mount | Mac의 디렉터리를 컨테이너에 연결 | 컨테이너 삭제와 로컬 폴더 삭제는 별개 |
| 네트워크(Network) | 컨테이너 간 통신 | Docker 기본 네트워크는 원래 존재 |

추가로 **빌드 캐시(Build Cache)**는 이미지 빌드에 사용한 중간 데이터를 저장한다. 삭제하면 재빌드가 느려질 수 있지만 소스코드를 직접 삭제하지는 않는다.

### 6.2 Stop, Start, Remove, Rebuild

- `stop`: 실행 중지. 컨테이너는 남는다.
- `start`: 기존 컨테이너 실행.
- `restart`: 기존 컨테이너 재시작.
- `rm`: 컨테이너 제거. 컨테이너의 쓰기 가능 레이어 데이터도 제거된다. (named volume은 남는다.)
- Dev Containers `Rebuild Container`: 개발 컨테이너 재생성. 수동 설치 내역은 사라질 수 있다.

### 6.3 로컬 터미널과 컨테이너 터미널

- **Mac 로컬:** `docker ps`, `docker images`, `docker volume ls`, `docker system df`, 삭제/초기화 명령.
- **Dev Container:** `cargo`, `rustc`, `rustup` 및 앱 실행.

`pwd`, `uname -a`, `which cargo`로 현재 위치·환경을 확인할 수 있다.

### 6.4 포트, localhost, 파일 경로

- 컨테이너의 `localhost`는 기본적으로 컨테이너 자신이다.
- 컨테이너에서 웹 서버 실행 시 VS Code의 포트 포워딩을 쓸 수 있다. 예: `devcontainer.json` 최상위에 `"forwardPorts": [3000]` 추가.
- 웹 서버가 모든 인터페이스에서 연결을 받아야 하는 상황에서는 앱을 `0.0.0.0`에 바인딩해야 할 수 있다. Mac에서 접속 가능한 주소는 VS Code의 **Ports** 패널에서 확인한다.
- Linux 컨테이너에서는 `/workspaces/...` 같은 경로를 쓰며 Mac의 `/Users/...`와 다르다.
- Apple Silicon Mac에서는 `arm64` 기반 이미지를 기본 사용하되, 일부 의존성의 아키텍처 호환성을 확인한다. `rustc -vV`와 `uname -m`으로 target을 확인할 수 있다.

### 6.5 보안과 재현성

- API 키, DB 비밀번호, 토큰은 소스에 하드코딩하지 않는다.
- `.env`를 `.gitignore`에 추가한다. `.env` 파일이 있다는 사실만으로 컨테이너 환경변수로 자동 전달되지는 않는다.
- `.env`와 `.env.*`는 `.dockerignore`에도 넣어 이미지 빌드 컨텍스트에 포함되지 않게 한다.
- 컨테이너는 non-root 사용자(`vscode`)로 실행한다. root 권한이 필요한 작업은 `Dockerfile` 빌드 단계에서 처리한다.
- **컨테이너는 완전한 보안 경계가 아니다.** Docker 소켓 마운트, 무분별한 `--privileged`, 호스트 디렉터리 권한 확장은 피한다. LLDB 디버깅이 필요할 때만 `vadimcn.vscode-lldb` 확장과 필요한 권한(`capAdd`/`securityOpt`)을 추가한다. 기본 구성에는 권한을 완화하는 옵션을 넣지 않았다.
- 이미지 버전, Rust 버전, 크레이트 버전을 기록한다. 정말 동일한 이미지 재현이 필요하면 digest 고정을 검토한다.
- 컨테이너 데이터베이스는 볼륨 백업이 필요하다. `docker volume rm` 및 공장 초기화는 데이터 복구를 어렵게 한다.
- **Bind Mount는 삭제 방지 장치가 아니다.** 컨테이너 내부 프로그램이 마운트된 소스 파일을 수정·삭제하면 Mac 원본에도 반영된다. Git 백업을 유지한다.

### 6.6 `Cargo.toml`과 `Cargo.lock`을 함께 이해하기

- `Cargo.toml`: 프로젝트가 **어떤 의존성을 허용하는지** 선언한다.
- `Cargo.lock`: 실제로 해석된 **정확한 의존성 버전**을 기록한다.
- 실행 프로그램은 대개 `Cargo.lock`을 Git에 포함한다. CI에서는 `cargo build --locked`로 잠금 파일과의 일치 여부를 보장할 수 있다.
- 툴체인까지 재현하려면 `rust-toolchain.toml`을 고정 버전으로 설정하고 Docker 이미지 digest도 관리한다.

### 6.7 `target/`의 저장 위치와 디스크 사용량

- Rust 빌드 결과는 기본적으로 프로젝트의 `target/`에 쌓인다.
- 이 구성은 소스는 Mac 폴더를 Bind Mount하지만, **`target/`과 Cargo 레지스트리 캐시는 Docker named volume**(3.2의 `mounts`)에 저장한다. macOS Bind Mount의 대량 파일 I/O 성능 저하를 피하기 위해서다.
- 따라서 **빌드 결과는 Mac 파일 탐색기에서 보이지 않는다.** 컨테이너 터미널에서 `target/` 경로로 접근하며, Mac에서는 `docker system df -v`로 볼륨 크기를 본다.
- 용량이 늘면 컨테이너에서 `cargo clean`으로 정리한다. **주의:** 이 명령은 빌드 결과를 삭제하므로 다음 빌드 시간이 길어진다.
- 볼륨을 삭제하면(8장) 빌드 캐시와 다운로드한 크레이트가 사라져 다음 빌드가 오래 걸린다. 소스코드에는 영향이 없다.

### 6.8 Linux와 macOS 빌드 대상이 다르다

- Dev Container에서 기본적으로 만든 실행 파일은 Linux용이다.
- Mac에서 네이티브 실행할 macOS 실행 파일이 필요하다면 별도의 크로스 컴파일 환경 또는 macOS 빌드가 필요하다. Docker에서 빌드했다는 이유만으로 macOS 바이너리가 되는 것은 아니다.

### 6.9 실행·배포용 Docker 이미지는 별도로 만든다

이 문서의 `Dockerfile`은 **개발 컨테이너용**이다. 서버 배포 등 실행용 이미지가 필요해지면 별도의 멀티스테이지 파일(예: `Dockerfile.release`)을 추가한다. 개발 이미지를 그대로 배포하지 않는다.

```dockerfile
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release --locked

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/rust-docker /usr/local/bin/app
CMD ["app"]
```

- `--locked`: `Cargo.lock`과 일치하지 않으면 빌드를 실패시켜 재현성을 보장한다.
- 의존성이 커지면 `cargo-chef`나 BuildKit 캐시 마운트로 의존성 레이어를 캐싱한다.
- 배포 빌드용 `[profile.release]`(`lto`, `codegen-units = 1`, `strip = true`)는 이때 `Cargo.toml`에 추가한다.
- 이미지 안의 바이너리 이름은 `Cargo.toml`의 `name`과 같아야 한다.

### 6.10 CI에서 같은 검사를 자동화

로컬에서 쓰는 검사 명령을 CI(GitHub Actions 등)에서도 동일하게 실행하면 품질이 일관된다.

```bash
cargo fmt --all -- --check
cargo clippy --all-targets --locked -- -D warnings
cargo test --locked
```

**기본 권장 개발 순서:** 코드 작성 → `cargo fmt` → `cargo clippy` → `cargo test` → 커밋.

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

프로젝트 빌드 결과만 지우려면 **컨테이너 터미널**에서:

```bash
cargo clean
```

사용하지 않는 빌드 캐시만 정리하는 예 (Mac 터미널):

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
- `Dockerfile`로 빌드한 개발 이미지는 보통 `vsc-<프로젝트명>-<해시>` 형태의 이름으로 로컬에 생성된다. `docker images`에서 확인하고, 베이스 이미지(`rust:1-bookworm`)는 다른 프로젝트가 쓸 수 있으니 함께 지우지 않도록 주의한다.
- 이 프로젝트는 `target/`과 Cargo 레지스트리 캐시용 볼륨(이름이 `...-target`, `...-cargo-registry`로 끝남)을 만든다. 컨테이너를 삭제한 뒤에도 볼륨은 남으므로, 프로젝트를 완전히 지울 때는 **컨테이너를 먼저 삭제한 뒤** 이 두 볼륨을 확인하고 삭제한다. 사용 중인 볼륨은 삭제되지 않는다.
- Dev Containers 확장 관련 공유 볼륨도 있을 수 있다.
- Mac의 `rust-docker` 폴더는 위 Docker 명령으로 삭제되지 않는다.
- 로컬 프로젝트 폴더 자체를 제거하려면 별도의 파일 삭제가 필요하지만, 중요한 코드라면 우선 Git 또는 백업으로 보관한다.

## 9. Docker 전체 정리 및 공장 초기화

> ⚠️ **파괴적 작업**: 아래 전체 삭제는 현재 Rust 프로젝트뿐 아니라 **다른 모든 Docker 프로젝트**에 영향을 줄 수 있다. DB 볼륨, 인증 상태, 컨테이너 데이터가 영구 삭제될 수 있다. 백업 확인 전에는 실행하지 않는다.

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

`docker volume prune -a`는 Docker 버전에 따라 지원 여부를 확인해야 한다. 지원하지 않는 버전에서는 `docker volume prune --help`를 확인한다. **컨테이너를 전부 삭제했다면 볼륨은 대부분 더 이상 참조되지 않으므로 데이터 유실 위험이 매우 크다.** (이 프로젝트의 `target/`·Cargo 캐시 볼륨도 삭제된다.)

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
| 현재 Rust 프로젝트만 제거 | [8장 프로젝트 개발환경 삭제](#8-프로젝트-개발환경-삭제) |
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
| `cargo: command not found` | 로컬 터미널인지 확인. Dev Container 안에서 실행할 것 |
| `cargo clippy` / `cargo fmt`를 찾지 못함 | 컨테이너에서 `rustup show` 실행(`rust-toolchain.toml`의 components 설치). 필요하면 `rustup component add clippy rustfmt` |
| 컨테이너 안에서 파일 쓰기 권한 오류 | non-root 사용자(`vscode`, UID 1000)로 실행 중이다. `target/`·`registry` 볼륨은 `postCreateCommand`의 `chown`이 실행됐는지 확인하거나 `sudo chown -R vscode: <경로>`. 시스템 패키지 설치는 `Dockerfile`에 추가한다 |
| `rustup`이 toolchain 설치 중 Permission denied | `Dockerfile`의 `chown -R ... /usr/local/cargo /usr/local/rustup`이 적용됐는지 확인 후 Rebuild |
| 확장이 예상과 다르게 설치됨 | 베이스 이미지 메타데이터가 확장을 추가할 수 있다. `rust:1-bookworm`은 없지만 이미지를 바꿨다면 `docker image inspect <이미지> --format '{{index .Config.Labels "devcontainer.metadata"}}'`로 확인 |
| rust-analyzer가 분석하지 못함 | 컨테이너 안에 확장이 설치됐는지, `Cargo.toml`이 워크스페이스 루트에 있는지, rust-analyzer 출력 패널 확인 |
| 패키지 빌드 시 OpenSSL 등 시스템 의존성 오류 | 필요한 Linux 패키지(`pkg-config`, `libssl-dev` 등)를 `Dockerfile`에 추가하고 Rebuild |
| 빌드가 느림 | 첫 다운로드/컴파일은 정상적으로 느릴 수 있음. `target/`이 named volume으로 분리되어 있는지(`mounts`) 확인 |
| Rebuild 후 도구가 사라짐 | 수동 설치(`cargo install` 등) 대신 `Dockerfile`/`postCreateCommand`에 기록 |
| 이미지 빌드 실패 | **Dev Containers: Show Container Log**에서 `Dockerfile` 오류 확인. 캐시 문제가 의심되면 **Rebuild Without Cache** |
| Mac 브라우저에서 웹 서버 접근 불가 | `forwardPorts`, 서버 바인딩 주소(`0.0.0.0`), VS Code Ports 패널 확인 |
| 디스크 용량 부족 | 먼저 `docker system df -v` 확인 후 `cargo clean` 또는 필요한 자원만 정리 |

### 10.2 다음에 처음부터 다시 구축할 때

- [ ] Docker Desktop을 설치하고 실행한다.
- [ ] VS Code와 **로컬 Dev Containers 확장**을 설치한다.
- [ ] Mac에 기존 프로젝트 파일이 있다면 폴더를 VS Code로 연다.
- [ ] 프로젝트에 `.devcontainer/devcontainer.json`, `Dockerfile`, `Cargo.toml`, `rust-toolchain.toml`, `src/main.rs`가 있는지 확인한다.
- [ ] VS Code 명령 팔레트 → **Dev Containers: Reopen in Container**.
- [ ] 컨테이너 터미널에서 `rustc --version`, `cargo --version`, `cargo clippy --version`을 확인한다.
- [ ] `cargo run`, `cargo check`, `cargo fmt --all -- --check`, `cargo test`를 실행한다.
- [ ] 개발 중 새 패키지가 필요하면 `cargo add <crate>`를 실행하고, 업데이트는 `cargo update`를 사용한다.
- [ ] Git에 `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml`, `Dockerfile`, Dev Container 설정을 보관한다.

### 10.3 전체 삭제 전 필수 체크리스트

- [ ] 프로젝트 소스코드가 Mac에 남아 있고 Git/백업이 준비되었다.
- [ ] `.devcontainer`, `Dockerfile`, `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml` 파일을 보관했다.
- [ ] Docker 볼륨에만 있는 데이터(DB 등)를 별도 백업했다. (`target/` 볼륨은 재생성 가능하므로 백업 불필요)
- [ ] 다른 프로젝트의 컨테이너·이미지·볼륨까지 삭제되는지 확인했다.
- [ ] 프로젝트만 지울지, 자원 전체를 지울지, Desktop 공장 초기화까지 할지 결정했다.

---

## 공식 문서

- [VS Code: Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)
- [Dev Container specification](https://containers.dev/implementors/json_reference/)
- [VS Code: Rust 언어 지원](https://code.visualstudio.com/docs/languages/rust)
- [rust-analyzer: VS Code](https://rust-analyzer.github.io/book/vs_code.html)
- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [Cargo Book](https://doc.rust-lang.org/cargo/)
- [Docker 공식 Rust 이미지](https://hub.docker.com/_/rust)
- [Docker: docker system prune](https://docs.docker.com/reference/cli/system/prune/)
- [Docker: docker volume prune](https://docs.docker.com/reference/cli/docker/volume/prune/)
- [Docker Desktop troubleshooting](https://docs.docker.com/desktop/troubleshoot/)

> 이 문서는 **로컬 Rust 개발 도구를 설치하지 않는 Dev Containers 중심 구성**을 기준으로 한다. Docker, VS Code 및 확장 기능은 업데이트되므로, 명령/설정이 달라졌다면 공식 문서를 함께 확인한다.
