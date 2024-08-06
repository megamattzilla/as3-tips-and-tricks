# AS3 Tips and Tricks! 

UNDER DEVELOPMENT

1.  [Increase Memory and Timeouts](#1)  
2.  [Use a Templating Engine](#2)  
3.  [Multi-Team: Refer to something outside AS3](#3)  
4.  [Multi-Team: Shared AS3 Workflow](#4)  
5.  [Always Dry Run in Prod](#5)  


## #1 Increase memory and timeouts to improve Big-IP REST <a name="1"></a>
Per [AS3 Best Practices guide](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/best-practices.html#increase-timeout-values-if-the-rest-api-is-timing-out)   

Increase internal timeouts from 60 to 600 seconds:
```
tmsh modify sys db icrd.timeout value 600
tmsh modify sys db restjavad.timeout value 600
tmsh modify sys db restnoded.timeout value 600
```

Increase RESTJAVAD memory (skip if experiencing memory pressure):
```
tmsh modify sys db provision.extramb value 2048
tmsh modify sys db provision.tomcat.extramb value 20
```
One of the following:    
Before TMOS 15.1.8:   
`tmsh modify sys db icrd.timeout restjavad.useextramb value true`   
OR  
After TMOS 15.1.9:  
`tmsh modify sys db icrd.timeout provision.restjavad.extramb value 600`  

save configuration:
```
tmsh save sys config
```
verify everything looks good:
```
tmsh list sys db icrd.timeout
tmsh list sys db restjavad.timeout  
tmsh list sys db restnoded.timeout 
tmsh list sys db provision.extramb
tmsh list sys db provision.tomcat.extramb
tmsh list sys db provision.restjavad.extramb
```

then restart services:
```
tmsh restart sys service restjavad
tmsh restart sys service restnoded
```

___

## #2 Use a Templating Engine <a name="2"></a>
Two common templating engines for AS3 schema is [FAST](https://clouddocs.f5.com/products/extensions/f5-appsvcs-templates/latest/) and [jinja2](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html)

To see how these templating engines can help, lets take a look at the AS3 schema [getting started simple HTTP server](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/declarations/getting-started.html#simple-http-application) in YAML  
* Omitted first 9 lines for brevity  

```yaml
    Sample_01:
        class: Tenant

        A1:
            class: Application

            service:
                class: Service_HTTP
                virtualAddresses:
                    - 10.0.1.10
                pool: web_pool

            web_pool:
                class: Pool
                monitors:
                    - http
                members:
                    -
                        servicePort: 80
                        serverAddresses:
                            - 192.0.1.10
                            - 192.0.1.11
```
This AS3 schema in 18 lines declares:
- One Tenant `Sample_01`
    - One Application `A1` 
        - One HTTP virtual server named `service`
            - IP 10.0.1.10
            - default pool `web_pool`    
        - One pool named `web_pool`
            - with members
                -   192.0.1.10:80
                -   192.0.1.11:80

What if we need to add two more virtual servers and two more pools?

Instead of copy/paste those 18 lines of YAML and making modifications, we can generate the AS3 schema using a jina2 template!  

Our inputs to the template will be the three virtual servers:
```YAML
    VIRTUALS:
      - name: "example1"
        virtualAddresses: "10.0.1.10"
        default_pool: "web_pool1"
      - name: "example2"
        virtualAddresses: "10.0.1.11"
        default_pool: "web_pool2"
      - name: "example3"
        virtualAddresses: "10.0.1.12"
        default_pool: "web_pool3"
```
and the three pools:
```yaml
    POOLS:
      - name: "web_pool1"
        servicePort: 80
        serverAddresses: 
          - "192.0.1.10"
          - "192.0.1.11"
      - name: "web_pool2"
        servicePort: 80
        serverAddresses: 
          - "192.0.2.10"
          - "192.0.2.11"
      - name: "web_pool3"
        servicePort: 80
        serverAddresses: 
          - "192.0.3.10"
          - "192.0.3.11"
```
The jinja2 template is located in `templates/example2.j2`

> Hands on activity:
> 1. git clone this repo to a system with ansible installed  
> `git clone git@github.com:megamattzilla/as3-tips-and-tricks.git`  
> 
> 2. Build the AS3 schema by running the template via ansible:  
> `ansible-playbook example2.yaml`
> 
> 3. profit
>
> Expected Output:
> ![alt text](2024-08-06_11-17-45.png)
>
> The created AS3 schema is in `files/example2-AS3.json`

___

## #3 Multi-Team: Refer to something outside AS3 <a name="3"></a>

In example #2, we created virtual servers and pools in AS3 schema. 

Sometimes, its helpful to have a virtual server created in AS3 reference a profile or policy that is created outside of AS3. 

You have two teams co-managing a device using AS3. 
- Team1 authors AS3 schema to create LTM objects
- Team2 manages an AWAF policy in Big-IP UI    

> Team1 could create the AWAF policy in AS3, however Team2 would be unable to use the Big-IP UI to make policy changes.  

Team1 could simply refer to the AWAF policy created by Team2 using the `bigip` pointer: 
```yaml
            service:
                class: Service_HTTP
                virtualAddresses:
                    - 10.0.1.10
                pool: web_pool
                policyWAF:
                    bigip: /Common/Team2_AWAF_Policy
```
Any changes Team2 makes to the AWAF policy `Team2_AWAF_Policy` will occur directly and without any change in AS3. 

___

## #4 Multi-Team: linked AS3 Workflow <a name="4"></a>

You have two teams co-managing a device using AS3. 
- Team1 can author a full AS3 post with entire AS3 schema
- Team2 needs to modify a small subset of AS3 schema. Cannot author a full AS3 post.   

Team1 and Team2 can use a linked workflow that will always independently generate the same AS3 desired state. 

Both teams have a workflow that refers to a shared source-of-truth such as a variable file stored in git.  

Example4 Use-Case:
- Team1 manages a complex AS3 schema with virtuals, pools, datagroups
- Team2 needs to update a datagroup quickly without re-running AS3

