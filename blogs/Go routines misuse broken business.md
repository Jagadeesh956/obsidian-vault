


We have a in house platform called Enterprise Configuration Management that manages configuration for various micro services to update the consuming services dynamically whenever someone changes configuration in version control systems like git , bitbucket . 


As the platform exposes REST API endpoints for any microservice to consume the config it needs, the platform acts similar to spring cloud consul which requires minimal configuration changes in the properties file along with many other features. 

Since this platform has various consumers that run their services in different tech stack like Go, Java , Python and Node.js ..... the platform provider decided to provide SDKs for each of these programming languages for easy interaction when the dependencies modules are imported and implemented by the consumers abstracting the calls to platform and periodically updating the config if changes were detected . 


As part of providing a feature called SSE [ server sent events ] , instead of client SDKs periodically fetching config every 2min in the Go SDK, the developer pushed a change that infinitely creates a go client [ a routine ] to connect to the platform in case of failure with retries . 

As the product is released , after few days a consumer from loyalty organization implemented the SSE and released to production on some X date , deployed the service into a K8s cluster provided by organization in private cloud  . 


**Time :-** T 

On the other side of story, a major SRB [ service restoration bridge ] arised from a very critical platorm called safekey [ OTP sender ] built for amex client transaction verification reporting impacts intermittently by multiple customers .

**"The ball is in safekey court to find out RCA"**

**Time :-** T + 2 hours 
The safekey team has no idea why it's happening since they haven't deployed any changes and started working with all dependent teams including the cloud operations , after few hours of debugging the cloud operations found that the istio-gateway pods are reporting liveness failures in a specific pattern matching to safekey failures  . As the gateway is shared by all the services deployed in that cluster , cloud operation teams are unable to identify what exactly is causing those failures. 

**"The ball is in cloudOps queue to find the RCA "**

**Time :-** T + 4 hours 

Time kept moving , many teams were involved including vendor for service mesh, OCP etc to help the cloud ops to identify the issue. Multiple leaders joined the call as it's impacting business . After 18+ hours of debugging , the cloud SRE team identified that the ECM platform service that is hosted on same cluster is seeing same pattern of crashloop backoff for the pods in the zones where safekey failures , istio-gateway pods are reporting . 

**Time :-** T + 10 hours 
An engineer tried to scale down all the pods of ECM in a specific zone to 0 and safekey failures stopped . "The eureka moment", however there is no correlation between safekey and ECM . 
ECM support was also reporting crashloop backoff since same time as safekey failures to cloud ops , however it was not cared much since it's not impacting entire platform or very intermittentlty happening . "No one really thought of correlated events ".

**"The ball is in ECM Ops queue to find the RCA"**
**Time :-**** T + 15 hours 
ECM support(our team ) got paging at 12AM , showing the evidence of safekey failures stopped after scaling down ECM Pods on a specific zone . As we are sure that it could happen only from external system calling us with huge number of requests, we were fighting back with cloud ops to not blame our system and figure out which system is bombarding the requests to ECM . [ platform side logging was poor on SSE related calls ], never shows in traffic similar to usual rest api calls.  After so much of discussion we convinced cloud ops to isolate our traffic to a specific zone and enable the other for safekey to debug further on this issue.

**Time :-** T+ 16 hours 
We tried to see anything that can be derived from our logs , I found that SSE related failure logs are reporting way huge sometimes [ level ERROR in Opensearch dashboard manual filteration ].
Somehow we got to know the only client who has been integrated with SSE feature and they were pulled to call , they confirmed that a change was deployed a day before enabling this feature provided by ECM . 

**"The ball is in loyalty team  queue to find the RCA"**


**Time :-** T+ 18 hours

The loyalty team reviewed the logs and confirmed they are seeing many failures calling ECM and a specific log saying "ECM client creation... " keeps on printing infinite times.

As the logic behind this client creation is provided by ECM , the ball came back to ECM court for root cause . 


All the teams were suggesting to backout the loyalty change that enabled SSE, however  the internal policies within loyalty domain are different to move a production change with many stages of testing , UAT signoff and validation etc... This has killed lot of time as well to bring in respective leaders onto call .

Once the change is reverted , loyalty team asked our ECM platform team to explain the root cause for those infinite client creations and our dev team confirmed it due to enormous retries happening for go routines [ a major retry bug ].

After few days , the dev team fixed the bug to exponentially
