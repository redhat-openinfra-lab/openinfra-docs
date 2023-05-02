# Ceph RGW Gateway

## Introduction / Information

Pool created for RGW:

| Pool Name | Description |
| --- | --- |
| .rgw.root | Stores information records | 
| default.rgw.control | Control pool |
| default.rgw.meta | User keys and critical metadata |
| default.rgw.log | Logs of all buckets and object actions | 
| default.rgw.buckets.index | Indexes of buckets |
| default.rgw.buckets.data | Bucket data |
| default.rgw.bucket.non-ec | MPU uploads |


## Service Specification File

```
# vi rgw-service.yaml
service_type: rgw
service_id: myrealm.myzone
service_name: rgw.myrealm.myzone
placement:
  count: 4
  hosts:
  - serverd.lab.example.com
  - servere.lab.example.com
spec:
  rgw_frontend_port: 8080
  rgw_realm: realm_name
  rgw_zone: zone_name
  ssl: true
  rgw_frontend_ssl_certificate: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  networks:
    - 172.20.17.0/24    
```

Deploy the service:
```
# ceph orch apply -i rgw-service.yaml
Scheduled rgw.myrealm.myzone update...
```

## Multisite Deployment

### Configuration

Zone: backed by its own RHCS cluster with one or more RGWs associated with it.  
Zone Group: set of one or more zones.  Data replicated to all zones in the zone group with one zone being the primary zone.  
Realm: Represents the global namespace; contains one or more zone groups; one zone group is the primary zone group.  

Configure the primary zone:

* Create a realm  
* Create a primary zone group  
* Create primary zone  
* Create system user  
* Commit the changes  
* Create RGW service for the primary zone  
* Udpate the configuration database  

```
# radosgw-admin realm create --default --rgw-realm=gold
# radosgw-admin zonegroup create --rgw-zonegroup=us --master --default --endpoints=http://cephstorage01:80
# radosgw-admin zone create --rgw-zone=datacenter01 --master --rgw-zonegroup=us --endpoints=http://cephstorage01:80 --access-key=blah --secret=blahblah --default
# radosgw-admin user create --uid=sysadm --display-name="SysAdmin" --access-key=blah --secret=unknown --system
# radosgw-admin period update --commit
# ceph orch apply rgw gold-service --realm=gold --zone=datacenter01 --placement="1 cephstorage01"
# ceph config set client.rgw rgw_zone datacenter01
```

Configure the secondary zone:  

* Pull the realm configuration  
* Pull the period  
* Create a secondary zone  
* Commit the changes  
* Create RGW service for the secondary zone  
* Update the configuration database  

```
# radosgw-admin realm pull --rgw-realm=gold --url=http://cephstorage01:80 --access-key=blah --secret=unknown --default
# radosgw-admin period pull --url=http://cephstorage01:8000 --access-key=blah --secret=unknown 
# radosgw-admin zone create --rgw-zone=datacenter02 --master --rgw-zonegroup=us --endpoints=http://cephstg01:80 --access-key=blah --secret=blahblah --default
# radosgw-admin period update --commit
# ceph orch apply rgw gold-service --realm=gold --zone=datacenter02 --placement="1 cephstg01"
# ceph config set client.rgw rgw_zone datacenter02
```

### Failover

During a primary site failure, the secondary site continues to serve read and write operations but new buckets and users cannot be created unless a secondary site is promoted as a replacement.

To promote a secondary zone, modify the zone and zone group, commit the period.
```
# radosgw-admin zone modify --master --rgw-zone=datacenter02
# radosgw-admin zonegroup modify --rgw-zonegroup=us --endpoints=http://cephstg01:80
# radosgw-admin period update --commit
```

## S3 API

### User Authentication

RADOS Gateway supports only a subset of the Amazon S3 API policy language applied to buckets. No policy support is available for users, groups, or roles. Bucket policies are managed through standard S3 operations rather than using the radosgw-admin command.

Create an RGW User:  
```
radosgw-admin user create --uid=s3user --display-name="User Name" --email=user@example.com [--access-key=blah] [--secret=unknown]  
```
> NOTE: If the access key and secret are not specified, the system will generate them.

Regenerate only the secret key of an existing user:  
```
# radosgw-admin key create --uid=s3user --access-key="8PI2D9ARWNGJI99K8TOS" --gen-secret
```

Generate additional access key for an existing user:  
```
# radosgw-admin key create --uid=s3user --gen-access-key
```

Remove an access key from a user:  
```
# radosgw-admin key rm --uid=s3user --access-key="8PI2D9ARWNGJI99K8TOS"
```

Disable or enable an S3 user:  
```
# radosgw-admin user [suspend|enable] --uid=s3user 
```

Modify user's access level:  
```
# radosgw-admin user modify --uid=s3user --access=[read|write|readwrite|full]
```
> NOTE: The `full` access includes readwrite and the access control management capability.

### Quotas

Set quota on number of objects; scope of user:  
```
# radosgw-admin quota set --quota-scope=user --uid=s3user --max-objects=1000
# radosgw-admin quota enable --quota-scope=user --uid=s3user
```

Set quota on size for loghistory; scope of bucket:  
```
# radosgw-admin quota set --quota-scope=bucket --uid=loghistory --max-objects=1024B
# radosgw-admin quota enable --quota-scope=bucket --uid=loghistory
```

Global quotas:  
```
# radosgw-admin global quota set --quota-scope=bucket --max-objects=2500
# radosgw-admin global quota enable --quota-scope=bucket 
```
> NOTE: Use the radosgw-admin period update --commit command to commit the changes or alternatively restart the RGW instances to pick up the change.  

Retrieve user info and stats:  
```
# radosgw-admin user info --uid=s3user
# radosgw-admin user stats --uid=s3user 
```

Usage statistics of a user at a specific time:  
```
# radosgw-admin usage show --uid=s3user --start-date=start --end-date=end
```

### URL Access Method

Default access method is http://server.example.com/bucket.  To enable wildcard URLs using the http://bucket.server.example.com method, set the `rgw_dns_name` parameter.  

```
# ceph config set client.rgw rgw_dns_name dns_suffix
```
> NOTE: Where dns_suffix is the FQDN to be used.  


### Bucket Metadata

View the metadata of a bucket:
```
# radosgw-admin metadata get bucket:testbucket
{
    "key": "bucket:testbucket",
    "ver": {
        "tag": "_hLrerQczwHjyMJIyj-TAdNU",
        "ver": 1
    },
    "mtime": "2023-04-20T18:40:27.109229Z",
    "data": {
        "bucket": {
            "name": "testbucket",
            "marker": "e85a3c82-f275-441f-acc1-e117e6b23cb9.28819.1",
            "bucket_id": "e85a3c82-f275-441f-acc1-e117e6b23cb9.28819.1",
            "tenant": "",
            "explicit_placement": {
                "data_pool": "",
                "data_extra_pool": "",
                "index_pool": ""
            }
        },
        "owner": "operator",
        "creation_time": "2023-04-20T18:40:23.269355Z",
        "linked": "true",
        "has_bucket_info": "false"
    }
}
```

### Swift API

Install the swift client:  
```
# sudo pip3 install --upgrade python-swiftclient
```

Create a subuser:  
```
# radosgw-admin subuser create --uid=s3user --subuser=username:swift --access=[read|write|readwrite|full]
```
> NOTE: --uid specifies an existing account.e   

Create swift authentication key:  
```
# radosgw-admin key create --subuser=username:swift --key-type=swift --secret=unknown
```

Remove swift subuser key:   
```
# radosgw-admin key rm --subuser=username:swift 
```

Modify swift user access:  
```
# radosgw-admin subuser modify --subuser=username:swift --access=[read|write|readwrite|full]
```

Remove swift user access:  
```
# radosgw-admin subuser rm --subuser=username:swift [--purge-data] [--purge-keys]
```

Creating users within tenants:
```
# radosgw-admin user create --tenant devtenant --uid=devuser --display-name="A Swift User" --subuser=devuser:swift --key-type swift --secret=unknown
```
> NOTE: Any further reference to the subuser, must include the tenant (i.e. devtenant$devuser:swift)

Get info/List containers
```
# swift -A http://172.20.17.120:8080/auth/1.0 -U operator:swift -K unknown stat [container]
# swift -A http://172.20.17.120:8080/auth/1.0 -U operator:swift -K unknown list
```  

Create a container:  
```
# swift -A http://172.20.17.120:8080/auth/1.0 -U operator:swift -K unknown post mycontainer
```  

Upload an object to a container:
```
# swift -A http://172.20.17.120:8080/auth/1.0 -U operator:swift -K unknown upload mycontainer /tmp/10B.bin
```  


