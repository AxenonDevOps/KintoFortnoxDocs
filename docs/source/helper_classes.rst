Helper classes
==============

Below APEX classes are helpers to the methods in the different :doc:`/batch_classes`.

FortnoxProductHelper
---------------------

**FortnoxProductHelper** class is a wrapper class for 
:ref:`**fortnoxproductgeneralcache**<helper_classes:fortnoxproductgeneralcache>`.
``idByViolationType()``, ``idBySearchName()``, ``vatBySearchName()``, ``vatByViolationType()``,
returns the ``searchProduct()`` method which initiates a new ``ProductCacheSearch`` class with
arguments from the respective methods and serializes it to JSON. ``searchProduct()`` in turn calls 
``Cache.Org.get(cacheBuilder, key)`` where the class that implements ``cacheBuilder`` is ``FortnoxProductGeneralCache``.
This minimizes SOQL query run against the Product2 object since ``FortnoxProductGeneralCache`` caches the results of a
SOQL query. The **FortnoxProductHelper** is primairly used in :doc:`/BatchCreateFortnoxPaymentFromPayment`.

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

**FortnoxProductGeneralCache** implements ``Cache.CacheBuilder``, and is used to
retrieve values from Product2 object in **Salesforce**. If the record is not found
it throws an error if a record is not found.

.. code-block:: javascript

    public class FortnoxProductGeneralCache implements Cache.CacheBuilder {

    public class FortnoxProductGeneralCacheException extends Exception {}

    public Object doLoad(String json) {

        ProductCacheSearch search = ProductCacheSearch.parse(json);
        List<sObject> products = new List<sObject>();

        if (search.inputType == 'violationType') {
            String violationType = search.inputValue;
            products = [
                SELECT Id, VAT_Decimals__c FROM Product2
                WHERE Violation_Type_Formula__c = :violationType
                AND CurrencyIsoCode = :search.currencyIsoCOde
                LIMIT 1
            ];
        } else if (search.inputType == 'searchName') {
            products = [
                SELECT Id, VAT_Decimals__c FROM Product2
                WHERE Search_Name__c = :search.inputValue
                AND CurrencyIsoCode = :search.currencyIsoCOde
                LIMIT 1
            ];
        }
        
        if (products.isEmpty()) {
            throw new FortnoxProductGeneralCacheException(
                'The Product2 could not be found for ' + search.inputType +
                ' = ' + search.inputValue + ', (' + search.currencyIsoCOde + ')!');
        }
        
        return products[0];
    }
}

FortnoxProductCache
--------------------