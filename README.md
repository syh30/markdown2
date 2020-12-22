# Spatial data를 10,000배 잘 다루게 된 방법
[https://b.luavis.kr/server/how-to-deal-with-spatial-data](https://b.luavis.kr/server/how-to-deal-with-spatial-data)   



올해 3월에 물류중점 회사 메쉬코리아로 이직하고 개인 사정도 있어서 블로그 작성을 게을리 했는데, 최근에 성능 개선한 case는 블로그 글로 작성해둘 필요가 있다고 생각되고, 이 글외에도 사내에서 공유했던 글들에 대해서도 잘 정제해서 블로그 글로 작성해야다는 생각이 들었습니다.   

메쉬코리아는 음식, 화장품, 편의점 상품과 같은 상품을 실시간으로 배달하는 서비스, 부릉(Vroong)을 운영하는 회사입니다. 배달 주문을 처리하기 위해서는 상품을 판매하는 상점과 상품을 주문한 고객 사이의 거리와 장애물등을 고려하여 처리할 수 있는 처리할 수 있는 주문건인가 배달 기사님에게는 얼마만큼의 금액을 지불해야하고 상점에서는 얼마만큼의 금액을 받아내야하는가를 판단하기 위해서 spatial data를 이용해 각 상점의 배송 가능한 권역들 배달이 불가능한 권역들을 관리하는 권역 시스템을 운영하고 있습니다.   

권역 시스템에서는 spatial data를 효율적으로 다루기 위해서 PostgreSQL에서 지원하는 PostGIS를 이용하고 있습니다. 여러가지 성능 문제가 있었지만 대부분의 경우는 가벼운 문제들이었습니다. 다만 지도 화면에서 보이는 영역과 행정동/법정동 polygon의 intersects를 계산하고 polygon을 가져오는 문제는 큰 성능 이슈를 갖고 있었습니다. worst case에서는 조회시간이 60초 정도 걸렸습니다. 문제가 되는 성능이 떨어지는 쿼리는 아래와 같았습니다.   

<pre>
<code>
<span style="color:green">SELECT</span> region.*
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
