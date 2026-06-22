1) How sudden network blips impacts working applications in an Enterprise  ?
2) What exactly does observability mean in  a critical workflow involving many services ?
3) Cost of Human Errors in production 
4) What does Major Incidents tell us about knowledge gaps ?
5) Why do incidents response or resolution time matters for production ?
   


**How sudden network blips impacts working applications in an Enterprise  ?**

*A Java application running on multiple VMs started throwing errors as "cannot UPSERT on a read only DB" , from Postgres server.  In simple terms , an application is trying to send a DB request to a server which is not a primary that can accepts writes out of a 3 node DB cluster.* 

**15min later when the alerts continues  to slack with more than 10K failures ....!** 

Support team opened a bridge , page Database Operations to get the reason for sudden error , resolution needed to solve it . 

* Application team agrees that same errors were 

DB team validates health of the cluster , confirms everything looking good and no changes at Database end . Application team asks multiple questions "what causes this error", "why it is happening without any change" and "why it continues to happen" so on....

DB team confirms open connections in the secondary/remote server of the cluster which should be the reason for the error . Requested by application team , DB team tries to clear the connections on remote server which keeps piling up again ..

The question still remains unanswered **" Why all of a sudden application is trying to connect to remote node of the cluster , not active master "**

**1 hour later** 
Debugging continues , more teams on to the call including LoadBalancer , Infrastructure Operations and downstreams that are impacted .... 

Loadbalancer team identifies single pool member of the VIP being active since many days while the other 2 nodes are inactive . People on the call raises questions against the behaviour , slack updates goes as "Pool members of VIP down for DB clusters" impacting multiple application servers doing DB updates.

DB team gets paged for similar bridges reporting similar DB errors that are impacts other apps and confirms BAU after sometime . No action from any team , apps are back BAU ... 

**An Enterprise level Issue for DB servers has been confirmed  with no root cause ** 

50-100 people on call, having cross questions to different teams , validating various things but nothing really concludes.... 

**2 hours since the start of the issue ...Application is still bleeding with same errors crossing 3M Errors ** 

DB team manually reviews logs at server level , matches the timeframe when issue started to an observation stating "Postgres process shutting down" log  on multiple nodes at same time . Issue concluded as "something happened at server level to bring down all database nodes at same time"

Someone from Engineering team suggests to restart an instance of impacted application to see if the failures stop from that instance .... **Boom, it works perfectly fine after restart** 

**..................A resolution is found ... just restart the impacted application.................** 


**3 hours from the beginning ** 

Everyone takes some breath , application servers starts coming back to BAU . Leaderships gets updates as we are BAU from application end .. But , question of root cause still remains unanswered.... debugging on what caused this still continues from multiple teams, application support wants root cause from DB teams .. 






