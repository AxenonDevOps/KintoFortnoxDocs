BatchCreateFortnoxPaymentFromPayment
=====================================

SOQL query
-----------

The batch class retrieves objects from ``Payment__c`` with below SOQL query:

.. code-block::

    SELECT Id,
        ...,
        FROM Payment__c
        WHERE 
            Fortnox_Invoice__c != null
            AND Fortnox_Invoice_Payments__c = null
            AND Fortnox_Invoice__r.Invoice_Sum__c != 0 
            AND STATE__c != 'FAILED' +
            AND (
                block_fare__c > 1 
                OR additional_mileage_charge__c > 1 
                OR credit_amount_used__c > '0' 
                OR is_membership__c = true 
                OR (
                    total_to_charge__c > 0 
                    AND subscription_id__c != NULL 
                    AND Type__c = lease 
                )
            );


Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Payment__c`` as parameter.
Before each ``Payment__c`` is processed a rollback point is set if any errors occurs. An ``Fortnox_Invoice_Payments__c`` 
is then  created from the ``createPayment()`` method, where the ``Payment__c`` is passed as an argument. 
The `Fortnox payment` is inserted after creation and it's `id` is assigned to the ``Payment__c.Fortnox_Invoice_Payments__c`` 
attribute as to create a relationship, before the payment is updated.  
``createPayment()`` is explained in further detail 
:ref:`here <BatchCreateFortnoxInvoiceFromRidecell:createPayment>`.

.. code-block:: javascript
    
    public void execute(Database.BatchableContext bc, List<Payment__c> payments) {
        for (Payment__c payment : payments) {
            //Set a savepoint to rollback to if any error occurs
            Savepoint savePoint = Database.setSavepoint();
            
            try {
                //Create the Salesforce Invoice Payment
                Fortnox_Invoice_Payments__c fortnoxPayment = createPayment(
                    payment,
                    payment.Fortnox_Invoice__c,
                    payment.total_to_charge__c
                );
            
                //Assign the Id to the Ridecell Invoice
                payment.Fortnox_Invoice_Payments__c = fortnoxPayment.Id;
                update payment;
                
            } catch (Exception e) {
                //Rollback database changes...
                Database.rollback(savePoint);
                
                //...and log the error
                insert new Fortnox_Integration_Error_Log__c (
                    Message__c = e.getMessage(),
                    Trace__c = e.getStackTraceString(),
                    Source__c = 'BatchCreateFortnoxPaymentFromPayment'
                );
            }
        }
    }


createPayment
------------------

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

