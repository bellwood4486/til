# HTTPまわりのコードリーディング

## (*Server).ListenAndServe()

```golang
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}
```

`net.Listen()`からは`net.Listener`を取得される。

## (*Server).Serve()

```go
func (srv *Server) Serve(l net.Listener) error {
	// ↓はテストコードからのフックをセットするためのもの
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // call hook with unwrapped listener
	}

	origListener := l
	// ここで作ったListenerは、後述の`trackListener()`メソッド内でも`Close()`が呼ばれる。
	// なので一度のクローズが保証されるようにしていると思われる。
	// `onceCloseListener`でやっている「ラップして特定のメソッドが1度しか呼ばれないことを保証する」アイディアは応用できそう。
	l = &onceCloseListener{Listener: l}
	defer l.Close()

	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	// ListenerをこのServerで追跡するものとして追加している。
	// 追加したListenerは`(*Server).closeListenersLocked()`で使われる。
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	// これあってもいいだろうけど必須なのかな？ やってることは追加するものから削除しているだけ。第2引数で追加と削除が切り替わる。
	defer srv.trackListener(&l, false)

	// ここの挙動は以下に記載されている。`BaseContext`とは着信するリクエスト用に使われる。
	// https://pkg.go.dev/net/http#Server.BaseContext
	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

	var tempDelay time.Duration // how long to sleep on accept failure

	// ここでは自分自身をコンテキストに結びつけている。
	// `srv.WriteTimeout`などが後段の処理で使われているので、そういうために受け渡している。
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		if err != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				// ここでやっている、if文の中で一時的に変数宣言して、その名前により意味を明確化するテクニックは使えそう。
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
		// `ConnContext`はコネクション1つと対応するコンテキスト。BaseContextはServerと対応するのでそこが異なる。
		// コネクションそれぞれにタイムアウトを設定したい場合は、この`ConnContext`のところでやるのかな。
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		// ここで、`net.Conn`をラップした`http.conn`構造体が作られる。
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}
```
