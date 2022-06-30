BatchCreateFortnoxRefundPayment
=====================================

SOQL query
-----------

The batch class retrieves objects from ``Fortnox_Invoice__c`` with below SOQL query:

.. code-block::

    SELECT Id
        ..., 
        FROM Fortnox_Invoice__c
        WHERE 
            Refunds_Payment__c != null 
            AND Fortnox_Refund_Invoice_Payment__c = null


Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Fortnox_Invoice__c`` as parameter.
Before each ``Fortnox_Invoice__c`` is processed a rollback point is set if any errors occurs. 
An ``Fortnox_Invoice_Payments__c`` is then created from the ``createRefundPayment()`` method, 
where the ``Fortnox_Invoice__c`` is passed as an argument. 
The `Fortnox payment` is inserted after creation and it's `id` is assigned 
to the ``Payment__c.Fortnox_Invoice_Payments__c`` 
attribute as to create a relationship, before the payment is updated.  
``createRefundPayment()`` is explained in further detail 
:ref:`here <BatchCreateFortnoxRefundPayment:createRefundPayment()>`.

.. code-block:: javascript
    
    public void execute(Database.BatchableContext bc, List<sObject> sObjects) {
        List<Fortnox_Invoice__c> invoices = sObjects;
        
        for (Fortnox_Invoice__c invoice : invoices) {
            //Set a savepoint to rollback to if any error occurs
            Savepoint savePoint = Database.setSavepoint();
            
            try {
                //Create the Fortnox Refund Invoice Payment
                createRefundPayment(invoice);
            } catch (Exception e) {
                //Rollback database changes...
                Database.rollback(savePoint);
                
                //...and log the error
                insert new Fortnox_Integration_Error_Log__c (
                    Message__c = e.getMessage(),
                    Trace__c = e.getStackTraceString(),
                    Source__c = 'BatchCreateFortnoxRefundPayment'
                );
            }
        }
    }


createRefundPayment()
---------------------

``createRefundPayment`` generates ``Fortnox_Invoice_Payments__c`` which is linked 
to a ``Fortnox_Invoice__c.``. 

.. code-block:: javascript

    public static Fortnox_Invoice_Payments__c createRefundPayment(Fortnox_Invoice__c invoice) {
        //Create the Salesforce Invoice Payment
        Fortnox_Invoice_Payments__c fortnoxPayment = new Fortnox_Invoice_Payments__c(
            Fortnox_Invoice__c = invoice.Id,
            Amount__c = invoice.Fakturabelopp_Ink_Moms__c,
            CurrencyIsoCode = invoice.CurrencyIsoCode,
            Payment_Date__c = invoice.CreatedDate.date()
        );
        
        //Insert it so we get an Id
        insert fortnoxPayment;
        
        //Assign the Id to the Fortnox Refund Invoice
        invoice.Fortnox_Refund_Invoice_Payment__c = fortnoxPayment.Id;
        update invoice;
        
        return fortnoxPayment;
    }fortnoxPayment;
    }