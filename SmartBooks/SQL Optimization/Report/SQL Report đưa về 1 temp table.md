
```sql
  
SELECT "itemId","parentId","level","itemName","itemName1","itemName2",  
       SUM("i1") AS "i1", SUM("i2") AS "i2"  
FROM (  
    -- =================================================================  
    -- 1. Contracts: A#01#VendorNo#ContractNo    -- =================================================================    SELECT 'A#01#' || A."VendorNo" || '#' || A."ContractNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."VendorName") AS "itemName1", MAX(A."ContractName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "contract" CT ON A."ContractID" = CT."ContractID" AND CT."DocumentType" = 1  
    WHERE A."VendorNo" != '' AND A."ContractNo" != '' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."VendorNo", A."ContractNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 2. Clearance self-cost: A#02#ClearanceNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#02#' || A."ClearanceNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."CompanyName") AS "itemName1", MAX(A."ClearanceName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" != '' AND E."isSelfpccosts" = 1 AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."ClearanceNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 3. Clearance other-cost: A#03#ClearanceNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#03#' || A."ClearanceNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", MAX(A."ClearanceName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" != '' AND E."isSelfpccosts" != 1 AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."ClearanceNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 4a. Partner QLDA: A#04#1#PartnerNo    -- =================================================================    SELECT 'A#04#1#' || A."PartnerNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", 'Chi phí QLDA' AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."VendorNo" = '' AND A."ContractNo" = '' AND A."ClearanceNo" = '' AND A."PartnerNo" != ''  
      AND SUBSTR(A."ExpenseNo",1,6) = '119803' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."PartnerNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 4b. Partner other: A#04#2#PartnerNo#ExpenseNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#04#2#' || A."PartnerNo" || '#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", MAX(E."ExpenseName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."VendorNo" = '' AND A."ContractNo" = '' AND A."ClearanceNo" = '' AND A."PartnerNo" != ''  
      AND SUBSTR(A."ExpenseNo",1,6) != '119803' AND SUBSTR(A."ExpenseNo",1,3) = '201'  
      AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."PartnerNo", A."ExpenseNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 5a. Unidentified QLDA: A#05#1#ExpenseNo    -- =================================================================    SELECT 'A#05#1#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           'Đối tượng chưa xác định' AS "itemName1", 'Chi phí quản lý dự án' AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" = '' AND A."ContractNo" = '' AND A."VendorNo" = '' AND A."PartnerNo" = ''  
      AND SUBSTR(A."ExpenseNo",1,6) = '119803' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."ExpenseNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 5b. Unidentified other: A#05#2#ExpenseNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#05#2#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           'Đối tượng chưa xác định' AS "itemName1", MAX(E."ExpenseName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" = '' AND A."ContractNo" = '' AND A."VendorNo" = '' AND A."PartnerNo" = ''  
      AND SUBSTR(A."ExpenseNo",1,6) != '119803' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
    GROUP BY A."ExpenseNo"  
) agg  
GROUP BY "itemId","parentId","level","itemName","itemName1","itemName2"  
ORDER BY "itemId";
```

SQL sau khi gộp về 1 temp và join temp đó 

```sql
  
WITH base AS (  
    SELECT /*+ MATERIALIZE */  
           A."VendorNo",A."AccountNo", A."ContractNo", A."ClearanceNo", A."PartnerNo", A."ExpenseNo",  
           A."VendorName", A."ContractName", A."CompanyName", A."ClearanceName", A."PartnerName",  
           A."PostType",A."LCAmount",A."ContractID",A."PostDate",A."InTransTypeID",A."ExpenseID"  
    FROM "glm_gl_books" A  
    WHERE A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ( SUBSTR(A."AccountNo",1,4)='2412'  
         OR (SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1) )  
)  
  
SELECT "itemId","parentId","level","itemName","itemName1","itemName2",  
       SUM("i1") AS "i1", SUM("i2") AS "i2"  
FROM (  
    -- =================================================================  
    -- 1. Contracts: A#01#VendorNo#ContractNo    -- =================================================================    SELECT 'A#01#' || A."VendorNo" || '#' || A."ContractNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."VendorName") AS "itemName1", MAX(A."ContractName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "contract" CT ON A."ContractID" = CT."ContractID" AND CT."DocumentType" = 1  
    WHERE A."VendorNo" != '' AND A."ContractNo" != ''  
    GROUP BY A."VendorNo", A."ContractNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 2. Clearance self-cost: A#02#ClearanceNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#02#' || A."ClearanceNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."CompanyName") AS "itemName1", MAX(A."ClearanceName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" != '' AND E."isSelfpccosts" = 1  
    GROUP BY A."ClearanceNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 3. Clearance other-cost: A#03#ClearanceNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#03#' || A."ClearanceNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", MAX(A."ClearanceName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" != '' AND E."isSelfpccosts" != 1  
    GROUP BY A."ClearanceNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 4a. Partner QLDA: A#04#1#PartnerNo    -- =================================================================    SELECT 'A#04#1#' || A."PartnerNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", 'Chi phí QLDA' AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."VendorNo" = '' AND A."ContractNo" = '' AND A."ClearanceNo" = '' AND A."PartnerNo" != ''  
      AND SUBSTR(A."ExpenseNo",1,6) = '119803'  
    GROUP BY A."PartnerNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 4b. Partner other: A#04#2#PartnerNo#ExpenseNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#04#2#' || A."PartnerNo" || '#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", MAX(E."ExpenseName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."VendorNo" = '' AND A."ContractNo" = '' AND A."ClearanceNo" = '' AND A."PartnerNo" != ''  
      AND SUBSTR(A."ExpenseNo",1,6) != '119803' AND SUBSTR(A."ExpenseNo",1,3) = '201'  
    GROUP BY A."PartnerNo", A."ExpenseNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 5a. Unidentified QLDA: A#05#1#ExpenseNo    -- =================================================================    SELECT 'A#05#1#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           'Đối tượng chưa xác định' AS "itemName1", 'Chi phí quản lý dự án' AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" = '' AND A."ContractNo" = '' AND A."VendorNo" = '' AND A."PartnerNo" = ''  
      AND SUBSTR(A."ExpenseNo",1,6) = '119803'  
    GROUP BY A."ExpenseNo"  
  
    UNION ALL  
  
    -- =================================================================  
    -- 5b. Unidentified other: A#05#2#ExpenseNo (I1 loại trừ QLDA)    -- =================================================================    SELECT 'A#05#2#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           'Đối tượng chưa xác định' AS "itemName1", MAX(E."ExpenseName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM base A  
    INNER JOIN "expense" E ON A."ExpenseID" = E."ExpenseID"  
    WHERE A."ClearanceNo" = '' AND A."ContractNo" = '' AND A."VendorNo" = '' AND A."PartnerNo" = ''  
      AND SUBSTR(A."ExpenseNo",1,6) != '119803'  
    GROUP BY A."ExpenseNo"  
) agg  
GROUP BY "itemId","parentId","level","itemName","itemName1","itemName2"  
ORDER BY "itemId";
```