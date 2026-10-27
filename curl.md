curl --location --request POST 'https://api-sandbox.duitku.com/api/merchant/createInvoice' \
--header 'x-duitku-signature: f40731428e7c3669bed8669b660afa4ef619754af46dfb2ad58b3ce223209bcf' \
--header 'x-duitku-timestamp: 1629219840000' \
--header 'x-duitku-merchantcode: DXXXX' \
--header 'Content-Type: application/json' \
--data-raw
{ 
   "paymentAmount":40000,
   "merchantOrderId":"1579838431",
   "productDetails":"Test Pay with duitku",
   "additionalParam":"",
   "merchantUserInfo":"",
   "paymentMethod":"",
   "customerVaName":"John Doe",
   "email":"test@test.com",
   "phoneNumber":"08123456789",
   "itemDetails":[ 
      { 
         "name":"Test Item 1",
         "price":10000,
         "quantity":1
      },
      { 
         "name":"Test Item 2",
         "price":30000,
         "quantity":3
      }
   ],
   "customerDetail":{ 
      "firstName":"John",
      "lastName":"Doe",
      "email":"test@test.com",
      "phoneNumber":"08123456789",
      "billingAddress":{ 
         "firstName":"John",
         "lastName":"Doe",
         "address":"Jl. Kembangan Raya",
         "city":"Jakarta",
         "postalCode":"11530",
         "phone":"08123456789",
         "countryCode":"ID"
      },
      "shippingAddress":{ 
         "firstName":"John",
         "lastName":"Doe",
         "address":"Jl. Kembangan Raya",
         "city":"Jakarta",
         "postalCode":"11530",
         "phone":"08123456789",
         "countryCode":"ID"
      },
      "merchantCustomerId":""
   },
   "creditCardDetail": {
        "saveCardToken": 0
   },
   "callbackUrl":"http:\/\/example.com\/api-pop\/backend\/callback.php",
   "returnUrl":"http:\/\/example.com\/api-pop\/backend\/redirect.php",
   "expiryPeriod":10
}
