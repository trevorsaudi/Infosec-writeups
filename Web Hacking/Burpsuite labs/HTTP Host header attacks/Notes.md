###What is the HTTP Host header?

-HTTP host header is a mandatory request header as of HTTP/1.1 specifies the domain name that the client wants to access

`For example, when a user visits https://portswigger.net/web-security, their browser will compose a request containing a Host header as follows:
GET /web-security HTTP/1.1
Host: portswigger.net`

-In some cases the request is forwarded by an intermediary system and the Host value may be altered before reaching the intended back-end component.

#Purpose of the HTTP Host header?
-If requests didn't contain Host headers, or if the Host header was malformed in some way, this could lead to issues when routing incoming requests to the intended application. 

#scenarios

1. Virtual hosting
- When a web server hosts multiple websites or applications. eg multiple websites with a single owner. Also possible for websites with different owners to be hosted on a single shared platform.
-Even though all these distinct websites have a different domain name, they may still share a similar IP address.

2. Routing traffic via an intermediary
- When websites are hosted on distinct back-end servers, but all traffic between the client and servers is routed through an intermediary system like a load balancer or a reverse proxy. 
- Even though hosted on different back end servers, they still resolve to a single IP address of the intermediary component.
- Challenge to virtual hosting as the reverse proxy or load balancer needs to know the appropriate back-end to which to route the request


#How does the HTTP Host header solve this problem?
- Host header is relied to specify the intended recipient. When a browser sends the request the target url resolves to the IP address of a particular server. It refers to the Host header to determine the intended back-end and forwards the request accordingly

#What is an HTTP Host header attack?
- If the server implicitly trusts the Host header, and fails to validate or escape it properly, an attacker may be able to use this input to inject harmful payloads that manipulate server-side behavior.
- Off-the-shelf web applications typically don't know what domain they are deployed on unless it is manually specified in a configuration file during setup.
- When they need to know the current domain, for example, to generate an absolute URL included in an email, they may resort to retrieving the domain from the Host header: 

`<a href="https://_SERVER['HOST']/support">Contact support</a> `

- The host header is user controllable. If not properly validated, it's a potential vector for exploiting a range of other vulnerabilties eg Web cache poisoning, Business logic flaws, Routing based SSRF, SQLi


#How do Host header vulnerabilities arise

- Flawed assumption that the header is not user controllable. Creates implicit trust in the host header and results in inadequate validation or escaping of its value
- Insecure configuration of componenets in the infrastructure

###Exploiting HTTP Host header vulnerabilities
-  To test whether a website is vulnerable to attack via the HTTP Host header, you will need an intercepting proxy, such as Burp Proxy, and manual testing tools like Burp Repeater and Burp Intruder.

- In short, you need to identify whether you are able to modify the Host header and still reach the target application with your request. If so, you can use this header to probe the application and observe what effect this has on the response.



16efbcee480f1bb7a0db0fdd5f66bad6