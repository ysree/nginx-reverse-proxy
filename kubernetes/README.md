
# Dynamic reverse proxy using nginx in Kubernetes

# Architecture 

    * Create a custom nginx deployment with a specific nginx.conf injected using a configmap.
    * Create a service which will expose this custom nginx deployment
    * Define an ingress with *.mydomain.com mapped to this service
    * With this setup we will have exper-0.mydomain.com pointing to exper-0.

## nginx.conf
```
server {
  listen 80;

  server_name ~^(?<subdomain>.*?)\.;
  resolver kube-dns.kube-system.svc.cluster.local valid=5s;

  location / {
    proxy_pass http://$subdomain.mynamespace.svc.cluster.local;
    proxy_set_header Host $host;
  }
}
```

Look at the configuration

* listen on port 80. Basic.
* For servername, we have a regex that will pull the first block of hostname to subdomain. This gets used a few lines down.
* Specify a resolver as nginx which has to resolve this into and actual IP(internal IP of svc).
* Now, inside location block grab everything and pass it over to http://$subdomain.mynamespace.svc.cluster.local which will resolve to the IP of the service.
* Also need to change Host as otherwise it will be set as http://$subdomain.mynamespace.svc.cluster.local instead of exper-0.mydomain.com.

We actually need a bit more sutff to support websocket.

```
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "Upgrade";
```

Add in a healthcheck.

```
location /healthz {
  return 200;
}
```

With all that in, it will look something like this.

```
server {
  listen 80;

  server_name ~^(?<subdomain>.*?)\.;
  resolver kube-dns.kube-system.svc.cluster.local valid=5s;

  location /healthz {
    return 200;
  }

  location / {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_pass http://$subdomain.msce0.svc.cluster.local;
    proxy_set_header Host $host;
  }
}
```

## Deploy
```
k apply -f configmap.yaml
k apply -f deployment.yaml
k apply -f service.yaml
k apply -f ingress.yaml
```

##### Reference

* https://blog.meain.io/2020/dynamic-reverse-proxy-kubernetes/