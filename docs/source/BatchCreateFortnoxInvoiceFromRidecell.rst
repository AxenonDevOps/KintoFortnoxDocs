BatchCreateFortnoxInvoiceFromRidecell
-------------------------------------

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