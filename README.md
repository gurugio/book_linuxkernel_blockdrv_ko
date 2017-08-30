# v4.4 커널에서 블럭 디바이스 드라이버 만들기

최근..은 아니고 몇년전 리눅스 커널에 큰 변화가 있었습니다. 커널에서 가장 성능에 민감한 부분이 메모리 할당과 블럭 장치인데요, 블럭 장치 관련 코드에 대규모 패치가 들어갔습니다.

이전에는 각 블럭 장치마다 하나의 큐를 가지고 있었습니다. 그래서 여러 프로세스들이 하나의 큐에 자신이 원하는 정보를 넣고, 블럭 장치 드라이버는 큐에서 정보를 하나씩 빼서 블럭 장치를 읽고 쓰고 했습니다.

딱 봐도 감이오시겠지만 멀티 코어 시대에 큐가 하나라니 당연히 성능에 발못잡는 코드였습니다. 이전에 하드디스크시대에는 디스크 자체가 느려서 큐가 하나뿐이어도 디스크의 최대 성능을 뽑아내는데 문제가 없었지만, SSD로 넘어오면서 디스크가 빨리지고 큐 하나로는 장치의 최대 성능을 뽑아낼 수 없게된 것입니다.

그래서 결국 큐의 갯수를 늘렸습니다. 말이 쉽지 어마어마한 작업이었습니다. 이론적인 배경은 아래 논문을 참고하시기 바랍니다.


> Bjørling, Matias, et al. "Linux block IO: Introducing multi-queue SSD access on multi-core systems." Proceedings of the 6th International Systems and Storage Conference. ACM, 2013. - http://kernel.dk/systor13-final18.pdf

이 강좌에서는 커널 4.4에 있는 brd 드라이버와 null_blk 드라이버를 짬뽕한 mybrd라는걸 만드는 과정을 설명합니다. mybrd를 만들면서 큐가 하나일때, 여러개일때 어떻게 드라이버가 달라지고, 드라이버가 달라지는것에 따라서 커널이 어떻게 다른 처리를 하는지 등을 따라하기 식으로 설명합니다. 이 강좌의 끝까지 따라하다보면 최종적으로  https://github.com/gurugio/mybrd/ 에 저장되어있는 mybrd.c 파일을 만들게됩니다. 아무일도 안하는 껍데기만 있는 드라이버부터 조금씩 만들어나가겠습니다.

제 지식에 한계가 있어서 따라하기 식으로 간단하게 설명할 수 밖에 없습니다만 그 배경에는 어마어마한 커널 코드들이 있습니다. 드라이버가 하나의 커널 함수를 호출할때 커널은 많은 일들을 하게됩니다. 이런 커널 코드를 일일이 설명할 수도 없고, 설명하는게 좋지만은 않습니다. 왜냐면 버전이 바뀌면서 코드가 달라지고 새로운 알고리즘들이 들어가기도하고 빠지기도하니까요.. 가장 좋은 것은 드라이버가 커널을 어떻게 사용하는지를 보고 커널의 동작 방식을 이해한 다음, 각 커널 함수들의 어떻게 구현되었는지를 코드를 보면서 이해하는 것입니다. 한가지 버전의 코드에 익숙해지면 그 다음버전 그 다음다음버전도 계속 쉽게 익숙해질 수 있습니다. 그러니까 이 강좌를 통해 커널의 흐름을 이해하고, 그 다음 커널 코드를 직접 읽어볼 필요가 있습니다.

이 강좌가 커널의 블럭 레이어 속으로 들어가는 길잡이가 되었으면 합니다.



PS.

강좌 수준은 커널 드라이버를 약간 만들어보거나 커널 관련 책을 반쯤 보다가 포기한 분들께 적당할것 같습니다. 특히 Linux device driver 책을 보긴했지만, 잘 이해가 안되는게 너무 많았던 분들을 대상으로 생각하고 썼습니다. 왜냐면 제가 그런 상태였고, 이 mybrd라는걸 만들어보면서 공부가 많이 됐었으니까요. 완전히 커널이나 드라이버를 접해보지 않으신 분들을 대상으로 하지는 않습니다. 그러면 강좌가 아니라 책 한두권이 될것이니까요.

PS.

참고로 메모리 할당쪽 코드는 블럭 레이어보다는 좀더 직관적입니다. 왜냐면 메모리 할당 함수가 실행되면 곧바로 메모리 할당을 실행해야하니까요. 메모리 할당 함수 코드를 어렵지만 잘 따라가다보면 동작 방식을 이해할 수 있습니다.

하지만 블럭 레이어는 좀 다릅니다. 디스크의 읽고 쓰기가 synchronous하게 실행되지 않습니다. 어플등에서 데이터를 요청하면 이 요청들이 어딘가에 모아지고, 나중에 어떤 시점에 모아진 요청들을 처리합니다. 그래서 커널 함수만 따라가면서 이해하는게 약간 더 힘이 듭니다. 이런 차이가 있다는걸 알면 커널을 시작하는데 도움이 될것입니다.

PS.

메모리쪽 코드를 이해하기 위해서는 메모리쪽 커널 메인테이너 Mel Gorman님이 쓰신 아래 책을 추천합니다. 버전이 달라져도 그 뼈대는 동일하니까 오래된 자료지만 아직도 유효합니다.

https://www.kernel.org/doc/gorman/pdf/understand.pdf

제가 위의 문서를 약간 번역한 자료입니다.
https://github.com/gurugio/book_linuxkernel_blockdrv_ko/blob/master/buddyslab_main_functions.pdf
https://github.com/gurugio/book_linuxkernel_blockdrv_ko/blob/master/understanding_buddy_slab.pdf


# INDEX

* [개발환경](environment.md)
* [mybrd 드라이버의 뼈대 빌드](mybrd_skeleton.md)
* [디스크 만들기](create_disk.md)
* [램디스크 구현](create_ramdisk.md)
* [request-mode](request-mode.md)
* [multiqueue-mode](multiqueue-mode.md)
* [pagecache_and_blockdriver](pagecacheand_blockdriver.md)
* [page-flags](page-flags.md)
* [시스템콜과 블럭 장치 flush](systemcall_flushblock.md)
* [per-cpu변수와 통계 정보(v2.6.11)](per-cpu_statistics.md)
* [wait-queue](wait-queue.md)
* [ida and request-queue](ida_and_request-queue.md)
* [spinlock](spinlock.md)
* [read-copy-update](read-copy-update.md)
* [work-queue](work-queue.md)
* [TODO](todo.md)

