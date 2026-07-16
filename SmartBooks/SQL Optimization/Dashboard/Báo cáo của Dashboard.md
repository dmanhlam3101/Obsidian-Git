# 1. /public-investment-project/b251dadb

```sql
  
--------------------------------------------------------------------------------  
-- 1. findB2515Vc  
--------------------------------------------------------------------------------  
SELECT  
    TRUNC(A."PostDate") AS "dDate",  
    SUM(A."LCAmount")   AS "value"  
FROM "glm_gl_books" A  
INNER JOIN "company" C ON A."CompanyID" = C."CompanyID"  
INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"  
LEFT JOIN (  
    SELECT * FROM (  
        SELECT PSI."ProjectID", PSI."StatusID", PSI."StatusValue",  
               ROW_NUMBER() OVER (PARTITION BY PSI."ProjectID"  
                   ORDER BY PSI."ProjectStatusDate" DESC, PSI."LineID" DESC) AS rn  
        FROM "project_status_item" PSI  
    ) t WHERE rn = 1  
) PSI ON A."ProjectID" = PSI."ProjectID"  
WHERE EXISTS (  
    SELECT 1 FROM "project_cate" PC  
    WHERE PC."ProjectID" = P."ProjectID" AND PC."CateNo" <> '999'  
)  
AND P."Level" = 1  
AND SUBSTR(A."AccountNo", 1, 3) IN ('008','009','006','007','012','011','005','Z13')  
AND A."PostType" = 1  
AND (A."InTransTypeID" NOT IN (32,33,34,35) OR A."InTransTypeID" IS NULL)  
AND A."PostDate" >= :fromDate  
AND A."PostDate" <= :toDate  
-- b2515WhereOra (nhánh mặc định)  
AND (PSI."StatusID" = 34 OR PSI."StatusID" IS NULL)  
GROUP BY TRUNC(A."PostDate")  
ORDER BY TRUNC(A."PostDate");  
  
  
--------------------------------------------------------------------------------  
-- 2. findB2515Klth  
--    Khác Vc: AccountNo 5 ký tự = '24121', KHÔNG lọc PostType  
--------------------------------------------------------------------------------  
SELECT  
    TRUNC(A."PostDate") AS "dDate",  
    SUM(A."LCAmount")   AS "value"  
FROM "glm_gl_books" A  
INNER JOIN "company" C ON A."CompanyID" = C."CompanyID"  
INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"  
LEFT JOIN (  
    SELECT * FROM (  
        SELECT PSI."ProjectID", PSI."StatusID", PSI."StatusValue",  
               ROW_NUMBER() OVER (PARTITION BY PSI."ProjectID"  
                   ORDER BY PSI."ProjectStatusDate" DESC, PSI."LineID" DESC) AS rn  
        FROM "project_status_item" PSI  
    ) t WHERE rn = 1  
) PSI ON A."ProjectID" = PSI."ProjectID"  
WHERE EXISTS (  
    SELECT 1 FROM "project_cate" PC  
    WHERE PC."ProjectID" = P."ProjectID" AND PC."CateNo" <> '999'  
)  
AND P."Level" = 1  
AND SUBSTR(A."AccountNo", 1, 5) = '24121'  
AND (A."InTransTypeID" NOT IN (32,33,34,35) OR A."InTransTypeID" IS NULL)  
AND A."PostDate" >= :fromDate  
AND A."PostDate" <= :toDate  
AND (PSI."StatusID" = 34 OR PSI."StatusID" IS NULL)  
GROUP BY TRUNC(A."PostDate")  
ORDER BY TRUNC(A."PostDate");  
  
  
--------------------------------------------------------------------------------  
-- 3. findB2515Gtgn  
--------------------------------------------------------------------------------  
SELECT  
    TRUNC(A."PostDate") AS "dDate",  
    SUM(ABS(  
        CASE  
            WHEN SUBSTR(A."AccountNo", 1, 3) IN ('007','006','008','009','011','012','Z13')  
                 AND A."PostType" = 2  
            THEN A."LCAmount"  
            WHEN SUBSTR(A."AccountNo", 1, 3) = '005'  
                 AND A."PostType" = 1  
            THEN A."LCAmount"  
            ELSE 0  
        END  
    )) AS "value"  
FROM "glm_gl_books" A  
INNER JOIN "company" C ON A."CompanyID" = C."CompanyID"  
INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"  
LEFT JOIN (  
    SELECT * FROM (  
        SELECT PSI."ProjectID", PSI."StatusID", PSI."StatusValue",  
               ROW_NUMBER() OVER (PARTITION BY PSI."ProjectID"  
                   ORDER BY PSI."ProjectStatusDate" DESC, PSI."LineID" DESC) AS rn  
        FROM "project_status_item" PSI  
    ) t WHERE rn = 1  
) PSI ON A."ProjectID" = PSI."ProjectID"  
WHERE EXISTS (  
    SELECT 1 FROM "project_cate" PC  
    WHERE PC."ProjectID" = P."ProjectID" AND PC."CateNo" <> '999'  
)  
AND P."Level" = 1  
AND SUBSTR(A."AccountNo", 1, 3) IN ('007','006','008','009','011','012','Z13','005')  
AND A."PostType" IN (1, 2)  
AND (A."InTransTypeID" NOT IN (32,33,34,35) OR A."InTransTypeID" IS NULL)  
AND A."PostDate" >= :fromDate  
AND A."PostDate" <= :toDate  
AND (PSI."StatusID" = 34 OR PSI."StatusID" IS NULL)  
GROUP BY TRUNC(A."PostDate")  
ORDER BY TRUNC(A."PostDate");  
  
  
--------------------------------------------------------------------------------  
-- 4. findB2515CarryLkvc  
--    Giống Vc nhưng không GROUP BY, khoảng ngày: :fromDateLk -> :dayBeforeFrom  
--    Để chạy riêng theo hiện trạng và sẽ không gộp sql này
--------------------------------------------------------------------------------  
SELECT SUM(A."LCAmount") AS "value"  
FROM "glm_gl_books" A  
INNER JOIN "company" C ON A."CompanyID" = C."CompanyID"  
INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"  
LEFT JOIN (  
    SELECT * FROM (  
        SELECT PSI."ProjectID", PSI."StatusID", PSI."StatusValue",  
               ROW_NUMBER() OVER (PARTITION BY PSI."ProjectID"  
                   ORDER BY PSI."ProjectStatusDate" DESC, PSI."LineID" DESC) AS rn  
        FROM "project_status_item" PSI  
    ) t WHERE rn = 1  
) PSI ON A."ProjectID" = PSI."ProjectID"  
WHERE EXISTS (  
    SELECT 1 FROM "project_cate" PC  
    WHERE PC."ProjectID" = P."ProjectID" AND PC."CateNo" <> '999'  
)  
AND P."Level" = 1  
AND SUBSTR(A."AccountNo", 1, 3) IN ('008','009','006','007','012','011','005','Z13')  
AND A."PostType" = 1  
AND (A."InTransTypeID" NOT IN (32,33,34,35) OR A."InTransTypeID" IS NULL)  
AND A."PostDate" >= :fromDateLk  
AND A."PostDate" <= :dayBeforeFrom  
AND (PSI."StatusID" = 34 OR PSI."StatusID" IS NULL);  
  
  
--------------------------------------------------------------------------------  
-- 5. findB2515Lkvc  
--    SQL giống hệt findB2515Vc — chỉ khác cách cộng dồn ở tầng service.  
--------------------------------------------------------------------------------  
SELECT  
    TRUNC(A."PostDate") AS "dDate",  
    SUM(A."LCAmount")   AS "value"  
FROM "glm_gl_books" A  
INNER JOIN "company" C ON A."CompanyID" = C."CompanyID"  
INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"  
LEFT JOIN (  
    SELECT * FROM (  
        SELECT PSI."ProjectID", PSI."StatusID", PSI."StatusValue",  
               ROW_NUMBER() OVER (PARTITION BY PSI."ProjectID"  
                   ORDER BY PSI."ProjectStatusDate" DESC, PSI."LineID" DESC) AS rn  
        FROM "project_status_item" PSI  
    ) t WHERE rn = 1  
) PSI ON A."ProjectID" = PSI."ProjectID"  
WHERE EXISTS (  
    SELECT 1 FROM "project_cate" PC  
    WHERE PC."ProjectID" = P."ProjectID" AND PC."CateNo" <> '999'  
)  
AND P."Level" = 1  
AND SUBSTR(A."AccountNo", 1, 3) IN ('008','009','006','007','012','011','005','Z13')  
AND A."PostType" = 1  
AND (A."InTransTypeID" NOT IN (32,33,34,35) OR A."InTransTypeID" IS NULL)  
AND A."PostDate" >= :fromDate  
AND A."PostDate" <= :toDate  
AND (PSI."StatusID" = 34 OR PSI."StatusID" IS NULL)  
GROUP BY TRUNC(A."PostDate")  
ORDER BY TRUNC(A."PostDate");
```

>Hiện trạng trong code class `GetB251DADBUseCase` -> metho `buildB2515()` đang call xuống db 4 lần để lấy ra data của:
>- B2515 VC — vốn cấp per period.
>- B2515 KLTH — khối lượng thực hiện per period.
>- B2515 GTGN — giá trị giải ngân per period.
>- B2515 LKVC — luỹ kế vốn cấp per period.
>
>`Giải pháp`: gộp trung vào 1 query call 1 lần lấy ra data của 4 loại trên:
>

```sql
SELECT  
    ${selectDate} AS "dDate",  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('008','009','006','007','012','011','005','Z13')  
              AND A."PostType" = 1  
             THEN A."LCAmount" ELSE 0 END)                     AS "vc",  
  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,5) = '24121'  
             THEN A."LCAmount" ELSE 0 END)                     AS "klth",  
  
    SUM(ABS(CASE  
              WHEN SUBSTR(A."AccountNo",1,3) IN ('007','006','008','009','011','012','Z13')  
                   AND A."PostType" = 2  
              THEN A."LCAmount"  
              WHEN SUBSTR(A."AccountNo",1,3) = '005' AND A."PostType" = 1  
              THEN A."LCAmount"  
              ELSE 0  
            END))                                              AS "gtgn",  
  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('008','009','006','007','012','011','005','Z13')  
              AND A."PostType" = 1  
             THEN A."LCAmount" ELSE 0 END)                     AS "lkvc"  -- Lưu ý: đang thấy sql giống hệt "vc"
  
FROM "glm_gl_books" A  
INNER JOIN "company" C ON A."CompanyID" = C."CompanyID"  
INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"  
LEFT JOIN (  
    SELECT * FROM (  
        SELECT PSI."ProjectID", PSI."StatusID", PSI."StatusValue",  
            ROW_NUMBER() OVER (PARTITION BY PSI."ProjectID"  
                ORDER BY PSI."ProjectStatusDate" DESC, PSI."LineID" DESC) AS rn  
        FROM "project_status_item" PSI  
    ) t WHERE rn = 1  
) PSI ON A."ProjectID" = PSI."ProjectID"  
WHERE EXISTS (  
    SELECT 1 FROM "project_cate" PC  
    WHERE PC."ProjectID" = P."ProjectID" AND PC."CateNo"<> '999'  
)  
AND P."Level" = 1  
AND (A."InTransTypeID" NOT IN (32,33,34,35) OR A."InTransTypeID" IS NULL)  
AND A."PostDate" >= :fromDate  
AND A."PostDate" <= :toDate  
GROUP BY ${selectDate}  
ORDER BY ${selectDate}
```

## **Code trong UseCase Sample**
Trước:
```
	List<B2515RawDto> vcRows   = mapper.findB2515Vc(filter, selectDate);
	List<B2515RawDto> klthRows = mapper.findB2515Klth(filter, selectDate);
	List<B2515RawDto> gtgnRows = mapper.findB2515Gtgn(filter, selectDate);
	List<B2515RawDto> lkvcRows = mapper.findB2515Lkvc(filter, selectDate);
```
sau

```
	List<B2515CombinedRawDto> rows = mapper.findB2515Combined(filter, selectDate);
	Map<String, B2515CombinedRawDto> byDate = rows.stream()
	        .collect(Collectors.toMap(B2515CombinedRawDto::getDDate, r -> r));
```