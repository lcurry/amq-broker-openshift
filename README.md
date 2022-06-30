# AMQ OpenShift Configuration

## Steps to install operator 


// Prep as cluster admin (up front before any user-specific project gets created)

// following as cluster-admin 

// Prep LMCO has 'project-admin' (different from 'admin') 
// so create ClusterRole 'project-admin'
$ oc create -f templates/project-admin.role.yaml 

// Assign project-admin role to user1 
$ oc adm policy add-role-to-user project-admin user1 -n user1-project 

// Create namespace to hold service account 
// this could potentially be created in any namespace (e.g. 'openshift-infra' namespace)
$ oc new-project amq-broker-operator 

// Create 'gitlab-amq-broker-operator-sa' service account in this namespace 
$ oc create sa gitlab-amq-broker-operator-sa -n amq-broker-operator

// create ClusterRole 'amq-broker-operator' (this *clusterrole* yaml comes from the operator package for 7.9.4)
$ oc apply -f templates/cluster_role_amq_broker_operator.yaml 

// Apply ClusterRole to 'gitlab-amq-broker-operator-sa' 
$ oc adm policy add-cluster-role-to-user admin system:serviceaccount:<namespace>:gitlab-amq-broker-operator-sa -n amq-broker-operator

// IS THIS NECESSARY??? Probably since the serviceaccount will need to create similar role and apply it in the user1-project namespace
// Apply ClusterRole 'amq-broker-operator' w/ ClusterRoleBinding 'amq-broker-operator' to 'gitlab-amq-broker-operator-sa' 
$ oc apply -f templates/cluster_role_binding_amq_broker_operator.yaml // apply 'amq-broker-operator' Role to gitlab-amq-broker-operator-sa
// Equivalent command to above without need for defining clusterrolebinding yaml 
// $ oc adm policy add-cluster-role-to-user amq-broker-operator system:serviceaccount:<namespace>:gitlab-amq-broker-operator-sa -n amq-broker-operator

// Create role needed by sa for customresourcedefinitions (not 100% sure this is required)
$ oc create clusterrole amq-broker-operator-customresourcedefinitions-clusterrole --verb=* --resource=customresourcedefinitions.apiextensions.k8s.io --resource-name=activemqartemises.broker.amq.io --resource-name=activemqartemisaddresses.broker.amq.io --resource-name=activemqartemisscaledowns.broker.amq.io --resource-name=activemqartemissecurities.broker.amq.io
// add role to sa 
$ oc adm policy add-cluster-role-to-user  amq-broker-operator-customresourcedefinitions-clusterrole system:serviceaccount:amq-broker-operator:gitlab-amq-broker-operator-sa -n amq-broker-operator 

// Check that serviceaccount has the three roles you added 
$ oc describe clusterrolebinding/admin
$ oc describe clusterrolebinding/amq-broker-operator
$ oc describe clusterrolebinding/amq-broker-operator-customresourcedefinitions-clusterrole

// Deploy the main broker CRD needed for amq-broker-operator (Note: CRDs are cluster-wide resources 
so you only need to perform this step once for entire cluster)
$ oc create -f templates/broker_activemqartemis_crd.yaml
$ oc create -f templates/broker_activemqartemisaddress_crd.yaml
$ oc create -f templates/broker_activemqartemisscaledown_crd.yaml
$ oc create -f templates/broker_activemqartemissecurity_crd.yaml
// to see the new crds 
$ oc get crds | grep amq 

// Get token and login 
$ oc sa get-token -n amq-broker-operator gitlab-amq-broker-operator-sa



// Things to do in PIpeline 
$ oc login --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IndObmJKdEFqQm1tc2w3TDRYNUFMdE1SYzJwWGNNZnJYZXlXWEdlYk9ucncifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJhbXEtYnJva2VyLW9wZXJhdG9yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImdpdGxhYi1hbXEtYnJva2VyLW9wZXJhdG9yLXNhLXRva2VuLWNrdGtwIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImdpdGxhYi1hbXEtYnJva2VyLW9wZXJhdG9yLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiY2FhMjljNjgtZmFlOC00NTg3LThlY2MtNzAyOTMzODI1NDMwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmFtcS1icm9rZXItb3BlcmF0b3I6Z2l0bGFiLWFtcS1icm9rZXItb3BlcmF0b3Itc2EifQ.5VUzl6K7h_E2hROt9D23BgNyrZAUP9EbcaLyREy9fn5ngTDueMvV9dS-0OycKgeNM71jmVUnUn7giV0kN5uBkkT1KPgAk2tGOuhiLTE5aOxXGD0O3v7EsdLY1SgTBr7cC1csp0FIP6ILyXMpgKG3Typ4WbKrIkQKraFCZcjwllDCpygYThIHfcz2-hRSsvOATMnNTLXELbZpZAnlTxu5u3t4JHv_rGPgL1Tvmxq8rko4V_CLvAFWJ7c2yEMa5BHbjQnEInxRBsmvi2rmZCD92pnzIzKgHMzu6hCSJhFp9w_MJMRTkTuNR5CbmkHVaOG7UlkxkeoqropZpodfBcSaKw --server=https://api.cluster-kjmg5.kjmg5.sandbox1612.opentlc.com:6443

$ oc project user1-project 

// Create the amq-broker-operator service account local to project.
    $ oc create serviceaccount amq-broker-operator -n user1-project 
// Create the amq-broker-operator role in your project. (can skip if re-use clusterrole.. see below)
    $ oc create -f templates/role_amq_broker_operator.yaml -n user1-project
// Create the amq-broker-operator role binding in your project. (can skip if re-use clusterrole.. see below)
    $ oc create -f templates/role_binding_amq_broker_operator.yaml -n user1-project
// (TBD why didn't this work?) Equivalent to above 2 commands re-use existing clusterrole (applied as a rolebinding) 
// without need for defining new role & rolebinding yaml 
$ oc adm policy add-role-to-user  amq-broker-operator amq-broker-operator -n user1-project  


// As 'gitlab-amq-broker-operator-sa' install operator 
$ oc apply -f templates/operator.yaml 
// apply proper roles to 'user1'  (applying a clusterrole as a "rolebing" (not clusterrolebiding) so not cluster-wide scope
$ oc adm policy add-role-to-user  amq-broker-operator-customresourcedefinitions-clusterrole user1 -n user1-project  
// add the amq role created previously to allow user view,edit, update on the crds (APis) via crs
$ oc adm policy add-role-to-user  amq-broker-operator user1 -n user1-project  

// Note: Above creates a new clusterrolebinding -0 that you can inspect to confirm clusterrole was added to user1 
$ oc describe rolebinding/amq-broker-operator-customresourcedefinitions-clusterrole -n user1-project


// opentlc-mgr 
oc login --token=sha256~7O1IwoPPUo3IpICsfKXdYmq59T1oJ4R-W4Z5IkIK3mM --server=https://api.cluster-kjmg5.kjmg5.sandbox1612.opentlc.com:6443
// user1 
oc login --token=sha256~fQJhg_caLzLIOpFn0K9klqMYXlCEc_7_ScMoE2OnTGY --server=https://api.cluster-kjmg5.kjmg5.sandbox1612.opentlc.com:6443
// serviceaccount gitlab-amq-broker-operator
oc login --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IndObmJKdEFqQm1tc2w3TDRYNUFMdE1SYzJwWGNNZnJYZXlXWEdlYk9ucncifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJhbXEtYnJva2VyLW9wZXJhdG9yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImdpdGxhYi1hbXEtYnJva2VyLW9wZXJhdG9yLXNhLXRva2VuLWNrdGtwIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImdpdGxhYi1hbXEtYnJva2VyLW9wZXJhdG9yLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiY2FhMjljNjgtZmFlOC00NTg3LThlY2MtNzAyOTMzODI1NDMwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmFtcS1icm9rZXItb3BlcmF0b3I6Z2l0bGFiLWFtcS1icm9rZXItb3BlcmF0b3Itc2EifQ.5VUzl6K7h_E2hROt9D23BgNyrZAUP9EbcaLyREy9fn5ngTDueMvV9dS-0OycKgeNM71jmVUnUn7giV0kN5uBkkT1KPgAk2tGOuhiLTE5aOxXGD0O3v7EsdLY1SgTBr7cC1csp0FIP6ILyXMpgKG3Typ4WbKrIkQKraFCZcjwllDCpygYThIHfcz2-hRSsvOATMnNTLXELbZpZAnlTxu5u3t4JHv_rGPgL1Tvmxq8rko4V_CLvAFWJ7c2yEMa5BHbjQnEInxRBsmvi2rmZCD92pnzIzKgHMzu6hCSJhFp9w_MJMRTkTuNR5CbmkHVaOG7UlkxkeoqropZpodfBcSaKw --server=https://api.cluster-kjmg5.kjmg5.sandbox1612.opentlc.com:6443


// Apply the required roles to user1 (where does the role come from)
// oc adm policy add-role-to-user activemqartemisaddresses.broker.amq.io-v2alpha2-admin  user1 -n user1-project 
// oc adm policy add-role-to-user activemqartemises.broker.amq.io-v2alpha4-admin $USER -n i$NAMESPACE
// oc adm policy add-role-to-user activemqartemisscaledowns.broker.amq.io-v2alpha1-admin $USER -n $NAMESPACE


// Apply CR for broker (tested and works for service account) 
$ oc apply -f templates/broker_activemqartemis_cr.yaml


## Demonstrated Capabilities

This repository includes demonstrations of various AMQP messaging scenarios and capabilities implemented with open source technologies.

- Operator-managed AMQ Broker instances
- TLS Certificate management (TODO)
- Message producer and consumer configuration
- Broker metrics and monitoring using Prometheus and charting using Grafana (TODO)
- Custom init image for advanced broker configuration (TODO)

## Pre-Requisites 

This demonstration has the following pre-requisites:

- AMQ Broker operator "AMQ Broker Operator for RHEL 8 (Multiarch)" has either been installed into the cluster with cluster-wide scope, or has been installed into the namespace \<your namespace\> with namespace scope.
- You have access to an Openshift cluster with the ability to login to the namespace \<your namespace\> with project-admin privaledges
- You have the 'oc' command-line tool available in your local environment 
- You have the Java Development Kit (or OpenJDK) and Maven installed locally. (This is only needed to build and run the external jave client)

## Provisioning Instructions

The following example commands use a namespace \<your namespace\> where your user has the project-admin role.  

### Provision Broker Certificates into namespace

In order to install a broker with SSL enabled (required for external client communication) you must provide a TLS certificate for the AMQ Broker. You must create the necessary certificate and create a secret in Openshift that will store the certificate for a successful TLS handshake between the broker and the client. 

The following steps will install necessary certificates to enable one-way TLS communication with the broker.

The 'keytool' utility can be used to generate self-signed certificates and is typcially provided with any standard JDK/JRE distributions.  Self-signed certificates are adquate for our need to test but for production you should utilize real certificates signed by a trusted CA.
From within the root directory of the git repository, run the following command to generate a self-signed certificate for the broker key store.

```
keytool -genkey -alias broker -keyalg RSA -keystore ./broker.ks
Enter keystore password:  changeit
Re-enter new password: changeit
What is your first and last name?
  [Unknown]:  
What is the name of your organizational unit?
  [Unknown]:  
What is the name of your organization?
  [Unknown]:  
What is the name of your City or Locality?
  [Unknown]:  
What is the name of your State or Province?
  [Unknown]:  
What is the two-letter country code for this unit?
  [Unknown]:  
Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
  [no]:  yes

Enter key password for <broker>
	(RETURN if same as keystore password):  

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore ./broker.ks -destkeystore ./broker.ks -deststoretype pkcs12".

```
When you are first prompted for a password, enter 'changeit' as the keystore password. 
You can accept the defaults (hitting enter) for all the others.  
Finally, type 'yes' at the end to confirm the values you entered. 

Export the certificate from the broker key store, so that it can be shared with clients. Export the certificate in the Base64-encoded pem format. For example:
```
keytool -export -alias broker -keystore ./broker.ks -file ./broker_cert.pem
Enter keystore password: changeit  
Certificate stored in file <./broker_cert.pem>

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore ./broker.ks -destkeystore ./broker.ks -deststoretype pkcs12".
```

Create the client trust store that imports the broker certificate.
```
keytool -import -alias broker -keystore ./client.ts -file ./broker_cert.pem
Enter keystore password:  changeit
Re-enter new password: changeit
Owner: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
Issuer: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
Serial number: 62c4cedd
Valid from: Mon Apr 04 14:13:56 EDT 2022 until: Sun Jul 03 14:13:56 EDT 2022
Certificate fingerprints:
	 MD5:  5D:8F:C9:9A:74:DB:48:86:5C:B6:0D:10:7F:2D:12:B7
	 SHA1: 66:FE:01:AD:E9:91:6C:3D:8E:4F:A7:49:E2:44:8D:30:86:1F:97:0A
	 SHA256: 01:5F:54:12:34:CF:E3:C1:AB:77:12:BF:B4:D5:A6:5B:15:BD:46:E0:AA:EE:A0:C4:23:46:ED:6F:7D:93:6C:F2
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: E6 5B A6 E4 9F 72 74 83   21 05 AD 4C 44 5F F9 3E  .[...rt.!..LD_.>
0010: 7E 26 25 90                                        .&%.
]
]

Trust this certificate? [no]:  yes
Certificate was added to keystore

```

Using the 'oc' command, switch to the namespace that will contain your broker deployment. Create the secrets in Openshift to store the TLS credentials. Use the password you used when you created the keystore.  You'll need to create two secrets holding the same keystore.
One secret is used on the route that handles AMQP(S) messaging traffic to the broker (acceptor), and the other secret is used on
the route that will handle HTTPS traffic for the admin console.  
Use the following commands to create both serets:

```
oc project <your namespace>
oc create secret generic iof-amq-secret --from-file=broker.ks=.\/broker.ks --from-file=client.ts=.\/broker.ks --from-literal=keyStorePassword=changeit --from-literal=trustStorePassword=changeit
oc create secret generic iof-amq-secret-wsconsj --from-file=broker.ks=.\/broker.ks --from-file=client.ts=.\/broker.ks --from-literal=keyStorePassword=changeit --from-literal=trustStorePassword=changeit
```
**NOTE** When creating a secret, OpenShift requires you to specify both a key store and a trust store. The trust store key is generically named client.ts. For one-way TLS between the broker and a client, a trust store is not actually required. However, to successfully generate the secret, you need to specify some valid store file as a value for client.ts. The preceding step provides a "dummy" value for client.ts by reusing the previously-generated broker key store file. This is sufficient to generate a secret with all of the credentials required for one-way TLS.


### Provision Broker 

These steps will deploy the broker CR which should result in a broker being started in \<your namespace\>
The broker is configured to use the secret you generated from the previous command that contains the broker keystore. 
```
oc project <your namespace>
oc apply -f amq-broker-iof/ActiveMQArtemis_broker-iof.yaml -n <your namespace>
```

The following command will give the status of the broker pod as it starts up. Eventually the pod should go to "Running" status. 
```
oc get pods
```

### Test the AMQ console 
Go to the Openshift console
On the left side navigator, click the project under  
Home->Projects->\<your namespace\>
Then on left side navigator 
Go to Networking->Routes->
Select the URL under "Location" for the route with "\<your namespace\>-wconsj-0-svc-rte

### Provision Fuse Client Application into openshift for internal client test

Provision the example AMQP producer and consumer application.

```
oc new-project amq-client
oc apply -f amq-client/ -n amq-client
```

The included BuildConfig will start its first build as soon as it is provisioned. The DeploymentConfig will trigger its first rollout as soon as the build is finished publishing the resulting image.

View the application logs of the example application to observe the producer and consumer perform a round-trip brokered message exchange. 

```
oc get pods
oc logs -f fuse-amq-client-1-v7mg6
```
Note: the last 5 characters on the pod name will be unique to the runtime pod on your system.

You should see similar output to the following:
```
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

https://github.com/lcurry/fuse-client-app

See the project README for running the amq test client locally against the broker.

