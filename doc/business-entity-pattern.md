# ABL Business Entity Architecture Pattern

## Overview

The Business Entity pattern provides a standardized, maintainable approach to data access in OpenEdge ABL applications. It separates UI logic from database operations through a layered architecture that promotes reusability, testability, and consistency across the application.

## Architecture Layers

### 1. UI Layer (Windows/Forms)
- **Responsibility**: User interaction and presentation
- **Access**: Never directly accesses database tables
- **Communication**: Calls Business Entity methods with datasets

### 2. Business Entity Layer
- **Responsibility**: Data access, business rules, validation
- **Inheritance**: Extends `OpenEdge.BusinessLogic.BusinessEntity`
- **Management**: Instantiated through EntityFactory (singleton pattern)

### 3. Database Layer
- **Responsibility**: Persistent storage
- **Access**: Only through data-sources attached to business entities

## Key Components

### EntityFactory (Singleton Pattern)

**Purpose**: Centralized management of business entity lifecycle

```abl
CLASS business.EntityFactory:
    /* Singleton instance */
    VAR PRIVATE STATIC EntityFactory objInstance.
    
    /* Entity instances */
    VAR PRIVATE CustomerEntity objCustomerEntityInstance.
    
    /* Private constructor prevents direct instantiation */
    CONSTRUCTOR PRIVATE EntityFactory():
    END CONSTRUCTOR.
    
    /* Public factory method */
    METHOD PUBLIC STATIC EntityFactory GetInstance():
        IF objInstance = ? THEN
            objInstance = NEW EntityFactory().
        RETURN objInstance.
    END METHOD.
    
    /* Entity getters with lazy initialization */
    METHOD PUBLIC CustomerEntity GetCustomerEntity():
        IF objCustomerEntityInstance = ? THEN
            objCustomerEntityInstance = NEW CustomerEntity().
        RETURN objCustomerEntityInstance.
    END METHOD.
END CLASS.
```

### Dataset Definition (.i Include Files)

```abl
/* Define temp-table with BEFORE-TABLE for change tracking */
DEFINE TEMP-TABLE ttCustomer BEFORE-TABLE bttCustomer
    FIELD CustNum AS INTEGER INITIAL "0" LABEL "Cust Num"
    FIELD Name AS CHARACTER LABEL "Name"
    /* ... additional fields ... */
    INDEX CustNum IS PRIMARY UNIQUE CustNum ASCENDING.

/* Define dataset containing the temp-table */
DEFINE DATASET dsCustomer FOR ttCustomer.
```

### Business Entity Class

```abl
CLASS business.CustomerEntity INHERITS BusinessEntity USE-WIDGET-POOL:
    
    /* Include dataset definition */
    {business/CustomerDataset.i}
    
    /* Define data sources - one per database table */
    DEFINE DATA-SOURCE srcCustomer FOR Customer.
    
    CONSTRUCTOR PUBLIC CustomerEntity():
        /* Pass dataset handle to parent class */
        SUPER(DATASET dsCustomer:HANDLE).

        /* Create array of data source handles */
        VAR HANDLE[1] hDataSourceArray = DATA-SOURCE srcCustomer:HANDLE.
        VAR CHARACTER[1] cSkipListArray = [""].
        
        /* Set parent class properties */
        THIS-OBJECT:ProDataSource = hDataSourceArray.
        THIS-OBJECT:SkipList = cSkipListArray.
    END CONSTRUCTOR.

END CLASS.
```

## Standard CRUD Operations

### Read (Query) Operations

```abl
METHOD PUBLIC LOGICAL GetCustomerByNumber(INPUT ipiCustNum AS INTEGER,
                                          OUTPUT DATASET dsCustomer):
    VAR CHARACTER cFilter.
    VAR LOGICAL lFound = FALSE.
    
    /* Build WHERE clause for database query */
    cFilter = "WHERE Customer.CustNum = " + STRING(ipiCustNum).
    
    /* Parent class ReadData() executes query and fills temp-table */
    THIS-OBJECT:ReadData(cFilter).

    /* Check if data was found */
    lFound = CAN-FIND(FIRST ttCustomer).
    
    RETURN lFound.
END METHOD.
```

### Update Operations

```abl
METHOD PUBLIC VOID UpdateCustomer(INPUT-OUTPUT DATASET dsCustomer):
    /* Parent class detects changes and updates database */
    THIS-OBJECT:UpdateData(DATASET dsCustomer BY-REFERENCE).
END METHOD.
```

## UI Integration Pattern

```abl
/* Include business entity classes */
USING business.CustomerEntity FROM PROPATH.
USING business.EntityFactory FROM PROPATH.

/* Include dataset definition */
{business/CustomerDataset.i}

/* Button event handler */
ON CHOOSE OF GetCustomer IN FRAME DEFAULT-FRAME:
    VAR INTEGER iCustomerNumber = INTEGER(CustomerNumber:screen-value).
    VAR EntityFactory objFactory = EntityFactory:GetInstance().
    VAR CustomerEntity objCustomerEntity = objFactory:GetCustomerEntity().
    VAR LOGICAL lCustomerFound.
    
    /* Call entity to fetch data - use OUTPUT DATASET */
    lCustomerFound = objCustomerEntity:GetCustomerByNumber(
        iCustomerNumber, 
        OUTPUT DATASET dsCustomer
    ).
    
    /* Update UI based on results */
    IF lCustomerFound THEN DO:
        FIND FIRST ttCustomer.
        IF AVAILABLE ttCustomer THEN DO:
            CustomerName = ttCustomer.Name.
            DISPLAY CustomerName WITH FRAME {&frame-name}.
        END.
    END.
    ELSE 
        MESSAGE "Customer not found" VIEW-AS ALERT-BOX.
END.
```