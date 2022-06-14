# webserv

### checklist
---
- (select) timeout은 select에 의해 변하지 않고, 후속 호출에서 재사용 가능하지만, 매 select마다 초기화 하는 것이 더 좋다
- (poll error) 내부 데이터 구조체 할당 실패, 후속 요청은 성공할 수 있음
- kevent

### 허용 함수
---

#### htonl, htons, ntohl, ntohs: 16, 32 bit quantities를 network byte order와 host byte order간 변환
```c++
uint32_t htonl(uint32_t hostlong)
uint16_t htons(uint16_t hosthshort)
uint32_t ntohl(uint32_t netlong)
uint16_t ntohs(uint16_t netshrot)
```

- network byte order는 big endian(MSB)
- network byte order와 머신이 같은 byte order일 경우, null macro로 루틴이 정의되어 있다

+ 이 루틴들은 gethostbyname과 getservent로 리턴되는 인터넷 주소와 포트와 함께 사용된다

---
#### int select(int nfds, fd_set *restrict readfds, writefds, errorfds, struct timeval *restirct timeout)<br/>: 아래 함수들로 리턴되는 I/O descriptor를 테스트 한다
- readfds: 읽기가 가능한 fd인지
- writefds: 쓰기가 가능한 fd인지
- errorfds: 예외 처리가 필요한 fd인지

+ descriptor sets는 int형 배열들에 bit field(비트 플래그 확인)로 저장된다 (주: 구현에 따라 다를 수 있음?)
+ 아래 매크로들은 그렇게 저장된 descriptor들을 조작하는 것에 사용된다
  - FD_ZERO(&fdset): fsdet을 NULL로 초기화
  - FD_SET(fd, &fdset): fdset에 있는 fd를 include한다
  - FD_ISSET(fd, &fdset): fd가 fdset에 있으면 0이 아닌 수, 없으면 0
  - FD_COPY(&fdset_orig, &fdset_copy): 이미 할당되어 있는 fdset_copy를 fdset_orig의 사본으로 대체한다
+ 이런 매크로들에 fd가 0보다 작거나, FD_SETSIZE(보통 시스템에서 지원하는 descriptor 숫자와 적어도 같은)보다<br/>크거나 같을 때의 동작은 undefined이다

- timeout이 non-nil pointer일 경우, selection이 완료되기 까지의 최대 interval을 나타낸다
- nil pointer인 경우, select는 무기한으로 block한다
- poll처럼 사용하기 위해, timeout은 zero-valued timeval structure를 가르키는 non-nil이어야 한다
- timeout은 select에 의해 변하지 않고, 후속 호출에서 재사용 가능하지만, 매 select마다 초기화 하는 것이 더 좋다
- 다루지 않으려는 readfds, writefds, errorfds는 nil pointer로 주어져야 한다

#### return
- descriptor sets에 준비된 descriptor들의 수를 반환, 에러 발생 시 -1
- time limite이 만료된 경우, 0을 리턴
- 에러를 반환하는 경우(interrupted call 포함), descriptor sets는 변하지 않고, errno가 설정 된다

#### error
- 커널이 요청된 숫자의 fd를 할당할 수 없음
- descriptor sets 중 하나가 invalid함
- time limit이 지나지 않았으면서 이벤트가 발생하기 전에 시그널을 받았을 때
- time limit이 invalid일 때, 음수거나 너무 큰 경우
- ndfs가 FD_SETSIZE보다 크고, _DARWIN_UNLIMITED_SELECT가 정의되지 않았을 때
---

#### poll<br/>: fd set이 I/O 가능한지, 혹은 특정 이벤트가 발생했는지 검사한다.

```c++
int poll(struct pollfd fds[], ndfs_t nfds, int timeout);
```

```c++
struct pollfd {
  int fd; // fd
  short events; // events to look for (poll 하려는 이벤트)
  short revents; // events returned (발생할 예정이거나 발생한 이벤트)
}
```
- events와 revents에는 아래 bitmask의 bit가 있다 (normal, priority, high-priority가 있음)
  - POLLERR: 디바이스나 소켓에 예외가 발생함. 이 플래그는 출력 전용이고, input events bitmask에 있을 경우 무시된다.
  - POLLHUP: 디바이스나 소켓의 연결이 끊어짐. 출력 전용이고, input에 있을 경우 무시된다.<br/>POLLHUP와 POLLOUT은 상호 배제적이고, 같은 revents bitmask에 있으면 안된다
  - POLLIN: 높은 우선 순위에 없는 데이터는 blocking없이 읽힌다. == (POLLRDNORM | POLLRDBAND)
  - POLLNVAL: fd가 열려있지 않음. 출력전용, input에 있을 경우 무시
  - POLLOUT: 일반 데이터는 blocking 없이 출력된다. == POLLWRNORM
  - POLLPRI: 높은 우선순위 데이터는 blocking 없이 읽힌다
  - POLLRDBAND: 우선 데이터는 blocking 없이 읽힌다
  - POLLRDNORM: 일반 데이터는 blocking 없이 읽힌다
  - POLLWRBAND: 우선 데이터는 blocking 없이 쓰인다
  - POLLWRNORM: 일반 데이터는 blocking 없이 쓰인다

- normal, priority, high-priority 데이터의 구별은 특정 파일 타입이나 장치에 따라 다르다

- timeout이 0보다 큰 경우, fd가 준비될 때 까지 기다리는 최대 interval을 나타낸다(ms 단위)
- timeout이 0인 경우, poll()은 blocking 없이 return한다
- timeout이 -1인 경우, poll은 무기한 block한다

#### return
I/O 가능한 descriptor 수를 리턴한다. 시간 제한이 지난 경우, 0을 리턴한다. interrupted call을 포함한 에러 발생 시,<br/>fds 배열은 변하지 않고, errno를 설정한다

#### error
- 내부 데이터 구조체 할당 실패, 후속 요청은 성공할 수 있음
- fds가 프로세스에 할당된 주소 공간의 바깥쪽을 가르킴
- 시간 제한이 지나지 않았고, 선택된 이벤트가 발생하기 전에 시그널을 받음
- nfds 인자가 OPEN_MAX보다 크거나, timeout 인자가 -1보다 작음

---
### kqueue, kevent: kqueue fd를 할당한다. 이 fd는 사용자에게 filters라고 하는 kernel code 조각을 통해 kernel event(kevent)가 발생 하거나 상태가 유지된 것을 알려준다
```c++
int kqueue(void);<br/>
int kevent(
  int kq, const struct kevent *changelist, int nchanges,
  struct kevent *eventlist, int nevents, const sturct timespec *timeout
);
```
```c++
struct kevent {
  uintptr_t ident;  // identifier for this event (이벤트의 소스를 식별하기 위한 값.
                    // 정확한 해석은 filter에 따라 다르지만, 주로 fd 이다)
  int16_t   filter; // filter for event (이벤트를 실행하기 위한 kernel filter를 나타냄. 아래에 서술)
  uint16_t  flags;  // general flags (이벤트에서 실행할 행동)
  uint32_t  fflgas; // filter-spcific flags
  intptr_t  data; // filter-specific data
  void      *udata; // opaque user data identifier (kernel을 통해 전달되는 이 값은 변경되지 않는다.
                    // kevent system의 고유한 결정의 일부일 수 있다)
};
```
- flags
  - EV_ADD: kqueue에 이벤트를 추가한다. 존재하는 이벤트를 다시 추가하면 원본 이벤트의 인자를 바꾸고, 중복 entry를 만들지 않는다.<br/> 이벤트를 추가하면, EV_DISABLE flag가 없는 이상, 자동적으로 활성화 된다
  - EV_ENABLE: 이벤트 발생 시 kevent()가 그것을 반환하도록 허용한다
  - EV_DISABLE: kevent()가 반환하지 않도록 이벤트를 비활성화 한다. 필터는 비활성화 되지 않는다
  - EV_DELETE: kqueue에서 이벤트를 제거한다. fd에 연결된 이벤트는 descriptor가 모두 닫히면 자동적으로 삭제된다
  - EV_RECEIPT: kqueue에 아무 대기중인 이벤트에 draining 하지 않고 대량의 변화를 줄 때 유용하다.<br/>input으로 전달 됐을 때, EV_ERROR가 항상 리턴되도록 강제한다. filter가 성공적으로 추가되면, data filed는 0이 된다
  - EV_ONESHOT: filter가 발동되는 첫 경우에만 리턴하도록 한다. 사용자가 kqueue에서 이벤트를 받은 후 삭제된다.
  - EV_CLEAR: 사용자가 이벤트를 받은 후, 해당 상태를 리셋한다. 현재 상태 대신 상태 변화를 알고 싶을 때 유용하다.<br/>일부 fileter들은 내부적으로 이 플래그를 자동 활성화 한다
  - EV_EOF: filter-specific eof 조건을 나타내기 위해 사용한다
  - EV_OOBAND: 소켓의 필터를 읽는 경우, out of band 데이터가 descriptor에 존재함을 나태내기 위해 설정한다
  - EV_ERROR: return value 참조

- predefined system filter: filter와 주거나 받는 인자들은 data, fflags를 통해 전달된다
  - EVFILT_READ: fd르르 식별자로 받고, 읽을 수 있는 데이터가 있으면 반환한다. descriptor type에 따라 조금씩 다르게 행동한다.
    - Sockets:
      - listen()에 전달되었던 소켓들은 들어오는 연결이 보류중일 경우 리턴된다. data는 listen backlog의 크기를 담는다
      - 다른 소켓 descriptor들은 소켓 버퍼의 SO_RCVLOWAT 값에 따라, 읽을 데이터가 있을 때 리턴된다.<br/>이것은 
