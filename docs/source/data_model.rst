Data Models
===========

In this section, the different data models will be explained.

Payment Data Model
-----------------------

The Payment data model consists of several connected objects.
The main four objects in Salesforce for this data model is:

* Payment
* Fortnox Invoice
* Fortnox Invoice Rows
* Fortnox Invoice Payment

.. graphviz::
    digraph LookupRelations {
        Payment -> Fortnox_Invoice [arrowtail = "crow", dir="back"]
        Payment -> Fornox_Invoice_Payment [arrowtail = "crow", dir="back"]
        Fortnox_Invoice -> Fortnox_Invoice_Rows [arrowhead = "crow"]
        Fortnox_Invoice -> Fornox_Invoice_Payment [arrowhead = "crow"]
    }



Ridecell Data Model
---------------------------