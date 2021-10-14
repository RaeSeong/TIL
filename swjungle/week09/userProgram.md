# week09
## pintOS project 2주차 - User Program

1주차를 진행했던 경험으로 2주차는 좀 더 스무스하게 진행할 수 있지 않을까 생각했는데 착각이었다. 어떻게 접근해야 하는지 이해했다고 생각했는데 #ifdef 구문으로 비활성화되어 있는 구문(USERPROG)이 참조로 검색할 수 없기도 하고 흐름대로 진행하다가 어느 시점에서 인터럽트가 되고 다시 다른 것이 실행되는지 파악하기가 어려웠다. 원래 알고 있던 단순한 하나의 흐름이 아니고 다른 흐름에 의해서 현재 흐름이 영향을 받기 때문에 ..

우선은 대략적인 흐름도를 보고 필요한 코드들을 입력하고 나중에 다시 파악해보자
# Argument Passing
명령어를 입력하면 해당 문자열을 실행할 파일, 인자1, 인자2 ... 를 구분해서 받아줘야 하고 그것들을 user stack 에 보내줘야 한다.
문자열을 구분하는 부분(1)과 스택으로 보내주는 부분(2)으로 나누어서 살펴보자.
```c
process_exec (void *f_name) {
	char *file_name = f_name;
	bool success;

	char *next_ptr;
	char *save_ptr = strtok_r(f_name, " ", &next_ptr);
	char *save_arg[128];
	int token_cnt = 1;
	save_arg[0] = save_ptr;

	while(save_ptr != NULL){
		save_ptr = strtok_r(NULL, " ", &next_ptr);
		save_arg[token_cnt] = save_ptr;
		token_cnt++;
	}
	/* We cannot use the intr_frame in the thread structure.
	 * This is because when current thread rescheduled,
	 * it stores the execution information to the member. */
	struct intr_frame _if;
	_if.ds = _if.es = _if.ss = SEL_UDSEG;
	_if.cs = SEL_UCSEG;
	_if.eflags = FLAG_IF | FLAG_MBS;

	/* We first kill the current context */
	process_cleanup ();

	/* And then load the binary */
	success = load (file_name, &_if);

	/* If load failed, quit. */
	palloc_free_page (file_name);
	if (!success)
		return -1;

	argument_stack(save_arg, token_cnt, &_if.rsp);

	_if.R.rdi = token_cnt - 1;
	_if.R.rsi = (uintptr_t *)_if.rsp + 1;

	/* Start switched process. */
	do_iret (&_if);
	NOT_REACHED ();
}
```
윗줄에 영어 주석이 달려있지 않은 코드들이 새로 추가된 코드들이다. strtok_r로 띄어쓰기가 있는 곳을 기준으로 입력받은 값들을 나눠서 save_arg배열에 각 포인터들을 저장한다. 인자를 나눈 배열, 인자의 수, 인터럽트 프레임을 받아서 argument_stack함수를 실행한다. rdi에 인자 갯수, rsi에 _if.rsp + 1 값을 저장한다.
```c
void argument_stack(char **parse, int count, void **rsp){
	char *word_address[128];
	int for_word_align = 0;

	for(int i = count - 2; i > -1; i--){
		for_word_align = for_word_align + strlen(parse[i]);

		for(int j = strlen(parse[i]); j > -1; j--){
			*rsp = (char *)*rsp - 1;
			**(char **)rsp = parse[i][j];
		}
		word_address[i] = (char *)*rsp;
	}

	*rsp = *(uintptr_t *)rsp & ~(uintptr_t)7;

	*rsp = (uintptr_t *)*rsp - 1;
	**(uint64_t **)rsp = NULL;

	for(int i = count - 2; i > -1; i--){
		*rsp = (uintptr_t *)*rsp - 1;
		**(uint64_t **)rsp = word_address[i];
	}

	*rsp = (uintptr_t *)*rsp - 1;
	**(uint64_t **)rsp = 0;
}
```
나눠진 인자들을 한 글자씩 스택포인터(rsp)에 넣으면서 포인터를 아래로 증가시킨다(-1).
해당 주소값을 word_address배열에 저장해뒀다가 & ~(uintptr_t)7 로 스택포인터가 8의 배수 값을 가리키도록 한 다음에 NULL 값을 하나 넣고 인자들의 주소값들(word_address)을 다시 추가한다. 이후에도 끝 부분을 구분할 수 있도록 0을 추가로 넣어준다.

# User Memory Access
user mode에서 system call을 호출할 때, 잘못된 위치(user memory를 벗어나는 범위)를 가리키게 되면 문제가 발생할 수 있기 때문에 이 문제를 해결해줘야 한다.
gitbook에 인자 받을 때 체크할 필요는 없다고 나와있는데 그냥 syscall_handler에서 바로 체크하면 되지 않을까 생각했다.
하지만 실제 체크가 필요한 함수는 exec, create, open, read, write 5개고 read, write의 체크가 필요한 인자는 다른 syscall과는 다른 레지스터에 등록되어 있어서 함수별로 적용하는 것으로 작성했다
```c
void check_address(const uint64_t *addr)	
{
	struct thread *cur = thread_current();
	if (addr == NULL || !(is_user_vaddr(addr)) || pml4_get_page(cur->pml4, addr) == NULL) {
		exit(-1);
	}
}
```
받아온 인자 주소값이 NULL이거나(1) 주소값이 user가상주소가 아니거나(2) 해당 주소에 page가 할당되어 있지 않으면(3) 종료하도록 구현했다.

# System Calls

이제 syscall 함수들을 구현해야 하는 차례인데
```c
void halt(void) {
	power_off();
}
```
halt
```c
void exit(int status) {
	struct thread *cur = thread_current();
	cur->exit_status = status;							// 프로그램이 정상적으로 종료되었는지 확인(정상적 종료 시 0)

	printf("%s: exit(%d)\n", thread_name(), status); 	// 종료 시 Process Termination Message 출력
	thread_exit();		// 스레드 종료
}
```
exit
```c
tid_t fork(const char *thread_name, struct intr_frame *f) {
	return process_fork(thread_name, f);
}
```
fork
> exec
```c
int exec(char *file_name) {
	check_address(file_name);

	int file_size = strlen(file_name)+1;
	char *fn_copy = palloc_get_page(PAL_ZERO);
	if (fn_copy == NULL) {
		exit(-1);
	}
	strlcpy(fn_copy, file_name, file_size);

	if (process_exec(fn_copy) == -1) {
		return -1;
	}

	NOT_REACHED();
	return 0;
}
```
exec
>create
```c
bool create(const char *file, unsigned initial_size) {		// file: 생성할 파일의 이름 및 경로 정보, initial_size: 생성할 파일의 크기
	check_address(file);
	return filesys_create(file, initial_size);
}
```
>remove
```c
bool remove(const char *file) {			// file: 제거할 파일의 이름 및 경로 정보
	// check_address(file);
	return filesys_remove(file);
}
```
>open
```c
int open(const char *file) {
	check_address(file);
	struct file *open_file = filesys_open(file);

	if (open_file == NULL) {
		return -1;
	}

	int fd = add_file_to_fdt(open_file);

	// fd table 가득 찼다면
	if (fd == -1) {
		file_close(open_file);
	}
	return fd;
}
```
>filesize
```c
int filesize(int fd) {
	struct file *open_file = find_file_by_fd(fd);
	if (open_file == NULL) {
		return -1;
	}
	return file_length(open_file);
}
```
>read
```c
int read(int fd, void *buffer, unsigned size)
{
	check_address(buffer);
	int read_result;
	struct thread *cur = thread_current();

	struct file *fileobj = find_file_by_fd(fd);
	if (fileobj == NULL)
		return -1;

	if (fileobj == 1)
	{
		if (cur->stdin_count == 0)
		{
			// Not reachable
			NOT_REACHED();
			remove_file_from_fdt(fd);
			read_result = -1;
		}
		else
		{
			int i;
			unsigned char *buf = buffer;
			for (i = 0; i < size; i++)
			{
				char c = input_getc();
				*buf++ = c;
				if (c == '\0')
					break;
			}
			read_result = i;
		}
	}
	else if (fileobj == 2)
	{
		read_result = -1;
	}
	else
	{
		lock_acquire(&filesys_lock);
		read_result = file_read(fileobj, buffer, size);
		lock_release(&filesys_lock);
	}
	return read_result;
}
```
>write
```
int write(int fd, const void *buffer, unsigned size) {
	check_address(buffer);
	int write_result;
	struct file *fileobj = find_file_by_fd(fd);

	if(fileobj == NULL)
		return -1;

	struct thread *cur = thread_current();

	if (fileobj == 2) {
		if(cur->stdout_count == 0){
			NOT_REACHED();
			remove_file_from_fdt(fd);
			write_result = -1;
		}
		putbuf(buffer, size);
		write_result = size;
	}
	else if(fileobj == 1){
		write_result = -1;
	}
	else {
		lock_acquire(&filesys_lock);
		write_result = file_write(fileobj, buffer, size);
		lock_release(&filesys_lock);
	}
	
	return write_result;
}
```
>seek
```c
void seek(int fd, unsigned position) {
	struct file *seek_file = find_file_by_fd(fd);
	if (seek_file <= 2) {		// 초기값 2로 설정. 0: 표준 입력, 1: 표준 출력
		return;
	}
	seek_file->pos = position;
}
```
>tell
```c
unsigned tell(int fd) {
	struct file *tell_file = find_file_by_fd(fd);
	if (tell_file <= 2) {
		return;
	}
	return file_tell(tell_file);
}
```
>close
```c
void close(int fd){
	struct file *fileobj = find_file_by_fd(fd);
	if (fileobj == NULL)
		return;

	struct thread *cur = thread_current();

	if (fd == 0 || fileobj == 1)
	{
		cur->stdin_count--;
	}
	else if (fd == 1 || fileobj == 2)
	{
		cur->stdout_count--;
	}

	remove_file_from_fdt(fd);
	if (fd <= 1 || fileobj <= 2)
		return;

	if (fileobj->dupCount == 0)
		file_close(fileobj);
	else
		fileobj->dupCount--;
}
```
# Process Termination Message
프로세스가 종료되면 메세지를 출력하라는 내용이다. 종료되는 부분에 printf 를 넣어주면 된다.

# Deny Write on Executables
실행 중인 파일은 write 기능을 제한하라는 내용이다. 파일이 오픈될 때, file_deny_write 함수를 활용해서 해당 파일 구조체의 deny_write 값을 true 로 만들어주고 inode의 deny_write_count 값을 올려준다. 해당 쓰레드에서 어떤 파일이 실행되고 있는지 알기 위해서 쓰레드 구조체의 running에 해당 파일을 넣어준다. 프로세스가 종료되거나 close 가 진행될 때 file_allow_write로 반대 행위를 한다. 코드에서는 file_write 함수 안에 inode_write_at 함수로 inode의 deny_write_count 값이 있으면 return 하게 되어 있다.

# Extend File Descriptor(Extra)
Project2 의 추가 과제로 특정 식별자가 이미 열려 있는 파일을 가리키도록 하는 함수를 구현하는 것이다. 같은 파일을 연다고 해도 열고 나면 각각의 공간에서 프로세스가 진행되기 때문에 이를 동기화해주는 역할이라고 본다.
```c
int dup2(int oldfd, int newfd) {
	struct file *file_fd = find_file_by_fd(oldfd);
	struct thread *cur = thread_current();
	struct file **fdt = cur->fd_table;

	if (file_fd == NULL)
		return -1;

	if (oldfd == newfd)
		return newfd;

	if (file_fd == 1)
		cur->stdin_count++;
	else if (file_fd == 2)
		cur->stdout_count++;
	else
		file_fd->dupCount++;
	
	close(newfd);
	fdt[newfd] = file_fd;
	return newfd;
}
```
없는 파일을 가리키려고 하면 -1 을 return하고 같은 식별자를 인자로 받으면 무의미한 행위여서 해당 식별자값을 return하고 그 외에 경우에 실제 dup2함수가 실행된다. stdin과 stdout은 프로세스 실행시 생성되는 기본 식별자여서 해당 쓰레드 구조체에 복제횟수를 관리하고, 이외의 파일들은 각 파일 구조체에 복제횟수를 관리한다. 여기서는 dup2이 정상 실행되면 플러스를 해주는데 그럼 어디에 저 값이 처음으로 지정되는지를 확인해야 한다. stdin과 stdout의 count 값은 쓰레드 구조체에 있으니 쓰레드가 처음 생성될 때 1로 초기화해주면 되고, 다른 파일의 dupCount는 파일이 생성될 때 0으로 초기화해준다(복제될 땐 해당 파일의 dupCount를 적용).  
그 이후에 newfd가 열어놓은 파일은 쓸모가 없어지니 close를 진행하고 쓰레드가 저장하고 있는 식별자테이블에 newfd가 가리키게 된 파일을 저장해준다.   
    
그런데 기존 상태로는 이러한 복제행위(dup2)가 fork로 추가되는 프로세스에는 반영되지 않는다. 이 부분을 해결해주기 위해 __do_fork 함수를 수정한다.
```c
struct MapElem
{
	uintptr_t key;
	uintptr_t value;
};
```
```c
static void __do_fork (void *aux)
	...
	const int MAPLEN = 10;
	struct MapElem map[10]; // key - parent's struct file * , value - child's newly created struct file *
	int dup_count = 0;		// index for filling map

	for (int i = 0; i < FDCOUNT_LIMIT; i++)
	{
		struct file *file = parent->fd_table[i];
		if (file == NULL)
			continue;

		// Project2-extra) linear search on key-pair array
		// If 'file' is already duplicated in child, don't duplicate again but share it
		bool found = false;
		for (int j = 0; j < MAPLEN; j++)
		{
			if (map[j].key == file)
			{
				found = true;
				current->fd_table[i] = map[j].value;
				break;
			}
		}
		if (!found)
		{
			struct file *new_file;
			if (file > 2)
				new_file = file_duplicate(file);
			else
				new_file = file; // 1 STDIN, 2 STDOUT

			current->fd_table[i] = new_file;
			if (dup_count < MAPLEN)
			{
				map[dup_count].key = file;
				map[dup_count++].value = new_file;
			}
		}
	}
	...
```
