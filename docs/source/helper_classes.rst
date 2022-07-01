Helper classes
==============

Below APEX classes are helpers to the methods in the different :doc:`/batch_classes`.

FortnoxProductHelper
---------------------

**FortNoxProductHelper** class is a wrapper class for 
:ref:`**fortnoxproductgeneralcache**<helper_classes:fortnoxproductgeneralcache>`.
``idByViolationType()``, ``idBySearchName()``, ``vatBySearchName()``, ``vatByViolationType()``,
returns the ``searchProduct()`` method which initiates a new ``ProductCacheSearch`` class with
arguments from the respective methods and serializes it to JSON. ``searchProduct()`` in turn calls 
``Cache.Org.get(cacheBuilder, key)`` where the class that implements ``cacheBuilder`` is ``FortnoxProductGeneralCache``.
This minimizes SOQL query run against the Product2 object since ``FortnoxProductGeneralCache`` caches the results of a
SOQL query.

.. code-block:: javascript
    
    public class FortnoxProductHelper {
    
    public static String idByViolationType(String violationType, String isoCurrencyCode) {
        return searchProduct(new ProductCacheSearch('violationType', violationType, isoCurrencyCode)).Id;
    }

    public static String idBySearchName(String searchName, String isoCurrencyCode) {
        return searchProduct(new ProductCacheSearch('searchName', searchName, isoCurrencyCode)).Id;
    }

    public static Decimal vatBySearchName(String name, String isoCurrencyCode) {
        return searchProduct(new ProductCacheSearch('searchName', name, isoCurrencyCode)).VAT_Decimals__c;
    }

    public static Decimal vatByViolationType(String violationType, String isoCurrencyCode) {
        return searchProduct(new ProductCacheSearch('violationType', violationType, isoCurrencyCode)).VAT_Decimals__c;
    }

    public static Product2 searchProduct(ProductCacheSearch search) {
        return ((Product2)Cache.Org.get(FortnoxProductGeneralCache.class, search.toJson()));
    }
    
}

FortnoxProductGeneralCache
---------------------------

FortnoxProductCache
--------------------