# Add a logger

Our application currently prints
the "Starting HTTP server" message to standard out,
and errors to standard error.
Both, standard out and error are also a form of global state.
We should print to a logger object.

We'll use [Zap](https://pkg.go.dev/go.uber.org/zap) in this section of the tutorial
but you should be able to use any logging system.

1. Provide a Zap logger to the application.
   In this tutorial, we'll use [`zap.NewExample`](https://pkg.go.dev/go.uber.org/zap#NewExample),
   but for real applications, you should use `zap.NewProduction`
   or build a more customized logger.

   ```go mdox-exec='region ex/get-started/04-logger/main.go provides'
       fx.Provide(
         NewHTTPServer,
         NewServeMux,
         NewEchoHandler,
         zap.NewExample,
       ),
   ```

2. Add a field to hold the logger on `EchoHandler`,
   and in `NewEchoHandler` add a new logger argument to set this field.

   ```go mdox-exec='region ex/get-started/04-logger/main.go echo-init'
   type EchoHandler struct {
   	log *zap.Logger
   }

   func NewEchoHandler(log *zap.Logger) *EchoHandler {
   	return &EchoHandler{log: log}
   }
   ```

3. In the `EchoHandler.ServeHTTP` method,
   use the logger instead of printing to standard error.

   ```go mdox-exec='region ex/get-started/04-logger/main.go echo-serve'
   func (h *EchoHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   	if _, err := io.Copy(w, r.Body); err != nil {
   		h.log.Warn("Failed to handle request", zap.Error(err))
   	}
   }
   ```

4. Similarly, update `NewHTTPServer` to expect a logger
   and log the "Starting HTTP server" message to that.

   ```go mdox-exec='region ex/get-started/04-logger/main.go http-server'
   func NewHTTPServer(lc fx.Lifecycle, mux *http.ServeMux, log *zap.Logger) *http.Server {
     srv := &http.Server{Addr: ":8080", Handler: mux}
     lc.Append(fx.Hook{
       OnStart: func(ctx context.Context) error {
         ln, err := net.Listen("tcp", srv.Addr)
         if err != nil {
           return err
         }
         log.Info("Starting HTTP server", zap.String("addr", srv.Addr))
         go srv.Serve(ln)
   ```

5. (**Optional**) You can use the same Zap logger for Fx's own logs as well.

   ```go mdox-exec='region ex/get-started/04-logger/main.go fx-logger'
   func main() {
     fx.New(
       fx.WithLogger(func(log *zap.Logger) fxevent.Logger {
         return &fxevent.ZapLogger{Logger: log}
       }),
   ```

   This will replace the `[Fx]` messages with messages printed to the logger.

6. Run the application.

   ```
   {"level":"info","msg":"provided","constructor":"main.NewHTTPServer()","type":"*http.Server"}
   {"level":"info","msg":"provided","constructor":"main.NewServeMux()","type":"*http.ServeMux"}
   {"level":"info","msg":"provided","constructor":"main.NewEchoHandler()","type":"*main.EchoHandler"}
   {"level":"info","msg":"provided","constructor":"go.uber.org/zap.NewExample()","type":"*zap.Logger"}
   {"level":"info","msg":"provided","constructor":"go.uber.org/fx.New.func1()","type":"fx.Lifecycle"}
   {"level":"info","msg":"provided","constructor":"go.uber.org/fx.(*App).shutdowner-fm()","type":"fx.Shutdowner"}
   {"level":"info","msg":"provided","constructor":"go.uber.org/fx.(*App).dotGraph-fm()","type":"fx.DotGraph"}
   {"level":"info","msg":"initialized custom fxevent.Logger","function":"main.main.func1()"}
   {"level":"info","msg":"invoking","function":"main.main.func2()"}
   {"level":"info","msg":"OnStart hook executing","callee":"main.NewHTTPServer.func1()","caller":"main.NewHTTPServer"}
   {"level":"info","msg":"Starting HTTP server","addr":":8080"}
   {"level":"info","msg":"OnStart hook executed","callee":"main.NewHTTPServer.func1()","caller":"main.NewHTTPServer","runtime":"6.292µs"}
   {"level":"info","msg":"started"}
   ```

7. Post a request to it.

   ```shell
   $ curl -X POST -d 'hello' http://localhost:8080/echo
   hello
   ```

**What did we just do?**

We added another component to the application with `fx.Provide`,
and injected that into other components that need to print messages.
To do that, we only had to add a new parameter to the constructors.

In the optional step,
we told Fx that we'd like to provide a custom logger for Fx's own operations.
We used the existing `fxevent.ZapLogger` to build this custom logger from our
injected logger, so that all logs follow the same format.
