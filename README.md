# Spatial data를 10,000배 잘 다루게 된 방법
[https://b.luavis.kr/server/how-to-deal-with-spatial-data](https://b.luavis.kr/server/how-to-deal-with-spatial-data)   



올해 3월에 물류중점 회사 메쉬코리아로 이직하고 개인 사정도 있어서 블로그 작성을 게을리 했는데, 최근에 성능 개선한 case는 블로그 글로 작성해둘 필요가 있다고 생각되고, 이 글외에도 사내에서 공유했던 글들에 대해서도 잘 정제해서 블로그 글로 작성해야다는 생각이 들었습니다.   

메쉬코리아는 음식, 화장품, 편의점 상품과 같은 상품을 실시간으로 배달하는 서비스, 부릉(Vroong)을 운영하는 회사입니다. 배달 주문을 처리하기 위해서는 상품을 판매하는 상점과 상품을 주문한 고객 사이의 거리와 장애물등을 고려하여 처리할 수 있는 처리할 수 있는 주문건인가 배달 기사님에게는 얼마만큼의 금액을 지불해야하고 상점에서는 얼마만큼의 금액을 받아내야하는가를 판단하기 위해서 spatial data를 이용해 각 상점의 배송 가능한 권역들 배달이 불가능한 권역들을 관리하는 권역 시스템을 운영하고 있습니다.   

권역 시스템에서는 spatial data를 효율적으로 다루기 위해서 PostgreSQL에서 지원하는 PostGIS를 이용하고 있습니다. 여러가지 성능 문제가 있었지만 대부분의 경우는 가벼운 문제들이었습니다. 다만 **지도 화면에서 보이는 영역과 행정동/법정동 polygon의 intersects를 계산하고 polygon**을 가져오는 문제는 큰 성능 이슈를 갖고 있었습니다. worst case에서는 조회시간이 60초 정도 걸렸습니다. 문제가 되는 성능이 떨어지는 쿼리는 아래와 같았습니다.   

<pre>
<code>
SELECT region.*
FROM region
INNER JOIN address_polygon on region.pk=address_polygon.region_pk
INNER JOIN address on address_polygon.address_pk=address.pk
WHERE address.address_type = 'LEGAL'
AND region.region_type = 'ADDRESS_REGION'
AND region.status = 'ENABLED'
AND st_intersects(
    region.polygon,
    ST_MakeEnvelope(126.9373539, 37.5210172, 127.0540836, 37.5873607, 4326)
);
</code>
</pre>
**[쿼리1]**  

이 글은 이 문제를 어떻게 해결할 수 있는가 방법에 대한 과정과 최대 **1만배** 성능을 튜닝한 이야기입니다.  

### Spatial data의 intersects의 계산 방법   
      
문제 상황에서 사용하는 spatial data는 2d의 polygon 입니다. 그리고 intersects는 두개의 polygon이 겹치는지를 판단하는 연산을 의미합니다. 이를 postgresql에서는 아래와 같이 사용합니다.    

```java
SELECT * FROM region WHERE region
AND st_intersects(
    region.polygon,
    ST_MakeEnvelope(126.9373539, 37.5210172, 127.0540836, 37.5873607, 4326)
);
```
**[쿼리2]**    

위 쿼리는 region 테이블의 polygon 컬럼과 서울 종로구 부근의 영역을 비교하여 겹치는 영역이 있는가를 조건으로 select 합니다. 약 천만개의 row가 있는 region 테이블에서도 0.5초내외의 성능을 보여주면서 의외로 빠른 성능을 보여줍니다. intersects 연산이 단순하게 생각하면 무거운 연산일거라 추측할 수 있습니다. 이 쿼리가 어떻게 빠르게 동작할 수 있는것인지 의문이 드는데, 이게 가능한 방법이 있습니다. [R-Tree](https://en.wikipedia.org/wiki/R-tree)라는 자료구조를 사용한 index 구조를 통해 계산하면 intersects 계산을 빠르게 할 수 있다고 합니다.    

R-tree의 인덱싱 방법은 B-tree와 유사한데, polygon 데이터들의 최소한의 bounding box를 구하고 그 박스간의 포함관계를 B-tree의 형식으로 관리하는 index tree입니다. 이렇게하면 B-tree의 범위 조회와 같은 계산 방법으로 계산하면 포함관계와 교차관계(intersects)를 쉽게 계산할 수 있습니다. index search를 통해서 대충 intersects하는 polygon을 bounding box가 아닌 실제 polygon과의 intersects 연산을 한번 더 진행해서 진짜 원하는 polygon이 맞는지 한번 더 선별과정을 거칩니다.     
![code](/img/oskar-yildiz-cOkpTiJMGzA-unsplash.jpg) 
<img src="/img/oskar-yildiz-cOkpTiJMGzA-unsplash.jpg" width="450px" height="400px" title="px(픽셀) 크기 설정" alt="code"></img><br/>

<p align="center">
  <img src="/img/oskar-yildiz-cOkpTiJMGzA-unsplash.jpg" width="450px" height="400px" title="px(픽셀) 크기 설정" alt="code"></img><br/>
  [사진이다]
</p>

이 권역 서비스는 처음부터 프로젝트에 involve된게 아니었기 때문에 위의 문제의 쿼리1을 만났을때는 쿼리2를 테스트 해보지 않았습니다. 따라서 천만개의 row에 대해서 intersects를 모두 계산한다고 생각해서 어떻게 해도 빠르게 할 수 있는 방법이 없다고 생각했습니다. 따라서 모든 폴리곤을 메모리에 올리고 최적화된 알고리즘을 구현해야겠다는 생각하면서 찾아보다가 R-tree에 대해서 알게되었습니다. R-tree 성능에 대해서 Python으로 구현해보고 충분히 빠른것을 확인했습니다. Python에서는 R-tree의 구현을 ctypes를 이용해 libspatialindex를 래핑한 [라이브러리](https://pypi.org/project/Rtree/)를 찾을 수 있었습니다.    

그래서 PostGIS는 이런거 지원 안하나 싶어서 찾아보게 되었는데 PostGIS도 R-tree와 같은 bounding box indexing을 지원한다는 것을 알게되고 심지어 해당 인덱싱이 위의 쿼리에 `polygon` 컬럼에 적용되어 있는것을 확인했습니다.    

정확히는 PostGIS는 R-Tree라는 이름으로 사용하지 않고 [GIST](https://postgis.net/workshops/postgis-intro/indexing.html) 라는 이름으로 index 방법을 제공하고 있는것을 알게되었습니다. 그리고 위 GIST의 링크를 읽어보면 나와 있는데, bounding box간의 연산을 위해서 postgresql에서는 [특수한 연산자](https://postgis.net/docs/reference.html#idm9871)를 따로 지원하고 있습니다. 또한 st_intersects는 이미 내부에서 `&&` 연산자를 통해서 bounding box간의 연산을 하고 `AND` 조건으로 실제 polygon이 intersects 하는 가 판단한다는 것을 알게되었습니다. 이 사실을 알고 위 쿼리 2를 작성해서 쿼리해보고 0.5초의 성능이면 1분에 비해서는 성능이 엄청나게 좋은 성능이 나온다는것을 알게되어서 어떻게 이정도의 차이를 갖게 되는지 좀 더 정확히 알아보고자 `explain analyze`를 해보게 되었습니다.    

```xml
Index Scan using polygon_geom_idx on region  (cost=0.42..513509.60 rows=... width=...) (actual time=0.136..515.878 rows=... loops=1)
    Index Cond: (polygon && '...'::geometry)
    Filter: _st_intersects(polygon, '...'::geometry)
    Rows Removed by Filter: 383
Planning Time: 0.244 ms
Execution Time: 522.464 ms
```
위 explain 결과를 보면 index condition으로 `polygon column` 과 `&&` 연산을 통해서 bounding box를 주어진 polygon 데이터와 비교하고 후에 filter 조건으로 _st_intersects를 사용하는것을 확인할 수 있습니다.    

### 그러면 문제는 어디서 발생하는걸까?    

그러면 문제가 된 쿼리 1은 도대체 무슨 문제가 있었던 것인지 싶어서 해당 쿼리도 `explain analyze`를 돌려보았습니다.     
```c++
Gather  (cost=1080.01..67698.36 rows=... width=1894) (actual time=117.101..66590.220 rows=... loops=1)
  Workers Planned: 1
  Workers Launched: 1
  ->  Nested Loop  (cost=80.01..66697.06 rows=8 width=1894) (actual time=66.379..65641.273 rows=134 loops=2)
        ->  Nested Loop  (cost=79.58..51221.10 rows=1996 width=8) (actual time=63.748..5975.698 rows=401872 loops=2)
              ->  Parallel Bitmap Heap Scan on address  (cost=79.14..10717.49 rows=1714 width=8) (actual time=57.163..61.659 rows=2526 loops=2)
                    Recheck Cond: ((address_type)::text = 'LEGAL'::text)
                    Heap Blocks: exact=107
                    ->  Bitmap Index Scan on address_address_type_index  (cost=0.00..78.42 rows=2914 width=0) (actual time=100.710..100.710 rows=5504 loops=1)
                          Index Cond: ((address_type)::text = 'LEGAL'::text)
              ->  Index Only Scan using address_polygon_pkey on address_polygon  (cost=0.43..23.23 rows=40 width=16) (actual time=0.579..2.307 rows=159 loops=5052)
                    Index Cond: (address_pk = address.pk)
                    Heap Fetches: 304308
        ->  Index Scan using region_primary_key on region  (cost=0.43..7.75 rows=1 width=1894) (actual time=0.147..0.147 rows=0 loops=803744)
              Index Cond: (pk = address_polygon.region_pk)
              Filter: ((optimized_polygon_30 && '...'::geometry) AND ((region_type)::text = 'ADDRESS_REGION'::text) AND ((status)::text = 'ENABLED'::text) AND _st_intersects(optimized_polygon_30, '...'::geometry)
              Rows Removed by Filter: 1
Planning Time: 3.403 ms
Execution Time: 66590.472 ms
```
> it is not always faster to do an index search: if the search is going to return every record in the table, traversing the index tree to get each record will actually be slower than just sequentially reading the whole table from the start.    

postgresql이 경우에 따라서 index를 타는 것이 비효율적이라 판단되면 풀서치하는 방향으로 optimize하는걸 알게되었습니다. 어떤 기준으로 하는지가 불명확하고 vacuuming 해도 index를 사용할때도 있고 안할때도 있어서 단순히 vacumming으로는 일관적인 성능을 보장하기는 어렵다고 판단했습니다.    

```json
CREATE MATERIALIZED VIEW address_region_view AS
SELECT ap.region_pk, region.polygon, ap.address_type FROM (
    SELECT address_polygon.region_pk, address.address_type FROM address
    LEFT JOIN address_polygon ON address.pk = address_polygon.address_pk
    WHERE address.address_type IN ('LEGAL', ...)
) ap
LEFT JOIN region ON region.pk = ap.region_pk
WHERE region.region_type = 'ADDRESS_REGION'
AND region.status = 'ENABLED';

CREATE INDEX address_region_view_idx ON address_region_view using gist(polygon);
CREATE INDEX address_region_view_address_type_idx ON address_region_view (address_type);
```
[내부링크](#10)
