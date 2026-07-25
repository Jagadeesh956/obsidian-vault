


We have a in house platform called Enterprise Configuration Management that manages configuration for various micro services to update the consuming services dynamically whenever someone changes configuration in version control systems like git , bitbucket . 


As the platform exposes REST API endpoints for any microservice to consume the config it needs, the platform acts similar to spring cloud consul which requires minimal configuration changes in the properties file along with many other features. 

Since this platform has various consumers that run their services in different tech stack like Go, Java , Python and Node.js ..... the platform provider decided to provide SDKs 