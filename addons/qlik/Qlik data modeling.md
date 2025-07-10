# Complete Qlik Data Modeling Compact Guide

## 3-Stage Architecture Overview
**STG** (Staging) → **TRF** (Transform) → **MDL** (Model)
- **Separation of Concerns**: Each stage has distinct purpose & optimization
- **Data Lineage**: Full traceability from source to model
- **Incremental Processing**: Each stage supports delta loads
- **Error Isolation**: Issues contained within specific stage

## Stage 1: Staging Layer (STG_*)
**Purpose**: Raw data preservation, extraction efficiency
**Naming**: `STG_TableName.qvd`
**Location**: `lib://Staging/`

### Staging Patterns
```sql
// Basic staging pattern
STG_Orders:
LOAD *, 
    Now() as StagingLoadTime,
    '$(vSourceFile)' as SourceFile
FROM [$(vSourcePath)Orders.csv] (txt, codepage is 1252, embedded labels, delimiter is ',');
STORE STG_Orders INTO [lib://Staging/STG_Orders.qvd] (qvd);
DROP TABLE STG_Orders;

// Incremental staging
STG_Orders_Incremental:
LOAD * FROM [lib://Staging/STG_Orders.qvd] (qvd)
WHERE NOT EXISTS(NewOrderID);

CONCATENATE (STG_Orders_Incremental)
LOAD *, OrderID as NewOrderID
FROM [$(vSourcePath)Orders_Delta.csv]
WHERE ModifiedDate > '$(vLastLoadDate)';

STORE STG_Orders_Incremental INTO [lib://Staging/STG_Orders.qvd] (qvd);
```

### Staging Best Practices
- **No Transformations**: Store data exactly as received
- **Metadata Addition**: LoadTime, SourceFile, BatchID
- **Error Handling**: `SET ErrorMode = 0;` continue on errors
- **Compression**: QVD format for optimal storage
- **Parallel Processing**: Multiple sources simultaneously

## Stage 2: Transform Layer (TRF_*)
**Purpose**: Business logic, data quality, enrichment
**Naming**: `TRF_TableName.qvd`
**Location**: `lib://Transform/`

### Data Quality Patterns
```sql
// Comprehensive transformation
TRF_Customers:
LOAD 
    // Key fields
    CustomerID,
    
    // Cleansed text fields
    Proper(Trim(CustomerName)) as CustomerName,
    Upper(Trim(ContactName)) as ContactName,
    
    // Standardized codes
    Upper(CountryCode) as CountryCode,
    ApplyMap('CountryMapping', CountryCode, 'Unknown') as CountryName,
    ApplyMap('RegionMapping', CountryCode, 'Unknown') as Region,
    
    // Business rules
    If(CreditLimit <= 0, 'No Credit',
       If(CreditLimit <= 10000, 'Low Credit',
          If(CreditLimit <= 50000, 'Medium Credit', 'High Credit'))) as CreditCategory,
    
    // Data type conversions
    Num(CreditLimit, '#,##0.00') as CreditLimit,
    Date(Floor(CreatedDate)) as CreatedDate,
    
    // Derived fields
    If(IsNull(Phone) OR Len(Trim(Phone)) = 0, 1, 0) as DQ_MissingPhone,
    If(CountryCode = 'Unknown', 1, 0) as DQ_UnknownCountry,
    
    // Audit fields
    Now() as TransformLoadTime
FROM [lib://Staging/STG_Customers.qvd] (qvd);

STORE TRF_Customers INTO [lib://Transform/TRF_Customers.qvd] (qvd);
```

### Reference Data & Mappings
```sql
// Country mapping table
CountryMapping:
MAPPING LOAD CountryCode, CountryName 
FROM [lib://Reference/Countries.qvd] (qvd);

RegionMapping:
MAPPING LOAD CountryCode, Region 
FROM [lib://Reference/Countries.qvd] (qvd);

// Currency conversion
CurrencyRates:
LOAD CurrencyCode, ExchangeRate, EffectiveDate
FROM [lib://Reference/CurrencyRates.qvd] (qvd);
```

### Business Logic Implementation
```sql
// Complex business rules
TRF_Orders:
LOAD 
    OrderID,
    CustomerID,
    EmployeeID,
    OrderDate,
    RequiredDate,
    ShippedDate,
    
    // Business classifications
    If(RequiredDate - OrderDate <= 7, 'Rush',
       If(RequiredDate - OrderDate <= 14, 'Standard', 'Extended')) as UrgencyLevel,
    
    If(ShippedDate <= RequiredDate, 'On Time',
       If(ShippedDate <= RequiredDate + 3, 'Late',
          'Very Late')) as DeliveryPerformance,
    
    // Calculated fields
    RequiredDate - OrderDate as LeadTimeDays,
    If(IsNull(ShippedDate), Null(), ShippedDate - OrderDate) as ActualLeadTime,
    
    // Freight categories
    If(Freight <= 10, 'Low',
       If(Freight <= 50, 'Medium',
          If(Freight <= 100, 'High', 'Premium'))) as FreightCategory
FROM [lib://Staging/STG_Orders.qvd] (qvd);
```

## Stage 3: Data Model Layer
**Purpose**: Dimensional model, associations, performance optimization
**Naming**: `FactTableName`, `DimTableName`, `LinkTableName`
**Location**: `lib://Model/`

### Fact Table Pattern
```sql
// Comprehensive fact table
FactSales:
LOAD 
    // Surrogate key
    AutoNumber(OrderID & '|' & ProductID, 'SalesKey') as SalesKey,
    
    // Foreign keys (for associations)
    OrderID,
    ProductID,
    CustomerID,
    EmployeeID,
    
    // Date keys
    OrderDate,
    RequiredDate,
    ShippedDate,
    
    // Measures
    Quantity,
    UnitPrice,
    Discount,
    
    // Calculated measures
    Quantity * UnitPrice as GrossAmount,
    Quantity * UnitPrice * (1 - Discount) as NetAmount,
    Quantity * UnitPrice * Discount as DiscountAmount,
    
    // Derived attributes from transformation
    UrgencyLevel,
    DeliveryPerformance,
    FreightCategory,
    
    // Audit fields
    Now() as ModelLoadTime
FROM [lib://Transform/TRF_OrderDetails.qvd] (qvd) o
LEFT JOIN [lib://Transform/TRF_Orders.qvd] (qvd) ord ON o.OrderID = ord.OrderID;

STORE FactSales INTO [lib://Model/FactSales.qvd] (qvd);
```

### Dimension Table Pattern
```sql
// Optimized dimension table
DimCustomer:
LOAD 
    // Natural key
    CustomerID,
    
    // Core attributes
    CustomerName,
    ContactName,
    
    // Geographic hierarchy
    CountryCode,
    CountryName,
    Region,
    City,
    PostalCode,
    
    // Business attributes
    CreditLimit,
    CreditCategory,
    
    // Data quality flags
    DQ_MissingPhone,
    DQ_UnknownCountry,
    
    // Temporal attributes
    CreatedDate,
    ModifiedDate,
    
    // Model metadata
    Now() as ModelLoadTime
FROM [lib://Transform/TRF_Customers.qvd] (qvd);

STORE DimCustomer INTO [lib://Model/DimCustomer.qvd] (qvd);
```

### Calendar Dimension
```sql
// Master calendar with business logic
LET vMinDate = YearStart(Date('2020-01-01'));
LET vMaxDate = YearEnd(AddYears(Today(), 2));

DimCalendar:
LOAD 
    // Primary date
    Date($(vMinDate) + IterNo() - 1) as Date,
    
    // Year attributes
    Year($(vMinDate) + IterNo() - 1) as Year,
    'CY ' & Year($(vMinDate) + IterNo() - 1) as YearLabel,
    
    // Quarter attributes
    Quarter($(vMinDate) + IterNo() - 1) as Quarter,
    'Q' & Quarter($(vMinDate) + IterNo() - 1) & ' ' & Year($(vMinDate) + IterNo() - 1) as QuarterLabel,
    
    // Month attributes
    Month($(vMinDate) + IterNo() - 1) as Month,
    MonthName($(vMinDate) + IterNo() - 1) as MonthName,
    Date(MonthStart($(vMinDate) + IterNo() - 1), 'MMM YYYY') as MonthYear,
    
    // Week attributes
    Week($(vMinDate) + IterNo() - 1) as Week,
    WeekYear($(vMinDate) + IterNo() - 1) as WeekYear,
    
    // Day attributes
    Day($(vMinDate) + IterNo() - 1) as Day,
    WeekDay($(vMinDate) + IterNo() - 1) as WeekDay,
    
    // Business day logic
    If(WeekDay($(vMinDate) + IterNo() - 1) >= 1 AND WeekDay($(vMinDate) + IterNo() - 1) <= 5, 1, 0) as IsBusinessDay,
    If(WeekDay($(vMinDate) + IterNo() - 1) >= 1 AND WeekDay($(vMinDate) + IterNo() - 1) <= 5, 'Weekday', 'Weekend') as DayType,
    
    // Relative dates
    If($(vMinDate) + IterNo() - 1 = Today(), 'Today',
       If($(vMinDate) + IterNo() - 1 = Today() - 1, 'Yesterday',
          If($(vMinDate) + IterNo() - 1 < Today(), 'Past', 'Future'))) as RelativeDate,
    
    // Fiscal year (April start)
    If(Month($(vMinDate) + IterNo() - 1) >= 4, 
       Year($(vMinDate) + IterNo() - 1), 
       Year($(vMinDate) + IterNo() - 1) - 1) as FiscalYear

AUTOGENERATE 1
WHILE $(vMinDate) + IterNo() - 1 <= $(vMaxDate);

STORE DimCalendar INTO [lib://Model/DimCalendar.qvd] (qvd);
```

## SCD2 Implementation (LOAD WHILE Pattern)
**Purpose**: Track historical changes without IntervalMatch
**Performance**: Superior to IntervalMatch for large datasets

### Complete SCD2 Implementation
```sql
// SCD2 Customer dimension with LOAD WHILE
SET vCurrentDate = Date(Today(), 'YYYY-MM-DD');
SET vMaxDate = Date(MakeDate(9999,12,31), 'YYYY-MM-DD');

// Load existing SCD2 data
IF NOT IsNull(FileSize('lib://SCD2/DimCustomer_SCD2.qvd')) THEN
    ExistingCustomerSCD2:
    LOAD * FROM [lib://SCD2/DimCustomer_SCD2.qvd] (qvd);
ELSE
    ExistingCustomerSCD2:
    LOAD * WHERE 1=0;
    LOAD 
        '' as CustomerSurrogateKey,
        '' as CustomerID,
        '' as CustomerName,
        '' as CountryCode,
        0 as CreditLimit,
        Date(1) as EffectiveStartDate,
        Date(1) as EffectiveEndDate,
        0 as IsCurrentRecord,
        0 as VersionNumber
    AUTOGENERATE 0;
END IF

// Load new customer data
NewCustomerData:
LOAD CustomerID, CustomerName, CountryCode, CreditLimit
FROM [lib://Transform/TRF_Customers.qvd] (qvd);

// Get customers to process
CustomersToProcess:
LOAD DISTINCT CustomerID RESIDENT NewCustomerData;

// Initialize result table
CustomerSCD2_Result:
LOAD * WHERE 1=0;
LOAD 
    '' as CustomerSurrogateKey,
    '' as CustomerID,
    '' as CustomerName,
    '' as CountryCode,
    0 as CreditLimit,
    Date(1) as EffectiveStartDate,
    Date(1) as EffectiveEndDate,
    0 as IsCurrentRecord,
    0 as VersionNumber
AUTOGENERATE 0;

// Process each customer with LOAD WHILE
LET vCustomerCount = NoOfRows('CustomersToProcess');
LET vProcessedCount = 0;

WHILE vProcessedCount < vCustomerCount;
    LET vCurrentCustomerID = Peek('CustomerID', $(vProcessedCount), 'CustomersToProcess');
    
    // Get new customer data
    CurrentCustomerNew:
    LOAD * RESIDENT NewCustomerData WHERE CustomerID = '$(vCurrentCustomerID)';
    
    // Get existing SCD2 records
    CurrentCustomerExisting:
    LOAD * RESIDENT ExistingCustomerSCD2 WHERE CustomerID = '$(vCurrentCustomerID)';
    
    LET vExistingRecords = NoOfRows('CurrentCustomerExisting');
    
    IF vExistingRecords = 0 THEN
        // New customer - create initial record
        CONCATENATE (CustomerSCD2_Result)
        LOAD 
            CustomerID,
            CustomerName,
            CountryCode,
            CreditLimit,
            Date('$(vCurrentDate)') as EffectiveStartDate,
            Date('$(vMaxDate)') as EffectiveEndDate,
            1 as IsCurrentRecord,
            1 as VersionNumber
        RESIDENT CurrentCustomerNew;
    ELSE
        // Existing customer - check for changes
        CurrentCustomerCurrent:
        LOAD * RESIDENT CurrentCustomerExisting WHERE IsCurrentRecord = 1;
        
        // Compare attributes for changes
        LET vCurrentName = Peek('CustomerName', 0, 'CurrentCustomerCurrent');
        LET vNewName = Peek('CustomerName', 0, 'CurrentCustomerNew');
        LET vCurrentCountry = Peek('CountryCode', 0, 'CurrentCustomerCurrent');
        LET vNewCountry = Peek('CountryCode', 0, 'CurrentCustomerNew');
        LET vCurrentCredit = Peek('CreditLimit', 0, 'CurrentCustomerCurrent');
        LET vNewCredit = Peek('CreditLimit', 0, 'CurrentCustomerNew');
        
        LET vHasChanges = 0;
        IF '$(vCurrentName)' <> '$(vNewName)' OR 
           '$(vCurrentCountry)' <> '$(vNewCountry)' OR 
           $(vCurrentCredit) <> $(vNewCredit) THEN
            LET vHasChanges = 1;
        END IF
        
        IF vHasChanges = 1 THEN
            // Changes detected - create new version
            LET vMaxVersion = Peek('VersionNumber', 0, 'CurrentCustomerCurrent');
            
            // Add historical records (unchanged)
            CONCATENATE (CustomerSCD2_Result)
            LOAD * RESIDENT CurrentCustomerExisting WHERE IsCurrentRecord = 0;
            
            // Close current record
            CONCATENATE (CustomerSCD2_Result)
            LOAD 
                CustomerID,
                CustomerName,
                CountryCode,
                CreditLimit,
                EffectiveStartDate,
                Date('$(vCurrentDate)') as EffectiveEndDate,
                0 as IsCurrentRecord,
                VersionNumber
            RESIDENT CurrentCustomerCurrent;
            
            // Add new current record
            CONCATENATE (CustomerSCD2_Result)
            LOAD 
                CustomerID,
                CustomerName,
                CountryCode,
                CreditLimit,
                Date('$(vCurrentDate)') as EffectiveStartDate,
                Date('$(vMaxDate)') as EffectiveEndDate,
                1 as IsCurrentRecord,
                $(vMaxVersion) + 1 as VersionNumber
            RESIDENT CurrentCustomerNew;
        ELSE
            // No changes - keep existing records
            CONCATENATE (CustomerSCD2_Result)
            LOAD * RESIDENT CurrentCustomerExisting;
        END IF
        
        DROP TABLE CurrentCustomerCurrent;
    END IF
    
    DROP TABLES CurrentCustomerNew, CurrentCustomerExisting;
    LET vProcessedCount = vProcessedCount + 1;
LOOP

// Add surrogate keys
DimCustomer_SCD2_Final:
LOAD 
    AutoNumber(CustomerID & '|' & VersionNumber, 'CustomerSurrogateKey') as CustomerSurrogateKey,
    *
RESIDENT CustomerSCD2_Result;

STORE DimCustomer_SCD2_Final INTO [lib://SCD2/DimCustomer_SCD2.qvd] (qvd);
```

### SCD2 Fact Integration
```sql
// Point-in-time lookup for fact tables
FactSales_WithSCD2:
LOAD 
    SalesKey,
    OrderID,
    ProductID,
    CustomerID,
    OrderDate,
    Quantity,
    Amount
FROM [lib://Model/FactSales.qvd] (qvd);

// Add SCD2 customer keys
FactSales_SCD2_Enhanced:
LOAD 
    SalesKey,
    OrderID,
    ProductID,
    CustomerID,
    OrderDate,
    Quantity,
    Amount,
    // Point-in-time lookup
    If(NoOfRows('CustomerSCD2_Lookup') > 0, 
       Peek('CustomerSurrogateKey', 0, 'CustomerSCD2_Lookup'), 
       'UNKNOWN') as CustomerSurrogateKey
FROM FactSales_WithSCD2 f;

CustomerSCD2_Lookup:
LOAD CustomerSurrogateKey
FROM [lib://SCD2/DimCustomer_SCD2.qvd] (qvd)
WHERE CustomerID = f.CustomerID
  AND f.OrderDate >= EffectiveStartDate
  AND f.OrderDate <= EffectiveEndDate;
```

## Preventing Circular References
**Problem**: Multiple association paths between tables
**Impact**: Incorrect selection propagation, performance issues

### Detection & Resolution
```sql
// Problem example - creates circular reference
Orders: LOAD OrderID, CustomerID, EmployeeID FROM [orders.csv];
Customers: LOAD CustomerID, CustomerName, RegionID FROM [customers.csv];
Employees: LOAD EmployeeID, EmployeeName, RegionID FROM [employees.csv];
Regions: LOAD RegionID, RegionName FROM [regions.csv];
// Circular: Orders->Customers->Regions<-Employees<-Orders

// Solution 1: Rename fields to break unwanted associations
Orders: LOAD OrderID, CustomerID, EmployeeID FROM [orders.csv];
Customers: LOAD CustomerID, CustomerName, RegionID FROM [customers.csv];
Employees: LOAD EmployeeID, EmployeeName, RegionID as EmpRegionID FROM [employees.csv];
Regions: LOAD RegionID, RegionName FROM [regions.csv];
EmpRegions: LOAD RegionID as EmpRegionID, RegionName as EmpRegionName FROM [regions.csv];

// Solution 2: Link table approach
LinkEmployeeRegion:
LOAD 
    EmployeeID & '|' & RegionID as EmpRegionKey,
    EmployeeID,
    RegionID as EmpRegionID
FROM [employee_regions.csv];

// Solution 3: Composite key strategy
Orders_Enhanced:
LOAD 
    OrderID,
    CustomerID,
    EmployeeID,
    CustomerID & '|' & EmployeeID as CustomerEmployeeKey
FROM [orders.csv];
```

## Preventing Synthetic Keys
**Problem**: Multiple shared fields create automatic synthetic keys
**Impact**: Performance degradation, model complexity

### Prevention Strategies
```sql
// Problem: Multiple shared fields
Orders: LOAD OrderID, CustomerID, EmployeeID, OrderDate FROM [orders.csv];
Shipments: LOAD ShipmentID, OrderID, CustomerID, EmployeeID, ShipDate FROM [shipments.csv];
// Creates synthetic key on CustomerID + EmployeeID

// Solution 1: QUALIFY statement
QUALIFY *;
UNQUALIFY OrderID, CustomerID, EmployeeID; // Keep association fields

Orders: LOAD OrderID, CustomerID, EmployeeID, OrderDate FROM [orders.csv];
Shipments: LOAD ShipmentID, OrderID, CustomerID, EmployeeID, ShipDate FROM [shipments.csv];
// Result: Orders.OrderDate, Shipments.ShipDate (qualified)
//         OrderID, CustomerID, EmployeeID (unqualified for association)

UNQUALIFY *; // Reset

// Solution 2: Strategic field renaming
Orders: LOAD OrderID, CustomerID, EmployeeID, OrderDate FROM [orders.csv];
Shipments: 
LOAD 
    ShipmentID,
    OrderID, // Keep for association
    CustomerID as ShipCustomerID, // Rename to prevent synthetic key
    EmployeeID as ShipEmployeeID, // Rename to prevent synthetic key
    ShipDate
FROM [shipments.csv];

// Solution 3: Composite key approach
Orders: LOAD OrderID, CustomerID, EmployeeID, OrderDate FROM [orders.csv];
Shipments:
LOAD 
    ShipmentID,
    OrderID & '|' & CustomerID & '|' & EmployeeID as OrderKey, // Single association
    ShipDate
FROM [shipments.csv];

Orders_Enhanced:
LOAD 
    OrderID & '|' & CustomerID & '|' & EmployeeID as OrderKey, // Matching key
    OrderID,
    CustomerID,
    EmployeeID,
    OrderDate
FROM [orders.csv];
```

## Link Tables for Many-to-Many
**Purpose**: Handle complex relationships, prevent circular references

### Implementation Patterns
```sql
// Employee-Territory many-to-many relationship
LinkEmployeeTerritory:
LOAD 
    // Composite key
    EmployeeID & '|' & TerritoryID as EmpTerritoryKey,
    
    // Individual keys for associations
    EmployeeID,
    TerritoryID,
    
    // Relationship attributes
    AssignmentDate,
    AssignmentEndDate,
    IsPrimaryTerritory,
    ResponsibilityLevel,
    
    // Derived attributes
    If(IsNull(AssignmentEndDate) OR AssignmentEndDate >= Today(), 1, 0) as IsCurrentAssignment,
    If(IsPrimaryTerritory = 1, 'Primary', 'Secondary') as AssignmentType
FROM [employee_territories.csv];

// Clean dimension tables (no conflicting fields)
DimEmployee:
LOAD 
    EmployeeID,
    EmployeeName,
    Title,
    Department,
    HireDate,
    ManagerID
    // No territory fields to prevent circular reference
FROM [employees.csv];

DimTerritory:
LOAD 
    TerritoryID,
    TerritoryName,
    RegionID,
    RegionName
FROM [territories.csv];
```

## Performance Optimization

### QVD Optimization
```sql
// Optimized load (maintains QVD optimization)
OptimizedCustomers:
LOAD CustomerID, CustomerName, CountryCode
FROM [lib://Model/DimCustomer.qvd] (qvd)
WHERE EXISTS(ActiveCustomer, CustomerID);

// Non-optimized (breaks optimization)
// LOAD CustomerID, Upper(CustomerName) as CustomerName // Transformation breaks optimization

// Maintain optimization with WHERE EXISTS pattern
ActiveCustomers:
LOAD DISTINCT CustomerID as ActiveCustomer
FROM [lib://Model/FactSales.qvd] (qvd);
```

### Memory Management
```sql
// Efficient memory pattern
LET vMemoryStart = DocumentMemoryUsage();

Table1: LOAD * FROM [source1.csv];
Table2: LOAD * FROM [source2.csv];

// Process and immediately drop
Result: 
LOAD * RESIDENT Table1
LEFT JOIN Table2;

DROP TABLES Table1, Table2; // Immediate cleanup

STORE Result INTO [result.qvd] (qvd);
DROP TABLE Result;

LET vMemoryEnd = DocumentMemoryUsage();
TRACE Memory used: $(vMemoryEnd) MB;
```

### Incremental Loading Patterns
```sql
// Delta loading pattern
ExistingData:
LOAD * FROM [data.qvd] (qvd)
WHERE NOT EXISTS(NewKey);

CONCATENATE (ExistingData)
LOAD *, 
    ID as NewKey
FROM [source.csv]
WHERE ModifiedDate > '$(vLastLoadDate)';

STORE ExistingData INTO [data.qvd] (qvd);

// Partial reload pattern
ADD LOAD * FROM [new_data.csv];
REPLACE LOAD * FROM [updated_data.csv] WHERE UpdateFlag = 1;
```

## Data Quality & Validation

### Validation Framework
```sql
// Comprehensive data quality checks
DataQualityReport:
LOAD 
    'FactSales' as TableName,
    'Record Count' as CheckType,
    NoOfRows('FactSales') as ActualValue,
    1000000 as ExpectedValue,
    If(NoOfRows('FactSales') >= 1000000, 'PASS', 'FAIL') as Status
AUTOGENERATE 1;

CONCATENATE (DataQualityReport)
LOAD 
    'DimCustomer' as TableName,
    'Null CustomerName' as CheckType,
    Count(If(IsNull(CustomerName), CustomerID)) as ActualValue,
    0 as ExpectedValue,
    If(Count(If(IsNull(CustomerName), CustomerID)) = 0, 'PASS', 'FAIL') as Status
RESIDENT DimCustomer;

CONCATENATE (DataQualityReport)
LOAD 
    'FactSales' as TableName,
    'Orphaned Records' as CheckType,
    Count(If(NOT EXISTS(CustomerID), SalesKey)) as ActualValue,
    0 as ExpectedValue,
    If(Count(If(NOT EXISTS(CustomerID), SalesKey)) = 0, 'PASS', 'FAIL') as Status
RESIDENT FactSales;
```

## Best Practices Summary
- **Architecture**: Clear 3-stage separation, consistent naming (STG_/TRF_/Fact/Dim/Link)
- **Performance**: QVD optimization, AutoNumber() keys, immediate table drops
- **Relationships**: Prevent circular refs & synthetic keys, use link tables for M:M
- **SCD2**: LOAD WHILE pattern for performance, point-in-time fact integration
- **Quality**: Comprehensive validation, data lineage, audit trails
- **Maintenance**: Modular design, error handling, monitoring & alerting
- **Documentation**: Business rules, transformation logic, relationship design

