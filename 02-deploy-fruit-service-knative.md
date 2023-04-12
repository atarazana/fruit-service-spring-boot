# Deploying the Fruit Service as a Knative Service

## Create database and label

export DEV_PROJECT=fruits-dev
oc label dc/postgresql-db app.kubernetes.io/part-of=fruit-service-app --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/postgresql-db app.openshift.io/runtime=postgresql --overwrite=true -n ${DEV_PROJECT}

## Knative service from git

x.1 Add -> From Git
    Git Repo URL: https://github.com/atarazana/fruit-service-spring-boot
    Java 11
    General Application: fruit-service-app 
    Name: fruit-service-kn-git <===
    Check Knative Service <===
    Click on Deployment to add env variables
    - DB_HOST: postgresql-db
    - DB_USER from secret postgresql-db...
    - DB_PASSWORD from secret postgresql-db...
    - DB_NAME from secret postgresql-db...
    - JAVA_OPTIONS: -Dspring.profiles.active=openshift-postgresql
    Click on BuildConfig <=== We need to do this because we have two different openshift profiles for oracle and postgresql
    - MAVEN_ARGS: -Popenshift-postgresql
    Click on labels
    - app=fruit-service-kn
    - version=1.0.0

DEV_PROJECT=fruits-dev 
oc label ksvc/fruit-service-kn-git app.openshift.io/runtime=spring --overwrite=true -n ${DEV_PROJECT} && \
oc annotate ksvc/fruit-service-kn-git app.openshift.io/connects-to=postgresql-db --overwrite=true -n ${DEV_PROJECT}

## Create a default worker

Let's create a default broker

kn broker create default

kn broker list

## Create a trigger from broker to fruit-service-kn with 

type: fruit-in-event
source: fruits-market

## Send out event to trigger function!

export DEV_PROJECT=fruits-dev
BROKER_URL=$(oc get broker/default -n ${DEV_PROJECT} -o jsonpath='{.status.address.url}')
curl -v $BROKER_URL \
  -H "Ce-specversion: 1.0" \
  -H "Ce-Id: 121212121212" \
  -H "Ce-Type: fruit-in-event" \
  -H "Ce-Source: fruits-market" \
  -H "Ce-User: user2" -H 'Content-Type: application/json' -d '{ "name": "Kiwi" }'

## See results, there should be a new fruit!


## Deploy a new version

REVISION_NAME=v3 kn service update fruit-service-kn --env VERSION=${REVISION_NAME} --revision-name ${REVISION_NAME}

kn service update fruit-service-kn-git --image image-registry.openshift-image-registry.svc:5000/fruits-dev/fruit-service-kn-git --revision-name v6

## Canary Release

