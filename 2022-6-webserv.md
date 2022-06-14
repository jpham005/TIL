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
- kevent는 (ident, filter, [udata value]) 쌍으로 식별된다. 이 쌍은 알고싶은 조건들을 특정한다<br/>
  한 쌍은 주어진 kqueue에서 단 한번만 나타날 수 있다. 주어진 kqueue에 같은 쌍을 등록하려는 후속 시도는,<br/>
  관찰중인 조건을 대체한다(추가가 아님)<br/>
  udata value가 쌍에 들어갈지 여부는 kevent의 EV_UDATA_SPECIFIC 플래그로 제어된다

- kevent에서 식별되는 filter는
  - 이벤트를 처음 등록 시, 이미 존재하는 조건이 있는지 탐지하기 위해 실행되고
  - 이벤트가 filter에 evaluation되기 위해 전달되면 언제든 실행된다

- filter는 kqueue로부터 kevent를 받으려는 사용자의 시도에서도 실행된다. 만약 filter가 발동된 이벤트가 더이상 갖고있지 않은<br/>
조건을 나태내면, kevent는 kqueue에서 제거되고 리턴되지 않는다

- filter를 발동하는 다중 이벤트들은 kqueue에 다중으로 kevent가 올라가지 않게 한다.<br/>
대신, filter는 하나의 struct kevent에 이벤트들을 모으게 된다. fd에 close()호출은 descriptor를 참조하는 모든 kevent들을 제거한다

- kqueue() 시스템콜은 새 kernel event queue를 만들고 descriptor를 반환한다. 이 큐는 child에게 fork로 상속되지 않는다

- kevent() 시스템 콜은 queue에 이벤트들을 등록하기 위해 호출하고, 유저에게 대기중인 이벤트들을 리턴한다.<br/>
changelist 인자는 kevent배열 포인터다. changelist에 담긴 모든 변화는 queue에서 대기중인 이벤트들을 읽기 전에 적용된다<br/>
nchanges 인자는 changelist의 크기를 제공한다

- eventlist 인자는 kevent structure의 배열 포인터이다. nevents 인자는 eventlist의 크기를 정한다

- timeout이 non-NULL pointer 일 때, 이벤트를 기다릴 최대 interval을 나타낸다. 이는 struct timespec으로 해석된다<br/>
만약 timeout이 NULL pointer 이면, kevent는 무기한 대기한다.<br/>
poll 처럼 사용하기 위해, timeout 인자는 non_NULL이고, zero-valued timespec 구조체를 가르켜야 한다.<br/>
같은 배열이 changelist와 eventlist에 대해서도 사용될 수 있다

- EV_SET() 매크로는 kevent 구조체의 초기화 편의를 위해 제공된다.

- flags
  - EV_ADD: kqueue에 이벤트를 추가한다. 존재하는 이벤트를 다시 추가하면 원본 이벤트의 인자를 바꾸고, 중복 entry를 만들지 않는다.<br/>
  이벤트를 추가하면, EV_DISABLE flag가 없는 이상, 자동적으로 활성화 된다
  - EV_ENABLE: 이벤트 발생 시 kevent()가 그것을 반환하도록 허용한다
  - EV_DISABLE: kevent()가 반환하지 않도록 이벤트를 비활성화 한다. 필터는 비활성화 되지 않는다
  - EV_DELETE: kqueue에서 이벤트를 제거한다. fd에 연결된 이벤트는 descriptor가 모두 닫히면 자동적으로 삭제된다
  - EV_RECEIPT: kqueue에 아무 대기중인 이벤트에 draining 하지 않고 대량의 변화를 줄 때 유용하다.<br/>
  input으로 전달 됐을 때, EV_ERROR가 항상 리턴되도록 강제한다. filter가 성공적으로 추가되면, data filed는 0이 된다
  - EV_ONESHOT: filter가 발동되는 첫 경우에만 리턴하도록 한다. 사용자가 kqueue에서 이벤트를 받은 후 삭제된다.
  - EV_CLEAR: 사용자가 이벤트를 받은 후, 해당 상태를 리셋한다. 현재 상태 대신 상태 변화를 알고 싶을 때 유용하다.<br/>
  일부 fileter들은 내부적으로 이 플래그를 자동 활성화 한다
  - EV_EOF: filter-specific eof 조건을 나타내기 위해 사용한다
  - EV_OOBAND: 소켓의 필터를 읽는 경우, out of band 데이터가 descriptor에 존재함을 나태내기 위해 설정한다
  - EV_ERROR: return value 참조

- predefined system filter: filter와 주거나 받는 인자들은 data, fflags를 통해 전달된다
  - EVFILT_READ: fd를 식별자로 받고, 읽을 수 있는 데이터가 있으면 반환한다. descriptor type에 따라 조금씩 다르게 행동한다. (socekt, vnode, fifo, pipe, device node 상세 내용은 man 참조)
  - EVEFILT_EXCEPT: descriptor를 식별자로 받고, 예외 조건이 descriptor에서 발생하면 리턴한다. 조건은 fflags에서<br/>
  특정한다. NOTE_OOB fileter tag를 사용하는 socket descriptor의 out-of-band 데이터 도착을 모니터링 할 때 사용 가능하다
  - EVFILT_WRITE: fd를 식별자로 받고, descriptor에 쓰기가 가능할 때 리턴한다.<br/>
  socket, pipe, fifo에선, data는 write buffer의 남은 양을 담는다. reader가 연결이 끊기면, EV_EOF로 filter가 세팅된다.<br/>
  fifo의 경우, 이것은 EV_CLEAR로 초기화 할 수 있다.<br/>
  해당 필터는 vnode를 지원하지 않는다
  - EVFILT_AIO: 현재 지원 안함
  - EVFILT_VNODE: fd를 식별자로 받고, fflags의 이벤트들을 살피고 descriptor에서 발생한 한개 이상의 이벤트들을 리턴한다.<br/>
  아래 이벤트들을 모니터링한다
    - NOTE_DELETE: unlink() 시스템콜이 descriptor로 참조한 file에 불림
    - NOTE_WRITE: descriptor로 참조한 file에 쓰기가 발생함
    - NOTE_EXTEND: descriptor로 참조한 file이 확장됨(extended)
    - NOTE_ATTRIB: descriptor로 참조한 file의 속성이 변함
    - NOTE_LINK: fild의 link count가 바뀜
    - NOTE_RENAME: descriptor로 참조한 file의 이름이 바뀜
    - NOTE_REVOKE: 파일 접근이 revoke()로 취소되거나, 파일 시스템이 unmount됨
    - NOTE_FUNLOCK: flock()나 close()를 통해 file이 unlocked 됨
    - 반환 시, fflags는 이 필터를 통해 관측된 이벤트들에 맞는 filter-specific flag들을 담는다
  - EVFILT_PROC: monitor의 pid를 식별자로 받고, 추가로 fflags의 관측할 이벤트들을 받는다. 프로세스가 요청된 이벤트들을<br/>
  한개 이상 실행했을 때 리턴한다. 프로세스가 평범하게 다른 프로세스를 볼 수 있다면, 다른 프로세스에 이벤트를 attach 할 수 있다.<br/>
  아래 이벤트들을 모니터링 한다
    - NOTE_EXIT: 프로세스가 종료됨
    - NOTE_EXITSTATUS: 프로세스가 종료되었고, 그 exit status가 filter specific data에 있다.<br./>
    자식 프로세스이고, NOTE_EXIT과 같이 사용될 때만 유효하다
    - NOTE_FORK: 프로세스가 자식 프로세스를 만듦
    - NOTE_EXEC: 프로세스가 새 프로세스를 실행함
    - NOTE_SIGNAL: 프로세스가 시그널을 받음
    - NOTE_REAP: 프로세스가 부모에 의해 수거됨
    - 반환 시, fflags는 filter를 작동시킨 이벤트가 담긴다
  - EVFILT_SIGNAL: 모니터링할 시그널 넘버를 식별자로 받고, 프로세스에서 해당 시그널이 발생하면 리턴한다.<br/>
  signal()과 sigaction() facilities와 공존하고, 낮은 순위를 갖는다. 오직 시그널들이 프로세스로 보내졌을때만 (스레드 아님) filter를 작동시킨다.<br/>
  필터는 SIG_IGN이 있어도 프로세스에 시그널을 전달하려는 모든 시도들을 기록한다.<br/>
  이 필터들은 내부적으로 EV_CLEAR flag가 자동으로 적용된다
  - EVEFILT_MACHPORT: ident의 mach port나 port set의 이름을 받고, 메세지가 port나 port set에서 enqueued 될 때 까지 기다린다.<br/>
  메세지가 감지되었지만 kevent call에 의해 직접적으로 받지 않았다면, 메세지가 enqueued된 port의 이름이 data에 반환된다.
  - EVFILT_TIMER: data가 timeout 기간을 특정하는 interval timer을 ident를 통해 식별해서 설정한다<br/>
  ffags는 아래 플래그 중 하나를 포함할 수 있다 (기본 ms)
    - NOTE_SECONDS: data is in secnods
    - NOTE_USECONDS: data is in microsecnods
    - NOTE_NSECONDS: data is in nanosecnods
    - NOTE_MACHTIME: data is in Mach abolute time units

    fflags는 EV_ONESHOT 타이머를 interval 대신 절대적인 기한으로 설정하는 NOTE_ABSOLUTE도 포함 가능하다.<br/>
    절대 기한은 gettimeofday로 표현된다. NOTE_MACHTIME이 있다면, mach_absolute_time()으로 표현된다.<br/>
    타이머는 power를 아끼기 위해 다른 타이머와 더해질 수 있다. 아래 flags는 이런 행동을 수정하기 위해 사용된다
      - NOTE_CRITICAL: 기본 power-saving 기술을 여유 값들을 더 엄격하게 지키도록 한다
      - NOTE_BACKGROUND: 더 많은 power-saving 기술을 다른 타이머와 더하기 위해 사용한다

    타이머는 EV_ONESHOT이 없는 이상 주기적이다. 반환 시, data는 타이머 이벤트의 마지막 arming이나 마지막 전달로부터<br/>
    몇번이나 timeout이 만료됐는지(expired) 담는다<br/>
    이 필터들을 EV_CLEAR flag를 자동 설정한다

#### return
kqueue() 시스템콜은 새 kernel event queue를 만들고 fd를 반환한다. 만들다 에러가 발생했다면, -1을 리턴하고 errno를 설정한다.

kevent() 시스템콜은 eventlist에 위치한 이벤트의 수를 리턴한다 (nevents 이하<br/>
changelist의 요소를 진행하던 중 에러가 발생하고, eventlist에 충분한 공간이 있다면, 이벤트는 eventlist에 저장 + flags에 EV_ERROR, <br/>
data에 system error 가 있는 상태로 설정된다. <br/>
위 경우가 아니면, -1을 리턴하고, errno를 설정한다.<br/>
만약 time limit이 만료되면, 0을 리턴한다
