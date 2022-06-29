Data Models
===========

In this section, the different data models will be explained.

Payment Data Model
-----------------------

The Payment data model consists of several connected objects.
The main four objects in Salesforce for this data model is:

* Payment (Payment__c)
* Fortnox Invoice (Fortnox_Invoice__c)
* Fakturarad (Fortnox_Invoice_Rows__c)
* Fortnox Invoice Payment (Fortnox_Invoice_Payments__c)

``Payment__c`` objects are created by **MiracleMill** and sent to Kinto Salesforce instance.
``Fortnox_Invoice__c`` objects are created in:

* ``BatchCreateFortnoxInvoiceFromPayment.cls``
* ``BatchCreateFortnoxInvoiceFromRidecell.cls``

When a ``Fortnox_Invoice__c`` is created  ``Fortnox_Invoice_Rows__c`` are subsequently created and linked to to the 
``Fortnox_Invoice__c`` object by a lookup field. Respective ``Payment__c`` are also linked to ``Fortnox_Invoice__c``
by a lookup field.


Ridecell Data Model
---------------------------