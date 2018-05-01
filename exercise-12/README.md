## Exercise 12 - Security

### Overview of Istio Mutual TLS

Istio provides transparent, and frankly magical, mutual TLS to services inside the service mesh when asked. By mutual TLS we understand that both the client and the server authenticate each others certificates as part of the TLS handshake.

### Enable Mutual TLS

Let the past go. Kill it, if you have to:
```
cd ~/istio
kubectl delete -f install/kubernetes/istio.yaml
kubectl delete all --all
```

It's the only way for TLS to be the way it was meant to be:

```
kubectl create -f install/kubernetes/istio-auth.yaml
```

We need to (re)create the auto injector. There is a script bundled that will do this but you will need to switch back to _this_ directory and give it the location of your istio install. Or you can redo the steps from exercise 6. Your call.

```
cd ~/istio-workshop/exercise-12
./install-auto-injector.sh ~/istio
```

Finally enable injection and deploy the thrilling Book Info sample.

```
cd ~/istio
kubectl label namespace default istio-injection=enabled
kubectl create -f samples/bookinfo/kube/bookinfo.yaml
```

## Testing mutual TLS security

At this point it might seem like nothing changed, but it has.
Let's disable the webhook in default for a second.

```
kubectl label namespace default istio-injection-
```

Validate that the `default` namespace has the istio-injection disabled.

```
kubectl get ns -L istio-injection
```

Now lets deploy a simple pod to validate that mutual TLS is working.

```
kubectl run toolbox -l app=toolbox  --image centos:7 /bin/sh -- -c 'sleep 84600'
```

First: let's prove to ourselves that we really are doing something with tls. From here on out assume names like foo-XXXX need to be replaced with the foo podname you have in your cluster. We pass `-k` to `curl` to convince it to be a bit laxer about cert checking.

```
tb=$(kubectl get po -l app=toolbox -o template --template '{{(index .items 0).metadata.name}}')
kubectl exec -it $tb curl -- https://details:9080/details/0 -k
```

Denied! You will not gain access because a certificate was not found.

Let's exfiltrate the certificates out of a proxy so we can pretend to be them (incidentally I hope this serves as a cautionary tale about the importance locking down pods).

```
pp=$(kubectl get po -l app=productpage -o template --template '{{(index .items 0).metadata.name}}')
mkdir ~/tmp # or wherever you want to stash these certs
cd ~/tmp
fs=(key.pem cert-chain.pem root-cert.pem)
for f in ${fs[@]}; do kubectl exec -c istio-proxy $pp /bin/cat -- /etc/certs/$f >$f; done
```

This should give you the certs. Now let us copy them into our toolbox.

```
for f in ${fs[@]}; do kubectl cp $f default/$tb:$f; done
```

Try once more to talk to the details service, but this time with feeling:

```
kubectl exec -it $tb curl -- https://details:9080/details/0 -v --key ./key.pem --cert ./cert-chain.pem --cacert ./root-cert.pem -k
```

Success! We really are protecting our connections with tls. Time to enjoy its magic from the inside. Let's enable the webhook and see how the system works normally.

Re-enable istio-injection and delete the `toolbox` pod.
```
kubectl label namespace default istio-injection=enabled
kubectl delete po $tb
```

Attempt to access the details service from within the toolbox after the istio sidecar has been injected.

```
tb=$(kubectl get po -l app=toolbox -o template --template '{{(index .items 0).metadata.name}}')
kubectl exec -it $tb curl -- http://details:9080/details/0
```

**_Notice the protocol._**

#### [Continue to Exercise 13 - Istio RBAC](../exercise-13/README.md)
