Bosh release for redis [![Build Status](https://travis-ci.org/cloudfoundry-community/redis-boshrelease.png?branch=master)](https://travis-ci.org/cloudfoundry-community/redis-boshrelease/)
===========================================================================================================================================================================================

One of the fastest ways to get [redis](http://redis.io) running on any infrastructure is to deploy this bosh release.

Supports [consul](http://consul.io) for service advertisements.

Usage
-----

To use this BOSH release, first upload it to your bosh:

```
bosh upload release https://redis-boshrelease.s3.amazonaws.com/boshrelease-redis-5.tgz
```

To deploy it you will need the source repository that contains templates:

```
git clone https://github.com/cloudfoundry-community/redis-boshrelease.git
cd redis-boshrelease
git checkout v5
```

For [bosh-lite](https://github.com/cloudfoundry/bosh-lite), you can quickly create a deployment manifest & deploy a 3 VM cluster:

```
templates/make_manifest warden normal
bosh -n deploy
```

For Openstack (Nova Networks), create a three-node cluster:

```
templates/make_manifest openstack-nova
bosh -n deploy
```

For AWS EC2, create a three-node clusterg:

```
templates/make_manifest aws-ec2
bosh -n deploy
```

To rename the deployment, set the `$NAME` environment variable:

```
NAME=my-redis templates/make_manifest warden normal
```

### Consul service advertisement

Co-locate the `consul` job template from the [consul-boshrelease](https://github.com/cloudfoundry-community/consul-boshrelease) and the deployment will automatically advertise each VM on [consul](http://consul.io).

There is an example available for bosh-lite.

First, deploy the `consul-boshrelease` into bosh-lite. The redis templates for consul are pre-configured to find the consul cluster:

```
templates/make_manifest warden consul
bosh -n deploy
```

To confirm that redis is now discoverable on any VM connected to the consul cluster (including the 3 redis VMs deployed above), target the [consul DNS](http://www.consul.io/docs/agent/dns.html) service:

```
$ bosh ssh redis_leader_z1/0

# dig @127.0.0.1 -p 8600 redis.service.consul +short
10.244.2.14
10.244.2.10
10.244.2.6

# dig @127.0.0.1 -p 8600 master.redis.service.consul +short
10.244.2.6
# dig @127.0.0.1 -p 8600 slave.redis.service.consul +short
10.244.2.14
10.244.2.10
```

From any machine running the consul agent you can also discover the redis cluster via the [HTTP API](http://www.consul.io/docs/agent/http.html):

```
$ curl 127.0.0.1:8500/v1/catalog/services
{"consul":null,"redis":["master","slave"]}

$ curl 127.0.0.1:8500/v1/catalog/service/redis?tag=master
[{"Node":"redis-warden-redis_leader_z1-0","Address":"10.244.2.6","ServiceID":"redis","ServiceName":"redis","ServiceTags":["master"],"ServicePort":6379}]
```

Create new final release
------------------------

To create a new final release you need to get read/write API credentials to the [@cloudfoundry-community](https://github.com/cloudfoundry-community) s3 account.

Please email [Dr Nic Williams](mailto:&#x64;&#x72;&#x6E;&#x69;&#x63;&#x77;&#x69;&#x6C;&#x6C;&#x69;&#x61;&#x6D;&#x73;&#x40;&#x67;&#x6D;&#x61;&#x69;&#x6C;&#x2E;&#x63;&#x6F;&#x6D;) and he will create unique API credentials for you.

Create a `config/private.yml` file with the following contents:

```yaml
---
blobstore:
  s3:
    access_key_id:     ACCESS
    secret_access_key: PRIVATE
```

You can now create final releases for everyone to enjoy!

```
bosh create release
# test this dev release
git commit -m "updated redis"
bosh create release --final
git commit -m "creating vXYZ release"
git tag vXYZ
git push origin master --tags
```
