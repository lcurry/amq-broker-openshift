# AMQ OpenShift Configuration

## Demonstrated Capabilities

This repository includes demonstrations of various AMQP messaging scenarios and capabilities implemented using Red Hat AMQ Broker.  We will deploy the broker on openshift with SSL security enabled to allow internal and external clients to produce and consume messages using the Advanced Message Queuing Protocol (AMQP). The AMQP protocal is an open standard application layer protocol for message-oriented middleware. The defining features of AMQP are message orientation, queuing, routing (including point-to-point and publish-and-subscribe), reliability and security.

This repository will provide necessary setup and configuration for the following features:
- Operator-managed AMQ Broker instances
- TLS Certificate management
- Message producer and consumer configuration
- Broker metrics and monitoring using Prometheus and charting using Grafana (TODO)
- Custom init image for advanced broker configuration (TODO)

## Pre-Requisites

This demonstration has the following pre-requisites:

~~AMQ Broker operator "AMQ Broker Operator for RHEL 8 (Multiarch)" has either been installed into the cluster with cluster-wide scope, or has been installed into the namespace \<your namespace\> with namespace scope.~~ Steps to provision the operator are included below

- You have access to an Openshift cluster with the ability to login to the namespace \<your namespace\> with project-admin privaledges
- You have the 'oc' command-line tool available in your local environment
- You have the Java Development Kit (or OpenJDK) 8+ and Maven 3.6.3+ installed locally. (This is only needed to build and run the external java client)

Note: If you are on Windows the paths in these instructions will be different. Make sure you use full paths or current path notation (i.e. remove the ~/ from the path)

## Secret management

Secrets inside the yaml file `/amq-client/DeploymentConfig_fuse-amq-client.yaml` should be edited to include unique values before executing any of the `oc` commands.

Likewise secret names inside the following yaml files should reflect the actual name of the secret that will be created in section ```3. Create secrets in Openshift to hold the keystores for broker and web console```

- /amq-client/ConfigMap_fuse-amq-client.yaml
- /amq-broker-iof/ActiveMQArtemis_broker-iof.yaml

## Provisioning Instructions

### Provisioning the Operator

PreRequisite:
Your account has the ClusterAdmin Role.

To install the AMQ operator Subscription:

``` Shell
Login to the OpenShift
oc.exe login --token=YOUR_TOKEN --server=https://api.ocp-uge1-dev.ecs.us.lmco.com:6443

Switch to your target project
oc.exe project \<YOUR_PROJECT\>

Apply the subscription yaml
oc.exe apply -f ./amq-operator/subscription.yaml
```

The following example commands use a namespace \<your namespace\> where your user has the project-admin role.

### Provision Broker Certificates into namespace

In order to install a broker with SSL enabled (required for external client communication) you must provide a TLS certificate for the AMQ Broker. You must acquire a valid certificate (typically done through Venafi), and then create a secret in Openshift that will store the certificate for a successful TLS handshake between the broker and the client.

The following steps will install the necessary certificates to enable one-way TLS communication with the broker.

The 'keytool' utility can be used to manage certificates and is typically included with any standard Java distributions.

This section assumes two certificates have been created (in PEM PKCS8 format) through Venafi (LMCO's enterprise certificate signing solution). One certificate will be used for the secure AMQP communication from external clients to the broker in Openshift.  The other certificate will be used for browser-based HTTPS communication from external clients to the AMQ web console in Openshift. Certificates obtained through Venafi will guarantee to have been signed by a trusted CA that, if running on a LMCO issued laptop, should be available in a client's local trusted CA list.

### Certificate Creation

While it is possible to create self-signed certificates following the instructions [here](https://access.redhat.com/documentation/en-us/red_hat_amq/2020.q4/html/deploying_amq_broker_on_openshift/assembly-br-configuring-operator-based-deployments_broker-ocp#proc-br-configuring-one-way-tls_broker-ocp]) it is recommended to issue valid certificates through the proper channel - i.e. Venafi - for production applications.

**1. Request 2 certificates for AMQ Broker**, one for the Broker and one for the Web console.

When requesting certificate for the AMQ Broker used the following names below as the Common Name when creating the certificate. The following applies and will be specific based on your \<broker-name\> and \<your-namespace\>.  Broker name is found in the yaml [file](/amq-broker-iof/ActiveMQArtemis_broker-iof.yaml)] used to create the broker, and namespace is the namespace name you are deploying into. Note: To keep it simple, we will make the namespace name and broker name the same.

|                 |  Common Name | Subject Alternative Names  |
|-----------------|--------------|-----------------|
| AMQ Broker |```<broker-name>-amq-0-svc-rte-<your-namespace>.apps.ocp-uge1-dev.ecs.us.lmco.com```|```<broker-name>-amq-0-svc, <broker-name>-amq-0-svc.<your-namespace>.svc, <broker-name>-amq-0-svc.<your-namespace>.svc.cluster.local```|
| Web Console     |```<broker-name>-wconsj-0-svc-rte-<your-namespace>.apps.ocp-uge1-dev.ecs.us.lmco.com```|```<broker-name>-wconsj-0-svc, <broker-name>-wconsj-0-svc.<your-namespace>.svc, <broker-name>-wconsj-0-svc.<your-namespace>.svc.cluster.local```|

You will receive 2 files one for each certificate that should be named as follows:

``` Text

<broker-name>-amq-0-svc-rte-<your-namespace>.apps.ocp-uge1-dev.ecs.us.lmco.com.pfx
<broker-name>-wconsj-0-svc-rte-<your-namespace>.apps.ocp-uge1-dev.ecs.us.lmco.com.pfx

```

You can view the details of the each cert with the following commands:

``` Shell

keytool -list -v -keystore <broker-name>-amq-0-svc-rte-<your-namespace>.apps.ocp-uge1-dev.ecs.us.lmco.com.pfx
keytool -list -v -keystore <broker-name>-wconsj-0-svc-rte-<your-namespace>.apps.ocp-uge1-dev.ecs.us.lmco.com.pfx

```

You should be able to see information such as CommonName, SubjectAlternativeName, Signature algorithm, etc.

**2. Create keystore for broker and web console**

In this step we will create broker and web console keystore from the .pfx *PKCS12* certs. The following command uses the 'keytool' command to create the keystore from the cert for broker and web console.

``` Shell

keytool -importkeystore -srckeystore  <exising cert .pfx file PKCS12 format> -srcalias <Subject name in cert> -srcstorepass <the certs key password> -srcstoretype pkcs12 -destkeystore broker.ks -destalias broker -deststoretype JKS -deststorepass <destination keystore password shuld be same as key trustStorePassword>

```

The same command format (different arguments) can be used to generate the keystore for the web console. Examples of each are given below.

NOTE:  use same password for the new destination keystore that was used for the original key. The keystore password and the cert key password should be the same.

For example here as an example command-line for createing keystore for the broker:

``` Shell

keytool -importkeystore -srckeystore  iof-amq-0-svc-rte-iof.apps.ocp-uge1-dev.ecs.us.lmco.com.pfx -srcalias iof-amq-0-svc-rte-iof.apps.ocp-uge1-dev.ecs.us.lmco.com -srcstorepass <keystore password> -srcstoretype pkcs12 -destkeystore broker.ks -destalias broker -deststoretype JKS -deststorepass <keystore password>

```

The output keystore will go to a file broker.ks

And here as an example command-line for createing keystore for the web console:

``` Shell

keytool -importkeystore -srckeystore  iof-wconsj-0-svc-rte-iof.apps.ocp-uge1-dev.ecs.us.lmco.com.pfx -srcalias iof-wconsj-0-svc-rte-iof.apps.ocp-uge1-dev.ecs.us.lmco.com -srcstorepass <keystore password> -srcstoretype pkcs12 -destkeystore console.ks -destalias console -deststoretype JKS -deststorepass <keystore password>

```

The output keystore will go to a file console.ks

**3. Create secrets in Openshift to hold the keystores for broker and web console**
In this step we will create the secrets in openshift containing the keystore files for broker and web console.

**Note: The instructions for generating self-signed certs in the redhat documentation linked at the start of this process already cover this but you will want to follow the steps below once you have the self signed cert files created.**

Using the 'oc' create the secrets in Openshift to store the TLS credentials. Use the password used when the certificates were created.  You'll need to create two secrets in Openshift to hold each of the keystores.
One secret is used on the route that handles AMQP(S) messaging traffic to the broker (acceptor), and the other secret is used on
the route that will handle HTTPS traffic for the admin console.
Use the following commands to create both serets:
Be sure you run these commands from the same directory where the keystore (.ks) files exist, and to replace \<password\> with your password for the cert, and \<your borker-name\> with the name you used when you created the broker.

``` Shell
oc project <your namespace>

oc create secret generic <your broker-name>-amq-secret --from-file=broker.ks=.\/broker.ks --from-file=client.ts=.\/broker.ks --from-literal=keyStorePassword=<keystore password> --from-literal=trustStorePassword=<truststore password>

oc create secret generic <your broker-name>-amq-secret-wsconsj --from-file=broker.ks=.\/console.ks --from-file=client.ts=.\/console.ks --from-literal=keyStorePassword=<keystore password> --from-literal=trustStorePassword=<truststore password> --from-literal=AMQ_CONSOLE_ARGS="--ssl-key /etc/<your broker-name>-amq-secret-wsconsj-volume/broker.ks --ssl-key-password <truststore password> --ssl-trust /etc/<your broker-name>-amq-secret-wsconsj-volume/client.ts --ssl-trust-password <truststore password>"
```

**NOTE** When creating a secret, OpenShift requires you to specify both a key store and a trust store. The trust store key is generically named client.ts. For one-way TLS between the broker and a client, a trust store is not actually required. However, to successfully generate the secret, you need to specify some valid store file as a value for client.ts. The preceding step provides a "dummy" value for client.ts by reusing the previously-generated broker key store file. This is sufficient to generate a secret with all of the credentials required for one-way TLS.

### Prepare user definition

As a project Admin you may want to create several users for your broker in order to operate the system. You can create them through the Web Console but you can also add them before the brokers creation by editing ```\amq-broker-iof\amq-security.yml```.

In this file you can add as many users as you need as well as define the scope. Note that you have to apply the file ***before you create the broker***

Once you have your user list you can apply the configuration as you would any other with the command:

``` Shell
oc project <your namespace>
oc apply -f amq-broker-iof/amq-security.yml -n <your namespace>
```

### Provision Broker

These steps will deploy the broker CR which should result in a broker being started in \<your namespace\>
The broker is configured to use the secret you generated from the previous command that contains the broker keystore.

``` Shell
oc project <your namespace>
oc apply -f amq-broker-iof/ActiveMQArtemis_broker-iof.yaml -n <your namespace>
```

The following command will give the status of the broker pod as it starts up. Eventually the pod should go to "Running" status.

``` Shell
oc get pods
```

### Known Issues

> :warning: **_NOTE:_**  Due to an issue in the web console, it is not currently possible to use a custom signed certificate in
>the keystore of the web console when using FIPs mode.  A ticket with Red Hat has been
>issued and is being tracked.  An official fix for this issue is in progress, but until
>fix is available you will need apply the following workaround.

Add a JAVA_OPTS environment variable to the init container for the amq broker pod that contains the value " -Dcom.redhat.fips=false".
The below command will add the " -Dcom.redhat.fips=false"  argument to the JAVA_OPTS environment variable for the broker turning off Fips for the broker pod's init container and is a workaround for the issue when changing the default certificate.

``` Shell

oc set env pod/iof-ss-0 -c iof-container-init JAVA_OPTS="-Dcom.redhat.fips=false"

```

Note: above command not working TODO determine correct command.
Instead of setting command line for now you can add/set the env variable on the init-container through the console.
Make sure you select the "iof-container-init" in the drop down (not the "iof-container")
You must restart the pod for this setting to take effect. (Deleting the pod will do this)

### Test the AMQ console
Go to the Openshift console
On the left side navigator, click the project under
Home->Projects->\<your namespace\>
Then on left side navigator
Go to Networking->Routes->
Select the URL under "Location" for the route with "\<your namespace\>-wconsj-0-svc-rte

> :warning: **_NOTE:_**  Due to the issue mentioned previously in the web console regarding the self-signed certificate, you will need to to manually indicate in browser that you wish to accept the self-signed certificate.  This is handled diffently in different browser but usually involves clicking "Advanced" and "proceed to untrusted site."

### Provision Fuse Client Application into openshift for internal client test

Provision the example AMQP producer and consumer application.

``` Shell
oc new-project amq-client
oc apply -f amq-client/ -n amq-client (Do this inside own project)
```

The included BuildConfig will start its first build as soon as it is provisioned. The DeploymentConfig will trigger its first rollout as soon as the build is finished publishing the resulting image.

View the application logs of the example application to observe the producer and consumer perform a round-trip brokered message exchange.

``` Shell
oc get pods
oc logs -f fuse-amq-client-1-v7mg6
```

Note: the last 5 characters on the pod name will be unique to the runtime pod on your system.

You should see similar output to the following:

``` Log
19:39:19.479 INFO  org.redhat.integration.Application - Started Application in 27.801 seconds (JVM running for 32.877)
19:39:20.502 INFO  producer - Test Message #1 at Mon Apr 04 19:39:20 UTC 2022
19:39:20.688 INFO  o.a.q.jms.sasl.SaslMechanismFinder - Best match for SASL auth was: SASL-PLAIN
19:39:20.769 INFO  org.apache.qpid.jms.JmsConnection - Connection ID:a0e983c3-438b-41db-bb69-0c4eff3e2a8b:1 connected to remote Broker: amqps://iof-amq-0-svc.iof.svc.cluster.local:61617
19:39:21.013 INFO  consumer - Test Message #1 at Mon Apr 04 19:39:20 UTC 2022
19:39:21.477 INFO  producer - Test Message #2 at Mon Apr 04 19:39:21 UTC 2022
19:39:21.484 INFO  consumer - Test Message #2 at Mon Apr 04 19:39:21 UTC 2022
19:39:22.476 INFO  producer - Test Message #3 at Mon Apr 04 19:39:22 UTC 2022
19:39:22.488 INFO  consumer - Test Message #3 at Mon Apr 04 19:39:22 UTC 2022
19:39:23.476 INFO  producer - Test Message #4 at Mon Apr 04 19:39:23 UTC 2022
19:39:23.483 INFO  consumer - Test Message #4 at Mon Apr 04 19:39:23 UTC 2022
19:39:24.476 INFO  producer - Test Message #5 at Mon Apr 04 19:39:24 UTC 2022
19:39:24.483 INFO  consumer - Test Message #5 at Mon Apr 04 19:39:24 UTC 2022
```

### Provision Fuse Client Application outside openshift for external client test

Open a new terminal and clone the repo holding test client code

There is a repo holding the test client code. This is the same client code you ran from within Openshift,
except this time you will run the client locally (external to Openshift).

https://gitlab.global.lmco.com/force/paas/middleware/fuse-client-app

See the project README for running the amq test client locally against the broker.
