BatchCreateFortnoxPaymentFromRidecell
=======================================

SOQL query
-----------

The batch class retrieves objects from ``Ridecell_Invoice__c`` with below SOQL query:

.. code-block::

    SELECT Id
        ...,
        FROM Ridecell_Invoice__c
        WHERE 
            Fortnox_Invoice__c != null
            AND Fortnox_Invoice_Payments__c = null
            AND violation_type__c != 'Failed payment' + //We handle this type in a different batch job: BatchCreateSpecialFortnoxPayment
            AND state__c = 'PAID'


Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Ridecell_Invoice__c`` as parameter.
Before each ``Ridecell_Invoice__c`` is processed a rollback point is set if any errors occurs. 
An ``Fortnox_Invoice_Payments__c`` is then  created from the ``createInvoicePayment()`` method, 
where the ``Ridecell_Invoice__c`` is passed as an argument. 
The `Fortnox payment` is inserted after creation and it's `id` is assigned to the 
``ridecellInvoice.Fortnox_Invoice_Payments__c`` attribute as to create a relationship, before the payment is updated.  
``createInvoicePayment()`` is explained in further detail  :ref:`here <BatchCreateFortnoxPaymentFromRidecell:createInvoicePayment>`.

.. code-block:: javascript
    
    public void execute(Database.BatchableContext bc, List<Ridecell_Invoice__c> ridecellInvoices) {
        for (Ridecell_Invoice__c ridecellInvoice : ridecellInvoices) {
            //Set a savepoint to rollback to if any error occurs
            Savepoint savePoint = Database.setSavepoint();
            
            try {
                //Create the Salesforce Invoice Payment
                Fortnox_Invoice_Payments__c payment = createInvoicePayment(ridecellInvoice);

                //Assign the Id to the Ridecell Invoice
                ridecellInvoice.Fortnox_Invoice_Payments__c = payment.Id;
                update ridecellInvoice;
                
            } catch (Exception e) {
                //Rollback database changes...
                Database.rollback(savePoint);
                
                //...and log the error
                insert new Fortnox_Integration_Error_Log__c (
                    Message__c = e.getMessage(),
                    Trace__c = e.getStackTraceString(),
                    Source__c = 'BatchCreateFortnoxPaymentFromRidecell'
                );
            }
        }
    }


createInvoicePayment
-----------------------

``createInvoicePayment`` generates ``Fortnox_Invoice_Payments__c`` which is linked 
to a ``Ridecell_Invoice__c`` and a ``Fortnox_Invoice__c.``. 

.. code-block:: javascript

    public static Fortnox_Invoice_Payments__c createInvoicePayment(Ridecell_Invoice__c ridecellInvoice) {
        //Create the Salesforce Invoice Payment
        Fortnox_Invoice_Payments__c payment = new Fortnox_Invoice_Payments__c(
            Amount__c = ridecellInvoice.Total_Charge__c,
            Fortnox_Invoice__c = ridecellInvoice.Fortnox_Invoice__c,
            CurrencyIsoCode = ridecellInvoice.CurrencyIsoCode,
            Payment_Date__c = ridecellInvoice.CreatedDate.date()
        );
        
        //Insert it so we get an Id
        insert payment;
        
        return payment;
    }