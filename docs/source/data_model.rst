Data Models
===========

In this section, the different data models will be explained. Each batch class is explained in further detail
in it's respective page.

Payment Data Model
-----------------------

The **Payment Data Model** consists of several connected objects, branching from the ``Payment__c`` object.
The main four objects in Salesforce for this data model is:

* Payment (``Payment__c``)
* Fortnox Invoice (``Fortnox_Invoice__c``)
* Fakturarad (``Fortnox_Invoice_Rows__c``)
* Fortnox Invoice Payment (``Fortnox_Invoice_Payments__c``)

``Payment__c`` objects are created by **MiracleMill** and sent to Kinto Salesforce instance via Azure Event Hub.

``Fortnox_Invoice__c`` and ``Fortnox_Invoice_Rows__c`` objects are created in the following batch class:

* :ref:`BatchCreateFortnoxInvoiceFromPayment`


When a ``Fortnox_Invoice__c`` is created ``Fortnox_Invoice_Rows__c`` are subsequently created and linked to to the 
``Fortnox_Invoice__c`` object by a lookup field. Respective ``Payment__c`` are also linked to ``Fortnox_Invoice__c``
by a lookup field.

``Fortnox_Invoice_Payments__c`` objects are created in the following batch classes:

* :ref:`BatchCreateFortnoxPaymentFromPayment`
* :ref:`BatchCreateFortnoxRefundPayment`
* :ref:`BatchCreateSpecialFortnoxPayment`

In short, these batch classes generete ``Fortnox_Invoice_Payments__c`` objects from ``Payment__c`` where a 
``Fortnox_Invoice__c`` has been previously created and linked to the ``Payment__c``. After creation of the the 
``Fortnox_Invoice_Payments__c`` it is linked to the ``Payment__c`` which it was created from.



Ridecell Data Model
---------------------------

The **Ridecell Data Model** has similair structure as the **Payment Data Model**,
and branches out from the Ridecell Invoice ``(Ridecell_Invoice__c)``.

``Fortnox_Invoice__c`` and ``Fortnox_Invoice_Rows__c`` objects are created in the following batch class:

* :ref:`BatchCreateFortnoxInvoiceFromRidecell`

The only difference from the **Payment flow** being that the objects are linked to ``Ridecell_Invoice__c`` 
instead of ``Payment__c`` objects.

* :ref:`BatchCreateFortnoxPaymentFromRidecell`

Above batch class is responsible for creating ``Fortnox_Invoice_Payments__c``object and
has the same logic as in the **Payment Flow**, only difference beign that the created objects are
linked to ``Ridecell_Invoice__c`` instead of ``Payment__c`` objects.