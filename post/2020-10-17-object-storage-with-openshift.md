---
categories:
- tech
date: '2020-10-17'
filename: 2020-10-17-object-storage-with-openshift.md
tags:
- openshift
- s3
- kubernetes
title: Object storage with OpenShift Container Storage

---

OpenShift Container Storage is ...


## Creating a bucket

Creating a bucket works a bit like creating a persistent volume, although
instead of starting with a `PersistentVolumeClaim` you instead start with
an `ObjectBucketClaim` ("`OBC`"). An `OBC` looks something like this:

```
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: mybucket
spec:
  generateBucketName: mybucket
  storageClassName: ocs-storagecluster-ceph-rgw
```

With OCS 4.5, your out-of-the-box choices for `storageClassName` will be
`ocs-storagecluster-ceph-rgw`, if you choose to use Ceph Radosgw, or
`openshift-storage.noobaa.io`, if you choose to use the Noobaa S3 endpoint.

Because we set `generateName` in the `ObjectBucketClaim`, the above
configuration will result in a bucket named `mybucket-<some random
string>`. Because buckets exist in a flat namespace, the OCS documentation
recommends always using `generateName`, rather than explicitly setting
`bucketName`.

This will create a bucket in your chosen backend, along with two other
resources:

- A `Secret` containing the AWS-style credentials for interacting with the
  bucket via the S3 API, and
- A `ConfigMap` containing information about the bucket.

Both will have the same name as your `OBC`.  The `Secret` contains
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` keys, like this:

```
apiVersion: v1
kind: Secret
metadata:
  name: mybucket
type: Opaque
data:
  AWS_ACCESS_KEY_ID: ...
  AWS_SECRET_ACCESS_KEY: ...
```

The `ConfigMap` creates a number of keys that provide you (or your code)
with the information necessary to access the bucket:


```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mybucket
data:
  BUCKET_HOST: rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc.cluster.local
  BUCKET_NAME: mybucket-3ffec462-3871-4963-a7e7-1eb7102ebef5
  BUCKET_PORT: '80'
  BUCKET_REGION: us-east-1
  BUCKET_SUBREGION: ''
```

Note that `BUCKET_HOST` contains the internal S3 API endpoint. You won't be
able to reach this from outside the cluster. We'll tackle that in just a
bit.

## External connections to S3 endpoints

If you look at the routes available in your `openshift-storage` namespace,
you'll find the following:


## Noobaa vs Ceph Radosgw
