**ELK Stack** (elastic.co)

------

why we use elk?

- issue debugging
- performance analysis
- security analysis
- predictive analysis
- use to identify automation operation

type of data

- structured (Structured data have a special pattern, so they are stored according to certain principles and a process is performed on them)
- unstructured

![](elk-stack.png)

##### install elasticsearch

```bash
rpm -ivh elastic-8.10.4-x86_64.rmp
```

> [!NOTE]
>
> After installation, it shows information about the user and password, as well as information about token creation and password recovery

------

##### install kibana

```bash
rpm -ivh kibana-8.10.4-x86_64.rpm
```

important path for configuration

- /etc/
- /var/lib/
- /var/log/
- /usr/share/

> [!IMPORTANT]
>
> In the configuration file of kibana (/etc/kibana/kibana.yml)
>
> Uncommenet **server.port**
>
> Set **server.host: "0.0.0.0"**

```bash
systemctl start kibana.service
systemctl enable kibana.service
systemctl status kibana.service
```

> [!WARNING]
>
> If the system RAM was low, we should increase the value of TimeoutStartSec in the /usr/lib/systemd/system/elasticsearch.service path

```bash
# health check elasticsearch command
curl -k -XGET -u elastic "my-password" https://localhost:9200
```

```bash
# open kibana in browser and then generate an enrollment for kibana
/usr/share/elasticsearch/bin/elastic-create-enrollment-token -s kibana
# copy and paste token to enrolment token for kibana browser
/usr/share/kibana/bin/kibana-verification-code
# get code and verrify your kibana
```

------

##### install filebeat

```bash
rpm -ivh filebeat-8.10.4-x86_64.rpm
```

Because Elastic Search works based on SSL, the modules that are connected to it also need a certificate. Kibana handles this automatically.

> [!NOTE]
>
> We temporarily disable 

`/etc/elasticsearch/elasticsearch.yml`

```yaml
xpac.security.http.ssl: # for clients such as Kibana, Logstash, etc
  enabled: false

xpac.security.transport.ssl: # for components for nodes
  enabled: false
```

`/etc/kibana/kibana.yml`

```bash
# change this line
elasticsearch.hosts: [http://...] # change https to http
elasticsearch.ssl.certificateAthurities # comment
elasticsearch.fleet.outputs #comment
```

------

