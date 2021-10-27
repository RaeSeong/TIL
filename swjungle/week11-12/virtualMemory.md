# week 11 - 12
## pintos project 4,5주차 - Virtual Memory

가상메모리

>주요 키워드
* page   

가상메모리를 구성하는 단위로 생각하면 쉽다. 보통 4kB의 크기로 구성하고 page를 관리하기 위해 관련 정보들을 64bit 구성으로는 Page Offset(12bit), Page-Table offset(9bit), Page-directory Offset(9bit), Page-Directory Pointer(9bit), Page-Map Level-4 Offset(9bit), 나머지(Sing Extend)로 보면 된다.

* frame

물리메모리에서 page 역할을 하는 것으로 이해했다. 

* Page Tables

page에서 frame 으로 매칭될 수 있도록 정보들을 관리하는 공간이다.

* Swap Slots

필요하지만 현재 사용중이 아닌 것들을 보관해두는 공간

>해야할 일

* Managin the Supplemental Page Table

page table 에 들어가지 않는 정보들을 따로 관리하는 또다른 page table. page fault가 발생하면 가상페이지를 찾을 목적과, 프로세스가 종료될 때 자원을 반환해주는 목적이 있다.

* Handling page fault

page fault가 발생했을 때 처리를 해줘야 하는데, 우선 제대로 된 접근인지 확인해야 한다.

# Memory Management
우선 page 구조체를 살펴보자
```c
struct page {
  const struct page_operations *operations;
  void *va;              /* Address in terms of user space */
  struct frame *frame;   /* Back reference for frame */

  union {
    struct uninit_page uninit;
    struct anon_page anon;
    struct file_page file;
#ifdef EFILESYS
    struct page_cache page_cache;
#endif
  };
};
```
다른 것들은 일반적으로 보던 구조체의 멤버들인데 *operation 과 uinon으로 묶인 멤버들은 자세히 살펴볼 필요가 있을 것 같다.
```c
struct page_operations {
  bool (*swap_in) (struct page *, void *);
  bool (*swap_out) (struct page *);
  void (*destroy) (struct page *);
  enum vm_type type;
};
```
*operation의 형식인 page_operations 구조체를 보면 함수 포인터들과 type 멤버로 이루어져 있는데 swap in, swap out, destroy 를 사용할 때 해당 page의 타입에 따라 구조체로 정의된 함수들을 사용하도록 구성해놓았다. union으로 묶인 멤버는 page의 type에 맞는 구조체 형식을 가질 수 있도록 정의해놓은 것이다.   
page type에 따라 구조체, 사용함수를 구분해놓는 목적인 것이다.

>Supplemental Page Table

이미 구성된 page table만으로는 page를 관리하기에는 부족한 정보들이 있어서 Supplemental Page Table(이하 spt)을 추가로 만들어줘야 한다. 배열, linked list 등으로도 만들 수 있지만 각각의 요소를 관리하기에 더 적합한 hash 구조로 만들기로 하자.   

프로세스가 시작될 때(initd함수), fork될 때(__do_fork 함수) spt를 초기화하게 되어 있다.
```c
void
supplemental_page_table_init (struct supplemental_page_table *spt UNUSED) {
	struct hash* page_table = malloc(sizeof (struct hash));
	hash_init (page_table, page_hash, page_less, NULL);
	spt -> page_table = page_table;
}

static uint64_t
page_hash (const struct hash_elem *p_, void *aux UNUSED) {
  const struct page *p = hash_entry (p_, struct page, hash_elem);
return hash_bytes (&p->va, sizeof p->va);
}

static bool
page_less (const struct hash_elem *a_,
           const struct hash_elem *b_, void *aux UNUSED) {
  const struct page *a = hash_entry (a_, struct page, hash_elem);
  const struct page *b = hash_entry (b_, struct page, hash_elem);

  return a->va < b->va;
}
```
spt초기화 작업은 hash 구조체만큼의 메모리를 할당해주고 그 hash구조체를 초기화한다. hash구조체를 초기화할 때, hash 값을 생성하는 함수(page_hash)와 hash_elem에 해당하는 page의 가상주소값을 비교하는 함수(page_less)를 넣어준다.

```c
struct page *
spt_find_page (struct supplemental_page_table *spt UNUSED, void *va UNUSED) {
	struct page page;
	page.va = pg_round_down (va);
	struct hash_elem *e = hash_find (spt -> page_table, &page.hash_elem);
	if (e == NULL) return NULL;
	struct page* result = hash_entry (e, struct page, hash_elem);
	ASSERT((va < result -> va + PGSIZE) && va >= result -> va);
	return result;
}
```
첫 번째 인자(spt)에서 두 번째 인자(va) 값이 들어 있는 page를 찾는 함수이다. pg_round_down으로 va가 포함된 page의 시작 주소를 찾고 그 주소에 해당하는 spt 의 요소를 찾아와서 그 요소가 가리키는 page를 return한다.

```c
bool
spt_insert_page (struct supplemental_page_table *spt,
		struct page *page) {
	struct hash_elem *result= hash_insert (spt -> page_table, &page -> hash_elem);
	return (result == NULL) ? true : false ;
}
```
hash_insert로 요소를 넣어준다.

>Frame Management
현재까지 구현된 코드에서는 page에 메타데이터를 가지고 있지 않아서 물리메모리를 관리하려면 frame이라는 개념이 필요하다.
```c
struct frame {
	void *kva;
	struct page *page;
	struct list_elem elem;
};
```
물리주소와 매핑되는 커널가상주소(kva)와 page 구조체를 멤버로 가지고 있고 frame들을 관리하기 위해 각각의 frame에 elem 값이 들어가도록 추가했다.
```c
static struct frame *
vm_get_frame (void) {
	struct frame * frame = malloc (sizeof (struct frame));
	frame -> kva = palloc_get_page (PAL_USER);
	frame -> page = NULL;

	ASSERT (frame->kva != NULL);
	return frame;
}
```
vm_get_frame으로 frame 구조체의 크기만큼 메모리를 할당하고 그 주소값을 palloc_get_page로 받아온다.

```c
static bool
vm_do_claim_page (struct page *page) {
	struct thread *curr = thread_current ();
	struct frame *frame = vm_get_frame ();

	/* Set links */
	ASSERT (frame != NULL);
	ASSERT (page != NULL);
	frame->page = page;
	page->frame = frame;

	/* TODO: Insert page table entry to map page's VA to frame's PA. */
	if (!pml4_set_page (curr -> pml4, page -> va, frame->kva, page -> writable))
		return false;
	return swap_in (page, frame->kva);
}
```
vm_get_frame으로 frame을 만들고 page와 서로를 가리키도록 설정한다. pml4_set_page를 진행하고 실패하면 false를 return해준다.

```c
bool
vm_claim_page (void *va UNUSED) {
	struct page *page = spt_find_page (&thread_current () ->spt, va);
	if (page == NULL) return false;
	return vm_do_claim_page (page);
}
```
가상주소를 인자로 받으면 spt에서 해당 page를 찾아와서 vm_do_claim_page를 진행한다.

# Anonymous Page
disk에서 가져온 정보가 아닌 것들은 실행 중일 때만 사용되기 때문에 아무 작업 없이 끝내버리면 해당 정보는 그걸로 끝이 난다. 이런 특성의 page를 어떻게 관리해야 하는지 알아보자.

```c
struct anon_page {
  struct thread* owner;
    size_t swap_slot_idx;
};
```
어떤 쓰레드의 page인지, swap_slot의 몇 번째인지에 대한 정보를 담게 했다. 처음 page의 해당 정보를 초기화해줘야 하는데 anon_page뿐만 아니라 각각의 type들이 초기화되는 과정을 같이 봐야 한다.

```c
bool
vm_alloc_page_with_initializer (enum vm_type type, void *upage, bool writable,
		vm_initializer *init, void *aux) {

	struct supplemental_page_table *spt = &thread_current ()->spt;

	/* Check wheter the upage is already occupied or not. */
	if (spt_find_page (spt, upage) == NULL) {
		ASSERT(type != VM_UNINIT);
		struct page* page = malloc (sizeof (struct page));
		/* TODO: Insert the page into the spt. */
		if (VM_TYPE(type) == VM_ANON){
			uninit_new (page, upage, init, type, aux, anon_initializer);
		}
		else if (VM_TYPE(type) == VM_FILE){
			uninit_new (page, upage, init, type, aux, file_backed_initializer);
		}

		page -> writable = writable;
		spt_insert_page (spt, page);
		return true;
	}
	return false;
}
```
커널이 새로운 page 요청을 받으면(setup_stack, load_segment, vm_stack_growth, mmap 등 스택이나 메모리 등에 추가 page가 필요한 경우) vm_alloc_page_with_initialize 함수를 호출한다. 이 함수에서 page 크기 메모리를 할당하고 타입에 맞는 초기화 함수를 가지고 있는 uninit 상태로 만들어 놓는다. 그 페이지를 spt에도 넣어준다.

```c
uninit_initialize (struct page *page, void *kva) {
	struct uninit_page *uninit = &page->uninit;

	/* Fetch first, page_initialize may overwrite the values */
	vm_initializer *init = uninit->init;
	void *aux = uninit->aux;

	/* TODO: You may need to fix this function. */
	return uninit->page_initializer (page, uninit->type, kva) &&
		(init ? init (page, aux) : true);
}
```
uninit이었던 page의 정보들을 가져와서 page_initializer를 실행하고 init은 있으면 해보고 없으면 true. 두 함수 모두 true를 반환하면 정상적으로 초기화가 됐다고 본다. <!-- 이 두 함수에 활용되는 실제 작동 함수들도 살펴보자. -->

```c
void
vm_anon_init (void) {
	/* TODO: Set up the swap_disk. */
	swap_disk = NULL;
}
```
init.c의 메인함수에서 vm_init이 실행될 때, vm_file_init과 함께 실행해주는데 anonymous type이 임시로 저장될 swap_disk를 초기화해준다.

```c
bool
anon_initializer (struct page *page, enum vm_type type, void *kva) {
	/* Set up the handler */
	page->operations = &anon_ops;
	if (type & VM_STACK) page->operations = &anon_stack_ops;
	struct anon_page *anon_page = &page->anon;
	anon_page->owner = thread_current ();
	return true;
}
```
uninit_initialize의 인자 page가 anon type이면 anon_initializer 함수가 들어있을 것이다. page operations로 swap_in, swap_out, destroy를 anon 형식에 맞는 함수로 설정해준다(anon_ops). 그 중에서도 stack이면 stack용 ops로 다시 설정해준다. 그러고서 anon 구조체의 owner, swap_slot_idx를 현재 쓰레드, 최대값으로 각각 초기화해준다.

```c
static bool
load_segment (struct file *file, off_t ofs, uint8_t *upage,
		uint32_t read_bytes, uint32_t zero_bytes, bool writable) {
	ASSERT ((read_bytes + zero_bytes) % PGSIZE == 0);
	ASSERT (pg_ofs (upage) == 0);
	ASSERT (ofs % PGSIZE == 0);

	off_t read_ofs = ofs;
	while (read_bytes > 0 || zero_bytes > 0) {
		/* Do calculate how to fill this page.
		 * We will read PAGE_READ_BYTES bytes from FILE
		 * and zero the final PAGE_ZERO_BYTES bytes. */
		size_t page_read_bytes = read_bytes < PGSIZE ? read_bytes : PGSIZE;
		size_t page_zero_bytes = PGSIZE - page_read_bytes;

		/* TODO: Set up aux to pass information to the lazy_load_segment. */
		struct load_info *aux = malloc (sizeof (struct load_info));
		aux -> file = file_reopen(file);
		aux -> ofs = read_ofs;
		aux -> page_read_bytes = page_read_bytes;
		aux -> page_zero_bytes = page_zero_bytes;
		if (!vm_alloc_page_with_initializer (VM_ANON, upage,
					writable, lazy_load_segment, aux))
			return false;

		/* Advance. */
		read_bytes -= page_read_bytes;
		zero_bytes -= page_zero_bytes;
		upage += PGSIZE;
		read_ofs += page_read_bytes;
	}
	return true;
}
```
vm 이전에는 segment마다(while문 한 번 돌 때마다) 메모리를 할당했는데 이제는 필요한 정보들을 aux에 저장하고 vm_alloc_page_with_initializer로 보내면서 lazy_load_segment 함수를 나중에 uninit_initialize 할 때 활용하도록 구현했다.

```c
static bool
lazy_load_segment (struct page *page, void *aux) {
	/* TODO: Load the segment from the file */
	/* TODO: This called when the first page fault occurs on address VA. */
	/* TODO: VA is available when calling this function. */
	struct load_info* li = (struct load_info *) aux;
	if (page == NULL) return false;
	ASSERT(li ->page_read_bytes <=PGSIZE);
	ASSERT(li -> page_zero_bytes <= PGSIZE);
	/* Load this page. */
	if (li -> page_read_bytes > 0) {
		file_seek (li -> file, li -> ofs);
		if (file_read (li -> file, page -> va, li -> page_read_bytes) != (off_t) li -> page_read_bytes) {
			vm_dealloc_page (page);
			free (li);
			return false;
		}
	}
	memset (page -> va + li -> page_read_bytes, 0, li -> page_zero_bytes);
	file_close (li -> file);
	free (li);
	return true;
}
```
page fault가 발생해서 uninit_initialize가 호출되면 인자로 받았던 lazy_load_segment가 page_initialiszer와 같이 실행된다. page 가상주소에 load_segment 때 받았던 정보들을 넣어준다.

```c
static bool
setup_stack (struct intr_frame *if_) {
	bool success = true;
	void *stack_bottom = (void *) (((uint8_t *) USER_STACK) - PGSIZE);

	if (!vm_alloc_page (VM_ANON | VM_STACK, stack_bottom ,true)) return false;;
	if (!vm_claim_page(stack_bottom)) return false;
	memset(stack_bottom, 0, PGSIZE);
	if_->rsp = USER_STACK;
	return success;
}
```
vm_type에 VM_STACK 값을 추가해서 stack 타입인지에 대한 정보를 추가로 지정해주고, 최초 스택은 page fault 없이 바로 할당해주면 된다.

```c
bool
vm_try_handle_fault (struct intr_frame *f UNUSED, void *addr UNUSED,
		bool user UNUSED, bool write UNUSED, bool not_present UNUSED) {
	struct thread *curr = thread_current ();
	struct supplemental_page_table *spt UNUSED = &curr->spt;
	
	/* TODO: Validate the fault */
	/* TODO: Your code goes here */
	if (is_kernel_vaddr (addr) && user) return false;

	// void *stack_bottom = pg_round_down(curr->saved_sp); 현재 스택 사이즈 상관 없이 최대 스택 사이즈로 조건 변경함
	if(write && (USER_STACK - (1<<20) - PGSIZE <= addr && (uintptr_t)addr < USER_STACK)){
		vm_stack_growth(addr);
		return true;
	}
	struct page* page = spt_find_page (spt, addr);
	if (page == NULL) return false;
	if (write && !not_present) return vm_handle_wp (page);
	return vm_do_claim_page (page);
}
```
fault 발생한 주소가 스택 공간(USER_STAKC 부터 스택 최대 크기-1MB)이면 스택크기를 늘려주고 주소에 맞는 페이지를 찾아 vm_do_claim_page를 호출한다.

>spt - revisit

fork 관련 테스트를 통과하려면 spt를 복사하는 것과 없애는 것을 구현해야 한다.

```c
bool supplemental_page_table_copy(struct supplemental_page_table *dst UNUSED,
																	struct supplemental_page_table *src UNUSED)
{
	struct hash_iterator i;
	hash_first(&i, src->page_table);
	while (hash_next(&i))
	{
		struct page *page = hash_entry (hash_cur (&i), struct page, hash_elem);

		if(page->operations->type == VM_UNINIT){
			vm_initializer *init = page->uninit.init;
			bool writable = page->writable;
			int type = page->uninit.type;
			if(type & VM_ANON){
				struct load_info *li = malloc(sizeof(struct load_info));
				li->file = file_duplicate(((struct load_info *)page->uninit.aux)->file);
				li->page_read_bytes = ((struct load_info *)page->uninit.aux)->page_read_bytes;
				li->page_zero_bytes = ((struct load_info *)page->uninit.aux)->page_zero_bytes;
				li->ofs = ((struct load_info *)page->uninit.aux)->ofs;
				vm_alloc_page_with_initializer(type, page->va, writable, init, (void *)li);
			}
			else if(type & VM_FILE){

			}
		}
		/* Handle ANON/FILE page*/
		else if (page_get_type(page) == VM_ANON)
		{
			if (!vm_alloc_page(page->operations->type, page->va, page->writable))
				return false;
			struct page *new_page = spt_find_page(&thread_current()->spt, page->va);
			if (!vm_do_claim_page(new_page))
				return false;
			memcpy(new_page->frame->kva, page->frame->kva, PGSIZE);
		}
		else if(page_get_type(page) == VM_FILE){

		}
	}
	return true;
}
```
hash_iterator 구조체를 활용해서 spt를 while문으로 작업을 해준다. uninit 상태인지 여부와 anon인지 file인지 여부를 각각 따져서 해당 type에 존재하는 정보들을 복사해준다(file 관련된 건 일단 비워둔다).

```c
static void
spt_destroy (struct hash_elem *e, void *aux UNUSED){
	struct page *page = hash_entry (e, struct page, hash_elem);
	ASSERT (page != NULL);
	destroy (page);
	free (page);
}

void
supplemental_page_table_kill (struct supplemental_page_table *spt) {
	/* TODO: Destroy all the supplemental_page_table hold by thread and
	 * TODO: writeback all the modified contents to the storage. */
	if (spt -> page_table == NULL) return;
	lock_acquire(&spt_kill_lock);
	hash_destroy(spt->page_table, spt_destroy);
	free(spt->page_table);
	lock_release(&spt_kill_lock);
}
```
spt를 kill하는 동안 lock을 걸어두고 #define으로 정의된 destroy로 페이지 타입별 destroy함수를 활용하도록 spt_destroy를 사용해서 hash구조체인 spt를 clear 시키고 메모리를 free해준다.
이제 각각의 destroy 함수를 구현해줘야 하는데 file은 제외하고 uninit과 anon 타입의 함수를 살펴보자.

```c
static void
uninit_destroy (struct page *page) {
	struct uninit_page *uninit UNUSED = &page->uninit;
	/* TODO: Fill this function.
	 * TODO: If you don't have anything to do, just return. */
	if (page->uninit.aux != NULL) free (page->uninit.aux);
	return;
}
```
uninit에 들어있던 정보를 free 시켜준다

```c
static void
anon_destroy (struct page *page) {
	if (&page->frame!= NULL){
		
	}
	else {
		// Swapped anon page case
		struct anon_page *anon_page = &page->anon;
		ASSERT (anon_page->swap_slot_idx != INVALID_SLOT_IDX);

		// Clear swap table
		bitmap_set (swap_table, anon_page->swap_slot_idx, false);
	}
}
```
swap_table 의 비트를

# Stack Growth
USER_STACK 위치에서 PGSIZE(4,096)만큼만 사용할 수 있었는데 스택 포인터가 그보다 더 아래를 가리키면 사용가능한 스택사이즈를 늘려서 더 사용할 수 있도록 구현해야 한다.
```c
bool
vm_try_handle_fault (struct intr_frame *f UNUSED, void *addr UNUSED,
		bool user UNUSED, bool write UNUSED, bool not_present UNUSED) {
	struct thread *curr = thread_current ();
	struct supplemental_page_table *spt UNUSED = &curr->spt;
	
	/* TODO: Validate the fault */
	/* TODO: Your code goes here */
	if (is_kernel_vaddr (addr) && user) return false;

	// void *stack_bottom = pg_round_down(curr->saved_sp); 현재 스택 사이즈 상관 없이 최대 스택 사이즈로 조건 변경함
	if(write && (USER_STACK - (1<<20) - PGSIZE <= addr && (uintptr_t)addr < USER_STACK)){
		vm_stack_growth(addr);
		return true;
	}
	struct page* page = spt_find_page (spt, addr);
	if (page == NULL) return false;
	if (write && !not_present) return vm_handle_wp (page);
	return vm_do_claim_page (page);
}
```

page fault가 발생하면 vm_try_handle_fault를 호출하게 되는데 fault가 발생한 주소값이 스택으로 사용할 수 있는 곳(여기서는 1MB로 한정 -> USER_STACK ~ 1<<20 까지, - PGSIZE를 추가해서 스택 공간이 초과되면 vm_stack_growth에서 PANIC 되도록 함)이면 vm_stack_growth를 호출하도록 한다.

```c
vm_stack_growth (void *addr UNUSED) {
	void *stack_bottom = pg_round_down(addr);
	size_t req_stack_size = USER_STACK - (uintptr_t)stack_bottom;
	if(req_stack_size > (1 << 20)) PANIC("Stack limit exceeded!\n");

	void *growing_stack_bottom = stack_bottom;
	while((uintptr_t)growing_stack_bottom < USER_STACK && vm_alloc_page(VM_ANON | VM_STACK, growing_stack_bottom, true)){
		growing_stack_bottom += PGSIZE;
	}
	vm_claim_page(stack_bottom);
}
```
addr을 포함하는 page의 하단을 이번에 할당할 스택 사이즈 끝 부분으로 설정하고 page 단위로 할당하면서 USER_STACK 범위를 벗어나거나 이미 할당되어 있으면 멈추는 반복문을 실행한다.

# Memory Mapped Files
file에 기반한 데이터를 활용하기 위해 mmap 과 munmap 시스템콜을 구현해주어야 한다.

```c
static void *mmap(void *addr, size_t length, int writable, int fd, off_t offset){
	if(addr == 0 || (!is_user_vaddr(addr)) || (uint64_t)addr % PGSIZE != 0 || offset % PGSIZE != 0
	|| (uint64_t)addr + length == 0 || !is_user_vaddr((uint64_t)addr + length) || length == 0)
		return NULL;
	for(uint64_t i = (uint64_t)addr; i < (uint64_t)addr + length; i += PGSIZE){
		if(spt_find_page(&thread_current()->spt, (void *)i) != NULL)
			return NULL;
	}
	struct file *file = find_file_by_fd(fd);
	if(file == NULL || fd == 0 || fd == 1)
		return NULL;

	return do_mmap(addr, length, writable, file, offset);
}
```
```c
static void munmap(void *addr){
	do_munmap(addr);
}
```
우선은 mmap의 경우는 유효하지 않은 요청들을 필터링하고, 각각 do_가 붙은 함수에서 실제 행위를 정의해준다.
```c
void
vm_file_init (void) {
	list_init(&mmap_file_list);
}
```
다음 함수들을 활용하기 위해 mmap_file_list를 전역변수로 설정하고 vm_file_init에서 초기화를 해준다.

```c
void *
do_mmap (void *addr, size_t length, int writable,
		struct file *file, off_t offset) {
	off_t ofs = offset;
	uint64_t initaddr = addr;
	uint64_t read_bytes;
	uint64_t real_len = length > file_length(file) ? file_length(file) : length;
	for(uint64_t i = real_len; i > 0; i -= read_bytes){
		struct mmap_info *mi = malloc(sizeof(struct mmap_info));
		read_bytes = i >= PGSIZE ? PGSIZE : i;
		mi->file = file_reopen(file);
		mi->offset = ofs;
		mi->read_bytes = read_bytes;
		vm_alloc_page_with_initializer(VM_FILE, (void *)((uint64_t)initaddr), writable, lazy_load_file, (void *)mi);
		ofs += read_bytes;
		initaddr += PGSIZE;
	}
	struct mmap_file_info *mfi = malloc(sizeof(struct mmap_file_info));
	mfi->start = (uint64_t)addr;
	mfi->end = (uint64_t)pg_round_down((uint64_t)addr + real_len - 1);
	list_push_back(&mmap_file_list, &mfi->elem);
	return addr;
}
```
인자로 받은 length(file길이보다 크면 file길이)를 PGSIZE 단위로 mmap_info(file, offset, read_bytes) 값을 vm_alloc_page_with_initializer로 보내서 초기화될 때 lazy_load_file을 호출하도록 한다. 어디서부터 어디까지 적용됐는지 mmap_file_list(전역변수)에도 정보를 저장한다. 최종적으로는 처음 addr를 return한다.

```c
static bool lazy_load_file(struct page *page, void *aux){
	struct mmap_info *mi = (struct mmap_info *)aux;
	file_seek(mi->file, mi->offset);
	page->file.size = file_read(mi->file, page->va, mi->read_bytes);
	page->file.ofs = mi->offset;
	if(page->file.size < PGSIZE)
		memset(page->va + page->file.size, 0, PGSIZE - page->file.size);
	pml4_set_dirty(thread_current()->pml4, page->va, false);
	free(mi);
	return true;
}
```
mmap에서 보낸 mmap_info를 aux로 받아와서 offset 위치를 찾고 va에서부터 read_bytes만큼 읽는다. read_bytes가 PGSIZE 보다 작으면 나머지 공간은 0으로 채워서 PGSIZE 단위를 유지한다. 이제 해당 va는 메모리와 매핑되었으니 dirty값을 false로 바꿔주고 다 사용햔 mmap_info는 free 해준다.

```c
void
do_munmap (void *addr) {
	if(list_empty(&mmap_file_list))
		return;
	for(struct list_elem *i = list_front(&mmap_file_list); i != list_end(&mmap_file_list); i = list_next(i)){
		struct mmap_file_info *mfi = list_entry(i, struct mmap_file_info, elem);
		if(mfi->start == (uint64_t)addr){
			// for(uint64_t j = (uint64_t)addr; j <= mfi->end; j += PGSIZE){
			// 	struct page *page = spt_find_page(&thread_current()->spt, (void *)j);
			// 	spt_remove_page(&thread_current()->spt, page);
			// }
			// list_remove(&mfi->elem);
			// free(mfi);
			// return;
			struct page *page = spt_find_page(&thread_current()->spt, addr);
			struct file_page *file_page UNUSED = &page->file;
			list_remove(&mfi->elem);
			if(pml4_is_dirty(thread_current()->pml4, page->va)){
				file_seek(file_page->file, file_page->ofs);
				file_write(file_page->file, page->va, file_page->size);
				pml4_set_dirty(thread_current()->pml4, page->va, false);
			}
			pml4_clear_page(thread_current()->pml4, page->va);
		}
	}
}
```
munmap 시스템콜이 실행될 때 do_munmap을 통해 우선적으로 mmap_file_list 가 없으면(mmap을 실행했으면 리스트에 있어야 한다) 아무것도 할 수 없게 한다. for문으로 mmap_file_list 중에 addr가 맞는 mmap 정보를 찾아 작업을 시작한다. spt에서 addr에 맞는 page를 찾고 그 페이지의 file_page도 찾아놓는다. 기본적으로 mmap_file_list에서 해당 요소를 삭제하고 pml4에서도 clear하는 작업을 진행한다. 만약 dirty 가 체크되어 있으면 추가 작업을 하는데, 내용이 바뀌었으니 바뀐 내용이 사라지기 전에 file에 적용하고 dirty 값을 초기화하는 것이다.

```c
bool
file_backed_initializer (struct page *page, enum vm_type type, void *kva) {
	struct file* file = ((struct mmap_info*)page ->uninit.aux)->file;
	page->operations = &file_ops;

	struct file_page *file_page = &page->file;
	file_page -> file = file;
	return true;
}
```
page형식 중 file타입으로 file_page를 정의하고 vm_alloc_page_with_initializer에서 uninit_new로 넘어올 때 받아온 aux의 file을 정의된 file_page의 file로 지정한다. 해당 페이지의 operations를 file_ops로 작동하도록 설정한다.

```c
static void
file_backed_destroy (struct page *page) {
	struct file_page *file_page UNUSED = &page->file;

	if(pml4_is_dirty(thread_current()->pml4, page->va)){
		file_seek(file_page->file, file_page->ofs);
		file_write(file_page->file, page->va, file_page->size);
	}
	file_close(file_page->file);
}
```
dirty 값이 체크되어 있으면 변경사항 적용해주고, 해당 file_page의 file은 닫는다.
# Swap In/Out
물리메모리가 가득차면 그 사용이 끝날 때까지 기다리는 게 아니라 swap in/out으로 유동적인 관리가 가능하다. anon type의 page와 file type의 page를 따로 관리해줘야 하는데 anon부터 살펴보자

```c
vm_anon_init (void) {
	/* TODO: Set up the swap_disk. */
	// swap_disk = NULL;
	swap_disk = disk_get(1,1);

	disk_sector_t num_sector = disk_size(swap_disk);
	size_t max_slot = num_sector / SECTORS_PER_PAGE;

	swap_table = bitmap_create(max_slot);
}
```
```c
bool
anon_initializer (struct page *page, enum vm_type type, void *kva) {
	
	...
	anon_page->swap_slot_idx = INVALID_SLOT_IDX;

	return true;
}
```
anon type은 실행 중에만 사용되기 때문에 swap out 처리를 하려면 임시 저장공간이 필요하다. swap_disk 를 생성해주고   
anon_initializer에서 slot_idx를 초기화해준다

```c
static bool
anon_swap_in (struct page *page, void *kva) {
	struct anon_page *anon_page = &page->anon;
	if(anon_page->swap_slot_idx == INVALID_SLOT_IDX)
		return false;
	
	disk_sector_t sec_no;

	for(int i = 0; i < SECTORS_PER_PAGE; i++){
		sec_no = (disk_sector_t)(anon_page->swap_slot_idx * SECTORS_PER_PAGE) + i;
		off_t ofs = i * DISK_SECTOR_SIZE;
		disk_read(swap_disk, sec_no, kva+ofs);
	}

	bitmap_set(swap_table, anon_page->swap_slot_idx, false);
	anon_page->swap_slot_idx = INVALID_SLOT_IDX;

	return true;
}
```
swap_slot_idx가 초기상태이면 swap_disk가 비어 있다는 뜻(swap out해준 적이 없음)이므로 false 를 return 해준다.   
swap disk 에서 내용을 읽어오고 swap_table에 해당 정보를 false 처리, swap_slot_idx 값을 초기 상태로 만들어서 swap_in이 완료된 것으로 처리한다.

```c
static bool
anon_swap_out (struct page *page) {
	struct anon_page *anon_page = &page->anon;

	size_t swap_slot_idx = bitmap_scan_and_flip(swap_table, 0, 1, false);
	if(swap_slot_idx == BITMAP_ERROR)
		PANIC("There is no free swap slot");
	
	if(page == NULL || page->frame == NULL || page->frame->kva == NULL)
		return false;
	
	disk_sector_t sec_no;

	for(int i = 0; i < SECTORS_PER_PAGE; i++){
		sec_no = (disk_sector_t)(swap_slot_idx * SECTORS_PER_PAGE) + i;
		off_t ofs = i * DISK_SECTOR_SIZE;
		disk_write(swap_disk, sec_no, page->frame->kva + ofs);
	}
	anon_page->swap_slot_idx = swap_slot_idx;

	pml4_clear_page(anon_page->owner->pml4, page->va);
	pml4_set_dirty(anon_page->owner->pml4, page->va, false);
	page->frame = NULL;

	return true;
}
```
swap_table에 저장할 수 있는 공간을 찾아와서 idx값을 받고 그곳에 정보들을 쓴다. 그리고 해당 페이지에 idx 정보를 저장하고 pml4_clear를 진행하고 dirty 값을 초기화하고 page의 frame도 NULL로 만들어준다.
```c
static bool
file_backed_swap_in (struct page *page, void *kva) {
	struct file_page *file_page UNUSED = &page->file;
	if(file_page->file == NULL)
		return false;
	
	file_seek(file_page->file, file_page->ofs);
	off_t read_size = file_read(file_page->file, kva, file_page->size);
	if(read_size != file_page->size)
		return;
	if(read_size < PGSIZE)
		memset(kva + read_size, 0, PGSIZE - read_size);
	
	return true;
}
```
file type의 page 는 file을 기준으로 작동하면 된다. file의 offset을 찾아서 size만큼 읽어주고 PGSIZE 보다 작으면 남은 공간을 0으로 채워준다.

```c
static bool
file_backed_swap_out (struct page *page) {
	struct file_page *file_page UNUSED = &page->file;
	struct thread *cur = thread_current();

	if(pml4_is_dirty(cur->pml4, page->va)){
		file_seek(file_page->file, file_page->ofs);
		file_write(file_page->file, page->va, file_page->size);
		pml4_set_dirty(cur->pml4, page->va, false);
	}

	pml4_clear_page(cur->pml4, page->va);
	page->frame = NULL;

	return true;
}
```
수정된 내용이 있으면 적용해주고 해당 메모리 관련된 내용들을 초기화해준다.

>evict

swap in/out의 동작들을 살펴봤고 그럼 이제는 어떤 page를 out시킬지에 대한 규칙을 정해야 한다.
```c
vm_evict_frame (void) {
	struct frame *victim UNUSED = vm_get_victim ();
	/* TODO: swap out the victim and return the evicted frame. */
	if(victim == NULL)
		return NULL;
	struct page *page = victim->page;
	bool swap_done = swap_out(page);
	if(!swap_done)
		PANIC("Swap is full\n");
	
	victim->page = NULL;
	memset(victim->kva, 0, PGSIZE);

	return victim;
}
```
swap_out을 진행하기 위해 위 함수를 실행한다. vm_get_victim으로 내보낼 frame을 설정하고 희생되는 frame 관련 정보들을 정리한다. swap을 제대로 못 했다면 swap공간이 모자라서 나타나는 현상으로 PANIC을 호출한다.

```c
static struct frame *
vm_get_victim (void) {
	struct frame *victim = NULL;
	 /* TODO: The policy for eviction is up to you. */
	struct thread *curr = thread_current();

	lock_acquire(&clock_lock);
	struct list_elem *cand_elem = clock_elem;
	if(cand_elem == NULL && !list_empty(&frame_list))
		cand_elem = list_front(&frame_list);
	while(cand_elem != NULL){
		victim = list_entry(cand_elem, struct frame, elem);
		if(!pml4_is_accessed(curr->pml4, victim->page->va))
			break;
		pml4_set_accessed(curr->pml4, victim->page->va, false);

		cand_elem = list_next_cycle(&frame_list, cand_elem);
	}

	clock_elem = list_next_cycle(&frame_list, cand_elem);
	list_remove(cand_elem);
	lock_release(&clock_lock);

	return victim;
}
```
희생자를 선정하기 위해 clock 알고리즘을 활용한다. 이전에 clock 알고리즘이 실행돼서 특정한 곳을 가리키고 있으면 그 부분부터 시작하고 그렇지 않으면 frame_list의 처음부터 시작한다. frame_list를 돌면서 access bit를 false로 바꿔주다가 이미 false인 frame을 희생자로 선정해준다. 희생자로 선정된 다음 위치를 기억하고, 희생자는 리스트에서 제거해준다.

# Copy-on-write (Extra)
프로세스가 fork 로 복제되면 부모의 resource 와 같은 것들을 활용하게 되는데 그러면 굳이 복사해줄 필요 없이 이미 존재하고 있는 resource를 활용하는 것이 효율적이다. 대신에 write 등으로 수정이 일어난다면 resource의 변화가 생긴 것이기 때문에 같은 것을 공유할 수 없고 이 때부터 분리된 상태로 관리하도록 해야 한다. 그래서 write-protect mechanism이 필요한데 실제 구현은 하지 못 했다.