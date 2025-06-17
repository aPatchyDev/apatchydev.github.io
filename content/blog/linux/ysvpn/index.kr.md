+++
date = '2025-03-16T11:33:49Z'
linktitle = 'YSVPN'
title = '리눅스에서 YSVPN 설치하기'
weight = 100
+++

YSVPN은 연세대학교에서 사용하는 VPN 프로그램으로 외부에서 교내 서버로 접속할 때 반드시 필요하다. 이전에는 허가받은 경우 외부IP를 사용할 수 있었지만 2022년도 말부터 VPN을 사용하는 것으로 전환된 것으로 기억한다.

일부 전공수업의 경우, 편의 또는 채점을 위해 서버를 제공하는데 VPN 없이는 집에서 작업할 수가 없다. 그러나 학교에서는 윈도우와 맥 버전만 제공한다.

네이티브로 리눅스를 돌리면서 솔루션을 찾는 소수의 동문을 위해 해결책을 먼저 제시하고 디버깅 과정은 그 뒤에 자세히 설명한다.

## 해결책

1. 카이스트 전산학부에서 [리눅스용 SecuwaySSL U V2.*](https://kcloud.kaist.ac.kr/index.php/pds/?mod=document&uid=225)을 다운받는다.
2. `SSU20-CFL-V2.*.tgz` 압축파일을 해제한다. 이 압축파일의 내용물만 필요하므로 다른 파일들은 삭제해도 된다.
3. `conf/client.info`를 다음과 같이 수정한다.

```text
userid: <YSVPN 아이디>
userpw: <YSVPN 비번>
vpn_ip: <YSVPN 웹페이지의 IPv4 주소>
vpn_port: 443
crypto: yes
log_size: 50
site-to-site: no
```

4. `SecuwaySSLU_client`를 루트 권한으로 실행시킨다
5. 연결을 해제할 때는 `sslvpn` 프로세스를 `kill` 명령어로 종료한다. 이때도 루트 권한이 필요하다

이때 `conf/client.info` 파일의 `userpw` 항목이 자동으로 삭제된다. 따라서 접속하기 전 매번 설정 파일에 비밀번호를 다시 입력해야 한다.

그리고 연결이 된 상태에서 다시 연결을 시도하면 기존 연결까지 끊어진다.

### 자동화

연결이 된 상태에서 재접속을 방지하고 비밀번호가 초기화되는 상황을 막아 사용성을 높이는 스크립트다. `SecuwaySSLU_client` 실행파일과 같은 경로에 놓고 `conf/client.info -> conf/client.info.cfg`로 변경한 뒤에 사용하면 된다.

```bash
#!/bin/bash

help() {
    >&2 echo "Usage: `basename $0` [on/off]"
}

if [ $# -ne 1 ]; then
    help
    exit 1
fi

# Modifying VPN tunnel requires root privilege
SUDO=''
if [ `id -u` -ne 0 ]; then
    SUDO='sudo'
fi

CHK=`ifconfig -s | grep tun`
CWD=`dirname $(realpath $0)`
case "$1" in
    "on")
        if [ -z "$CHK" ]; then
            cp "$CWD/conf/client.info.cfg" "$CWD/conf/client.info"
            $SUDO "$CWD/SecuwaySSLU_client"
        else
            >&2 echo 'VPN already connected'
        fi
        ;;
    "off")
        if [ -z "$CHK" ]; then
            >&2 echo 'VPN not connected'
        else
            $SUDO kill `pgrep -f sslvpn`
            if [ $? -eq 0 ]; then
                >&2 echo 'Exit successful'
            else
                >&2 echo 'Exit unsuccessful'
                exit 1
            fi
        fi

        ;;
    *)
        help
        exit 1
        ;;
esac
```

## 분석 과정

YSVPN에 접속하면 설치하라고 안내하는 프로그램이 `SecuwaySSL`이다. 이와 관련된 분석글을 찾아보니 카이스트에서도 같은 소프트웨어를 사용하고 있는 것을 알게 되었고, [맥 버전을 분석한 글](https://velog.io/@predict-woo/Secuway-SSL-VPN-%EB%B6%84%EC%84%9D-1)을 발견했다.

### 윈도우 기초 분석

맥 버전을 분석한 글이 있음에도 윈도우 버전을 분석한 몇 가지 이유가 있다.

1. 맥 버전의 번들된 앱을 리눅스 환경에서 언패킹하는 것이 쉽지 않았다
2. 와인으로 돌아갈 가능성을 확인하고 싶다
3. 리버싱을 한다면 x64가 더 익숙하기 때문에

윈도우 버전이 활용하는 경로는 2가지가 있다.

- C:\Program Files (x86)\SecuwaySSLU\
- %homepath%\AppData\LocalLow\SecuwaySSLU\

설명의 편의상 첫 번째를 `설치 경로`, 두 번째를 `사용자 경로`라고 부르겠다.

### 왜 와인을 쓸 수 없는가

와인은 아직 장치 드라이버를 수정하는 윈도우 소프트웨어를 지원하지 않는데 `설치경로\driver`, `설치경로\driver64`가 있는 것으로 보아 네트워크 드라이버를 수정하는 것으로 보인다. 실제로 와인으로 설치하려고 할 때에도 많은 에러 메시지가 떴다.

### 설치파일 조사

우선 설치경로와 사용자 경로에 있는 파일을 탐색했다.

VPN이 실행되지 않는 상태에서는 `사용자 경로`에는 로고 이미지 파일과 로그만 있어 볼 것이 없었다. ~~설마 이미지 파일 내 stegonography?~~

`설치 경로`를 뒤저본 결과, 몇 가지 흥미로운 파일들을 발견했다.

- secuwiz-cert.pem
- secuwiz-key.pem
- ca.cer
- cert8.db
- key3.db
- secmod.db


`.pem / .cert` 파일은 비대칭 암호키일 것이 자명하니 `.db` 파일을 분석해 어떤 데이터베이스 형식을 사용하는지 알아보려고 했다.

해당 파일을 [여기](https://www.checkfiletype.com/)와 같은 온라인 파일 타입 검사 사이트에 넣으면 `Berkeley DB`가 나오는데 해당 DB 프로그램으로 파일을 열려고 하면 DB 파일에 문제가 있어 열 수 없다고 한다. [조사를 하다보니](https://serverfault.com/questions/811680/how-can-i-browse-the-contents-of-a-berkeley-db-file) 파일 형식을 잘못 알려주는 경우가 있다는 것을 알게 되었다.

이 뒤에는 SQLCipher 형식 등 몇 가지 시도를 하다가 진척이 없어 다른 방법으로 눈을 돌렸다.

#### OpenVPN 설정 파일 탈취

맥 버전의 분석글에서 `SecuwaySSL`은 `OpenVPN`을 가공한 프로그램임을 알 수 있었다. 그렇다면 VPN 연결에 필요한 정보가 `.ovpn` 파일에 저장이 될 텐데, 이것을 얻을 수만 있다면 리눅스에서 OpenVPN Client로 접속할 수 있을 것이다.

그러나 `설치 경로 / 사용자 경로` 모두 `.ovpn` 파일은 찾을 수 없었다. 파일을 생성했다가 삭제하는지, 레지스트리에 저장하는지 알 수 없었기에 동적으로 분석해보기로 했다. [MS ProcessMonitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon)를 사용하면 원하는 프로세스가 수행하는 파일 입출력, 레지스트리 접근을 포함해 다양한 리소스 사용 패턴을 알 수 있다.

`ProcessMonitor`로 `SecuwayVPN` 프로세스를 감시하니 `사용자 경루\config\` 폴더에 임시 파일을 생성하는 것을 볼 수 있었다. 그러나 실제로 해당 경로에 가보면 아무 흔적도 없었다. 가장 처음 시도한 것은 데이터 복구 프로그램을 사용해보는 것이었다.

- [EaseUS Data Recovery](https://www.easeus.com/datarecoverywizard/free-data-recovery-software.htm): 파일 흔적은 찾았지만 파일 데이터가 소실된 부분이 많아 복구가 안 됐다
- [Recuva](https://www.ccleaner.com/recuva): 좀 더 잘 복구가 됐으나 정말 완벽하게 복구된 것인지는 알 수 없었다

그래서 단순하게 무한 루프로 해당 경로의 모든 파일을 바탕화면으로 복사하는 `Powershell script`를 만들었다. 스크립트를 실행한 다음에 VPN을 실행해 아래의 파일들을 추출할 수 있었다

- ca.cert
- client.crt
- client.key
- sslvpn.ovpn (연결을 끊기 전까지 계속 남아있다)

`sslvpn.ovpn` 파일을 추출하고, OpenVPN에서 LEA 암호화 알고리즘도 쓰도록 변경했지만 암호키 파일에 암호가 걸려있었다. 이 키 파일 암호는 어딘가에 하드코딩이 되어있을 텐데 그게 바이너리 안에 난독화된 상수로 존재하는지, 복호화를 하지 못한 `key3.db` 파일에 있는지는 잘 모르겠다.

여기에서 막힌 뒤 카이스트의 리눅스용 VPN 메뉴얼을 발견했기 때문에 분석은 중단했다. `ovpn` 파일과 인증키 파일을 사용하려면 인증키 비밀번호를 알아야 해서 `bruteforcing / 리버싱`이 필요해 보인다.
