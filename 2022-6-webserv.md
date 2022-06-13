# webserv

### 허용 함수
---

#### htonl, htons, ntohl, ntohs: 16, 32 bit quantities를 network byte order와 host byte order간 변환
uint32_t htonl(uint32_t hostlong)
uint16_t htons(uint16_t hosthshort)
uint32_t ntohl(uint32_t netlong)
uint16_t ntohs(uint16_t netshrot)

- network byte order는 big endian(MSB)
- network byte order와 머신이 같은 byte order일 경우, null macro로 루틴이 정의되어 있다

+ 이 루틴들은 gethostbyname과 getservent로 리턴되는 인터넷 주소와 포트와 함께 사용된다

---
#### int select(int nfds, fd_set *restrict readfds, writefds, errorfds, struct timeval *restirct timeout)<br/>: 아래 함수들로 리턴되는 I/O descriptor를 테스트 한다
- readfds: 읽기가 가능한 fd인지
- writefds: 쓰기가 가능한 fd인지
- errorfds: 예외 처리가 필요한 fd인지

+ descriptor sets는 int형 배열들에 bit field(비트 플래그 확인)로 저장된다(구현에 따라 다를 수 있음?)
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

#### int poll(struct pollfd fds[], ndfs_t nfds, int timeout)<br/>: fd set이 I/O 가능한지, 혹은 특정 이벤트가 발생했는지 검사한다.

```c++
struct pollfd {
  int fd; // fd
  short events; // events to look for (poll 하려는 이벤트)
  short revents; // events returned (발생할 예정이거나 발생한 이벤트)
}
```
- events와 revents에는 아래 bitmask의 bit가 있다
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

- 
