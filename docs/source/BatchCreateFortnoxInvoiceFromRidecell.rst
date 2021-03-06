BatchCreateFortnoxInvoiceFromRidecell
=====================================

SOQL query
-----------

The batch class retrieves objects from ``Ridecell_Invoice__c`` with below SOQL query:

.. code-block::

    SELECT id__c, 
        ...,
        FROM Ridecell_Invoice__c
        WHERE 
            Fortnox_Invoice__c = null
            AND violation_type__c != 'Failed payment'
            AND CreatedDate >= 2020-12-01T11:00:00Z
            AND Rental_ID_Raw__c != null

Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Ridecell_Invoice__c`` as parameter.
Before each ``Ridecell_Invoice__c`` is processed a rollback point is set if any errors occurs. An ``Fortnox_Invoice__c`` is then created
from the ``createInvoice()`` method, where the ``Ridecell_Invoice__c`` is passed as an argument. 
The invoice is inserted after creation and it's `id` is assigned to the ``Ridecell_Invoice__c.Fortnox_Invoice__c`` 
attribute as to create a relationship, before the payment is updated. After the `invoice` is created, 
``createInvoiceRows()`` method is called, which populate the invoice with billable 
items, dependent on different attribute values in the ``Ridecell_Invoice__c`` that is passed as an argument. 
``createInvoiceRows()`` is explained in further detail :ref:`here <BatchCreateFortnoxInvoiceFromRidecell:createInvoiceRows()>`.

.. code-block:: javascript
    
    public void execute(Database.BatchableContext bc, List<Ridecell_Invoice__c> ridecellInvoices) {
        List<Fortnox_Invoice__c> sfInvoices = new List<Fortnox_Invoice__c>();
        for (Ridecell_Invoice__c ridecellInvoice : ridecellInvoices) {
            //Set a savepoint to rollback to if any error occurs
            Savepoint savePoint = Database.setSavepoint();
            
            try {
                //Create the Salesforce Invoice
                Fortnox_Invoice__c invoice = createInvoice(ridecellInvoice);
                
                //Create the Salesforce Invoice Rows
                createInvoiceRows(ridecellInvoice, invoice.Id);
                
            } catch (Exception e) {
                //Rollback database changes...
                Database.rollback(savePoint);
                
                //...and log the error
                insert new Fortnox_Integration_Error_Log__c (
                    Message__c = e.getMessage(),
                    Trace__c = e.getStackTraceString(),
                    Source__c = 'BatchCreateFortnoxInvoiceFromRidecell'
                );
            }
        }
    }


createInvoiceRows()
--------------------

``createInvoiceRows`` generates ``Fortnox_Invoice_Rows__c`` which are linked to a ``Fortnox_Invoice__c``. Multiple
`invoice rows` can be linked to a single `invoice`. Billable items from **Ridecell** are stipulated in ``Product2`` 
object in **Salesforce** include 
``Towing, Overduefee, Parkingfine, Excess, Billinginvoicetypephrasekeyrepair,
TrafficExces, Tolls, BillinginvoicetypephrasekeyspecialCleaning, Other``. 

.. code-block:: javascript

    public static List<Fortnox_Invoice_Rows__c> createInvoiceRows(Ridecell_Invoice__c ridecellInvoice, String invoiceId) {
        List<Fortnox_Invoice_Rows__c> invoiceRows = new List<Fortnox_Invoice_Rows__c>();

        invoiceRows.add(new Fortnox_Invoice_Rows__c(
            Product__c = FortnoxProductHelper.idByViolationType(
                ridecellInvoice.violation_type__c,
                ridecellInvoice.CurrencyIsoCode
            ),
            Antal__c = 1,
            A_Pris__c = ridecellInvoice.violation_amount__c,
            Row_Sum_With_VAT__c = ridecellInvoice.violation_amount__c * (1 + FortnoxProductHelper.vatByViolationType(
                ridecellInvoice.violation_type__c, ridecellInvoice.CurrencyIsoCode
            )),
            Fortnox_Invoice__c = invoiceId
        ));
        
        if (ridecellInvoice.handling_fee_amount__c > 0) {
            invoiceRows.add(new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idByViolationType(
                    'Administrative fee',
                    ridecellInvoice.CurrencyIsoCode
                ),
                Antal__c = 1,
                A_Pris__c = ridecellInvoice.handling_fee_amount__c / 1.25, //Ridecell's value is VAT included
                Row_Sum_With_VAT__c = ridecellInvoice.handling_fee_amount__c,
                Fortnox_Invoice__c = invoiceId
            ));
        }

        for (Fortnox_Invoice_Rows__c invoiceRow : invoiceRows) {
            invoiceRow.CurrencyIsoCode = ridecellInvoice.CurrencyIsoCode;
        }
        
        insert invoiceRows;
        
        return invoiceRows;
    }

