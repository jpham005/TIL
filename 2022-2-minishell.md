# Minishell


### ansi escape sequence
---
미니쉘에서 커서 조정을 하려다가 발견한 사실인데, 색상까지 조정이 가능하다. 24비트 색상까지 사용 가능하기 때문에, 만족스러운 결과물을 만들 수 있다.<br/>
[위키](https://en.wikipedia.org/wiki/ANSI_escape_code)<br/>
단, 색상을 넣게 되면 readline 라이브러리에서 커서관련 버그가 생기기 때문에, (readline 라이브러리에서 길이를 측정하는 방식에서 생긴다고 한다. 버그인지 의도인지는 잘 모르겠다.) "\001" 와 "\002"로 감싸주어야 한다.

---
### signal 관리
---
fork 한 상태에서 시그널을 보내면 부모와 자식이 모두 시그널을 받기 때문에, 적절히 처리해서 진행해야 한다.

---
### set
---
shell 에서 $? 으로 exit status를 얻어올 수 있는 이유는, set으로 찾을 수 있는 변수목록에 ?에 저장되기 때문이다.
printenv등으로 환경변수만 찾아보면 알 수 없는 부분. 이 값은 쉘 자체의 exit status로도 쓰인다.

### set, env in shell
---
set은 지역 변수, env는 환경 변수를 보여준다.<br/>
따라서 a=1형태로 선언한 지역 변수는 set으로만 볼 수 있고, env로는 볼 수 없다.<br/>
set이 env보다 큰 개념이라고 생각하면 된다. (exit status도 local variable이기 때문에 set으로만 볼 수 있다.)

### tild expansion
---
~ 는 tild expansion이라는 기능으로 따로 정의되어 있다.
