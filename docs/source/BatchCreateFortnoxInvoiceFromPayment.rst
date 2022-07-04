BatchCreateFortnoxInvoiceFromPayment
=====================================

SOQL query
-----------

The batch class retrieves objects from ``Payment__c`` with below SOQL query.

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
                AND Type__c = 'lease'
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

Execution method
-----------------

The class implements ``Database.Batchable<sObject>`` and executes with a list of ``Payment__c`` as parameter.
Before each ``Payment__c`` is processed a rollback point is set if any errors occurs. An ``Fortnox_Invoice__c`` is then created
from the ``createInvoice()`` method, where the ``Payment__c`` is passed 
as an argument. The `Fortnox invoice` is inserted after creation and it's `id` is assigned to the 
``Payment__c.Fortnox_Invoice__c``  attribute as to create a relationship, before the payment is updated. After the `invoice` is created, 
``createInvoiceRows()`` method is called, which populate the invoice with billable 
items, dependent on different attribute values in the ``Payment__c`` that is passed as an argument. 
``createInvoiceRows()`` is explained in further detail :ref:`here <BatchCreateFortnoxInvoiceFromPayment:createInvoiceRows()>`.

.. code-block:: javascript
    
    public void execute(Database.BatchableContext bc, List<Payment__c> payments) {
        List<Fortnox_Invoice__c> sfInvoices = new List<Fortnox_Invoice__c>();
        for (Payment__c payment : payments) {
            //Set a savepoint to rollback to if any error occurs
            Savepoint savePoint = Database.setSavepoint();
            
            try {
                //Create the Salesforce Invoice
                Fortnox_Invoice__c invoice = createInvoice(payment);

                //Create the Salesforce Invoice Rows
                createInvoiceRows(payment, invoice.Id);
    

Before finishing the the ``Payment__c`` object is updated with summurized credit and debit amount from
``Fortnox_Invoice_Rows__c`` objects matching the `payment.id` and `invoice.id` respectivly.

.. code-block:: javascript

                //And rollup the invoice sums
                FortnoxCreditTotals creditTotals = new FortnoxCreditTotals(payment);
                FortnoxDebitTotals debitTotals = new FortnoxDebitTotals(invoice.Id);
                payment.Fortnox_Refunded_Total__c = creditTotals.includingVat;
                payment.Fortnox_Refunded_Total_Ex_Vat__c = creditTotals.excludingVat;
                payment.Fakturerat_inkl_moms_N__c  = debitTotals.includingVat;
                payment.Fakturerat_exkl_moms_N__c = debitTotals.excludingVat;

                if (payment.Refunded_Membership_Amount__c != null) {
                    payment.is_membership_success_and_refund__c = true;
                }

                update payment;
                
            } catch (Exception e) {
                //Rollback database changes...
                Database.rollback(savePoint);
                //...and log the error
                insert new Fortnox_Integration_Error_Log__c (
                    Message__c = e.getMessage(),
                    Trace__c = e.getStackTraceString(),
                    Source__c = 'BatchCreateFortnoxInvoiceFromPayment'
                );
                System.debug('BatchCreateFortnoxInvoiceFromPayment exception: ' + e);
            }
        }
    }

If any errors occur, the database is rolled back to the latest savepoint and an error log is inserted into 
the ``Fortnox_Integration_Error_Log__c`` object.

createInvoiceRows()
--------------------

``createInvoiceRows`` generates ``Fortnox_Invoice_Rows__c`` which are linked to a ``Fortnox_Invoice__c``. Multiple
`invoice rows` can be linked to a single `invoice`. Before any `invoice rows` are created, a sign factor is set
as to accommodate `payments` that have been refunded. If a `payment` has been refunded, indicated by the ``STATE__c``
attribute, the factor is set to ``-1``. 

.. code-block:: javascript

        public static List<Fortnox_Invoice_Rows__c> createInvoiceRows(Payment__c payment, String invoiceId) {
        List<Fortnox_Invoice_Rows__c> invoiceRows = new List<Fortnox_Invoice_Rows__c>();
        Decimal factor = (payment.STATE__c == 'REFUNDED' ? -1 : 1);
        ....

.. list-table:: Fortnox Invoice Rows Conditional Table


 * - **Invoice Type**
   - **Condition**
   - **Value**
   - **Method**
 * - Membership
   - ``is_membership__c``
   - ``true``
   - :ref:`membership()<BatchCreateFortnoxInvoiceFromPayment:Membership>`
 * - Milage charge
   - ``additional_mileage_charge__c``
   - ``> 1``
   - :ref:`milageCharge()<BatchCreateFortnoxInvoiceFromPayment:Milage charge>`
 * - Late Return Fee
   - ``Late_Return_Fee__c``
   - ``> 0``
   - :ref:`lateReturnFee()<BatchCreateFortnoxInvoiceFromPayment:Late Return Fee>`
 * - Promo Credit
   - ``Credit_amount_used_N__c``
   - ``> 0``
   - :ref:`promoCredit()<BatchCreateFortnoxInvoiceFromPayment:Promo Credit>`
 * - Addon Charge
   - ``addon_charge__c``
   - ``> 0``
   - :ref:`addonCharge()<BatchCreateFortnoxInvoiceFromPayment:Addon Charge>`
 * - Block Fare
   - | ``block_fare__c``
     | ``subscription_id__c``
     | ``Type__c``
   - | ``> 1``
     | ``NULL``
     | ``lease``
   - :ref:`blockFare()<BatchCreateFortnoxInvoiceFromPayment:Block Fare>`
 * - Flex Booking Time
   - | ``is_membership__c``
     | ``total_to_charge__c``
     | ``Type__c``
   - | ``false``
     | ``NULL``
     | ``lease``
   - :ref:`flexBookingTime()<BatchCreateFortnoxInvoiceFromPayment:Flex Booking Time>`



Membership
^^^^^^^^^^^
If the customer has a memberhip, indicated by ``Payment.is_membership__c``, only one `invoice row` will be inserted.

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c membership(Payment__c payment, String invoiceId, Decimal factor, String currencyIsoCode) {
        Decimal vatRate = FortnoxProductHelper.vatBySearchName(
            'subscription_temp',
            currencyIsoCode
        );

        Decimal price = payment.total_to_charge__c;

        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName(
                    'subscription_temp',
                    currencyIsoCode
                ),
                Antal__c = 1,
                A_Pris__c = factor * price / (1 + vatRate),
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = factor * price
            );
    }

Block fare
^^^^^^^^^^^

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c blockFare(Decimal blockFare, String invoiceId, Decimal factor, String currencyIsoCode) {
        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName('booking', currencyIsoCode),
                Antal__c = 1,
                A_Pris__c = blockFare * factor,
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = blockFare * factor * (1 + FortnoxProductHelper.vatBySearchName('booking', currencyIsoCode))
            );
    }

Milage charge
^^^^^^^^^^^^^^

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c milageCharge(Decimal mileageCharge, String invoiceId, Decimal factor, String currencyIsoCode) {
        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName('usage', currencyIsoCode),
                Antal__c = 1,
                A_Pris__c = mileageCharge * factor,
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = mileageCharge * factor * (1 + FortnoxProductHelper.vatBySearchName('usage', currencyIsoCode))
            );
    }

Late return fee
^^^^^^^^^^^^^^^^

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c lateReturnFee(Decimal lateReturnFee, String invoiceId, Decimal factor, String currencyIsoCode) {
        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName('booking', currencyIsoCode),
                Antal__c = 1,
                A_Pris__c = lateReturnFee * factor,
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = lateReturnFee * factor * (1 + FortnoxProductHelper.vatBySearchName('booking', currencyIsoCode))
            );
    }

Promo credit
^^^^^^^^^^^^^

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c promoCredit(Decimal creditAmountUsed, String invoiceId, Decimal factor, String currencyIsoCode) {
        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName('promo_credits', currencyIsoCode),
                Antal__c = 1,
                A_Pris__c = -creditAmountUsed * factor / 1.25, //VAT excluded
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = -creditAmountUsed * factor
            );
    }

Addon charge
^^^^^^^^^^^^^

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c addonCharge(Decimal addonCharge, String invoiceId, Decimal factor, String currencyIsoCode) {
        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName('insurance', currencyIsoCode),
                Antal__c = 1,
                A_Pris__c = addonCharge * factor,
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = addonCharge * factor
            );
    }

Flex Booking Time
^^^^^^^^^^^^^^^^^^^^

.. code-block:: javascript

    public static Fortnox_Invoice_Rows__c flexBookingTime(Decimal flexBookingTime, String invoiceId, Decimal factor, String currencyIsoCode) {
        return new Fortnox_Invoice_Rows__c(
                Product__c = FortnoxProductHelper.idBySearchName('flextime', currencyIsoCode),
                Antal__c = 1,
                A_Pris__c = flexBookingTime / (1 + FortnoxProductHelper.vatBySearchName('flextime', currencyIsoCode)) ,
                Fortnox_Invoice__c = invoiceId,
                Row_Sum_With_VAT__c = flexBookingTime 
            );
    }