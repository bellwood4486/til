# ioutil.ReverseProxyのタイムアウト

ReverseProxyの[`Transport`](https://pkg.go.dev/net/http/httputil#ReverseProxy.Transport)フィールドは未指定だと、デフォルトで[`DefaultTransport`](https://pkg.go.dev/net/http#DefaultTransport)を使う。
この構造体の`Timeout`フィールドがデフォルトのタイムアウトになる。

```go
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```
