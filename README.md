# AWS S3 operator
This document describes how to deploy or run AWS S3 k8s operator on k8s Cluster.
## Getting Started
### Prerequisites
This project requires a k8s environment and Minikube needs to be installed before we start.<br>
Install Minikube
```
 $ brew install minikube
 $ minikube start --kubernetes-version v1.17.0
 
```
### Installing
1. Download objectbucket_v1alpha1_objectbucket_crd.yaml and objectbucket_v1alpha1_objectbucketclaim_crd.yaml from https://github.com/kube-object-storage/lib-bucket-provisioner/tree/master/deploy/crds, then create the ObjectBucket and ObjectBucketClaim
```
 $ kubectl create -f objectbucket_v1alpha1_objectbucket_crd.yaml
 $ kubectl create -f objectbucket_v1alpha1_objectbucketclaim_crd.yaml
 
```

2. Deploy the latest AWS S3 Provisioner and create a ClusterRoleBinding.
AWS s3 Provisioner link https://github.com/yard-turkey/aws-s3-provisioner/blob/master/examples/awss3provisioner-deployment.yaml<br>
ClusterRoleBinding
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-s3-provisioner-deployment
  namespace: s3-provisioner
  labels:
    app: aws-s3-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aws-s3-provisioner
  template:
    metadata:
      labels:
        app: aws-s3-provisioner
    spec:
      containers:
      - name: aws-s3-provisioner
        image: quay.io/screeley44/aws-s3-provisioner:v1.0.0
        imagePullPolicy: Always
      restartPolicy: Always
```
3. Create a ClusterRoleBinding for the default serviceaccount that will run your provisioner.
``` 
  kubectl create clusterrolebinding <rolename> --clusterrole=cluster-admin --user=system:serviceaccount:<namespace>:default
  i.e.
  # kubectl create clusterrolebinding cluster-admin-aws --clusterrole=cluster-admin --user=system:serviceaccount:s3-  provisioner:default
```

### Administrator Creates Secret
This secret will contain the elevated/admin privileges needed by the provisioner to properly access and create S3 Buckets and IAM users and policies. The AWS Access ID and AWS Secret Key will be needed for this.<br>
1. Create the Kubernetes Secret for the Provisioner's Owner Access.
``` 
apiVersion: v1
kind: Secret
metadata:
  name: s3-bucket-owner [1]
  namespace: s3-provisioner [2]
type: Opaque
data:
  AWS_ACCESS_KEY_ID: *base64 encoded value* [3]
  AWS_SECRET_ACCESS_KEY: *base64 encoded value* [4]
``` 
	1	Name of the secret, this will be referenced in StorageClass.
	2	Namespace where the Secret will exist.
	3	Your AWS_ACCESS_KEY_ID base64 encoded. Encode your AWS_ACCESS_KEY_ID of your AWS account by https://www.base64encode.org/ 
	4	Your AWS_SECRET_ACCESS_KEY base64 encoded. Encode your AWS_SECRET_ACCESS_KEY of your AWS account by https://www.base64encode.org/ 
Then use the following commond
``` 
 # kubectl create -f creds.yaml
 secret/s3-bucket-owner created
``` 

Administrator Creates StorageClass
The StorageClass defines the name of the provisioner and holds other properties that are needed to provision a new bucket, including the Owner Secret and Namespace, and the AWS Region.
	1	Create the Kubernetes StorageClass for the Provisioner.
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: s3-buckets [1]
provisioner: aws-s3.io/bucket [2]
parameters:
  region: us-west-1 [3]
  secretName: s3-bucket-owner [4]
  secretNamespace: s3-provisioner [5]
reclaimPolicy: Delete [6]
	1	Name of the StorageClass, this will be referenced in the User ObjectBucketClaim.
	2	Provisioner name
	3	AWS Region that the StorageClass will serve
	4	Name of the bucket owner Secret created above
	5	Namespace where the Secret will exist
	6	reclaimPolicy (Delete or Retain) indicates if the bucket can be deleted when the OBC is deleted.
NOTE: the absence of the bucketName Parameter key in the storage class indicates this is a new bucket and its name is based on the bucket name fields in the OBC.
 # kubectl create -f storageclass-greenfield.yaml
storageclass.storage.k8s.io/s3-buckets created

User Creates ObjectBucketClaim
An ObjectBucketClaim follows the same concept as a PVC, in that it is a request for Object Storage, the user doesn't need to concern him/herself with the underlying storage, just that they need access to it. The user will work with the cluster/storage administrator to get the proper StorageClass needed and will then request access via the OBC.
	1	Create the ObjectBucketClaim.
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: myobc [1]
  namespace: s3-provisioner [2]
spec:
  generateBucketName: mybucket-prefix- [3]
  bucketName: my-awesome-bucket [4]
  storageClassName: s3-buckets [5]
	1	Name of the OBC
	2	Namespace of the OBC
	3	Name prepended to a random string used to generate a bucket name. It is ignored if bucketName is defined
	4	Name of new bucket which must be unique across all AWS regions, otherwise an error occurs when creating the bucket. If present, this name overrides generateName
	5	StorageClass name
NOTE: if both generateBucketName and bucketName are omitted, and the storage class does not define a bucket name, then a new, random bucket name is generated with no prefix.
 # kubectl create -f obc-brownfield.yaml
objectbucketclaim.objectbucket.io/myobc created

Results and Recap
Let's pause for a moment and digest what just happened. After creating the OBC, and assuming the S3 provisioner is running, we now have the following Kubernetes resources: . a global ObjectBucket (OB) which contains: bucket endpoint info (including region and bucket name), a reference to the OBC, and a reference to the storage class. Unique to S3, the OB also contains the bucket Amazon Resource Name (ARN).Note: there is always a 1:1 relationship between an OBC and an OB. . a ConfigMap in the same namespace as the OBC, which contains the same endpoint data found in the OB. . a Secret in the same namespace as the OBC, which contains the AWS key-pairs needed to access the bucket.
And of course, we have a new AWS S3 Bucket which you should be able to see via the AWS Console.
ObjectBucket
 # kubectl get ob obc-s3-provisioner-my-awesome-bucket -o yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucket
metadata:
  creationTimestamp: "2019-04-03T15:42:22Z"
  generation: 1
  name: obc-s3-provisioner-my-awesome-bucket
  resourceVersion: "15057"
  selfLink: /apis/objectbucket.io/v1alpha1/objectbuckets/obc-s3-provisioner-my-awesome-bucket
  uid: 0bfe8e84-576d-4c4e-984b-f73c4460f736
spec:
  Connection:
    additionalState:
      ARN: arn:aws:iam::<accountid>:policy/my-awesome-bucket-vSgD5 [1]
      UserName: my-awesome-bucket-vSgD5 [2]
    endpoint:
      additionalConfig: null
      bucketHost: s3-us-west-1.amazonaws.com
      bucketName: my-awesome-bucket [3]
      bucketPort: 443
      region: us-west-1
      ssl: true
      subRegion: ""
  claimRef: null [4]
  reclaimPolicy: null
  storageClassName: s3-buckets [5]
	1	The AWS Policy created for this user and bucket.
	2	The new user generated by the Provisioner to access this existing bucket.
	3	The bucket name.
	4	The reference to the OBC (not filled in yet).
	5	The reference to the StorageClass used.
ConfigMap
 # kubectl get cm myobc -n s3-provisioner -o yaml
apiVersion: v1
data:
  BUCKET_HOST: s3-us-west-1.amazonaws.com [1]
  BUCKET_NAME: my-awesome-bucket [2]
  BUCKET_PORT: "443"
  BUCKET_REGION: us-west-1
  BUCKET_SSL: "true"
  BUCKET_SUBREGION: ""
kind: ConfigMap
metadata:
  creationTimestamp: "2019-04-01T19:11:38Z"
  finalizers:
  - objectbucket.io/finalizer
  name: my-awesome-bucket
  namespace: s3-provisioner
  resourceVersion: "892"
  selfLink: /api/v1/namespaces/s3-provisioner/configmaps/my-awesome-bucket
  uid: 2edcc58a-aff8-4a29-814a-ffbb6439a9cd
	1	The AWS S3 host.
	2	The name of the new bucket we are gaining access to.
Secret
 # kubectl get secret my-awesome-bucket -n s3-provisioner -o yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: *the_new_access_id* [1]
  AWS_SECRET_ACCESS_KEY: *the_new_access_key_value* [2]
kind: Secret
metadata:
  creationTimestamp: "2019-04-03T15:42:22Z"
  finalizers:
  - objectbucket.io/finalizer
  name: my-awesome-bucket
  namespace: s3-provisioner
  resourceVersion: "15058"
  selfLink: /api/v1/namespaces/s3-provisioner/secrets/screeley-provb-5
  uid: 225c71a5-9d75-4ccc-b41f-bfe91b272a13
type: Opaque
	1	The new generated AWS Access Key ID.
	2	The new generated AWS Secret Access Key.
What happened in AWS? The first thing we do on any OBC request is create a new IAM user and generate Access ID and Secret Keys. This allows us to also better control ACLs. We also create a policy in IAM which we then attach to the user and bucket. We also created a new bucket, called my-awesome-bucket.
When the OBC is deleted all of its Kubernetes and AWS resources will also be deleted, which includes: the generated OB, Secret, ConfigMap, IAM user, and policy. If the retainPolicy on the StorageClass for this bucket is "Delete", then, in addition to the above cleanup, the physical bucket is also deleted.
NOTE: The actual bucket is only deleted if the Storage Class's reclaimPolicy is "Delete".

User Creates Pod
Now that we have our bucket and connection/access information, a pod can be used to access the bucket. This can be done in several different ways, but the key here is that the provisioner has provided the proper endpoints and keys to access the bucket. The user then simply references the keys.
	1	Create a Sample Pod to Access the Bucket.
apiVersion: v1
kind: Pod
metadata:
  name: photo1
  labels:
    name: photo1
spec:
  containers:
  - name: photo1
    image: docker.io/screeley44/photo-gallery:latest
    imagePullPolicy: Always
    envFrom:
    - configMapRef:
        name: my-awesome-bucket <1>
    - secretRef:
        name: my-awesome-bucket <2>
    ports:
    - containerPort: 3000
      protocol: TCP
	1	Name of the generated configmap from the provisioning process
	2	Name of the generated secret from the provisioning process
[Note] Generated ConfigMap and Secret are same name as the OBC!
Lastly, expose the pod as a service so you can access the url from a browser. In this example, I exposed as a LoadBalancer
  # kubectl expose pod photo1 --type=LoadBalancer --name=photo1 -n your-namespace
To access via a url use the EXTERNAL-IP
  # kubectl get svc photo1
  NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
  photo1                       LoadBalancer   100.66.124.105   a00c53ccb3c5411e9b6550a7c0e50a2a-2010797808.us-east-1.elb.amazonaws.com   3000:32344/TCP   6d
NOTE: This is just one example of a Pod that can utilize the bucket information, there are several ways that these pod applications can be developed and therefore the method of getting the actual values needed from the Secrets and ConfigMaps will vary greatly, but the idea remains the same, that the pod consumes the generated ConfigMap and Secret created by the provisioner.
