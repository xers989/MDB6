## Relational Database Migrator
운영 중인 Relational Database를 migration 하기 위한 솔루션으로 스키마 전환 및 데이터 migration을 클러스터와 클러스터간 데이터 동기화입니다.


### Relationaal Database Migrator 설치
다음 사이트에서 프로그램을 다운로드 후 설치 합니다.

### Relational Database Source 선택
Relational Migrator를 실행 하면 브라우저가 오픈 되고 다음주소로 접속 하게 됩니다.    
<img src="/relationmigrator/images/images02.png" width="90%" height="90%">     

현재 버전 v.1.0.28 기준 데이터베이스로 부터 마이그레이션과 schemafile 을 이용한 마이그레이션이 지원 됩니다.   
준비된 데이터베이스에 접속하여 마이그레이션을 진행 할 것임으로 Connect live database를 선택 합니다.

연결할 데이터베이스 정보를 입력 하여 줍니다. 지원 되는 데이터베이스는 Oracle, SQL Server, MySQL, PostgreSQL등이며 Oracle, SQL Server 의 경우 JDBC Driver 설치가 필요 합니다.
<img src="/relationmigrator/images/images01.png" width="90%" height="90%">     


접근하기 위한 JDBC 주소와 계정 정보를 입력 하여 줍니다.
<img src="/relationmigrator/images/images03.png" width="90%" height="90%">     

Connect를 클릭 하면 데이터 베이스에 접근 하여 스키마를 분석하는 작업이 진행 됩니다.

### Schema 선택
접속한 데이터베이스의 정보와 테이블 정볼르 볼 수 있습니다. 마이그레이션으로 northwind를 진행 할 것임으로 이를 선택 하여 줍니다.

<img src="/relationmigrator/images/images04.png" width="90%" height="90%">     

선택 후 다음을 클릭 한 후 마이그레이션 프로젝트 이름을 지정 하여 줍니다.
<img src="/relationmigrator/images/images05.png" width="90%" height="90%">     

### Migration Schema 지정

분석된 스키마 정보중 orders를 선택 하면 order_details 와 1:N 관계로 지정 된 것을 볼 수 있습니다.
<img src="/relationmigrator/images/images06.png" width="90%" height="90%">     

Orders 와 order_details 관계를 하나의 Orders 컬렉션으로 표현 하여 줍니다. 이를 위해 Child 형태 테이블인 order_details 테이블을 선택 한 후 오른쪽 패널에서 Mapping 정보를 추가 하여 줍니다.
<img src="/relationmigrator/images/images07.png" width="90%" height="90%">    

1:N 관계 임으로 Array in parent documents 를 선택 하여 줍니다. 이는 parent 문서 (Order)에 order_details 가 복수개(array)로 표현 되는 것임니다.  이후 Parent collection을 선택 하고 Field 정보중 order_id 는 중복된 값으로 불필요 함으로 uncheck 하여 줍니다
<img src="/relationmigrator/images/images08.png" width="90%" height="90%"> 
이후 저장 하여 줍니다.

기본적으로 테이블은 하나의 컬렉션으로 매핑됩니다. order_details 의 경우 order 컬렉션의 하위 컬렉션으로 매핑하였음으로 필요 없는 매핑 정보는 삭제 하여 줍니다.
<img src="/relationmigrator/images/images09.png" width="90%" height="90%"> 


orders 와 매핑이 되는 정보로 products 테이블을 매핑 하여 줍니다. 이를 위해 products 테이블을 선택 하고 오른쪽 패널에서 Add를 클릭 합니다.
<img src="/relationmigrator/images/images10.png" width="90%" height="90%"> 

products는 상위 정보로 order_details 의 products_id 와 1:1 관계를 가져 가기 때문에 Fields in child document 를 선택 하고 Parent collection 을 orders를 선택 하여 줍니다. 이후 Root path 는 order_details.product로 하여 줍니다. (문서에 order_details의 하위로 생성 하여줍니다) 데이터의 경우 Product 정보 중 이름과 가격 정보만 order_details 에 보여 줄것임으로 이를 제외한 나머지 필드는 uncheck 하여 줍니다.
<img src="/relationmigrator/images/images11.png" width="90%" height="90%"> 
매핑 정보를 저장 하여 줍니다.

### Data Migration
데이터 마이그레이션 탭을 선택 후 Create Sync job 버튼을 클릭하여 줍니다.
<img src="/relationmigrator/images/images12.png" width="90%" height="90%"> 

마이그레이션 소스 정보를 입력 하여 줍니다.
<img src="/relationmigrator/images/images13.png" width="90%" height="90%"> 

대상 정보를 입력 하여 줍니다.
<img src="/relationmigrator/images/images15.png" width="90%" height="90%"> 

마이그레이션 방법을 선택 하여 주고 작업을 시작 하여 줍니다.
<img src="/relationmigrator/images/images16.png" width="90%" height="90%"> 


마이그레이션이 진행 됩니다.
<img src="/relationmigrator/images/images17.png" width="90%" height="90%"> 


작업이 완료 되면 complete로 상태가 변경 됩니다.
<img src="/relationmigrator/images/images18.png" width="90%" height="90%"> 

### Data 확인
MongoDB 에 접속 하여 마이그레이션 데이터를 확인 합니다.
<img src="/relationmigrator/images/images19.png" width="90%" height="90%"> 

