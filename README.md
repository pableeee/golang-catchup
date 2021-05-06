# Interfaces

- Go no es un lenguaje OOP
- No soporta herencia (JAG approves)
- Soporta interfaces (polimorfismo)
- La implementacion de una interfaz no es explicita

`interface{}` seria el equivalente a `java.lang.Object`.

Cualquier tipo "implementa" un interfaz vacia

# Type Assertion

El operador para el type assertion es "`.(type)`"

```go
func main() {
  var i interface{} = "hello"

  s := i.(string)
  fmt.Println(s)

  s, ok := i.(string)
  fmt.Println(s, ok)

  f, ok := i.(float64)
  fmt.Println(f, ok)

  f = i.(float64) // panic
  fmt.Println(f)
} 
```

## Type switches

Esto seria como un switch mezclado con un `instanceof`. Adicionalmente, dentro del `case` la variable ya fue convertida al tipo del case.

```go
func main() {
  do(21)
  do("hello")
  do(true)
}

func do(i interface{}) {
  switch v := i.(type) {
  case int:
    // v ya esta convertido a int
    fmt.Printf("Twice %v is %v\n", v, v*2)
  case string:
    // v ya esta convertido a string
    fmt.Printf("%q is %v bytes long\n", v, len(v))
  default:
    fmt.Printf("I don't know about type %T!\n", v)
  }
}
```

# Error Handling

Golang es un poco verboso con los errores. Basicamente, cualquier metodo/funcion que puede fallar, retorna como ultimo parametro un `error`. `error` es una interface

```go
type error interface {
  Error() string
}
```

## Sentinel errors

Los sentinel errors (errores definidos estaticamente), forman parte del API pública porque el usuario del api va a comparar contra ese error.

```go
var ErrInvalidKey = errors.New("invalid key")

func (s *service) Get(key string) (*Value, error) {
  if len(key) == 0 {
    return nil, ErrInvalidKey
  }
}

...
if err := service.Get(""); err == ErrFoo {
  // handles ErrInvalidKey
}
```

> A la hora de comprar contra sentinel errors, ss preferible usar `errors.Is` al operator `==`

## Custom Error types

Crear un tipo de error custom que implemente la interface `error`, y pueda tener informacion relevante.

```go
type BarError struct {
  Reason string
}

func (e BarError) Error() string {
  return fmt.Sprintf("bar error: %s", e.Reason)
}
```

## Wrapping errors

Es una buena practica, no retornar errores que devuelvan otras API, sino mas bien wrappearlos y que devuelvan adicianalmente un contexto mayor del fallo. En ese caso se tiene que proveer adicionalmente la forma de hacer el `Unwrap`.

```go
type BazError struct {
  Reason string
  Inner  error
}

func (e BazError) Error() string {
  if e.Inner != nil {
    return fmt.Sprintf("baz error: %s: %v", e.Reason, e.Inner)
  }
  return fmt.Sprintf("baz error: %s", e.Reason)
}

func (e BazError) Unwrap() error {
  return e.Inner
}
```

Tambien hay formas de wrappear errores, ni necesidad de usar un custom error type.

```go
func process(j Job) error {
  result, err := preprocess(j)
  if err != nil {
    // wrapeamos el error de preprocess con un mensaje para dar contexto del error.
    return fmt.Errorf("error preprocessing job: %w", err)
  }
  ...
```

> By default, when you encounter an error in a function and need to return it to the caller, wrap it with some context about what went wrong, using `fmt.Errorf` and the new `%w` verb.

Para validad si un error wrappea a otro error en particular, usamos `errors.Is` que cheaquea recursivamente si internamente el error es de ese tipo.

```go
err := f()
if errors.Is(err, ErrFoo) {
  // you know you got an ErrFoo
  // respond appropriately
}
```

Si queremos validar si un `error` es de un tipo en particular (type assertion), es preferible utilizar `errors.As` a utilizar el operador de type assertion `.()`.

```go
var bar *BarError
if errors.As(err, &bar) {
  // you know you got a BarError
  // bar's fields are populated
  // respond appropriately
}
```

En el ejemplo de arriba, `As` retorna `true` si `err` es del tipo `BarError` y dentro del cuerpo del if, la variable `bar` esta correctamente seteada.

# Concurrency

## mutex (semaforos)

```go
type ConcurrentCounter struct {
  mutex *sync.Mutex
  count int
}

func (c *ConcurrentCounter) Add(value int) int {
  mutex.Lock()
  defer mutex.Unlock()
  c.count += 1
  
  return c.count
}
```

Tambien existen tambien lost `RWMutex`, que permiten ser mas granulares con las primitivas de acceso.

```go
func (c *ConcurrentCounter) Get() int {
  mutex.RLock()
  defer mutex.RUnlock()
  
  return c.count
}
```

En este caso, multiples go routines pueden obtener el `RLock` de forma simultanea.

## Waitgroup

```go
func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }

    // llamada bloqueante q espera todos los wg.Done()
    // similar a un join para threads/procesos
    wg.Wait()
}

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("Worker %d starting\n", id)

    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}
```

# Channels

- Son colas de comunicacion *_sincrónica_* entre go routines
- Tienen multiples formas de uso
- Tanto los reads como los writes son bloqueantes

Algunos ejemplos

## Como implementar Futures con channels

```go
future := make(chan int, 1)
go func() { future <- process() }()
result := <-future
```

## Como implementar Async + await

```go
c := make(chan int, 1)
go func() { c <- process() }() // async
v := <-c                       // await
```

## Scatter + Gather

```go
// Scatter
c := make(chan result, 10)
for i := 0; i < cap(c); i++ {
    go func() {
        val, err := process()
        c <- result{val, err}
    }()
}

// Gather
var total int
for i := 0; i < cap(c); i++ {
    res := <-c
    if res.err != nil {
        total += res.val
    }
}
```

## Fan-out

```go
func main() {
  concurrency := 10 
  
  // el segundo parametro, es para poder escribir 
  c := make(chan int, concurrency)
  // lanzo el fan-out con 10 go routines
  fanOut(c, concurrency)

  for i := 0; i < 100; i ++ {
    // mando los valores en el channel de entrada
    c <- i
  }

  // cierro el channel para que los workers salgan del for + range
  close(c)
}

func fanOut(input <-chan int, concurrency int) {
  for i := 0; i < concurrency; i++ {
    go func() {
      for value := range input {
        // process value
        process(value)
      }
    }()
  }
}
```

## Fan-in

```go
func fanIn(input1, input2 <-chan int ) <-chan int {
  c := make(chan int)
  go func() {
    for {
      select {
        case s:= <- input1
          c <- s
        case s:= <- input2
          c <- s
      }
    }
  }()
  return c
}
```

> Hay formas de hacer esto mismo, pero mas genérico con un [array de channels](https://medium.com/justforfunc/two-ways-of-merging-n-channels-in-go-43c0b57cd1de)

# Repository Structure

The basic idea is to have two top-level directories, pkg and cmd. Underneath pkg, create directories for each of your libraries. Underneath cmd, create directories for each of your binaries. All of your Go code should live exclusively in one of these locations.

```bash
github.com/peterbourgon/foo/
  circle.yml
  Dockerfile
  cmd/
    foosrv/
      main.go
    foocli/
      main.go
  pkg/
    fs/
      fs.go
      fs_test.go
      mock.go
      mock_test.go
    merge/
      merge.go
      merge_test.go
    api/
      api.go
      api_test.go
```

# Dep Injection

```go
type UserRepository interface {
  GetUser(id string) (*User, error)
  // tambien podria devolver un User
}

type RetrieveUser struct {
  repo UserRepository
}

// Constructor para dep injection
func NewRetrieveUser(repo UserRepository) *RetrieveUser {
  return &RetrieveUser{repo: repo}
}

func (r *RetrieveUser) Retrieve(id string) (*User, error) {
  return r.repo.GetUser(id)
}
```

# Context & cancel propagation

## Funciones del pacakge context

```go
// retorna el context y la funcion q puedo llamar para cancelarlo
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// retorna contextos que se cancelan automaticamente expirado el tiempo definido. La funcion de cancel se usar para liberar los recursos de timers alocados para los TOs.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// Esta dos funciones retornan un context "vacio", aunque tienen propositos semánticos de uso distintos.
func Background() Context
func TODO() Context

// context con valores, similar a lo que vemos en flux
func WithValue(parent Context, key, val interface{}) Context
```

## Ejemplo de cancelacion con timeout

```go
// recibo un context por parametro
func main() {
  // retorna un context generico "vacio"
  ctx := context.Background()

  // creo un context con timeout despues de 3 segundos
  ctx, cancel := context.WithTimeout(ctx, time.Second * 3)
  // me aseguro de llamar la func cancel para liberar los recursos de los timers del TO
  defer cancel()

  // hago la llamada a foo, con el ctx + TO
  foo(ctx)
}

func foo(ctx context.Context) {
  // creo el channel para la respuesta
  future := make(chan int, 1)
  
  go func() { 
    // process invoca un api call que demora algunos segundos
    future <- process()
  }()
  
  // el select ejecuta el primer evento que llegue de alguno de estos dos channel
  select {
    case value <- future:
      // proceso la respuesta del api call
      fmt.Println(value)
    case <- ctx.Done():
      // se vencio el TO antes q finalice la llamada a process
      fmt.Println("el context se cancelo antes que termine el api call")
  }
}
```
