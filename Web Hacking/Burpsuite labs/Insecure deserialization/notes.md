#Insecure deserialization
- Identifying insecure deserialization is relatively simple regardless of whether you are whitebox or blackbox testing
- Look at all data being passed into the website and try to identify anything that looks like serialized data

###PHP serialization format

- PHP uses a mostly human-readable string format, with letters representign the data types and numbers representing the length of each entry. For example, consider User object with the attributes

```
$user->name = "carlos";
$user->isLoggedIn= true;
```
When serialized, this object may look something like this:

```
O:4:"User":2:{s:4:"name":s:6:"carlos"; s:10:"isLoggedIn":b:1;}
```
- This can be interpreted as follows:

```

    O:4:"User" - An object with the 4-character class name "User"
    2 - the object has 2 attributes
    s:4:"name" - The key of the first attribute is the 4-character string "name"
    s:6:"carlos" - The value of the first attribute is the 6-character string "carlos"
    s:10:"isLoggedIn" - The key of the second attribute is the 10-character string "isLoggedIn"
    b:1 - The value of the second attribute is the boolean value true

```
- The native methods for PHP serialization are serialize() and unserialize(). If you have source code access, you should start by looking for unserialize() anywhere in the code and investigating further

###Java serialization format
- Java uses binary serialization formats. This is more difficult to read.
- Serialized Java objects always begin with the same butes which are encoded as ``` ac ed ``` in hexadecimal and ``` ro0``` in Base64
- Any class that implements the interface ```java.io.Serializable``` can be serialized and deserialized. If you have source code accessm take note of any code that uses the ```readObject()``` method, which is used to read and deserialize data from ```InputStream```


###Manipulating serialized objects
- Exploiting some deserialization vulnerabilities can be easy as changing an attribute in a serialized object. 
- As the object state is persisted, you can study the serialized data to identify and edit interesting attribute values
- You can then pass the malicious object into the website via its deserialization process. This is the initial step for a basic deserialization exploit. 

###Modifying object attributes
- When tampering with the data, as long as the attacker preserves a valid serialized object, the deserialization process will create a server-side object with the modified attribute values. 
-  As a simple example, consider a website that uses a serialized User object to store data about a user's session in a cookie. If an attacker spotted this serialized object in an HTTP request, they might decode it to find the following byte stream:
```O:4:"User":2:{s:8:"username":s:6:"carlos"; s:7:"isAdmin":b:0;} ```
- Changing the boolean value to 1, may overrite the current cookie with its modified value. 


###What is the impact of insecure deserialization?

The impact of insecure deserialization can be very severe because it provides an entry point to a massively increased attack surface. It allows an attacker to reuse existing application code in harmful ways, resulting in numerous other vulnerabilities, often remote code execution.

Even in cases where remote code execution is not possible, insecure deserialization can lead to privilege escalation, arbitrary file access, and denial-of-service attacks