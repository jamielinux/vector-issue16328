# Steps to reproduce

## Docker environment

If you have a docker environment environment already, skip this bit.

Create a compute instance (eg, a DigitalOcean droplet) with Ubuntu 22.04 and
run the following as root:

```
apt-get update && DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y && reboot
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## Generate certificates

I already generated certs in the `tls` folder. These are the commands I ran:

```bash
wget https://github.com/smallstep/cli/releases/download/v0.23.2/step-cli_0.23.2_amd64.deb
dpkg -i ./step-cli_0.23.2_amd64.deb
mkdir tls
cd tls
step certificate create --profile root-ca "Root CA" ./root_ca.pem ./root_ca.key.pem --no-password --insecure
step certificate create --profile intermediate-ca "Intermediate CA" ./intermediate_ca.pem ./intermediate_ca.key.pem \
    --no-password --insecure --ca ./root_ca.pem --ca-key ./root_ca.key.pem
create vector-sink ./vector-sink.pem ./vector-sink.key.pem --profile leaf --no-password --insecure \
    --ca ./intermediate_ca.pem  --ca-key ./intermediate_ca.key.pem --bundle
create vector-source ./vector-source.pem ./vector-source.key.pem --profile leaf --no-password --insecure \
    --ca ./intermediate_ca.pem  --ca-key ./intermediate_ca.key.pem --bundle
```

## Run the test case

```bash
cd /root
git clone https://github.com/jamielinux/vector-issue16328.git
cd vector-issue16328
docker compose up -d
docker compose logs -f
```

You'll get errors from `vector-sink` container that look like this:

```
vector-sink    | 2023-02-07T17:03:50.375059Z ERROR vector::topology::builder: msg="Healthcheck failed." error=Request failed: status: Unavailable, message: "error trying to connect: error:14094418:SSL routines:ssl3_read_bytes:tlsv1 alert unknown ca:ssl/record/rec_layer_s3.c:1555:SSL alert number 48", details: [], metadata: MetadataMap { headers: {} } component_kind="sink" component_type="vector" component_id=to_vector component_name=to_vector
```

### Test with root cert

To change `tls.ca_file` from a client cert to a root cert, comment out line 24
and uncomment line 25. The demo logs will begin working (visible in the logs
for the `vector-source` container).

``` 
23    # Use one of the following two lines. root_ca.pem works. vector-sink.pem doesn't.
24    - ./tls/vector-sink.pem:/etc/vector/ca.pem:ro
25    # - ./tls/root_ca.pem:/etc/vector/ca.pem:ro
```
