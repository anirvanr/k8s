## Deploy ingress 
```
kubectl apply -f ingress-nginx.yaml
```

## Create a simple ingress
```
kubectl run echoheaders --image=k8s.gcr.io/echoserver:1.4 --replicas=1 --port=8080
kubectl expose deployment echoheaders --port=80 --target-port=8080 --name=echoheaders-x
kubectl expose deployment echoheaders --port=80 --target-port=8080 --name=echoheaders-y

kubectl apply -f https://raw.githubusercontent.com/anirvanr/k8s/master/nginx-ingress/ingress-echoheaders.yaml

kubectl get services ingress-nginx -n kube-ingress -owide

NAME            CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)          AGE       SELECTOR
ingress-nginx   100.71.196.44   a29c28f4b8b0811e685cb0a924c5a8a1-1593015597.us-east-1.elb.amazonaws.com   80/TCP,443/TCP   13m       app=ingress-nginx

curl -v -H "Host: bar.baz.com" http://a29c28f4b8b0811e685cb0a924c5a8a1-1593015597.us-east-1.elb.amazonaws.com/bar
```
