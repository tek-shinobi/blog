---
title: "Golang Google Cloud PubSub Tutorial"
date: 2023-02-18T20:27:27+02:00
draft: false 
---

brief cheatsheet on doing a simple pubsub publish and subscribe in golang in a synchronous fashion.


```go
package main

import (
	"cloud.google.com/go/pubsub"
	"context"
	"fmt"
	"github.com/google/uuid"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"os"
	"time"
)

const timeout = 10 * time.Second

// GCPubSub wrapper for Google Cloud PubSub
type GCPubSub struct {
	client *pubsub.Client
}

func main() {
	err := os.Setenv("PUBSUB_EMULATOR_HOST", "localhost:8085")
	if err != nil {
		fmt.Println("Error setting env:", err)
		panic(err)
	}
	projectID := "your-project-id"
	ctx, cancelTO := context.WithTimeout(context.Background(), timeout)
	defer cancelTO()
	client, err := NewGCPubSub(ctx, projectID)
	if err != nil {
		fmt.Println("Error creating workflow client:", err)
		panic(err)
	}
	defer client.Close()

	topicID := fmt.Sprintf("topic-%s", uuid.New().String())
	topic, err := client.CreateTopic(ctx, topicID)
	if err != nil && status.Code(err) != codes.AlreadyExists {
		fmt.Println("Error creating topic:", err)
		panic(err)
	}

	defer func(ctx context.Context, topic *pubsub.Topic) {
		// cleanup
		err := topic.Delete(ctx)
		if err != nil {
			fmt.Println("Error deleting topic:", err)
			panic(err)
		}
	}(ctx, topic)

	fmt.Println("Topic created:", topic)

	subID := fmt.Sprintf("sub-%s", uuid.New().String())
	sub, err := client.CreateSubscription(ctx, subID, topic)
	if err != nil && status.Code(err) != codes.AlreadyExists {
		fmt.Println("Error creating subscription:", err)
		panic(err)
	}
	fmt.Println("Subscription created:", sub)
	defer func(ctx context.Context, sub *pubsub.Subscription) {
		err = sub.Delete(ctx)
		if err != nil {
			fmt.Println("Error deleting subscription:", err)
			panic(err)
		}
	}(ctx, sub)

	msg := uuid.New().String()
	payload := pubsub.Message{
		Data: []byte(msg),
		Attributes: map[string]string{
			"origin": "golang",
		},
	}
	fmt.Println("msg:", msg)
	msgID, err := client.Publish(ctx, topic, &payload)
	if err != nil {
		fmt.Println("Error publishing message:", err)
		panic(err)
	}

	fmt.Println("Message published with ID:", msgID)

	ctx, cancel := context.WithCancel(ctx)

	err = client.Subscribe(ctx, sub, func(ctx context.Context, msg *pubsub.Message) {
		defer cancel()
		fmt.Println("Received message:", string(msg.Data))
		msg.Ack()
	})
	if err != nil {
		fmt.Println("Error subscribing to topic:", err)
		panic(err)
	}
	fmt.Println("all done")
}

func NewGCPubSub(ctx context.Context, projectID string) (*GCPubSub, error) {
	client, err := pubsub.NewClient(ctx, projectID)
	if err != nil {
		return nil, err
	}
	return &GCPubSub{client: client}, nil
}

func (g *GCPubSub) Subscribe(ctx context.Context, sub *pubsub.Subscription, f func(ctx context.Context, msg *pubsub.Message)) error {
	return sub.Receive(ctx, f)
}

func (g *GCPubSub) Publish(ctx context.Context, topic *pubsub.Topic, message *pubsub.Message) (string,
	error) {
	// block until the publisher is done publishing
	serverID, err := topic.Publish(ctx, message).Get(ctx)
	if err != nil {
		return "", fmt.Errorf("message publish fail: %w", err)
	}
	return serverID, nil
}

func (g *GCPubSub) Close() error {
	return g.client.Close()
}

func (g *GCPubSub) CreateTopic(ctx context.Context, topicID string) (*pubsub.Topic,
	error) {
	topic := g.client.Topic(topicID)

	exists, err := topic.Exists(ctx)
	if err != nil && status.Code(err) != codes.NotFound {
		return nil, fmt.Errorf("could not verify topic existence: %w", err)
	}
	if !exists {
		topic, err = g.client.CreateTopic(ctx, topicID)
		if err != nil {
			return nil, fmt.Errorf("could not create topic: %w", err)
		}
	}

	return topic, nil
}

func (g *GCPubSub) CreateSubscription(ctx context.Context, subID string, topic *pubsub.Topic) (*pubsub.Subscription, error) {
	sub := g.client.Subscription(subID)

	exists, err := sub.Exists(ctx)
	if err != nil && status.Code(err) != codes.NotFound {
		return nil, fmt.Errorf("could not verify subscription existence: %w", err)
	}

	if !exists {
		sub, err = g.client.CreateSubscription(ctx, subID, pubsub.SubscriptionConfig{
			Topic:                     topic,
			EnableExactlyOnceDelivery: true,
			AckDeadline:               timeout,
		})
		if err != nil {
			return nil, fmt.Errorf("could not create subscription: %w", err)
		}
	}

	return sub, nil
}
```
The steps are:
1.  Create a new client
2. Create a new topic
3. Create a Subscribe object to the topic
4. Publish a message to the topic
5. Subscribe to the topic

>Point 3 is very important. If after we created a topic, we did step 4 and then step 3 and 5, we would get blocked at step 5 (at `err = client.Subscribe`)

Note few points:
-   Messages can be received by a consumer after it subscribes. Any messages published to topic before subscription will **NOT** be received by the consumer.
-   The publish is synchronous, and it will block until the message is published.
-  The subscribe is synchronous, and it will block until the message is received. Note this code snippet that blocks until the message is received:
```go
err = client.Subscribe(ctx, sub, func(ctx context.Context, msg *pubsub.Message) {
        defer cancel()
        fmt.Println("Received message:", string(msg.Data))
        msg.Ack()
    })
```
