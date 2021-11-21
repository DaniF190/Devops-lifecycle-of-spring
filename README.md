**Part A - Basic git server setup**
-
***
***Vagrant - this tool is useful for setting up virtual machines.***
1. To use Vagrant, you need to configue a Vagrantfile, but for now we will just use a basic configuration with 2 cpus and 2 gb of ram.
2. Create ssh-key folder wher Vagrantfile is located.
3. Generate ssh keypair in ssh-key folder with cd ssh-key && ssh-keygen -f -C "vagrant@buster", do not use passphrase when its ask, just hit enter.
4. Add provisioning in Vagrantfile that sets another public key in .ssh/authorized_keys file inside vagrantbox (replace the part PUT_CONTENT_OF_THE_ID_RSA_PUB_FILE with the the content of ssh-key/ida_rsa.pub)

        config.vm.provision "shell" do |s|
            s.inline = <<-SHELL
            echo "PUT_CONTENT_OF_THE_ID_RSA_PUB_FILE_HERE" >> /home/vagrant/.ssh/authorized_keys
            SHELL
        end
5. Start vagrantbox with: vangrant halt and vagrant up --provision
6. Check that you can connect to the box with Fix IP and using the ssh-key ssh -o StrictHostKeyChecking=no -i ssh-key/id_rsa vagrant@192.168.33.4

***Ansible - this is a powerful scripting tool for setting up environments in virtual machines***
1. The next section happens in the virtual machine!!!
2. run the following commands:
   sudo apt-get update
   sudo apt-get install ansible -y
3. Check the existence of the tool on system by running "ansible --version" command. Expected output should be:
    ansible 2.9.16 (it might differ the time you read this)<br/>
    config file = ../ansible/ansible.cfg<br/>
    configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']<br/>
    ansible python module location = /usr/lib/python3/dist-packages/ansible<br/>
    executable location = /usr/bin/ansible<br/>
    python version = 3.7.3 (default, Jan 22 2021, 20:04:44) [GCC 8.3.0]
4. Create ssh keypair inside vagrantbox by running command ssh-keygen, do not use passphrase when its ask, just hit enter
5. Add created public ssh key to the .ssh/authorized_keys file by running the following commands:
   cd ~/.ssh
   cat id_rsa.pub >> authorized_keys
6. Now exit the virtual machine/use another terminal for the next command and on 8. return.
7. Transfer ansible project into vagrantbox e.g. scp -o StrictHostKeyChecking=no -i ssh-key/id_rsa -r ../../elte-devops/gyakorlat3 vagrant@192.168.33.4:.
8. Go to task3/ansible folder and test ssh connection by running ansible -i inventory all -m ping command. Expected output should be:
    localhost | SUCCESS => {<br/>
        "ansible_facts": {<br/>
            "discovered_interpreter_python": "/usr/bin/python"<br/>
        },<br/>
        "changed": false,<br/>
        "ping": "pong"
    }

***Git - The tool is most definitely used by any dev - version management tool***
1. Run the the following command:
   ansible-playbook playbooks/create-git-server.yml -i inventory
2. Check if the repository is accessible buy running the following command:
   git clone git@localhost:/home/git/spring-devops

**Part B - Spring boot application and documentation**
-
***
***Spring boot application - going to monitor this***
1. Because this assingment is not focused on programming, but devops, we are going to skip this part. Basically just create any type of rest application with some basic API endpoints configured to it. This guid uses maven, if you use gradle, you should use other methods at some points.

***Open API 3.0 - Swagger - with this fairly new and useful tool you can document your api endpoint with pretty nice design***
1. Add swagger dependencys to your pom.xml.

        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-ui</artifactId>
            <version>1.5.12</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>3.0.0</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>3.0.0</version>
        </dependency>

2. Configure the endpoints for the documentation and swagger-ui in your application.properties:

        springdoc.api-docs.path=/api-docs
        spring.mvc.pathpattern.matching-strategy=ant_path_matcher
        springdoc.swagger-ui.path=/swagger-ui.html

3. Now you can access trough these endpoints the nice looking swagger ui.

        localhost:8080/swagger-ui.html

**Part C - Monitoring your application**
-
***
***Adding prometheus metrics to your application***
1. First you need to add actuator dependency to your code that will give some default statitics.

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
            <version>2.6.0</version>
        </dependency>

2. You can curl the actuator endpoint for the default stats of your endpoints.

        curl http://localhost:8080/actuator | python -mjson.tool

3. After this you can add prometheus to your dependency and you need one more line to your application.properties.

        management.endpoints.web.exposure.include=health,info,metrics,prometheus

    b. pom.xml: 

        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <version>1.8.0</version>
        </dependency>

4. Try out the new request:

        curl http://localhost:8080/actuator/metrics/http.server.requests | python -mjson.tool

***Running prometheus docker***

1. To be able to see the statistics in a nicer environment you need a prometheus docker and config for it.<br/> Create a prometheus.yml file in your desired location and add the following code.<br/> All you need to change is the first target ip as 'host.docker.internal only works for windows and the HOST_IP to your own ip as localhost wont work here.

        # my global config
        global:
        scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
        evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
        # scrape_timeout is set to the global default (10s).

        # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
        rule_files:
        # - "first_rules.yml"
        # - "second_rules.yml"

        # A scrape configuration containing exactly one endpoint to scrape:
        # Here it's Prometheus itself.
        scrape_configs:
        # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
        - job_name: 'prometheus'
            # metrics_path defaults to '/metrics'
            # scheme defaults to 'http'.
            static_configs:
            - targets: ['host.docker.internal:9090']

        - job_name: 'spring-actuator'
            metrics_path: '/actuator/prometheus'
            scrape_interval: 5s
            static_configs:
            - targets: ['HOST_IP:8080']

2. Start the docker with the following command, if you didn't put your config file in C:\tmp\ then change the first path variable.

        docker run -d --name prometheus -p 9090:9090 -v C:\tmp\prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus --config.file=/etc/prometheus/prometheus.yml

3. Now you able to see to your application statistics if you chose any method in the search and execute. For basic stats use the following:

        http_server_requests_seconds_max

***Adding prometheus to grafana***

1. For the maximum beauty of display we can run another tool on docker and add prometheus to it.<br/> Run the following command:

        docker run --name grafana -d -p 3000:3000 grafana/grafana

2. Now you can acces this page at localhost:3000. At this page a login will welcome you, on default the login credentials are admin/admin, after the login you can change the password.
3. Inside click the add data source button and add prometheus.
4. Fill the url with your prometheus url and set http access to browser and click save & test.
5. In the left sidebar, click the + sign and choose Import. Enter the url given below and load:

        https://grafana.com/grafana/dashboards/4701

6. Enter a meaningful name and select your prometheus data source and import.
7. Change the range to 30 minutes as you only just started your server and enjoy the nice looking statistics of your spring boot application.