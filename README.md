## Hello World Go RabbitMQ
**This is basic implement rabbitMQ in GO programming.**

To install package RabbitMQ in GO programming

```bash
go get github.com/rabbitmq/amqp091-go
```
* *main source : [RabbitMQ GO tutorial](https://www.rabbitmq.com/tutorials/tutorial-one-go)**
## Sending
In `` producer/main.go ``, we need to import the library first:
```go
package main

import (
  "context"
  "log"
  "time"

  amqp "github.com/rabbitmq/amqp091-go"
)

func errorHanlder(err error) {
    if err != nil {
        fmt.Printf("Error: %v", err)
        os.Exit(1)
    }
}
```
then create the main function and connect to RabbitMQ server
```go
func main() {
  conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
  errorHanlder(err)
  defer conn.Close()
}
```
The connection abstracts the socket connection, and takes care of protocol version negotiation and authentication and so on for us. Next we create a channel, which is where most of the API for getting things done resides:
```go
ch, err := conn.Channel()
errorHanlder(err)
defer ch.Close()
```
To send, we must declare a queue for us to send to; then we can publish a message to the queue:
```go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
errorHanlder(err)

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

body := "Hello World!"
err = ch.PublishWithContext(ctx,
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing {
    ContentType: "text/plain",
    Body:        []byte(body),
  })
errorHanlder(err)

log.Printf(" [x] Sent %s\n", body)
```
Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.
## Receiving
The code (in ` consumer/main.go `) has the same import and helper function as `producer/main.go`:
```go
package main

import (
  "log"

  amqp "github.com/rabbitmq/amqp091-go"
)

func errorHanlder(err error) {
    if err != nil {
        fmt.Printf("Error: %v", err)
        os.Exit(1)
    }
}
```
Setting up is the same as the publisher; we open a connection and a channel, and declare the queue from which we're going to consume. Note this matches up with the queue that send publishes to.
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
errorHanlder(err)
defer conn.Close()

ch, err := conn.Channel()
errorHanlder(err)
defer ch.Close()

q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
errorHanlder(err)
```
Note that we declare the queue here, as well. Because we might start the consumer before the publisher, we want to make sure the queue exists before we try to consume messages from it.

We're about to tell the server to deliver us the messages from the queue. Since it will push us messages asynchronously, we will read the messages from a channel (returned by `amqp::Consume`) in a goroutine.
```go
msgs, err := ch.Consume(
  q.Name, // queue
  "",     // consumer
  true,   // auto-ack
  false,  // exclusive
  false,  // no-local
  false,  // no-wait
  nil,    // args
)
errorHanlder(err)

var forever chan struct{}

go func() {
  for d := range msgs {
    log.Printf("Received a message: %s", d.Body)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
## Putting it all together
Now we can run both scripts. In a terminal, run the publisher:

at project producer `/producer`
```bash
go run main.go
```
then, run the consumer at `/consumer`:
```bash
go run main.go
```
The consumer will print the message it gets from the publisher via RabbitMQ. The consumer will keep running, waiting for messages (Use Ctrl-C to stop it), so try running the publisher from another terminal.