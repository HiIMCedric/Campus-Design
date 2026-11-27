# Enterprise Campus Network Security & Application Defense

A four-subnet campus simulation built with Containerlab. An Alpine router enforces a
stateful default-deny iptables policy, and an OWASP ModSecurity WAF protects a
deliberately vulnerable web application (OWASP Juice Shop).

> Juice Shop is intentionally insecure and the database uses the lab password
> `postgres`. Run this only on an isolated lab machine.

## 1. Topology

```
 student-client        faculty-client        admin-client
  10.10.10.10           10.10.20.10           10.10.30.10
       |                     |                     |
   sw-student            sw-faculty             sw-admin
       |                     |                     |
  eth1 | 10.10.10.1     eth2 | 10.10.20.1     eth3 | 10.10.30.1
       +---------------------+---------------------+
                             |
                          router   (iptables firewall)
                             | eth4  10.10.40.1
                             |
                        sw-services
        +--------------------+--------------------+
        |                    |                    |
    waf-proxy          juiceshop-app          postgres-db
   10.10.40.80          10.10.40.81           10.10.40.100
     :8080                 :3000                  :5432
```

| Subnet   | Network       | Router interface | Gateway    |
|----------|---------------|------------------|------------|
| Student  | 10.10.10.0/24 | eth1             | 10.10.10.1 |
| Faculty  | 10.10.20.0/24 | eth2             | 10.10.20.1 |
| IT Admin | 10.10.30.0/24 | eth3             | 10.10.30.1 |
| Services | 10.10.40.0/24 | eth4             | 10.10.40.1 |

## 2. Repository layout

```
.
├── README.md
├── lab.clab.yml
├── config/
│   ├── iptables.rules
│   └── modsec-custom.conf
└── images/
    ├── juiceshop/Dockerfile
    └── modsec/Dockerfile
```

## 3. Prerequisites

- A Linux host (or Linux VM / WSL2) with root access
- Docker Engine
- Containerlab ([installation guide](https://containerlab.dev/install/))
- Internet access: the image builds download packages, and the Alpine nodes run
  `apk add` during deployment
- About 8 GB of RAM and 10 GB of free disk are a comfortable allowance for the
  Juice Shop build

## 4. Build the local images

Run from the repository root. Both must be built before deploying.

```bash
docker build -t custom-juiceshop:latest ./images/juiceshop
docker build -t custom-waf:latest ./images/modsec
```

The Juice Shop build compiles the application from source and can take 10 minutes
or more. It clones the latest upstream code, so a future upstream change could
affect the build.

## 5. Deploy

```bash
sudo clab deploy -t lab.clab.yml
```

Juice Shop needs a little while to start. Until it is ready the WAF returns `502`.
This command waits for it:

```bash
until [ "$(docker exec clab-campus-network-security-student-client \
  curl -s -m 3 -o /dev/null -w '%{http_code}' http://10.10.40.80:8080/)" = 200 ]; do
  sleep 5; echo "waiting for Juice Shop..."
done; echo ready
```

## 6. Security policy

### Firewall (`config/iptables.rules`)

| Source                    | Destination             | Port     | Action |
|---------------------------|-------------------------|----------|--------|
| Any, established/related  | Any                     | any      | ACCEPT |
| IT Admin                  | Any campus subnet       | tcp/22   | ACCEPT |
| Everyone else             | Any, including router   | tcp/22   | DROP   |
| Student, Faculty, Admin   | WAF 10.10.40.80         | tcp/8080 | ACCEPT |
| Any routed source         | Backend 10.10.40.81     | any (except Admin SSH) | DROP |
| Faculty, Admin            | Database 10.10.40.100   | tcp/5432 | ACCEPT |
| Student and others        | Database 10.10.40.100   | tcp/5432 | DROP   |
| Anything else             | Anything                | any      | DROP (default policy) |

- Spoofed source addresses are dropped on each router interface.
- IP forwarding is switched on only after the rules load successfully.
- The WAF reaches the backend directly inside the Services subnet.
- Ping between subnets is not permitted by the policy, so it fails by design.
  Pinging your own gateway works.

### WAF (`config/modsec-custom.conf`)

The file is loaded after the OWASP Core Rule Set (CRS).

| Rule ID       | Detects                                       | Example           |
|---------------|-----------------------------------------------|-------------------|
| 10001 / 10002 | Quote-based tautology                         | `' OR '1'='1`     |
| 10003 / 10004 | `UNION SELECT`, numeric tautology             | `1 OR 1=1`        |
| 10005 / 10006 | Quote followed by `--` or a stacked statement | `admin'--`        |

Odd IDs inspect query strings and cookies in phase 1, so they block before the CRS
reaches its verdict. Even IDs inspect request arguments (including JSON bodies) in
phase 2. For body attacks the CRS usually blocks first, and the custom rules are a
second layer.

These are signature rules for a lab. They do not cover every injection technique,
and unusual input that looks like SQL (for example a password containing `'--`)
is rejected.

## 7. Verification

### Test 1: legitimate web access through the WAF (expect `200`)

```bash
docker exec -it clab-campus-network-security-student-client \
  curl -s -o /dev/null -w "%{http_code}" http://10.10.40.80:8080/
```

### Test 2: SQL injection blocked (expect `403`)

```bash
docker exec -it clab-campus-network-security-student-client \
  curl -s -o /dev/null -w "%{http_code}" "http://10.10.40.80:8080/rest/user/login?email='OR'1'='1"
```

Show that the custom rule did the blocking (expect a line containing `10001`):

```bash
docker logs clab-campus-network-security-waf-proxy 2>&1 | grep CAMPUS-WAF | tail -1
```

### Test 3: Faculty can reach the grading database (expect `succeeded`)

```bash
docker exec -it clab-campus-network-security-faculty-client nc -zv -w 3 10.10.40.100 5432
```

### Test 4: Students cannot reach the database (expect a timeout)

```bash
docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 5432
```

### Test 5: IT Admin SSH is forwarded (expect `Connection refused`)

The packet reaches the database host, where nothing listens on port 22.

```bash
docker exec -it clab-campus-network-security-admin-client nc -zv -w 3 10.10.40.100 22
```

### Test 6: Student SSH is dropped (expect a timeout)

```bash
docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 22
```

### Run all six at once

Paste into a shell after the lab reports ready:

```bash
P=clab-campus-network-security
check() {  # name, expected text, command...
  name=$1; want=$2; shift 2
  out=$(docker exec "$@" 2>&1)
  case "$out" in
    *"$want"*) echo "PASS  $name" ;;
    *)         echo "FAIL  $name -> $out" ;;
  esac
}
check "1 web via WAF"      200       $P-student-client curl -s -o /dev/null -w "%{http_code}" http://10.10.40.80:8080/
check "2 SQLi blocked"     403       $P-student-client curl -s -o /dev/null -w "%{http_code}" "http://10.10.40.80:8080/rest/user/login?email='OR'1'='1"
check "3 faculty to DB"    succeeded $P-faculty-client nc -zv -w 3 10.10.40.100 5432
check "4 student DB drop"  "timed out" $P-student-client nc -zv -w 3 10.10.40.100 5432
check "5 admin SSH"        refused   $P-admin-client   nc -zv -w 3 10.10.40.100 22
check "6 student SSH drop" "timed out" $P-student-client nc -zv -w 3 10.10.40.100 22
```

### Additional evidence

Direct backend access is blocked (expect a timeout):

```bash
docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.81 3000
```

SQL injection in a JSON login body is blocked (expect `403`):

```bash
docker exec -it clab-campus-network-security-student-client \
  curl -s -o /dev/null -w "%{http_code}" -H "Content-Type: application/json" \
  -d "{\"email\":\"' OR 1=1--\",\"password\":\"x\"}" http://10.10.40.80:8080/rest/user/login
```

Firewall packet counters:

```bash
docker exec -it clab-campus-network-security-router iptables -L -n -v
```

## 8. Design notes

- **Default route.** Containerlab gives each node a default route through its
  management interface (`eth0`), so the topology uses `ip route replace`.
- **No management-network bypass.** After installing packages, the three clients
  disable `eth0`. All their traffic must then cross the campus router. Use
  `docker exec` to open a shell on them.
- **Management plane.** Servers, switches and the router keep `eth0`, and the
  Docker host can reach them directly. That path is outside the simulated campus.

## 9. Troubleshooting

| Symptom | Check |
|---------|-------|
| `502` from the WAF | Juice Shop is still starting: `docker logs clab-campus-network-security-juiceshop-app` |
| Every cross-subnet test times out | `docker exec clab-campus-network-security-router sysctl net.ipv4.ip_forward` should be `1`; look for failed exec commands in the deploy output |
| `apk add` fails during deploy | Confirm containers have internet access, then redeploy |
| Networking is lost after a container or host restart | Destroy the lab and deploy it again so every node is reconfigured |

## 10. Destroy

```bash
sudo clab destroy -t lab.clab.yml
```

Add `--cleanup` to also delete the generated lab directory.
