# gRPC

RPC-фреймворк от Google. Бинарный протокол поверх HTTP/2. Использует Protocol Buffers для сериализации.

## Почему gRPC

- **Производительность** — бинарная сериализация (protobuf) быстрее JSON в 5-10x
- **Контракт** — `.proto` файл описывает API, генерируется код для клиента и сервера
- **Стриминг** — bidirectional streaming через HTTP/2
- **Мультиязычность** — кодогенерация для Go, Java, Python, Rust и др.

## Типы RPC

| Тип | Описание |
|-----|----------|
| Unary | Один запрос → один ответ (как REST) |
| Server streaming | Один запрос → поток ответов |
| Client streaming | Поток запросов → один ответ |
| Bidirectional streaming | Поток в обе стороны |

## Proto файл

```protobuf
syntax = "proto3";
package user;
option go_package = "gen/user";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);                          // unary
  rpc ListUsers(ListUsersRequest) returns (stream User);               // server stream
  rpc UploadUsers(stream User) returns (UploadResponse);               // client stream
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);           // bidi stream
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}
```

## Go-сервер

```go
type userServer struct {
    pb.UnimplementedUserServiceServer
    repo UserRepository
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.repo.FindByID(ctx, req.GetId())
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user not found: %v", err)
    }
    return toProto(user), nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    srv := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
    )
    pb.RegisterUserServiceServer(srv, &userServer{})
    srv.Serve(lis)
}
```

## Коды ошибок gRPC

| Код | Описание | HTTP аналог |
|-----|----------|-------------|
| OK | Успех | 200 |
| INVALID_ARGUMENT | Невалидные данные | 400 |
| NOT_FOUND | Не найден | 404 |
| ALREADY_EXISTS | Уже существует | 409 |
| PERMISSION_DENIED | Нет прав | 403 |
| UNAUTHENTICATED | Не аутентифицирован | 401 |
| INTERNAL | Внутренняя ошибка | 500 |
| UNAVAILABLE | Сервис недоступен | 503 |
| DEADLINE_EXCEEDED | Таймаут | 504 |

## Interceptors (middleware)

```go
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("method=%s duration=%s err=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}
```

## Метаданные (аналог HTTP headers)

```go
// Клиент — отправка
md := metadata.Pairs("authorization", "Bearer token123")
ctx = metadata.NewOutgoingContext(ctx, md)

// Сервер — чтение
md, ok := metadata.FromIncomingContext(ctx)
token := md.Get("authorization")
```

## gRPC vs REST

| Критерий | gRPC | REST |
|----------|------|------|
| Формат | Protobuf (binary) | JSON (text) |
| Контракт | .proto файл | OpenAPI (опционально) |
| Стриминг | Да | Нет (только SSE/WebSocket) |
| Браузер | Нужен grpc-web | Нативно |
| Тулинг | protoc, buf | curl, Postman |
| Применение | Микросервисы, internal API | Public API, web |

## Полезные инструменты

- **buf** — линтер и codegen для proto
- **grpcurl** — curl для gRPC
- **Evans** — REPL для gRPC
- **grpc-gateway** — REST-прокси перед gRPC
