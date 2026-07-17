
# 1. Case `SUBSTR(AccountNo,1,3)/,1,4)/,1,5)`

Phương án sử dụng `Virtual Generated Colum` của Oracle

```sql
ALTER TABLE "glm_gl_books" ADD (
    "AccountNo3" GENERATED ALWAYS AS (SUBSTR("AccountNo",1,3)) VIRTUAL,
    "AccountNo4" GENERATED ALWAYS AS (SUBSTR("AccountNo",1,4)) VIRTUAL,
    "AccountNo5" GENERATED ALWAYS AS (SUBSTR("AccountNo",1,5)) VIRTUAL
);

CREATE INDEX "IX_GLM_GL_BOOKS_ACCTNO3" ON "glm_gl_books" ("AccountNo3");
CREATE INDEX "IX_GLM_GL_BOOKS_ACCTNO4" ON "glm_gl_books" ("AccountNo4");
CREATE INDEX "IX_GLM_GL_BOOKS_ACCTNO5" ON "glm_gl_books" ("AccountNo5");
```


> [!NOTE]
> - không cần sửa lại các file SQL đang có -> oracle tự route sang cột này và index mới
> - vẫn giải quyết  được vđề index thay vì `function-based index`
> - SQL mới có thể dùng cột mới hoặc giữ nguyên các việc cũ `SUBSTR`
> - Đảm bảo dynamic được giá trị thay đổi trong khoảng của `SUBSTR` khi AccountNo thay đổi

