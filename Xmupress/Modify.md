## 审批节点调整
```sql
update WorkFlowBookProcess set HandleMemberId=11063
where ModuleName='Review' and BookId=63589 and States<>-1
and ProcessNodeName='SocietyReviewSecond'
```

## CIP写入
```sql
INSERT into ChiefEditorCIP ([Guid], [MemberId], [ChiefEditorId], [BookId], [Content], [ExportDate], [AllocationDate], [IpAddress], [CreateDate], [LastUpdatedDate], [States],
CIPSerialNumber,CIPClassifyNumber,OldFileName)
select REPLACE(lower(newid()),'-',''),MemberId,(select max(Id) from ChiefEditor where ReportTypeName='CIP'),bc.BookId,
null,null,null,null,CreateDate,null,5
,CIPSerialNumber,CIPClassifyNumber,''  
from BookCIP bc  where BookId=59841 and States<>-1
```