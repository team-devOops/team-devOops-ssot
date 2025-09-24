CQRS 공부하려다 보니, 이해가 잘 안되어서 더 기본적인 개념부터 보는 것을 추천한다고 하여 CQS부터 공부하고자 합니다.

# 1. CQS란?
명령(Command)과 조회(Query)를 코드 레벨에서 분리한 것을 의미합니다.

버트랑 메이어가 소개를 한 개념으로, 함수는 상태를 변경하는 것(Command)과 값을 조회하고 상태를 변경해서는 안 되는 것(Query)으로 나눌 수 있고,
둘 중 하나만 해야 합니다.

- 명령(Command) : 상태를 변경함. 반환 값이 없어도 됨
    - 생성 데이터 식별 값, 혹은 상태 변경 여부 정도는 내려줄 수 있음
- 조회(Query) : 상태 변경하지 않음. 데이터만 반환
    - 조회로 인한 부작용이 발생하지 않음을 보장해야 함

# 2. CQS로 나누는 이유
1. 직관적
    - 해당 기능이 **쓰기**인지 **읽기** 역할인지 구분이 명확하게 됨
2. 테스트 용이
    - Command와 Query의 분리로 각각 테스트가 되었기에 합쳐 쓸때는 최소한의 테스트만 해도 됨
3. 버그 감소
    - Query는 상태를 변경하지 않기 때문에 사이드 이펙트 차단

단점으로는, 코드나 클래스 수가 증가해 복잡해질 수 있습니다.
또한 Command와 Query를 조합하기 위한 조합 계층이 별도로 필요하게 됩니다.
# 3. CQS 사용 가이드
정확한 표준을 가지고 있는 것은 아니지만, 아래 기준으로 역할 분리를 하게 되면, 유지보수와 테스트가 용이해질 것 같아 가이드를 정리해 보았습니다.

## 3-1. 서비스 클래스명
- Command : ~CommandService
    - 예) PostCommandService, BoardCommandService
- Query : ~QueryService
    - 예) PostQueryService, BoardQueryService

> 각 클래스를 분리함으로써, 서로 다른 @Service 빈으로 분리해 **셀프 인보케이션** 문제를 피할 수 있습니다.
## 3-2. 레파지토리명
- Command : ~CommandRepository
    - 예) PostCommandRepository, BoardCommandRepository
- Query : ~QueryRepository
    - 예) PostQueryRepository, BoardQueryRepository

> 저장, 수정, 삭제 등의 상태 변경 작업의 경우 Command 레파지토리에서 처리하고,
> 복잡한 JOIN 조회나, 집계 쿼리의 경우 Query 레파지토리 분리하여 사용합니다.
### 3-3. 메서드명
- Command : crate, update, delete, approve 등, **상태 변경 의도**가 드러나는 이름
- Query : get, find, list, search 등, **조회 의도**가 드러나는 이름

> Command에서 화면에 내려줄 DTO를 조립하지 않습니다.
## 3-4. 트랜잭션
- Command : @Transactional
- Query : @Transactional(readOnly = true)

> Command에서 최소한의 조회를 함으로써, 최소한의 로직 유지 및 경합을 방지할 수 있습니다.
## 3-5. 패키지 구조
더 나아가 패키지 구조도 Command와 Query로 분리가 가능합니다.
```
com.example.board
 ├─ controller
 ├─ command
 │   ├─ application  (UseCase/Service)  → PostService, CommentService
 │   └─ domain       (Aggregate/Entity)
 │
 └─ query
     ├─ application  (QueryService)     → PostQuery, CommentQuery
     └─ model        (View/DTO/Projection)
```
패키지 path가 보이기 때문에, 클래스 명에서 Command나 Query 같은 접미사를 생략하여 이름을 짧게 작성할 수 있습니다.
# 4. CQS 예시
## 4-1. CQS를 위반한 코드
```java
@Service
@RequiredArgsConstructor
public class PostService {

  private final PostRepository postRepository;
  private final CommentRepository commentRepository;
	
   @Transactional
   public PostResponse post(final PostSaveRequest request) {
		// (1) 쓰기: 게시글 생성
	      final Post post = 
		      Post.create(request.title(), request.content(), request.authorId());

            postRepository.save(post);

            // (2) 읽기: 목록 + 댓글 수 등 무거운 조회 (JOIN/집계)
            // 게시글 목록을 띄워주기 위한 최신 글 조회
            final List<PostSummary> list = postRepository.fetchLatestSummaries(1, 20); // JOIN/집계 가정

		// 최신 글의 댓글 수 조회
            final Map<Long, Long> commentCounts = commentRepository.countByPostIds(
                  list.stream()
                  .map(PostSummary::id)
                  .toList()
            );
            
            // (3) 화면 DTO 가공(응답 조립)
            return new PostResponse(list, commentCounts);
   }
}
```
`post()`의 경우 아래와 같은 기능을 합니다.
1. 게시글 생성 (쓰기)
2. 게시글 목록 조회 (읽기)
3. 목록 게시글의 댓글 수 조회 (읽기)

상태 변경(쓰기)와 데이터 조회(읽기)가 하나의 메서드 내에서 처리가 되기에 **CQS 원칙을 위배**하였습니다.
이로 인해, 다음과 같은 문제가 발생하게 됩니다.
- 메서드 책임 불분명, 해당 기능이 읽기인지 쓰기인지 직관적이지 않음
- 테스트 시, 쓰기와 읽기에 대한 전체적인 모킹으로 인해 테스트 비용 상승
- 유지보수 시, 조회 로직 변경이 되면 의도치 않게 쓰기에도 영향 끼칠 가능성 증가
## 4-2. CQS 준수한 코드
위의 `post()`를 CQS 준수한 코드로 개선을 해본다면, 아래와 같이 작성할 수 있습니다.
#####  1) Command
```java
@Service
@RequiredArgsConstructor
public class PostCommandService {
	private final PostCommandRepository postCommandRepository;
	
	@Transactional
	public Long post(final PostSaveRequest request) {
		// 의사결정용 조회가 필요하다면, 동일 트랜잭션에서 수행합니다.
		// 예로, 중복처리 등
		
            final Post post = postRepository.save(
	            Post.create(request.title(), request.content(), request.authorId()));
            
            return post.getId(); // 식별자 반환
	}
}
```
`PostCommandService`의 경우 게시글 저장을 담당합니다.
추가로 수정이라던가 삭제 등의 상태 변경하는 로직들을 처리할 수 있습니다.

또한, 메서드 내에서 **의사결정에 필요한 조회**(예, 중복 여부 확인)가 필요하다면, 동일 트랜잭션 내에서 처리합니다.

반환 값이 없거나, 혹은 식별자 값 또는 상태 변경 성공 여부에 대한 값을 반환해 줄 수 있습니다.
##### 2) Query
```java
@Service
@RequiredArgsConstructor
public class PostQueryService {
	private final PostQueryRepository postQueryRepository; // 조회 전용 리포지토리(비정규화/뷰 조회 가능)
	private final CommentQueryRepository commentQueryRepository; // 댓글 카운트 조회 전용
	
	@Transactional(readOnly = true)
	public PostResponse getPostSummary(final int page, int size) {
		final List<PostSummary> list 
				= postQueryRepository.fetchLatestSummaries(page, size);

            final Map<Long, Long> commentCounts = commentRepository.countByPostIds(
                  list.stream()
                  .map(PostSummary::id)
                  .toList()
            );

            return new PostResponse(list, commentCounts);
	}
}
```
`PostQueryService`의 경우 **조회 기능**만을 합니다.
조회 로직만을 분리했기 때문에, 화면에 필요한 데이터 조회와 응답 조립에 집중할 수 있습니다.

해당 로직에서 더욱 책임과 역할을 분리하고 싶다면, `CommentQueryService`로도 분리하여 세분화할 수 있습니다.

### 3) 상위에서 조합
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/posts")
class PostController {

    private final PostCommandService commandService;
    private final PostQueryService queryService;

    // 동기 조합(단순 시스템): 생성 후 최신 보드 뷰 반환
    @PostMapping
    public PostResponse createAndList(@RequestBody final PostSaveRequest request) {
        final Long postId = commandService.create(request);   // 쓰기
        return queryService.getBoardView(1, 20);        // 읽기
    }
}
```
`Controller`에서는 Command와 Query를 조합하는 역할을 하게 됩니다.
Controller 외에도 Service에서도 조합이 가능합니다.

이처럼 Command와 Query는 각자 책임에만 집중하고, 상위 계층에서 조합하여 분리할 수 있습니다.

---
# 느낀점
처음에는 CQS란, 읽기와 쓰기를 절대적으로 분리해야 되는 원칙이라고 생각했었습니다.
하지만 공부해 보니 반드시 그런 것은 아니고, 언제 분리를 해야 하는지에 대한 개념을 다시 정리할 수 있었습니다.

특히 Command의 경우, 의사결정에 필요한 최소한의 조회는 허용된다는 점이 처음에는 헷갈렸습니다.
이는 '명령을 실행할 때 최소한 필요한 조회'라는 것을 알려주는 방식으로 유지보수에 큰 도움이 될 것이라고 이해
되었습니다.

또한 읽기와 쓰기가 한 서비스에 있는 거랑, CQS로 분리하는 거랑 결국 테스트는 똑같이 작성해야하지 않나? 싶었지만, 읽기와 쓰기가 분리 된 상태에서는 각각의 단일 테스트가 만족이 되어 있고, 상위 계층에서는 단순 흐름 테스트를 하는 부분에서 테스트가 용이한 구조임을 알 수 있었습니다.

이러한 개념 정리가 이후 CQRS 개념을 또 잡는 데에 도움이 되면 좋겠습니다.