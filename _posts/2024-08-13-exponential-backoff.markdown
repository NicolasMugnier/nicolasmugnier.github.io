---
tags: algorithm
author: Nicolas Mugnier
---

:pushpin: Algorithme "d'exponentiel backoff", mais qu'est-ce que c'est ?

Un algorithme d'exponentiel backoff est un algorithme permettant de temporiser les tentatives d'exécution d'un bout de code.

:grey_question: Quand utiliser ce type d'algorithme ?

L'algorithme d'exponential backoff est à utiliser lorsque l'exécution d'un bout de code provoque une erreur pouvant être considérée comme étant temporaire (à court terme). Dans ce cas il peut être pertinent de relancer l'exécution du code.

Prenons comme exemple concret les requêtes sur une base DynamoDB. Que ce soit au niveau d'une capacité provisionnée ou à la demande, le service d'Amazon impose des limites permettant d'éviter la sur-consommation des resources. La limite se présente sous forme d'un nombre de requêtes par seconde. Une fois le seuil franchit, le service retourne alors une erreur de type "Throttle Request". Dans ce cas il est tout a fait pertinent d'utiliser l'exponential backoff afin de temporiser les requêtes et ainsi de diminuer la pression sur le service.


:bulb: Point d'attention : dans tous les cas, l'utilisation d'un tel algorithme nécessite la mise en place d'une limite permettant de caper le nombre d'essais afin d'éviter une boucle infinie.

Voici un exemple en PHP : 
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

Dans cet exemple, la méthode `__invoke` prend en paramètre l'url d'une API externe. On utilise cette url afin de réaliser un `GET` sur l'API.

Un test particulier est réalisé sur le code HTTP `429`, ce code retourné par le serveur distant indique que celui-ci ne peut plus traiter la demande car trop de requêtes ont été envoyées.

Dans ce genre de cas il peut être pertinent de temposriser la requête et de la rejouer. La temporisation est ici réalisée via l'instruction `usleep()`, le temps d'attente entre deux tentatives croit de façon exponentiel en fonction du nombre de tentatives effectuées.

Dans le cas où le nombre limite de tentatives (`$maxRetry`) a été atteinds la condition de sortie du `do{}wile();` est alors satisfaite et une exception est levée afin d'indiquer que l'appel API n'a pas pu être réalisé.