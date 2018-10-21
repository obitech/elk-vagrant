$basic = <<SCRIPT
echo "==> Adding packages"
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-6.x.list

echo "==> Installing basic packages"
apt-get update
apt-get install -y vim curl wget build-essential tree bash-completion apt-transport-https unzip openjdk-8-jre stress apache2-utils

echo "==> Grabbing dotfiles"
curl -s https://raw.githubusercontent.com/obitech/dotfiles/master/.vimrc -o /home/vagrant/.vimrc
curl -s https://raw.githubusercontent.com/obitech/dotfiles/master/.bashrc -o /home/vagrant/.bashrc
sed -i 's/colorscheme*/colo default/g' /home/vagrant/.vimrc
SCRIPT

$apache = <<SCRIPT
echo "==> Installing Apache"
apt-get update
apt-get install -y apache2
echo "ServerName ${APACHE_HOST}" >> /etc/apache2/apache2.conf
systemctl restart apache2
SCRIPT

$elk = <<SCRIPT
############# ELASTICSEARCH ############# 
echo "==> Installing ElasticSearch 6.x"
apt-get update && apt-get install -y elasticsearch

sed -i "s/^#network\.host.*/network.host: \"$ELASTICSEARCH_HOST\"/g" /etc/elasticsearch/elasticsearch.yml 

systemctl enable elasticsearch
systemctl start elasticsearch

############# KIBANA ############# 
echo "==> Installing Kibana 6.x"
apt-get install -y kibana

sed -i "s%^#server\.host.*%server.host: \"$ELASTICSEARCH_HOST\"%g" /etc/kibana/kibana.yml
sed -i "s%^#elasticsearch\.url.*%elasticsearch.url: \"http://$ELASTICSEARCH_HOST:9200\"%g" /etc/kibana/kibana.yml

systemctl enable kibana
systemctl start kibana

############# LOGSTASH ############# 
echo "==> Installing Logstash 6.x"
apt-get install -y logstash
cat << EOF > /etc/logstash/conf.d/first-pipeline.conf
input {
  beats {
    port => "5044"
  }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  geoip {
    source => "clientip"
  }
}

output {
  elasticsearch {
    hosts => [ "${ELASTICSEARCH_HOST}:9200" ]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}" 
  }
  stdout { codec => rubydebug }
}
EOF
systemctl enable logstash
systemctl start logstash
SCRIPT

$beats = <<SHELL
############# FILEBEAT ############# 
echo "==> Installing Filebeat 6.x"
apt-get update
apt-get install -y filebeat

# curl -sSL https://download.elastic.co/demos/logstash/gettingstarted/logstash-tutorial.log.gz | gunzip > /home/vagrant/example.log

mv /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak
cat << EOF > /etc/filebeat/filebeat.yml
filebeat.prospectors:
- type: log
  enabled: true
  paths:
    - /var/log/apache2/access.log
output.logstash:
  hosts: ["${ELASTICSEARCH_HOST}:5044"]
EOF

systemctl enable filebeat
systemctl start filebeat

############# METRICBEAT #############
echo "==> Installing Metricbeat 6.x"
apt-get update
apt-get install -y metricbeat

mv /etc/metricbeat/metricbeat.yml /etc/metricbeat/metricbeat.yml.bak
cat << EOF > /etc/metricbeat/metricbeat.yml
metricbeat.modules:
- module: system
  metricsets:
    - cpu
    - filesystem
    - memory
    - network
    - process
    - process_summary
    - diskio
    - load
  enabled: true
  period: 10s
  processes: ['.*']
  cpu_ticks: false
- module: apache
  metricsets: ["status"]
  enabled: true
  period: 1s
  hosts: ["http://${APACHE_HOST}"]

# Uncomment this if you want to send straight to es
# output.elasticsearch:
#   hosts: ["${ELASTICSEARCH_HOST}:9200"]

# Sending to Logstash
output.logstash:
  hosts: ["${ELASTICSEARCH_HOST}:5044"]

setup.kibana:
  host: "${ELASTICSEARCH_HOST}:5601"
EOF

# Setting up dashboards
/usr/share/metricbeat/bin/metricbeat -c /etc/metricbeat/metricbeat.yml \
    -E "setup.dashboards.directory=/usr/share/metricbeat/kibana" \
    setup --dashboards

systemctl enable metricbeat
systemctl start metricbeat
SHELL

Vagrant.configure("2") do |config|
  os = "bento/ubuntu-16.04"
  base_name = "elk"

  # IP block used for VMs
  ip = "192.168.55"

  # Our main ELK server, running ElasticSearch, Logstash and Kibana
  main_ip = "#{ip}.10"
  main_name = "#{base_name}-main"

  # The IP where the Apache Server will be reachable under 
  apache_ip ="#{ip}.20"
 
  config.vm.define "#{main_name}", primary: true do |main|
    main.vm.box = "#{os}"
    main.vm.host_name = base_name + "-main"
    
    # If you change this IP, you also need to change it in the above $elk script
    main.vm.network "private_network", ip: "#{main_ip}"

    main.vm.provision "basic", 
        type: "shell", 
        inline: $basic
    main.vm.provision "elk", 
        type: "shell", 
        env: {"ELASTICSEARCH_HOST" => main_ip}, 
        inline: $elk

    main.vm.provider "virtualbox" do |vb|
      vb.name = "#{base_name}-main"
      # At least 2048MB of RAM is required for the ELK server
      vb.memory = 2048
    end
  end

  # Node1, running Apache
  config.vm.define "#{base_name}-n1" do |n1|
    n1.vm.box = "#{os}"
    n1.vm.host_name = base_name + "-n1"

    n1.vm.network "private_network", ip: "#{apache_ip}"

    n1.vm.provision "basic", 
        type: "shell",
        inline: $basic
    n1.vm.provision "apache", 
        type: "shell", 
        env: {"ELASTICSEARCH_HOST" => main_ip, "APACHE_HOST" => apache_ip}, 
        inline: $apache
    n1.vm.provision "beats", 
        type: "shell", 
        env: {"ELASTICSEARCH_HOST" => main_ip, "APACHE_HOST" => apache_ip}, 
        inline: $beats

    n1.vm.provider "virtualbox" do |vb|
      vb.name = "#{base_name}-n1"
      vb.memory = 1024
    end
  end

end
