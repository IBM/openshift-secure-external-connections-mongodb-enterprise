# Secure external connections to MongoDB Enterprise on Red Hat OpenShift

In my previous tutorial [Secure MongoDB Enterprise on Red Hat OpenShift](https://developer.ibm.com/tutorials/secure-mongo-db-enterprise-on-red-hat-openshift), I walk you through deploying and securing a MongoDB cluster. That tutorial only covers the use of the MongoDB cluster from resources inside the cluster. There are times when you would also want to expose the cluster externally so that other resources outside your cluster can directly connect to it.

Implementing tight security in your container environment, particularly when exposing a database or other resources externally, is imperative. Encryption and authentication ensure that only the right person or resources can access the cluster, which protects your application's user data as well. The previous tutorial covers setting up encryption and authentication for your MongoDB cluster.

This tutorial shows you how to:

- Expose a MongoDB database from MongoDB Enterprise Operator
- Use a public domain name for the MongoDB instance
- Configure a certificate to use the public domain name
- Connect to a MongoDB instance outside of OpenShift or Kubernetes

# Prerequisites

* OpenShift cluster
* OpenShift CLI (oc)

# Steps
1. [Complete the previous tutorial "Secure MongoDB Enterprise on Red Hat OpenShift"](#1-Complete-the-previous-tutorial-Secure-MongoDB-Enterprise-on-Red-Hat-OpenShift)
2. [Expose MongoDB replicas through NodePort](#2-Expose-MongoDB-replicas-through-NodePort)
3. [Configure domain name](#3-Configure-domain-name)
4. [Configure existing certificates](#4-Configure-existing-certificates)
5. [Configure MongoDB Database resource](#5-Configure-MongoDB-Database-resource)
6. [Test connection from client outside of OpenShift or Kubernetes](#6-Test-connection-from-client-outside-of-OpenShift-or-Kubernetes)

## 1. Complete the previous tutorial "Secure MongoDB Enterprise on Red Hat OpenShift"

Complete the steps outlined in the [Secure MongoDB Enterprise on Red Hat OpenShift](https://developer.ibm.com/tutorials/secure-mongo-db-enterprise-on-red-hat-openshift) tutorial to install the MongoDB Enterprise operator, deploy MongoDB resources, and secure the MongoDB database with authentication and encryption.

## 2. Expose MongoDB replicas through NodePort

Now that you have the MongoDB replica set ready<!--EM: is this done in the first tutorial?-->, you can expose its replicas through NodePort. You can do this in the command line:

```
oc expose pod/example-mongo-0 --type="NodePort" --port 27017
oc expose pod/example-mongo-1 --type="NodePort" --port 27017
oc expose pod/example-mongo-2 --type="NodePort" --port 27017
```

You should see the following services created:

```
oc get services

NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)           AGE
example-mongo-0        NodePort       172.21.174.50    <none>         27017:31385/TCP   72m
example-mongo-1        NodePort       172.21.218.87    <none>         27017:30131/TCP   72m
example-mongo-2        NodePort       172.21.170.154   <none>         27017:30656/TCP   72m
...
```

## 3. Configure domain name

The previous tutorial only covered the use of MongoDB internally, so the certificate you generated in that tutorial is only valid for the internal domain name.

You can choose to do one of the following:

* [Use a public domain](#Use-a-public-domain)
* [Use a cluster node's IP address for testing](#Use-a-cluster-node's-IP-address-for-testing)

### Use a public domain

If you have a public domain ready for use, configure its A records <!--EM: is this common? I see the Example A below, but I keep reading this and thinking--what are A records? I might try and reword to introduce the example A here first--> to point to the IP addresses of your cluster nodes.

To get the cluster IP of your nodes:

```
oc get nodes -o wide

NAME           STATUS   ROLES           AGE   VERSION           INTERNAL-IP    EXTERNAL-IP      OS-IMAGE   KERNEL-VERSION                CONTAINER-RUNTIME
10.221.22.40   Ready    master,worker   66d   v1.17.1+c5fd4e8   10.221.22.40   169.xx.xx.xx     Red Hat    3.10.0-1127.19.1.el7.x86_64   cri-o://1.17.5-7.rhaos4.4.git6b97f81.el7
10.221.22.44   Ready    master,worker   66d   v1.17.1+c5fd4e8   10.221.22.44   150.xx.xx.xx     Red Hat    3.10.0-1127.19.1.el7.x86_64   cri-o://1.17.5-7.rhaos4.4.git6b97f81.el7
```

Example A record from a DNS Registrar:

![a-records](docs/images/arecord.png)

You can now proceed to the next step when your domain name is now pointing to the cluster nodes.

### Use a cluster node's IP address for testing

If you don't have a readily available public domain, you can use one of the cluster node's IP addresses with [nip.io](https://nip.io/). nip.io allows you to map the IP address to a hostname.

Get one of the cluster IP of your nodes:

```
oc get nodes -o wide

NAME           STATUS   ROLES           AGE   VERSION           INTERNAL-IP    EXTERNAL-IP      OS-IMAGE   KERNEL-VERSION                CONTAINER-RUNTIME
10.221.22.40   Ready    master,worker   66d   v1.17.1+c5fd4e8   10.221.22.40   169.xx.xx.xx     Red Hat    3.10.0-1127.19.1.el7.x86_64   cri-o://1.17.5-7.rhaos4.4.git6b97f81.el7
10.221.22.44   Ready    master,worker   66d   v1.17.1+c5fd4e8   10.221.22.44   150.xx.xx.xx     Red Hat    3.10.0-1127.19.1.el7.x86_64   cri-o://1.17.5-7.rhaos4.4.git6b97f81.el7
```

Thee hostname you use for the next steps will be `<cluster-ip-address>.nip.io` (For example, `169.123.123.123.nip.io`).

## 4. Configure existing certificates

You can now add your domain name to the certificates you created from Step 1. You need to do this so that the certificate will be considered valid.

1. Edit the `cert-manager/certificates.yaml` from your cloned repo <!--EM: Where did they clone the repo? We've gotten some feedback that says people don't know  exactly what that means, so I wanted to make sure it was explicitly clearn in the above steps-->. Add your domain name to each certificate resource in the `dnsNames` section. For example:

    ```
      dnsNames:
      - 169.123.123.123.nip.io
      - example-mongo-0.example-mongo-svc.mongodb.svc.cluster.local
    ---
      dnsNames:
      - 169.123.123.123.nip.io
      - example-mongo-1.example-mongo-svc.mongodb.svc.cluster.local
    ---
      dnsNames:
      - 169.123.123.123.nip.io
      - example-mongo-2.example-mongo-svc.mongodb.svc.cluster.local
    ```

1. Then apply the new configuration:

    ```
    oc apply -f cert-manager/certificates.yaml
    ```

1. You also need to update the PEM files used by the MongoDB so that it will use the updated certificates.

    ```
    oc get secret example-mongo-0 -o jsonpath='{.data.tls\.crt}{.data.tls\.key}' | base64 --decode > example-mongo-0-pem

    oc get secret example-mongo-1 -o jsonpath='{.data.tls\.crt}{.data.tls\.key}' | base64 --decode > example-mongo-1-pem

    oc get secret example-mongo-2 -o jsonpath='{.data.tls\.crt}{.data.tls\.key}' | base64 --decode > example-mongo-2-pem
    ```

1. Delete the old secret and create a new one using the new PEM files

    ```
    oc delete example-mongo-cert

    oc create secret generic example-mongo-cert --from-file=example-mongo-0-pem --from-file=example-mongo-1-pem --from-file=example-mongo-2-pem
    ```

## 5. Configure the MongoDB Database resource

Before connecting to the database externally, you will need to add your domain name with the node port in the replica set horizons in your MongoDB database resource. You can do this by adding a spec `spec.connectivity.replicaSetHorizons` in your `deployment/2-mongodb-deployment.yaml` file from your cloned repo.

Get the node port using:

```
oc get services

NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)           AGE
example-mongo-0        NodePort       172.21.174.50    <none>         27017:31385/TCP   72m
example-mongo-1        NodePort       172.21.218.87    <none>         27017:30131/TCP   72m
example-mongo-2        NodePort       172.21.170.154   <none>         27017:30656/TCP   72m
```

The replica set horizon values would be an array in the format of `<replica-set>: <domain-name>:<port>`

For example:

```
spec:
  connectivity:
    replicaSetHorizons:
    - example-mongo: 169.123.123.123.nip.io:31385
    - example-mongo: 169.123.123.123.nip.io:30131
    - example-mongo: 169.123.123.123.nip.io:30656
  # the name of the secret in 1-ops-manager-config-secret.yaml
  credentials: ops-manager-organization-apikey
  members: 3
  ...
```

Then apply the new configuration:

```
oc apply -f deployment/2-mongodb-deployment.yaml
```

> Make sure TLS and authentication are still enabled!

## 6. Test connection from client outside of OpenShift or Kubernetes

Now that you have the connection enabled, you can test it by connecting to it from your local machine. You can either use the `mongo` CLI or use the example application from the previous tutorial.

1. If you have the `mongo` CLI installed, you can execute:

    ```
    mongo \
      --host example-mongo/169.63.221.247.nip.io:31385,169.63.221.247.nip.io:30131,169.63.221.247.nip.io:30656 \
      --tls \
      --tlsCAFile ca.crt \
      -u mongodb-example-user \
      -p password \
      --authenticationDatabase example

    MongoDB shell version v4.4.1
    connecting to: mongodb://169.xx.nip.io:31385,169.xx.nip.io:30131,169.xx.nip.io:30656/?authSource=example&compressors=disabled&gssapiServiceName=mongodb&      replicaSet=example-mongo
    Implicit session: session { "id" : UUID("15d79cf9-1017-4230-a1f3-5f5e312309ad") }
    MongoDB server version: 4.2.2
    WARNING: shell and server versions do not match
    MongoDB Enterprise example-mongo:PRIMARY>
    ```  

1. Or you can also use the example Node.js application from the previous tutorial. To do so, export the following environment variables with your public domain names and respective node ports.

    ```
    cd example-applications/nodejs

    export MONGODB_REPLICA_HOSTNAMES=169.123.123.123.nip.io:31385,169.123.123.123.nip.io:30131,169.123.123.123.nip.io:30656
    export MONGODB_REPLICA_SET=example-mongo
    export MONGODB_USER=mongodb-example-user
    export MONGODB_PASSWORD=password
    export MONGODB_AUTH_DBNAME=example
    export MONGODB_CA_PATH=$(pwd)/../../ca.crt
    export MONGODB_DBNAME=example
    ```

1. Run the example application:

    ```
    npm install
    node app.js
    ```

1. Open `localhost:8080/api-docs` through your browser and try the APIs:

    ![example output](docs/images/example-output.png)

# Conclusion

Hopefully this tutorial has shown you one of the ways you can expose a MongoDB Database so that external applications or resources can directly connect to it. In addition to that, the MongoDB database is secured with authentication and encryption which ensures your data is safe.

## Learn More

* [Learn more about using Mongoose with Node.js](https://mongoosejs.com/docs/)
* [Learn about Data Modeling with MongoDB](https://docs.mongodb.com/manual/core/data-modeling-introduction/)
* [Official Docs on using MongoDB Enterprise Operator](https://docs.mongodb.com/kubernetes-operator/stable/)
* [Learn more about cert-manager](https://cert-manager.io/docs/)
* [Red Hat Marketplace](https://marketplace.redhat.com/)

## License

This tutorial is licensed under the Apache License, Version 2. Separate third-party code objects invoked within this tutorial are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1](https://developercertificate.org/) and the [Apache License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache License FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
