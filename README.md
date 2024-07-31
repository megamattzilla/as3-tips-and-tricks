# AS3 Tips and Tricks! 

1.  [Increase Memory and Timeouts](#1)  
2.  [Use a Template](#labdevices)  
3.  [Refer to something outside AS3](#labnetworks)  
4.  [Shared AS3 Workflow](#publicip)  
5.  [Always Dry Run in Prod](#howtosteps1)  


### #1 Improve Big-IP REST durability by increasing memory and timeouts <a name="1"></a>
Per [AS3 Best Practices guide](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/best-practices.html#increase-timeout-values-if-the-rest-api-is-timing-out)   

Increase internal timeouts from 60 to 600 seconds:
```
tmsh modify sys db icrd.timeout value 600
tmsh modify sys db restjavad.timeout value 600
tmsh modify sys db restnoded.timeout value 600
```

Increase RESTJAVAD memory:
```
tmsh modify sys db icrd.timeout provision.extramb value 2048
tmsh modify sys db icrd.timeout provision.tomcat.extramb value 20

#TMOS pre 15.1.8: 
tmsh modify sys db icrd.timeout restjavad.useextramb value true 
#TMOS after 15.1.9: 
tmsh modify sys db icrd.timeout provision.restjavad.extramb value 600
```

then restart services
```
tmsh restart sys service restjavad
tmsh restart sys service restnoded
```

### #2 Use a Template <a name="2"></a>