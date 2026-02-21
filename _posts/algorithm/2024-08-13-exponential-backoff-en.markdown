---
tags: algorithm
author: Nicolas Mugnier
categories: algorithm
title: Exponential Backoff Algorithm
description: Overview of the exponential backoff algorithm, its purpose, and when and how to use it.
image: /assets/img/exponential-backoff.png
locale: en_US
---

An overview of the exponential backoff algorithm, its purpose, and when and how to use it.

An exponential backoff algorithm is an algorithm that progressively increases the delay between retry attempts when executing a piece of code.

:grey_question: When should you use this type of algorithm?

The exponential backoff algorithm should be used when a code execution produces an error that can be considered temporary (short-lived). In such cases, it may be relevant to retry the execution.

Let's take a concrete example: queries against a DynamoDB database. Whether using provisioned or on-demand capacity, Amazon's service imposes limits to prevent resource overconsumption. The limit is expressed as a number of requests per second. Once the threshold is exceeded, the service returns a "Throttle Request" error. In this case, it makes perfect sense to use exponential backoff to throttle requests and reduce the pressure on the service.

:bulb: Important note: in all cases, using such an algorithm requires setting a maximum number of retries to avoid an infinite loop.

Here is an example in PHP:
```php
<?php

declare(strict_types=1);

interface HttpClientInterface
{
    public function get(string $url): HttpResponseInterface;
    public function post(string $url, array $body): HttpResponseInterface;
}

interface HttpResponseInterface
{
    public function getCode(): int;
}

readonly class ServiceGateway
{
    public function __construct(private HttpClientInterface $httpClient)
    {
    }

    public function __invoke(string $url): HttpResponseInterface
    {
        $retries = 0;
        $maxRetry = 5;

        do {
            if ($retries > 0) {
                $this->wait($retries);
            }
            $response = $this->httpClient->get($url);
        } while ($response->getCode() === 429 && $retries++ < $maxRetry);

        if ($response->getCode() === 429) {
            throw new \Exception('Too many requests');
        }

        return $response;
    }

    private function wait(int $retries): void
    {
        $waitTime = \pow(2, $retries) * 100;
        \usleep($waitTime * 1000);
    }
}
```

In this example, the `__invoke` method takes the URL of an external API as a parameter. This URL is used to perform a `GET` request against the API.

A specific check is performed on the HTTP `429` status code — this code returned by the remote server indicates that it can no longer process the request because too many requests have been sent.

In such cases, it makes sense to throttle the request and replay it. The throttling is achieved here via the `usleep()` instruction, where the wait time between two attempts grows exponentially based on the number of attempts made.

If the maximum number of retries (`$maxRetry`) has been reached, the `do{}while();` exit condition is satisfied and an exception is thrown to indicate that the API call could not be completed.
