# week08
## pintOS project 1주차 - Thread

https://casys-kaist.github.io/pintos-kaist/ 의 내용을 기준으로 project1 을 진행했다. (영어가 잘 안 읽힌다 ㅠㅠ)

여러 소스 파일들을 확인하면서 수정해야 하는 일이 처음이다 보니 갈피를 제대로 잡지 못 했고 어설프게 따라한 것이 동작하지 않아서 당황스러웠다. 그래서 다시 처음부터 초기 코드를 살펴보고 흐름이 어떻게 이어지는지를 확인했다. 기본 베이스를 이해하는 것이 중요하고 기본이 된 상태에서 쌓아올려야 올려진 것들이 의미가 있다는 걸 새삼 느끼게 됐다. 우선 pintOS의 기본부터 살펴보자.
# Thread

1주차 프로젝트의 주제다.   

&nbsp;&nbsp;정확한 개념을 설명하라고 하면 잘 못할 것 같지만 나는 실행 과정의 기본 단위 정도로 이해했다. 프로세스와 비슷하게, 어떤 내용이 진행되는 과정인데 그게 프로세스 안에서 이루어지는 것이라 여러 개의 쓰레드가 프로세스의 자원을 공유할 수 있다는 특징을 장점으로 활용할 수 있기 때문에 중요하다고 한다.   
&nbsp;&nbsp;pintOS 프로젝트에서는 한 번에 하나의 쓰레드만 실행될 수 있기 때문에 하나의 프로세스처럼 보인다. 4kb의 크기로 되어 있고 0부터 Thread 구조체의 사이즈만큼 구조체에 대한 정보를 가지고 있고(구조체 정보가 많아지면 커널 스택의 공간이 좁아지기 때문에 1kb 이내로 유지하는 게 좋다고 한다.) 나머지 부분은 커널 스택으로 위에서부터 아래로 내려온다.   

&nbsp;&nbsp;Thread가 작동하는 원리는 해결해야 할 과제들을 진행하면서 파악해보자

# 해결해야 할 과제
* Alarm Clock
* Priority Scheduling
* Advanced Scheduler

## Alarm clock
&nbsp;&nbsp;첫 번째 과제는 쓰레드가 작동할 시간이 되면 알려주는 방식을 바꾸는 것이다. 기존에는 1 tick(pintOS에서 흘러가는 시간 단위, 약 10 ms)이 지날 때마다, 내 차례야? 라고 물어보게 돼 있다. 시간이 될 때까지 계속 아니라고 답할 질문인데 반복하는 건 딱 봐도 몹시 비효율적인 방식이다. 그래서 이것을 시간이 되면 차례를 알려주는 방식으로 변경해보려고 한다.   
```C
void timer_sleep (int64_t ticks) {
	int64_t start = timer_ticks ();

	ASSERT (intr_get_level () == INTR_ON);
	while (timer_elapsed (start) < ticks)
		thread_yield ();
}
```
&nbsp;&nbsp;timer_sleep함수의 while문을 대체할 방법을 찾아야 하는데 thread_sleep 함수를 구현해서 대체했다.
```C
void thread_sleep(int64_t ticks) {
	struct thread *cur;

	// 인터럽트를 금지하고 이전 인터럽트 레벨을 저장함
	enum intr_level old_level;
	old_level = intr_disable();
    
	// idle 스레드는 sleep되지 않아야 함
	cur = thread_current();
	ASSERT(cur != idle_thread);

	// awake 함수가 실행되어야 할 tick 값을 update
	update_next_tick_to_awake(cur->wakeup_tick = ticks);

	// 현재 스레드를 슬립 큐에 삽입한 후 스케줄
	list_push_back(&sleep_list, &cur->elem);

	// 이 스레드를 블락하고 다시 스케줄 될 때까지 블락된 상태로 대기
	thread_block();

	// 인터럽트를 다시 받아들이도록 수정
	intr_set_level(old_level);
}
```
이 함수 안에도 구현해야 하는 함수가 있는데 update_next_tick_to_awake가 그 함수다.
```c
void update_next_tick_to_awake(int64_t ticks) {
	// next_tick_to_awake 변수가 깨워야 하는 스레드의 깨어날 tick 값 중 가장 작은 tick
	// 갖도록 업데이트
	next_tick_to_awake = (next_tick_to_awake > ticks) ? ticks : next_tick_to_awake;
}
```