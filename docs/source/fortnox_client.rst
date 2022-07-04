Fortnox client
==============

``FortnoxClient.cls`` is the APEX class used to make external calls to the Fortnox API.
**Fortnox** provides both `API DOCS <https://apps.fortnox.se/apidocs>`_ and 
`General Guidelines <https://developer.fortnox.se/general/>`_.


Headers
-----------

The base URL for the Fortnox API: ``https://api.fortnox.se/3``.

Below headers are used for authentication and to get response in the desired format:

.. list-table:: 
   :widths: 50 50
   :header-rows: 1

   * - Key
     - Value
   * - ``Accept``
     - ``application/json``
   * - ``Content-Type``
     - ``application/json``
   * - ``Client-Secret``
     - ``clientSecret()``
   * - ``Access-Token``
     - ``accessToken()``

``clientSecret()`` and ``accessToken()`` are method to retreive respective values which are stored 
as ``Custom Labels`` in Salesforce.

Methods
--------

Below follows a table with implemented methods in the Fortnox Client.

.. csv-table:: Table Title
   :file: FortnoxClientDocsTable.csv
   :widths: 25 25 5 10 15 20
   :header-rows: 1