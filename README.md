[![Build Status](https://travis-ci.org/IBM/spring-boot-microservices-on-kubernetes.svg?branch=master)](https://travis-ci.org/IBM/spring-boot-microservices-on-kubernetes)

# Build and deploy Java Spring Boot microservices on Kubernetes

*Note*: This in an abbreviated version of the IBM Code Pattern [Build and deploy Java Spring Boot microservices on Kubernetes](https://github.com/IBM/spring-boot-microservices-on-kubernetes)

Spring Boot is one of the popular Java microservices framework. Spring Cloud has a rich set of well integrated Java libraries to address runtime concerns as part of the Java application stack, and Kubernetes provides a rich featureset to run polyglot microservices. Together these technologies complement each other and make a great platform for Spring Boot applications.

In this code we demonstrate how a simple Spring Boot application can be deployed on top of Kubernetes. This application, Office Space, mimicks the fictitious app idea from Michael Bolton in the movie [Office Space](http://www.imdb.com/title/tt0151804/). The app takes advantage of a financial program that computes interest for transactions by diverting fractions of a cent that are usually rounded off into a seperate bank account.

The application uses a Java 8/Spring Boot microservice that computes the interest then takes the fraction of the pennies to a database. Another Spring Boot microservice is the notification service. It sends email when the account balance reach more than $50,000. It is triggered by the Spring Boot webserver that computes the interest. The frontend uses a Node.js app that shows the current account balance accumulated by the Spring Boot app. The backend uses a MySQL database to store the account balance.

## Flow

![spring-boot-kube](images/architecture.png)

1. The Transaction Generator service written in Python simulates transactions and pushes them to the Compute Interest microservice.
2. The Compute Interest microservice computes the interest and then moves the fraction of pennies to the MySQL database to be stored. The database can be running within a container in the same deployment or on a public cloud such as IBM Cloud.
3. The Compute Interest microservice then calls the notification service to notify the user if an amount has been deposited in the user’s account.
4. The Notification service  sends an email message to the user.
5. The user retrieves the account balance by visiting the Node.js web interface.

## Featured Technologies

* [Container Orchestration](https://www.ibm.com/cloud/container-service): Automating the deployment, scaling and management of containerized applications.
* [Databases](https://en.wikipedia.org/wiki/IBM_Information_Management_System#.22Full_Function.22_databases): Repository for storing and managing collections of data.
*
# Prerequisite

* Create a Kubernetes cluster with either [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) for local testing, [IBM Cloud Private](https://github.com/IBM/Kubernetes-container-service-GitLab-sample/blob/master/docs/deploy-with-ICP.md), or with [IBM Cloud Kubernetes Service](https://github.com/IBM/container-journey-template) to deploy in cloud. The code here is regularly tested against [Kubernetes Cluster from IBM Cloud](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov) using Travis.



# Steps
1. [Clone the repo](#1-clone-the-repo)
2. [Create the Database service](#2-create-the-database-service)
3. [Create the Spring Boot Microservices](#3-create-the-spring-boot-microservices)
4. [Deploy the Microservices](#5-deploy-the-microservices)
5. [Access Your Application](#6-access-your-application)

### 1. Clone the repo

Clone this repository. In a terminal, run:

```
$ git clone https://github.com/djccarew/spring-boot-microservices-on-kubernetes
$ cd spring-boot-microservices-on-kubernetes
```

### 2. Create the Database service

The backend consists of a MySQL database and the Spring Boot app. Each
microservice has a Deployment and a Service. The deployment manages
the pods started for each microservice. The Service creates a stable
DNS entry for each microservice so they can reference their
dependencies by name.

* Use MySQL in container

```bash
$ kubectl create -f account-database.yaml
service "account-database" created
deployment "account-database" created
```
Default credentials are already encoded in base64 in secrets.yaml.
> Encoding in base64 does not encrypt or hide your secrets. Do not put this in your Github.

```
$ kubectl apply -f secrets.yaml
secret "demo-credentials" created
```


### 3. Create the Spring Boot Microservices
You will need to have [Maven installed in your environment](https://maven.apache.org/index.html).
If you want to modify the Spring Boot apps, you will need to do it before building the Java project and the docker image.

The Spring Boot Microservices are the **Compute-Interest-API** and the **Send-Notification**.

**Compute-Interest-API** is a Spring Boot app configured to use a MySQL database. The configuration is located in `compute-interest-api/src/main/resources/application.properties` in `spring.datasource.*`

The `application.properties` is configured to use MYSQL_DB_* environment variables. These are defined in the `compute-interest-api.yaml` file. It is already configured to get the values from the Kubernetes Secrets that was created earlier.

The **Send-Notification** can be configured to send notification through gmail. The notification is sent when the account balance on the MySQL database goes over $50,000.


* Using default email service (gmail) with Notification service

To use the email feature, you will need to modify the **environment variables** in the `send-notification.yaml`:
```yaml
    env:
    - name: GMAIL_SENDER_USER
       value: 'username@gmail.com' # ask your instructor for this info
    - name: GMAIL_SENDER_PASSWORD
       value: 'password' # ask your instructor for this info
    - name: EMAIL_RECEIVER
       value: 'sendTo@gmail.com' # change this to your email address
```


### 4. Deploy the Microservices

* Deploy Spring Boot Microservices

```bash
$ kubectl apply -f compute-interest-api.yaml
service "compute-interest-api" created
deployment "compute-interest-api" created
```

```bash
$ kubectl apply -f send-notification.yaml
service "send-notification" created
deployment "send-notification" created
```

* Deploy the Frontend service

The UI is a Node.js app serving static files (HTML, CSS, JavaScript) that shows the total account balance.

```bash
$ kubectl apply -f account-summary.yaml
service "account-summary" created
deployment "account-summary" created
```

* Deploy the Transaction Generator service
The transaction generator is a Python app that generates random transactions with accumulated interest.

Create the transaction generator **Python** app:
```bash
$ kubectl apply -f transaction-generator.yaml
service "transaction-generator" created
deployment "transaction-generator" created
```

### 5. Access Your Application
You can access your app publicly through your Cluster IP and the NodePort. The NodePort should be **30080**.


* To find the NodePort of the account-summary service:
```bash
$ kubectl get svc
NAME                    CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
...
account-summary         10.10.10.74    <nodes>       80:30080/TCP                                                                 2d
...
```
* On your browser, go to `http://<your-cluster-IP>:30080`
![Account-balance](images/balance.png)

If you set up email notification make sure you received an email when your balance went above $50K

## Troubleshooting
* To start over, delete everything: `kubectl delete svc,deploy -l app=office-space`


## References
* [John Zaccone](https://github.com/jzaccone) - The original author of the [office space app deployed via Docker](https://github.com/jzaccone/office-space-dockercon2017).
* The Office Space app is based on the 1999 film that used that concept.

## License
This code pattern is licensed under the Apache Software License, Version 2.  Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](http://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](http://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
