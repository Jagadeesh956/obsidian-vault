1) How sudden network blips impacts working applications in an Enterprise  ?
2) What exactly does observability mean in  a critical workflow involving many services ?
3) Cost of Human Errors in production 
4) What does Major Incidents tell us about knowledge gaps ?
5) Why do incidents response or resolution time matters for production ?
   


**How sudden network blips impacts working applications in an Enterprise  ?**

*A Java application running on multiple VMs started throwing errors as "cannot UPSERT on a read only DB" , from Postgres server.  In simple terms , an application is trying to send a DB request to a server which is not a primary that can accepts writes .* 

**15min later when the alert triggered to slack ....!** 

Support team opened a bridge , page Database Operations to get the reason for sudden error , resolution needed to solve it . 

DB team validates health of the cluster , confirms everything looking good and no changes at Database end . Application team asks multiple questions "what causes this error ", "why it is happening without any change" and so on....

DB team confirms open connections in the secondary/remote server of the cluster which should be the reason for the error . Requested by application team , DB team tries to clear the connections on remote server which keeps piling up again ..

The question still remains unanswered **" Why all of a sudden application is trying to connect to remote node of the cluster , not active master "**






