---
id: 42
layout: post
title: "Better logs in Symfony"
perex: '''
    Monolog is useful library for managing logs. 
    But when it comes to adding more information to your logs, things can easily become tricky.
'''
author: 15
lang: en
related_posts: [41]
---

Monolog offers great way to modify your logs by adding [processors](https://github.com/Seldaek/monolog/blob/master/doc/02-handlers-formatters-processors.md#processors) to your handlers.
Every log coming to the handler is processed by all processors specified for this handler.

## Adding WebProcessor

Very useful processor is [WebProcessor](https://github.com/Seldaek/monolog/blob/master/src/Monolog/Processor/WebProcessor.php), 
which adds these information to every log: `url`, `ip`, `http_method`, `server`, `referrer`. 
[Symfony has class](https://github.com/symfony/symfony/blob/master/src/Symfony/Bridge/Monolog/Processor/WebProcessor.php) 
extending this processor by using information from Request. 
It has onKernelRequest method, so it can be registered as EventListener.

To use this processor, we need to register it as service and tag it.
 
```yaml
services:
    Symfony\Bridge\Monolog\Processor\WebProcessor:
        tags:
            - { name: monolog.processor, handler: main }
            - { name: kernel.event_listener, event: kernel.request, priority: 100 }
```  

In the `monolog.procesor` tag, we specify, which handler will use this formatter.
In the `kernel.event_listener` tag, we attach it to `kernel.request` tag and specify the priority. 

There are already [many services](http://symfony.com/doc/current/reference/events.html#kernel-request) in the Symfony application listening to the `kernel.request` event.

So the priority 100 means this service listens between the `SessionListener` and the `RouterListener`.

## Adding session and user ID

In the [Symfony documentation](https://symfony.com/doc/current/logging/processors.html#adding-a-session-request-token) there is
example Processor for adding session token. 

But in the read world application, we also want to have user ID and username in our request.

So we modify this example SessionRequestProcessor to read user id from the TokenStorage.
```php
<?php declare(strict_types=1);

namespace AppBundle\Monolog;

use Symfony\Component\HttpFoundation\Session\SessionInterface;
use Symfony\Component\Security\Core\Authentication\Token\AnonymousToken;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;

class SessionRequestProcessor {
	/** @var SessionInterface */
	private $session;
	/** @var TokenStorageInterface */
	private $tokenStorage;

	public function __construct(SessionInterface $session, TokenStorageInterface $tokenStorage) {
		$this->session = $session;
		$this->tokenStorage = $tokenStorage;
	}

	/**
	 * @param  mixed[] $record
	 * @return mixed[]
	 */
	public function processRecord(array $record): array {
		try {
			$tokenString = substr($this->session->getId(), 0, 10);
		} catch (\RuntimeException $e) {
			$tokenString = '??????????';
		}

		$record['extra']['token'] = $tokenString;
		$token = $this->tokenStorage->getToken();
		if ($token !== null && (!$token instanceof AnonymousToken)) {
			$user = $token->getUser();
			$record['extra']['user_id'] = (string) $user->getId();
			$record['extra']['user_username'] = $user->getUsername();
		} else {
			$record['extra']['user_id'] = 'not logged in';
			$record['extra']['user_username'] = '';
		}

		return $record;
	}
}
```

In this example, I assume you have User object with methods `getId()` and `getUsername()`.

and then we register it as a service:

```yaml
services:
    AppBundle\Monolog\SessionRequestProcessor:
        tags:
            - { name: monolog.processor, method: processRecord, handler: main }
```

But there is one downside in this setup. Token with authenticated user is loaded to the TokenStorage during the `kernel.request`, 
in the Firewall, which is the last listener for this event. This results in TokenStorage not having the token with user before this moment.

Luckily, Monolog has handler, which can help us addressing this issue. [BufferHandler](https://github.com/Seldaek/monolog/blob/master/src/Monolog/Handler/BufferHandler.php) 
is a wrapping handler with a wrapped handler specified, it holds all logs until the end of request, 
and then it sends all logs to the wrapped handler at once. 

The `SessionRequestProcessor` is used by main handler, which means it processes logs when they are sent to the handler.

With the `BufferHandler`, processor is called in the end of script execution, when the token with authenticated user is already loaded.
So we wrap our main handler in the configuration:

```yaml
monolog:
    handlers:
        buffer:
            type:    buffer
            handler: main
        main:
        	...
        ...
```

and voilà, all logs have username and id of authenticated user now.