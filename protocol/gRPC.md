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

これは仕様通り。[Core concepts, architecture and lifecycle | gRPC](https://grpc.io/docs/what-is-grpc/core-concepts/#deadlines)に次のように書かれている。
> gRPC allows clients to specify how long they are willing to wait for an RPC to complete before the RPC is terminated with a DEADLINE_EXCEEDED error. 

`GODEBUG=http2debug=2`を付けてHTTPS2のフレームを見てみると、`RST_STREAM`が1秒後にクライントから送られてきているのがわかる。

client側
```
❯ GODEBUG=http2debug=2 go run greeter_client/main.go
2021/02/28 00:10:26 http2: Framer 0xc00021a000: wrote SETTINGS len=0
2021/02/28 00:10:26 http2: Framer 0xc00021a000: read SETTINGS len=6, settings: MAX_FRAME_SIZE=16384
2021/02/28 00:10:26 http2: Framer 0xc00021a000: wrote SETTINGS flags=ACK len=0
2021/02/28 00:10:26 http2: Framer 0xc00021a000: read SETTINGS flags=ACK len=0
2021/02/28 00:10:26 http2: Framer 0xc00021a000: wrote HEADERS flags=END_HEADERS stream=1 len=95
2021/02/28 00:10:26 http2: Framer 0xc00021a000: wrote DATA flags=END_STREAM stream=1 len=12 data="\x00\x00\x00\x00\a\n\x05world"
2021/02/28 00:10:26 http2: Framer 0xc00021a000: read WINDOW_UPDATE len=4 (conn) incr=12
2021/02/28 00:10:26 http2: Framer 0xc00021a000: read PING len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"
2021/02/28 00:10:26 http2: Framer 0xc00021a000: wrote PING flags=ACK len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"
2021/02/28 00:10:27 could not greet: rpc error: code = DeadlineExceeded desc = context deadline exceeded
2021/02/28 00:10:27 http2: Framer 0xc00021a000: wrote RST_STREAM stream=1 len=4 ErrCode=CANCEL
exit status 1
```

server側
```
❯ GODEBUG=http2debug=2 go run greeter_server/main.go
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: wrote SETTINGS len=6, settings: MAX_FRAME_SIZE=16384
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: read SETTINGS len=0
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: wrote SETTINGS flags=ACK len=0
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: read SETTINGS flags=ACK len=0
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: read HEADERS flags=END_HEADERS stream=1 len=95
2021/02/28 00:10:26 http2: decoded hpack field header field ":method" = "POST"
2021/02/28 00:10:26 http2: decoded hpack field header field ":scheme" = "http"
2021/02/28 00:10:26 http2: decoded hpack field header field ":path" = "/helloworld.Greeter/SayHello"
2021/02/28 00:10:26 http2: decoded hpack field header field ":authority" = "localhost:50051"
2021/02/28 00:10:26 http2: decoded hpack field header field "content-type" = "application/grpc"
2021/02/28 00:10:26 http2: decoded hpack field header field "user-agent" = "grpc-go/1.37.0-dev"
2021/02/28 00:10:26 http2: decoded hpack field header field "te" = "trailers"
2021/02/28 00:10:26 http2: decoded hpack field header field "grpc-timeout" = "999965u"
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: read DATA flags=END_STREAM stream=1 len=12 data="\x00\x00\x00\x00\a\n\x05world"
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: wrote WINDOW_UPDATE len=4 (conn) incr=12
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: wrote PING len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"
2021/02/28 00:10:26 http2: Framer 0xc0001e6000: read PING flags=ACK len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"
2021/02/28 00:10:27 http2: Framer 0xc0001e6000: read RST_STREAM stream=1 len=4 ErrCode=CANCEL
2021/02/28 00:10:28 Received: world
```

なお、クライアントはタイムアウトの情報を `grpc-timeout` ヘッダーに付けてリクエストしている。このヘッダーの仕様は [grpc/PROTOCOL-HTTP2.md at master · grpc/grpc](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) に書かれている。
```
2021/02/28 00:10:26 http2: decoded hpack field header field "grpc-timeout" = "999965u"
```
1秒を指定しているが、なぜか若干小さい値になっている。


## 参考文献

- [gRPC Internal - gRPC の設計と内部実装から見えてくる世界 | Wantedly Engineer Blog](https://www.wantedly.com/companies/wantedly/post_articles/219429)
