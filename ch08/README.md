# Chapter 8: Protocols

The examples below assume you have netcat (`nc`) installed on your machine.
Netcat is like telnet, but easier to script.

## Listing 8.x: Virtualizing a TCP updateInventory call

````
mb --configfile examples/updateInventory.json &

# Should respond with "0\n1343"
echo "updateInventory\n5131\n-5" | nc localhost 3000

mb stop
````

## Listing 8.x: Using a TCP proxy

````
mb --configfile examples/textProxy.json &

# Should respond with "0\n1343"
# Note the -q 1 flag, which unfortunately doesn't work on my Mac but does on Linux
# It prevents netcat from prematurely closing the connection while mountebank negotiates the proxy
# You can leave it off and the first call will fail (by the time mb tries to respond, the
# nc client will have closed the connection), but the subsequent calls will work because
# mountebank saved the response
echo "updateInventory\n5131\n-5" | nc -q 1 localhost 3000

# Should respond with "0\n1343" (proxy serving saved response)
echo "updateInventory\n5131\n-5" | nc localhost 3000

# Should respond with "0\n0" if you go to the origin server
echo "updateInventory\n5131\n-5" | nc localhost 3333

mb stop
````

## Listing 8.x: Using a TCP proxy with predicate generators

````
mb --configfile examples/proxyWithPredicateGenerators.json &

# Should respond with "0\n1343"
echo "updateInventory\n5131\n-5" | nc -q 1 localhost 3000

# Should respond with "0\n1343" (saved response, no -q needed)
echo "updateInventory\n5131\n-5" | nc localhost 3000

# Should respond with "0\0"
echo "updateInventory\n5040\n-5" | nc -q 1 localhost 3000

# Should respond with "0\0" (saved response, no -q needed)
echo "updateInventory\n5040\n-5" | nc localhost 3000

# See the generated stubs
curl http://localhost:2525/imposters/3000

mb stop
````

## Listing 8.x: Using a TCP proxy with predicate generators with an XML payload

````
mb --configfile examples/proxyWithPredicateGeneratorsXML.json &

# Write first test data
cat <<EOF > mbtemp
<functionCall>
  <functionName>updateInventory</functionName>
  <parameters>
    <parameter name="productId" value="5131" />
    <parameter name="amount" value="-5" />
  </parameters>
</functionCall>
EOF

# Should respond with 1343 inventory
cat mbtemp | nc -q 1 localhost 3000

# Should respond with 1343 (saved response, no -q needed)
cat mbtemp | nc localhost 3000

# Write second test data
cat <<EOF > mbtemp
<functionCall>
  <functionName>getInventory</functionName>
  <parameters>
    <parameter name="productId" value="5040" />
  </parameters>
</functionCall>
EOF

# Should respond with 0 inventory
cat mbtemp | nc -q 1 localhost 3000

# Should respond with 0 (saved response, no -q needed)
cat mbtemp | nc localhost 3000

rm mbtemp

# Show new stubs
curl http://localhost:2525/imposters/3000

mb stop
````

## Listing 8.x - 8.x: Converting to Base64 and back

````
# Encodes in base64
node -e 'console.log(new Buffer("Hello, world!").toString("base64"));'

# Decodes from base64
node -e 'console.log(new Buffer("SGVsbG8sIHdvcmxkIQ==", "base64").toString("utf8"));'
````

## Listing 8.x: Configuring mountebank for a binary response

````
mb --configfile examples/binaryResponse.json &

# Returns "Hello, world!" to the console
echo "Testing" | nc localhost 3000

mb stop
````

## Listing 8.x: Using a Base64-encoded contains predicate

````
mb --configfile examples/binaryContains.json &

# Should return 'Did not match' since it doesn't contain 0x49 0x11
node -e 'console.log(new Buffer([0x10, 0x49, 0x10]).toString("base64"));' | base64 --decode | nc localhost 3000

# Should return 'Matched' since it doesn't start with 0x49 0x11
node -e 'console.log(new Buffer([0x10, 0x49, 0x11]).toString("base64"));' | base64 --decode | nc localhost 3000

mb stop
````