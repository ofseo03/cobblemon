# Cobblemon 서버 직접 호스팅하기

내가 없어도 친구들이 웹사이트에서 버튼 하나로 서버를 켜고 Cobblemon을 할 수 있게 만드는 저장소입니다.

서버 관리 웹사이트는 오픈소스 게임 서버 패널인 [Pterodactyl](https://github.com/pterodactyl/panel)을 그대로 쓰고,
이 저장소에는 Pterodactyl에 넣을 **Cobblemon 전용 서버 템플릿(Egg)** 과 설치 가이드가 들어 있습니다.

| 파일 | 설명 |
|------|------|
| [`eggs/egg-cobblemon.json`](eggs/egg-cobblemon.json) | Pterodactyl에 가져오는 Cobblemon 서버 템플릿 |

## 전체 구조

```
친구 브라우저 ──▶ Panel (웹사이트, 80/443) ──▶ Wings (8080) ──▶ Docker 컨테이너: Cobblemon 서버
       └───── 콘솔 화면은 브라우저가 Wings(8080)에 직접 연결 ─────┘
친구 마인크래프트 ──────────────────────────────────────────────▶ 25565
```

- **Panel**: 로그인, 서버 켜기/끄기, 콘솔, 파일 관리를 하는 웹사이트
- **Wings**: Panel의 명령을 받아 Docker 컨테이너로 서버를 실제로 실행하는 프로그램
- 두 프로그램을 **24시간 켜져 있는 컴퓨터 한 대**에 같이 설치합니다. 이 컴퓨터가 꺼지면 아무도 서버를 켤 수 없습니다.

## 1. 준비물

- **Linux 컴퓨터** (Ubuntu 22.04 또는 24.04, 64비트 권장)
  - 집 PC, 남는 노트북, VPS 모두 가능합니다. Windows는 Pterodactyl이 공식 지원하지 않습니다.
  - Docker가 돌아가야 하므로 OpenVZ, LXC 방식의 VPS는 쓸 수 없습니다 (KVM 방식은 가능).
- **RAM 8GB 이상 권장**: Cobblemon 서버 4~6GB + 패널과 시스템 2GB 정도
- **도메인 (권장)**: 패널 주소와 HTTPS 인증서용. 무료 DDNS 도메인도 됩니다.
- **열어야 하는 포트** (집이라면 공유기 포트포워딩, VPS라면 방화벽)

  | 포트 | 용도 |
  |------|------|
  | 80, 443 | 패널 웹사이트 |
  | 8080 | Wings (브라우저 콘솔 연결) |
  | 2022 | SFTP 파일 업로드 (선택) |
  | 25565 | 마인크래프트 접속 |

## 2. Pterodactyl 설치

공식 문서 순서대로 **Panel → Wings** 를 설치합니다.

1. Panel: <https://pterodactyl.io/panel/1.0/getting_started.html>
2. Wings: <https://pterodactyl.io/wings/1.0/installing.html>

참고:
- 패널을 HTTPS로 열었다면 Wings에도 같은 도메인의 SSL 인증서를 설정해야 합니다. 안 그러면 브라우저에서 콘솔이 연결되지 않습니다.
- 명령어 몇 줄로 끝내 주는 비공식 자동 설치 스크립트([pterodactyl-installer](https://github.com/pterodactyl-installer/pterodactyl-installer))도 있습니다. 편하지만 공식이 아니니 내용을 확인하고 쓰세요.

## 3. 노드와 포트 등록 (패널 관리자 화면)

1. **Admin → Locations** 에서 위치를 하나 만듭니다 (예: `home`).
2. **Admin → Nodes → Create New** 에서 노드를 만듭니다.
   - FQDN에는 이 컴퓨터의 도메인(또는 IP)을 적습니다.
   - 만든 뒤 **Configuration** 탭의 내용을 Wings의 `/etc/pterodactyl/config.yml`에 붙여 넣고 Wings를 다시 시작합니다.
3. 노드의 **Allocation** 탭에서 이 컴퓨터의 IP와 포트 `25565`를 추가합니다.

## 4. Cobblemon Egg 가져오기

1. 이 저장소의 [`eggs/egg-cobblemon.json`](eggs/egg-cobblemon.json)을 내려받습니다 (GitHub에서 파일을 열고 **Download raw file**).
2. **Admin → Nests** 에서 **Import Egg** 를 누르고 파일과 넣을 Nest(예: Minecraft)를 고릅니다.

Egg가 하는 일:
- Modrinth에서 Cobblemon과 필수 의존 모드(Fabric API, Fabric Language Kotlin 등)를 받고 sha512 해시를 확인합니다.
- Cobblemon 버전에 맞는 마인크래프트 버전으로 Fabric 서버를 설치합니다.
- 버전을 바꿔 재설치하면 이전에 이 Egg가 설치한 모드만 지웁니다. 월드와 직접 넣은 모드는 그대로 둡니다.

| 변수 | 기본값 | 설명 |
|------|--------|------|
| 마인크래프트 버전 | `latest` | `latest`면 최신 Cobblemon 정식 버전이 지원하는 버전 |
| Cobblemon 버전 | `latest` | 예: `1.6.1+1.21.1` (Modrinth 버전 번호 또는 버전 ID) |
| Fabric Loader 버전 | `latest` | 최신 안정 버전 |
| 추가 모드 | (비움) | 같이 설치할 Modrinth 모드, 쉼표로 구분 (예: `journeymap,jei`) |
| 서버 Jar 파일 | `server.jar` | 보통 바꿀 필요 없음 |

## 5. 서버 만들기

**Admin → Servers → Create New**

- **Server Owner**: 나
- **Nest / Egg**: Cobblemon (Fabric)
- **Docker Image**: `Java 21` (설치 로그 마지막 줄에 권장 Java 버전이 나옵니다. 마인크래프트 26.x 이상이면 `Java 25`)
- **Memory**: 6144MB 이상 권장, **Disk**: 10GB 이상
- **Default Allocation**: 3번에서 만든 `25565`
- **Backup Limit**: 기본값이 0이라 백업을 못 만듭니다. 3 정도로 올려 두세요.

만들면 설치가 자동으로 시작됩니다. 서버 페이지의 콘솔에서 설치 로그를 볼 수 있고,
처음 켤 때 마인크래프트 EULA 동의 창이 뜨면 동의합니다.

## 6. 친구가 혼자 서버를 켤 수 있게 하기

Pterodactyl에는 회원가입 기능이 없어서 관리자(나)가 친구 계정을 만들어 줍니다.

1. **Admin → Users → Create New** 에서 친구 계정을 만들고 비밀번호를 정해 줍니다.
   - 메일 발송(SMTP)을 설정해 두었다면 이 단계 없이 2번에서 이메일만 넣어도 계정이 만들어지고 비밀번호 설정 메일이 갑니다.
2. 내 서버 페이지의 **Users** 탭 → **New User** 에서 친구 이메일을 넣고 권한을 고릅니다.

   | 권한 | 하는 일 |
   |------|---------|
   | Control → Console | 콘솔 보기, 명령어 입력 |
   | Control → Start | 서버 켜기 |
   | Control → Stop | 서버 끄기 |
   | Control → Restart | 서버 재시작 |

3. 친구는 패널 주소로 로그인해서 서버를 고르고 **Start** 를 누르면 됩니다. 접속 주소는 `도메인:25565` 입니다.

서버 페이지의 **Schedules** 탭에서 매일 자동 재시작이나 백업도 예약할 수 있습니다.

## 7. 친구 컴퓨터(클라이언트) 준비

서버와 **같은 마인크래프트 버전, 같은 Cobblemon 버전**이 필요합니다.
설치 로그 마지막에 나오는 버전과 모드 목록(서버 폴더의 `cobblemon-egg-mods.txt`)을 친구들에게 알려 주세요.

- [Modrinth App](https://modrinth.com/app)에서 Fabric 프로필을 같은 마인크래프트 버전으로 만들고 Cobblemon, Fabric API, Fabric Language Kotlin을 설치하는 게 가장 쉽습니다.

## 버전 올리기

1. 서버 페이지 **Startup** 탭에서 변수(예: Cobblemon 버전)를 바꿉니다.
2. **Settings → Reinstall Server** 를 누릅니다.

재설치 전에 **Backups** 탭에서 백업을 만들어 두는 것을 권장합니다. 마인크래프트 버전을 올리면 월드가 변환되어 되돌릴 수 없습니다.

## 문제 해결

| 증상 | 해결 |
|------|------|
| 설치 로그에 `조건에 맞는 Cobblemon 버전이 없습니다` | 마인크래프트 버전과 Cobblemon 버전 조합을 Modrinth에서 확인하거나 둘 다 `latest`로 |
| 서버가 `UnsupportedClassVersionError`로 꺼짐 | Java 버전이 낮습니다. 뜨는 창이나 **Startup** 탭에서 Docker Image를 바꾸세요 |
| 브라우저에서 콘솔이 안 열림 | 8080 포트가 열려 있는지, 패널이 HTTPS라면 Wings에도 SSL이 설정됐는지 확인 |
| 친구가 접속 못 함 | 25565 포트포워딩, 클라이언트 모드 버전 확인 |
| 메모리 부족으로 꺼짐 | 서버의 Memory를 늘리세요 (모드 서버는 4GB 이상 권장) |
