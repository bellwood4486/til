- Protocol Buffers と HTTP/2 の技術の上に成り立っている。
- 設計原則は [gRPC Motivation and Design Principles](https://grpc.io/blog/principles/) に書かれている。16項目ある。

## 実験

### タイムアウト発生時のエラーログは何か？

ClientがタイムアウトするようにSleepを入れる。

https://github.com/bellwood4486/grpc-go/blob/837130428446606afb939d8ae30e85813caf796d/examples/helloworld/greeter_server/main.go#L44-L44
```go
// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	// long process...
	time.Sleep(2 * time.Second)
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}
```

Clientを実行する。エラーは`code = DeadlineExceeded desc = context deadline exceeded`となる。
```
❯ go run greeter_client/main.go
2021/02/27 23:07:45 could not greet: rpc error: code = DeadlineExceeded desc = context deadline exceeded
exit status 1
```

## 参考文献

- [gRPC Internal - gRPC の設計と内部実装から見えてくる世界 | Wantedly Engineer Blog](https://www.wantedly.com/companies/wantedly/post_articles/219429)
