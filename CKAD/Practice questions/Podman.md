### Create a Dockerfile to deploy an Apache HTTP Server which hosts a custom main page
```bash
❯ cat Dockerfile
FROM 'docker.io/httpd:latest'
RUN echo "custom-page" > /usr/local/apache2/htdocs/index.html
```
>[!NOTE]
>just like Nginx has `/usr/share/nginx/html` , Apache has `/usr/local/apache2/htdocs`

---
### Build and see how many layers the image consists of
```bash
❯ podman build -t testapp .
STEP 1/2: FROM docker.io/httpd:latest
STEP 2/2: RUN echo "custom-page" > /usr/local/apache2/htdocs/index.html
COMMIT testapp
--> 0a0c15e1a00b
Successfully tagged localhost/testapp:latest
0a0c15e1a00bf1e62ce05cf12b07809dfb5a5e9344d872bfa578404d0854f0be
❯ podman images
REPOSITORY               TAG         IMAGE ID      CREATED             SIZE
localhost/testapp        latest      0a0c15e1a00b  About a minute ago  120 MB
docker.io/library/httpd  latest      1c65d54bce7e  7 days ago          120 MB
❯ podman image tree 0a0c15e1a00b
Image ID: 0a0c15e1a00b
Tags:     [localhost/testapp:latest]
Size:     120.2MB
Image Layers
├── ID: f4f8b983b714 Size: 81.05MB
├── ID: c333604d62d1 Size:  2.56kB
├── ID: 76ee51b9cb12 Size: 1.024kB
├── ID: 00461b2448cf Size: 6.023MB
├── ID: fbd4c7d8c989 Size: 33.13MB
├── ID: 3de31987e2c2 Size: 3.584kB Top Layer of: [docker.io/library/httpd:latest]
└── ID: a785e4342aab Size: 5.632kB Top Layer of: [localhost/testapp:latest]
```
>[!NOTE]
>To build images via podman, you can do it the same was as with docker. that being `podman/docker build -t <tag> <path>`

---
### Run the image locally, inspect its status and logs, finally test that it responds as expected
```bash
❯ podman run -p 9000:80 localhost/testapp
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.1.30. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.1.30. Set the 'ServerName' directive globally to suppress this message
[Thu Jun 18 19:32:42.419240 2026] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.68 (Unix) configured -- resuming normal operations
[Thu Jun 18 19:32:42.419319 2026] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
fd7a:115c:a1e0::d63b:2d64 - - [18/Jun/2026:19:32:54 +0000] "GET / HTTP/1.1" 200 12

# in another terminal
❯ curl localhost:9000
custom-page
```
---
### Run a command inside the pod to print out the index.html file
```bash
❯ podman exec -it b1f48653b2f0 bash
root@b1f48653b2f0:/usr/local/apache2# cat /usr/local/apache2/htdocs/index.html
custom-page
root@b1f48653b2f0:/usr/local/apache2# exit
exit
```
---
### Tag the image with ip and port of a private local registry and then push the image to this registry
```bash
❯ podman tag localhost/testapp $registryIP:$port/simpleapp
❯ podman push $registryIP:$port/simpleapp
```
>[!IMPORTANT]
>In order to tag an image , use `podman tag <registry>/<name> <ip>:<port>/<name`
>and to push it use `podman push <ip>:<port>/<name>`

>[!TIP]
>port for pushing images to a registry is usually 5000



---
### Verify that the registry contains the pushed image and that you can pull it
```bash
curl http://$registryIP:5000/v2/_catalog
{"repositories":["simpleapp"]}

# remove the image already present
podman rmi $registryIP:5000/simpleapp

podman pull $registryIP:5000/simpleapp

```
>[!NOTE]
>To check for the images in a registry, you can do so via `curl http://<ip>:5000/v2/_catalog`, and to pull an image from a registry you can use `podman pull <ip>:5000/<name>`

---
### Create a container without running/starting it
```bash
❯ podman create nginx
✔ docker.io/library/nginx:latest
Trying to pull docker.io/library/nginx:latest...
Getting image source signatures
Copying blob 868d78dceaed done   |
Copying blob ac5e3151b8c0 done   |
Copying blob 99181f19640f done   |
Copying blob 3461bb328618 done   |
Copying blob 72c03230f136 skipped: already exists
Copying blob 8da80f8205ea done   |
Copying blob c82e83b01f69 done   |
Copying config eaf6f38605 done   |
Writing manifest to image destination
5f1507054417c254a6c1909b5ac49912ec08492d8093fde66f9927417334d709
❯ podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS                     PORTS                 NAMES
5f1507054417  docker.io/library/nginx:latest  nginx -g daemon o...  6 seconds ago   Created                    80/tcp                eloquent_mendeleev
```
>[!IMPORTANT]
>To create a container without running it in podman you can use `podman create <image>`

---
### Export a container to output.tar file
```bash
❯ podman create nginx
d06384cee839c973938016c8c770bc3fc70a5160876252040a7b3153c0d008bf
❯ podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED        STATUS      PORTS       NAMES
d06384cee839  docker.io/library/nginx:latest  nginx -g daemon o...  3 seconds ago  Created     80/tcp      peaceful_wilson
❯ podman export d06384cee839 --output=output.tar
❯ ls output.tar
output.tar
```
>[!IMPORTANT]
>To export a container to an output.tar file, use the `podman export` with the `--output=<path>` flag

---
### Run a pod with the image pushed to the registry
```bash
kubectl run simpleapp --image=$registryIP:5000/simpleapp --port=80
```
---
### Log into a remote registry server and then read the credentials from the default file
```bash
podman login --username $YOUR_USER --password $YOUR_PWD docker.io
cat ${XDG_RUNTIME_DIR}/containers/auth.json
{
        "auths": {
                "docker.io": {
                        "auth": "Z2l1bGl0JLSGtvbkxCcX1xb617251xh0x3zaUd4QW45Q3JuV3RDOTc="
                }
        }
}
```
>[!IMPORTANT]
>To login to a registry server via podman , use the following command `podman login --username <username> --password <password> <host>`

