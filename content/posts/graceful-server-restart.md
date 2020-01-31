+++
draft = false
date = 2019-01-03T07:01:27+01:00
title = "Gracefully restarting a GoLang web server"
slug = "graceful-server-restart"
tags = ["golang","development","webserver"]
categories = ["development","server"]
+++

Our team has developed a number of APIs using GoLang, to provide the necessary
functionality for our applications. Naturally we are using the `net/http`
package for the web server part of the APIs, since it is such a powerful
package and provides all that one needs to develop a web server using GoLang.
Each API takes a series of steps when it is booting up, which include reading
and parsing the configuration, setting up loggers, setting up the database
connection pool, starting the web server, preparing the internal router which
routes HTTP requests from the web server to our handler code, and so on. A few
weeks ago, we faced an issue, where we had to manually restart the API due to a
configuration change, which created a problem due to high traffic, and we logged
a couple of failed requests, since we were forced to restart the API, and it was
unable to fulfill requests when it was shut down or was booting up.

After some research we found two acceptable solutions. First one was to develop
a series of CLI commands that would allow us to reload and/or restart certain
parts of the API, or to have the API restart gracefully without cancelling any
open request or dropping any new incoming requests. Since using specific
commands would mean we would have to re-work nearly all parts of the API to
allow for reloads, as well as adding new commands every time we added anything
to the API that would require a reload, we immediately opted for a graceful
restart.

### The theory

In theory, we wanted to start a new instance of the API which would load the
configuration anew, and reconfigure everything needed and start as a fresh copy
of the API in question. In theory this sounds very simple, but since the API is
listening on a TCP port, this complicates things, since we can not have two
processes listening on the same TCP port.

During our research we discovered two possible solutions to this issue:

* set the `SO_REUSERPORT` flag on the socket, allowing multiple processes to
bind to the same port
* duplicate the socket and pass it as a file to the child process and recreate
it there

We opted for the later approach, since it is simpler and being a traditional
UNIX fork/exec spawning model, where all open files of a process are fed to the
child process. Although the `os/exec` package of GoLang allows us to pass only
the `stdin`, `stdout`, and `stderr` to the child process. Thankfully though, the
`os` package provides lower level primitives that can be used to pass additional
files to the child process.

The plan was now relatively simple on paper:

1. listen for the `SIGHUP` signal and trigger the graceful restart when received
2. start a new UNIX domain socket for communication between the parent and child
processes
3. fork the running process passing `stdin`, `stdout`, `stderr`, and the
listener files to the child
4. wait for the child process to load the configuration and prepare everything
for a start
5. request listener metadata information from the parent through the UNIX domain
socket
6. attempt to recreate the socket in the child
7. gracefully shutdown the parent web server

Between points 6 and 7 when both parent and child are using the same socket, the
kernel will be routing all new incoming requests to both processes in a
Round-Robin fashion, until one is stopped. After the parent stops however, the
kernel proceeds to feed only the child with new requests.

### The magic

To achieve the desired result, we take a few extra steps when starting the
server, since we are creating the listening socket separately instead of just
starting the server from the `net/http` package:

```go
package main

import (
	"fmt"
	"net"
	"net/http"
)

var cfg *srvCfg

type srvCfg struct {
	// Socket file location
	sockFile string

	// Listen address
	addr string

	// Listener
	ln net.Listener
}

// create a new listener and return it
func createListener() (net.Listener, error) {
	ln, err := net.Listen("tcp", cfg.addr)
	if err != nil {
		return nil, err
	}

	return ln, nil
}

// obtain a network listener
func getListener() (net.Listener, error) {
	// try to import a listener if we are a fork
	// TODO

	fmt.Println("listener not imported")

	// couldn't import a listener, let's create one
	ln, err := createListener()
	if err != nil {
		return nil, err
	}

	return ln, err
}

// start the server and register the handler
func start(handler http.Handler) *http.Server {
	srv := &http.Server{
		Addr: cfg.addr,
	}

	srv.Handler = handler

	go srv.Serve(cfg.ln)

	return srv
}

// obtain a listener, start the server
func serve(config srvCfg, handler http.Handler) {
	cfg = &config

	var err error
	cfg.ln, err = getListener()
	if err != nil {
		panic(err)
	}

	start(handler)

	// TODO: listen for signals
}

func main() {
	serve(srvCfg{
		sockFile: "/tmp/api.sock",
		addr:     ":8000",
	}, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`Hello, world!`))
	}))
}
```

Next we wanted to add a signal listener, that would listen first only listen for
`SIGTERM` and `SIGINT` signals, and gracefully stop the server with a given
timeout. Note that the examples show only changed sections of code.

```go
import (
	"context"
	"fmt"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

type srvCfg struct {
	// ...

	// Amount of time allowed for requests to finish before server shutdown
	shutdownTimeout time.Duration
}

// ...

// listen for signals
func waitForSignals(srv *http.Server) error {
	sig := make(chan os.Signal, 1024)
	signal.Notify(sig, syscall.SIGTERM, syscall.SIGINT)
	for {
		select {
		case s := <-sig:
			switch s {
			case syscall.SIGTERM, syscall.SIGINT:
				return shutdown(srv)
			}
		}
	}
}

// gracefully shutdown the server
func shutdown(srv *http.Server) error {
	fmt.Println("Server shutting down")

	ctx, cancel := context.WithTimeout(context.Background(),
		cfg.shutdownTimeout)
	defer cancel()

	return srv.Shutdown(ctx)
}

// obtain a listener, start the server
func serve(config srvCfg, handler http.Handler) {
    // ...

	srv := start(handler)

	err = waitForSignals(srv)
	if err != nil {
		panic(err)
	}
}

func main() {
	serve(srvCfg{
		sockFile:        "/tmp/api.sock",
		addr:            ":8000",
        shutdownTimeout: 5 * time.Second,
	}, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`Hello, world!`))
	}))
}
```

To start a graceful restart we opted for the `SIGHUP` signal. When this signal
is received, we create a new UNIX domain socket that will be used for the
communication between the parent and the child processes. The child process will
attempt to connect to this socket once it has started and is ready to start its
own internal web server. On a successful connection it will transmit the
`get_listener` message to the parent. The parent will wait for
`srvCfg.childTimeout` amount of time, before considering the child failed to
start, and it will abort the shutdown of itself.

```go
import (
	"context"
	"fmt"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

type srvCfg struct {
	// ...

	// Amount of time allowed for a child to properly spin up and request the listener
	childTimeout time.Duration
}

// ...

// accept connections on the socket
func acceptConn(l net.Listener) (c net.Conn, err error) {
	chn := make(chan error)
	go func() {
		defer close(chn)
		c, err = l.Accept()
		if err != nil {
			chn <- err
		}
	}()

	select {
	case err = <-chn:
		if err != nil {
			fmt.Printf("error occurred when accepting socket connection: %v\n",
				err)
		}

	case <-time.After(cfg.childTimeout):
		fmt.Println("timeout occurred waiting for connection from child")
	}

	return
}

// create a new UNIX domain socket and handle communication
func socketListener(chn chan<- string, errChn chan<- error) {
	ln, err := net.Listen("unix", cfg.sockFile)
	if err != nil {
		errChn <- err
		return
	}
	defer ln.Close()

	// signal that we created a socket
	chn <- "socket_opened"

	// accept
	c, err := acceptConn(ln)
	if err != nil {
		errChn <- err
		return
	}

	// read from the socket
	buf := make([]byte, 512)
	nr, err := c.Read(buf)
	if err != nil {
		errChn <- err
		return
	}

	data := buf[0:nr]
	switch string(data) {
	case "get_listener":
		fmt.Println("get_listener received - sending listener information")
		// TODO: send listener
		chn <- "listener_sent"
	}
}

// handle SIGHUP - create socket, fork the child, and wait for child execution
func handleHangup() error {
	c := make(chan string)
	defer close(c)
	errChn := make(chan error)
	defer close(errChn)

	go socketListener(c, errChn)

	for {
		select {
		case cmd := <-c:
			switch cmd {
			case "socket_opened":
				// TODO: fork

			case "listener_sent":
				fmt.Println("listener sent - shutting down")

				return nil
			}

		case err := <-errChn:
			return err
		}
	}

	return nil
}

// listen for signals
func waitForSignals(srv *http.Server) error {
	sig := make(chan os.Signal, 1024)
	signal.Notify(sig, syscall.SIGHUP, syscall.SIGTERM, syscall.SIGINT)
	for {
		select {
		case s := <-sig:
			switch s {
			case syscall.SIGHUP:
				err := handleHangup()
				if err == nil {
					// no error occured - child spawned and started
					return shutdown(srv)
				}

			// ...
			}
		}
	}
}

func main() {
	serve(srvCfg{
		sockFile:        "/tmp/api.sock",
		addr:            ":8000",
		shutdownTimeout: 5 * time.Second,
		childTimeout:    5 * time.Second,
	}, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`Hello, world!`))
	}))
}
```

When the UNIX domain socket is created, the `socketListener` function will send
the `socket_opened` message through the channel, signaling `handleHangup`
function that the socket has been created. Once the socket is created, we can go
ahead and fork the process, passing it handles of opened files including the
listener file handle, by using the `os.StartProcess` function.

```go
import (
	"context"
	"fmt"
	"net"
	"net/http"
	"os"
	"os/signal"
	"path/filepath"
	"syscall"
	"time"
)

// obtain the *os.File from the listener
func getListenerFile(ln net.Listener) (*os.File, error) {
	switch t := ln.(type) {
	case *net.TCPListener:
		return t.File()

	case *net.UnixListener:
		return t.File()
	}

	return nil, fmt.Errorf("unsupported listener: %T", ln)
}

// fork the process
func fork() (*os.Process, error) {
	// get the file descriptor and pack it up in the metadata
	lnFile, err := getListenerFile(cfg.ln)
	if err != nil {
		return nil, err
	}
	defer lnFile.Close()

	// pass the stdin, stdout, stderr, and the listener files to the child
	files := []*os.File{
		os.Stdin,
		os.Stdout,
		os.Stderr,
		lnFile,
	}

	// get process name and dir
	execName, err := os.Executable()
	if err != nil {
		return nil, err
	}
	execDir := filepath.Dir(execName)

	// spawn a child
	p, err := os.StartProcess(execName, []string{execName}, &os.ProcAttr{
		Dir:   execDir,
		Files: files,
		Sys:   &syscall.SysProcAttr{},
	})
	if err != nil {
		return nil, err
	}

	return p, nil
}

// handle SIGHUP - create socket, fork the child, and wait for child execution
func handleHangup() error {
	// ...

	for {
		select {
		case cmd := <-c:
			switch cmd {
			case "socket_opened":
				p, err := fork()
				if err != nil {
					fmt.Printf("unable to fork: %v\n", err)
					continue
				}
				fmt.Printf("forked (PID: %d), waiting for spinup", p.Pid)

		// ...
		}
	}

	return nil
}
```

The child process should now have started, but it will fail to start completely
since it will attempt to bind a new socket to the same address. To prevent this
the child attempts to connect to the opened UNIX domain socket and signal the
parent with the message `get_listener`. The parent will then transmit all the
metadata of the listener to the child, and signal the `handleHangup` function
with the `listener_sent` message, which will then proceed with a graceful
shutdown. The child has already received the listener file when it was started
as a fork, but it must still attempt to recreate/rebuild the underlying
`*os.File` for the listener.


```go
import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net"
	"net/http"
	"os"
	"os/signal"
	"path/filepath"
	"sync"
	"syscall"
	"time"
)

type listener struct {
	// Listener address
	Addr string `json:"addr"`

	// Listener file descriptor
	FD int `json:"fd"`

	// Listener file name
	Filename string `json:"filename"`
}

// import the listener from the UNIX domain socket and attempt to
// recreate/rebuild the underlying *os.File
func importListener() (net.Listener, error) {
	c, err := net.Dial("unix", cfg.sockFile)
	if err != nil {
		return nil, err
	}
	defer c.Close()

	var lnEnv string
	wg := sync.WaitGroup{}
	wg.Add(1)
	go func(r io.Reader) {
		defer wg.Done()

		buf := make([]byte, 1024)
		n, err := r.Read(buf[:])
		if err != nil {
			return
		}

		lnEnv = string(buf[0:n])
	}(c)

	_, err = c.Write([]byte("get_listener"))
	if err != nil {
		return nil, err
	}

	wg.Wait()

	if lnEnv == "" {
		return nil, fmt.Errorf("Listener info not received from socket")
	}

	var l listener
	err = json.Unmarshal([]byte(lnEnv), &l)
	if err != nil {
		return nil, err
	}
	if l.Addr != cfg.addr {
		return nil, fmt.Errorf("unable to find listener for %v", cfg.addr)
	}

	// the file has already been passed to this process, extract the file
	// descriptor and name from the metadata to rebuild/find the *os.File for
	// the listener.
	lnFile := os.NewFile(uintptr(l.FD), l.Filename)
	if lnFile == nil {
		return nil, fmt.Errorf("unable to create listener file: %v", l.Filename)
	}
	defer lnFile.Close()

	// create a listerer with the *os.File
	ln, err := net.FileListener(lnFile)
	if err != nil {
		return nil, err
	}

	return ln, nil
}

// obtain a network listener
func getListener() (net.Listener, error) {
	// try to import a listener if we are a fork
	ln, err := importListener()
	if err == nil {
		fmt.Printf("imported listener file descriptor for addr: %s\n", cfg.addr)
		return ln, nil
	}

	// ...
}

// send listener file metadata over the UNIX domain socket
func sendListener(c net.Conn) error {
	lnFile, err := getListenerFile(cfg.ln)
	if err != nil {
		return err
	}
	defer lnFile.Close()

	l := listener{
		Addr:     cfg.addr,
		FD:       3,
		Filename: lnFile.Name(),
	}

	lnEnv, err := json.Marshal(l)
	if err != nil {
		return err
	}

	_, err = c.Write(lnEnv)
	if err != nil {
		return err
	}

	return nil
}

// create a new UNIX domain socket and handle communication
func socketListener(chn chan<- string, errChn chan<- error) {
	// ...

	switch string(data) {
	case "get_listener":
		fmt.Println("get_listener received - sending listener information")

		err := sendListener(c)
		if err != nil {
			errChn <- err
			return
		}

		chn <- "listener_sent"
	}
}
```

Now when the child starts and is ready to start its web server it will attempt
to recreate/rebuild the `*os.File` of the received listener file and signal the
parent that it has reached this point. The parent then proceeeds by attempting
to gracefully shutdown, allowing all open connections to the parent up to
`srvCfg.shutdownTimeout` amount of time to finish up, if all requests have
finished processing before that, the server will shut down, if the timeout is
reached, it will be killed, regardless of the state or amount of requests still
open.

### Conclusion

This is a more simplified representation of the idea, and should be used with
caution, althought this example does work. Since the code is broken up into many
snippets here, I have prepared a repository on GitHub with the similar code
[here](https://github.com/slax0rr/go-httpserver).

There are still some missing features, checks and precautions in this example,
and it might lead to instability or crashes in your application, should you
choose to use this code as is.
