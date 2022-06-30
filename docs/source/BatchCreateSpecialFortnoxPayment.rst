BatchCreateSpecialFortnoxPayment
=====================================

SOQL query
-----------

The batch class retrieves objects from ``Ridecell_Invoice__c`` with below SOQL query:

.. code-block::

    SELECT Id
        ..., 
        FROM Ridecell_Invoice__c
        WHERE 
            Fortnox_Invoice__c = null
            AND Fortnox_Invoice_Payments__c = null
            AND violation_type__c = 'Failed payment'
            AND Rental_ID_Raw__c != null
            AND state__c = 'PAID'


Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Ridecell_Invoice__c`` as parameter.
Before each ``Ridecell_Invoice__c`` is processed a rollback point is set if any errors occurs. A ``Fortnox_Invoice__c`` 
matching the ``Ridecell_Invoice__c`` based on attributes is then retrieved in the ``getMatchingInvoice()`` method,
explained more :ref:`here<BatchCreateSpecialFortnoxPayment:getMatchingInvoice>`.

An ``Fortnox_Invoice_Payments__c`` is then created from the ``createInvoicePayment()`` method, 
where the ``Fortnox_Invoice__c.Id``,  ``Ridecell_Invoice__c.Total_Amount__c``, ``Ridecell_Invoice__c.CreatedDate.date()``,
``Ridecell_Invoice__c.CurrencyIsoCode`` is passed as arguments. 
The `Fortnox payment` is inserted after creation and it's `id` is assigned 
to the ``Ridecell_Invoice__c.Fortnox_Invoice_Payments__c`` 
attribute as to create a relationship, before the payment is updated.  
``createInvoicePayment()`` is explained in further detail 
:ref:`here <BatchCreateSpecialFortnoxPayment:createInvoicePayment>`.

.. code-block:: javascript
    
    public void execute(Database.BatchableContext bc, List<sObject> sObjects) {
        List<Ridecell_Invoice__c> ridecellInvoices = sObjects;
        
        for (Ridecell_Invoice__c ridecellInvoice : ridecellInvoices) {
            //Set a savepoint to rollback to if any error occurs
            Savepoint savePoint = Database.setSavepoint();
            
            try {
                //First we must find the corresponding Fortnox Invoice
                //that already has been created based on a failed Payment
                Fortnox_Invoice__c fortnoxInvoice = getMatchingInvoice(ridecellInvoice);
                
                if (fortnoxInvoice == null) {
                    //No match found
                    continue;
                }
                
                //Create the Fortnox Invoice Payment
                Fortnox_Invoice_Payments__c fortnoxPayment = createInvoicePayment(
                    fortnoxInvoice.Id,
                    ridecellInvoice.Total_Amount__c,
                    ridecellInvoice.CreatedDate.date(),
                    ridecellInvoice.CurrencyIsoCode
                );
                
                //Assign the payment and invoice lookups to the Ridecell Invoice
                ridecellInvoice.Fortnox_Invoice_Payments__c = fortnoxPayment.Id;
                ridecellInvoice.Fortnox_Invoice__c = fortnoxInvoice.Id;
                update ridecellInvoice;
                
            } catch (Exception e) {
                //Rollback database changes...
                Database.rollback(savePoint);
                
                //...and log the error
                insert new Fortnox_Integration_Error_Log__c (
                    Message__c = e.getMessage(),
                    Source__c = 'BatchCreateSpecialFortnoxPayment',
                    Trace__c = e.getStackTraceString()
                );
                
            }
        }
    }


getMatchingInvoice
---------------------

.. code-block:: javascript
    
    public static Fortnox_Invoice__c getMatchingInvoice(Ridecell_Invoice__c ridecellInvoice) {
        //Find the corresponding Fortnox Invoice
        //that already has been created based on a failed Payment
        
        String rentalId = String.valueOf((ridecellInvoice.Rental_ID_Raw__c).setScale(0));
        
        try {
            return [
                SELECT Id
                FROM Fortnox_Invoice__c
                WHERE Ert_ordernummer__c = :rentalId
                AND Antal_Fortnox_Betalningar__c = 0
                AND Fakturabelopp_Ink_Moms__c  = :ridecellInvoice.Total_Amount__c
                LIMIT 1
            ];
        } catch (Exception e) {
            return null;
        }
    }

createInvoicePayment
---------------------

``createRefundPayment`` generates ``Fortnox_Invoice_Payments__c`` which is linked 
to a ``Fortnox_Invoice__c.``. 

.. code-block:: javascript

    public static Fortnox_Invoice_Payments__c createInvoicePayment(String fortnoxInvoiceId, Decimal amount, Date paymentDate, String currencyIsoCode) {
        //Create the Salesforce Invoice Payment
        Fortnox_Invoice_Payments__c payment = new Fortnox_Invoice_Payments__c(
            Amount__c = amount,
            Fortnox_Invoice__c = fortnoxInvoiceId,
            CurrencyIsoCode = currencyIsoCode,
            Payment_Date__c = paymentDate
        );
        
        //Insert it so we get an Id
        insert payment;
        
        return payment;
    }