## About CQRS - Command Query Responsibility Segregation

According with [Martin Folwer](https://martinfowler.com/bliki/CQRS.html)
> At its heart is the notion that you can use a different model to update information than the model you use to read information.
> For some situations, this separation can be valuable, but beware that for most systems CQRS adds risky complexity.

## The application

Simulates a bank account scenario where an end user adds a income or expense transaction, and it is processed in a ascyncronous event sourcing and CQRS architecture to recalculate the user's bank account balance. The user can also request the balance of it's account. Down here you can see the design:

![Design](/images/design.png)

## Deploying the external services

```
docker-compose up -d --build
```
It will deploy four docker containers on your environment with MongoDB, PostgreSQL, Kafka and Zookepper (required by Kafka)

After deploying Kafka, you'll need to [create the topic on the Kafka cluster](https://kafka.apache.org/quickstart). For example:

```
docker exec -it bankaccount-kafka \
  ./bin/kafka-topics.sh --create \
  --topic transactions \
  --zookeeper bankaccount-zookeeper:2181 \
  --replication-factor 1 \
  --partitions 1
```

## Testing the application

#### Running a CURL request to create a income transaction
```
curl -X POST -H "Content-Type: application/json" -d @income-transaction.json http://localhost:8080/transactions
```
#### Running a CURL request to create a expense transaction
```
curl -X POST -H "Content-Type: application/json" -d @expense-transaction.json http://localhost:8080/transactions
```
#### Running CURL request to fetch the balance
```
curl http://localhost:8081/balance\?accountId\=wesley | json_pp
```
#### Running [K6's](https://k6.io) simple performance test
````
k6 run --vus 10 --duration 60s performance-tests/income.js
k6 run --vus 10 --duration 60s performance-tests/expense.js
````

## Deploying to Minikube - local kubernetes

Run the following commands in order to deploy the app on *minikube*:

### Running all dependencies

1. Create minikube cluster and enable ingress: `minikube -p dev.to start --cpus 2 --memory=4096 && minikube -p dev.to addons enable ingress`
2. Open minikube's dashboard: `minikube -p dev.to dashboard`
3. Open another terminal, enter on folder *kubefiles* and run the following commands, one by one:
```
kubectl apply -f 01-namespace.yml
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka 
kubectl apply -f 02-postgres-deployment.yml
kubectl apply -f 03-mongo-deployment.yml
```

### Running transaction-service and balance-service

*transaction-service*

Replace the content of the following properties on `application.properties`
```
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://POSTGRES_IP_ON_MINIKUBE:5432/bankaccount
%prod.mp.messaging.outgoing.transactions.bootstrap.servers: KAFKA_IP_ON_MINIKUBE:9092
%prod.kafka.bootstrap.servers: KAFKA_IP_ON_MINIKUBE:9092
```

*balance-service*

Replace the content of the following properties on `application.properties`
```
%prod.quarkus.mongodb.connection-string = mongodb://MONGO_IP_ON_MINIKUBE:27017
%prod.quarkus.kafka-streams.bootstrap-servers=KAFKA_IP_ON_MINIKUBE:9092
%prod.quarkus.kafka-streams.application-server=KAFKA_IP_ON_MINIKUBE:8080
```

Build project and build docker images for both projects.

Upload docker images to minikube cache, so it finds the image and do not need to download the image from docker registry:

```
minikube -p dev.to image load quarkus/balance-service-jvm:latest
minikube -p dev.to image load quarkus/transaction-service-jvm:latest
``` 

List all images to make sure the images we built were successfully added to minikube cache:

```
minikube -p dev.to image list
```

Use the command `minikube -p dev.to ip` to find out the minikube ip and map it as `dev.local` on `/ect/hosts`

Finally, apply the deployment file:
```
kubectl apply -f 04-transaction-app-deployment.yml
kubectl apply -f 05-balance-app-deployment.yml
kubectl apply -f 06-ingress.yml
```

Test the endpoints using the examples `expense-transaction.json` and `income-transaction.json`

### Contributing
I'd love to have a frontend for it! [Please reach me out if you got interested](MailTo:wesley.fuchter@gmail.com)
