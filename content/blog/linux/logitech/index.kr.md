+++
date = '2025-01-02T07:04:18Z'
title = 'Logitech 마우스 설정'
weight = 100
+++

## 환경

- OS: Fedora 41 KDE
- 마우스: Logitech MX Master 2S

## 마우스 세부설정 소프트웨어

윈도우와 맥에서는 공식 `Logitech Options` 소프트웨어를 사용하여 마우스 설정을 할 수 있지만 리눅스는 지원하지 않는다.

대신 `Solaar`라는 소프트웨어를 사용하면 마우스 설정을 할 수 있다.

`sudo dnf install solaar`로 설치하고 USB receiver를 뺐다 다시 꽂으면 된다.

## 마우스 버튼 고급 기능

윈도우에서는 `뒤로 가기` 버튼은 `Ctrl + 좌클릭`, `앞으로 가기` 버튼은 `창 최소화`로 사용해왔던 만큼 동일하게 세팅을 하고 싶었다.

`solaar` 기본 선택지에는 이런 고급 기능이 없으므로 직접 매크로를 만들고 이를 호출하게 만들어야 한다. 마우스 버튼 기능을 `diverted`로 설정하면 해당 버튼에 대한 처리를 매크로에 맡기게 된다.

### Ctrl Click 모방하기

`Logitech Options`에서 `Ctrl click`은 버튼이 눌리는 동안 계속 `Control` 키가 눌린 것으로 인식한다. 웹 브라우징 중 `새 탭 열기` 뿐만 아니라 화면 줌도 한 손으로 조절할 수 있게 해줘 편하다.

1. `Key/Button Diversion`에서 `Back Button / Diverted` 선택

![solaar config](./solaar_config.avif)

2. 오른쪽 하단의 `Rule Editor` 클릭

3. `User-defined rules` 우클릭 > `Insert new rule`

![solaar rule editor](./solaar_rules.avif)

4. 새 `Rule` 아래의 `[empty]` 우클릭 > `Insert here` > `Condition` > `Key`

5. 화면 하단에 `Back Button` 입력 (대소문자 맞게) > `Key down` 선택

6. `Rule` 아래의 `Key` 우클릭 > `Insert below` > `Action` > `Key press`

7. 화면 하단에 `Control_L` 입력 > `Depress` 선택

8. `Rule` 아래의 `Key press` 우클릭 > `Insert below` > `Action` > `Mouse click`

9. `Button = left`, `Count and Action = 1, click` 기본값 그대로 유지

10. #3 ~ #9번을 반복하되 #5번은 `Key down -> Key up`, #7번은 `Depress -> Release`로 변경

11. 화면 상단에 `Save changes` 버튼으로 저장

### 창 최소화 모방하기

`Logitech Options`에서는 창 최소화가 선택할 수 있는 action 중 하나여서 간단했지만 `solaar`에서는 이 기능을 지원하지 않는다.

그래서 그 대안으로 본인의 `Desktop Environment / Window Manager`가 지원하는 키보드 단축키를 실행하게 만들어야 한다.

본인은 `KDE`를 사용하고 있으므로 아래와 같이 `Ctrl + M` 단축키를 추가했다.

![kde minimize shortcut settings](./kde_minimize.avif)

그 다음에는 `solaar`에서 단축키를 실행하도록 `rule`을 생성하면 된다.

1. `Key/Button Diversion`에서 `Back Button / Diverted` 선택

2. `Rule Editor` > `User-defined rules` 우클릭 > `Insert new rule`

3. 새 `Rule` 아래의 `[empty]` 우클릭 > `Insert here` > `Condition` > `Key`

4. 화면 하단에 `Forward Button` 입력 (대소문자 맞게) > `Key down` 선택

5. `Rule` 아래의 `Key` 우클릭 > `Insert below` > `Action` > `Key press`

6. 화면 하단에 `Add key` 버튼 클릭 > 왼쪽은 `Control_L`, 오른쪽은 `m` > `Depress` 선택

7. 화면 상단에 `Save changes` 버튼으로 저장

### 완료 후 모습

![solaar rule editor complete](./solaar_rules_done.avif)

## 스크롤 속도

마우스 사용 중 스크롤 속도가 일정하지 않은 것을 발견했다. 문서나 페이지를 스크롤할 때는 별로 신경 쓰이지는 않지만 화면 줌이나 유튜브 볼륨 조절을 스크롤휠로 할 때 한 칸 아래로 내렸다가 다시 올려도 원복이 안 되는 것이 싫었다. 줌은 휠 한 칸에 10%씩, 볼륨은 5%씩 일정하게 움직이기를 바랐고 많은 검색 끝에 방법을 찾았다.

1. `solaar`에서 `Scroll Wheel Resolution`을 비활성화한다

2. `hid_logitech_hidpp` 커널 모듈을 비활성화한다

- 이후 실행할 명령어는 `root` 권한이 필요하므로 `sudo -i`로 root shell을 띄운다
- 현재 시스템에서 모듈을 제거하기 위해 `modprobe -r hid_logitech_hidpp`를 실행한다
- 재부팅 후에도 영구적으로 제거하기 위해 `echo "blacklist hid_logitech_hidpp" >> /etc/modprobe.d/logitech-blacklist.conf`를 실행한다
- 기존의 `initramfs`에서 모듈이 이미 로딩되므로 모듈이 제거된 `initramfs`를 생성하기 위해 `dnf reinstall kernel-core`를 실행한다

`lsmod | grep logitech`에서 `hid_logitech_hidpp`가 없다면 커널 모듈이 성공적으로 제거된 것이다. 이제 스크롤 속도는 일정하게 되었을 것이다.
