BatchCreateFortnoxInvoiceFromPayment
----------------------------------------

The batch class retrieves objects from ``Payment__c`` with below SOQL query:

.. code-block::
    
    SELECT Id, ...,
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
        AND (block_fare__c > 1 OR additional_mileage_charge__c > 1 OR Credit_amount_used_N__c > 0 OR is_membership__c = true OR (total_to_charge__c > 0 AND subscription_id__c != NULL AND Type__c = lease ))
        AND (Rental_ID_Raw__c != null OR is_membership__c = true OR (total_to_charge__c > 0 AND subscription_id__c != NULL AND Type__c = :picklastVal ))
        AND (NOT (Is_B2B_Payment__c = true AND Rental_Has_Ended__c = false))