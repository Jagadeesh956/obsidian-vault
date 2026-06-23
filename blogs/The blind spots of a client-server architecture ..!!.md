



An application [server ] exposing a REST endpoint and client connects to it from another web application is quite common in this world of internet.

As the server is deployed in a microservice architecture with scalable instances like pods , exposed as a service , backed by a VIP or LB provisioned from a provider , the complexity of a request reaching to actual instance of the service that needs to serve the request grows enormously . 

In my current projects , most of the request path is same as below ...

**Client -> LB -> Ingress Gateway -> SideCar Proxy -> App container** 


Now, the application owner writes logging for a request that gets tracked when it reaches the actual pod [ infact app container ] . **How about requests that never reaches there for any reason in this blind path ??** Additionally , there are chances of missing metrics when server has bad logging which is very common ...
   1) Requests hits the Filter in Spring Application , fails to authenticate but logging starts inside a controller.
   2) There is no tracing enabled , only raw logs with status codes attached as tags if requests passes through that method.
   3) 


Client Application sees 503[ Service Unavailable ] which is not even a configured response code in the Server Application ..

I always had these questions in my mind ...
1) Where does this response come from ? 
2) How does App teams even identify them as requests ?? 
3) How accurate are the metrics my leadership celebrates as highest availability/ reliability ? 
4) How to make metrics more reliable and realistic ? 
   

The answer to these questions are >>>>

