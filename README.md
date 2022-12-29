# BAMOE-KieServer-TestTimerOpenshift

work in progress....

## Env var for timers data storage into db


```
-e DATASOURCES= \
-e DB_SERVICE_PREFIX_MAPPING=node1-postgresql=RHPAM
```

## Deploy Timer application

### Create a dedicated namespace

```
TNS=bamoe-kieservers-timers
oc new-project ${TNS}
```

### Deploy Operator CR

```
cat << EOF | oc create -f - 
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: ${TNS}-operator-
  name:  ${TNS}-operator
  namespace: ${TNS}
spec:
  targetNamespaces:
    - ${TNS}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: bamoe-businessautomation-operator
  namespace: ${TNS}
spec:
  channel: 8.x-stable
  installPlanApproval: Automatic
  name: bamoe-businessautomation-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: bamoe-businessautomation-operator.8.0.1-2
EOF
```

### Deploy KieApp CR 


```
KIE_ENV_TYPE=rhpam-authoring
KIE_APP_NAME=my-app-timers
KIE_ADMIN_USER=admin
KIE_PASSWORD=passw0rd
KIE_DB_SRV_PRE_MAP=${KIE_APP_NAME}-postgresql=RHPAM
KIE_SRV_ID=${KIE_APP_NAME}-srv-id
KIE_REPLICAS=3
```

```
cat << EOF | oc create -f -
apiVersion: app.kiegroup.org/v2
kind: KieApp
metadata:
  name: ${KIE_APP_NAME}
  namespace: ${TNS}
spec:
  commonConfig:
    amqPassword: ${KIE_PASSWORD}
    disableSsl: true
    adminPassword: ${KIE_PASSWORD}
    amqClusterPassword: ${KIE_PASSWORD}
    dbPassword: ${KIE_PASSWORD}
    adminUser: ${KIE_ADMIN_USER}
    applicationName: ${KIE_APP_NAME}
    keyStorePassword: ${KIE_PASSWORD}
  objects:
    servers:
      - env:
          - name: DATASOURCES
          - name: DB_SERVICE_PREFIX_MAPPING
            value: ${KIE_DB_SRV_PRE_MAP}
        database:
          type: postgresql
        jbpmCluster: true
        deployments: 1
        id: ${KIE_SRV_ID}
        replicas: ${KIE_REPLICAS}
  environment: ${KIE_ENV_TYPE}
EOF
```

```
oc get KieApp
```

### Show the configuration file in pod

```
ONE_OF_KIE_SRV=$(oc get pods -n ${TNS} | grep ${KIE_APP_NAME}-kieserver | grep -v postgres | grep Running | head -1 | awk '{print $1}')

oc cp ${ONE_OF_KIE_SRV}:/opt/eap/standalone/configuration/standalone-openshift.xml ./${ONE_OF_KIE_SRV}-standalone-openshift.xml

grep -i "xa-datasource" ./${ONE_OF_KIE_SRV}-standalone-openshift.xml

grep -i -m 1 -A 6 "timer-service" ./${ONE_OF_KIE_SRV}-standalone-openshift.xml
```


### Get server urls

```
KIE_SERVER_URL=http://$(oc get route ${KIE_APP_NAME}-kieserver -o jsonpath='{.spec.host}')
BC_SERVER_URL=http://$(oc get route ${KIE_APP_NAME}-rhpamcentr -o jsonpath='{.spec.host}')
```


### Deploy app

In BC import

https://github.com/marcoantonioni/BAMOE-KieServer-TestTimer1


attendere dopo deploy ...nuovi pod: my-app-timers-kieserver-2...


### Test app

a cluster pulito eseguire un solo avvio e attendere esito nei log, buone probabilità di vedere avvio su un server e terminazione su altro, dimostra bilanciamento di carico

avvio su thread pool default

08:32:08,837 INFO [stdout] (default task-1) ===> TestTimer1 ENTRY pid[1] delay[PT22S]

terminazione su thread pool EJB

08:54:08,373 INFO [stdout] (EJB default - 1) ===> TestTimer1 EXIT pid[1] delay[PT22S]


```
# set env vars
USER_PASSWORD=${KIE_ADMIN_USER}:${KIE_PASSWORD}
SERVER_URL=${KIE_SERVER_URL}
CTR_ID=TestTimer1_1.0.0-SNAPSHOT
PROCESS_TEMPL_ID="TestTimer1.TestTimer1"
```

```
# run a single instance
MIN_SECS=5
RANGE_SECS=30
DELAY=$(( $RANDOM % $RANGE_SECS + $MIN_SECS ))
PROC_INSTANCE_ID=$(curl -s -k -u ${USER_PASSWORD} -H 'content-type: application/json' -H 'accept: application/json' -X POST ${SERVER_URL}/services/rest/server/containers/${CTR_ID}/processes/${PROCESS_TEMPL_ID}/instances -d "{\"delay\":\"PT${DELAY}S\"}")
echo "PID: "${PROC_INSTANCE_ID}
```


## db queries

```
ONE_OF_PG_SRV=$(oc get pods -n ${TNS} | grep ${KIE_APP_NAME}-kieserver-postgresql | grep Running | head -1 | awk '{print $1}')
oc rsh ${ONE_OF_PG_SRV} /bin/bash

oc exec ${ONE_OF_PG_SRV} -- psql -d rhpam7 -c "select id, partition_name, node_name from jboss_ejb_timer where node_name != '';"

oc exec ${ONE_OF_PG_SRV} -- psql -d rhpam7 -c "select count (timerId) from TimerMappingInfo;"

oc exec ${ONE_OF_PG_SRV} -- psql -d rhpam7 -c "select count (status) from processinstancelog where status = 1;"

oc exec ${ONE_OF_PG_SRV} -- psql -d rhpam7 -c "select count (status) from processinstancelog where status = 2;"

oc exec ${ONE_OF_PG_SRV} -- psql -d rhpam7 -c "select count (status) from processinstancelog where status = 3;"


```

