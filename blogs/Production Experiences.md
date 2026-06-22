1) How maintenance impacts working applications in an Enterprise in an onprem datacenter ?
2) What exactly does observability mean in  a critical workflow involving many services ?
3) Cost of Human Errors in production 
4) What does Major Incidents tell us about knowledge gaps ?
5) Why do incidents response or resolution time matters for production ?
   


**How maintenance impacts working applications in an Enterprise in an on prem datacenter ?**

*A Java application running on multiple VMs started throwing errors as "cannot UPSERT on a read only DB" , from Postgres server.  In simple terms , an application is trying to send a DB request to a server which is not a primary that can accepts writes .* 

**15min later when the alert triggered to slack ....!** 

Support team opened a bridge , page Database Operations to get the reason for sudden error , resolution needed to solve it . 

DB team validates health of the cluster , confirms everything looking good and no changes at Database end . Application team asks multiple questions "what causes this error ", "why it is happening without any change" and so on....

Someone asks DB team to see where the connections are going to , while 


