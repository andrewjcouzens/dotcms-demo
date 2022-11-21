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
- SOLUTION: got out the duct tape, women will at least find you handy if they don't find you handsome
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

- SSL: likely we require some ingress configured to redirect the traffic hitting the API server to go to the app instead. (https://kubernetes.github.io/ingress-nginx/examples/rewrite/ or perhaps https://kubernetes.github.io/ingress-nginx/examples/tls-termination/)
- OBSERVATION: dotcms pod should wait for db pod to spin up before trying to connect

#### Install into GCP

 - Created GKE cluster
`gcloud container clusters create --zone us-central1-b --machine-type=n1-standard-4 --num-nodes=3 dotcms-demo-cluster`

 - PROBLEM: CrashLoopBackOff on db, investigate by viewing logs `kubectl logs db-0`

```
12:17 $ kubectl logs db-0
W1120 12:17:59.587402   39093 gcp.go:119] WARNING: the gcp auth plugin is deprecated in v1.22+, unavailable in v1.26+; use gcloud instead.
To learn more, consult https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
initdb: hint: Using a mount point directly as the data directory is not recommended.
Create a subdirectory under the mount point.
The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.
```

- SOLUTION: Modified StatefulSet to use PGDATA env variable and appended additional directory "dotcms" which resolved issue.  This solution is more likely to succeed everywhere with less exposure compared to just deleting the offending lost+found directory with an initContainer
- PROBLEM: Restarted cluster dotcms pod throwing error:

```
12:39 $ kubectl logs dotcms-77bf444c5-pkrb6
W1120 12:40:20.121715   42870 gcp.go:119] WARNING: the gcp auth plugin is deprecated in v1.22+, unavailable in v1.26+; use gcloud instead.
To learn more, consult https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
touch: cannot touch '/data/shared/custom_starter.zip': Permission denied
```

- SOLUTION: Examined the dotcms pod and found dotCMS crashing due to permission issues.  Checked /etc/group and found the uid/gid of the dotcms user to be 65001.  Modified the dotcms deployment to use a securityContext definition to set the uid/gid to 65001 and then leveraged an initContainers directive to then chown the cms shared volume to be uid/gid 65001 
- PROBLEM: dotcms can't contact elastic search

```
23:06:20.297  ERROR util.DotRestHighLevelClientProvider$1 - [host=https://opensearch:9200]
23:06:20.304  ERROR business.ESIndexAPI - Elasticsearch Attempt #1 : Connection refused
```

- SOLUTION: OpenSearch was crashing due to permission issues.  Examined the opensearch pod and found the uid/gid of the opensearch user to be 1000.  Modified the opensearch deployment to use a securityContext definition identical to dotcms.

- Setup a port-forward using kubectl and was then able to use HTTPS to access dotCMS in GKE.  A production solution would be to setup GKE Ingress either terminating the SSL at the load balancer and forwarding on the traffic encrypted or passing the SSL connection through.

```
21-Nov-2022 00:04:38.770 INFO [url:POST//default/api/v1/authentication | lang:1 | ip:127.0.0.1 | Admin:false | start:11-20-2022 11:42:51 UTC  ref:https://localhost:8443/dotAdmin/] com.dotcms.repackage.org.hibernate.validator.internal.util.Version.<clinit> HV000001: Hibernate Validator 4.3.2.Final
00:04:39.182  INFO  util.SecurityLogger - class com.dotmarketing.cms.login.factories.LoginFactory : User password was rehash with id: dotcms.org.1 -- ip:127.0.0.1,user:null
00:04:39.475  WARN  auth.PrincipalThreadLocal - getName null
00:04:39.528  INFO  util.SecurityLogger - class com.dotcms.cms.login.LoginServiceAPIFactory$LoginServiceImpl : User dotcms.org.1 has successfully login from IP: 127.0.0.1 -- ip:127.0.0.1,user:Admin User [ID: dotcms.org.1][email:admin@dotcms.com]
```


# INSTALL:

```helm install dotcms-demo helm```

or

```
cd helm/templates
files=`ls | grep yaml`; for f in $files; do kubectl apply -f $f; done
```

