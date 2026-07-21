# 1. POST /PIO208DADH/B208DADH
Desciption: báo cáo giá trị quyết toán — đề nghị/thẩm tra/phê duyệt (I1/I2/I3) từ giao dịch PAB.

| Vị trí                                    | Nguyên nhân                                                             | Loại vấn đề                                 | Đề xuất giải pháp                                                                                                                                                                                   |
| ----------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pab_trans` full scan                     |                                                                         |                                             | đánh index cho TransDate                                                                                                                                                                            |
| `pab_trans_item4` full scan + window sort | Tính rank trên toàn bảng trước khi lọc theo giao dịch trong khoảng ngày | Thiếu lọc sớm (predicate pushdown thất bại) | Viết lại: join `pab_trans` (đã lọc ngày) trước, rồi mới tính `ROW_NUMBER()` trên tập con; hoặc thêm điều kiện lọc `TransID IN (SELECT TransID FROM pab_trans WHERE ...)` ngay trong CTE `item_pick` |
| `project` full scan (nhánh rootproj)      | `TO_CHAR(ProjectID) = rootProj` (string vs number)                      | Function trên cột → mất index               | Đổi `rootProj` (hiện đang là string ghép) thành so sánh number, hoặc tạo **function-based index** `CREATE INDEX idx_project_id_char ON project(TO_CHAR(ProjectID))` nếu không sửa được logic        |

1. **`pab_trans` full scan**
	CREATE INDEX idx_pab_trans_transdate ON "pab_trans" ("TransDate");
```
**Kết quả**: pab_trans sẽ không full scan nữa mà INDEX RANGE SCAN.
```

2. **** `pab_trans_item4` full scan + window sort***
	CREATE INDEX idx_pab_trans_item4_transid ON "pab_trans_item4" ("TransID");
```
*Thay đổi:** thêm bộ lọc `TransID` ngay trong CTE `item_pick`.
**Lý do:** bản cũ tính `ROW_NUMBER()` trên toàn bộ `pab_trans_item4` rồi mới join → window sort trên full table.
**Kết quả:** chỉ rank trên tập con thuộc khoảng ngày cần báo cáo..
```


```sql
-- item_pick: giữ nguyên phần SELECT, chỉ THÊM đoạn dưới
WITH item_pick AS (  
    SELECT  
        it."TransID"          AS transId,  
        it."TransItemID"      AS transItemId,  
        it."ExpenseID"        AS expenseId,  
        it."Description"      AS description,  
        it."ProjectID"        AS itemProjectId,  
        it."LCRequestAmount"  AS reqAmt,  
        it."LCAuditAmount"    AS audAmt,  
        it."LCAprovalAmount"  AS aprAmt,  
        ROW_NUMBER() OVER (PARTITION BY it."TransID" ORDER BY it."TransItemID" DESC) rn  
    FROM "pab_trans_item4" it   
	WHERE it."TransID" IN ( -- ★ MỚI THÊM
	    SELECT PA."TransID"
	    FROM "pab_trans" PA
	    WHERE (TO_CHAR(PA."TransTypeID") LIKE '368%'
	        OR TO_CHAR(PA."TransTypeID") LIKE '369%'
	        OR TO_CHAR(PA."TransTypeID") LIKE '370%')
	      AND PA."TransDate" >= :fromDate
	      AND PA."TransDate" <= :toDate
	)
```

3.  **`project` full scan (nhánh rootproj)**
```sql
leaf_base AS (  
    SELECT  
        PA."TransNo"       AS transNo,  
        PA."TransDate"     AS td,  
        ip.transItemId,  
        ip.expenseId,  
        ip.description,  
        A."ProjectName"    AS projectName,  
        A."TabmisNo"       AS tabmisNo,  
        A."ParentID"       AS parentPid,  
        ec.cateName,  
        NVL(TO_CHAR(A."ParentID"), TO_CHAR(A."ProjectID"))  AS rootProj,     -- STRING, như bản gốc  
        NVL(A."ParentID", A."ProjectID")                    AS rootProjNum,  -- NUMBER, thêm mới  
        CASE WHEN A."ParentID" IS NULL THEN '**' ELSE TO_CHAR(A."ProjectID") END AS flag,  
        CASE WHEN PA."TransTypeID" LIKE '368%' THEN ip.reqAmt ELSE 0 END AS i1,  
        CASE WHEN PA."TransTypeID" LIKE '369%' THEN ip.audAmt ELSE 0 END AS i2,  
        CASE WHEN PA."TransTypeID" LIKE '370%' THEN ip.aprAmt ELSE 0 END AS i3  
    FROM "pab_trans" PA  
    JOIN item_pick ip ON ip.transId = PA."TransID" AND ip.rn = 1  
    JOIN "project" A  ON A."ProjectID" = ip.itemProjectId  
    JOIN exp_cate ec  ON ec.expenseId = ip.expenseId  
    WHERE (PA."TransTypeID" LIKE '368%' OR PA."TransTypeID" LIKE '369%' OR PA."TransTypeID" LIKE '370%')  
      AND PA."TransDate" >= :fromDate  
      AND PA."TransDate" <= :toDate  
      AND EXISTS (SELECT 1 FROM "project_cate" PC WHERE PC."ProjectID" = A."ProjectID")  
    leaf AS (  
        SELECT  
            rootProj || '#' || flag || '#' || TO_CHAR(expenseId) || '#' || TO_CHAR(transItemId) AS itemId,  
            TO_CHAR(expenseId)                        AS parentId,  
            CASE WHEN flag = '**' THEN 3 ELSE 4 END   AS lvl,  
            description                               AS itemName,  
            CAST(NULL AS VARCHAR2(255))               AS itemNo,  
            transNo                                   AS transNo,  
            td                                        AS td,  
            1 AS allowPrint, 0 AS fontWeight, 0 AS italic, 0 AS center,  
            :expenseShow                               AS isShow,  
            SUM(i1) AS i1, SUM(i2) AS i2, SUM(i3) AS i3,  
            rootProj                                  AS rootProj,  
            rootProjNum                               AS rootProjNum,  
            flag                                      AS flag,  
            TO_CHAR(expenseId)                        AS expenseId,  
            MIN(cateName)                             AS cateName,  
            MIN(projectName)                          AS projectName,  
            MIN(tabmisNo)                             AS tabmisNo,  
            MIN(TO_CHAR(parentPid))                   AS parentPid  
        FROM leaf_base  
        GROUP BY rootProj,rootProjNum, flag, expenseId, transItemId, description, transNo, td  
),  
    rootproj AS (  
        SELECT  
            l.rootProj                   AS itemId,  
            CAST(NULL AS VARCHAR2(255))  AS parentId,  
            1                            AS lvl,  
            MIN(P."ProjectName")         AS itemName,  
            MIN(P."TabmisNo")            AS itemNo,  
            CAST(NULL AS VARCHAR2(255))  AS transNo,  
            MAX(l.td)                    AS td,  
            1 AS allowPrint, 1 AS fontWeight, 0 AS italic, 0 AS center, 1 AS isShow,  
            SUM(l.i1) AS i1, SUM(l.i2) AS i2, SUM(l.i3) AS i3  
        FROM leaf l  
        JOIN "project" P ON P."ProjectID" = l.rootProjNum  
        WHERE l.flag <> '**'  
        GROUP BY l.rootProj  
),
```


> Tại `leaf_base` select thêm `rootProjNum`
> ```
>NVL(TO_CHAR(A."ParentID"), TO_CHAR(A."ProjectID"))  AS rootProj,     -- STRING, như bản gốc  
>NVL(A."ParentID", A."ProjectID")                    AS rootProjNum,  -- NUMBER, thêm mới 
> ```
> 
> Mục đích để thay đổi cách JOIN
> từ
> ```
> JOIN "project" P ON TO_CHAR(P."ProjectID") = l.rootProj  
> ```
> sang
> ```
> JOIN "project" P ON P."ProjectID" = l.rootProjNum  
> ```
> 
> => tận dụng được unique index của oracle khi join

# 2. POST /PIO207DADH/B207DADH

``` sql
WITH filtered_books AS (
    SELECT

        A."ProjectID"     AS "projectId",
        P."ParentID"      AS "parentProjectId",
        A."CapitalNo"     AS "capitalNo",
        A."CapitalName"   AS "capitalName",
        A."AccountNo"     AS "accountNo",
        A."PostType"      AS "postType",
        A."InTransTypeID" AS "inTransTypeId",
        A."PostDate"      AS "postDate",
        A."LCAmount"      AS "lcAmount"
    FROM "glm_gl_books" A
    INNER JOIN "project" P ON A."ProjectID" = P."ProjectID"
    WHERE A."PostType" IN (1, 2)
      AND NULLIF(TRIM(A."CapitalNo"), '') IS NOT NULL
      AND A."LCAmount" <> 0
      AND A."PostDate" <= :toDate
      AND (
            SUBSTR(A."AccountNo",1,3) = 'A24'
         OR (
               (SUBSTR(A."AccountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                AND A."InTransTypeID" NOT IN (33,34,35))
               OR SUBSTR(A."AccountNo",1,3) = '005'
            )
         OR (SUBSTR(A."AccountNo",1,4) = '2412' AND A."InTransTypeID" NOT IN (33,34))
         OR (
               (SUBSTR(A."AccountNo",1,3) IN ('009','008','006','007','012','011','Z13') AND A."InTransTypeID" = 2)
               OR (SUBSTR(A."AccountNo",1,3) IN ('009','008','006','007','012','005','011','Z13') AND A."InTransTypeID" = 7)
               OR (SUBSTR(A."AccountNo",1,3) = '005' AND A."InTransTypeID" = 2)
            )
         OR (
               (SUBSTR(A."AccountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                AND A."InTransTypeID" = 4 AND A."LCAmount" > 0)
               OR SUBSTR(A."AccountNo",1,3) IN ('005','011')
            )
         OR (SUBSTR(A."AccountNo",1,3) IN ('009','008','006','007','012','011','Z13')
             AND A."InTransTypeID" IN (3,5,30,38,8))
          )
      -- AND A."ProvinceID" = :provinceId
      -- AND A."CommuneID" = :communeId
      -- AND (A."ProjectID" = :projectId OR P."ParentID" = :projectId)
),
node_capital AS (
    SELECT
        fb."projectId" AS "projectId",
        MAX(fb."parentProjectId") AS "parentProjectId",
        fb."capitalNo" AS "capitalNo",
        MAX(fb."capitalName") AS "capitalName",
        SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = 'A24' AND fb."postDate" >= :fromDate
                 THEN fb."lcAmount" ELSE 0 END) AS "i1",
        SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = 'A24'
                 THEN fb."lcAmount" ELSE 0 END) AS "i2",
        (
            SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                      AND fb."inTransTypeId" NOT IN (33,34,35) AND fb."postType" = 1
                      AND fb."postDate" >= :fromDate
                     THEN fb."lcAmount" ELSE 0 END)
            + GREATEST(SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = '005' AND fb."postDate" >= :fromDate
                                THEN fb."lcAmount" ELSE 0 END), 0)
        ) AS "i3",
        (
            SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                      AND fb."inTransTypeId" NOT IN (33,34,35) AND fb."postType" = 1
                     THEN fb."lcAmount" ELSE 0 END)
            + GREATEST(SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = '005' THEN fb."lcAmount" ELSE 0 END), 0)
        ) AS "i4",
        SUM(CASE WHEN SUBSTR(fb."accountNo",1,4) = '2412' AND fb."inTransTypeId" NOT IN (33,34)
                  AND fb."postDate" >= :fromDate
                 THEN fb."lcAmount" ELSE 0 END) AS "i5",
        SUM(CASE WHEN SUBSTR(fb."accountNo",1,4) = '2412' AND fb."inTransTypeId" NOT IN (33,34)
                 THEN fb."lcAmount" ELSE 0 END) AS "i6",
        (
            SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                      AND fb."inTransTypeId" = 2 AND fb."postType" = 2
                      AND fb."postDate" >= :fromDate
                     THEN -fb."lcAmount" ELSE 0 END)
            + SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','005','011','Z13')
                        AND fb."inTransTypeId" = 7 AND fb."postType" = 2
                        AND fb."postDate" >= :fromDate
                       THEN fb."lcAmount" ELSE 0 END)
            + GREATEST(SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = '005' AND fb."inTransTypeId" = 2
                                AND fb."postDate" >= :fromDate
                               THEN fb."lcAmount" ELSE 0 END), 0)
        ) AS "i7",
        (
            SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                      AND fb."inTransTypeId" = 2 AND fb."postType" = 2
                     THEN -fb."lcAmount" ELSE 0 END)
            + SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','005','011','Z13')
                        AND fb."inTransTypeId" = 7 AND fb."postType" = 2
                       THEN fb."lcAmount" ELSE 0 END)
            + GREATEST(SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = '005' AND fb."inTransTypeId" = 2
                               THEN fb."lcAmount" ELSE 0 END), 0)
        ) AS "i8",
        (
            SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                      AND fb."inTransTypeId" = 4 AND fb."postType" = 2 AND fb."lcAmount" > 0
                      AND fb."postDate" >= :fromDate
                     THEN -fb."lcAmount" ELSE 0 END)
            + SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = '005' AND fb."postType" = 1
                            AND fb."postDate" >= :fromDate
                       THEN fb."lcAmount"
                       WHEN SUBSTR(fb."accountNo",1,3) = '011' AND fb."postType" = 2
                            AND fb."postDate" >= :fromDate
                       THEN -fb."lcAmount"
                       ELSE 0 END)
        ) AS "i9",
        (
            SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                      AND fb."inTransTypeId" = 4 AND fb."postType" = 2 AND fb."lcAmount" > 0
                     THEN -fb."lcAmount" ELSE 0 END)
            + SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) = '005' AND fb."postType" = 1
                       THEN fb."lcAmount"
                       WHEN SUBSTR(fb."accountNo",1,3) = '011' AND fb."postType" = 2
                       THEN -fb."lcAmount"
                       ELSE 0 END)
        ) AS "i10",
        SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                  AND fb."inTransTypeId" IN (3,5,30,38,8) AND fb."postType" = 2
                  AND fb."postDate" >= :fromDate
                 THEN fb."lcAmount" ELSE 0 END) AS "i11",
        SUM(CASE WHEN SUBSTR(fb."accountNo",1,3) IN ('009','008','006','007','012','011','Z13')
                  AND fb."inTransTypeId" IN (3,5,30,38,8) AND fb."postType" = 2
                 THEN fb."lcAmount" ELSE 0 END) AS "i12"
    FROM filtered_books fb
    GROUP BY fb."projectId", fb."capitalNo"
),
leaf_amounts AS (
    SELECT nc.* FROM node_capital nc
    WHERE (nc."parentProjectId" IS NOT NULL AND nc."parentProjectId" <> nc."projectId")
       OR (nc."parentProjectId" IS NULL
           AND NOT EXISTS (SELECT 1 FROM "project" C WHERE C."ParentID" = nc."projectId"))
),
level3 AS (
    SELECT
        CASE WHEN la."parentProjectId" IS NOT NULL AND la."parentProjectId" <> la."projectId"
             THEN TO_CHAR(la."parentProjectId") || '#' || TO_CHAR(la."projectId") || '#' || la."capitalNo"
             ELSE TO_CHAR(la."projectId") || '#' || la."capitalNo" END AS "itemId",
        CASE WHEN la."parentProjectId" IS NOT NULL AND la."parentProjectId" <> la."projectId"
             THEN TO_CHAR(la."parentProjectId") || '#' || TO_CHAR(la."projectId")
             ELSE TO_CHAR(la."projectId") END AS "parentId",
        la."projectId",
        la."capitalName" AS "itemName",
        la."capitalNo" AS "itemName1",
        3 AS "level", 0 AS "fontWeight",
        la."i1", la."i2", la."i3", la."i4", la."i5", la."i6",
        la."i7", la."i8", la."i9", la."i10", la."i11", la."i12"
    FROM leaf_amounts la
),
child_projects AS (
    SELECT DISTINCT fb."parentProjectId" AS "rootProjectId", fb."projectId" AS "childProjectId"
    FROM filtered_books fb
    WHERE fb."parentProjectId" IS NOT NULL
),
level2 AS (
    SELECT
        TO_CHAR(cp."rootProjectId") || '#' || TO_CHAR(cp."childProjectId") AS "itemId",
        TO_CHAR(cp."rootProjectId") AS "parentId",
        cp."childProjectId" AS "projectId",
        P."ProjectName" AS "itemName",
        P."TabmisNo" AS "itemName1",
        2 AS "level", 0 AS "fontWeight",
        COALESCE(SUM(l3."i1"),0) AS "i1", COALESCE(SUM(l3."i2"),0) AS "i2",
        COALESCE(SUM(l3."i3"),0) AS "i3", COALESCE(SUM(l3."i4"),0) AS "i4",
        COALESCE(SUM(l3."i5"),0) AS "i5", COALESCE(SUM(l3."i6"),0) AS "i6",
        COALESCE(SUM(l3."i7"),0) AS "i7", COALESCE(SUM(l3."i8"),0) AS "i8",
        COALESCE(SUM(l3."i9"),0) AS "i9", COALESCE(SUM(l3."i10"),0) AS "i10",
        COALESCE(SUM(l3."i11"),0) AS "i11", COALESCE(SUM(l3."i12"),0) AS "i12"
    FROM child_projects cp
    JOIN "project" P ON P."ProjectID" = cp."childProjectId"
    LEFT JOIN level3 l3 ON l3."parentId" = TO_CHAR(cp."rootProjectId") || '#' || TO_CHAR(cp."childProjectId")
    GROUP BY cp."rootProjectId", cp."childProjectId", P."ProjectName", P."TabmisNo"
),
level1_rollup AS (
    SELECT CAST("parentId" AS NUMBER) AS "rootProjectId",
        SUM("i1") "i1", SUM("i2") "i2", SUM("i3") "i3", SUM("i4") "i4",
        SUM("i5") "i5", SUM("i6") "i6", SUM("i7") "i7", SUM("i8") "i8",
        SUM("i9") "i9", SUM("i10") "i10", SUM("i11") "i11", SUM("i12") "i12"
    FROM level2
    GROUP BY "parentId"
),
level1_from_children AS (
    SELECT
        TO_CHAR(lr."rootProjectId") AS "itemId",
        CAST(NULL AS VARCHAR2(100)) AS "parentId",
        lr."rootProjectId" AS "projectId",
        P."ProjectName" AS "itemName",
        P."ProjectNo" AS "itemName1",
        1 AS "level", 0 AS "fontWeight",
        lr."i1", lr."i2", lr."i3", lr."i4", lr."i5", lr."i6",
        lr."i7", lr."i8", lr."i9", lr."i10", lr."i11", lr."i12"
    FROM level1_rollup lr
    JOIN "project" P ON P."ProjectID" = lr."rootProjectId"
),
level1_standalone AS (
    SELECT
        TO_CHAR(l3."projectId") AS "itemId",
        CAST(NULL AS VARCHAR2(100)) AS "parentId",
        l3."projectId",
        P."ProjectName" AS "itemName",
        P."ProjectNo" AS "itemName1",
        1 AS "level", 0 AS "fontWeight",
        SUM(l3."i1") "i1", SUM(l3."i2") "i2", SUM(l3."i3") "i3", SUM(l3."i4") "i4",
        SUM(l3."i5") "i5", SUM(l3."i6") "i6", SUM(l3."i7") "i7", SUM(l3."i8") "i8",
        SUM(l3."i9") "i9", SUM(l3."i10") "i10", SUM(l3."i11") "i11", SUM(l3."i12") "i12"
    FROM level3 l3
    JOIN "project" P ON P."ProjectID" = l3."projectId"
    WHERE l3."parentId" = TO_CHAR(l3."projectId")
    GROUP BY l3."projectId", P."ProjectName", P."ProjectNo"
),
level1 AS (
    SELECT * FROM level1_from_children
    UNION ALL
    SELECT * FROM level1_standalone
),
total_row AS (
    SELECT
        '' AS "itemId", CAST(NULL AS VARCHAR2(100)) AS "parentId", CAST(NULL AS NUMBER) AS "projectId",
        'Tổng cộng' AS "itemName", CAST(NULL AS VARCHAR2(15)) AS "itemName1",
        0 AS "level", 1 AS "fontWeight",
        SUM("i1") "i1", SUM("i2") "i2", SUM("i3") "i3", SUM("i4") "i4",
        SUM("i5") "i5", SUM("i6") "i6", SUM("i7") "i7", SUM("i8") "i8",
        SUM("i9") "i9", SUM("i10") "i10", SUM("i11") "i11", SUM("i12") "i12"
    FROM level1
)
SELECT "itemId","parentId","projectId","itemName","itemName1","level","fontWeight",
       "i1","i2","i3","i4","i5","i6","i7","i8","i9","i10","i11","i12"
FROM (
    SELECT l.*, 0 AS ord FROM level1 l
    UNION ALL
    SELECT l.*, 0 AS ord FROM level2 l
    UNION ALL
    SELECT l.*, 0 AS ord FROM level3 l
    UNION ALL
    SELECT t.*, 1 AS ord FROM total_row t
) all_rows
ORDER BY ord, "itemId";
```


>Bảng `glm_gl_books` đang full scan và trong các query , điều kiện có khả năng loại bỏ dữ liệu lớn nhất là:
> 
> ```
> A.PostDate <= :toDate
> ```
> 
> Sau khi tạo index trên `PostDate`
> ```
> CREATE INDEX IDX_GLM_BOOKS_POSTDATE  ON "glm_gl_books"("PostDate");
> ```
> execution plan chuyển từ:
> ```
> TABLE ACCESS FULL GLM_GL_BOOKS
> ```
> 
> sang:
> 
> ```
> INDEX RANGE SCAN IDX_POSTDATE
> TABLE ACCESS BY INDEX ROWID
> ```
> 
> nên Oracle chỉ cần đọc tập dữ liệu thỏa mãn điều kiện ngày thay vì quét toàn bộ bảng.

# 3. POST /PIMTT912025TTBTC/MB11QTDAP2

```sql
SELECT  
    D."LineID"              AS "LineID",  
    P."ProjectID"           AS "ProjectID",  
    D."AccountNo"           AS "AccountNo",  
    D."PostType"            AS "PostType",  
    D."InTransTypeID"       AS "InTransTypeID",  
    D."PostDate"            AS "PostDate",  
    D."LCAmount"            AS "LCAmount",  
    PSI."StatusValue"       AS "StatusValue",  
    PSI."StatusID"          AS "StatusID",  
    D."BudgetLevel"         AS "BudgetLevel",  
    PSI."ProjectStatusDate" AS "ProjectStatusDate",  
    P."ManagementLevel"     AS "ManagementLevel"  
FROM "glm_gl_books" D  
JOIN "project" P ON D."ProjectID" = P."ProjectID"  
JOIN "project_status_item" PSI ON P."ProjectID" = PSI."ProjectID"  
JOIN (  
    SELECT "ProjectID", "StatusID", MAX("LineID") AS "MaxLineID"  
    FROM "project_status_item"  
    GROUP BY "ProjectID", "StatusID"  
) x ON PSI."ProjectID" = x."ProjectID"  
   AND PSI."StatusID"  = x."StatusID"  
   AND PSI."LineID"    = x."MaxLineID"  
WHERE 1 = 1  
  AND (  
        (PSI."StatusID" = 34 AND PSI."StatusValue" IN (3, 4))  
     OR PSI."StatusID" IN (10, 11, 12)  
  )  
GROUP BY D."LineID", P."ProjectID", PSI."StatusID", PSI."StatusValue",  
         D."AccountNo", D."PostType", D."InTransTypeID", D."PostDate",  
         D."LCAmount", D."BudgetLevel", PSI."ProjectStatusDate", P."ManagementLevel";
```


>Bảng `glm_gl_books` đang full scan và trong query và được join vào `project` qua một cột `D."ProjectID"`:
> 
> ```
> JOIN "project" P ON D."ProjectID" = P."ProjectID"
> ```
> 
> Sau khi tạo index cho cột FK `ProjectID`
> ```
> CREATE INDEX IDX_GLM_BOOKS_PROJECT ON "glm_gl_books"("ProjectID");
> ```
> execution plan chuyển từ:
> ```
> TABLE ACCESS FULL GLM_GL_BOOKS
> ```
> 
> sang:
> 
> ```
> INDEX RANGE SCAN IDX_GLM_BOOKS_PROJECT
> TABLE ACCESS BY INDEX ROWID
> ```
> 
> nên Oracle chỉ cần đọc tập dữ liệu thỏa mãn điều kiện ngày thay vì quét toàn bộ bảng.

# 4. POST /PIMTT912025TTBTC/MB11QTDAP3
```sql

WITH tmp_glm_books AS (
    SELECT
        A."AccountNo", A."PostType", A."ProjectID", A."LineID",
        P."ProgramID"           AS "ProgramID",
        P."ProjectName"         AS "ProjectName",
        PG."ProgramName"        AS "ProgramName",
        A."LCAmount", A."PostDate", A."InTransTypeID", A."BudgetLevel",
        P."Group"               AS "Group",
        COALESCE(EXTRACT(YEAR FROM P."StartedDate"), 0)  AS "StartedYear",
        COALESCE(EXTRACT(YEAR FROM P."FinishedDate"), 0) AS "FinishedYear",
        PG."ProgramType"        AS "ProgramType",
        COALESCE(PS."Condition1", 0) AS "Condition1",
        COALESCE(PS."Condition2", 0) AS "Condition2",
        COALESCE(PS."Condition3", 0) AS "Condition3",
        COALESCE(PS."Condition4", 0) AS "Condition4",
        COALESCE(PS."Condition5", 0) AS "Condition5",
        COALESCE(PS."Condition6", 0) AS "Condition6",
        COALESCE(PS."Condition7", 0) AS "Condition7",
        COALESCE(PS."Condition8", 0) AS "Condition8",
        CASE WHEN SUBSTR(A."AccountNo", 1, 3) = 'D25' AND A."PostType" = 1
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
             THEN A."LCAmount" ELSE 0 END AS "I1",
        CASE WHEN SUBSTR(A."AccountNo", 1, 3) = 'D25' AND A."PostType" = 1
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD') AND A."BudgetLevel" = 1
             THEN A."LCAmount" ELSE 0 END AS "I2",
        CASE WHEN SUBSTR(A."AccountNo", 1, 5) = '24121'
                  AND A."InTransTypeID" NOT IN (33, 34, 35)
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
             THEN A."LCAmount" ELSE 0 END AS "I3",
        CASE WHEN SUBSTR(A."AccountNo", 1, 5) = '24122'
                  AND A."PostType" = 1 AND A."InTransTypeID" = 35
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
             THEN A."LCAmount" ELSE 0 END AS "I4",
        CASE WHEN SUBSTR(A."AccountNo", 1, 3) IN ('006','007','008','009','011','005')
                  AND A."PostType" = 1
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
             THEN A."LCAmount" ELSE 0 END AS "I5",
        CASE WHEN SUBSTR(A."AccountNo", 1, 3) IN ('006','007','008','009','011','005')
                  AND A."PostType" = 1 AND A."BudgetLevel" = 1
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
             THEN A."LCAmount" ELSE 0 END AS "I6",
        CASE
            WHEN SUBSTR(A."AccountNo", 1, 3) IN ('006','007','008','009','011')
                 AND A."PostType" = 2
                 AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
            THEN A."LCAmount" * (-1)
            WHEN SUBSTR(A."AccountNo", 1, 3) = '005' AND A."PostType" = 1
                 AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
            THEN A."LCAmount" ELSE 0
        END AS "I7",
        CASE
            WHEN SUBSTR(A."AccountNo", 1, 3) IN ('006','007','008','009','011')
                 AND A."PostType" = 2 AND A."BudgetLevel" = 1
                 AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
            THEN A."LCAmount" * (-1)
            WHEN SUBSTR(A."AccountNo", 1, 3) = '005' AND A."PostType" = 1
                 AND A."BudgetLevel" = 1
                 AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
            THEN A."LCAmount" ELSE 0
        END AS "I8",
        CASE WHEN SUBSTR(A."AccountNo", 1, 4) = 'A241'
                  AND A."PostType" = 1 AND A."BudgetLevel" = 1
                  AND A."PostDate" <= TO_DATE(:toDateStr, 'YYYY-MM-DD')
             THEN A."LCAmount" ELSE 0 END AS "I10"
    FROM "glm_gl_books_test" A
    LEFT JOIN "project" P  ON P."ProjectID"  = A."ProjectID"
    LEFT JOIN "program" PG ON P."ProgramID"  = PG."ProgramID"
    LEFT JOIN (
        SELECT psi."ProjectID",
            MAX(CASE WHEN psi."StatusID" = 12 AND psi."StatusValue" = 3 THEN 1 ELSE 0 END) AS "Condition1",
            MAX(CASE WHEN (psi."StatusID" = 12 AND psi."StatusValue" IN (1, 2)) OR (psi."StatusID" IS NULL) THEN 1 ELSE 0 END) AS "Condition2",
            MAX(CASE WHEN psi."StatusID" = 10 AND psi."StatusValue" = 2 THEN 1 ELSE 0 END) AS "Condition3",
            MAX(CASE WHEN (psi."StatusID" = 10 AND psi."StatusValue" = 1) OR (psi."StatusID" NOT IN (10, 12, 11)) THEN 1 ELSE 0 END) AS "Condition4",
            MAX(CASE WHEN psi."StatusID" = 12 AND psi."StatusValue" = 4 THEN 1 ELSE 0 END) AS "Condition5",
            MAX(CASE WHEN psi."StatusID" = 10 AND psi."StatusValue" NOT IN (2, 3, 4, 5) THEN 1 ELSE 0 END) AS "Condition6",
            MAX(CASE WHEN psi."StatusID" = 11 AND psi."StatusValue" = 4 THEN 1 ELSE 0 END) AS "Condition7",
            MAX(CASE WHEN psi."StatusID" = 10 AND psi."StatusValue" IN (5, 6) THEN 1 ELSE 0 END) AS "Condition8"
        FROM "project_status_item" psi
        JOIN (SELECT "ProjectID", "StatusID", MAX("LineID") AS "MaxLineID"
              FROM "project_status_item" GROUP BY "ProjectID", "StatusID") x
          ON psi."ProjectID" = x."ProjectID" AND psi."StatusID" = x."StatusID" AND psi."LineID" = x."MaxLineID"
        WHERE ((psi."StatusID" = 34 AND psi."StatusValue" IN (3, 4)) OR (psi."StatusID" IN (10, 11, 12)))
        GROUP BY psi."ProjectID"
    ) PS ON PS."ProjectID" = A."ProjectID"
    WHERE 1 = 1
    -- ================== CÁC FILTER OPTIONAL ĐÃ BỎ ==================
    -- (mở comment + thay giá trị nếu muốn test case có filter)
    -- AND SUBSTR(A."CompanyNo", 1, LENGTH(:companyNoPrefix)) = :companyNoPrefix
    -- AND A."ProvinceNo" = :provinceNo
    -- AND A."CommuneNo"  = :communeNo
    -- AND A."ProjectID"  = :projectId
    -- ================================================================
),
project_rows AS (
    SELECT
        MAX(CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01#01#' || B."ProjectID"
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02#01#' || B."ProjectID"
            WHEN B."Condition4" = 1 THEN 'A#04#03#01#' || B."ProjectID"
            WHEN B."Condition5" = 1 THEN 'A#05#01#01#' || B."ProjectID"
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1)
                 THEN 'A#05#02#01#' || B."ProjectID"
            WHEN B."Condition8" = 1 THEN 'A#05#03#01#' || B."ProjectID"
        END)                                    AS "ItemID",
        CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01'
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02'
            WHEN B."Condition4" = 1 THEN 'A#04#03'
            WHEN B."Condition5" = 1 THEN 'A#05#01'
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1) THEN 'A#05#02'
            WHEN B."Condition8" = 1 THEN 'A#05#03'
        END                                     AS "ParentID",
        CAST(NULL AS VARCHAR2(50))               AS "STT",
        MAX(B."ProjectName")                     AS "ItemName",
        3                                        AS "Level",
        1                                        AS "TypeChild",
        MAX(B."ProjectID")                       AS "ProjectID",
        MAX(B."ProjectName")                     AS "ProjectName",
        CAST(NULL AS VARCHAR2(255))              AS "ProjectItemID",
        CAST(MAX(B."Group") AS VARCHAR2(255))    AS "CountOrGroup",
        CAST(MAX(B."StartedYear") AS VARCHAR2(10)) || ' - ' || CAST(MAX(B."FinishedYear") AS VARCHAR2(10)) AS "StartToFinish",
        SUM(B."I1")  AS "I1",  SUM(B."I2")  AS "I2",
        GREATEST(SUM(B."I3"), 0) AS "I3", SUM(B."I4") AS "I4",
        SUM(B."I5")  AS "I5",  SUM(B."I6")  AS "I6",
        SUM(B."I7")  AS "I7",  SUM(B."I8")  AS "I8",
        CASE WHEN SUM(B."I4") <> 0 THEN SUM(B."I4") - SUM(B."I5")
             ELSE SUM(B."I3") - SUM(B."I5") END AS "I9",
        SUM(B."I10") AS "I10",
        0 AS "FontWeight", 0 AS "Italic",
        1 AS "Detail", 1 AS "Sumup", 1 AS "isShow", 0 AS "Center"
    FROM tmp_glm_books B
    WHERE B."ProjectID" > 0 AND (B."ProgramID" IS NULL OR B."ProgramID" = 0)
    GROUP BY B."ProjectID",
        CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01'
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02'
            WHEN B."Condition4" = 1 THEN 'A#04#03'
            WHEN B."Condition5" = 1 THEN 'A#05#01'
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1) THEN 'A#05#02'
            WHEN B."Condition8" = 1 THEN 'A#05#03'
        END
    HAVING
        MAX(CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01#01#' || B."ProjectID"
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02#01#' || B."ProjectID"
            WHEN B."Condition4" = 1 THEN 'A#04#03#01#' || B."ProjectID"
            WHEN B."Condition5" = 1 THEN 'A#05#01#01#' || B."ProjectID"
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1)
                 THEN 'A#05#02#01#' || B."ProjectID"
            WHEN B."Condition8" = 1 THEN 'A#05#03#01#' || B."ProjectID"
        END) IS NOT NULL
),
program_rows AS (
    SELECT
        MAX(CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01#02#' || B."ProgramID"
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02#02#' || B."ProgramID"
            WHEN B."Condition4" = 1 THEN 'A#04#03#02#' || B."ProgramID"
            WHEN B."Condition5" = 1 THEN 'A#05#01#02#' || B."ProgramID"
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1)
                 THEN 'A#05#02#02#' || B."ProgramID"
            WHEN B."Condition8" = 1 THEN 'A#05#03#02#' || B."ProgramID"
        END)                                    AS "ItemID",
        CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01#02'
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02#02'
            WHEN B."Condition4" = 1 THEN 'A#04#03#02'
            WHEN B."Condition5" = 1 THEN 'A#05#01#02'
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1) THEN 'A#05#02#02'
            WHEN B."Condition8" = 1 THEN 'A#05#03#02'
        END                                     AS "ParentID",
        CAST(NULL AS VARCHAR2(50))               AS "STT",
        MAX(B."ProgramName")                     AS "ItemName",
        4                                        AS "Level",
        2                                        AS "TypeChild",
        MAX(B."ProjectID")                       AS "ProjectID",
        CAST(NULL AS VARCHAR2(255))              AS "ProjectName",
        CAST(NULL AS VARCHAR2(255))              AS "ProjectItemID",
        CAST(COUNT(DISTINCT B."ProjectID") AS VARCHAR2(255)) AS "CountOrGroup",
        CAST(NULL AS VARCHAR2(255))              AS "StartToFinish",
        SUM(B."I1")  AS "I1",  SUM(B."I2")  AS "I2",
        GREATEST(SUM(B."I3"), 0) AS "I3", SUM(B."I4") AS "I4",
        SUM(B."I5")  AS "I5",  SUM(B."I6")  AS "I6",
        SUM(B."I7")  AS "I7",  SUM(B."I8")  AS "I8",
        CASE WHEN SUM(B."I4") <> 0 THEN SUM(B."I4") - SUM(B."I5")
             ELSE SUM(B."I3") - SUM(B."I5") END AS "I9",
        SUM(B."I10") AS "I10",
        0 AS "FontWeight", 0 AS "Italic",
        1 AS "Detail", 1 AS "Sumup", 1 AS "isShow", 0 AS "Center"
    FROM tmp_glm_books B
    WHERE B."ProjectID" > 0 AND B."ProgramID" > 0 AND B."ProgramType" = 1
    GROUP BY
        CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01#02'
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02#02'
            WHEN B."Condition4" = 1 THEN 'A#04#03#02'
            WHEN B."Condition5" = 1 THEN 'A#05#01#02'
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1) THEN 'A#05#02#02'
            WHEN B."Condition8" = 1 THEN 'A#05#03#02'
        END,
        B."ProgramID"
    HAVING
        MAX(CASE
            WHEN B."Condition1" = 1 THEN 'A#04#01#02#' || B."ProgramID"
            WHEN B."Condition2" = 1 AND B."Condition3" = 1 THEN 'A#04#02#02#' || B."ProgramID"
            WHEN B."Condition4" = 1 THEN 'A#04#03#02#' || B."ProgramID"
            WHEN B."Condition5" = 1 THEN 'A#05#01#02#' || B."ProgramID"
            WHEN (B."Condition2" = 1 AND B."Condition6" = 1) OR (B."Condition2" = 1 AND B."Condition7" = 1)
                 THEN 'A#05#02#02#' || B."ProgramID"
            WHEN B."Condition8" = 1 THEN 'A#05#03#02#' || B."ProgramID"
        END) IS NOT NULL
)
SELECT * FROM project_rows
UNION ALL
SELECT * FROM program_rows;
```

> 2 Bảng `project_rows` và `program_rows` đều đang join vào bảng temp `tmp_glm_books` đang lấy ra dữ liệu với điều kiện WHERE như sau:
> 
> ```
> WHERE B."ProjectID" > 0 AND (B."ProgramID" IS NULL OR B."ProgramID" = 0)
> WHERE B."ProjectID" > 0 AND B."ProgramID" > 0 AND B."ProgramType" = 1
> ```
> 
> Đưa điều kiện chung `B."ProjectID" > 0` lên WHERE của  bảng temp `tmp_glm_books
> => đkien của 2 bảng temp còn lại là
> ```
> WHERE  (B."ProgramID" IS NULL OR B."ProgramID" = 0)
> WHERE  B."ProgramID" > 0 AND B."ProgramType" = 1
> ```
> execution plan chuyển từ:
> ```
> TABLE ACCESS FULL GLM_GL_BOOKS
> ```
> 
> sang:
> 
> ```
> INDEX RANGE SCAN IDX_GLM_BOOKS_PROJECT
> TABLE ACCESS BY INDEX ROWID
> ```
> 
> nên Oracle chỉ cần đọc tập dữ liệu thỏa mãn điều kiện ngày thay vì quét toàn bộ bảng.

# 5. findMasterAggregates

> [!NOTE]
> **Lý do**
> Đang scan bảng `glm_gl_books` 9 lần trong 1 query
> 
>  **Giải pháp**
> 9 scalar subquery riêng, cùng lọc `ProjectID = :projectId` nhưng lại đang full-scan 9 lần mỗi lần load
> => scan bảng `glm_gl_books` 1 lần theo `ProjectID = :projectId`  sau đó các subquery sau dùng data temp đó để lọc

 **--- SQL ---**

Hiện trạng:
```sql
SELECT  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE SUBSTR(A."AccountNo", 1, 3) IN ('D25', 'D26')  
       AND A."ProjectID" = :projectId  
       AND A."PostDate" <= :toDate  
       AND A."PostType" = 1  
    ) AS tongDauTu,  
  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE SUBSTR(A."AccountNo", 1, 3) IN ('D35', 'D36')  
       AND A."ProjectID" = :projectId  
       AND A."PostDate" <= :toDate  
       AND A."PostType" = 1  
    ) AS tongDuToan,  
  
    -- Bug (1): chiều so sánh ngược — ported as-is  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE SUBSTR(A."AccountNo", 1, 3) = 'D13'  
       AND A."ProjectID" = :projectId  
       AND A."PostDate" <= :fromDate  
       AND A."PostDate" >= :toDate  
       AND A."PostType" = 1  
       AND A."PeriodType" IN (9, 10)  
    ) AS khvTrungHan,  
  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE SUBSTR(A."AccountNo", 1, 3) IN ('009', '008', '006', '007', '005')  
       AND A."ProjectID" = :projectId  
       AND A."PostType" = 1  
       AND A."PostDate" <= :toDate  
       AND A."PostDate" >= :fromDate  
       AND A."PeriodType" = 1  
    ) AS khvHangNam,  
  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE SUBSTR(A."AccountNo", 1, 3) = 'A25'  
       AND A."ProjectID" = :projectId  
       AND A."PostType" = 1  
    ) AS uth,  
  
    -- Bug/đặc thù (3): MAX(CASE) trên nhóm (AccountNo, PostType) — ported as-is  
    (SELECT COALESCE(MAX(CASE WHEN SUBSTR(C."AccountNo", 1, 5) = '24121' AND C."PostType" = 1 THEN C.amt ELSE 0 END), 0)  
          - COALESCE(MAX(CASE WHEN SUBSTR(C."AccountNo", 1, 5) = '24121' AND C."PostType" = 2 THEN -C.amt ELSE 0 END), 0)  
     FROM (  
         SELECT A."AccountNo" AS "AccountNo", A."PostType" AS "PostType", SUM(A."LCAmount") AS amt  
         FROM "glm_gl_books" A  
         WHERE SUBSTR(A."AccountNo", 1, 5) = '24121'  
           AND A."PostType" IN (1, 2)  
           AND A."InTransTypeNo" NOT IN ('32', '33', '34', '35')  
           AND A."PostDate" <= :toDate  
           AND A."ProjectID" = :projectId  
         GROUP BY A."AccountNo", A."PostType"  
     ) C  
    ) AS klNghiemThu,  
  
    -- Bug/đặc thù (3): cùng dạng MAX(CASE) theo nhóm — ported as-is  
    (SELECT COALESCE(MAX(CASE WHEN SUBSTR(C."AccountNo", 1, 3) = '005' AND C."PostType" = 1 THEN C.amt ELSE 0 END), 0)  
          + COALESCE(MAX(CASE WHEN SUBSTR(C."AccountNo", 1, 3) IN ('009', '008', '006', '007', '011', '012', 'Z13') AND C."PostType" = 2 THEN -C.amt ELSE 0 END), 0)  
     FROM (  
         SELECT A."AccountNo" AS "AccountNo", A."PostType" AS "PostType", SUM(A."LCAmount") AS amt  
         FROM "glm_gl_books" A  
         WHERE SUBSTR(A."AccountNo", 1, 3) IN ('009', '008', '006', '007', '011', '012', 'Z13', '005')  
           AND A."PostType" IN (1, 2)  
           AND A."ProjectID" = :projectId  
         GROUP BY A."AccountNo", A."PostType"  
     ) C  
    ) AS klGiaiNgan,  
  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE SUBSTR(A."AccountNo", 1, 3) = 'C24'  
       AND A."ProjectID" = :projectId  
       AND A."PostType" = 1  
    ) AS klKiemToan,  
  
    -- Bug (2): vế OR đầu KHÔNG có ProjectID/PostType — ported as-is  
    (SELECT COALESCE(SUM(A."LCAmount"), 0)  
     FROM "glm_gl_books" A  
     WHERE (SUBSTR(A."AccountNo", 1, 5) = '24122' AND SUBSTR(A."CoAccountNo", 1, 3) IN ('211', '213'))  
        OR (SUBSTR(A."AccountNo", 1, 5) = '24122' AND SUBSTR(A."CoAccountNo", 1, 3) IN ('343', '812', '211', '213')  
            AND A."ProjectID" = :projectId AND A."PostType" = 2)  
    ) AS klQuyetToan  
FROM DUAL;
```

sau
```sql
WITH base AS (  
    SELECT /*+ MATERIALIZE */  
        A."AccountNo", A."CoAccountNo", A."PostType", A."PostDate",  
        A."PeriodType", A."InTransTypeNo", A."LCAmount"  
    FROM "glm_gl_books" A  
    WHERE A."ProjectID" = :projectId  
),  
grouped AS (  
    SELECT "AccountNo", "PostType", SUM("LCAmount") AS amt  
    FROM base  
    WHERE "PostType" IN (1, 2)  
      AND (  
            SUBSTR("AccountNo",1,3) IN ('009','008','006','007','011','012','Z13','005')  
         OR (SUBSTR("AccountNo",1,5) = '24121'  
             AND "InTransTypeNo" NOT IN ('32','33','34','35')  
             AND "PostDate" <= :toDate)  
      )  
    GROUP BY "AccountNo", "PostType"  
)  
SELECT  
    COALESCE(m.tongDauTu, 0)   AS tongDauTu,  
    COALESCE(m.tongDuToan, 0)  AS tongDuToan,  
    COALESCE(m.khvTrungHan, 0) AS khvTrungHan,  
    COALESCE(m.khvHangNam, 0)  AS khvHangNam,  
    COALESCE(m.uth, 0)         AS uth,  
    COALESCE(m.klKiemToan, 0)  AS klKiemToan,  
    (SELECT COALESCE(MAX(CASE WHEN g."PostType"=1 THEN g.amt ELSE 0 END),0)  
          - COALESCE(MAX(CASE WHEN g."PostType"=2 THEN -g.amt ELSE 0 END),0)  
     FROM grouped g  
     WHERE SUBSTR(g."AccountNo",1,5) = '24121') AS klNghiemThu,  
    (SELECT COALESCE(MAX(CASE WHEN SUBSTR(g."AccountNo",1,3)='005' AND g."PostType"=1 THEN g.amt ELSE 0 END),0)  
          + COALESCE(MAX(CASE WHEN SUBSTR(g."AccountNo",1,3) IN ('009','008','006','007','011','012','Z13') AND g."PostType"=2 THEN -g.amt ELSE 0 END),0)  
     FROM grouped g  
     WHERE SUBSTR(g."AccountNo",1,3) IN ('009','008','006','007','011','012','Z13','005')) AS klGiaiNgan,  
    (SELECT COALESCE(SUM(A."LCAmount"),0)   -- OR-branch không lọc ProjectID: bug ported as-is  
     FROM "glm_gl_books" A  
     WHERE (SUBSTR(A."AccountNo",1,5)='24122' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
        OR (SUBSTR(A."AccountNo",1,5)='24122' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')  
            AND A."ProjectID" = :projectId AND A."PostType" = 2)  
    ) AS klQuyetToan  
FROM (  
    SELECT  
        SUM(CASE WHEN SUBSTR("AccountNo",1,3) IN ('D25','D26') AND "PostType"=1 AND "PostDate"<=:toDate THEN "LCAmount" ELSE 0 END) AS tongDauTu,  
        SUM(CASE WHEN SUBSTR("AccountNo",1,3) IN ('D35','D36') AND "PostType"=1 AND "PostDate"<=:toDate THEN "LCAmount" ELSE 0 END) AS tongDuToan,  
        SUM(CASE WHEN SUBSTR("AccountNo",1,3)='D13' AND "PostType"=1 AND "PeriodType" IN (9,10)  
                 AND "PostDate"<=:fromDate AND "PostDate">=:toDate THEN "LCAmount" ELSE 0 END) AS khvTrungHan,  -- bug đảo chiều: as-is  
        SUM(CASE WHEN SUBSTR("AccountNo",1,3) IN ('009','008','006','007','005') AND "PostType"=1 AND "PeriodType"=1  
                 AND "PostDate"<=:toDate AND "PostDate">=:fromDate THEN "LCAmount" ELSE 0 END) AS khvHangNam,  
        SUM(CASE WHEN SUBSTR("AccountNo",1,3)='A25' AND "PostType"=1 THEN "LCAmount" ELSE 0 END) AS uth,  
        SUM(CASE WHEN SUBSTR("AccountNo",1,3)='C24' AND "PostType"=1 THEN "LCAmount" ELSE 0 END) AS klKiemToan  
    FROM base  
) m
```

> [!NOTE]
> Query sau sửa sẽ còn full scan bảng `glm_gl_books` 2 lần 
> Vẫn dư lên 1 lần thành 2 là bởi vì trong code có đoạn fix bug PHP và sql đó k lọc theo ProjectID


Với dữ liệu bảng glm_gl_books 1 triệu bản ghi cost giảm từ 245K -> xuống 54584


# 6. B209M1Leaf

> [!NOTE]
> **Lý do**
> Đang scan bảng `glm_gl_books` 8 lần trong 1 query
> 
>  **Giải pháp**
> 1 LẦN SCAN duy nhất thay cho 8 lần. Điều kiện của mỗi nhánh gốc được đưa vào CASE WHEN riêng, KHÔNG đổi ngữ nghĩa, KHÔNG gộp cột.

 **--- SQL ---**

Hiện trạng:

```sql
  
WITH leaves AS (  
    -- Sub 1: I2 - LCAmount dương, tài khoản 2431/2412  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        SUM(A."LCAmount") AS "I2", 0 AS "I4", 0 AS "I5", 0 AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND SUBSTR(A."AccountNo",1,4) IN ('2431','2412')  
      AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- === m1WhereBase (tùy filter truyền vào) ===  
      AND SUBSTR(A."CompanyNo", 1, :companyPrefixLen) = :companyNoPrefix   -- nếu filter.companyNoPrefix != null  
      AND A."ProjectNo" LIKE :projectNo || '%'                              -- nếu filter.projectNo != null  
      AND A."ProvinceNo" = :provinceNo                                     -- nếu filter.provinceNo != null  
      AND A."CommuneNo" = :communeNo                                       -- nếu filter.communeNo != null  
      AND A."ProjectID" = :projectId                                       -- nếu filter.projectId != null  
      -- ============================================      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") > 0  
  
    UNION ALL  
  
    -- Sub 2: I4 - tạm ứng nhóm 1 (008xxx/009xx/004xx/006xx, 014/097)  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", -SUM(A."LCAmount") AS "I4", 0 AS "I5", 0 AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."InTransTypeID" IN (2,11,21,7,17,26)  
      AND A."PostType" = 2  
      AND (SUBSTR(A."AccountNo",1,6) IN ('008111','008121','008211','008221')  
        OR SUBSTR(A."AccountNo",1,5) IN ('00911','00921','00931','00411','00421','00621','00611')  
        OR SUBSTR(A."AccountNo",1,3) IN ('014','097'))  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase (giống trên)  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
  
    UNION ALL  
  
    -- Sub 3: I4 - tài khoản 013, PostType=2  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", -SUM(A."LCAmount") AS "I4", 0 AS "I5", 0 AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."PostType" = 2  
      AND SUBSTR(A."AccountNo",1,3) IN ('013')  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
  
    UNION ALL  
  
    -- Sub 4: I5 - hoàn ứng nhóm 1 (bao gồm 012/013/014/097), toàn kỳ đến toDate  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", 0 AS "I4", -SUM(A."LCAmount") AS "I5", 0 AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."InTransTypeID" IN (3,12,22,8,18,27,30,38,36)  
      AND A."PostType" = 2  
      AND (SUBSTR(A."AccountNo",1,6) IN ('008112','008122','008212','008222')  
        OR SUBSTR(A."AccountNo",1,5) IN ('00912','00922','00932','00412','00422','00622','00612')  
        OR SUBSTR(A."AccountNo",1,3) IN ('012','014','013','097'))  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
  
    UNION ALL  
  
    -- Sub 5: I5 - hoàn ứng tài khoản 013, GIỚI HẠN trong khoảng fromDate..toDate  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", 0 AS "I4", -SUM(A."LCAmount") AS "I5", 0 AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" >= TO_DATE(:fromDateIso,'YYYY-MM-DD')  
      AND A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."PostType" = 2  
      AND SUBSTR(A."AccountNo",1,3) IN ('013')  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
  
    UNION ALL  
  
    -- Sub 6: I6 - nhóm InTransTypeID (2,11,21,14,7,17,26), PostType=2  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", 0 AS "I4", 0 AS "I5", -SUM(A."LCAmount") AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."InTransTypeID" IN (2,11,21,14,7,17,26)  
      AND A."PostType" = 2  
      AND (SUBSTR(A."AccountNo",1,6) IN ('008111','008121','008211','008221')  
        OR SUBSTR(A."AccountNo",1,5) IN ('00911','00921','00931','00411','00421','00621','00611')  
        OR SUBSTR(A."AccountNo",1,3) IN ('014','097'))  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
  
    UNION ALL  
  
    -- Sub 7: I6 - nhóm InTransTypeID (4,13,23,14), PostType=2 (không lọc theo AccountNo cụ thể trong join này nhưng vẫn có điều kiện account như trên)  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", 0 AS "I4", 0 AS "I5", -SUM(A."LCAmount") AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."InTransTypeID" IN (4,13,23,14)  
      AND A."PostType" = 2  
      AND (SUBSTR(A."AccountNo",1,6) IN ('008111','008121','008211','008221')  
        OR SUBSTR(A."AccountNo",1,5) IN ('00911','00921','00931','00411','00421','00621','00611')  
        OR SUBSTR(A."AccountNo",1,3) IN ('014','097'))  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
  
    UNION ALL  
  
    -- Sub 8: I6 - tài khoản 013, PostType=1  
    SELECT  
        A."ProjectNo" || '#' || EC."CateNo" || '#' || EC."ExpenseNo" AS "ItemID",  
        A."ProjectNo"                       AS "EntityID",  
        A."ProjectNo" || '#' || EC."CateNo" AS "ParentID",  
        MIN(A."ExpenseName")                AS "ItemName",  
        A."ProjectNo"                       AS "ProjectNo",  
        MIN(A."ProjectName")                AS "ProjectName",  
        EC."ExpenseNo"                      AS "ExpenseNo",  
        EC."CateNo"                         AS "ExpenseCateNo",  
        MIN(A."ExpenseName")                AS "ExpenseName",  
        A."ContractNo"                      AS "ContractNo",  
        MIN(A."ContractID")                 AS "ContractID",  
        A."VendorNo"                        AS "VendorNo",  
        A."PartnerNo"                       AS "PartnerNo",  
        A."ClearanceNo"                     AS "ClearanceNo",  
        0 AS "I2", 0 AS "I4", 0 AS "I5", -SUM(A."LCAmount") AS "I6"  
    FROM "glm_gl_books_v1" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."PostType" = 1  
      AND SUBSTR(A."AccountNo",1,3) IN ('013')  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
      -- + m1WhereBase  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
    HAVING SUM(A."LCAmount") != 0  
),  
  
glm094 AS (  
    SELECT A."ExpenseNo" AS en, A."ProjectNo" AS pn, SUM(A."LCAmount") AS s  
    FROM "glm_gl_books_v1" A  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND A."PostType" = 1  
      AND SUBSTR(A."AccountNo",1,3) = '094'  
      -- + m1WhereBase (áp dụng trực tiếp trên A, không qua EC)  
    GROUP BY A."ExpenseNo", A."ProjectNo"  
),  
  
cq AS (  
    SELECT "ContractID" AS cid, "VendorNo" AS vn, "ExpenseNo" AS en, SUM("LCAmount") AS amt  
    FROM "contract_quantity"  
    GROUP BY "ContractID", "VendorNo", "ExpenseNo"  
),  
  
ctid AS (  
    SELECT "ContractID" AS cid, MAX("DocumentType") AS dt  
    FROM "contract"  
    GROUP BY "ContractID"  
),  
  
ctno AS (  
    SELECT "ContractNo" AS cno, MIN("ContractID") AS cid, MAX("DocumentType") AS dt  
    FROM "contract"  
    GROUP BY "ContractNo"  
),  
  
classified AS (  
    SELECT l.*,  
        CASE  
            WHEN l."ClearanceNo" IS NULL AND l."ContractNo" IS NULL AND v."VendorName" IS NOT NULL THEN '6'  
            WHEN l."VendorNo" IS NULL AND l."ContractNo" IS NULL AND l."ClearanceNo" IS NULL THEN '5'  
            WHEN l."ContractNo" IS NULL AND l."ClearanceNo" IS NOT NULL THEN '4'  
            WHEN l."ClearanceNo" IS NULL AND l."ContractNo" IS NOT NULL AND p."PartnerName" IS NOT NULL THEN '2'  
            WHEN l."ClearanceNo" IS NULL AND l."ContractNo" IS NOT NULL AND v."VendorName" IS NOT NULL THEN '1'  
            WHEN l."ContractNo" IS NOT NULL AND l."ClearanceNo" IS NOT NULL AND v."VendorName" IS NOT NULL THEN '3'  
        END AS "TH",  
        v."VendorName" AS vname,  
        p."PartnerName" AS pname  
    FROM leaves l  
    LEFT JOIN "vendor"  v ON v."VendorNo"  = l."VendorNo"  
    LEFT JOIN "partner" p ON p."PartnerNo" = l."PartnerNo"  
),  
  
resolved AS (  
    SELECT  
        c."ItemID", c."EntityID", c."ParentID", c."ItemName", c."ProjectNo", c."ProjectName",  
        c."ExpenseNo", c."ExpenseCateNo", c."ExpenseName", c."ContractNo", c."ContractID",  
        c."VendorNo", c."PartnerNo", c."ClearanceNo", c."I2", c."I4", c."I5", c."I6", c."TH",  
        CASE c."TH"  
            WHEN '5' THEN :companyName  
            WHEN '4' THEN :companyName  
            WHEN '2' THEN c.pname  
            ELSE c.vname  
        END AS "ItemName1",  
        CASE c."TH"  
            WHEN '5' THEN g.s * SUM(CASE WHEN c."TH" = '5' THEN 1 ELSE 0 END)  
                                OVER (PARTITION BY c."ExpenseCateNo", c."ExpenseNo", c."ProjectNo")  
            WHEN '4' THEN cl."ClearanceAmount"  
            WHEN '3' THEN CASE WHEN cti.dt IN (1,3) THEN cq3.amt END  
            WHEN '1' THEN CASE WHEN ctn.dt IN (1,3) THEN cq1.amt END  
            WHEN '2' THEN CASE WHEN cti.dt IN (1,3) THEN cq2.amt END  
        END AS "I1"  
    FROM classified c  
    LEFT JOIN glm094 g       ON g.en = c."ExpenseNo" AND g.pn = c."ProjectNo"  
    LEFT JOIN "clearance" cl ON cl."ClearanceNo" = c."ClearanceNo"  
    LEFT JOIN cq cq3         ON cq3.cid = c."ContractID" AND cq3.vn = c."VendorNo"  AND cq3.en = c."ExpenseNo"  
    LEFT JOIN cq cq2         ON cq2.cid = c."ContractID" AND cq2.vn = c."PartnerNo" AND cq2.en = c."ExpenseNo"  
    LEFT JOIN ctno ctn       ON ctn.cno = c."ContractNo"  
    LEFT JOIN cq cq1         ON cq1.cid = ctn.cid AND cq1.vn = c."VendorNo" AND cq1.en = c."ExpenseNo"  
    LEFT JOIN ctid cti       ON cti.cid = c."ContractID"  
)  
  
SELECT * FROM resolved;
```

sau khi sửa:

```sql
  
WITH  
gl_grp AS (  
    /*+ MATERIALIZE */  
    -- 1 LẦN SCAN duy nhất thay cho 8 lần. Điều kiện của mỗi nhánh gốc    -- được đưa vào CASE WHEN riêng, KHÔNG đổi ngữ nghĩa, KHÔNG gộp cột.    SELECT  
        A."ProjectNo"     AS "ProjectNo",  
        EC."ExpenseNo"    AS "ExpenseNo",  
        EC."CateNo"       AS "ExpenseCateNo",  
        A."ContractNo"    AS "ContractNo",  
        A."VendorNo"      AS "VendorNo",  
        A."ClearanceNo"   AS "ClearanceNo",  
        A."PartnerNo"     AS "PartnerNo",  
        MIN(A."ExpenseName")  AS "ExpenseName",  
        MIN(A."ProjectName")  AS "ProjectName",  
        MIN(A."ContractID")   AS "ContractID",  
  
        -- B1 (I2, gốc: sub-select 1)  
        SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2431','2412')  
                  AND A."InTransTypeID" NOT IN (33,34,35)  
                 THEN A."LCAmount" ELSE 0 END) AS b1_amt,  
  
        -- B2 (I4 phần 1, gốc: sub-select 2)  
        SUM(CASE WHEN A."InTransTypeID" IN (2,11,21,7,17,26) AND A."PostType" = 2  
                  AND (SUBSTR(A."AccountNo",1,6) IN ('008111','008121','008211','008221')  
                    OR SUBSTR(A."AccountNo",1,5) IN ('00911','00921','00931','00411','00421','00621','00611')  
                    OR SUBSTR(A."AccountNo",1,3) IN ('014','097'))  
                 THEN A."LCAmount" ELSE 0 END) AS b2_amt,  
  
        -- B3 (I4 phần 2 - 013, gốc: sub-select 3)  
        SUM(CASE WHEN A."PostType" = 2 AND SUBSTR(A."AccountNo",1,3) IN ('013')  
                 THEN A."LCAmount" ELSE 0 END) AS b3_amt,  
  
        -- B4 (I5 phần 1, gốc: sub-select 4)  
        SUM(CASE WHEN A."InTransTypeID" IN (3,12,22,8,18,27,30,38,36) AND A."PostType" = 2  
                  AND (SUBSTR(A."AccountNo",1,6) IN ('008112','008122','008212','008222')  
                    OR SUBSTR(A."AccountNo",1,5) IN ('00912','00922','00932','00412','00422','00622','00612')  
                    OR SUBSTR(A."AccountNo",1,3) IN ('012','014','013','097'))  
                 THEN A."LCAmount" ELSE 0 END) AS b4_amt,  
  
        -- B5 (I5 phần 2 - 013, GIỚI HẠN fromDate..toDate, gốc: sub-select 5)  
        -- Lưu ý: điều kiện PostDate <= toDateIso đã có ở WHERE ngoài (áp dụng chung mọi nhánh),        -- ở đây chỉ cần bổ sung cận dưới fromDateIso cho đúng nhánh gốc.        SUM(CASE WHEN A."PostDate" >= TO_DATE(:fromDateIso,'YYYY-MM-DD')  
                  AND A."PostType" = 2 AND SUBSTR(A."AccountNo",1,3) IN ('013')  
                 THEN A."LCAmount" ELSE 0 END) AS b5_amt,  
  
        -- B6 (I6 phần 1, gốc: sub-select 6)  
        SUM(CASE WHEN A."InTransTypeID" IN (2,11,21,14,7,17,26) AND A."PostType" = 2  
                  AND (SUBSTR(A."AccountNo",1,6) IN ('008111','008121','008211','008221')  
                    OR SUBSTR(A."AccountNo",1,5) IN ('00911','00921','00931','00411','00421','00621','00611')  
                    OR SUBSTR(A."AccountNo",1,3) IN ('014','097'))  
                 THEN A."LCAmount" ELSE 0 END) AS b6_amt,  
  
        -- B7 (I6 phần 2, gốc: sub-select 7)  
        SUM(CASE WHEN A."InTransTypeID" IN (4,13,23,14) AND A."PostType" = 2  
                  AND (SUBSTR(A."AccountNo",1,6) IN ('008111','008121','008211','008221')  
                    OR SUBSTR(A."AccountNo",1,5) IN ('00911','00921','00931','00411','00421','00621','00611')  
                    OR SUBSTR(A."AccountNo",1,3) IN ('014','097'))  
                 THEN A."LCAmount" ELSE 0 END) AS b7_amt,  
  
        -- B8 (I6 phần 3 - 013, PostType=1, gốc: sub-select 8)  
        SUM(CASE WHEN A."PostType" = 1 AND SUBSTR(A."AccountNo",1,3) IN ('013')  
                 THEN A."LCAmount" ELSE 0 END) AS b8_amt  
  
    FROM "glm_gl_books" A  
    INNER JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD')  
      AND EC."CateNo" IN ('2005203','2005204','2005205','2005206','2005207','2005208','2005209')  
--       <if test="filter.companyNoPrefix != null">  
--       AND SUBSTR(A."CompanyNo", 1, #{filter.companyPrefixLen}) = #{filter.companyNoPrefix}  
--       </if>  
--       <if test="filter.projectNo != null">  
--       AND A."ProjectNo" LIKE #{filter.projectNo} || '%'  
--       </if>  
--       <if test="filter.provinceNo != null">  
--       AND A."ProvinceNo" = #{filter.provinceNo}  
--       </if>  
--       <if test="filter.communeNo != null">  
--       AND A."CommuneNo" = #{filter.communeNo}  
--       </if>  
--       <if test="filter.projectId != null">  
--       AND A."ProjectID" = #{filter.projectId}  
--       </if>  
      AND A."ProjectNo" IS NOT NULL AND EC."ExpenseNo" IS NOT NULL  
    GROUP BY A."ProjectNo", EC."ExpenseNo", EC."CateNo", A."ContractNo", A."VendorNo", A."ClearanceNo", A."PartnerNo"  
),  
  
leaves AS (  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo" AS "ItemID",  
           g."ProjectNo" AS "EntityID",  
           g."ProjectNo" || '#' || g."ExpenseCateNo" AS "ParentID",  
           g."ExpenseName" AS "ItemName",  
           g."ProjectNo" AS "ProjectNo",  
           g."ProjectName" AS "ProjectName",  
           g."ExpenseNo" AS "ExpenseNo",  
           g."ExpenseCateNo" AS "ExpenseCateNo",  
           g."ExpenseName" AS "ExpenseName",  
           g."ContractNo" AS "ContractNo",  
           g."ContractID" AS "ContractID",  
           g."VendorNo" AS "VendorNo",  
           g."PartnerNo" AS "PartnerNo",  
           g."ClearanceNo" AS "ClearanceNo",  
           g.b1_amt AS "I2", 0 AS "I4", 0 AS "I5", 0 AS "I6"  
    FROM gl_grp g WHERE g.b1_amt > 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, -g.b2_amt, 0, 0  
    FROM gl_grp g WHERE g.b2_amt != 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, -g.b3_amt, 0, 0  
    FROM gl_grp g WHERE g.b3_amt != 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, 0, -g.b4_amt, 0  
    FROM gl_grp g WHERE g.b4_amt != 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, 0, -g.b5_amt, 0  
    FROM gl_grp g WHERE g.b5_amt != 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, 0, 0, -g.b6_amt  
    FROM gl_grp g WHERE g.b6_amt != 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, 0, 0, -g.b7_amt  
    FROM gl_grp g WHERE g.b7_amt != 0  
  
    UNION ALL  
    SELECT g."ProjectNo" || '#' || g."ExpenseCateNo" || '#' || g."ExpenseNo", g."ProjectNo",  
           g."ProjectNo" || '#' || g."ExpenseCateNo", g."ExpenseName", g."ProjectNo", g."ProjectName",  
           g."ExpenseNo", g."ExpenseCateNo", g."ExpenseName", g."ContractNo", g."ContractID",  
           g."VendorNo", g."PartnerNo", g."ClearanceNo",  
           0, 0, 0, -g.b8_amt  
    FROM gl_grp g WHERE g.b8_amt != 0  
),  
  
glm094 AS (  
    SELECT A."ExpenseNo" AS en, A."ProjectNo" AS pn, SUM(A."LCAmount") AS s  
    FROM "glm_gl_books_v1" A  
    WHERE A."PostDate" <= TO_DATE(:toDateIso,'YYYY-MM-DD') AND A."PostType" = 1  
      AND SUBSTR(A."AccountNo",1,3) = '094'  
--       <include refid="m1WhereBase"/>  
    GROUP BY A."ExpenseNo", A."ProjectNo"  
),  
cq AS (  
    SELECT "ContractID" AS cid, "VendorNo" AS vn, "ExpenseNo" AS en, SUM("LCAmount") AS amt  
    FROM "contract_quantity" GROUP BY "ContractID", "VendorNo", "ExpenseNo"  
),  
ctid AS (  
    SELECT "ContractID" AS cid, MAX("DocumentType") AS dt FROM "contract" GROUP BY "ContractID"  
),  
ctno AS (  
    SELECT "ContractNo" AS cno, MIN("ContractID") AS cid, MAX("DocumentType") AS dt  
    FROM "contract" GROUP BY "ContractNo"  
),  
  
classified AS (  
    SELECT l.*,  
        CASE  
            WHEN l."ClearanceNo" IS NULL AND l."ContractNo" IS NULL AND v."VendorName" IS NOT NULL THEN '6'  
            WHEN l."VendorNo" IS NULL AND l."ContractNo" IS NULL AND l."ClearanceNo" IS NULL THEN '5'  
            WHEN l."ContractNo" IS NULL AND l."ClearanceNo" IS NOT NULL THEN '4'  
            WHEN l."ClearanceNo" IS NULL AND l."ContractNo" IS NOT NULL AND p."PartnerName" IS NOT NULL THEN '2'  
            WHEN l."ClearanceNo" IS NULL AND l."ContractNo" IS NOT NULL AND v."VendorName" IS NOT NULL THEN '1'  
            WHEN l."ContractNo" IS NOT NULL AND l."ClearanceNo" IS NOT NULL AND v."VendorName" IS NOT NULL THEN '3'  
        END AS "TH",  
        v."VendorName" AS vname, p."PartnerName" AS pname  
    FROM leaves l  
    LEFT JOIN "vendor"  v ON v."VendorNo"  = l."VendorNo"  
    LEFT JOIN "partner" p ON p."PartnerNo" = l."PartnerNo"  
),  
resolved AS (  
    SELECT c."ItemID", c."EntityID", c."ParentID", c."ItemName", c."ProjectNo", c."ProjectName",  
           c."ExpenseNo", c."ExpenseCateNo", c."ExpenseName", c."ContractNo", c."ContractID",  
           c."VendorNo", c."PartnerNo", c."ClearanceNo", c."I2", c."I4", c."I5", c."I6", c."TH",  
        CASE c."TH" WHEN '5' THEN :companyName WHEN '4' THEN :companyName  
                    WHEN '2' THEN c.pname ELSE c.vname END AS "ItemName1",  
        CASE c."TH"  
            WHEN '5' THEN g.s * SUM(CASE WHEN c."TH" = '5' THEN 1 ELSE 0 END)  
                                OVER (PARTITION BY c."ExpenseCateNo", c."ExpenseNo", c."ProjectNo")  
            WHEN '4' THEN cl."ClearanceAmount"  
            WHEN '3' THEN CASE WHEN cti.dt IN (1,3) THEN cq3.amt END  
            WHEN '1' THEN CASE WHEN ctn.dt IN (1,3) THEN cq1.amt END  
            WHEN '2' THEN CASE WHEN cti.dt IN (1,3) THEN cq2.amt END  
        END AS "I1"  
    FROM classified c  
    LEFT JOIN glm094 g       ON g.en = c."ExpenseNo" AND g.pn = c."ProjectNo"  
    LEFT JOIN "clearance" cl ON cl."ClearanceNo" = c."ClearanceNo"  
    LEFT JOIN cq cq3         ON cq3.cid = c."ContractID" AND cq3.vn = c."VendorNo"  AND cq3.en = c."ExpenseNo"  
    LEFT JOIN cq cq2         ON cq2.cid = c."ContractID" AND cq2.vn = c."PartnerNo" AND cq2.en = c."ExpenseNo"  
    LEFT JOIN ctno ctn       ON ctn.cno = c."ContractNo"  
    LEFT JOIN cq cq1         ON cq1.cid = ctn.cid AND cq1.vn = c."VendorNo" AND cq1.en = c."ExpenseNo"  
    LEFT JOIN ctid cti       ON cti.cid = c."ContractID"  
)  
SELECT * FROM resolved;
```


Với dữ liệu bảng glm_gl_books 1 triệu bản ghi cost giảm từ 245K -> xuống 54552

# 7. B209Leaf
	Làm tương tự với `6. B209M1Leaf` vì Cùng pattern với B209M1

# 8.findMb07QtdaRows

> [!NOTE]
> **Lý do**
> Đang scan bảng `glm_gl_books` 7 lần trong 1 query
> 
>  **Giải pháp**
> 1 LẦN SCAN duy nhất thay cho 7 lần. KHÔNG đổi ngữ nghĩa, KHÔNG gộp cột để giảm tải việc roud-trip xuống db => sau đó giữ nguyên nghiệp vụ cũ

Hiện trạng:
```sql
SELECT "itemId","parentId","level","itemName","itemName1","itemName2",  
       SUM("i1") AS "i1", SUM("i2") AS "i2"  
FROM (  
    -- 1. Contracts: A#01#VendorNo#ContractNo  
    SELECT 'A#01#' || A."VendorNo" || '#' || A."ContractNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."VendorName") AS "itemName1", MAX(A."ContractName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "contract" CT ON A."ContractID"=CT."ContractID" AND CT."DocumentType"=1  
    WHERE A."VendorNo"!='' AND A."ContractNo"!='' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  -- nếu filter.companyNoPrefix != null  
      AND A."CompanyID" = :companyId                                          -- nếu filter.companyId != null  
      AND A."ProjectID" = :projectId                                          -- nếu filter.projectId != null  
      AND A."VendorID" = :vendorId                                            -- nếu filter.vendorId != null  
      AND A."PartnerID" = :partnerId                                          -- nếu filter.partnerId != null  
    GROUP BY A."VendorNo", A."ContractNo"  
  
    UNION ALL  
  
    -- 2. Clearance self-cost: A#02#ClearanceNo (I1 loại trừ QLDA)  
    SELECT 'A#02#' || A."ClearanceNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."CompanyName") AS "itemName1", MAX(A."ClearanceName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "expense" E ON A."ExpenseID"=E."ExpenseID"  
    WHERE A."ClearanceNo"!='' AND E."isSelfpccosts"=1 AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  
      AND A."CompanyID" = :companyId  
      AND A."ProjectID" = :projectId  
      AND A."VendorID" = :vendorId  
      AND A."PartnerID" = :partnerId  
    GROUP BY A."ClearanceNo"  
  
    UNION ALL  
  
    -- 3. Clearance other-cost: A#03#ClearanceNo (I1 loại trừ QLDA)  
    SELECT 'A#03#' || A."ClearanceNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", MAX(A."ClearanceName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "expense" E ON A."ExpenseID"=E."ExpenseID"  
    WHERE A."ClearanceNo"!='' AND E."isSelfpccosts"!=1 AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  
      AND A."CompanyID" = :companyId  
      AND A."ProjectID" = :projectId  
      AND A."VendorID" = :vendorId  
      AND A."PartnerID" = :partnerId  
    GROUP BY A."ClearanceNo"  
  
    UNION ALL  
  
    -- 4a. Partner QLDA: A#04#1#PartnerNo  
    SELECT 'A#04#1#' || A."PartnerNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", 'Chi phí QLDA' AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "expense" E ON A."ExpenseID"=E."ExpenseID"  
    WHERE A."VendorNo"='' AND A."ContractNo"='' AND A."ClearanceNo"='' AND A."PartnerNo"!=''  
      AND SUBSTR(A."ExpenseNo",1,6)='119803' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  
      AND A."CompanyID" = :companyId  
      AND A."ProjectID" = :projectId  
      AND A."VendorID" = :vendorId  
      AND A."PartnerID" = :partnerId  
    GROUP BY A."PartnerNo"  
  
    UNION ALL  
  
    -- 4b. Partner other: A#04#2#PartnerNo#ExpenseNo (I1 loại trừ QLDA)  
    SELECT 'A#04#2#' || A."PartnerNo" || '#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(A."PartnerName") AS "itemName1", MAX(E."ExpenseName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "expense" E ON A."ExpenseID"=E."ExpenseID"  
    WHERE A."VendorNo"='' AND A."ContractNo"='' AND A."ClearanceNo"='' AND A."PartnerNo"!=''  
      AND SUBSTR(A."ExpenseNo",1,6)!='119803' AND SUBSTR(A."ExpenseNo",1,3)='201'  
      AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  
      AND A."CompanyID" = :companyId  
      AND A."ProjectID" = :projectId  
      AND A."VendorID" = :vendorId  
      AND A."PartnerID" = :partnerId  
    GROUP BY A."PartnerNo", A."ExpenseNo"  
  
    UNION ALL  
  
    -- 5a. Unidentified QLDA: A#05#1#ExpenseNo  
    SELECT 'A#05#1#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           'Đối tượng chưa xác định' AS "itemName1", 'Chi phí quản lý dự án' AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "expense" E ON A."ExpenseID"=E."ExpenseID"  
    WHERE A."ClearanceNo"='' AND A."ContractNo"='' AND A."VendorNo"='' AND A."PartnerNo"=''  
      AND SUBSTR(A."ExpenseNo",1,6)='119803' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  
      AND A."CompanyID" = :companyId  
      AND A."ProjectID" = :projectId  
      AND A."VendorID" = :vendorId  
      AND A."PartnerID" = :partnerId  
    GROUP BY A."ExpenseNo"  
  
    UNION ALL  
  
    -- 5b. Unidentified other: A#05#2#ExpenseNo (I1 loại trừ QLDA)  
    SELECT 'A#05#2#' || A."ExpenseNo" AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           'Đối tượng chưa xác định' AS "itemName1", MAX(E."ExpenseName") AS "itemName2",  
           SUM(CASE WHEN SUBSTR(A."ExpenseNo",1,6)='119803' THEN 0  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=1 THEN A."LCAmount"  
                    WHEN SUBSTR(A."AccountNo",1,4)='2412' AND A."PostType"=2 THEN -A."LCAmount" ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
                       OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1))  
                    THEN A."LCAmount"*(3-2*A."PostType") ELSE 0 END) AS "i2"  
    FROM "glm_gl_books" A INNER JOIN "expense" E ON A."ExpenseID"=E."ExpenseID"  
    WHERE A."ClearanceNo"='' AND A."ContractNo"='' AND A."VendorNo"='' AND A."PartnerNo"=''  
      AND SUBSTR(A."ExpenseNo",1,6)!='119803' AND A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  
      AND A."CompanyID" = :companyId  
      AND A."ProjectID" = :projectId  
      AND A."VendorID" = :vendorId  
      AND A."PartnerID" = :partnerId  
    GROUP BY A."ExpenseNo"  
) agg  
GROUP BY "itemId","parentId","level","itemName","itemName1","itemName2"  
ORDER BY "itemId";
```


sau:
```sql
  
WITH gl_grp AS (  
    SELECT /*+ MATERIALIZE */  
        A."VendorNo"      AS vendorNo,  
        A."ContractNo"    AS contractNo,  
        A."ContractID"    AS contractId,  
        A."ClearanceNo"   AS clearanceNo,  
        A."PartnerNo"     AS partnerNo,  
        A."ExpenseNo"     AS expenseNo,  
        A."ExpenseID"     AS expenseId,  
        A."AccountNo"     AS accountNo,        
        A."PostType"      AS postType,         
        A."LCAmount"      AS lcAmount,        
        A."VendorName"    AS vendorName,       
        A."ContractName"  AS contractName,  
        A."CompanyName"   AS companyName,  
        A."ClearanceName" AS clearanceName,  
        A."PartnerName"   AS partnerName  
    FROM "glm_gl_books_v1" A  
    WHERE A."InTransTypeID" NOT IN (33,34,35)  
      AND EXTRACT(YEAR FROM A."PostDate") <= :year  
      AND ((SUBSTR(A."AccountNo",1,4)='2412')  
        OR ((SUBSTR(A."AccountNo",1,3) IN ('006','007','008','009','011','012','013','Z13') AND A."PostType"=2)  
         OR (SUBSTR(A."AccountNo",1,3)='005' AND A."PostType"=1)))  
      AND SUBSTR(A."CompanyNo",1,LENGTH(:companyNoPrefix)) = :companyNoPrefix  -- nếu có filter  
      AND A."CompanyID" = :companyId    -- nếu có filter  
      AND A."ProjectID" = :projectId    -- nếu có filter  
      AND A."VendorID"  = :vendorId     -- nếu có filter  
      AND A."PartnerID" = :partnerId    -- nếu có filter  
)  
SELECT "itemId","parentId","level","itemName","itemName1","itemName2",  
       SUM("i1") AS "i1", SUM("i2") AS "i2"  
FROM (  
  
    SELECT 'A#01#' || G.vendorNo || '#' || G.contractNo AS "itemId",  
           '' AS "parentId", 1 AS "level", NULL AS "itemName",  
           MAX(G.vendorName) AS "itemName1", MAX(G.contractName) AS "itemName2",  
           SUM(CASE WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END) AS "i1",  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END) AS "i2"  
    FROM gl_grp G  
    INNER JOIN "contract" CT ON G.contractId = CT."ContractID" AND CT."DocumentType" = 1  
    WHERE G.vendorNo != '' AND G.contractNo != ''  
    GROUP BY G.vendorNo, G.contractNo  
  
    UNION ALL  
  
    SELECT 'A#02#' || G.clearanceNo,  
           '', 1, NULL,  
           MAX(G.companyName), MAX(G.clearanceName),  
           SUM(CASE WHEN SUBSTR(G.expenseNo,1,6)='119803' THEN 0  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END),  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END)  
    FROM gl_grp G  
    INNER JOIN "expense" E ON G.expenseId = E."ExpenseID"  
    WHERE G.clearanceNo != '' AND E."isSelfpccosts" = 1  
    GROUP BY G.clearanceNo  
  
    UNION ALL  
  
    SELECT 'A#03#' || G.clearanceNo,  
           '', 1, NULL,  
           MAX(G.partnerName), MAX(G.clearanceName),  
           SUM(CASE WHEN SUBSTR(G.expenseNo,1,6)='119803' THEN 0  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END),  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END)  
    FROM gl_grp G  
    INNER JOIN "expense" E ON G.expenseId = E."ExpenseID"  
    WHERE G.clearanceNo != '' AND E."isSelfpccosts" != 1  
    GROUP BY G.clearanceNo  
  
    UNION ALL  
  
    SELECT 'A#04#1#' || G.partnerNo,  
           '', 1, NULL,  
           MAX(G.partnerName), 'Chi phí QLDA',  
           SUM(CASE WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END),  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END)  
    FROM gl_grp G  
    INNER JOIN "expense" E ON G.expenseId = E."ExpenseID"  
    WHERE G.vendorNo='' AND G.contractNo='' AND G.clearanceNo='' AND G.partnerNo != ''  
      AND SUBSTR(G.expenseNo,1,6)='119803'  
    GROUP BY G.partnerNo  
  
    UNION ALL  
  
    SELECT 'A#04#2#' || G.partnerNo || '#' || G.expenseNo,  
           '', 1, NULL,  
           MAX(G.partnerName), MAX(E."ExpenseName"),  
           SUM(CASE WHEN SUBSTR(G.expenseNo,1,6)='119803' THEN 0  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END),  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END)  
    FROM gl_grp G  
    INNER JOIN "expense" E ON G.expenseId = E."ExpenseID"  
    WHERE G.vendorNo='' AND G.contractNo='' AND G.clearanceNo='' AND G.partnerNo != ''  
      AND SUBSTR(G.expenseNo,1,6) != '119803' AND SUBSTR(G.expenseNo,1,3)='201'  
    GROUP BY G.partnerNo, G.expenseNo  
  
    UNION ALL  
  
    SELECT 'A#05#1#' || G.expenseNo,  
           '', 1, NULL,  
           'Đối tượng chưa xác định', 'Chi phí quản lý dự án',  
           SUM(CASE WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END),  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END)  
    FROM gl_grp G  
    INNER JOIN "expense" E ON G.expenseId = E."ExpenseID"  
    WHERE G.clearanceNo='' AND G.contractNo='' AND G.vendorNo='' AND G.partnerNo=''  
      AND SUBSTR(G.expenseNo,1,6)='119803'  
    GROUP BY G.expenseNo  
  
    UNION ALL  
  
    SELECT 'A#05#2#' || G.expenseNo,  
           '', 1, NULL,  
           'Đối tượng chưa xác định', MAX(E."ExpenseName"),  
           SUM(CASE WHEN SUBSTR(G.expenseNo,1,6)='119803' THEN 0  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=1 THEN G.lcAmount  
                    WHEN SUBSTR(G.accountNo,1,4)='2412' AND G.postType=2 THEN -G.lcAmount ELSE 0 END),  
           SUM(CASE WHEN ((SUBSTR(G.accountNo,1,3) IN ('006','007','008','009','011','012','013','Z13') AND G.postType=2)  
                       OR (SUBSTR(G.accountNo,1,3)='005' AND G.postType=1))  
                    THEN G.lcAmount*(3-2*G.postType) ELSE 0 END)  
    FROM gl_grp G  
    INNER JOIN "expense" E ON G.expenseId = E."ExpenseID"  
    WHERE G.clearanceNo='' AND G.contractNo='' AND G.vendorNo='' AND G.partnerNo=''  
      AND SUBSTR(G.expenseNo,1,6) != '119803'  
    GROUP BY G.expenseNo  
) agg  
GROUP BY "itemId","parentId","level","itemName","itemName1","itemName2"  
ORDER BY "itemId";
```

Với dữ liệu bảng glm_gl_books 1 triệu bản ghi cost giảm từ 81968 -> xuống 27292


# 9.findB222Leaf

> [!NOTE]
> **Lý do**
> Đang scan bảng `glm_gl_books` 5 lần trong 1 query
> 
>  **Giải pháp**
> 1 LẦN SCAN duy nhất thay cho 5 lần. KHÔNG đổi ngữ nghĩa, KHÔNG gộp cột để giảm tải việc roud-trip xuống db => sau đó giữ nguyên nghiệp vụ cũ

Hiện trạng:
```sql
-- ===== TH1: hợp đồng (key = ContractID) =====  
SELECT  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END || '#ContractID' || TO_CHAR(A."ContractID") AS "itemId",  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END AS "parentId",  
    2 AS "level",  
    MIN(A."ContractName") AS "itemName", MIN(A."TabmisNo") AS "tabmisNo",  
  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('091','092') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i1,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('093','094') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i2,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i5,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=2  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i7,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i6,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=2  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i8,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i9n,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=2  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i9c,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i10n,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=2  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i10c,  
    SUM(CASE WHEN A."PostType"=2 AND (  
            (SUBSTR(A."AccountNo",1,4)='2412' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
         OR (SUBSTR(A."AccountNo",1,4)='2432' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')))  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i11,  
    SUM(CASE WHEN A."PostType"=2 AND (  
            (SUBSTR(A."AccountNo",1,4)='2412' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
         OR (SUBSTR(A."AccountNo",1,4)='2432' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')))  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i12,  
  
    (SELECT COALESCE(SUM(CQ."Quantity"),0) FROM "contract_quantity" CQ WHERE CQ."ContractID" = A."ContractID") AS i3,  
    (SELECT COALESCE(SUM(CQ."LCAmount"),0) FROM "contract_quantity" CQ WHERE CQ."ContractID" = A."ContractID") AS i4  
FROM "glm_gl_books" A  
JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
WHERE EC."CateNo" IN ('201520101','201520102','201520103','201520104','201520105','201520108')  
  AND ((A."ContractID" IS NOT NULL AND A."ContractID" > 0)  
    OR (A."VendorID" IS NOT NULL AND A."VendorID" > 0)  
    OR (A."PartnerID" IS NOT NULL AND A."PartnerID" > 0))  
  AND A."ProjectID" = :projectId          -- nếu filter.projectId != null  
  AND A."ProjectNo" = :projectNo          -- nếu filter.projectNo != null  
  AND A."ProvinceNo" = :provinceNo        -- nếu filter.provinceNo != null  
  AND A."DistrictNo" = :districtNo        -- nếu filter.districtNo != null  
  AND A."CommuneNo" = :communeNo          -- nếu filter.communeNo != null  
  -- === companyCondition (chọn đúng 1 nhánh theo kind) ===  AND A."CompanyManagementLevel" >= :managementLevel                              -- kind = MANAGEMENT_LEVEL_GTE  
  -- AND A."CompanyID" IN (:companyIds)                                           -- kind = ID_IN  -- AND A."CompanyID" = :companyId                                              -- kind = ID_EQ  -- AND SUBSTR(A."CompanyNo",1,:companyNoPrefixLength) = :companyNoPrefix       -- kind = NO_PREFIX  -- ========================================================GROUP BY  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END,  
    A."ContractID"  
  
UNION ALL  
  
-- ===== TH2: khoản-chi không hợp đồng/nghiệm thu (key = ExpenseID), A5/A8 → Level 1 leaf =====  
SELECT  
    CASE WHEN (CASE EC."CateNo"  
                    WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
                    WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
                    WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
                    WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
               END) IN ('A5','A8')  
         THEN (CASE EC."CateNo"  
                    WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
                    WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
                    WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
                    WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
               END)  
         ELSE (CASE EC."CateNo"  
                    WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
                    WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
                    WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
                    WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
               END) || '#ExpenseID' || TO_CHAR(A."ExpenseID")  
    END AS "itemId",  
    CASE WHEN (CASE EC."CateNo"  
                    WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
                    WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
                    WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
                    WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
               END) IN ('A5','A8')  
         THEN CAST(NULL AS VARCHAR2(1))  
         ELSE (CASE EC."CateNo"  
                    WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
                    WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
                    WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
                    WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
               END)  
    END AS "parentId",  
    CASE WHEN (CASE EC."CateNo"  
                    WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
                    WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
                    WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
                    WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
               END) IN ('A5','A8')  
         THEN 1 ELSE 2  
    END AS "level",  
    MIN(A."ExpenseName") AS "itemName", MIN(A."TabmisNo") AS "tabmisNo",  
  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('091','092') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i1,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('093','094') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i2,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i5,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=2  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i7,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i6,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=2  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i8,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i9n,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=2  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i9c,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i10n,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=2  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i10c,  
    SUM(CASE WHEN A."PostType"=2 AND (  
            (SUBSTR(A."AccountNo",1,4)='2412' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
         OR (SUBSTR(A."AccountNo",1,4)='2432' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')))  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i11,  
    SUM(CASE WHEN A."PostType"=2 AND (  
            (SUBSTR(A."AccountNo",1,4)='2412' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
         OR (SUBSTR(A."AccountNo",1,4)='2432' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')))  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i12,  
  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='094' AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i3,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='094' AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i4  
FROM "glm_gl_books" A  
JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
WHERE EC."CateNo" IN ('201520101','201520102','201520103','201520104','201520105','201520106','201520108','201520109')  
  AND (A."ContractID" IS NULL OR A."ContractID" = 0) AND (A."ClearanceID" IS NULL OR A."ClearanceID" = 0)  
  AND A."ProjectID" = :projectId  
  AND A."ProjectNo" = :projectNo  
  AND A."ProvinceNo" = :provinceNo  
  AND A."DistrictNo" = :districtNo  
  AND A."CommuneNo" = :communeNo  
  AND A."CompanyManagementLevel" >= :managementLevel  
  -- (hoặc 1 trong 3 nhánh companyCondition khác, xem TH1)  
GROUP BY  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END,  
    A."ExpenseID"  
  
UNION ALL  
  
-- ===== TH3a: nghiệm thu — cột kế toán (key = A2#ClearanceID<v>), I3/I4=0 =====  
SELECT  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END || '#ClearanceID' || TO_CHAR(A."ClearanceID") AS "itemId",  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END AS "parentId",  
    2 AS "level",  
    MIN(A."ClearanceName") AS "itemName", MIN(A."TabmisNo") AS "tabmisNo",  
  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('091','092') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i1,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3) IN ('093','094') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i2,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i5,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=2  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i7,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i6,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,3)='073' AND A."PostType"=2  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i8,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=1  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i9n,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=2  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i9c,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=1  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN A."LCAmount" ELSE 0 END) AS i10n,  
    SUM(CASE WHEN SUBSTR(A."AccountNo",1,4) IN ('2412','2431') AND A."PostType"=2  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (35,34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i10c,  
    SUM(CASE WHEN A."PostType"=2 AND (  
            (SUBSTR(A."AccountNo",1,4)='2412' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
         OR (SUBSTR(A."AccountNo",1,4)='2432' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')))  
        AND A."PostDate" >= :fromDate AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i11,  
    SUM(CASE WHEN A."PostType"=2 AND (  
            (SUBSTR(A."AccountNo",1,4)='2412' AND SUBSTR(A."CoAccountNo",1,3) IN ('211','213'))  
         OR (SUBSTR(A."AccountNo",1,4)='2432' AND SUBSTR(A."CoAccountNo",1,3) IN ('343','812','211','213')))  
        AND A."PostDate" <= :toDate  
        AND (A."InTransTypeID" IS NULL OR A."InTransTypeID" NOT IN (34,33,32))  
        THEN -A."LCAmount" ELSE 0 END) AS i12,  
  
    0 AS i3, 0 AS i4  
FROM "glm_gl_books" A  
JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
WHERE EC."CateNo" = '201520102'  
  AND (A."ContractID" IS NULL OR A."ContractID" = 0) AND (A."ClearanceID" IS NOT NULL AND A."ClearanceID" > 0)  
  AND A."ProjectID" = :projectId  
  AND A."ProjectNo" = :projectNo  
  AND A."ProvinceNo" = :provinceNo  
  AND A."DistrictNo" = :districtNo  
  AND A."CommuneNo" = :communeNo  
  AND A."CompanyManagementLevel" >= :managementLevel  
  -- (hoặc 1 trong 3 nhánh companyCondition khác, xem TH1)  
GROUP BY  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END,  
    A."ClearanceID"  
  
UNION ALL  
  
-- ===== TH3b: nghiệm thu — I3/I4 từ clearance_quantity (key = A2#<v>, KHÔNG có 'ClearanceID') =====  
SELECT  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END || '#' || TO_CHAR(A."ClearanceID") AS "itemId",  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END AS "parentId",  
    2 AS "level",  
    (SELECT MIN(CQ."ClearanceName") FROM "clearance_quantity" CQ WHERE CQ."ClearanceID" = A."ClearanceID") AS "itemName",  
    MIN(A."TabmisNo") AS "tabmisNo",  
    0 AS i1, 0 AS i2, 0 AS i5, 0 AS i7, 0 AS i6, 0 AS i8,  
    0 AS i9n, 0 AS i9c, 0 AS i10n, 0 AS i10c, 0 AS i11, 0 AS i12,  
    (SELECT COALESCE(SUM(CQ."LCAmount"),0) FROM "clearance_quantity" CQ WHERE CQ."ClearanceID" = A."ClearanceID") AS i3,  
    (SELECT COALESCE(SUM(CQ."LCAmount"),0) FROM "clearance_quantity" CQ WHERE CQ."ClearanceID" = A."ClearanceID") AS i4  
FROM "glm_gl_books" A  
JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"  
WHERE EC."CateNo" = '201520102'  
  AND (A."ContractID" IS NULL OR A."ContractID" = 0) AND (A."ClearanceID" IS NOT NULL AND A."ClearanceID" > 0)  
  AND A."ProjectID" = :projectId  
  AND A."ProjectNo" = :projectNo  
  AND A."ProvinceNo" = :provinceNo  
  AND A."DistrictNo" = :districtNo  
  AND A."CommuneNo" = :communeNo  
  AND A."CompanyManagementLevel" >= :managementLevel  
  -- (hoặc 1 trong 3 nhánh companyCondition khác, xem TH1)  
GROUP BY  
    CASE EC."CateNo"  
        WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'  
        WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'  
        WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'  
        WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'  
    END,  
    A."ClearanceID";
```


sau:
```sql
WITH gl_grp AS (
    SELECT /*+ MATERIALIZE */
        CASE EC."CateNo"
            WHEN '201520101' THEN 'A1' WHEN '201520102' THEN 'A2'
            WHEN '201520103' THEN 'A3' WHEN '201520104' THEN 'A4'
            WHEN '201520106' THEN 'A5' WHEN '201520105' THEN 'A6'
            WHEN '201520108' THEN 'A7' WHEN '201520109' THEN 'A8'
        END           AS cate,
        A."ContractID"    AS contractId,
        A."ExpenseID"     AS expenseId,
        A."ClearanceID"   AS clearanceId,
        A."VendorID"      AS vendorId,
        A."PartnerID"     AS partnerId,
        A."AccountNo"     AS accountNo,
        A."CoAccountNo"   AS coAccountNo,
        A."PostType"      AS postType,
        A."PostDate"      AS postDate,
        A."InTransTypeID" AS inTransTypeId,
        A."LCAmount"      AS lcAmount,
        A."ContractName"  AS contractName,
        A."ExpenseName"   AS expenseName,
        A."ClearanceName" AS clearanceName,
        A."TabmisNo"      AS tabmisNo
    FROM "glm_gl_books" A
    JOIN "expense_cate" EC ON A."ExpenseID" = EC."ExpenseID"
    WHERE EC."CateNo" IN ('201520101','201520102','201520103','201520104','201520105','201520106','201520108','201520109')
      AND A."ProjectID" = :projectId          -- nếu filter.projectId != null
      AND A."ProjectNo" = :projectNo          -- nếu filter.projectNo != null
      AND A."ProvinceNo" = :provinceNo        -- nếu filter.provinceNo != null
      AND A."DistrictNo" = :districtNo        -- nếu filter.districtNo != null
      AND A."CommuneNo" = :communeNo          -- nếu filter.communeNo != null
      -- companyCondition: chọn đúng 1 nhánh theo kind (giống bản gốc)
      AND A."CompanyManagementLevel" >= :managementLevel
      -- (hoặc ID_IN / ID_EQ / NO_PREFIX)
)
SELECT "itemId","parentId","level","itemName","tabmisNo",
       i1, i2, i5, i7, i6, i8, i9n, i9c, i10n, i10c, i11, i12, i3, i4
FROM (

    -- ===== TH1: hợp đồng (key = ContractID) =====
    SELECT
        G.cate || '#ContractID' || TO_CHAR(G.contractId) AS "itemId",
        G.cate AS "parentId",
        2 AS "level",
        MIN(G.contractName) AS "itemName", MIN(G.tabmisNo) AS "tabmisNo",

        SUM(CASE WHEN SUBSTR(G.accountNo,1,3) IN ('091','092') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i1,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3) IN ('093','094') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i2,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i5,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=2
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i7,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i6,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=2
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i8,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i9n,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=2
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i9c,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i10n,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=2
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i10c,
        SUM(CASE WHEN G.postType=2 AND (
                (SUBSTR(G.accountNo,1,4)='2412' AND SUBSTR(G.coAccountNo,1,3) IN ('211','213'))
             OR (SUBSTR(G.accountNo,1,4)='2432' AND SUBSTR(G.coAccountNo,1,3) IN ('343','812','211','213')))
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i11,
        SUM(CASE WHEN G.postType=2 AND (
                (SUBSTR(G.accountNo,1,4)='2412' AND SUBSTR(G.coAccountNo,1,3) IN ('211','213'))
             OR (SUBSTR(G.accountNo,1,4)='2432' AND SUBSTR(G.coAccountNo,1,3) IN ('343','812','211','213')))
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i12,

        (SELECT COALESCE(SUM(CQ."Quantity"),0) FROM "contract_quantity" CQ WHERE CQ."ContractID" = G.contractId) AS i3,
        (SELECT COALESCE(SUM(CQ."LCAmount"),0) FROM "contract_quantity" CQ WHERE CQ."ContractID" = G.contractId) AS i4
    FROM gl_grp G
    WHERE G.cate IN ('A1','A2','A3','A4','A6','A7')
      AND ((G.contractId IS NOT NULL AND G.contractId > 0)
        OR (G.vendorId  IS NOT NULL AND G.vendorId  > 0)
        OR (G.partnerId IS NOT NULL AND G.partnerId > 0))
    GROUP BY G.cate, G.contractId

    UNION ALL

    -- ===== TH2: khoản-chi không hợp đồng/nghiệm thu (key = ExpenseID) =====
    SELECT
        CASE WHEN G.cate IN ('A5','A8') THEN G.cate
             ELSE G.cate || '#ExpenseID' || TO_CHAR(G.expenseId) END AS "itemId",
        CASE WHEN G.cate IN ('A5','A8') THEN CAST(NULL AS VARCHAR2(1))
             ELSE G.cate END AS "parentId",
        CASE WHEN G.cate IN ('A5','A8') THEN 1 ELSE 2 END AS "level",
        MIN(G.expenseName) AS "itemName", MIN(G.tabmisNo) AS "tabmisNo",

        SUM(CASE WHEN SUBSTR(G.accountNo,1,3) IN ('091','092') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i1,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3) IN ('093','094') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i2,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i5,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=2
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i7,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i6,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=2
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i8,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i9n,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=2
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i9c,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i10n,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=2
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i10c,
        SUM(CASE WHEN G.postType=2 AND (
                (SUBSTR(G.accountNo,1,4)='2412' AND SUBSTR(G.coAccountNo,1,3) IN ('211','213'))
             OR (SUBSTR(G.accountNo,1,4)='2432' AND SUBSTR(G.coAccountNo,1,3) IN ('343','812','211','213')))
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i11,
        SUM(CASE WHEN G.postType=2 AND (
                (SUBSTR(G.accountNo,1,4)='2412' AND SUBSTR(G.coAccountNo,1,3) IN ('211','213'))
             OR (SUBSTR(G.accountNo,1,4)='2432' AND SUBSTR(G.coAccountNo,1,3) IN ('343','812','211','213')))
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i12,

        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='094' AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i3,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='094' AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i4
    FROM gl_grp G
    WHERE (G.contractId IS NULL OR G.contractId = 0)
      AND (G.clearanceId IS NULL OR G.clearanceId = 0)
    GROUP BY G.cate, G.expenseId

    UNION ALL

    -- ===== TH3a: nghiệm thu — cột kế toán (key = A2#ClearanceID<v>), I3/I4=0 =====
    SELECT
        G.cate || '#ClearanceID' || TO_CHAR(G.clearanceId) AS "itemId",
        G.cate AS "parentId",
        2 AS "level",
        MIN(G.clearanceName) AS "itemName", MIN(G.tabmisNo) AS "tabmisNo",

        SUM(CASE WHEN SUBSTR(G.accountNo,1,3) IN ('091','092') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i1,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3) IN ('093','094') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i2,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i5,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=2
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i7,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i6,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,3)='073' AND G.postType=2
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i8,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=1
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i9n,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=2
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i9c,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=1
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN G.lcAmount ELSE 0 END) AS i10n,
        SUM(CASE WHEN SUBSTR(G.accountNo,1,4) IN ('2412','2431') AND G.postType=2
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (35,34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i10c,
        SUM(CASE WHEN G.postType=2 AND (
                (SUBSTR(G.accountNo,1,4)='2412' AND SUBSTR(G.coAccountNo,1,3) IN ('211','213'))
             OR (SUBSTR(G.accountNo,1,4)='2432' AND SUBSTR(G.coAccountNo,1,3) IN ('343','812','211','213')))
            AND G.postDate >= :fromDate AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i11,
        SUM(CASE WHEN G.postType=2 AND (
                (SUBSTR(G.accountNo,1,4)='2412' AND SUBSTR(G.coAccountNo,1,3) IN ('211','213'))
             OR (SUBSTR(G.accountNo,1,4)='2432' AND SUBSTR(G.coAccountNo,1,3) IN ('343','812','211','213')))
            AND G.postDate <= :toDate
            AND (G.inTransTypeId IS NULL OR G.inTransTypeId NOT IN (34,33,32))
            THEN -G.lcAmount ELSE 0 END) AS i12,

        0 AS i3, 0 AS i4
    FROM gl_grp G
    WHERE G.cate = 'A2'
      AND (G.contractId IS NULL OR G.contractId = 0)
      AND (G.clearanceId IS NOT NULL AND G.clearanceId > 0)
    GROUP BY G.cate, G.clearanceId

    UNION ALL

    -- ===== TH3b: nghiệm thu — I3/I4 từ clearance_quantity (key = A2#<v>, quirk giữ nguyên) =====
    SELECT
        G.cate || '#' || TO_CHAR(G.clearanceId) AS "itemId",
        G.cate AS "parentId",
        2 AS "level",
        (SELECT MIN(CQ."ClearanceName") FROM "clearance_quantity" CQ WHERE CQ."ClearanceID" = G.clearanceId) AS "itemName",
        MIN(G.tabmisNo) AS "tabmisNo",
        0 AS i1, 0 AS i2, 0 AS i5, 0 AS i7, 0 AS i6, 0 AS i8,
        0 AS i9n, 0 AS i9c, 0 AS i10n, 0 AS i10c, 0 AS i11, 0 AS i12,
        (SELECT COALESCE(SUM(CQ."LCAmount"),0) FROM "clearance_quantity" CQ WHERE CQ."ClearanceID" = G.clearanceId) AS i3,
        (SELECT COALESCE(SUM(CQ."LCAmount"),0) FROM "clearance_quantity" CQ WHERE CQ."ClearanceID" = G.clearanceId) AS i4
    FROM gl_grp G
    WHERE G.cate = 'A2'
      AND (G.contractId IS NULL OR G.contractId = 0)
      AND (G.clearanceId IS NOT NULL AND G.clearanceId > 0)
    GROUP BY G.cate, G.clearanceId

) t;
```

Với dữ liệu bảng glm_gl_books 1 triệu bản ghi cost giảm từ 109K -> xuống 27363



