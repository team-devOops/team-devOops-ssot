# 1. gRPC란?
Google이 만든 오픈소스 **RPC(Remote Procedure Call)** 프레임워크입니다.
- 🔗 공식 문서 : https://grpc.io/docs/what-is-grpc/introduction/

> RPC란?
> - 로컬 함수를 호출하는 것처럼 원격 서버의 함수를 호출할 수 있게 만들어주는 **통신 방식**입니다.

gRPC는 RPC를 HTTP/2 프로토콜 위에서 동작하고, **Protocol Buffers(proto)** 를 **직렬화 포맷**으로 사용하여 구현한 것입니다.

즉, 개발자는 서버의 함수를 단순히 호출하는 것처럼 보이지만,
내부에서는 알아서 역/직렬화 처리가 되어 통신 되는 방식을 효율적으로 표준화한 프레임워크입니다.

## 1.1 기존 REST의 한계
gRPC가 등장하기 전에는 REST 방식을 주로 사용하였지만, 다음과 같은 제약이 존재하였습니다.
- HTTP/1.1 연결 제약
	- 요청 하나당 하나의 응답만 처리
	- HOL(Head of Line) 블로킹 문제 : 선두 요청이 지연되면 뒤의 요청 모두 지연
- JSON 역/직렬화 비용 발생
	- JSON은 텍스트 기반이라 사람 친화적이지만, 그만큼 기계 입장에서 불필요한 오버헤드 발생
- API 스펙 보장의 한계
	- OpenAPI 같은 명세가 있음에도 별도 문서이기 때문에 실제 코드와 불일치할 가능성 존재
	- 서버와 클라이언트 간 계약 불일치 (예, 필드 이름 오타, 타입 불일치) 하더라도 **런타임**때 발견

## 1-2. gRPC 특징
gRPC는 REST의 한계를 보완하며, 다음과 같은 특징을 갖습니다.
- HTTP/2 기반
	- 하나의 커넥션에 여러 개의 요청을 동시에 보낼 수 있는 **멀티플렉싱** 지원
	- **헤더 압축**이나 **양방향 스트리밍** 가능
- Protocol Buffers (proto)
	- JSON 대신 ProtoBuf라는 **바이너리 직렬화 포맷** 사용
	- 메세지 크기가 작고, 역/직렬화가 빠름
	- `.proto`파일로 서비스와 데이터 구조 정의 -> 언어별 코드 **자동 생성**
- 다양한 통신 패턴 지원
	- Unary PRC : 요청 1 <-> 응답 1
	- Server Streaming : 요청 1 <-> 응답 N
	- Client Streaming : 요청 N <-> 응답 1
	- Bidirectinal Streaming : 요청 N <-> 응답 N
- 엄격한 인터페이스 계약
	- `.proto` 파일로 서비스 스펙 고정
		- 컴파일 단계에서 오류 발견 가능
	- 다양한 언어로 자동 스텁(stub) 생성 가능

### REST, gRPC 비교
| 구분      | REST                | gRPC              |
| ------- | ------------------- | ----------------- |
| 전송 방식   | JSON (텍스트)          | ProtoBuf (바이너리)   |
| 프로토콜    | 주로 HTTP/1.1         | HTTP/2            |
| 성능      | 역/직렬화 비용이 큼         | 바이너리 형식으로 빠르고 효율적 |
| 스펙      | OpenAPI 명세가 있으나 느슨함 | `.proto` 기반으로 엄격함 |
| 스트리밍    | 별도 구현 필요            | 기본 내장             |
| 브라우저 지원 | 우수                  | 제한적(gRPC-Web 필요)  |

# 2. 주요 개념 정리
## 2-1. Service 정의
gRPC에서 모든 것은 `.proto` 파일로 시작됩니다.
해당 파일 내에 이런 서비스가 있고, 이 서비스는 이런 메서드를 제공한다. 라고 정의할 수 있습니다.

예로, 아래와 같이 `HelloService`라는 서비스를 정의하고자, `hello.proto`를 작성할 수 있습니다.
```proto
syntax = "proto3";

service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```
- HelloService : 서비스 이름
- SayHello : 메서드 이름
	- HelloRequest : 요청 타입
	- HelloResponse : 응답 타입
- message : 요청과 응답 데이터 구조 정의

`.proto` 파일은 인터페이스 역할을 합니다.
해당 파일을 바탕으로 원하는 언어로 코드가 **자동 생성**될 수 있습니다.

## 2-2. Stub
`.proto`파일을 컴파일하면, 클라이언트와 서버가 사용할 수 있는 코드가 생성됩니다.
- 서버 : `HelloServiceImplBase` 같은 **추상 클래스**가 생성됩니다. 서버 개발자가 이 클래스를 상속받아 직접 구현합니다.
- 클라이언트 : Stub 객체 생성되며, 원격 서버 함수를 로컬 메서드처럼 호출할 수 있습니다.

### 종류 (Java 기준)
- BlockingStub: 동기 호출, 함수가 끝날 때까지 대기
- AsyncStub: 비동기 호출, Callback 등록 가능
	- 기본적으로 gPRC는 Async 방식 사용
- FutureStub: Java의 Future 기반 비동기 호출

## 2-3. 동작 흐름
1. 클라이언트 호출
	- 개발자는 blockingStub.sayHello(request) 같은 코드를 호출합니다.
2. Stub 처리
	- Stub이 요청 객체를 ProtoBuf로 직렬화하고, 이를 HTTP/2 기반으로 서버에 전송합니다.
3. 서버 처리
	- 서버는 `.proto`로 생성된 HelloServiceImplBase를 상속받아 구현한 메서드를 실행합니다.
	- 비즈니스 로직 처리 후 응답 객체를 생성합니다.
4. 응답 반환
	- 서버 응답은 ProtoBuf 형식으로 직렬화되어 클라이언트에 Stub으로 전달됩니다.
	- Stub은 이를 역직렬화하여 클라이언트 코드에 최종 객체로 반환합니다.

---
# 느낀점
gRPC는 함수를 호출하는 방식으로 사용된다는 것이
기존 REST 방식으로만 통신을 주고받던 방식으로만 개발하던 저에게는 감이 잘 안 오는 방식이라 어떻게 처리가 되는 건지 궁금하여 공부해 보고 싶었습니다.
처음 개념 파악을 하면서 어떤 느낌인지 감을 잡을 수 있었고,
빠르게 직접 코드를 작성하고 실습하고 싶다는 생각이 들었습니다.

아직 완벽하게 이해를 한 것은 아니지만, 기존 REST 방식보다는 확실히 편하게 코드를 작성할 수 있을 것으로 기대가 됩니다.

