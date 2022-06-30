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
            )


Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Payment__c`` as parameter.
Before each ``Payment__c`` is processed a rollback point is set if any errors occurs. An ``Fortnox_Invoice_Payments__c`` 
is then  created from the ``createPayment()`` method, where the ``Payment__c`` is passed as an argument. 
The `Fortnox payment` is inserted after creation and it's `id` is assigned to the ``Payment__c.Fortnox_Invoice_Payments__c`` 
attribute as to create a relationship, before the payment is updated.  
``createPayment()`` is explained in further detail 
:ref:`here <BatchCreateFortnoxPaymentFromPayment:createPayment()>`.

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


createPayment()
------------------

``createPayment`` generates ``Fortnox_Invoice_Payments__c`` which is linked 
to a ``Payment__c`` and a ``Fortnox_Invoice__c.``. 

.. code-block:: javascript

     public static Fortnox_Invoice_Payments__c createPayment(Payment__c payment, String invoiceId, Decimal amount) {
        //Create the Salesforce Invoice Payment
        Fortnox_Invoice_Payments__c fortnoxPayment = new Fortnox_Invoice_Payments__c(
            Fortnox_Invoice__c = invoiceId,
            Amount__c = amount,
            CurrencyIsoCode = payment.CurrencyIsoCode,
            Payment_Date__c = payment.CreatedDate.date()
        );
        
        
        //Insert it so we get an Id
        insert fortnoxPayment;

        
        //Assign the Id to the Ridecell Invoice
        payment.Fortnox_Invoice_Payments__c = fortnoxPayment.Id;
        update payment;
        
        return fortnoxPayment;
    }