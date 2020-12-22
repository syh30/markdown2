# Spatial data를 10,000배 잘 다루게 된 방법
[https://b.luavis.kr/server/how-to-deal-with-spatial-data](https://b.luavis.kr/server/how-to-deal-with-spatial-data)   



올해 3월에 물류중점 회사 메쉬코리아로 이직하고 개인 사정도 있어서 블로그 작성을 게을리 했는데, 최근에 성능 개선한 case는 블로그 글로 작성해둘 필요가 있다고 생각되고, 이 글외에도 사내에서 공유했던 글들에 대해서도 잘 정제해서 블로그 글로 작성해야다는 생각이 들었습니다.   

메쉬코리아는 음식, 화장품, 편의점 상품과 같은 상품을 실시간으로 배달하는 서비스, 부릉(Vroong)을 운영하는 회사입니다. 배달 주문을 처리하기 위해서는 상품을 판매하는 상점과 상품을 주문한 고객 사이의 거리와 장애물등을 고려하여 처리할 수 있는 처리할 수 있는 주문건인가 배달 기사님에게는 얼마만큼의 금액을 지불해야하고 상점에서는 얼마만큼의 금액을 받아내야하는가를 판단하기 위해서 spatial data를 이용해 각 상점의 배송 가능한 권역들 배달이 불가능한 권역들을 관리하는 권역 시스템을 운영하고 있습니다.   

권역 시스템에서는 spatial data를 효율적으로 다루기 위해서 PostgreSQL에서 지원하는 PostGIS를 이용하고 있습니다. 여러가지 성능 문제가 있었지만 대부분의 경우는 가벼운 문제들이었습니다. 다만 지도 화면에서 보이는 영역과 행정동/법정동 polygon의 intersects를 계산하고 polygon을 가져오는 문제는 큰 성능 이슈를 갖고 있었습니다. worst case에서는 조회시간이 60초 정도 걸렸습니다. 문제가 되는 성능이 떨어지는 쿼리는 아래와 같았습니다.   

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

이 글은 이 문제를 어떻게 해결할 수 있는가 방법에 대한 과정과 최대 1만배 성능을 튜닝한 이야기입니다.  

### Spatial data의 intersects의 계산 방법   
      
문제 상황에서 사용하는 spatial data는 2d의 polygon 입니다. 그리고 intersects는 두개의 polygon이 겹치는지를 판단하는 연산을 의미합니다. 이를 postgresql에서는 아래와 같이 사용합니다.    

<pre>
<code>
SELECT * FROM region WHERE region
AND st_intersects(
    region.polygon,
    ST_MakeEnvelope(126.9373539, 37.5210172, 127.0540836, 37.5873607, 4326)
);
</code>
</pre>
**[쿼리2]**    

위 쿼리는 region 테이블의 polygon 컬럼과 서울 종로구 부근의 영역을 비교하여 겹치는 영역이 있는가를 조건으로 select 합니다. 약 천만개의 row가 있는 region 테이블에서도 0.5초내외의 성능을 보여주면서 의외로 빠른 성능을 보여줍니다. intersects 연산이 단순하게 생각하면 무거운 연산일거라 추측할 수 있습니다. 이 쿼리가 어떻게 빠르게 동작할 수 있는것인지 의문이 드는데, 이게 가능한 방법이 있습니다. R-Tree라는 자료구조를 사용한 index 구조를 통해 계산하면 intersects 계산을 빠르게 할 수 있다고 합니다.    

R-tree의 인덱싱 방법은 B-tree와 유사한데, polygon 데이터들의 최소한의 bounding box를 구하고 그 박스간의 포함관계를 B-tree의 형식으로 관리하는 index tree입니다. 이렇게하면 B-tree의 범위 조회와 같은 계산 방법으로 계산하면 포함관계와 교차관계(intersects)를 쉽게 계산할 수 있습니다. index search를 통해서 대충 intersects하는 polygon을 bounding box가 아닌 실제 polygon과의 intersects 연산을 한번 더 진행해서 진짜 원하는 polygon이 맞는지 한번 더 선별과정을 거칩니다.    
