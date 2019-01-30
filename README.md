![](atsea_store.png)
#  AtSea Shop Demonstration Application

The AtSea Shop is a demonstration application comprised of: 

* Java REST application written using Spring-Boot, 
* a database for product inventory, customer data, and orders,
* a React shopping cart,
* a payment gateway to simulate certificate management

# Requirements

This example uses features in Docker Enterprise 2.1. Install this version to run the example.

# Building and Running the AtSea Shop

## Secrets

This application uses Docker secrets to secure the application components. The payment requires a password stored as a secret. The database also requires a password store as a secret. To create a password and add as a secret:

```
openssl rand -base64 12 | docker secret create postgres_password -
```

To create a secret for staging the payment gateway:

```
echo staging | docker secret create staging_token - 
```

## Run as an application on Docker Enterprise

To run the AtSea shop as an application:
```
docker stack deploy -c docker-stack.yml atsea
```

## The AtSea Shop 

The URL for the content is `http://localhost:8080/`


## Run on Kubernetes on Docker Desktop Enterprise

To run the AtSea shop as an application:
```
kubectl apply -f test.yaml
```

The URL for the content is `http://localhost:30080/`


# REST API

Documentation for REST calls: [REST API](./REST.md)


