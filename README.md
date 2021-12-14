# Simplified Minikube deployment 

This document is targeting developers, testers, and all the other interested parties who would like to learn or test prow, but do not have resources other than a local workstation. It describes a simplified minikube deployment of crucial prow components:

* [`crier`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/crier)
* [`deck`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/deck)
* [`hook`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/hook)
* [`horologium`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/horologium) 
* [`prow-controller-manager`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/prow-controller-manager) 
* [`sinker`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/sinker) 
* [`tide`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/tide)
* [`status-reconciler`](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/status-reconciler)

More information about each component function can be found [here](https://github.com/kubernetes/test-infra/tree/master/prow/cmd/). Other components can be easily added as well, depending on user needs. While any serious prow installation uses [`ghproxy`](https://github.com/kubernetes/test-infra/tree/master/ghproxy) to handle GitHub client request throttling, this tutorial aims at simplicity and does not use it. Adding to that, a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) is used as a basic authentication mechanism, due to its simplicity and possibility of quick deployment.

## Prerequisites

* Create a new GitHub project or choose an existing one that will be connected to your localhost prow.
* Please follow [**GitHub App**](https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md) section from **Deploying Prow** tutorial. During configuration please be aware, that fields like ``Homepage URL`` or ``Callback URL`` don't matter that much in the test deployment. You can skip them, and if not possible, an invalid URL also should work. The most important field for this tutorial ``Webhook URL`` should be filled using instruction below.
* Install configured application into your project (``Settings -> Developer settings -> GitHub Apps -> YourAppName -> Edit -> Install App``)
* Generate a new Personal access token (``Settings -> Developer settings -> GitHub Apps -> Personal access token -> Generate new token``). Save it for now, as you won't be able to see it again.
* Install docker and minikube. 

## Filling Webhook URL

The deployment will work on your localhost machine, but it needs to connect to the GitHub application configured in the previous section. There are several methods to do this with a limited access to the public IP. The simplest would be to use dedicated services like [Hookdeck](https://hookdeck.com/) or [smee.io](https://smee.io/). They have usually a limited number of actions available per hook, and the webhooks have living period limited. Because of these reasons they should not be used for the production environment. Please visit [smee.io](https://smee.io/) to generate a new channel. Follow the instruction to install the tool (using ``npm``) after clicking the ``Start a new channel`` button. Paste generated link in the ``Webhook URL`` config field in your application configuration.

The ``Webhook URL`` should be configured with a secret. To generate a secret please run in the console:
```
$ openssl rand -hex 20
```
Save the result locally, and paste it in the field ``Webhook secret (optional)`` during GitHub App configuration.

## Deployment on Minikube

Start Minikube using:
```
$ minikube start
```
If you have an option to choose, prefer deployment on ``docker``. ``KVM`` deployment also has been tested.
Get Minikube's IP and use it to connect using Webhook (``smee`` as an example):
```
$ smee -u https://smee.io/wz9VwP1DSreFbXP --target $(minikube ip):32106/hook
```
You also need to apply the prow CRDs using:
```
$ kubectl apply --server-side=true -f https://raw.githubusercontent.com/kubernetes/test-infra/master/config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml
```

### Update the sample manifest
The sample manifest in this repository is based on [starter-s3.yaml](https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/starter/starter-s3.yaml). It has a reduced number of components, but keeps [``minio``](https://min.io/) as storage backend.
There are several fields that has to be configured before applying the manifest. They are all marked using ``TODO`` tag inside the file. Please adjust:

* The github app cert by replacing the `<<insert-downloaded-cert-here>>` string
* The github app id by replacing the `<<insert-the-app-id-here>>` string
* The hmac token by replacing the `<< insert-hmac-token-here >>` string
* The personal access token by replacing the `<< insert-personal-access-token-here >>` string
* Your github repository by searching all the rest ``TODO`` strings and replacing them with your ``user/repo``

Please also note that domain addresses are set to ``localhost.localdomian``. You may want to change it, but it's not necessary since it's a test deployment.
``Minio`` credentials are also fixed to ``user: minio`` and ``password: minio123``, but can be changed.

After filling all the fields, you need to apply the manifest. It can be done using:
```
$ minikube kubectl -- apply -f starter-s3-minikube.yaml 
```
A few minutes later, all the components should be running:
```
$ minikube kubectl -- get pods -n prow
NAME                                       READY   STATUS         RESTARTS   AGE
crier-799894b7bd-7wndj                     1/1     Running        0          3m35s
deck-5d87d79f5d-b2vxr                      0/1     Running        0          3m35s
hook-d696d8c9b-lzsrr                       1/1     Running        0          3m35s
horologium-64dccd58f5-l22jt                1/1     Running        0          3m36s
minio-85f4b8f4bf-9tfbx                     1/1     Running        0          3m35s
prow-controller-manager-6646597778-dkn7b   1/1     Running        0          3m35s
sinker-68cd54cd4b-s96gh                    1/1     Running        0          3m36s
statusreconciler-64cd58c84f-v5hkd          1/1     Running        0          3m35s
tide-85f8bdd9f7-w5jq6                      1/1     Running        1          3m35s
```
### Test your deployment
The simplest way to test the deployment to see if the webhook is passing data and the components are interacting with the GitHub is to open a PR in your repository. At the bottom jobs should be visible if the CI is properly configured. You can also post some simple comments (like ``/meow`` or ``/bark``) to see if messages get the hook component.
There could be several problems, the list below will give some hints how to debug them:
* Check if your GitHub App is installed in your repository
* Check if webhook is still active. If it expired, paste the new one in the ``Webhook URL`` field and use it locally re-running ``smee``
* Check correctness of your ``minikube ip``, it can change after cluster recreation

### Ports
You can access ``hook`` using port ``30001``, ``deck`` is listening on port ``30002`` and ``minio`` on port ``30003``. Use ``minikube ip`` to get the IP address.

### Cluster recreation
After filling up the manifest file with your own data, you can easily recreate the whole cluster using the following sequence of commands:
```
$ minikube delete
$ minikube start
$ smee -u https://smee.io/wz9VwP1DSreFbXP --target $(minikube ip):32106/hook
$ kubectl apply --server-side=true -f https://raw.githubusercontent.com/kubernetes/test-infra/master/config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml
$ minikube kubectl -- apply -f starter-s3-minikube.yaml 
```
It should take no more than 5-7 minutes.

## Where to go next?
This section will be filled with information on how this test deployment could help you in developing and testing prow.

### Deploying and testing your own images
With this test installation, there is a possibility of deploying your own images in the puppy environment, before submitting a PR with a fix. It can be also useful for debugging. The simplest way would be to modify the existing image by adding your own binary. Let's assume you want to modify tide. You can create a ``Dockerfile`` that will create a modified image:
```
#change the image version to the newer one
FROM gcr.io/k8s-prow/tide:v20211119-1147d04f59
COPY /path/to/your/tide /app/prow/cmd/tide/app.binary.runfiles/io_k8s_test_infra/prow/cmd/tide/app.binary_/app.binary
```
Please remember that the component should be compiled with static linking:
```
$ CGO_ENABLED=0 go build -ldflags="-extldflags=-static"
```
Image can be loaded to Minikube using different methods. The simplest would be to tag the image and load it using (please add ``imagePullPolicy: IfNotPresent`` to the deployment you want to modify):
```
$ minikube image load my_image
```
Other methods can be explored using [Pushing images](https://minikube.sigs.k8s.io/docs/handbook/pushing/) documentation site.
