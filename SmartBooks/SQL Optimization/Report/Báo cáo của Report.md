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