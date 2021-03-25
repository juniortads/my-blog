+++
date = "2019-01-15"
title = "Istio Service Mesh + Apollo Server for GraphQL [en-US]"
+++

`A Match made in heaven.`

New architectural patterns new problems to be thought of and solved. That’s for sure when you work with system architecture and not different with microservices architecture adoption. I don’t want to make you give up for sure :) Overall there are more advantages than problems.
Basically, you need think on D+0 in traffic monitoring, access control, discovery, security, resilience.

![Istio](/images/istio-1.jpeg)

There are a lot of good things, right?

>I’m assuming that you have some basic concepts about Kubernetes (k8s)

https://kubernetes.io/

#### Service Mesh

![Istio](/images/istio-2.png)

So, What is Service Mesh? It is a configurable infrastructure layer for microservices application. It makes communication between service instances flexible, reliable, and fast… it provides: service discovery, load balancing, encryption, authentication and authorization, support for the circuit breaker and other capabilities. Istio does all that, but it doesn’t require any changes to the code of any of those services.

##### How does Istio work?

![Istio](/images/istio-3.png)

The magic happens, Istio deploys a proxy (sidecar) next to each service. Istio service mesh is a sidecar container implementation of the features and functions needed when creating and managing microservices.
By using the sidecar model, Istio runs in a Linux container in your Kubernetes pods.

#### Setup
Download and extract the latest release.

```bash
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.5
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
```
After Installing Istio with default mutual TLS authentication, when you use this option deployed workloads are guaranteed to have Istio sidecars installed.

```bash
kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```
If you want other options for install check the following link, please https://istio.io/docs/setup/kubernetes/quick-start/

Don't forget to enable Istio injection configuration. It automatically inject Envoy containers into your application pods.

```bash
kubectl label namespace <namespace> istio-injection=enabled
```
Or if you prefer, you can inject manually
```bash
istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -
```
#### Verifying the installation

```bash
kubectl -n istio-system get pods
```
![Istio](/images/istio-4.png)

```bash
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath=’{.items[0].metadata.name}’) 3000:3000
```

Go http://localhost:3000

![Istio](/images/istio-5.png)

#### Apollo Server for GraphQL

![graphql](/images/graphql-1.gif)

`GraphQL` is a query language for APIs developed and used by Facebook since 2012. GraphQL delivers to the client only what was requested. Another benefit I like is add new fields and types to your GraphQL API without impacting existing queries, basically you have a single evolving version. So, Why should I use GraphQL along with my microservices?

The other day I was reading Jeff Handley’s blog, where he describes his experiences when he started using GraphQL

>…Another detail we have found important and highly successful: Our GraphQL layer IS NOT implemented or operated by the teams building RESTful services. The UI teams build that layer and Howard’s team provides the platform and runs the service.

> This lets the service teams concentrate on REST and the domain model. GraphQL is an implementation detail of the UI layer–a technology chosen by UI, not services. This has avoided the whole REST vs. GraphQL debate with each of the dozens of service teams building APIs. They get to do their thing the way they want. For all they care, the UI consumes their services directly. We just happen to put a GraphQL server in between. We can centralize the GraphQL implementation into a smaller community of developers where we can foster reuse and commonalities more easily.

`COOL`. That’s it, I like it because that happens most of the time.
Sam Newman said in his article that Phil calçado called it Backend For Frontend (BFF).

I strongly recommend that you read this article!

https://samnewman.io/patterns/architectural/bff/

#### What is Backends For Frontends?

>Conceptually, you should think of the user-facing application as being two components — a client-side application living outside your perimeter, and a server-side component (the BFF) inside your perimeter. Sam Newman

Then, how can I implement GraphQL with Istio Service Mesh?
I have opted to use the apollo server platform as an implementation of GraphQL. Apollo Server is really easy to start.

```javascript
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');

const books = [
  {
    title: 'Harry Potter and the Chamber of Secrets',
    author: 'J.K. Rowling',
    age: 17
  },
  {
    title: 'Jurassic Park',
    author: 'Michael Crichton',
    age: 20
  },
];

const resolvers = {
  Query: {
    books: () => books,
  },
};

const server = new ApolloServer({ 
  typeDefs, 
  resolvers,
  engine: {
    apiKey: process.env.ENGINE_API_KEY
  } 
});

server.listen().then(({ url }) => { 
  console.log(`🚀  Server ready at ${url} ${process.env.ENGINE_API_KEY} `); 
});
```

---

```javascript
const { gql } = require('apollo-server');

const typeDefs = gql`
  type Book {
    title: String
    author: String
    age: Int
  }
  type Query {
    books: [Book]
  }
`;

module.exports = typeDefs;
```

Now you can create Dockerfile

```bash
FROM node:10-alpine
EXPOSE 4000
COPY package.json package-lock.json ../
RUN npm install && npm cache clean --force
COPY src/ ./src/
USER 1000 
CMD ["npm", "start"]
```

```json
...

"dependencies": {
    "apollo-server": "^2.3.1",
    "graphql": "^14.0.2"
 }
```

…to install the dependencies used:

```bash
npm install --save apollo-server graphql
npm i graphql
```

Finally, we can deploy for Kubernetes.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: graphql
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: graphql
        app: graphql
        version: 1.0.0
    spec:
      serviceAccountName: default
      containers:
      - name: graphql
        image: juniortads/demo-app:latest
        env:
        - name: ENGINE_API_KEY
          value: "CREATE API ENGINE KEY - https://engine.apollographql.com"
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 4000
          name: http-port
---
apiVersion: v1
kind: Service
metadata:
  name: graphql
spec:
  ports:
  - port: 4000
    name: http
  selector:
    app: graphql
```

```bash
kubectl apply -f graphql-app.yaml
```
Service and Pods should be working now!
But to expose your Graphql Service to the cluster's external world , you need to first configure control ingress traffic.

https://istio.io/latest/docs/tasks/traffic-management/ingress/

Basically, The Istio Gateway is what tells the istio-ingressgateway pods which ports to open up and for which hosts.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: graphql-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: graphql
spec:
  hosts:
  - "*"
  gateways:
  - graphql-gateway
  http:
  - match:
    - uri:
        exact: /graphql
    route:
    - destination:
        host: graphql
        port:
          number: 4000
```

```bash
kubectl apply -f gateway-graphql.yaml
```

Open your browser:

```bash
http://${GATEWAY_URL}/graphql
```
![Istio](/images/istio-6.png)

You can monitor the traffic of your Apollo Server using Grafana

![Istio](/images/istio-7.png)

#### Conclusion
Istio brings increased visibility to your applications. There are a lot different features with Istio that I didn't talk about in this article, but I encourage you to continue exploring.

![Istio](/images/istio-8.gif)

Through GraphQL we have the perfect match in Backends For Frontends usage when you need to create experience APIs for each communication channel. Using GraphQL eliminates ad hoc endpoints and round-trip object retrieval making it simple.
With simplicity comes stability.

I hope this was helpful and I look forward to hearing from you about it.
https://github.com/juniortads/demo-app

#### References
- https://blog.jayway.com/2018/10/22/understanding-istio-ingress-gateway-in-kubernetes/
- https://jeffhandley.com/2018-09-13/graphql-is-not-odata
- https://samnewman.io/patterns/architectural/bff/