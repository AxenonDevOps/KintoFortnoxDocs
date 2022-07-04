BatchRescuePaymentData
========================

SOQL query
-----------

The batch class retrieves objects from ``Payment__c`` with below SOQL query:

.. code-block::

    SELECT Id,
        ...,
        FROM Payment__c
        WHERE 
        Fortnox_Invoice__c = null
        AND LastModifiedDate > :periodStart
        AND (
            customer_charge_id__c = null 
            OR Customer_ID_Raw__c = null 
            OR is_exempted__c = null 
            OR debit_failed_at__c = null 
            OR Rental_ID_Raw__c = null 
            OR scheduled_rental_request_id_raw__c = null
        )
        AND Rescued_At__c = null


Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Payment__c`` as parameter.
For each ``Payment__c`` the method ``rescuedPayment()`` is called that executes the logic. More details
for the method is found :ref:`here<batchrescuepaymentdata:rescuedPayment()>`.
The purpose of the logic is to check and add missing data to the ``Payment__c``object. 
If a ``Payment__c`` has been amended by a log gets created and inserted into ``Payment_Rescue_Log__c``
object in **Salesforce**.

.. code-block:: javascript
    
        //Convert to payments
        List<Payment__c> payments = sObjects;
        
        List<Payment__c> rescuedPayments = new List<Payment__c>();
        List<Payment_Rescue_Log__c> log = new List<Payment_Rescue_Log__c>();
        
        for (Payment__c payment : payments) {
            Payment__c rescuedPayment = rescuedPayment(payment, response);
            
            //Skip if not found
            if (rescuedPayment == null) {
                continue;
            }
            
            rescuedPayment.Rescued_At__c = now;
            rescuedPayments.add(rescuedPayment);
            log.add(new Payment_Rescue_Log__c(
                Payment__c = rescuedPayment.Id
            ));
        }
        
        if (! rescuedPayments.isEmpty()) {
            update rescuedPayments;
        }
        
        if (! log.isEmpty()) {
            insert log;
        }
    }


rescuedPayment()
------------------

``rescuedPayment()`` takes a ``Payment__c``object and a ``HttpResponse`` as arguments. 
If the ``HttpResponse`` is empty, it calls the ``PaymentRescueClient.get()`` method.
More info about the *PaymentRescueClient* can be found :doc:`here</payment_rescue_client>`.
The method sends a GET request to *PaymentRescue*, which is an endpoint stored
as a Named Credential in **Salesforce**. The endpoint returns a bulk JSON response,
meaning it only needs to be called once. The response and the exernal *Payment Id*
is then past as argument to ``findAndMerge()``. The method parses the JSON response
into ``PaymentRescue`` objects and then iterates over those objects trying to match
*Payment Id* with an object in the parsed JSON response. If a match is found, attribute
values from the object in the response is past the to ``Payment__c``, in the ``mergeRescues()``
method.

.. code-block:: javascript

    private static Payment__c rescuedPayment(Payment__c payment, HttpResponse response) {
        //We only fetch the response once since it can be used for all payments
        if (response == null) {
            response = PaymentRescueClient.get();
        }
        
        PaymentRescue rescue = PaymentRescueClient.findAndMerge((Integer)payment.payment_id__c, response);
     
        //Check if there is no need for rescue
        if (rescue == null || equals(payment, rescue)) {
            return null;
        }
   
        //Overwrite old data if new data is not null
        payment.customer_charge_id__c = (rescue.customerChargeId == null ? payment.customer_charge_id__c : rescue.customerChargeId);
        payment.Customer_ID_Raw__c = (rescue.customerId == null ? payment.Customer_ID_Raw__c : rescue.customerId);
        payment.is_exempted__c = (rescue.isExempted == null ? payment.is_exempted__c : rescue.isExempted);
        payment.debit_failed_at__c = (rescue.debitFailedAt == null ? payment.debit_failed_at__c : rescue.debitFailedAt);
        payment.Rental_ID_Raw__c = (rescue.rentalId == null ? payment.Rental_ID_Raw__c : rescue.rentalId);
        payment.scheduled_rental_request_id_raw__c = (rescue.scheduledRentalRequestId == null ? payment.scheduled_rental_request_id_raw__c : rescue.scheduledRentalRequestId);
        
        return payment;
    }


