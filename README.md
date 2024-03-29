# respwriter package
[![Go Reference](https://pkg.go.dev/badge/github.com/jimlambrt/respwriter/respwriter.svg)](https://pkg.go.dev/github.com/jimlambrt/respwriter)
[![Go Report Card](https://goreportcard.com/badge/github.com/jimlambrt/respwriter)](https://goreportcard.com/report/github.com/jimlambrt/respwriter)

<hr>

`respwriter` is a Go pkg that provides a meikg/dns.HandlerFunc with support for request
timeouts via a dns.ResponseWriter wrapper.  Among other things, it provides: 

* `NewHandler(...)`:  Creates a new dns.HandlerFunc that wraps the given handler
  with a RespWriter. The returned handler will use the given logger and
  requestTimeout to create the RespWriter. 
* `NewRespWriter(...)`: Creates a RespWriter which is a wrapper around
  dns.ResponseWriter that provides "base" capabilities for the wrapped writer.
  Among other things, this is useful for ensuring that the wrapped writer is not
  used after the context is canceled. 


## Example 

[./exampes/simple/main.go](./examples/simple/main.go)

```go
  import (
	"fmt"
	"net"
	"time"

	"github.com/jimlambrt/respwriter"
	"github.com/miekg/dns"
)

func main() {
	mux := dns.NewServeMux()
	// wrap the handler with a 100ms timeout
	handlerWithTimeout, err := respwriter.NewHandlerFunc(100*time.Millisecond, new(dnsHandler).ServeDNS)
	if err != nil {
		fmt.Printf("Failed to create handler: %s\n", err.Error())
		return
	}
	mux.HandleFunc(".", handlerWithTimeout)
	pc, err := net.ListenPacket("udp", ":0")

	if err != nil {
		fmt.Printf("Failed to start listener: %s\n", err.Error())
		return
	}

	server := &dns.Server{
		PacketConn: pc,
		Net:        "udp",
		Handler:    mux,
		UDPSize:    65535,
		ReusePort:  true,
	}

	fmt.Printf("Starting DNS server on %s\n", pc.LocalAddr())
	err = server.ListenAndServe()
	if err != nil {
		fmt.Printf("Failed to start server: %s\n", err.Error())
		return
	}
}

type dnsHandler struct{}

func (h *dnsHandler) ServeDNS(w dns.ResponseWriter, r *dns.Msg) {
	_, ok := w.(*respwriter.RespWriter)
	if !ok {
		// this cannot happen given the way we're using
		// respwriter.NewHandlerFunc to wrap ServeDNS
		fmt.Println("Failed to cast to RespWriter")
	}
	msg := new(dns.Msg)
	msg.SetReply(r)
	msg.Authoritative = true
	for _, question := range r.Question {
		fmt.Printf("Received query: %s\n", question.Name)
		answers := resolve(question.Name, question.Qtype)
		msg.Answer = append(msg.Answer, answers...)
	}
	w.WriteMsg(msg)
}

func resolve(domain string, qtype uint16) []dns.RR {
	m := new(dns.Msg)
	m.SetQuestion(dns.Fqdn(domain), qtype)
	m.RecursionDesired = true

	c := new(dns.Client)
	in, _, err := c.Exchange(m, "8.8.8.8:53")
	if err != nil {
		fmt.Println(err)
		return nil
	}
	return in.Answer
}
```
  