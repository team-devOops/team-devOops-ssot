# 1. 2일차 학습 목표
직접 손으로 proto 파일을 작성하고 코드 생성까지 해보는 단계
1. **ProtoBuf 문법(proto3)** 이해
2. .proto 파일 직접 작성
3. protoc 컴파일러로 서버 코드 생성
4. 생성된 코드 구조 확인

# 2. ProtoBuf 기본 구조 (proto3 기준)
🔗 Protobufpal (https://www.protobufpal.com/) 를 통해 프로토 메세지의 유효성 검증과 직렬화/역직렬화, 인코딩/디코딩 등 간단하게 proto 메세지 테스트가능합니다.
```proto
syntax = "proto3";

service UserService {
	rpc SignUp(Person) returns Person;
}

message Person {
  string name = 1;
  int32 age = 2;
  bool is_active = 3;
}
```

## 2-1. syntax
`.proto` 타입 상단에는 `syntax`를 선언하여, 어떤 프로토콜 버전 문법을 따를 것인지 지정이 필요합니다.
```proto
syntax = "proto3";
```
- 해당 `.proto` 파일은 **Protocol Buffers v3 문법 (proto3)**을 따르는 것을 의미합니다.
- 문법 버전에 따라 지원되는 타입, 필드 기본값 처리 방식, optional/required 지원 여부 등이 달라집니다.
- proto2 문법의 경우 deprecated 상태는 아니지만, 대부분 gRPC 공식 문서 및 최신 라이브러리를 proto3을 전제로 합니다.

#### proto2
- Protocol Buffers 2.x 문법, 2008년 발표
- required, optional, repeated 키워드 모두 사용 가능
- default 값을 명시적으로 지정 가능
- Google 내부 시스템에서 오래 사용되던 형식

#### proto3
- Protocol Buffers 3.x 문법, 2016년 발표, **현재 표준**
- required 키워드 제거, 모든 필드는 기본적으로 optional (단, repeated는 여전히 사용 가능)
- 필드에 값을 주지 않으면 **언어별 기본값**으로 처리
    - 예, string : "", int: 0, bool: false
- 단순화되고 언어별 바인딩이 일관됨
- gRPC 표준은 proto3 기반

### 기본값 차이 예시
```
// proto2
message Person {
  required string name = 1;
  optional int32 age = 2 [default = 18];
}

// proto3
message Person {
  string name = 1; // 기본: ""
  int32 age = 2;   // 기본: 0
}
```
- proto2에서는 required를 지정하지 않으면, 런타임 오류 가능성이 커서 관리가 까다롭습니다.
- proto3은 단순화 대신, 모든 필드가 optional 처럼 동작하고 기본값이 자동으로 적용됩니다.

## 2-2. 메세지
메세지란, 하나의 데이터 구조(클래스, 객체 개념)을 정의하는 키워드를 의미합니다.
Java에서는 `Person` 클래스가 자동으로 생성되게 됩니다.
```proto
message Person {
  string name = 1;
  int32 age = 2;
  bool is_active = 3;
}
```
- message : 하나의 데이터 구조를 정의하는 키워드, 보통 PascalCase로 작성
- string name = 1
    - string : 타입, int32, string, bytes 같은 타입 선언
    - name : 필드명, snake_case 권장, 자동 생성되는 코드는 언어별 네이밍 컨벤션에 자동 맞춤 (예, getName())
    - 1 : 태그 번호, 직렬화할 때 해당 필드를 식별하는 고유 번호
        - 메세지 전송 시, 필드명이 아닌 번호를 기준으로 데이터 인코딩/디코딩
        - 한 번 사용한 번호는 절대 재사용하지 않는 것이 원칙

### 태그 번호 규칙
- 범위: 1 ~ 2^29-1 (약 5억)
- 1~15: 직렬화 시 **1바이트**로 표현되므로 자주 쓰이는 필드에 배치하면 효율적
- 16~2047: 일반 필드용으로 많이 사용
- 재사용 금지: 한 번 사용되었다가 삭제된 번호는 **reserved** 키워드로 명시
```proto
message Person {
  reserved 4, 5;
  reserved "old_field";
}
```

### 중첩 메세지, 메세지 재사용
메세지 안에 또 다른 메세지를 정의하거나, 다른 메세지를 재사용할 수도 있습니다.
```proto
message Address {
  string city = 1;
  string street = 2;
}

message Person {
  string name = 1;
  int32 age = 2;
  Address address = 3; // 다른 message 타입 사용
}
```

## 2-3. 서비스
gRPC에서 제공하는 서비스 단위를 정의하는 키워드입니다.
이 안에 여러 개의 rpc 메서드 선언이 가능합니다.
```
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
  
  // 서버 스트리밍
  rpc ListOrders (OrderListRequest) returns (stream OrderResponse);

  // 클라이언트 스트리밍
  rpc UploadOrders (stream OrderRequest) returns (UploadSummary);

  // 양방향 스트리밍
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```
- service : 서비스 단위를 정의하는 키워드, 보통 PascalCase로 작성
- rpc : Remote Procedure Call을 정의하는 키워드
    - 단순한 메세지 하나일 수도 있고, stream 키워드를 붙여 스트리밍 응답으로 만들 수도 있습니다.
- returns: 메서드 응답 타입

## 2-4.  Enum
상태 값이나 제한된 선택지를 정의할 때 사용하게 됩니다.
Java enum과 거의 유사하지만, 직렬화, 호환성 면에서 몇 가지 특성이 존재합니다.

또한 작성 시, **파스칼 케이스**(PascalCase)를 사용합니다.
```proto
enum OrderStatus {
  ORDER_STATUS_UNKNOWN = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_COMPLETED = 2;
  ORDER_STATUS_CANCELLED = 3;
}
```
- enum OrderStatus: Enum 타입 이름, PascalCase 권장
- 상수 값: 대문자 스네이크 케이스 권장
- 할당된 숫자: 태그, 직렬화 시 실제로 전송되는 값

### 특징
1. 숫자 기반 직렬화
    - enum은 내부적으로 **int32** 숫자 값으로 직렬화
    - 예, ORDER_STATUS_PENDING -> 1
2. 첫 번째 값은 반드시 0이어야 함
    - proto3에서는 첫 값 = 기본값 규칙이 있기 때문에 **꼭 0을 지정** 필수
    - 보통 UNKNOWN 같은 이름 설정
3. 하위 호환성 유지
    - 새로운 enum 값을 추가하는 것은 안전
    - 다만, 이미 사용 중인 번호는 재사용 불가 (삭제 시 reserved 선언 권장)
4. 알 수 없는 값 처리
    - 클라이언트가 모르는 값 (서버가 새 enum을 추가한 경우)을 받으면 기본적으로 0으로 처리
5. 메서드 자동 생성
    - getNumber(), forNumber 메서드가 자동으로 생성

### RPC 메서드명
```proto
rpc GetUserProfile (UserRequest) returns (UserProfile);
```
역시 **파스칼 케이스** (PascalCase)를 사용합니다.

# 3. ProtoBuf 기본 자료형
| 타입           | 설명                   | Java 매핑 타입                     | 기본 값       | 비고                 |
| ------------ | -------------------- | ------------------------------ | ---------- | ------------------ |
| 정수형          |                      |                                |            |                    |
| **int32**    | 32비트 정수 (가변 길이 인코딩)  | int                            | 0          | 음수 비효율적, 일반 정수에 사용 |
| **int64**    | 64비트 정수 (가변 길이 인코딩)  | long                           | 0          |                    |
| **uint32**   | 부호 없는 32비트 정수        | int (음수 해석 주의)                 | 0          | 음수 없을 때 사용         |
| **uint64**   | 부호 없는 64비트 정수        | long (음수 해석 주의)                | 0          |                    |
| **sint32**   | 32비트 정수 (ZigZag 인코딩) | int                            | 0          | 음수가 많을 때 효율적       |
| **sint64**   | 64비트 정수 (ZigZag 인코딩) | long                           | 0          |                    |
| **fixed32**  | 고정 크기 32비트 정수        | int                            | 0          | 큰 값에 효율적           |
| **fixed64**  | 고정 크기 64비트 정수        | long                           | 0          |                    |
| **sfixed32** | 고정 크기 32비트 정수(부호 있음) | int                            | 0          |                    |
| **sfixed64** | 고정 크기 64비트 정수(부호 있음) | long                           | 0          |                    |
| 실수형          |                      |                                |            |                    |
| **float**    | 32비트 부동소수점           | float                          | 0.0        | IEEE 754           |
| **double**   | 64비트 부동소수점           | double                         | 0.0        | IEEE 754           |
| 논리형          |                      |                                |            |                    |
| **bool**     | 불리언 값                | boolean                        | false      |                    |
| 문자형          |                      |                                |            |                    |
| **string**   | UTF-8 문자열            | String                         | “” (빈 문자열) | 길이 제한 없음           |
| 바이너리         |                      |                                |            |                    |
| **bytes**    | 임의의 이진 데이터           | com.google.protobuf.ByteString | 빈 바이트      | 이미지/파일 데이터 가능      |
- **uint32/uint64**: Java에는 부호 없는 정수 타입이 없기 때문에 int / long 에 매핑되며, **음수처럼 보일 수 있음**. 필요시 `Integer.toUnsignedLong()`, `Long.toUnsignedString()` 같은 메서드로 변환.
- **bytes**: Java의 `byte[]` 가 아니라 `ByteString` 으로 매핑됩니다. 필요하면 `toByteArray()`로 변환 가능.
- **enum**: Java에서는 **enum 클래스**로 생성되며, 값은 int로 내부 표현됩니다.

# 4. 실습하기
## 4-1-1. protoc 설치
IntelliJ를 사용하면, 4-2의 플러그인 설치로 대체 가능

`.proto` 파일을 읽어서 각 언어에서 사용할 수 있는 소스 코드를 자동 생성해 주기 위한 도구를 설치
protoc은 Protocol Buffers Compiler 의 줄임말
### 설치
```
brew install protobuf
```

### 버전 확인
```
protoc --version
```

사람이 직접 `.proto` 파일을 파싱해 언어별 클래스를 만드는 건 휴먼 에러 발생 가능성이 큼
protoc이 표준에 맞춰 자동 생성해 주기 때문에, **언어별 코드가 일관되고, 서버-클라이언트가 같은 계약(Contract)을 공유**할 수 있습니다.

즉, protoc은, proto 파일은 실행 가능한 코드로 변환해 주는 컴파일러입니다.

## 4-1-2. protobuf 플러그인 설치
```kts
// build.gradle.kts
import com.google.protobuf.gradle.*  
  
plugins {  
    ...
    id("com.google.protobuf") version "0.9.4"  
}  
  
dependencies {  
    implementation("io.grpc:grpc-netty-shaded:1.62.2")
    implementation("io.grpc:grpc-protobuf:1.62.2")
    implementation("io.grpc:grpc-stub:1.62.2")
    
    compileOnly("javax.annotation:javax.annotation-api:1.3.2")  
	annotationProcessor("javax.annotation:javax.annotation-api:1.3.2")
}  

...
  
protobuf {  
    protoc {  
       artifact = "com.google.protobuf:protoc:3.25.2"  
    }  
    plugins {  
       id("grpc") {  
          artifact = "io.grpc:protoc-gen-grpc-java:1.62.2"  
       }  
    }  
    generateProtoTasks {  
       all().forEach { task ->  
          task.plugins {  
             id("grpc")  
          }  
       }    
   }
}  
```
- import com.google.protobuf.gradle.*
    - `probobuf { ... }` DSL을 사용하기 위해 import 추가
- plugins
    - id("com.google.protobuf") version "0.9.4"
        - Google에서 제공하는 **protobuf-gradle-plugin** 적용
        - `.proto` 파일을 자동으로 찾아서 protoc 실행 -> Java 소스 코드 생성
- dependencies
    - grpc-netty-shaded: gRPC 서버/클라이언트를 실행하기 위한 Netty 기반 전송 계층 라이브러리
    - grpc-protobuf: ProtoBuf 메시지를 gRPC에서 쓰기 위한 직렬화 지원
    - grpc-stub: Stub 클래스(gRPC 클라이언트 프록시 객체)를 제공
- protobuf
    - protoc: proto 컴파일러(Protocol Buffers Compiler) 버전 지정, 3.25.2 버전 사용
    - plugins: 추가 코드 생성 플러그인 지정
        - grpc: gRPC 서비스용 Java 코드 생성기 (protoc-gen-grpc-java) -> ImplBase, Stub 등 생성
    - generateProtoTasks
        - `.proto` 파일 빌드 시 적용할 작업 정의
        - 모든 proto 파일에 대해 gRPC 플러그인을 적용(`task.plugins { id("grpc") }`)
        - 이로인해 메시지 클래스 + gRPC 서버/클라이언트 Stub 코드 자동 생성

플러그인 적용 후 빌드 시,
`extractIncludeProto`, `extractProto`, `generateProto` 같은 태스크를 실행하여 `.proto` 파일을 찾아 컴파일합니다.
```
> Task :extractIncludeProto
> Task :extractProto
> Task :generateProto 
```

## 4-2. proto 파일 작성
`/src/main/proto` 디렉터리 하위에 `hello.proto` 파일을 생성합니다.
```proto
syntax = "proto3";  
  
option java_multiple_files = true;  
option java_package = "com.example.grpc.hello";  
option java_outer_classname = "HelloProto";  
  
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
- option java_multiple_files = true;
    - 기본값은 false
        - .proto 안의 모든 message, enum, service가 하나의 최상위 클래스(outer class
          안의 **static inner class**로 생성됩니다.
    - true의 경우, 각각 독립된 **public 클래스**로 생성됩니다.
- option java_package = "com.example.grpc.hello";
    - 생성되는 Java 클래스들이 속할 **패키지 경로**를 지정합니다.
    - 지정하지 않으면, `.proto` 안에 `package` 경로를 Java 패키지 그대로 사용합니다.
    - `package` 또한 없을 경우, **기본 패키지(default package)**에  생성되게 됩니다.
- option java_outer_classname = "HelloProto";
    - `.proto` 파일이 여러 클래스를 담을 때, 이들을 감싸는 outer class 이름을 지정합니다.
    - java_multiple_files가 `false`일 때 효과가 가장 큽니다.
    - java_multiple_files가 `true`이게 되면
        - 최상위 `container` 클래스는 남아 있고, 내부 상수, 메타 정보(Desciptor) 관련 메서드들이 들어가게 됩니다.


## 4.3  ./gradlew generateProto
해당 명령어를 통해, `.proto` 파일을 찾아 컴파일하게 됩니다.

![Pasted image 20250922112934.png](image/Pasted%20image%2020250922112934.png)

## 4-4. gRPC 서버 설정
### 의존성  추가
```
dependencies {
	...
	implementation("net.devh:grpc-server-spring-boot-starter:3.0.0.RELEASE")
}
```

## 4-5. 구현체 작성하기
```java
@GrpcService
public class HelloService extends HelloServiceGrpc.HelloServiceImplBase {  
  
    @Override  
    public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {  
        final String message = "반갑습니다~ " + request.getName() + "님";  
  
        final HelloResponse response = HelloResponse.newBuilder()  
            .setMessage(message)  
            .build();  
  
        responseObserver.onNext(response);  // 응답 전달  
        responseObserver.onCompleted();     // 스트림 종료  
    }  
}
```
- @GrpcService
    - Spring Boot gRPC Starter에서 제공하는 어노테이션입니다.
    - 스프링 빈 + gRPC 서비스 구현체로 등록하기 위한 Bean입니다.
    - 내부적으로는 HelloServiceGrpc.HelloServiceImplBase(proto에서 생성된 추상 클래스)를 상속한 이 구현체를 gRPC 서버에 자동으로 addService(...) 해 줍니다.
    - 즉, @RestController가 HTTP 엔드포인트를 등록하듯, @GrpcService가 gRPC 엔드포인트를 등록하는 역할입니다.
- responseObserver.onNext(response);
    - gRPC에서는 **StreamObserver** 를 통해 응답을 클라이언트에게 보냅니다.
    - Unary RPC(요청 1 → 응답 1)에서는 이 메서드를 딱 한 번만 호출합니다.
    - 추가로 스트리밍 RPC(Server streaming, Bidirectional)에서는 여러 번 호출해서 여러 응답을 보낼 수도 있습니다.
- responseObserver.onCompleted();
    - onNext()로 응답을 모두 보낸 뒤, **스트림을 종료한다**는 신호를 의미합니다.
    - 클라이언트는 이 신호를 받아 “응답이 끝났구나” 하고 채널을 닫습니다.
    - 호출하지 않으면 클라이언트는 계속 응답을 기다리다가 타임아웃 날 수 있습니다.
### 설정 추가
```
grpc:  
  server:  
    port: 9090 # gRPC 포트 (기본 9090)    
    reflection-service-enabled: true  # grpcurl 등 도구용 리플렉션  
    security:  
      enabled: false             # 로컬 개발은 평문(PLAINTEXT). 운영은 TLS 권장
```
- grpc.server.reflection-service-enabled
    - gRPC 서버에 Server Reflection Service를 열어줄지 여부를 결정합니다.
    - Reflection Service란, 서버가 `.proto` 정의를 클라이언트에게 런타임에 노출하는 기능입니다.
        - 이 덕분에 클라이언트가 `.proto` 파일 없이도 서버에 어떤 서비스, 메서드가 있는지 조회할 수 있습니다.
        - 주로 디버깅, 테스트에 유용합니다.
            - grpcurl 같은 CLI 툴이 서버에 직접 접속해서 서비스 목록, 메서드, 메시지 구조를 확인 가능
    - 주로 로컬 개발이나 테스트 환경에서 `true`
    - 운영 환경에서는 비활성화 false 권장
- grpc.server.security.enabled
    - gRPC 서버의 TLS 보안 연결(SSL) 사용 여부를 설정합니다.
    - gRPC는 기본적으로 HTTP/2 기반이라 TLS와 잘 맞습니다.
    - true: TLS 활성화
        - 서버는 인증서/키를 읽어 TLS 연결을 수립합니다.
        - 클라이언트 역시 TLS 채널로 연결해야 합니다.
        - certificateChain, privateKey 설정이 추가로 요구됩니다.
    - false: TLS 비활성화 (PLAINTEXT 모드)
        - 로컬 개발/테스트 환경에서는 TLS 없이 간단히 통신 가능합니다.
        - 운영 환경에서 데이터가 평문으로 전송되어서는 안 되기에 비권장됩니다.

## 4-6. 동작 확인
스프링 부트 서버를 실행 후 gRPC 클라이언트로 호출하거나,
리플렉션을 켰다면 grpcurl 같은 도구로 호출할 수 있습니다.

![Pasted image 20250922133258.png](image/Pasted%20image%2020250922133258.png)
스프링 서버 올릴 때, gRPC 설정도 함께 올라오는 것을 확인 가능

클라이언트가 별도로 존재하지 않기 때문에 grpcurl 도구를 설치해 보도록 하겠습니다.
### 1) grpcurl 설치
```sh
brew install grpcurl
```

### 2) grpcurl 버전확인
```sh
grpcurl --version
```

### 3) 서비스 목록 확인
```sh
grpcurl -plaintext 127.0.0.1:9090 list
```

#### 결과
```sh
HelloService
grpc.health.v1.Health
grpc.reflection.v1alpha.ServerReflection
```

### 4. 서비스 스펙 확인
해당 서비스 안에 어떠한 RPC 메서드가 존재하는지, 요청/응답 타입이 무엇인지 확인 가능합니다.
```
grpcurl -plaintext localhost:9090 describe HelloService
```

#### 결과
```
HelloService is a service:
service HelloService {
  rpc SayHello ( .HelloRequest ) returns ( .HelloResponse );
}
```

### 5. 실행
```
grpcurl -plaintext -d '{"name":"zinzo"}' localhost:9090 HelloService/SayHello
```

#### 결과
```json
{
  "message": "반갑습니다~ zinzo님"
}
```

---

# 느낀점
직접 코드를 작성하고 실행 결과까지 확인해 보니, gRPC가 무엇인지 조금 더 느껴볼 수 있게 되었습니다.
아직은 메서드 호출 방식 자체가 낯설게 느껴지기지만,
자동으로 proto 파일에 맞춰 코드를 생성해 준다는 점과 서버와 클라이언트 간 동일한 스펙을 강하게 지킬 수 있다는 점이 REST 방식보다 훨씬 매력적으로 다가왔습니다.

클라이언트와 더욱 명확한 소통을 하기 위해서는 proto 타입을 어떻게 관리할 수 있을지 궁금한데, 이 부분도 추가적으로 찾아봐야겠습니다.