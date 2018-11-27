# Setting a private registry

This is used to setup a private registry for installing ICP.

Will take `registry.ibm.local` as the private registry domain and `192.168.122.15` as the registry ip.

## 1. Create registry related directories

```
mkdir -p /var/lib/private-registry/{auth,registry,certs}
```

## 2. Generate self-signed certificate

```
cd /var/lib/private-registry/certs
openssl req -newkey rsa:4096 -sha256 -nodes -keyout registry.ibm.local.key -x509 -days 36500 -out registry.ibm.local.crt -subj "/C=US/ST=California/L=Los Angeles/O=Your Domain Inc/CN=registry.ibm.local"
```

## 3. Trust the private registry on all nodes

```
echo 192.168.122.15 registry.ibm.local >> /etc/hosts

mkdir -p /etc/docker/certs.d/registry.ibm.local
scp /var/lib/private-registry/certs/registry.ibm.local.crt /etc/docker/certs.d/registry.ibm.local/ca.crt
```

## 4. Create the private registry username and password

The registry username is `user1` and password is `pass1`.

```
docker run --rm --entrypoint htpasswd ibmcom/registry:2.6.2 -Bbn user1 pass1 > /var/lib/private-registry/auth/htpasswd
```

## 5. Start the private registry

```
cd /var/lib/private-registry

docker run -d \
    --restart=always \
    --name registry \
    --net host \
    -v $(pwd)/auth:/auth \
    -v $(pwd)/certs:/certs \
    -v $(pwd)/registry:/var/lib/registry \
    -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.ibm.local.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/registry.ibm.local.key \
    -e REGISTRY_AUTH=htpasswd \
    -e REGISTRY_AUTH_HTPASSWD_REALM=Private-Registry-Realm \
    -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
    ibmcom/registry:2.6.2
```

## 6. Login the private registry

```
docker login -u user1 -p pass1 registry.ibm.local
```

## 7. Push all ICP images to private registry

```
docker tag ibmcom/pause-amd64:3.1 registry.ibm.local/ibmcom/pause:3.1
docker push registry.ibm.local/ibmcom/pause:3.1
...
...
...
docker tag ibmcom/xxx-amd64:zzz registry.ibm.local/ibmcom/xxx:zzz
docker push registry.ibm.local/ibmcom/xxx:zzz
```

## 8. Remove all local images

This is an optional step, but after running this, we can save a lot of disk space.

```
docker rmi -f $(docker images -q)
```

## 9. Configure config.yaml with private registry

Append following lines in `cluster/config.yaml`

```
image_repo: registry.ibm.local/ibmcom
private_registry_enabled: true
docker_username: user1
docker_password: pass1
```

## 10. Start the ICP installation

Now, you can start installing ICP by using private registry.

## 11. View the Registry

```
# cat listRegistry
#!/bin/bash
# %W% %G% %U%
#
#       View private registry v2,
#               if PERSISTENT_REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY is mounted on your system
#
#       Enter the full path to your private registry v2
PERSISTENT_REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY="/var/lib/private-registry/registry"
#
find $PERSISTENT_REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY -print | \
        grep 'v2/repositories' | \
        grep 'current' | \
        grep -v 'link' | \
        sed -e 's/\/_manifests\/tags\//:/' | \
        sed -e 's/\/current//' | \
        sed -e 's/^.*repositories\//    /' | \
        sort > /tmp/a1
cat /tmp/a1
wc -l /tmp/a1 > /tmp/a2
echo "Number of images: `cat /tmp/a2 | awk {'print $1'}`"
echo "Disk space used:  `du -hs $PERSISTENT_REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY`"
#rm /tmp/a1 /tmp/a2

```

Curl Commands

curl -k -X GET https://user:password@registry.ibm.local/v2/_catalog | python -m json.tool

List 2000 entries:

curl -k -X GET https://user:password@registry.ibm.local/v2/_catalog?n=2000 | python -m json.tool

## 12. Retag all ICP images by removing -amd64 from the image and push to registry

ICP - load ICP 3.1 images

```
docker images --format 'docker tag  {{.Repository}}:{{.Tag}} registry.ibm.local/{{.Repository}}:{{.Tag}}' | sed 's/-amd64//2' > tags.sh
docker images --format 'docker push registry.ibm.local/{{.Repository}}:{{.Tag}}' | sed 's/-amd64//1' > push.sh
chmod +x tags.sh push.sh
./tags.sh
./push.sh
```
** Credit ** : Zhi Wei Chen

## 13. Using Letsencrypt Certificates

### Rename SSL certificates
### [https://community.letsencrypt.org/t/how-to-get-crt-and-key-files-from-i-just-have-pem-files/7348](https://community.letsencrypt.org/t/how-to-get-crt-and-key-files-from-i-just-have-pem-files/7348)

```
cd /etc/letsencrypt/live/domain.example.com/
cp privkey.pem domain.key
cat cert.pem chain.pem > domain.crt
chmod 777 domain.crt
chmod 777 domain.key
```

### https://docs.docker.com/registry/deploying/
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /etc/letsencrypt/live/domain.example.com:/certs \
  -v /opt/docker-registry:/var/lib/registry \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
  
### List images
#### https://domain.example.com/v2/_catalog
