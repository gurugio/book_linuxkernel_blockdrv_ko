# 램디스크 구현

이전장에서는 bio가 뭔지만 확인했습니다. 가상의 램디스크를 만들어서 bio가 어떻게 IO처리에 이용되는지 알아보겠습니다. 램디스크는 어플이 저장한 데이터를 램에 저장해놨다가, 다시 읽혀질 때 저장된 값을 반환하는 장치입니다. 하드디스크를 메모리로 흉내냈다고 보면 됩니다. 재부팅되면 데이터가 사라지는 것만 다릅니다.

참고로 /drivers/block/brd.c 파일을 보면 이미 램디스크 드라이버가 있습니다. 이 램디스크를 따라만든다고 생각하시면 됩니다.

소스: https://github.com/gurugio/mybrd/blob/ch03-ramdisk/mybrd.c

##radix tree를 이용해서 페이지 저장하기

###radix tree 자료구조 소개

참고: https://lwn.net/Articles/175432/

일반적인 radix tree 자료구조가 아니라 리눅스에서 구현된 radix tree를 기준으로 설명하겠습니다.

radix tree는 페이지를 저장하는 자료구조입니다. 드라이버에서 페이지를 할당하고, 이 페이지를 키 값을 지정해서 트리에 저장합니다. 그리고 나중에 키 값을 이용해서 다시 해당 페이지를 찾는 것이지요. 구현이나 좀더 다양한 활용법은 참고 링크를 확인하시고, 우리는 그냥 페이지를 저장했다가 찾는 간단한 기능만 사용하겠습니다.

###struct radix_tree_root

트리의 루트를 표현하는 구조체입니다. INIT_RADIX_TREE 매크로로 초기화합니다.

###radix_tree_preload()/_end() & radix_tree_insert()

radix-tree에 페이지를 넣는게 radix_tree_inset() 함수입니다. 사용법은 간단합니다. 루트 노트의 주소와 페이지의 키값, 페이지 포인터만 지정하면 됩니다. 키 값을 뭘로 정할지는 드라이버가 알아서하면 됩니다. 잠시후 보겠지만 우리는 섹터 번호를 키 값으로 사용합니다. 왜냐면 디스크에서 유니크한 값이 섹터 번호니까요.

그런데 radix_tree_preload()라는게 있습니다. 꼭 필요한건 아니지만 많이 사용되는 함수입니다. 참고 문서를 보시면 radix_tree_insert()가 실패하면 안되는 상황에서 사용한다고 설명하고 있습니다. 트리에 어떤 데이터를 넣으려면 트리 노드를 표현하는 객체를 만들어야됩니다. 따라서 메모리 할당을 해야합니다. radix_tree_preload()는 메모리 할당을 미리 해놓는 것입니다. 트리란건 사실 데이터와 키 값을 저장하는 자료구조중에 속도가 필요할 때 쓰는 것입니다. 그런데 트리에 데이터를 넣을때마다 메모리 할당을 한다면 속도도 느려지고, 트리에 대한 동기화 문제도 복잡해집니다. 메모리 할당은 메모리가 부족한 경우 프로세스를 잠들게 할 수 있습니다. 만약 락이 걸린 상태에서 잠든다면 데드락의 가능성도 생기고 문제가 많아집니다.

그래서 preload를 합니다. 만약 preload가 실패한다면 메모리가 부족한 상황이므로 트리에 접근을 안하고 그러면 동기화 문제가 없어집니다. preload가 성공한다면 메모리 부족으로 실패할 일은 없으니 크리티컬 섹션에서 프로세스가 잠들 일도 없습니다.

그래서 결론은 radix_tree_preload()를 호출한 후에 성공하면 radix_tree_insert()를 호출하도록 구현한다는 것입니다. radix_tree_insert() 다음에는 radix_tree_preload_end()를 반드시 호출해야합니다. 왜인지는 커널 소스를 보시면 아시게 됩니다.

참고로 커널에서 뭔가 이거다음에 반드시 이거를 호출해라라고 규정할 때가 자주 있습니다. 이런건 반드시 지켜야합니다. 대부분 동기화와 관련되있어서 문제가 발생할 경우 디버깅하기가 어렵습니다. 그리고 커널 문서에 그런 사항들이 모두 기록된게 아닙니다. 왜냐면 코드가 자주 바뀌는 부분은 문서 자체를 아예 안만드니까요. 주석으로 설명하는 것도 한계가 있습니다. 가장 좋은 방법은 다른 드라이버 코드를 참고하는 것입니다. 소스 태깅 툴로 radix_tree_preload()를 호출하는 다른 코드들을 확인해보세요. 어떤 커널 함수를 사용할 때는 그 함수를 이미 사용하는 다른 코드를 찾아보는게 가장 좋은 레퍼런스가 됩니다. 문서나 주석은 바뀐 코드를 반영하지 않을 수 있지만, 코드 자체는 항상 옳으니까요.

###radix_tree_lookup()

트리에서 특정 키 값을 갖는 페이지를 찾는 함수입니다. 사용법도 간단합니다. 루트와 키 값만 전달하면 페이지 포인터를 반환합니다.

###mybrd_lookup_page()

모든 IO는 섹터단위로 이루어집니다. 몇번 섹터에서 몇개의 섹터를 읽기/쓰기가 IO가 동작하는 방식입니다. 따라서 radix-tree의 키 값도 섹터가 되야합니다.

mybrd_lookup_page()는 트리에서 특정 섹터의 데이터를 가지고 있는 페이지를 찾는 함수입니다. 함수 인자로 섹터 번호를 지정하면 그 섹터를 포함하고 있는 페이지를 반환합니다.

함수 내부를 보겠습니다. 먼저 rcu_read_lock()이 보입니다. RCU lock에 대해서는 다음 문서로 설명을 대신하겠습니다.

https://lwn.net/Articles/262464/

rcu_read_lock()의 핵심은 트리를 읽는 쓰레드는 배타적으로 실행될 필요없이 동시에 실행될 수 있다는 것입니다. 트리에서 페이지를 찾는 동작은 트리를 읽기만하지 뭔가를 바꾸지 않습니다. 따라서 rcu_read_lock()를 사용할 수 있습니다.

그 다음은 섹터 번호를 가지고 키 값을 만드는 것입니다. 하나의 페이지에는 4096바이트이고 섹터는 512바이트이므로 총 8개의 섹터가 저장될 수 있는데 이중에 어떤 섹터 번호를 키 값으로 할거냐가 문제가 됩니다. 우리는 항상 페이지에 저장되는 최초의 섹터 번호를 키값을 정하겠습니다. 따라서 (섹터번호*512/4096)라는 공식이 생성됩니다.

예로 몇개의 (섹터 키) 값의 쌍을 한번 계산해보겠습니다.
```
0 0

7 0

8 1

9 1

16 2
```
이걸 비트 연산 코드로 만들면 sector >> (12 - 9) 가 됩니다. 그런데 페이지 크기는 플랫폼마다 다를 수 있습니다. 커널에서는 플랫폼마다 다른 페이지 크기를 매크로로 정의하고 있습니다. PAGE_SIZE, PAGE_SHIFT등의 매크로를 사용하면 플랫폼 상관없이 페이지 크기를 확인할 수 있습니다. 따라서 최종 코드는 sector >> (PAGE_SHIFT - 9)가 됩니다.

키 값을 계산했으면 radix_tree_lookup()을 호출해서 페이지를 찾습니다. 페이지가 없을 수 있습니다. 아직 해당 섹터에 데이터를 안쓴것입니다.

###mybrd_insert_page()

트리에 페이지를 추가합니다. 가장 먼저 이미 해당 섹터가 트리에 있는지를 확인합니다. 트리에 찾고자하는 페이지가 없다면 먼저 페이지를 할당합니다. 

페이지 할당에서 중요한 사항이 바로 할당 플래그입니다. 우리는 디스크 IO를 처리하는 드라이버를 만들고있으므로 GFP_NOIO 플래그를 사용해야합니다. 그 이유는 김민찬님의 강좌에서 잘 설명하고 있습니다.

https://www.linux.co.kr/home2/board/subbs/board.php?bo_table=lecture&wr_id=1641

GFP_NOIO와 GFP_NOFS는 현 재 메모리가 모자라 회수를 해야 되는 상황인데 함수를 호출한 context가 block I/O code라던가 파일시스템 코드 일경우 사용되게 된다. 예를 들면, block I/O의 코드 중에 buffer_head를 할당하는 함수가 있다. 그 경우, 커널이 buffer_head를 할당하려다가 메모리가 모자라다는 것을 알아 차렸다면 페이지 회수 알고리즘이 동작하기 시작하게 된다. 이때 메모리를 회수하기 위해 page cache에 있는 페이지들중 dirty 페이지를 하드디스크에 sync시켜 버리고 물리메모리에서 제거하면 그 메모리는 free 메모리로 사용할 수 있게 된다. 페이지를 sync하기 위해서 즉 하드디스크에 write하기위해서는 I/O를 수행하여야 하며 그러기 위해선 또 다시buffer_head를 할당해야만 하는 웃지 못할 일이 발생한다. 이런 문제들을 방지하기 위하여 있는 플래그이다.

간단하게 설명하면 GFP_NOIO 플래그를 지정하지않으면 드라이버에서 메모리를 할당하려고했는데 돌고돌아서 다시 드라이버가 호출될 수 있다는 것입니다. 결국 또 메모리 할당 함수가 호출되고 무한 루프가 될 수 있습니다.

그리고 이전에 설명한대로 preload()를 실행하고 spin-lock을 잡습니다.

그 다음은 섹터 번호로 키 값을 만듭니다. 그리고 page 구조체의 index필드에 키값을 저장하고 radix_tree_insert()함수를 이용해서 트리에 페이지를 추가하면 됩니다.

##사용자 어플과 커널간의 데이터 이동

### copy_from_user_to_mybrd()

사용자로부터 커널을 통해 전달된 데이터를 램디스크에 쓰는 함수입니다.

함수 인자는

* src_page: 데이터가 저장된 페이지
* len: 데이터의 크기
* src_offset: 페이지 안에 어디서부터 데이터가 시작하는지를 알려주는 offset
* sector: 커널이 요청한 섹터 번호

섹터 번호를 알고있으니 트리에서 해당 섹터를 포함하는 페이지를 찾아올 수 있습니다. 그런데 그 페이지 전체의 데이터를 사용할게 아니지요. 그 중에 요청된 섹터만 필요합니다. 따라서 섹터 번호를 가지고 트리에서 찾아낸 페이지 안에 어디부터가 우리가 쓸 데이터인지를 알아내야합니다. 그게 바로 target_offset입니다. sector & (8-1)은 섹터 번호를 8로 나눈 나머지를 의미합니다. 한 페이지에 8개의 섹터가 들어가니까요. 이렇게하면 페이지 안에 섹터의 위치를 알 수 있고 여기에 512를 곱하면 offset이 됩니다.

그 다음은 얼마만큼 복사할지 copy 변수에 저장합니다. 하나의 bio가 가질 수 있는 데이터의 크기는 4096바이트입니다. 우리가 디스크의 블럭 사이즈를 4096으로 지정했기 때문입니다. 세그먼트는 블럭 내에 저장되는 단위이므로 한 블럭의 크기를 초과할 수 없고, 결국 한 페이지보다 클 수 없습니다. 하지만 만약 offset이 2048이고 총 복사해야할 길이가 4096이라면 두개의 페이지를 찾아야합니다. 따라서 copy값을 계산할 때 len값을 그대로 쓸지, 한 페이지에서 복사할 크기만을 쓸지를 선택합니다.

그 다음은 요청된 섹터를 갖는 페이지를 찾습니다. 페이지가 없을 수 있습니다. 처음 쓰기가 발생한 섹터라면 트리에 페이지가 없겠지요. 그럴때는 트리에 페이지를 추가해주면 됩니다. 이때도 마찬가지로 2개의 페이지에 걸쳐서 데이터가 있을 수 있으므로 2개의 페이지를 추가할 수 있습니다.

이제 페이지에 있는 데이터를 복사해와야합니다. 커널이 전달한 페이지는 src_page이고 우리가 할당한 페이지는 dst_page이므로 각각 kmap()함수를 써서 메모리 주소를 알아낸 다음 메모리 복사를 하면 됩니다.

###copy_from_mybrd_to_user()

램디스크의 데이터를 사용자에게 보내주는 함수입니다. 커널이 이미 어떤 페이지의 어떤 위치에 데이터를 저장하라고 지정해줬기 때문에 드라이버는 자신이 가진 데이터를 정해진 위치에 복사하기만하면 됩니다.

그런데 처음 접근되는 섹터라면 해당되는 페이지가 없을 수 있습니다. 그럴때는 데이터가 없는 영역이므로 0으로된 데이터를 보냅니다. 나중에 해당 섹터에 쓰기가 발생되면 트리에 데이터를 추가합니다.

##bio를 읽고 IO 실행

###mybrd_make_request_fn에 데이터 처리 추가

이제 사용할 함수들이 다 만들어졌으니 mybrd_make_request_fn()에서 호출하기만하면 됩니다. bio_for_each_segment() 루프안에서 읽기일때는 copy_from_mybrd_to_user()를 호출하고 쓰기일때는 copy_from_user_to_mybrd()를 호출하면 됩니다.

한가지 주의할게 루프내에서 섹터 번호를 증가시켜야 한다는 것입니다. 섹터 번호는 bio구조체에 첫번째 섹터번호만 전달됩니다. 따라서 데이터를 처리한 후에는 처리된 데이터만큼 섹터 번호를 증가시켜야합니다. 데이터 크기를 512로 나누면 섹터 크기가 되겠지요.

###데이터 캐시 처리

여기서 한가지 매우 중요한 사항이 있습니다. 눈에 잘 보이지는 않지만 만약 잊어버렸을경우 심각하면서도 찾기 힘든 버그가 생길 수 있는 것인데 바로 캐시에 대한 처리입니다. 김민찬님의 글을 보면 잘 설명이 되어있습니다.

http://barriosstory.blogspot.de/2009/01/flushdcachepage-kmapatomic.html

우리 드라이버는 사용자가 요청한 데이터를 읽고 씁니다. 따라서 데이터를 읽고 써야할 타겟 페이지는 사용자의 주소 공간에 맵핑된 페이지입니다. 원래 사용자가 사용하는 페이지인데 커널이 그 페이지를 커널의 주소 공간에 맵핑한다음 드라이버에 데이터를 요청하는 것입니다. 따라서 커널이 그냥 데이터를 쓰기만하면 메모리에는 데이터가 복사될 수 있어도 프로세서의 캐시에는 데이터가 업데이트되지 않을 수 있습니다. 그러므로 캐시에 있는 데이터를 무효화하는 과정이 필요합니다.

읽기 요청일때는 드라이버가 가진 데이터를 사용자 페이지로 복사합니다. 드라이버는 이미 커널 영역의 페이지에 데이터를 가지고 있으므로 드라이버가 자신의 페이지에 접근할 때는 캐시 무효화가 필요없습니다. 하지만 데이터 복사가 끝난 뒤에는 캐시 무효화를 해줘서 사용자가 페이지에 접근할 때 메모리로부터 다시 캐시로 데이터가 전송되도록 만듭니다.

반대로 쓰기 요청일때는 커널에 사용자의 페이지에 접근해야합니다. 커널이 페이지를 읽기전에 캐시를 무효화해놓고 메모리를 읽어야합니다. 따라서 copy_from_user_to_mybrd()를 호출하기전에 캐시 무효화함수를 호출합니다.

캐시관리가 좀 어려운 이유가 하드웨어 플랫폼에따라 VIVT니 VIPT같은 캐시 정책이 다르기 때문입니다. 이런 캐시 무효화도 플랫폼에 따라 필요없을 수 있습니다. 어쨌든 커널은 모든 플랫폼에서 사용될 수 있는 함수들을 제공하고 있으므로, 플랫폼에 상관없이 캐시 처리를 해준다면 커널이 플랫폼에 따라서 실제로 뭔가를 할 수도 있고 안할 수도 있습니다.
##데이터 읽기 쓰기 실험

###bio 확인 및 데이터 저장 확인

bio 실험에서도 쓰기가 좀더 간단하다는걸 알았습니다. 먼저 쓰기가 잘 되는지를 확인해보겠습니다.
```
/ # dd if=/dev/urandom of=/dev/mybrd bs=4096 count=2
[   56.168167] random: dd urandom read with 52 bits of entropy available
[   56.169718] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[   56.170757] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
[   56.171345] mybrd: segment-info: len=4096 p=ffffea0000005340 offset=0
[   56.171992] mybrd: lookup: page-          (null) index--1 sector-0
[   56.172631] mybrd: lookup: page-          (null) index--1 sector-0
[   56.173274] mybrd: insert: page-ffffea0000198740 index=0 sector-0
[   56.174157] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[   56.174780] mybrd: copy: ffff88000661d000 <- ffff88000014d000 (4096-bytes)
[   56.175493] mybrd: d7 22 c2 77 c3 dc ec ed
[   56.175913] mybrd: d7 22 c2 77 c3 dc ec ed
[   56.176353] mybrd: end mybrd_make_request_fn
[   56.176791] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[   56.177729] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[   56.178326] mybrd: segment-info: len=4096 p=ffffea00001988c0 offset=0
[   56.179001] mybrd: lookup: page-          (null) index--1 sector-8
[   56.179666] mybrd: lookup: page-          (null) index--1 sector-8
[   56.180456] mybrd: insert: page-ffffea0000196900 index=1 sector-8
[   56.181155] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[   56.181836] mybrd: copy: ffff8800065a4000 <- ffff880006623000 (4096-bytes)
[   56.182535] mybrd: 3f 3 a0 b9 38 8e a4 d
[   56.182833] mybrd: 3f 3 a0 b9 38 8e a4 d
[   56.183148] mybrd: end mybrd_make_request_fn
2+0 records in
2+0 records out
8192 bytes (8.0[   56.183787] dd (1048) used greatest stack depth: 13832 bytes left
KB) copied, 0.015329 seconds, 521.9KB/s
```
2개의 write용 bio가 발생했습니다. 섹터는 0번과 8번입니다. 각각 8개의 섹터니까 4096바이트 크기네요. 세그먼트 정보와도 일치합니다. 섹터 0에 해당하는 페이지를 찾아봤지만 실패했다는게 나왔고 그래서 새로운 페이지를 할당해서 트리에 추가했고, 그걸 다시 읽어와서 데이터를 복사했다는걸 알 수 있습니다. 데이터를 /dev/urandom에서 읽어왔습니다. 이 장치는 난수를 발생시키는 장치입니다. 커널 로그에도 d7 22 등등 난수들이 저장된걸 확인할 수 있습니다.

이제 데이터가 잘 저장된건가 확인하기위해 파일을 읽어보겠습니다.
```
/ # dd if=/dev/mybrd of=/dev/null bs=4096 count=1
[  316.710833] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  316.711772] mybrd: bio-info: sector=0 end_sector=32 rw=READ
[  316.712281] mybrd: segment-info: len=4096 p=ffffea0000004b40 offset=0
[  316.712833] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  316.713373] mybrd: copy: ffff88000012d000 <- ffff88000661d000 (4096-bytes)
[  316.713963] mybrd: d7 22 c2 77 c3 dc ec ed
[  316.714339] mybrd: d7 22 c2 77 c3 dc ec ed
[  316.714702] mybrd: segment-info: len=4096 p=ffffea0000005300 offset=0
[  316.715270] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[  316.715791] mybrd: copy: ffff88000014c000 <- ffff8800065a4000 (4096-bytes)
[  316.716400] mybrd: 3f 3 a0 b9 38 8e a4 d
[  316.716740] mybrd: 3f 3 a0 b9 38 8e a4 d
[  316.717101] mybrd: segment-info: len=4096 p=ffffea0000005340 offset=0
[  316.717659] mybrd: lookup: page-          (null) index--1 sector-16
[  316.718212] mybrd: copy: ffff88000014d000 <- 0 (4096-bytes)
[  316.718696] mybrd: 0 0 0 0 0 0 0 0
[  316.718992] mybrd: segment-info: len=4096 p=ffffea0000005380 offset=0
[  316.719561] mybrd: lookup: page-          (null) index--1 sector-24
[  316.720111] mybrd: copy: ffff88000014e000 <- 0 (4096-bytes)
[  316.720598] mybrd: 0 0 0 0 0 0 0 0
[  316.720895] mybrd: end mybrd_make_request_fn
1+0 records in
1+0 records out
4096 bytes (4.0KB) copied, 0.010485 seconds, 381.5KB/s
```
1개 페이지만 읽어봤는데 이전과 마찬가지로 read-ahead가 실행되서 32개의 섹터를 읽어들이네요. 0번 섹터에 해당하는 페이지를 읽었고 써진 값이 d7 22 등 이전에 저장한 값과 같습니다. 써진 값을 제대로 읽어왔네요. 다음 페이지도 써진 값이 다시 읽혀진걸 확인할 수 있습니다. 그 다음 페이지들은 아직 데이터가 없는 페이지이므로 0으로 반환되었습니다.

###파일시스템 생성 실험

그냥 dd만 가지고 실험하면 좀 심심하니까 본격적으로 mybrd를 진짜 디스크라고 생각하고 파일시스템을 생성해보겠습니다. 방법은 간단합니다. mkfs.vfat를 쓰면 간단하게 vfat 파일시스템을 생성할 수 있습니다.
```
/ # mkfs.vfat /dev/mybrd 
[  449.918935] mybrd: start mybrd_ioctl
[  449.919710] mybrd: end mybrd_ioctl
mkfs.vfat: for t[  449.920526] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
his device secto[  449.922345] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
r size is 4096
[  449.923659] mybrd: segment-info: len=4096 p=ffffea00001a1d40 offset=0
[  449.925387] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  449.926476] mybrd: copy: ffff88000661d000 <- ffff880006875000 (4096-bytes)
[  449.927712] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.928443] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.929119] mybrd: end mybrd_make_request_fn
[  449.929821] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.931139] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[  449.931984] mybrd: segment-info: len=4096 p=ffffea00001a7b80 offset=0
[  449.932824] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[  449.933496] mybrd: copy: ffff8800065a4000 <- ffff8800069ee000 (4096-bytes)
[  449.934241] mybrd: 52 52 61 41 0 0 0 0
[  449.934731] mybrd: 52 52 61 41 0 0 0 0
[  449.935031] mybrd: end mybrd_make_request_fn
[  449.935373] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.936096] mybrd: bio-info: sector=16 end_sector=24 rw=WRITE
[  449.936557] mybrd: segment-info: len=4096 p=ffffea00001a3700 offset=0
[  449.937062] mybrd: lookup: page-          (null) index--1 sector-16
[  449.937586] mybrd: lookup: page-          (null) index--1 sector-16
[  449.938085] mybrd: insert: page-ffffea00001abf80 index=2 sector-16
[  449.938580] mybrd: lookup: page-ffffea00001abf80 index-2 sector-16
[  449.939067] mybrd: copy: ffff880006afe000 <- ffff8800068dc000 (4096-bytes)
[  449.939605] mybrd: 0 0 0 0 0 0 0 0
[  449.939879] mybrd: 0 0 0 0 0 0 0 0
[  449.940149] mybrd: end mybrd_make_request_fn
[  449.940490] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.941199] mybrd: bio-info: sector=24 end_sector=32 rw=WRITE
[  449.941650] mybrd: segment-info: len=4096 p=ffffea00001af8c0 offset=0
[  449.942152] mybrd: lookup: page-          (null) index--1 sector-24
[  449.942654] mybrd: lookup: page-          (null) index--1 sector-24
[  449.943150] mybrd: insert: page-ffffea00001902c0 index=3 sector-24
[  449.943599] mybrd: lookup: page-ffffea00001902c0 index-3 sector-24
[  449.944123] mybrd: copy: ffff88000640b000 <- ffff880006be3000 (4096-bytes)
[  449.944619] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.944905] mybrd: eb 58 90 6d 6b 64 6f 73
[  449.945290] mybrd: end mybrd_make_request_fn
[  449.945654] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.946371] mybrd: bio-info: sector=32 end_sector=40 rw=WRITE
[  449.946832] mybrd: segment-info: len=4096 p=ffffea0000190240 offset=0
[  449.947320] mybrd: lookup: page-          (null) index--1 sector-32
[  449.947816] mybrd: lookup: page-          (null) index--1 sector-32
[  449.948311] mybrd: insert: page-ffffea0000195540 index=4 sector-32
[  449.948793] mybrd: lookup: page-ffffea0000195540 index-4 sector-32
[  449.949313] mybrd: copy: ffff880006555000 <- ffff880006409000 (4096-bytes)
[  449.949891] mybrd: 52 52 61 41 0 0 0 0
[  449.950180] mybrd: 52 52 61 41 0 0 0 0
[  449.950475] mybrd: end mybrd_make_request_fn
[  449.950815] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.951515] mybrd: bio-info: sector=40 end_sector=48 rw=WRITE
[  449.951961] mybrd: segment-info: len=4096 p=ffffea0000196980 offset=0
[  449.952454] mybrd: lookup: page-          (null) index--1 sector-40
[  449.952936] mybrd: lookup: page-          (null) index--1 sector-40
[  449.953426] mybrd: insert: page-ffffea00001af880 index=5 sector-40
[  449.954024] mybrd: lookup: page-ffffea00001af880 index-5 sector-40
[  449.954523] mybrd: copy: ffff880006be2000 <- ffff8800065a6000 (4096-bytes)
[  449.955044] mybrd: 0 0 0 0 0 0 0 0
[  449.955303] mybrd: 0 0 0 0 0 0 0 0
[  449.955568] mybrd: end mybrd_make_request_fn
[  449.955897] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.956591] mybrd: bio-info: sector=48 end_sector=56 rw=WRITE
[  449.957028] mybrd: segment-info: len=4096 p=ffffea0000198140 offset=0
[  449.957519] mybrd: lookup: page-          (null) index--1 sector-48
[  449.958053] mybrd: lookup: page-          (null) index--1 sector-48
[  449.958659] mybrd: insert: page-ffffea00001a1dc0 index=6 sector-48
[  449.959227] mybrd: lookup: page-ffffea00001a1dc0 index-6 sector-48
[  449.959676] mybrd: copy: ffff880006877000 <- ffff880006605000 (4096-bytes)
[  449.960163] mybrd: f0 ff ff f ff ff ff ff
[  449.960452] mybrd: f0 ff ff f ff ff ff ff
[  449.960744] mybrd: end mybrd_make_request_fn
[  449.961056] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.961705] mybrd: bio-info: sector=56 end_sector=64 rw=WRITE
[  449.962120] mybrd: segment-info: len=4096 p=ffffea0000196480 offset=0
[  449.962582] mybrd: lookup: page-          (null) index--1 sector-56
[  449.963034] mybrd: lookup: page-          (null) index--1 sector-56
[  449.963602] mybrd: insert: page-ffffea00001955c0 index=7 sector-56
[  449.964043] mybrd: lookup: page-ffffea00001955c0 index-7 sector-56
[  449.964487] mybrd: copy: ffff880006557000 <- ffff880006592000 (4096-bytes)
[  449.964980] mybrd: 0 0 0 0 0 0 0 0
[  449.965228] mybrd: 0 0 0 0 0 0 0 0
[  449.965476] mybrd: end mybrd_make_request_fn
[  449.965782] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.966430] mybrd: bio-info: sector=64 end_sector=72 rw=WRITE
[  449.966842] mybrd: segment-info: len=4096 p=ffffea0000198080 offset=0
[  449.967300] mybrd: lookup: page-          (null) index--1 sector-64
[  449.967750] mybrd: lookup: page-          (null) index--1 sector-64
[  449.968197] mybrd: insert: page-ffffea00001987c0 index=8 sector-64
[  449.968637] mybrd: lookup: page-ffffea00001987c0 index-8 sector-64
[  449.969073] mybrd: copy: ffff88000661f000 <- ffff880006602000 (4096-bytes)
[  449.969568] mybrd: 0 0 0 0 0 0 0 0
[  449.969817] mybrd: 0 0 0 0 0 0 0 0
[  449.970064] mybrd: end mybrd_make_request_fn
[  449.970368] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.971016] mybrd: bio-info: sector=72 end_sector=80 rw=WRITE
[  449.971414] mybrd: segment-info: len=4096 p=ffffea0000196fc0 offset=0
[  449.971860] mybrd: lookup: page-          (null) index--1 sector-72
[  449.972314] mybrd: lookup: page-          (null) index--1 sector-72
[  449.972746] mybrd: insert: page-ffffea0000198800 index=9 sector-72
[  449.973276] mybrd: lookup: page-ffffea0000198800 index-9 sector-72
[  449.973695] mybrd: copy: ffff880006620000 <- ffff8800065bf000 (4096-bytes)
[  449.974162] mybrd: 0 0 0 0 0 0 0 0
[  449.974398] mybrd: 0 0 0 0 0 0 0 0
[  449.974634] mybrd: end mybrd_make_request_fn
[  449.974935] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.975531] mybrd: bio-info: sector=80 end_sector=88 rw=WRITE
[  449.975879] mybrd: segment-info: len=4096 p=ffffea00001de9c0 offset=0
[  449.976330] mybrd: lookup: page-          (null) index--1 sector-80
[  449.976814] mybrd: lookup: page-          (null) index--1 sector-80
[  449.977320] mybrd: insert: page-ffffea0000198300 index=10 sector-80
[  449.977745] mybrd: lookup: page-ffffea0000198300 index-10 sector-80
[  449.978185] mybrd: copy: ffff88000660c000 <- ffff8800077a7000 (4096-bytes)
[  449.978711] mybrd: f0 ff ff f ff ff ff ff
[  449.979069] mybrd: f0 ff ff f ff ff ff ff
[  449.979359] mybrd: end mybrd_make_request_fn
[  449.979693] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.980305] mybrd: bio-info: sector=88 end_sector=96 rw=WRITE
[  449.980721] mybrd: segment-info: len=4096 p=ffffea00001a44c0 offset=0
[  449.981190] mybrd: lookup: page-          (null) index--1 sector-88
[  449.981647] mybrd: lookup: page-          (null) index--1 sector-88
[  449.982113] mybrd: insert: page-ffffea00001988c0 index=11 sector-88
[  449.982739] mybrd: lookup: page-ffffea00001988c0 index-11 sector-88
[  449.983186] mybrd: copy: ffff880006623000 <- ffff880006913000 (4096-bytes)
[  449.983654] mybrd: 0 0 0 0 0 0 0 0
[  449.983894] mybrd: 0 0 0 0 0 0 0 0
[  449.984153] mybrd: end mybrd_make_request_fn
[  449.984499] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.985157] mybrd: bio-info: sector=96 end_sector=104 rw=WRITE
[  449.985567] mybrd: segment-info: len=4096 p=ffffea00001dbc40 offset=0
[  449.986018] mybrd: lookup: page-          (null) index--1 sector-96
[  449.986459] mybrd: lookup: page-          (null) index--1 sector-96
[  449.986965] mybrd: insert: page-ffffea00001dad40 index=12 sector-96
[  449.987478] mybrd: lookup: page-ffffea00001dad40 index-12 sector-96
[  449.987904] mybrd: copy: ffff8800076b5000 <- ffff8800076f1000 (4096-bytes)
[  449.988375] mybrd: 0 0 0 0 0 0 0 0
[  449.988611] mybrd: 0 0 0 0 0 0 0 0
[  449.988881] mybrd: end mybrd_make_request_fn
[  449.989179] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.989797] mybrd: bio-info: sector=104 end_sector=112 rw=WRITE
[  449.990220] mybrd: segment-info: len=4096 p=ffffea00001d8700 offset=0
[  449.990684] mybrd: lookup: page-          (null) index--1 sector-104
[  449.991119] mybrd: lookup: page-          (null) index--1 sector-104
[  449.991557] mybrd: insert: page-ffffea00001dd400 index=13 sector-104
[  449.992019] mybrd: lookup: page-ffffea00001dd400 index-13 sector-104
[  449.992547] mybrd: copy: ffff880007750000 <- ffff88000761c000 (4096-bytes)
[  449.993018] mybrd: 0 0 0 0 0 0 0 0
[  449.993273] mybrd: 0 0 0 0 0 0 0 0
[  449.993511] mybrd: end mybrd_make_request_fn
[  449.993807] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  449.994423] mybrd: bio-info: sector=112 end_sector=120 rw=WRITE
[  449.994832] mybrd: segment-info: len=4096 p=ffffea00001dff40 offset=0
[  449.995271] mybrd: lookup: page-          (null) index--1 sector-112
[  449.995706] mybrd: lookup: page-          (null) index--1 sector-112
[  449.996156] mybrd: insert: page-ffffea0000001900 index=14 sector-112
[  449.996591] mybrd: lookup: page-ffffea0000001900 index-14 sector-112
[  449.997057] mybrd: copy: ffff880000064000 <- ffff8800077fd000 (4096-bytes)
[  449.997524] mybrd: 0 0 0 0 0 0 0 0
[  449.997760] mybrd: 0 0 0 0 0 0 0 0
[  449.997995] mybrd: end mybrd_make_request_fn
/ # mount /[  453.806303] random: nonblocking pool is initialized
dev/mybrd ./mnt
[  459.351005] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.352306] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.353042] mybrd: segment-info: len=4096 p=ffffea00001dfd00 offset=0
[  459.353898] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.354478] mybrd: copy: ffff8800077f4000 <- ffff88000661d000 (4096-bytes)
[  459.354939] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.355230] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.355485] mybrd: end mybrd_make_request_fn
[  459.355765] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.356347] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.356680] mybrd: segment-info: len=4096 p=ffffea0000001940 offset=0
[  459.357107] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.357486] mybrd: copy: ffff880000065000 <- ffff88000661d000 (4096-bytes)
[  459.357907] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.358177] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.358436] mybrd: end mybrd_make_request_fn
[  459.358712] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.359314] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.359647] mybrd: segment-info: len=4096 p=ffffea0000001940 offset=0
[  459.360033] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.360405] mybrd: copy: ffff880000065000 <- ffff88000661d000 (4096-bytes)
[  459.360819] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.361067] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.361327] mybrd: end mybrd_make_request_fn
[  459.361668] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.362466] mybrd: bio-info: sector=0 end_sector=8 rw=READ
[  459.362841] mybrd: segment-info: len=4096 p=ffffea0000008100 offset=0
[  459.363316] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.363821] mybrd: copy: ffff880000204000 <- ffff88000661d000 (4096-bytes)
[  459.364423] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.364708] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.364990] mybrd: end mybrd_make_request_fn
[  459.365291] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.365891] mybrd: bio-info: sector=8 end_sector=16 rw=READ
[  459.366287] mybrd: segment-info: len=4096 p=ffffea0000008140 offset=0
[  459.366734] mybrd: lookup: page-ffffea0000196900 index-1 sector-8
[  459.367210] mybrd: copy: ffff880000205000 <- ffff8800065a4000 (4096-bytes)
[  459.367681] mybrd: 52 52 61 41 0 0 0 0
[  459.367938] mybrd: 52 52 61 41 0 0 0 0
[  459.368191] mybrd: end mybrd_make_request_fn
[  459.368501] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.369152] mybrd: bio-info: sector=48 end_sector=56 rw=READ
[  459.369550] mybrd: segment-info: len=4096 p=ffffea0000008180 offset=0
[  459.369992] mybrd: lookup: page-ffffea00001a1dc0 index-6 sector-48
[  459.370456] mybrd: copy: ffff880000206000 <- ffff880006877000 (4096-bytes)
[  459.370935] mybrd: f0 ff ff f ff ff ff ff
[  459.371211] mybrd: f0 ff ff f ff ff ff ff
[  459.371500] mybrd: end mybrd_make_request_fn
[  459.371839] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.372486] mybrd: bio-info: sector=112 end_sector=120 rw=READ
[  459.372934] mybrd: segment-info: len=4096 p=ffffea00000081c0 offset=0
[  459.373377] mybrd: lookup: page-ffffea0000001900 index-14 sector-112
[  459.373842] mybrd: copy: ffff880000207000 <- ffff880000064000 (4096-bytes)
[  459.374327] mybrd: 0 0 0 0 0 0 0 0
[  459.374570] mybrd: 0 0 0 0 0 0 0 0
[  459.374810] mybrd: end mybrd_make_request_fn
[  459.375118] mybrd: start mybrd_make_request_fn: block_device=ffff88000619c340 mybrd=ffff8800069daa40
[  459.375745] mybrd: bio-info: sector=0 end_sector=8 rw=WRITE
[  459.376106] mybrd: segment-info: len=4096 p=ffffea0000008100 offset=0
[  459.376505] mybrd: lookup: page-ffffea0000198740 index-0 sector-0
[  459.377127] mybrd: copy: ffff88000661d000 <- ffff880000204000 (4096-bytes)
[  459.377757] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.378095] mybrd: eb 58 90 6d 6b 64 6f 73
[  459.378424] mybrd: end mybrd_make_request_fn
/ # mount
rootfs on / type rootfs (rw,size=53748k,nr_inodes=13437)
none on /proc type proc (rw,relatime)
none on /sys type sysfs (rw,relatime)
/dev/mybrd on /mnt type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro)
```

좀 깁니다. 그만큼 데이터를 많이 읽고쓰고 있네요. mount 명령으로 잘 마운트가 됩니다. 파일시스템이 생겼으니 당연히 파일을 한번 만들어봐야지요.
```
/ # cd mnt
/mnt # ls
/mnt # touch ddd
/mnt # cat > ddd
asdf
asdf
/mnt # 
```
파일이 생기고 파일에 데이터도 썼는데 드라이버가 아무런 동작도 하지 않습니다. 왜 그럴까요? 파일데이터가 바로 파일에 써지는게 아님을 알 수 있습니다. 우리가 파일에 데이터를 쓰면 파일시스템은 데이터를 메모리에 저장합니다. 이걸 버퍼 캐시라고 합니다.

데이터가 메모리에 저장되므로 드라이버에는 아직 데이터를 쓰라는 명령이 안온겁니다. 그럼 메모리의 데이터를 디스크에 저장하는 sync 프로그램을 실행해보겠습니다.
```
/mnt # sync
[  742.641735] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.644677] mybrd: bio-info: sector=48 end_sector=56 rw=WRITE
[  742.646476] mybrd: segment-info: len=4096 p=ffffea00001ff140 offset=0
[  742.648129] mybrd: lookup: page-ffffea0000198500 index-6 sector-48
[  742.649640] mybrd: copy: ffff880006614000 <- ffff880007fc5000 (4096-bytes)
[  742.651147] mybrd: f0 ff ff f ff ff ff ff
[  742.652050] mybrd: f0 ff ff f ff ff ff ff
[  742.653033] mybrd: end mybrd_make_request_fn
[  742.654187] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.656014] mybrd: bio-info: sector=80 end_sector=88 rw=WRITE
[  742.657160] mybrd: segment-info: len=4096 p=ffffea0000008400 offset=0
[  742.658498] mybrd: lookup: page-ffffea0000198600 index-10 sector-80
[  742.659742] mybrd: copy: ffff880006618000 <- ffff880000210000 (4096-bytes)
[  742.661136] mybrd: f0 ff ff f ff ff ff ff
[  742.661932] mybrd: f0 ff ff f ff ff ff ff
[  742.662620] mybrd: end mybrd_make_request_fn
[  742.663187] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.664410] mybrd: bio-info: sector=112 end_sector=120 rw=WRITE
[  742.665205] mybrd: segment-info: len=4096 p=ffffea0000062700 offset=0
[  742.666068] mybrd: lookup: page-ffffea0000198700 index-14 sector-112
[  742.666858] mybrd: copy: ffff88000661c000 <- ffff88000189c000 (4096-bytes)
[  742.667724] mybrd: 41 64 0 64 0 64 0 0
[  742.668159] mybrd: 41 64 0 64 0 64 0 0
[  742.668678] mybrd: end mybrd_make_request_fn
[  742.669284] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.670541] mybrd: bio-info: sector=120 end_sector=128 rw=WRITE
[  742.671346] mybrd: segment-info: len=4096 p=ffffea00000081c0 offset=0
[  742.672218] mybrd: lookup: page-ffffea0000198740 index-15 sector-120
[  742.673046] mybrd: copy: ffff88000661d000 <- ffff880000207000 (4096-bytes)
[  742.673962] mybrd: 61 73 64 66 a 61 73 64
[  742.674514] mybrd: 61 73 64 66 a 61 73 64
[  742.675022] mybrd: end mybrd_make_request_fn
[  742.675591] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.676769] mybrd: bio-info: sector=8 end_sector=16 rw=WRITE
[  742.677538] mybrd: segment-info: len=4096 p=ffffea00000778c0 offset=0
[  742.678383] mybrd: lookup: page-ffffea0000001cc0 index-1 sector-8
[  742.679156] mybrd: copy: ffff880000073000 <- ffff880001de3000 (4096-bytes)
[  742.680024] mybrd: 52 52 61 41 0 0 0 0
[  742.680516] mybrd: 52 52 61 41 0 0 0 0
[  742.681004] mybrd: end mybrd_make_request_fn
[  742.681545] mybrd: start mybrd_make_request_fn: block_device=ffff880006c0b740 mybrd=ffff88000771ce40
[  742.682698] mybrd: bio-info: sector=112 end_sector=120 rw=WRITE
[  742.683450] mybrd: segment-info: len=4096 p=ffffea0000062700 offset=0
[  742.684271] mybrd: lookup: page-ffffea0000198700 index-14 sector-112
[  742.685103] mybrd: copy: ffff88000661c000 <- ffff88000189c000 (4096-bytes)
[  742.685970] mybrd: 41 64 0 64 0 64 0 0
[  742.686459] mybrd: 41 64 0 64 0 64 0 0
[  742.686908] mybrd: end mybrd_make_request_fn
/mnt # ls
ddd
/mnt # cat ddd
asdf
asdf
```
이제야 데이터가 드라이버에 전달됩니다. 파일에 쓴 데이터는 잘 전달됐을까요? asdf라고 썼으니 각각 아스키코드가 0x61 0x73 0x64 0x66입니다. 커널 로그 중간에 같은 값들이 전달된게 보이네요. 그 다음 값이 0xa인걸보니 개행문자까지도 저장된게 보입니다. asdf를 두줄썼으니까 또 0x61 0x73 0x64가 보이네요.

마지막으로 통계 정보를 확인해보겠습니다.
```
/ # cat /sys/block/mybrd/stat
      10        0      104       39       18        0      144       91        0       93       93
```