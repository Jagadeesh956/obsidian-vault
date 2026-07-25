


We have a in house platform called Enterprise Configuration Management that manages configuration for various micro services to update the consuming services dynamically whenever someone changes configuration in version control systems like git , bitbucket . 


As the platform exposes REST API endpoints for any microservice to consume the config it needs, the platform acts similar to spring cloud consul which requires minimal configuration changes in the properties file along with many other features. 

Since this platform has various consumers that run their services in different tech stack like Go, Java , Python and Node.js ..... the platform provider decided to provide SDKs for each of these programming languages for easy interaction when the dependencies modules are imported and implemented by the consumers abstracting the calls to platform and periodically updating the config if changes were detected . 


As part of providing a feature called SSE [ server sent events ] , instead of client SDKs periodically fetching config every 2min in the Go SDK, the developer pushed a change that infinitely creates a go client [ a routine ] to connect to the platform in case of failure with retries . 

As the product is released , after few days a consumer implemented the SSE and released to production on some X date , deployed the service into a K8s cluster provided by organization in private cloud  . 



On the other side of story, a major SRB [ service restoration bridge ] arised from a very critical platorm called safekey [ OTP sender ] built for amex client transaction verification reporting impacts intermittently by multiple customers .

The safekey team has no idea why it's happening since they haven't deployed any changes and started working with all dependent teams including the cloud operations , after few hours of debugging the cloud operations found that the istio-gateway pods are reporting liveness failures in a specific pattern matching to safekey failures  . As the gateway is shared by all the services deployed in that cluster , cloud operation teams are unable to identify what exactly is causing those failures. 

Time kept moving , many teams were involved including vendor for service mesh, OCP etc to help the cloud ops to identify the issue. Multiple leaders joined the call as it's impacting business . After 18+ hours of debugging , the cloud SRE team identified that the ECM platform service that is hosted on same cluster is seeing same pattern of crashloop backoff for the pods in the zones where safekey failures , 