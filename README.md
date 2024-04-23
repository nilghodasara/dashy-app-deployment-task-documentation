# dashy-app-deployment-task-documentation

---------------Deployment of Dashy app with kubernetes-------------------

setp-1
-> setup eks cluster in aws 

eksctl create cluster --name my-eks-cluster --region ap-south-1 --node-type t3.medium --nodes 2

-> and configure

aws eks --region ap-south-1 update-kubeconfig --name my-eks-cluster

step-2
-> setup ingress(alb) on this cluster

step-3
-> install metrix server (for scalable application with HPA)

step-4
-> now deploy the app with kubernetes. Here some yaml files which required for deployment dashy app.

-> use latest image of dashy app(lissy93/dashy)

1 --> dashy-deployment.yaml (kubectl apply -f dashy-deployment.yaml)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: dashy-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dashy
  template:
    metadata:
      labels:
        app: dashy
    spec:
      containers:
      - name: dashy
        image: lissy93/dashy
        ports:
          - containerPort: 8080
        resources:
          requests:
            cpu: "400m"
            memory: "1024Mi"
          limits:
            cpu: "400m"
            memory: "1024Mi"

-- deployment

kubectl get deployment dashy-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
dashy-deployment   1/1     1            1           11h

after apply this yaml 2 replicas created for this deployment with lissy93/dashy
image with cont port 8080

2 --> dashy-service.yaml (kubctl apply -f dashy-service.yaml)

apiVersion: v1
kind: Service
metadata:
  name: dashy-service
spec:
  selector:
    app: dashy
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP

-- service

kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
dashy-service   ClusterIP   10.100.120.61   <none>        80/TCP    21h

this file make service for this deployment with clusterip type

3 --> dashy-ingress.yaml (kubectk apply -f dashy-ingress.yaml)

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashy-ingress
  annotations:
    alb.ingress.kubernetes.io/target-type: "ip"  # Specify the target type
    alb.ingress.kubernetes.io/scheme: "internet-facing"  # Specify the scheme of the ALB
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dashy-service
                port:
                  number: 80

-- ingress

 kubectl get ingress
NAME            CLASS   HOSTS   ADDRESS                 PORTS    AGE
dashy-ingress   alb     *       k8s-default- .......    80      12h

after apply this yaml it makes ingress alb in aws and create target group in this alb

4 --> dashy.HPA.yaml (kubectl apply -f dashy.HPA.yaml)

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: dashy-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dashy-deployment
  minReplicas: 2  # Minimum number of pods
  maxReplicas: 3  # Maximum number of pods
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10  

-- HPA

kubectl get hpa
NAME      REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
dashy-hpa Deployment/dashy-deployment   0%/10%    2         3        2          10h


this yaml file make HPA for deployment. it means when pods get traffic it will automatic sacle up on base of averageutilization and when pods have no traffic and averageutilization goes low then it will automatic sale down.

in this yaml min replicas - 2
             max replicas - 3
             averageutilization - 10%(for this practical only)


----- Let's talk about what i do for this task ----

after make deployment, service , ingress and hpa i check the pods its running or not then i will check alb automatic creted in aws and with its targetgroup and last i will check my dasy app with aws alb DNS and dashy app properly working.

----- for auto scaling the pods ----

for autosacling i give load to pod because pods cpu utilization goes high and Trigger my hpa configration of average cpu utilization goes up to 10% then it will sacle up by one pod and max to 3 pods.newly created pod also goes in aws alb automatically.

for give load to pod use stress-ng(for alpine) cmd 

login in the pod and run this cmds

apk update
apk add stress-ng
stress-ng -c 50 -t 20m &


now i kill all process which are running in this pod with stress-ng cmd now i will kill all process with 

pkill stress-ng       (cmd)

after do that average cpu utilization goes down and decrese pod to desire capacity.
and this pod also draining from aws alb also...

---------------------------- THANK YOU ------------------------------------
