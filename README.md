# Problems encountered

I encountered numerous problems with this project. 

Did everything as mentioned in the instructions but just could not get the app to run.

After debugging and trying things out for more than 1 week and burning through almost all the budget I was still encountering the same kind of problem:


- after exposing the frontend and the reverseproxy, changing the environment.ts files apiHost to the reverse proxys external IP and accessing the frontend from the frontend API I kept getting an "Unknown error" popup in the frontend and - with Developer Tools enabled in Chrome - it always showed a CORS error about `No Access-Control-Allow-Origin header is present on the requested resource`
- went through countless of threads in the Udacity forums and saw numerous students having the same issue. But most of the solutions involved wrong AWS credentials.
- The services would just not talk to each other. In the backend-user and -feed logs there were no user activities being logged. I also could not even send a postman request to the reverse proxys external endpoint on `8080/api/v0/feed` (would always get a 502 Bad gateway response)

# Solution
0. Create the eks cluster using `eksctl`. It takes care of everything and does not require setting up roles for the cluster and the nodegroup
1. Set up the reverse proxy at last, i.e. set up deployment and service of backend-user, backend-feed and frontend. Then do the reverseproxy.
2. In the `nginx.conf` make sure the name of the service matches the servicename as defined in the corresponding service yaml files.
```
# E.g. if this is the entry in nginx.conf

upstream user {
     server backend-user:8080;  
 }

 # then in the corresponding service yaml file the service has to be called 'backend-user' as well! not 'backend-user-svc' or 'udacity-api-user' or whatever!

 E.g. in <backend-user-service.yaml>
 

 metadata:
  labels:
    service: backend-feed
  name: backend-feed

```
3. In the deployment yaml files add the cpu and memory requests and limit as resources. Although not needed for running the app it is seemingly required for the autoscaling. If the resources are not set then after applying auto scaling and running `kubectl describe hpa` the current load will be shown as `unknown` and the submission will not pass review.
4. This is the most important part (!!!). If the deployment yaml file uses an entry called `app` like this 
```
template:
    metadata:
      labels:
        app: backend-feed
```

Then in the corresponding service yaml file under `selector` the entry should be named `app` as well, not `service` or `run` or whatever!

```
selector:
    # service: backend-feed  # this is wrong and will cause the run to fail 
    app: backend-feed
```

Unfortunately this inconspicuous detail is not mentioned in the instructions. I encountered it by chance in this thread https://knowledge.udacity.com/questions/423408

5. `kubectl apply` the deployments and services. Then expose frontend and reverseproxy as mentioned in the instructions.
6. Afterwards change the `apiHost` entry in the frontends environment.ts and environment.prod.ts files, using the reverseproxys external address as mentioned in the instructions. Note that the 8080/api/v0/feed has to be included as well!
7. The rolling update of the frontend image with `kubectl set image` did not seem to do anything for me. I could set a tag for the frontend image that does not even exist and there would be no error message. Just to be sure I deleted the frontend deployment and reapplied the deployment config.
8. For me there was no need to update also the URL in the configmap yaml file with the frontends external IP.
9. Afterwards I was finally able to access the frontend using the external IP, this time without any error messages, and was able to see the feed, login or register new users, etc.
10. Set up a metrics server https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html. This was not mentioned anywhere in the instructions and I had to find out after I failed the first submission. First deploy metrics server
`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`. Then verify it is running `kubectl get deployment metrics-server -n kube-system`. Then set up autoscaling for one of the pods `kubectl autoscale deployment backend-user --cpu-percent=70 --min=2 --max=3`. Then run `kubectl describe hpa` but wait a few minutes until the current load does not show up as `unknown` anymore.
11. For the port-forwarding solution I had to port-forward the deployment, not the service, otherwise the terminal would hang and would error out after a few minutes. I.e. `kubectl port-forward deployment/reverseproxy 8080:8080` -> Open another terminal -> `kubectl port-forward deployment/frontend 8100:80` -> access `localhost:8100` in Browser.



In the end I had a problem with even setting up an EKS cluster in the Udacity AWS account since I ran against some kind of EC2 limit. So I had to set up an EKS cluster in a totally different account, while exposing a public Endpoint for it so that it would be publicly accessible. Meaning I had an S3 bucket for media storage in one AWS accoung and the RDS + EKS cluster set up in another account.

