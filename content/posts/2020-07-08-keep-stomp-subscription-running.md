---
title: Keep Stomp Subscription Running With Server Sent Heart Beat
date: 2020-07-08
categories:
---

Recently I get a chance to use Apache ActiveMQ in one or our services. One of the conditions that we have is that the ActiveMQ server is behind a load balancer. This makes keeping long running connection to the ActiveMQ server impossible as AWS load balancer default times out in 60s if there is no data transfer happens between the two parties.

## Solution

In order to keep the connection active and to be able to instantly receive any message in the subscribed queue we must enable a ping / pong mechanism. In ApacheMQ context it is known as Heart Beat.

Following in the [PHP Client](https://github.com/stomp-php/stomp-php) for STOMP protocol we will tell the server that we expect from you to send us a heart beat every 5000ms.

The benefit of this is that ApacheMQ will send us a heartbeat request every 5 seconds, hence for the AWS load balancer it is considered as an active connection.

```php
use Stomp\Client;
use Stomp\StatefulStomp;
use Stomp\Network\Observer\ServerAliveObserver;

class StompFactory
{
    public static function create()
    {
        $client = new Client("apachemq-host");
        $client->setLogin("user", "pass");
        $client->setHeartbeat(0, 5000);

        $observer = new ServerAliveObserver();
        $client->getConnection()->getObservers()->addObserver($observer);

        $stomp = new StatefulStomp($client);

        return $stomp;
    }
}

```

ApacheMQ also gives us option to send heart beat from client side, but that is the topic for another post :)
