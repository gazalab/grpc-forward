# grpc-forward
gRPC library to forward data to all connected clients.

## Summary
1. [Problem](#problem)

### Problem
Let's say you have the following `.proto` file:
```proto
message Empty {}

message MessageResponse {
	uint32 id = 1;
	string msg = 2;
}

service Message {
 rpc Subscribe(Empty) returns (stream MessageResponse) {}
}
```

Your server looks like this:
```golang
type MessageServer struct {
	proto.UnimplementedMessageServer
	messageChan chan *proto.MessageResponse
}

func (m *MessageServer) Start() {
    for i := 0; i < 300; i++ {
        m.messageChan <- &proto.MessageResponse{
            Id:  uint32(i),
            Msg: fmt.Sprintf("Message %d", i),
        }
        time.Sleep(time.Second)
    }
}

func (m *MessageServer) Subscribe(_ *proto.Empty, stream proto.Message_SubscribeServer) error {

	for {
		msg, ok := <-m.messageChan
		if ok {
			if err := stream.Send(msg); err != nil {
				return err
			}
		} else {
			return fmt.Errorf("chan not ok")
		}
	}
}

func main() {
    // bootstraping everything

	m := &MessageServer{
		messageChan: make(chan *proto.MessageResponse, 1),
	}

    go m.Start()

    // start server
}
```

And your client looks like this:
```golang
stream, err := client.Subscribe(context.Background(), empty)
if err != nil {
    panic(err)
}
for {
    msg, err := stream.Recv()
    if err != nil {
        panic(err)
    }
    fmt.Println("ID: %d | Message: %s", msg.Id, msg.Msg)
}
```

What if, you need to forward a single `MessageResponse` to all connected clients?
If you have multiple connected clients, each of them would receive a different message, that's the problem that `grpc-forward` solves.

