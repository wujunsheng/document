# 报表
## 区间入库
```sql
DECLARE @minInboundCount INT;
DECLARE @timeStart DATE;
DECLARE @timeEnd DATE;
DECLARE @inventoryType INT;

SET @minInboundCount = 5;
SET @timeStart = '2021-01-11';
SET @timeEnd = GETDATE();
SET @inventoryType = 1;

/*
    1	Sale		销售库存
    2	Underwrite	作者包销库存
    4	Sample		样书库存
    8	Scrap		报废库存
    16	Incomplete	报残库存
 */

SELECT r.WarehouseBookGuid,
       r.BookEncode,
       r.书名 AS BookName,
       r.ScanISBN AS ISBN,
       r.入库数 AS InboundCount,
       r.入库日期 AS InboundDate,
       w.UnitPrice,            -- 定价,
       w.AuthorName,           -- 作者,
       wbi.AvailableInventory, -- 可销售库存,
       w.PublicationDate,      -- 出书时间,
       b.BookSeriesName,       -- 丛书名,
       b.Classify,             -- 分类,
       b.ContentSummary,       -- 简介,
       w.PackageUnitCount,     -- 包册数,
       w.DutyEditor,           -- 责编,
       b.Folio,                -- 开本,
       b.AuthorProfile,        -- 作者简介,
       bb.WordCount,           -- [字数(千字)],
       bb.PagesNumber          -- 码数
FROM
(
    SELECT r.WarehouseBookGuid,
           WarehouseBookId,
           r.BookEncode,
           --LEFT(r.BookEncode,1) 分类,
           r.书名,
           r.ScanISBN,
           SUM(r.入库数) 入库数,
           MAX(r.入库日期) 入库日期
    FROM
    ( --带印刷厂
        SELECT w.Guid AS WarehouseBookGuid,
               wib.Id,
               wib.WarehouseInboundId,
               w.BookPublishId,
               wib.WarehouseBookId,
               b.CustomerName 印刷厂,
               w.BookName 书名,
               w.BookEncode,
               w.ScanISBN,
               wib.Quantity 入库数,
               wib.PathZhCNName,
               wib.CreateDate 入库日期
        FROM dbo.WarehouseInboundBookSchedule wib
            LEFT JOIN WarehouseBook w
                ON w.Id = wib.WarehouseBookId
            LEFT JOIN dbo.BookPublish b
                ON b.Id = w.BookPublishId
        WHERE wib.WarehouseInboundId IN
              (
                  SELECT Id
                  FROM WarehouseInbound
                  WHERE InboundTypeDescription = '新书入库'
                        AND CreateDate > '2017-01-01'
              )
              AND wib.CreateDate
              BETWEEN @timeStart AND @timeEnd --检索起始日期，默认结束时间为当前时间
    ) r --入库图书明细
    GROUP BY r.BookEncode,
             r.书名,
             r.ScanISBN,
             r.WarehouseBookId,
             r.WarehouseBookGuid
) r
    LEFT JOIN dbo.WarehouseBookInventory wbi
        ON wbi.WarehouseBookId = r.WarehouseBookId
           AND wbi.InventoryTypeIdent = @inventoryType
    LEFT JOIN WarehouseBook w
        ON w.Id = r.WarehouseBookId
    LEFT JOIN dbo.Book b
        ON b.Id = w.BookId
    LEFT JOIN dbo.BookPublishBinding bb
        ON bb.BookPublishId = w.BookPublishId
           AND bb.States <> -1
WHERE wbi.AvailableInventory > 0
      AND r.入库数 > @minInboundCount --不完全排除法，去除掉小于5册的情况
ORDER BY r.入库日期;
```

## 确认三审三审检索
```sql
SELECT b.Id,
       b.BookName 书名,
       b.BookEncode 书目编码,
       b.DutyEditor 责编,
       b.AuthorName 作者,
       m.Name 审批人,
       r.Title 审批节点,
       r.CreateDate 审批操作时间,
       r.AuditStartDate 开始时间,
       r.AuditEndDate 结束时间,
       r.Content 审批内容
FROM XmuPublishManage.dbo.BookReviewAudit r --WHERE r.MemberId=11023
    LEFT JOIN XmuPublishManage.dbo.Book b
        ON b.Id = r.BookId
    LEFT JOIN XmuPublishManage.dbo.WorkFlowBookProcess w
        ON w.ProcessNodeDescription = r.Title
           AND w.BookId = r.BookId
    LEFT JOIN dbo.Member m
        ON m.Id = w.HandleMemberId
WHERE w.HandleMemberId IN ( 11029 )
      AND
      (
          r.Title LIKE '%复审意见%'
          OR r.Title LIKE '%终审意见%'
      )
      AND r.States <> -1
      AND b.Id IS NOT NULL
      AND w.States <> -1
      AND r.CreateDate
      BETWEEN '2020-01-01' AND '2020-12-31' --AND  b.EditorRoomName='理工编辑室' 
ORDER BY w.HandleMemberId,
         r.Title,
         r.CreateDate DESC;
```

## 处理退书翻倍
```sql
BEGIN TRAN;

SELECT *
FROM WarehouseReturned
WHERE Id = 41653;

UPDATE WarehouseReturned
SET TotalSortingQuantity = TotalSortingQuantity / 2,
    TotalQuantity = TotalQuantity / 2,
    TotalFixedPrice = TotalFixedPrice / 2,
    TotalDiscountPrice = TotalDiscountPrice / 2,
    TotalInvoiceAmount = TotalInvoiceAmount / 2,
    AuditAmount = AuditAmount / 2,
    TotalBackQuantity = TotalBackQuantity / 2,
    TotalScrapQuantity = TotalScrapQuantity / 2,
    TotalIncompleteQuantity = TotalIncompleteQuantity / 2
WHERE Id = 41653;

SELECT *
FROM dbo.WarehouseReturned
WHERE Id = 41653;

SELECT *
FROM dbo.WarehouseReturnedBook
WHERE WarehouseReturnedId = 41653;

UPDATE dbo.WarehouseReturnedBook
SET UnitPrice = UnitPrice / 2,
    SortingQuantity = SortingQuantity / 2,
    AvailableQuantity = AvailableQuantity / 2,
    Quantity = Quantity / 2,
    Discount = Discount / 2,
    FixedPrice = FixedPrice / 2,
    DiscountPrice = DiscountPrice / 2,
    BackQuantity = BackQuantity / 2,
    ScrapQuantity = ScrapQuantity / 2,
    IncompleteQuantity = IncompleteQuantity / 2,
    InvoiceAmount = InvoiceAmount / 2
WHERE WarehouseReturnedId = 41653;

SELECT *
FROM dbo.WarehouseReturnedBook
WHERE WarehouseReturnedId = 41653;

ROLLBACK;
```