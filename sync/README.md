### Cluster to Cluster Sync
클러스터와 클러스터간 데이터 동기화입니다.
2개의 MongoDB 를 이용하여 Active-inActive 형태로 구동되며 두개 모두 MongoDB6.0 이상이여야 합니다.   
Replica Set, Shard 모두 지원 하며 현재 Capped Collection, TimeSeries Collection은 지원 되지 않습니다.   
MongoDB Sync 는 ReadConcern, WriteConcern 모두 majority를 사용합니다.    

##### 2개의 Cluter 준비
MongoDB 와 Atlas 를 한개씩 준비 합니다.   

온프렘에 리플리카 셋을 준비 합니다.
````
$ mongosh mongodb://localhost:27101/admin?readPreference=secondary --username admin
Enter password: *********
Current Mongosh Log ID: 62c04b22327ba6343cef0220
Connecting to:          mongodb://<credentials>@localhost:27001/admin?readPreference=secondary&directConnection=true&appName=mongosh+1.5.0
Using MongoDB:          6.0.0-rc10
Using Mongosh:          1.5.0

For mongosh info see: https://docs.mongodb.com/mongodb-shell/

------
   The server generated these startup warnings when booting
   2022-07-02T13:40:27.181+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
   2022-07-02T13:40:28.735+00:00: vm.max_map_count is too low
------

Enterprise replica [direct: primary] admin> show databases
admin                                172.00 KiB
config                               228.00 KiB
idx                                  860.00 KiB
local                                128.72 MiB
Enterprise replica [direct: primary] admin> use idx
switched to db idx
Enterprise replica [direct: primary] idx> show tables
scores
Enterprise replica [direct: primary] idx> db.scores.find().count()
10000
````
동기화를 위한 계정을 준비 하여 줍니다.  

Atlas 에 동기화를 위한 클러스터를 준비 합니다.    
<img src="/sync/images/image01.png" width="90%" height="90%">    

동기화를 위한 계정 및 Sync 프로세스가 접근 할 수 있도록 Network Access 를 추가 하여 줍니다.   
<img src="/sync/images/image02.png" width="90%" height="90%">    

동기화 계정의 경우 Atlas 의 계정은 반드시 AtlasAdmin권한을 가지고 있어야 합니다. 또한 Reverse 가 가능하기 위해서는 다음과 같은 custom role 을 추가 하여야 합니다.   
MongoDB 6 에는 다음과 두개의 Role 이 추가 되었습니다. (데이터 쓰기에 대한 Block을 제어 하기 위한 권한)   
<img src="/sync/images/image07.png" width="60%" height="60%">    

계정에 추가한 권한을 추가 하여 생성 하여 줍니다.    
<img src="/sync/images/image08.png" width="70%" height="70%">   

Sync가 구동될 컴퓨트에서 Atlas 클러스터에 접근 합니다. (Sync는 DNS Seed list 연결 주소 형식 mongodb+srv를 지원 하지 않음으로 표준 연결 주소로 접근 테스트 합니다.)

````
$ mongosh mongodb://cluster0-shard-00-00.uj****.mongodb.net:27017,cluster0-shard-00-01.uj****.mongodb.net:27017,cluster0-shard-00-02.uj****.mongodb.net:27017 --username admin --tls
Enter password: *********
Current Mongosh Log ID: 62c0565cf33d2d3268945081
Connecting to:          mongodb://<credentials>@cluster0-shard-00-00.uj****.mongodb.net:27017,cluster0-shard-00-01.uj****.mongodb.net:27017,cluster0-shard-00-02.uj****.mongodb.net:27017/?tls=true&appName=mongosh+1.5.0
Using MongoDB:          6.0.0-rc12
Using Mongosh:          1.5.0

For mongosh info see: https://docs.mongodb.com/mongodb-shell/

Atlas atlas-ka9jxe-shard-0 [primary] test> show databases
admin   272.00 KiB
config  224.00 KiB
local   532.00 KiB
Atlas atlas-ka9jxe-shard-0 [primary] test> 
````

##### Sync 설치 구동

Amazon EC2 에 설치하는 예이며 Linux 와 Ubuntu, windows, MacOS 등이 지원 됩니다.
우선 Sync 를 다운로드 합니다.

MongoDB Donwnload Center (https://www.mongodb.com/try/download/mongosync)

````
$ tar -zxvf mongosync-amazon2-x86_64-0.9.0.tgz
$ cd mongosync-amazon2-x86_64-0.9.0/
$ sudo cp ./bin/mongosync /usr/local/bin/
````

다음과 같이 Sync 를 구동 하여 줍니다.   
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/[defaultauthdb][?options]]    

````
$ mongosync -cluster0 mongodb://admin:****@ip-10-0-0-210.ap-northeast-2.compute.internal:27101,ip-10-0-0-210.ap-northeast-2.compute.internal:27102,ip-10-0-0-210.ap-northeast-2.compute.internal:27103 -cluster1 mongodb://admin:****@cluster0-shard-00-00.uj****.mongodb.net:27017,cluster0-shard-00-01.uj****.mongodb.net:27017,cluster0-shard-00-02.uj****.mongodb.net:27017/?tls=true --verbosity TRACE --logPath /data/sync

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)
````

Terminal 을 추가 오픈 하여 상태를 확인 합니다.   
````
$ curl localhost:27182/api/v1/progress -XGET
{"progress":{"state":"IDLE","canCommit":false,"info":null,"lagTimeSeconds":null,"collectionCopy":null,"directionMapping":null}}
````
현재는 IDLE 상태로 시작을 해주면 데이터 동기화가 진행 됩니다.  

##### Sync Start
cluster0, cluster1 2개 클러스터가 등록 되어 있으며 Cluster0 을 소스로 지정 하고 Cluster1을 목적지로 지정 합니다
다음은 Reverse를 테스트 해보기 위한 것으로 reversible, enableUserWirteBlock 을 설정하여 시작 한 것입니다.   
````
$ curl localhost:27182/api/v1/start -XPOST --data '{ "source":"cluster0","destination":"cluster1","reversible":true,"enableUserWriteBlocking":true}'
{"success":true}

````
단순한 동기화 테스트의 경우 다음으로 실행 하여 줍니다
````
$ curl localhost:27182/api/v1/start -XPOST --data '{ "source":"cluster0","destination":"cluster1"}'
{"success":true}

````

목적지인 Atlas 의 데이터를 확인 하면 다음과 같이 데이터가 생성 된 것을 확인 할 수 있습니다.
<img src="/sync/images/image03.png" width="90%" height="90%">    

cluster0 에서 데이터를 추가로 생성 하여 데이터를 확인 합니다.
````
Enterprise source [primary] idx> for (var idx = 0; idx < 10000; idx++) { var name = "" + Math.floor(Math.random() * 10000); var score = Math.floor(Math.random() * 100); db.scores.insertOne({ name: name, score: score }); }
{
  acknowledged: true,
  insertedId: ObjectId("62c068fe691f0445507a1750")
}
````

데이터 복제가 빠르게 진행 됩니다.   
<img src="/sync/images/image04.png" width="90%" height="90%"> 


##### Sync Pause
동기화를 멈춤은 다음 명령어로 진행 합니다.

````
$ curl localhost:27182/api/v1/pause -XPOST --data '{}'
{"success":true}
````

데이터를 추가로 생성 시켜 줍니다.

````
Enterprise source [primary] idx> for (var idx = 0; idx < 10000; idx++) { var name = "" + Math.floor(Math.random() * 10000); var score = Math.floor(Math.random() * 100); db.scores.insertOne({ name: name, score: score }); }
{
  acknowledged: true,
  insertedId: ObjectId("62c06a93691f0445507a3e60")
}

````

Atlas 의 데이터는 증가 되지 않는 것을 확인 할 수 있습니다.
<img src="/sync/images/image05.png" width="90%" height="90%"> 

##### Sync Resume
동기화를 재기동 하여 줍니다.   
````
$ curl localhost:27182/api/v1/resume -XPOST --data '{}'
{"success":true}[
````

Atlas 를 확인 하면 데이터가 바로 생성 되어 있는 것을 확인 할 수 잇습니다.
<img src="/sync/images/image06.png" width="90%" height="90%"> 


##### Sync Commit
최종 동기화가 완료 되는 시점에 수행 하여 줍니다. commit 은 동기화가 종료 되는 것으로 클러스터를 마이그레이션 하는 시점입니다. Commit 이후의 데이터는 동기화 되지 않음으로 주의 하여야 합니다. Commit 하기 전 사용자 애플리케이션을 중지 하여 추가 데이터 생성을 중단하고 Commit 실행 후 애플리케이션의 MongoDB 접근 주소를 Cluster1의 주소로 변경 하여 줍니다.   

````
$ curl localhost:27182/api/v1/commit -XPOST --data '{ }'
{"success":true}
$ curl localhost:27182/api/v1/progress -XGET
{"progress":{"state":"COMMITTED","canCommit":false,"info":"commit completed","lagTimeSeconds":0,"collectionCopy":{"estimatedTotalBytes":478928,"estimatedCopiedBytes":478928},"directionMapping":{"Source":"cluster0: ip-10-0-0-210.ap-northeast-2.compute.internal:27101,ip-10-0-0-210.ap-northeast-2.compute.internal:27102,ip-10-0-0-210.ap-northeast-2.compute.internal:27103","Destination":"cluster1: cluster0-shard-00-00.uj****.mongodb.net:27017,cluster0-shard-00-01.uj****.mongodb.net:27017,cluster0-shard-00-02.uj****.mongodb.net:27017"}}}
````
상태 정보가 COMMITTED 로 변경 된 것을 확인 할 수 있습니다.


##### Sync Reverse
동기화가 Commit 이 완료 되어 Cluster1이 주 데이터베이스로 전환이 완료 되어 있는 경우 이제 역방향으로 데이터 동기화가 필요 하게 됩니다. 역방향 동기화를 위한 명령어로 다음을 수행 하여 줍니다. (Cluster1 에서 Cluster0로 동기화가 진행 됨)   

````
$ curl localhost:27182/api/v1/reverse -XPOST --data '{ }'
{"success":true}
````

Atlas Cluster 에 접속 하여 데이터를 생성 합니다.    
````
Atlas atlas-ka9jxe-shard-0 [primary] idx> for (var idx = 0; idx < 10000; idx++) { var name = "" + Math.floor(Math.random() * 10000); var score = Math.floor(Math.random() * 100); db.scores.insertOne({ name: name, score: score }); }
{
  acknowledged: true,
  insertedId: ObjectId("62c0f74d49f86411a5cf49a9")
}
Atlas atlas-ka9jxe-shard-0 [primary] idx> db.scores.find({}).count()
40000
````

역방향 동기화 확인을 위해 Cluster0에 접속 하여 데이터를 확인 하여 보면 다음과 같이 데이터가 동기화 된것을 알 수 있습니다.   
````
Enterprise source [primary] idx> db.scores.find().count()
40000
````
