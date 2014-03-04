---
layout: post
title: "OAuth2 Explained: Part 5 - Implementing OAuth2 Client with Symfony2"
date: 2014-03-04 18:45
comments: true
categories: [OAuth, Symfony2]
published: true
---

- [Part 1 - Principles and Terminology](http://blog.tankist.de/blog/2013/07/16/oauth2-explained-part-1-principles-and-terminology/)
- [Part 2 - Setting up OAuth2 with Symfony2 using FOSOAuthServerBundle](http://blog.tankist.de/blog/2013/07/17/oauth2-explained-part-2-setting-up-oauth2-with-symfony2-using-fosoauthserverbundle/)
- [Part 3 - Using OAuth2 with your bare hands](http://blog.tankist.de/blog/2013/07/18/oauth2-explained-part-3-using-oauth2-with-your-bare-hands/)
- [Part 4 - Implementing Custom Grant Type](http://blog.tankist.de/blog/2013/08/20/oauth2-explained-part-4-implementing-custom-grant-type-symfony2-fosoauthserverbundle/)
- [Part 5 - Implementing OAuth2 Client with Symfony2](http://blog.tankist.de/blog/2014/03/04/oauth2-explained-part-5-implementing-oauth2-client-with-symfony2/)

Intro
-----
In this article I would like to describe implementation of an OAuth2 Client. Please keep in mind that this is not an authentification provider. To authenticate against third party services there are [well maintained bundles](https://github.com/hwi/HWIOAuthBundle) that do just that. My target is to provide a solution to consume the API from the [OAuth2 Server](https://github.com/iamtankist/oauth-server) we provided in the previous articles.

Overview
-----
For this implementation I had two options. To use [lightweight OAuth2 client library](https://github.com/adoy/PHP-OAuth2) or to implement Guzzle Plugin. I have chosen the first approach, since it covers several grants at once, when Guzzle plugin will solve the more specific problem, and might be a better solution in your specific case.

In this article I will focus on authorization_code and client_credentials grant types.

<!-- more -->

Setup
-----
Let's start by requiring the library from the project root directory
```bash
composer require adoy/oauth2:dev-master
```
This is shorthand for adding the `adoy/oauth2:dev-master` to your `composer.json` file and executing `composer install` afterwards.

In case you don't have a client on the server side, please create one. Execute
```bash
php app/console stark:oauth-server:client:create --redirect-uri="http://oauth-client.local/authorize" --grant-type="authorization_code" --grant-type="password" --grant-type="refresh-token" --grant-type="token" --grant-type="client_credentials"
```
redirect-uri specifies where will the server redirect, to provide the code from authorization_code grant. As OAuth2 consumer we will need to provide an endpoint for that, and the url for that endpoint is specified here. I will cover how to do that later in this article, all you need to know for now is the url for this endpoint.

As a result of execution you will have client_id and client_secret. We will need this right away. Specify the following parameters in parameters.yml.dist
```yaml
# app/config/parameters.yml.dist
parameters:
    # ...
    oauth2_client_id:       3_24slxofityxw0w4o8gsg4ssg0kow4wc8gwccg8ccosg4w0wwgc
    oauth2_client_secret:   2wxfg6tpcuwws8444ksgck8804sg88wowc80s8ww8owc00ckcg
    oauth2_redirect_url:    http://oauth-client.local/authorize
    oauth2_auth_endpoint:   http://oauth-server.local/oauth/v2/auth
    oauth2_token_endpoint:  http://oauth-server.local/oauth/v2/token
```
Now you just need to execute `composer install` and it will apply the parameters into `parameters.yml`.

Since we don't have a bundle for Symfony2 for this library at the moment of writing, we will build a facade for it for our specific needs and register the facade with Symfony2 Dependency Injection Container.

The Facade
------
```php
<?php
// src/StarkIndustries/ClientBundle/Service/OAuth2Client.php
namespace StarkIndustries\ClientBundle\Service;

use OAuth2;

class OAuth2Client
{
    protected $client;
    protected $authEndpoint;
    protected $tokenEndpoint;
    protected $redirectUrl;
    protected $grant;
    protected $params;

    public function __construct(OAuth2\Client $client, $authEndpoint, $tokenEndpoint, $redirectUrl, $grant, $params)
    {
        $this->client = $client;
        $this->authEndpoint = $authEndpoint;
        $this->tokenEndpoint = $tokenEndpoint;
        $this->redirectUrl = $redirectUrl;
        $this->grant = $grant;
        $this->params = $params;
    }

    public function getAuthenticationUrl() {
        return $this->client->getAuthenticationUrl($this->authEndpoint, $this->redirectUrl);
    }

    public function getAccessToken($code = null)
    {
        if ($code !== null) {
            $this->params['code'] = $code;
        }

        $response = $this->client->getAccessToken($this->tokenEndpoint, $this->grant, $this->params);
        if(isset($response['result']) && isset($response['result']['access_token'])) {
            $accessToken = $response['result']['access_token'];
            $this->client->setAccessToken($accessToken);
            return $accessToken;
        }

        throw new OAuth2\Exception(sprintf('Unable to obtain Access Token. Response from the Server: %s ', var_export($response)));
    }

    public function fetch($url)
    {
        return $this->client->fetch($url);
    }
}
```

Now let's register this with DIC.
```xml
<?xml version="1.0" ?>
<!-- src/StarkIndustries/ClientBundle/Resources/config/services.xml-->
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <parameters>
        <parameter key="adoy_oauth2.client.class">OAuth2\Client</parameter>
        <parameter key="stark_industries_client.client.class">StarkIndustries\ClientBundle\Service\OAuth2Client</parameter>
    </parameters>

    <services>
        <service id="adoy_oauth2.client" class="%adoy_oauth2.client.class%">
            <argument>%oauth2_client_id%</argument>
            <argument>%oauth2_client_secret%</argument>
        </service>

        <service id="stark_industries_client.credentials_client" class="%stark_industries_client.client.class%">
            <argument type="service" id="adoy_oauth2.client" />
            <argument>%oauth2_auth_endpoint%</argument>
            <argument>%oauth2_token_endpoint%</argument>
            <argument>%oauth2_redirect_url%</argument>
            <argument>client_credentials</argument>
            <argument type="collection">
                <argument key="client_id">%oauth2_client_id%</argument>
                <argument key="client_secret">%oauth2_client_secret%</argument>
            </argument>
        </service>

        <service id="stark_industries_client.authorize_client" class="%stark_industries_client.client.class%">
            <argument type="service" id="adoy_oauth2.client" />
            <argument>%oauth2_auth_endpoint%</argument>
            <argument>%oauth2_token_endpoint%</argument>
            <argument>%oauth2_redirect_url%</argument>
            <argument>authorization_code</argument>
            <argument type="collection">
                <argument key="redirect_uri">%oauth2_redirect_url%</argument>
            </argument>
        </service>
    </services>
</container>
```
As you can see, we have done the following. Registered the library class itself in the DIC, then we have created two clients with different configurations and parameter sets, for client_credentials and authorization code accordingly. Let's see how one can use them in the code. We will create a console command for client credentials, and controller for authorization code (since it has to provide a uri we mentioned earlier to be redirected to).

Client Credentials Command
----
I tried to keep this pretty much self explanatory

```php
<?php
// src/StarkIndustries/ClientBundle/Command/CredentialsCommand.php
namespace StarkIndustries\ClientBundle\Command;

use OAuth2;
use Symfony\Bundle\FrameworkBundle\Command\ContainerAwareCommand;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class CredentialsCommand extends ContainerAwareCommand
{
    protected function configure()
    {
        $this
            ->setName('oauth2:credentials')
            ->setDescription('Executes OAuth2 Credentials grant');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $credentialsClient = $this->getContainer()->get('stark_industries_client.credentials_client');
        $accessToken = $credentialsClient->getAccessToken();
        $output->writeln(sprintf('Obtained Access Token: <info>%s</info>', $accessToken));

        $url = 'http://oauth-server.local/api/articles';
        $output->writeln(sprintf('Requesting: <info>%s</info>', $url));
        $response = $credentialsClient->fetch($url);
        $output->writeln(sprintf('Response: <info>%s</info>', var_export($response, true)));
    }
}
```
After obtaining access token you can use other HTTP libraries to make the request, Guzzle for instance, just don't forget to append `access_token` GET parameter or transmit a Bearer header, or use one of OAuth2 supported methods to transport the token. Or use the fetch method from the library, as it is done in the example.


Authorization Code Controller
---------------

This one is a bit more complicated. You can split this into two pieces. One that makes the call, other that handles redirect, but for simplicity I combined all into one controller.

```php
<?php
// src/StarkIndustries/ClientBundle/Controller/AuthController.php
namespace StarkIndustries\ClientBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\Request;

// these import the "@Route" and "@Template" annotations
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;

use OAuth2;


class AuthController extends Controller
{
    /**
     * @Route("/authorize", name="auth")
     */
    public function authAction(Request $request)
    {
        $authorizeClient = $this->container->get('stark_industries_client.authorize_client');

        if (!$request->query->get('code')) {
            return new RedirectResponse($authorizeClient->getAuthenticationUrl());
        }

        $authorizeClient->getAccessToken($request->query->get('code'));

        return new Response($authorizeClient->fetch('http://oauth-server.local/api/articles'));
    }
}
```

That's basically everything what you need to consume an OAuth2 API. In case you need a working example, please refer to a [reference implementation](https://github.com/iamtankist/oauth-client).
