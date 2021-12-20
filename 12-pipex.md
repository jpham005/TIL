# PIPEX

### open
---
파일을 오픈할 때, o_flag에 여러 값들을 넣어서 권한 설정을 할 수 있다. 이때, O_CREAT으로 없으면 만들면서 열 수 있는데, 이 경우 t_mode 형 변수를 받아서 생성되는 파일의 권한을 umask와 비교해서 생성한다.
이러한 umask값은 기본 022로 정해져있는데, 파일의 경우 666, 디렉토리의 경우 777에서 umask값을 빼서 생성한다.
ex. touch test ; ls -l => 666 - 022 = 644이므로, -rw-r--r-- 로 권한이 설정되는걸 확인 가능하다.
open 함수에서는, mask(인자) & (~umask & 777) 형태로 권한이 생성된다.

이 때 o_flag에 오는 값들은 헤더에 미리 정의되어 있는 매크로들이 있는데, O_RDONLY, O_WRONLY, O_RDWR | 옵션... 형태로 사용하면 된다.
그 중 난해한 옵션들을 정리해보았다.
- O_SHLOCK : 읽을 때 사용되는 lock. 해당 락이 걸려있으면, 그 상태에 쓰기를 위해 Exclusive Lock을 걸지 못하도록 한다.
- O_EXLOCK : 쓸 때 사용되는 lock. 이 경우에는 쓰기 위해 추가로 Exclusive Lock을 거는 것도 막는다.
- O_NONBLOCK : 위와 같은 상황에서, lock이 풀리는걸 기다리지 않고, 바로 -1 리턴 하도록 한다.\

### here doc
---
cmd1 << LIMITER | cmd2 >> file
cmd1 : redirection 가능한 명령
LIMITER : limit string => 이 시점부터 limit string을 입력받을때 까지 표준입력으로 받는다. 입력받은 데이터들을 모두 cmd1로 리다이렉션 하게 되는 것.
>> file : append to file
즉, << 과 >> 이 매우 다르게 쓰인다는 걸 유의해야 한다.
