# DAX-Query-View

/* ####################################################################################################
/*
​1. Measures
​2. Calculated Columns
​3. Calculated Tables
​4. Parameters
​5. Tables
​6. Partitions
​7. Columns
​8. Relationships
​9. Partitions - with Row count
​10. Relationships - from and to column datatypes
*/
// 1. Measures:
EVALUATE
SELECTCOLUMNS (
INFO.VIEW.MEASURES (),
"Table", [Table],
"Measure Name", [Name],
"DAX Expression", [Expression], // [Description]
"Display Folder", [DisplayFolder]
// [State]
)
// 2. Calculated Columns:
EVALUATE
SELECTCOLUMNS (
FILTER (
INFO.VIEW.COLUMNS (),
//      [DataCategory] <> "RowNumber" && [Table] <> "xTables" && NOT ISBLANK ( [EXPRESSION] )
[DataCategory] <> "RowNumber"
&& [Table] <> "xTables"
&& [Type] = "Calculated" // Type has values Calculated  and Data
),
[Table],
"Calculated Column Name", [Name],
// [Description],
"DAX Expression", [Expression] // [DataCategory],
// [DataType],
// [IsHidden]
)
ORDER BY
[Table],
[Calculated Column Name]
// 3. Calculated Tables:
EVALUATE
SELECTCOLUMNS (
FILTER (
INFO.VIEW.TABLES (),
ISBLANK ( [Expression] )
= FALSE () // Keep only calculated tables; TRUE() will show all regular tables
),
"Calculated Table", [Name],
"Storage Mode", [StorageMode],
"Expression", [Expression]
)
ORDER BY [Calculated Table] ASC
// 4. Parameters:
EVALUATE
SELECTCOLUMNS ( INFO.EXPRESSIONS (), "Parameter Name", [Name], [Expression] )
// 5. Tables:
EVALUATE
SELECTCOLUMNS (
INFO.VIEW.TABLES (),
"Table Name", [Name],
[StorageMode],
[Expression]
)
ORDER BY
[StorageMode],
[Table Name]
// 6. Partitions:
EVALUATE
SELECTCOLUMNS (
FILTER (
INFO.PARTITIONS (),
[Type] <> 2 //Calculated Table Type = 2, Regular Table Type = 4
),
"Table or Partition Name", [Name],
"Query", [QueryDefinition]
)
// 7. Columns:
EVALUATE
VAR ColumnInfo =
FILTER ( INFO.VIEW.COLUMNS (), [DataCategory] <> "RowNumber" )
VAR TableCounts =
GROUPBY ( ColumnInfo, [Table], "# columns", COUNTX ( CURRENTGROUP (), 1 ) )
RETURN
SELECTCOLUMNS (
NATURALLEFTOUTERJOIN ( ColumnInfo, TableCounts ),
"Table", [Table],
"Column Name", [Name],
"DataType", [DataType],
"Column Count", [# columns]
)
ORDER BY
[Table],
[DataType],
[Column Name]
// 8. Relationships:
EVALUATE
SELECTCOLUMNS (
INFO.VIEW.RELATIONSHIPS (),
[Name],
[Relationship],
[FromTable],
[FromColumn],
[FromCardinality],
[ToTable],
[ToColumn],
[ToCardinality],
[IsActive],
[CrossFilteringBehavior],
[RelyOnReferentialIntegrity],
[SecurityFilteringBehavior]
)
// 9. Partitions - with Row count
EVALUATE
VAR _partition = SELECTCOLUMNS(
INFO.PARTITIONS(),
"TableID", [TableID],
"PartitionStorageID", [PartitionStorageID],
"Name", [Name],
"Query", [QueryDefinition]
)
VAR _table = SELECTCOLUMNS(
INFO.TABLES(),
"TableID", [ID],
"TableName", [Name]
)
RETURN
SELECTCOLUMNS(
NATURALINNERJOIN(
NATURALINNERJOIN(
_partition,
INFO.SEGMENTMAPSTORAGES()
),
_table
),
"TableName", [TableName],
"PartitionName", [Name],
"Query", [Query],
"Rows", [RecordCount]
)
// 10. Relationships - from and to column datatypes
EVALUATE
VAR Cols =
SELECTCOLUMNS (
INFO.VIEW.COLUMNS(),
"ColumnID", [ID],
"TableName", [Table],
"ColumnName", [Name],
"DataType", [DataType]
)
VAR FromCols =
SELECTCOLUMNS (
Cols,
"FromColumnID", [ColumnID],
"FromTable", [TableName],
"FromColumn", [ColumnName],
"FromDataType", [DataType]
)
VAR ToCols =
SELECTCOLUMNS (
Cols,
"ToColumnID", [ColumnID],
"ToTable", [TableName],
"ToColumn", [ColumnName],
"ToDataType", [DataType]
)
RETURN
SELECTCOLUMNS (
NATURALLEFTOUTERJOIN (
NATURALLEFTOUTERJOIN (
INFO.RELATIONSHIPS(),
FromCols
),
ToCols
),
"Relationship", [Name],
"FromTable", [FromTable],
"FromColumn", [FromColumn],
"FromDataType", [FromDataType],
"ToTable", [ToTable],
"ToColumn", [ToColumn],
"ToDataType", [ToDataType],
"DataTypeMatch", [FromDataType] = [ToDataType]
)
​/* ####################################################################################################
/*
11.Roles
12.Table Permissions
13.Column Permissions
14.Dependencies
15.Dependencies
*/
​// 11.Roles
EVALUATE
VAR _Roles =
INFO.ROLES()
VAR ModelPermission =
DATATABLE (
"ModelPermission", INTEGER,
"PermissionDescription", STRING,
{
{ 1, "None: The role has no access to the model." },
{ 2, "Read: The role can read metadata and data of the model." },
{ 3, "ReadRefresh: The role has read and refresh permission." },
{ 4, "Refresh: The role can refresh the data and calculations in the model." },
{ 5, "Administrator: The role can administer the model." }
}
)
VAR _CombinedTable =
NATURALLEFTOUTERJOIN (
_Roles,
ModelPermission
)
RETURN
SELECTCOLUMNS (
_CombinedTable,
"Role Name", [Name],
"Model Permission",[ModelPermission],
"Model Permission Desc", [PermissionDescription],
"ID", [ID],
"Model ID", [ModelID],
"Modified Time", [ModifiedTime]
)
ORDER BY [Role Name]
​// 12. Table Permissions
EVALUATE
VAR _INFO = INFO.TABLEPERMISSIONS()
VAR MetadataPermissionEnum =
DATATABLE(
"MetadataPermission",INTEGER,
"MetadataDescription",STRING,
{
{0,"Default behavior - no access control is enforced"},
{1,"Access to the data is restricted"},
{2,"Access to the data is unrestricted"}
}
)
VAR StateEnum =
DATATABLE(
"State",INTEGER,
"StateName",STRING,
{
{1,"Ready"},
{3,"NoData"},
{4,"CalculationNeeded"},
{5,"SemanticError"},
{6,"EvaluationError"},
{7,"DependencyError"},
{8,"Incomplete"},
{10,"ForceCalculationNeeded"}
}
)
VAR _CombinedTable =
NATURALLEFTOUTERJOIN(
_INFO,
MetadataPermissionEnum
)
VAR _CombinedTable2 =
NATURALLEFTOUTERJOIN(
_CombinedTable,
StateEnum
)
VAR _ModelRoles =
SELECTCOLUMNS(
INFO.ROLES(),
"RoleID", [ID],
"Role Name", [Name]
)
VAR _ModelTables =
SELECTCOLUMNS(
INFO.TABLES(),
"TableID", [ID],
"Table Name", [Name]
)
VAR _CombinedTable3 =
NATURALLEFTOUTERJOIN(
_CombinedTable2,
_ModelRoles
)
VAR _CombinedTable4 =
NATURALLEFTOUTERJOIN(
_CombinedTable3,
_ModelTables
)
​RETURN
SELECTCOLUMNS(
_CombinedTable4,
"Role Name", [Role Name],
"Table Name", [Table Name],
"Table Permission ID", [ID],
"Filter Expression", [FilterExpression],
"Metadata permission description", [MetadataDescription],
"State Name", [StateName],
"Modified Time", [ModifiedTime],
"Error Message", [ErrorMessage]
)
​ORDER BY [Table Permission ID]
​// 13. Column Permissions
EVALUATE
VAR _ColumnPermissions =
INFO.COLUMNPERMISSIONS()
VAR MetadataPermissionEnum =
DATATABLE(
"MetadataPermission", INTEGER,
"MetadataDescription", STRING,
{
{0, "Default behavior - no access control is enforced"},
{1, "Access to the data is restricted"},
{2, "Access to the data is unrestricted"}
}
)
VAR _TablePermissions =
SELECTCOLUMNS(
INFO.TABLEPERMISSIONS(),
"TablePermissionID", [ID],
"RoleID", [RoleID],
"TableID", [TableID]
)
VAR _Roles =
SELECTCOLUMNS(
INFO.ROLES(),
"RoleID", [ID],
"Role Name", [Name]
)
VAR _Tables =
SELECTCOLUMNS(
INFO.TABLES(),
"TableID", [ID],
"Table Name", [Name]
)
VAR _Columns =
SELECTCOLUMNS(
INFO.COLUMNS(),
"ColumnID", [ID],
"Column Name", [ExplicitName]
)
VAR _CombinedWithPermissionDesc =
NATURALLEFTOUTERJOIN(
_ColumnPermissions,
MetadataPermissionEnum
)
VAR _CombinedWithTablePermissions =
NATURALLEFTOUTERJOIN(
_CombinedWithPermissionDesc,
_TablePermissions
)
VAR _CombinedWithRoles =
NATURALLEFTOUTERJOIN(
_CombinedWithTablePermissions,
_Roles
)
VAR _CombinedWithTables =
NATURALLEFTOUTERJOIN(
_CombinedWithRoles,
_Tables
)
VAR _CombinedWithColumns =
NATURALLEFTOUTERJOIN(
_CombinedWithTables,
_Columns
)
RETURN
SELECTCOLUMNS(
_CombinedWithColumns,
"Role Name", [Role Name],
"Table Name", [Table Name],
"Column Name", [Column Name],
"Metadata Permission", [MetadataPermission],
"Permission Definition", [MetadataDescription],
"Modified Time", [ModifiedTime]
)
ORDER BY
[Role Name],
[Table Name],
[Column Name]
​// 14. Dependencies
EVALUATE
VAR _Dependencies =
INFO.DEPENDENCIES()
VAR _Tables =
SELECTCOLUMNS(
INFO.TABLES(),
"TABLE", [Name],
"Table ID", [ID]
)
VAR _ReferencedTables =
SELECTCOLUMNS(
INFO.TABLES(),
"REFERENCED_TABLE", [Name],
"Referenced Table ID", [ID]
)
VAR _CombinedWithTables =
NATURALLEFTOUTERJOIN(
_Dependencies,
_Tables
)
VAR _CombinedWithReferencedTables =
NATURALLEFTOUTERJOIN(
_CombinedWithTables,
_ReferencedTables
)
RETURN
SELECTCOLUMNS(
_CombinedWithReferencedTables,
"Object Type", [OBJECT_TYPE],
"Table", [TABLE],
"Object", [OBJECT],
"Referenced Object Type", [REFERENCED_OBJECT_TYPE],
"Referenced Table", [REFERENCED_TABLE],
"Referenced Object", [REFERENCED_OBJECT]
)
ORDER BY [TABLE], [OBJECT]
​// 15. Dependencies
EVALUATE
INFO.CALCDEPENDENCY()
​/* #################################################################################################### */
​/*
Prompt after attaching Partitions and Parameters output to LLM:
Extract Source Schema and Table name from this Power Query M expressions into a table form: Partition Name, Schema Name, Table Name. Add serial number to each row of table as first column
Replace SecuritySchemaName as OPBISDMPQA
*/
​// Tables based on Storage mode:
EVALUATE
SELECTCOLUMNS(
FILTER(
INFO.VIEW.TABLES(),
[StorageMode] = "DirectQuery" //Storage mode = DirectQuery or Import
),
"Table", [Name],
"Storage Mode", [StorageMode],
"Expression", [Expression]
)
ORDER BY [Storage Mode] ASC
