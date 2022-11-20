# dotcms-demo
dotCMS k8s demo

## Notes

- downloaded dotCMS and ran in docker to familiarize self with app

#### Initially I thought to use Amazon EKS so gave it a whirl:

- install aws cli https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
  - downloaded pkg and installed for all users
- install kubectl https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
  - we have client Version: v1.22.5 skipping
- install eksctl https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html
  - requires Weaveworks Homebrew tap: brew tap weaveworks/tap
  - install eksctl: brew install weaveworks/tap/eksctl
  - unable to install eksctl

#### I then decided it was better to just use minikube locally

- decision to install minikube
- used kompose to create initial k8s yaml files based on docker-compose.yaml provided by dotCMS.com
- deployment not suitable for database, switched to statefulset for postgres
- kompose failed to create service definition for postgres; created
- db_net needed a better name (under scores not allowed)
- PROBLEM: dotcms can't talk to database, can't resolve 'db'
  - tested for potential DNS issue; resolved by hard coding the IP
  - discovered issue with DB service manifest resulting in DNS lookup failure
- PROBLEM: cannot access site using HTTPS; 403 thrown by minikube api server is answering the call:

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/dotAdmin/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

- SSL: likely we require some ingress rule to redirect the traffic hitting the API server to go to the app instead.
- OBSERVATION: dotcms pod should wait for db pod to spin up before trying to connect

# INSTALL:

```helm install dotcms-demo helm```

or

```
cd helm/templates
files=`ls | grep yaml`; for f in $files; do kubectl apply -f $f; done
```

