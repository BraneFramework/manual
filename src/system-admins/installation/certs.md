# Certificate management
This chapter describes the certificate management for Brane central-, worker- and proxy-nodes.

First, ensure that you generated the basic certification as described in the respective installation pages ([central nodes](./control-node.md), [worker nodes](./worker-node.md) or [proxy nodes](./proxy-node.md)).


## Overview
Currently, Brane uses certificates only when interaction with the `brane-reg`-service occurs. This service acts as an endpoint for accessing domain-local, potentially sensitive data, and hence needs the additional security of TLS and identifying clients that submit requests.

To achieve this, two kinds of certificates are used:
- _Server certificates_ are hosted by the `brane-reg`-service and are used for the basic TLS encryption layer, as well as providing the guarantee to clients that they are talking to the `brane-reg` they think they are (as per usual); and
- _Client certificates_ are used when downloading datasets to ensure the identity of the receive party for policy purposes.

Since `brane-reg` _always_ requires TLS to be setup, the first type of certificate is involved in every request send to `brane-reg`. However, the second type is only involved when submitting a data transfer request.


## Server certificates
Because every worker node has a `brane-reg` service, every worker node subsequently also needs a server certificate to establish the incoming TLS connections.

During node creation, you can generate the server certificates for a worker domain using the following command:
```sh
branectl generate certs server <LOCATION_ID> -H <HOSTNAME>
```
where `<LOCATION_ID>` is the identifier for that worker node, and `<HOSTNAME>` is the hostname with which the other domains need to reach this node.

> <img src="../../assets/img/warning.png" alt="warning" width="16" style="margin-top: 3px; margin-bottom: -3px"/> If you're using external procies that routes incoming traffic for `brane-reg`, then be sure to use the hostname of the **proxy node** instead of the worker node. See [below](#external-proxies) for more details.

This command generates two certificates:
1. The `ca.pem` (public) and `ca-key.pem` (private) certificate and -key are the _root_ or _certificate authority_ certificates of the node. This is used to sign all the other certificates coming from this domain.
2. The `server.pem` (public) and `server-key.pem` (private) certificate and -key is the one used to setup the TLS connection, as discussed [above](#overview).

The **public** node certificate (`ca.pem`) should be shared with any other node that intends to connect to `brane-reg` in some form or another, as they will need it to verify that the connection is genuine and to encrypt is. Typically, this is distributed through public certificate authorities, but that is not yet implemented in Brane. For now, it sufficies to put it in the `config/certs/`-directory of the client node (see [below](#storing-certificates) for more information).

> <img src="../../assets/img/warning.png" alt="warning" width="16" style="margin-top: 3px; margin-bottom: -3px"/> In this case, the central node also connects to `brane-reg`, and will need to have the server certificates of all relevant worker nodes!

_To recap:_
- Every **worker node** needs to generate a server certificate & key; and
- Every _other_ **worker node** that connects to it requires to have the public certificate to verify the initial worker's identity and to establish the TLS tunnel.
    - Here, the **central node** also needs to access `brane-reg` of **every worker node in the Brane instance**. Hence, it will need public certificates of all workers.
    - Along the same vein, every **end user** who whishes to download data of the worker also needs to have the public certificates.


## Client certificates
Then, to verify the identity of clients that want to download data of a worker node, client certificates are used to prove that identity.

This works by the worker node generating certificates for every client who (may) wish(es) to download data. This can be done using the following command:
```sh
branectl generate certs client <LOCATION_ID> -H <HOSTNAME>
```
where `<LOCATION_ID>` is the identifier of the **client** (_not_ the current worker!), and `<HOSTNAME>` is the **client**'s hostname.

> <img src="../../assets/img/info.png" alt="info" width="16" style="margin-top: 3px; margin-bottom: -3px"/> Note, however, that currently, the hostname part of client certificates is ignored by the framework - it is here for future compatability.

This command uses the worker's private `ca-key.pem` to generate a new, **private** `client-id.pem` file that is the client certificate that the client can use to identify themselves. Note that this **must be kept secret at all times**; think of it as a password which, once leaked, immediately allows any one with access to it to pretend they are the client for whom it was generated.

_To recap:_
- For every (potential) client, a **worker node** generates a client certificate;
- That client requires access to that certificate to establish a connection for downloading data, which proves their identity to the worker. It is crucial the certificate remains **private**.
    - Note that the _central node_ **never** downloads data, and as such, does not need client certificates.
    - End users, on the other hand, **may** download data. End users authorised to do so should therefore also receive a newly generated client certificate.
- Client certificates are _only_ for identification; policy decides whether access is granted or not.


## Storing certificates
### Nodes
Storing certificates is done in the `certs`-directory of every node (defined in `node.yml`, `node/paths/certs`).

This directory has the following structure:
- (If the node is a worker), it should contain the generated `ca.pem`, `ca-key.pem`, `server.pem` and `server-key.pem`;
- For all nodes, it should contain a folder named after the identifier of every node to who's `brane-reg` it will establish and **outgoing connection** to, which contains:
    - A `ca.pem` to verify that other node's identity; and
    - (Only if transferring data) A `client-id.pem` used to identify this node to the other node.

For example, in a system with three workers (`amy`, `bob` and `dan`), where only `amy` and `bob` will exchange data:
- The central node has the following `certs/`-directory:
  ```
  certs/
    amy/
      ca.pem
    bob/
      ca.pem
    dan/
      ca.pem
  ```
- `amy` has the following `certs/`-directory:
  ```
  certs/
    ca.pem
    ca-key.pem
    server.pem
    server-key.pem
    bob/
      ca.pem
      client-id.pem
  ```
- `bob` has the following `certs/`-directory:
  ```
  certs/
    ca.pem
    ca-key.pem
    server.pem
    server-key.pem
    amy/
      ca.pem
      client-id.pem
  ```
- `dan` has the following `certs/`-directory:
  ```
  certs/
    ca.pem
    ca-key.pem
    server.pem
    server-key.pem
  ```

Note that access is assymmetrical; i.e., it could be that `amy` wants to download from `bob` but not the other way around. In that case, only `amy` needs a `client-id.pem` file received from `bob` but not the other way around.

### Users
End users can add certificates to their `brane` client store by using the following command:
```sh
brane certs add ca.pem client-id.pem -i <INSTANCE>
```
where `<INSTANCE>` is the local _brane instance_ to add the certificates to (see [this chapter](../../scientists/instances.md) for more details). `ca.pem` and `client-id.pem` are files received from the worker that the end user wants to download files from.

By default, the command will try to read the location ID of the worker in question from the certificates provided. If this somehow fails, use the `-d <LOCATION_ID>`-option to provide it manually.


## External proxies
Since 1.0.0, Brane has the `brane-prx` service to route outgoing and incoming traffic from- and to a node, respectively. Ever since, it is the `brane-prx` service that is responsible for managing certificates of **outgoing connections**.

Normally, this is not really noticable. However, in the case of an [external proxy](./proxy-node.md), this means the certificates have to be distributed over the node and its external proxy. This works as follows:
- The node itself (e.g., worker) needs to have all **incoming** certificates. These are the generated `server.pem`, `server-key.pem`, `ca.pem` and `ca-key.pem`-certificates.
    - Note that a central node does not have these certs, and as such, needs none.
- The proxy node needs all **outgoing** certificates. This is the `ca.pem` and optional `client-id.pem` for every worker node that will be connected to, structured as above.

If you're also routing _incoming_ traffic to the `brane-reg` service, then **the certificates do NOT have to move**. The `brane-prx` will forward the TLS negotation to the `brane-reg` service. However, as a consequence, note that the **hostname provided when generating the server certificates should be that of the proxy node**, as that is the perceived termination point of the TLS tunnel of the client. To understand this better, consider the following piece of ASCII art:
```
        Secured w/TLS
        ~~~~~~~~~~~~~
client  ------------>  proxy  ------------>  worker
^                      ^                     ^
amy.nl                 bob.nl                internal.bob.nl

*ca.pem                                      *ca.pem *ca-key.pem
*client-id.pem                               *server.pem *server-key.pem
```

Even though the worker has the certs, the client thinks it communicates with `bob.nl`. As such, the certificates need to be created for `bob.nl` instead of `internal.bob.nl`.

To understand advanced networking in Brane better, see the [networking chapter](TODO).
