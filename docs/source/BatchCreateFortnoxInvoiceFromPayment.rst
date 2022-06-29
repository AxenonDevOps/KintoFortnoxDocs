BatchCreateFortnoxInvoiceFromPayment
----------------------------------------

The batch class retrieves objects from ``Payment__c`` with below SOQL query:

.. code-block::

    SELECT Id, 
    ...,
    FROM Payment__c 
    WHERE 
        Fortnox_Invoice__c = null
        AND STATE__c != 'CANCELLED'
        AND STATE__c != 'MEMBERSHIP FAILED PAYMENT'
        AND STATE__c != 'INCOMPLETE'
        AND STATE__c != 'REFUNDED'
        AND Fortnox_Sync_Onhold__c = false
        AND is_exempted__c = false
        AND CreatedDate >= 2020-12-01T11:00:00Z
        AND (
            block_fare__c > 1 
            OR additional_mileage_charge__c > 1 
            OR Credit_amount_used_N__c > 0 
            OR is_membership__c = true 
            OR (
                total_to_charge__c > 0 
                AND subscription_id__c != NULL 
                AND Type__c = lease 
            )
        )
        AND (
            Rental_ID_Raw__c != null 
            OR is_membership__c = true 
            OR (
                total_to_charge__c > 0 
                AND subscription_id__c != NULL 
                AND Type__c = 'lease' 
            )
        )
        AND (NOT (
            Is_B2B_Payment__c = true 
            AND Rental_Has_Ended__c = false
        ))

``STATE__c`` is a formula field that take on different values based on below conditions from
other fields in the ``Payment__c`` object:

.. code-block:: python

    if is_refunded__c == True:
        STATE__c = 'REFUNDED'
    elif is_cancelled_payment == True:
        STATE__c = 'CANCELLED'
    elif is_mempership__c == True and debit_failed_at__c != None:
        STATE__c = 'MEMBERSHIP FAILED PAYMENT'
    elif debit_failed_at__c != None:
        STATE__c = 'FAILED'
    elif Is_incomplete__c == True:
        STATE__c = 'INCOMPLETE'
    elif is_membership__c == True and is_membership_succes_and_refund__c == True and Refunded_Membership_Amount__c != None:
        STATE__c = 'MEMBERSHIP REFUNDED'
    else:
        STATE__c = 'SUCCESSED'


The SOQL query captures three states, mainly; ``FAILED``, ``MEMBERSHIP REFUNDED``, and ``SUCCESSED``. 
It then moves on to check if ``Payment__c`` is exempted from payment, e.g., "free of payment" and that
created date is later than 2020-12-01. 

The execution method 

.. code-block::
    
    public void execute(Database.BatchableContext bc, List<Payment__c> payments) {
        ...business logic
    }


