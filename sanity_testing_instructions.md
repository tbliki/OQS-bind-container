# Sanity Testing Instructions
## Initialisation
```bash
# First we build the dockerfile as usual:
podman build -f Dockerfile --tag=pqc-bind-oqs-snova --no-cache

# We work in a tmp directory
export TESTDIR=$(mktemp -d)
cd $TESTDIR

# Create an example testing zone
cat <<EOF >example.nl
\$ORIGIN example.nl.
\$TTL 86400
@       IN      SOA     dns1.example.nl.        hostmaster.example.nl. (
                        2001062501 ; serial
                        21600      ; refresh after 6 hours
                        3600       ; retry after 1 hour
                        604800     ; expire after 1 week
                        86400 )    ; minimum TTL of 1 day
        IN      NS      dns1.example.nl.
        IN      NS      dns2.example.nl.
        IN      MX      10      mail.example.nl.
        IN      MX      20      mail2.example.nl.
dns1    IN      A       10.0.1.1
dns2    IN      A       10.0.1.2
server1 IN      A       10.0.1.5
server2 IN      A       10.0.1.6
ftp     IN      A       10.0.1.3
        IN      A       10.0.1.4
mail    IN      CNAME   server1
mail2   IN      CNAME   server2
www     IN      CNAME   server1
EOF

# Obtain KSK (key signing key) and ZSK (zone signing key) for both algorithms
podman run -it -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-keygen -a SNOVA37172 -K /dns -f KSK example.nl
podman run -it -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-keygen -a SNOVA2454 -K /dns -f KSK example.nl
podman run -it -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-keygen -a SNOVA37172 -K /dns example.nl
podman run -it -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-keygen -a SNOVA2454 -K /dns example.nl

# Sign our zone (until 2030), automatically validating it as well
podman run --rm -it -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-signzone -S -s '20241111111111' -j 0 -e '20301111111111' -K /dns -f /dns/example.nl.signed -o example.nl. /dns/example.nl

# Obtain DS record (fingerprint of KSK) for putting in parent zone, save in $TESTDIR
# Ignore NSEC already added warning by writing it to /dev/null with 2>
podman run --rm -i -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-dsfromkey -K /dns -f /dns/example.nl.signed example.nl > dsrecord.txt 2> /dev/null
```
## Manual Validation:
```bash
# Perform manual validation to double-check:
podman run --rm -it -v $(pwd):/dns localhost/pqc-bind-oqs-snova:latest dnssec-verify -o example.nl. /dns/example.nl.signed
```
## Testing resolver:
It's important before testing the resolver that you find the Key Tag and Hash values from the DS record.
We stored this in a file which you can view with:
```bash
cat dsrecord.txt
```
For our toy example, it should look like:
```txt
example.nl. IN DS <KeyTag> 247 2 <Hash>
example.nl. IN DS <KeyTag> 248 2 <Hash>
```
Make sure to replace the instances of `<KeyTag>` and `<Hash>` with these values below!!
```bash
# Create the BIND configuration to serve the authoritative zone
cat <<EOF > named.conf
options {
    directory "/dns";
    listen-on { any; };
    listen-on-v6 { any; };
    recursion no;
    allow-query { any; };
};

zone "example.nl" IN {
    type master;
    file "/dns/example.nl.signed";
};
EOF

# Run the container in the background
podman run -d --replace --name pqc-validator \
  -v $(pwd):/dns \
  -p 5300:53/udp \
  localhost/pqc-bind-oqs-snova:latest \
  named -c /dns/named.conf -g -d 3

# Execute the validation daemon and test it
# Make sure to replace the initial-ds values with your specific keys from dsrecord.txt
podman exec -it pqc-validator bash -c 'cat <<EOF > /dns/validator.conf
options {
    directory "/dns";
    listen-on port 5353 { 127.0.0.1; };
    recursion yes;
    dnssec-validation yes;
    allow-query { any; };
    forwarders { 127.0.0.1 port 53; };
    forward only;
};

trust-anchors {
    "example.nl" static-ds <KeyTag> 247 2 "<Hash>";
    "example.nl" static-ds <KeyTag> 248 2 "<Hash>";
};
EOF
named -g -c /dns/validator.conf -p 5353 & 
sleep 2 
dig @127.0.0.1 -p 5353 www.example.nl A +dnssec
'
```
If it is properly validated, expect to see the `ad` flag returned.
