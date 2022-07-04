PaymentRescueClient
====================

``PaymentRescueClient`` is a client used to call an **Azure Website** page 
containing information about ``Payment__c`` objects.  

.. code-block:: javascript
    
    public class PaymentRescueClient {
    
    public class PaymentRescueException extends Exception {}
    
    public static PaymentRescue findAndMerge(Integer paymentId, HttpResponse response) {
        //Parse the response
        List<PaymentRescue> rescues = PaymentRescue.parse(response.getBody());
        
        Boolean found = false;
        PaymentRescue mergedRescue = new PaymentRescue();
        
        for (PaymentRescue rescue : rescues) {
            if (Integer.valueOf(rescue.paymentId) == paymentId) {
                found = true;
                mergedRescue = mergeRescues(mergedRescue, rescue);
            }
        }
        
        return found ? mergedRescue : null;
    }
    
    public static PaymentRescue mergeRescues(PaymentRescue mergedRescue, PaymentRescue rescue) {
        if (rescue.rentalId != null) {
            mergedRescue.rentalId = rescue.rentalId;
        }
        
        if (rescue.customerId != null) {
            mergedRescue.customerId = rescue.customerId;
        }
        
        if (rescue.customerChargeId != null) {
            mergedRescue.customerChargeId = rescue.customerChargeId;
        }
        
        mergedRescue.isExempted = rescue.isExempted;
        mergedRescue.paymentId = rescue.paymentId;
        
        if (rescue.debitFailedAt != null) {
            mergedRescue.debitFailedAt = rescue.debitFailedAt;
        }
        
        if (rescue.scheduledRentalRequestId != null) {
            mergedRescue.scheduledRentalRequestId = rescue.scheduledRentalRequestId;
        }
        
        return mergedRescue;
    }
    
    public static HttpResponse get() {
        HttpRequest request = new HttpRequest();
        request.setEndpoint('callout:PaymentRescue');
        request.setMethod('GET');
        
        return send(request);
    }
    
    public static HttpResponse send(HttpRequest request) {
        HttpResponse response = (new Http()).send(request);
        
        if (! successful(response)) {
            throw new PaymentRescueException('Could not connect to Payment Rescue!');
        }
        
        return response;
    }
    
    public static Boolean successful(HttpResponse response) {
        return response.getStatusCode() >= 200 && response.getStatusCode() < 300;
    }
    
}
