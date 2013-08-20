---
layout: post
title: "OAuth2 Explained: Part 4 - Implementing Custom Grant Type with Symfony2 and FOSOAuthServerBundle"
date: 2013-08-20 14:05
comments: true
categories: [OAuth, Symfony2]
published: true
---

# We need something custom

In the previous part we have tested several standard grant-types that come out-of the box with FOSOAuthServerBundle, but probably you will need something more specific for your application. For example it is common to assign specific user API keys to allow access to the application. That way you don't expose user password to the API on the other you can control API keys, make them expire, revoke, etc. 

It's possible to define your custom grant-type, which will authenticate the user based on his API key. 

## Preparations

First, let's modify the User entity. It needs to hold the API key from now on.

<!-- more -->

``` PHP
	<?php 

	namespace Acme/DemoBundle/Entity;

	use Symfony/Component/Security/Core/User/UserInterface;
	use Doctrine/ORM/Mapping as ORM;

	/**
	 * Acme/UserBundle/Entity/User
	 *
	 * @ORM/Table(name="acme_users")
	 * @ORM/Entity(repositoryClass="Acme/DemoBundle/Repository/UserRepository")
	 */
	class User implements UserInterface, /Serializable
	{

		// ... many other great properties on the class

	    /**
	     * @ORM/Column(type="string", length=32)
	     */
	    private $apiKey;

	    /**
	     * @param mixed $apiKey
	     */
	    public function setApiKey($apiKey)
	    {
	        $this->apiKey = $apiKey;
	    }

	    /**
	     * @return mixed
	     */
	    public function getApiKey()
	    {
	        return $this->apiKey;
	    }
	}
```

I'll use built-in doctrine command which ships with Symfony2 Standard Edition for the simplicity

``` bash
	php app/console doctrine:schema:update --force
``` 	

but for the project which will hit the Internets I would highly recommend to use [Doctrine Migrations](https://github.com/doctrine/DoctrineMigrationsBundle) instead.

Use your favorite DB management tool to set some random api keys to some users on the user table.

Now it's time to get to the topic

## Implementing custom Grant Type

You will need a class which implements the logic for verifying the provided apiKey agains database. 

``` php
	<?php

	// src/Acme/DemoBundle/OAuth/ApiKeyGrantExtension.php

	namespace Acme\DemoBundle\OAuth;

	use Doctrine\Common\Persistence\ObjectRepository;
	use FOS\OAuthServerBundle\Storage\GrantExtensionInterface;
	use OAuth2\Model\IOAuth2Client;

	/**
	 * Play at bingo to get an access_token: May the luck be with you!
	 */
	class ApiKeyGrantExtension implements GrantExtensionInterface
	{

	    private $userRepository;

	    public function __construct(ObjectRepository $userRepository)
	    {
	        $this->userRepository = $userRepository;
	    }

	    /*
	     * {@inheritdoc}
	     */
	    public function checkGrantExtension(IOAuth2Client $client, array $inputData, array $authHeaders)
	    {
	        $user = $this->userRepository->findOneByApiKey($inputData['api_key']);

	        if ($user) {
	            //if you need to return access token with associated user
	            return array(
	                'data' => $user
	            );

	            //if you need an anonymous user token
	            return true;
	        }

	        return false;
	    }
	}
```

Now we need to register this class as a service tagged as grant extension in DIC

``` xml

	// src/Acme/DemoBundle/Resources/config/services.xml

	<services>

		...

		<service id="platform.grant_type.api_key" class="Acme\DemoBundle\OAuth\ApiKeyGrantExtension">
			<tag name="fos_oauth_server.grant_extension" uri="http://platform.local/grants/api_key" />
			<argument type="service" id="platform.user.repository"/>
		</service>

		...

	</services>
```

Note: http://platform.local/grants/api_key it must not be a valid url that leads you somewhere, it's just a way to namespace your URL according to the standard. Also it's a reference to your extension as you will see in the next section.

## Time to test

First you need to create a client which grants this custom grant extension

	php app/console acme:oauth-server:client:create --grant-type="http://platform.local/grants/api_key" 

Expected output looks like this

	Added a new client with public id CLIENT_ID, secret CLIENT_SECRET

Next we need to fire an http request against the oauth provider with this credentials and api key

	curl -XGET "http://portal.local/oauth/v2/token?grant_type=http://platform.local/grants/api_key&client_id=CLIENT_ID&client_secret=CLIENT_SECRET&api_key=API_KEY"

If the API_KEY is one of the keys you had set on the user table, you will get success response 

	{"access_token":"OTQ2OTNkY2VkMmI3MzQ4MDUwMTY2YjUwOWZhMjBjYmM5NGI2N2UwNDIwNDhkNTY2MWNlNTk1MmE5MmNhMTJjNA","expires_in":3600,"token_type":"bearer","scope":null,"refresh_token":"NTBkZDgxOGJiYmExYzZhNzQ5MmMwNTZjNjAyYzQzMmU1OTQ2NmRmMzljYzQxNmM3OGQ5ZDhhMjRhMjZiZTZmMA"}

otherwise an

	{"error":"invalid_grant"}

## Summary

As you see implementing custom authorisation mechanisms with OAuth2 and Symfony2 using FOSOAuthServerBundle is really easy. One other example when you need this, is when you implement an API which supports a mobile app, and one of the features is Facebook login on the mobile app. Then you need to somehow login user on the backend as well. The correct way to do this is passing a facebook access token through a custom grant extension to the backend, backend then makes a request to the facebook, to make sure the token is correct, finds out user from it, and gives back the mobile app an access_token with a backend user associated to it. 
