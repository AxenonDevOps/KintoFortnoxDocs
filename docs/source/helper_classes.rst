Helper classes
==============

Below APEX classes are helpers to the methods in the different :doc:`/batch_classes`.

FortnoxProductHelper
---------------------

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